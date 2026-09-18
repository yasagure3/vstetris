# 対戦(コアゲームプレイ)

- 種別: 機能設計書
- 対象 UC: UC-004(ステップ5〜11、代替フローA〜D、例外E2〜E4)

## 何を作るか

1vs1のリアルタイムテトリス対戦本編。テトリミノ操作、おじゃまライン攻撃・相殺、盤面同期、切断検知・タイムアウト、決着判定、アクセシビリティ(色覚サポート・キー割当)、対戦結果・リプレイの永続化、進行中対戦レジストリ(KV)の同期を扱う。docs/design/FEASIBILITY.md PoC-1で通信遅延・切断検知・タイムアウトを、PoC-5でHibernation APIでの対戦者+観戦者ブロードキャストとKVレジストリ同期を実測済み。

### 権威分担

サーバー権威検証(不正操作対策)はスコープ外(docs/design/USER_STORIES.md US-026、Won't)。よって**盤面の描画・落下・回転・消去判定はクライアント(`tetrisEngine.ts`)が権威を持ち**、クライアントは「何ライン消したか」だけをサーバへ報告する。**おじゃまラインの量計算(対応表・相殺・`pendingAttack`の値)はBattleRoom DO側が唯一の権威を持つ**(`garbageCalculator`)。これによりクライアント間で盤面がわずかにズレても、対戦の勝敗に関わる攻撃量計算だけは両者で一致する。

## 入出力と振る舞い

| 操作 | 画面に起きること |
|---|---|
| 移動・回転・ハードドロップ・ホールド | 自盤面に即座に反映、`board_update`で相手・観戦者に配信 |
| 2ライン以上同時消去 | `lines_cleared`をサーバへ送信、サーバが攻撃量を計算し相手へ`garbage_incoming`を予告(サーバ権威の`pendingAttack`が変化する) |
| 相手の`pendingAttack`が増える | 相手クライアントの次の着地時に、サーバから配信される最新の`pendingAttack`値をもとに自盤面へおじゃまラインを反映する |
| 2人が揃う | 全員(player+spectator)に`battle_start`が届き、対戦開始 |
| 自分の盤面が最上段まで積み上がる | `game_over`をサーバへ送信、敗北として対戦終了、結果画面へ |
| 相手が切断 | 再接続待ちバナー表示(猶予15秒) |
| 猶予超過 | 自分の不戦勝として結果画面へ |
| 「通報する」を選ぶ | ReportModalを開き、`reporterLabel`(自分の`displayName`)・`targetLabel`(相手の`displayName`、`battle_start.peers`から取得。`joined.peers`は接続時点で自分1件しか含まれない場合があるため使わない)・`roomId`を渡して送信する(features/report.md参照) |
| 色覚サポートON | ブロックに記号を重ねて表示(US-025) |
| キー割当変更 | 以降の操作入力に反映(登録プレイヤーはアカウント設定`account_settings`から、ゲストはブラウザのlocalStorageから対戦開始時に読み込む) |
| 結果画面で「もう一度対戦」 | `/ws/matchmake`への再接続(新規チケット取得から)。同一BattleRoomでの再戦ではなく、通常のマッチメイキングをやり直す扱いとする(UC-004代替フローD。ルーム対戦(features/room.md)で決着した場合も同様にランダムマッチングへ飛ぶ。同じ相手との再戦には新規ルーム作成が必要) |

UI_SKETCH.html「Battle」画面に対応(色覚サポートトグル・切断バナー・デモ用の決着ボタンは実装物には含めない)。

### アクセシビリティ設定の保存先(UC-004代替B/C)

登録プレイヤーは`PATCH /api/account/settings`(features/account.md参照)でD1に保存する。**ゲストは`account_settings`に行を持てない(認証必須)ため、ブラウザのlocalStorage(`vstetris.guestAccessibility`)に保存し、対戦開始時にクライアント側でのみ適用する**(サーバには送らない、サーバは色覚表示に関与しないため問題ない)。

### 盤面スナップショットの形式

クライアント→サーバ(`board_update`、自分の盤面の見た目を配信するための情報のみ。攻撃量はサーバ権威のため含めない):

```json
{
  "cells": [["", "", "I", ...], ...],   // 20行 x 10列。各セルは "" (空) または "I"|"O"|"T"|"S"|"Z"|"J"|"L"|"G"(おじゃま)
  "current": { "type": "T", "x": 4, "y": 0, "rotation": 0 },
  "next": ["I", "O", "T"],
  "hold": "S"
}
```

サーバ→クライアント(中継時。DOが権威の`pendingAttack`を注入して配信する):

```json
{ "cells": [...], "current": {...}, "next": [...], "hold": "S", "pendingAttack": 2 }
```

`board_update`メッセージはこのスナップショットを毎回まるごと送る(差分配信はしない。PoC-1の実測でメッセージサイズによるレイテンシ悪化は確認されなかったため、MVPでは単純さを優先する)。

### おじゃまライン対応表(BR-001の確定値)

| 消去ライン数 | 攻撃力 |
|---|---|
| 1(Single) | 0 |
| 2(Double) | 1 |
| 3(Triple) | 2 |
| 4(Tetris) | 4 |

Back-to-Back・コンボボーナスはMVPスコープ外(Won't相当、将来拡張候補)。

### 相殺計算(BR-002の確定ロジック)

1. 各プレイヤーは「受信待ちの攻撃力」(`pendingAttack`)を1つ保持する(BattleRoom DO側で管理する唯一の権威値)
2. 自分がラインを消すと、まず自分の`pendingAttack`から消去による攻撃力を差し引く(相殺)
3. 差し引いてなお攻撃力が残れば、その分だけ相手の`pendingAttack`に加算する
4. `pendingAttack`は次に自分のミノが着地した瞬間に盤面下部へ「おじゃまライン」として反映され、0にリセットされる。1回の着地で反映する行数に上限は設けない(MVPでは緩和ルールなし)

### 再接続猶予(BR-003の確定値)

**15秒**。`in_progress`状態での切断検知(WebSocket close/error)からこの秒数以内に同一`playerRef.id`で新しいチケットを取得し`/ws/rooms/:roomId?role=player&ticket=<新規チケット>`へ再接続すれば対戦を継続する。超過したらDOの`alarm()`でタイムアウト処理を発火する(PoC-1の実装パターンに準拠)。`waiting`状態での切断には適用しない(下記「BattleRoomの状態機械」参照)。

### BattleRoomの状態機械

- `waiting`: player 1名のみ接続。**最初のplayer接続時にKVレジストリ`match:<roomId>`を`status: "waiting"`で作成する**(features/spectate.mdの一覧取得の前提)。このときKV期限チェックを行う: roomIdが8文字(Crockford Base32、`POST /api/rooms`発行)の場合のみ`room_created:<roomId>`キーの存在(=未失効)を必須とし、無ければ接続をHTTP 404で拒否する。roomIdが26文字(ULID、matchmaking.md発行)の場合はこのチェックを行わない。roomIdがどちらの形式(8文字のCrockford Base32、または26文字のULID)にも一致しない場合もHTTP 404で拒否する(不正なroomIdでの空DO生成を防ぐ)
  - `waiting`のタイムアウトは経路により異なる: roomIdが26文字(マッチメイキング経由)は**15秒**、roomIdが8文字(ルーム対戦経由)は`room_created:`と同じ**10分**(人間がルームIDを共有する時間を確保するため)。超過したら「対戦不成立」として全接続に`{ "type": "error", "code": "ROOM_DISBANDED", "message": "対戦相手が見つからないため解散しました" }`を送信してからcloseし、KVレジストリ(`match:<roomId>`)からも削除する
- `in_progress`: player 2名が揃った瞬間に遷移し、`match:<roomId>`を`status: "in_progress"`に更新、全員(player+spectator)へ`{ "type": "battle_start", "peers": [{ "side": "P1", "displayName": string }, { "side": "P2", "displayName": string }] }`を配信する。`waiting`中の切断は「対戦不成立」の解散として扱い、再接続猶予(BR-003)は適用しない
- `finished`: `battle_end`送信後。`match:<roomId>`をKVから削除する。以後の`input`等は無視する

## API

### WS /ws/rooms/:roomId?role=player|spectator&ticket=<ticket>
- 認証: DESIGN.md横断規約「WebSocket認証(チケット方式)」参照。`role=player`は`ticket`必須(`scope: "room"`、DOが`playerRef`と`displayName`を検証・解決する)。検証失敗(署名不正・期限切れ・scope不一致・使用済み)はHTTP 401でアップグレードを拒否する。`role=spectator`は`ticket`省略可(匿名観戦を許容する。省略時、または`ticket`を付けても`displayName`は常に固定値`"観戦者"`として扱う)
- `role=player`は最大2接続(3人目以降は**WebSocketアップグレード自体をHTTP 409で拒否**する。DESIGN.md横断規約「WebSocket接続の役割分離」参照)。加えて、同一`playerRef.id`が既に`role=player`で接続中の場合、2本目の接続はHTTP 409で拒否する(ルーム対戦での自己対戦・二重枠占有の防止)
- roomId・KV期限・タイムアウトの判定は上記「BattleRoomの状態機械」参照
- クライアント→サーバ(playerのみ):
  - `{ "type": "input", "action": "moveLeft"|"moveRight"|"rotate"|"softDrop"|"hardDrop"|"hold" }`
  - `{ "type": "board_update", "seq": number, "board": <盤面スナップショット(クライアント→サーバ形式)> }`
  - `{ "type": "lines_cleared", "count": number }`
  - `{ "type": "game_over" }`(自分の盤面が最上段まで積み上がった)
- サーバ→クライアント(player + spectator):
  - `{ "type": "joined", "role": "player"|"spectator", "side": "P1"|"P2"|null, "peers": [{ "side": "P1"|"P2", "displayName": string }] }`(`side`は`role=player`のとき自分の確定値、`spectator`はnull。`peers`は接続時点で既にいる対戦者(自分を含む、最大2件)の一覧)
  - `{ "type": "battle_start", "peers": [{ "side": "P1"|"P2", "displayName": string }] }`(2人揃った瞬間に全員へ配信)
  - `{ "type": "board_update", "seq": number, "from": "P1"|"P2", "board": <盤面スナップショット(サーバ→クライアント形式、pendingAttack込み)> }`(中継。観戦者が途中参加した場合は`joined`直後に両者の最新スナップショットを個別送信する)
  - `{ "type": "garbage_incoming", "amount": number, "target": "P1"|"P2" }`
  - `{ "type": "opponent_disconnected" }` / `{ "type": "opponent_reconnected" }`
  - `{ "type": "battle_end", "winner": "P1"|"P2"|null, "reason": "normal"|"forfeit_timeout"|"draw_timeout", "summary": { "P1": { "linesCleared": number, "linesSent": number }, "P2": { "linesCleared": number, "linesSent": number }, "durationMs": number } }`(`winner`は引き分け時null。`summary.P1`/`summary.P2`は消去/送出ラインのみを持つ。`durationMs`は両者共通の値としてトップレベルに置く。フロントは結果表示時、消去/送出ラインは`summary[自分のside]`から、対戦時間は`summary.durationMs`から読む)
  - WebSocketエラー形式: DESIGN.md横断規約「WebSocketエラー応答形式」参照(`ROOM_DISBANDED`等)
- 決着後、BattleRoomはservice binding(`env.API`、DESIGN.md「インフラ」参照)経由で`POST https://internal/api/battle-results`(内部専用)を呼びD1へ永続化する。リクエストヘッダ`X-Internal-Secret: <BATTLE_RESULTS_INTERNAL_SECRET環境変数>`で保護し、Hono側はこのヘッダを検証する(JWT/Cookieによる認証は行わない、DOはユーザーの資格情報を持たないため)。リクエストボディ: `{ "roomId": string, "players": [{ "playerRef": <両者分。登録・ゲストとも含む>, "side": "P1"|"P2", "displayName": string, "result": "win"|"lose"|"draw", "linesCleared": number, "linesSent": number }], "durationMs": number, "endedReason": "normal"|"forfeit_timeout"|"draw_timeout", "replayLog": <inputメッセージのタイムスタンプ付き配列> }`。Hono側の挙動:
  - `players`のうち`playerRef.kind: "registered"`の要素についてのみ`battle_results`行を作成する(ゲストの要素は`battle_results`を作らないが、相手の`opponent_label`の出所として使う)。各行の`opponent_label`は、もう一方の`players`要素の`displayName`をそのまま使う(相手がゲストでも`displayName`は必ず入っているため問題ない)。`opponent_id`は相手が`kind: "registered"`のときだけそのCognito subを設定し、ゲストならnullのままにする
  - `battle_results`行を1件も作らない場合(両者ともゲスト)は`replays`行も作成しない(参照元が無いため)
  - `battle_results`行を作成するたびに、**それぞれに1件ずつ**`replays`行を作成する(`log`の内容はどちらの行も同じ`replayLog`。`replays.id`を複数の`battle_results`行で共有しない。理由: `replays.battle_result_id`は単一行を指すFKであり、共有すると一方の`battle_results.id`からの`GET /api/battle-results/:id/replay`が解決できなくなるため)、`expires_at`は`created_at + 30日`で計算する

## 実装の配置

| 処理 | 層 | 実装先ファイル |
| --- | --- | --- |
| BattleRoom Durable Object本体(WebSocket中継・状態機械・切断/タイムアウト管理) | DO | `src/server/battle/battleRoom.ts` |
| WebSocketアップグレードのルーティング(`/ws/rooms/:roomId`、role判定・roomId形式検証) | adapter | `src/server/modules/battle/adapter/wsRoute.ts` |
| おじゃまライン対応表・相殺計算 | domain | `src/server/modules/battle/domain/garbageCalculator.ts` |
| 対戦終了時のD1永続化(内部API、battle_results・replays INSERT) | adapter | `src/server/modules/battle/adapter/battleResults.ts` |
| KVレジストリ(`match:<roomId>`)の読み書き関数(Honoルートではなく純粋なKVアクセス関数。DOと`modules/spectate/adapter/activeRooms.ts`の双方から呼ばれる) | adapter | `src/server/modules/spectate/adapter/matchRegistry.ts` |
| テトリミノ操作・盤面ロジック(クライアント側の実際のゲームロジック) | front | `src/front/game/tetrisEngine.ts` |
| 対戦画面・色覚サポート表示・切断バナー | front | `src/front/pages/Battle.tsx` |
| キー割当・色覚設定の読み込み・適用(登録: account_settings、ゲスト: localStorage) | front | `src/front/game/keyBindings.ts` |

## エッジケースの決定

- **空・最小データ**: 対戦開始直後(ライン消去0回)でゲームオーバーになるケースも決着として正常処理する
- **上限・境界値**: `pendingAttack`に明示的な上限は設けない(理論上は積み上がり続ける。MVPでは緩和ルールなし、将来拡張候補)
- **エラー時に見えるもの**: WebSocket接続確立に失敗した場合はネットワークエラー画面。3人目以降・二重枠の`role=player`接続は409でアップグレード自体が拒否されるため、UIは観戦へ誘導する。roomId不正・期限切れは404で「ルームが見つかりません」を表示する
- **並行操作・二重実行**: 双方が同時に`game_over`を送った場合(理論上ほぼ同時)は引き分け(`winner: null`, `reason: "normal"`)として扱う
- **再表示時の整合**: `board_update`はプレイヤーごとの`seq`(単調増加)を持たせ、観戦者が途中から接続した場合は`joined`直後に現在の盤面スナップショットを個別送信して同期させる(差分ではなくフルスナップショットを都度送るため、途中参加でも整合する)

## テスト方針

- 単体: `garbageCalculator`(対応表・相殺ロジックの境界値: 1/2/3/4ライン、pendingAttackが0の場合とある場合)
- 結合: WebSocket経由での2クライアント間中継、3人目・二重枠`role=player`接続の409拒否、不正roomId/期限切れの404拒否、`waiting`状態でのタイムアウト(26文字roomIdで15秒、8文字roomIdで10分)による解散、切断検知〜猶予15秒〜タイムアウトのタイマー動作(強制切断・正常切断・猶予内再接続の3パターン。docs/design/FEASIBILITY.md PoC-1で検証したシナリオに準ずる)
- E2E(golden path): マッチング成立 → 双方が操作 → 片方がゲームオーバーになり結果画面へ遷移する一連(Playwright、2ブラウザコンテキスト)
