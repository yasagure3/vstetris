# 対戦(コアゲームプレイ)

- 種別: 機能設計書
- 対象 UC: UC-004(ステップ5〜11、代替フローA〜D、例外E2〜E4)

## 何を作るか

1vs1のリアルタイムテトリス対戦本編。テトリミノ操作、おじゃまライン攻撃・相殺、盤面同期、切断検知・タイムアウト、決着判定、アクセシビリティ(色覚サポート・キー割当)、対戦結果・リプレイの永続化、進行中対戦レジストリ(KV)の同期を扱う。docs/design/FEASIBILITY.md PoC-1で通信遅延・切断検知・タイムアウトを、PoC-5でHibernation APIでの対戦者+観戦者ブロードキャストとKVレジストリ同期を実測済み。

### 権威分担

サーバー権威検証(不正操作対策)はスコープ外(docs/design/USER_STORIES.md US-026、Won't)。よって**盤面の描画・落下・回転・消去判定はクライアント(`tetrisEngine.ts`)が権威を持ち**、クライアントは着地ごとの消去数と盤面をサーバへ報告する。**おじゃまラインの量計算(対応表・相殺・`pendingAttack`の値)はBattleRoom DO側が唯一の権威を持つ**(`garbageCalculator`)。これによりクライアント間で盤面がわずかにズレても、対戦の勝敗に関わる攻撃量計算だけは両者で一致する。

## 入出力と振る舞い

| 操作 | 画面に起きること |
|---|---|
| 移動・回転・ハードドロップ・ホールド | 自盤面に即座に反映、`board_update`で相手・観戦者に配信 |
| 2ライン以上同時消去 | `piece_locked`をサーバへ送信、サーバが攻撃量を計算し相手へ`garbage_incoming`を予告(サーバ権威の`pendingAttack`が変化する) |
| 相手の`pendingAttack`が増える | 相手の次の着地要求をDOが処理し、`lock_result`で確定した行だけを相手盤面へ適用する |
| 2人が揃う | 全員(player+spectator)に`battle_start`が届き、対戦開始 |
| 自分の盤面が最上段まで積み上がる | `game_over`をサーバへ送信、敗北として対戦終了、結果画面へ |
| 相手が切断 | 再接続待ちバナー表示(猶予15秒) |
| 猶予超過 | 自分の不戦勝として結果画面へ |
| 「通報する」を選ぶ | ReportModalを開き、`reporterLabel`(自分の`displayName`)・`targetLabel`(相手の`displayName`、`battle_start.peers`から取得。`joined.peers`は接続時点で自分1件しか含まれない場合があるため使わない)・`roomId`を渡して送信する(features/report.md参照) |
| 色覚サポートON | ブロックに記号を重ねて表示(US-025) |
| キー割当変更 | 以降の操作入力に反映(登録プレイヤーはアカウント設定`account_settings`から、ゲストはブラウザのlocalStorageから対戦開始時に読み込む) |
| 結果画面で「もう一度対戦」 | `/ws/matchmake`への再接続(新規チケット取得から)。同一BattleRoomでの再戦ではなく、通常のマッチメイキングをやり直す扱いとする(UC-004代替フローD。ルーム対戦(features/room.md)で決着した場合も同様にランダムマッチングへ飛ぶ。同じ相手との再戦には新規ルーム作成が必要) |

UI_SKETCH.html「Battle」画面に対応(色覚サポートトグル・切断バナーは実装する。デモ用の決着ボタンのみ実装物に含めない)。

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
4. 着地要求を処理する同じDOトランザクションで1〜3の相殺・送出を行い、その後に残った自分のpendingAttackを消費して0にする。具体的な適用は下記の着地プロトコルに従う。1回の着地で反映する行数にゲーム上の上限は設けない。

### 再接続猶予(BR-003の確定値)

**15秒**。`in_progress`状態での切断検知(WebSocket close/error)からこの秒数以内に同一`playerRef.id`で新しいチケットを取得し`/ws/rooms/:roomId?role=player&ticket=<新規チケット>`へ再接続すれば対戦を継続する。超過したらDOの`alarm()`でタイムアウト処理を発火する(PoC-1の実装パターンに準拠)。`waiting`状態での切断には適用しない(下記「BattleRoomの状態機械」参照)。

### BattleRoomの状態機械

- `waiting`: WorkerまたはMatchmakingQueueの内部RPCによる初期化済みDOだけを入室対象とする。storageにroomId、source(random/private)、createdAt、expiresAt、予約playerRefとsideを保存する。randomは初期化から15秒、privateは作成から10分を入室期限とする。ID形式やKVの有無で入室を許可しない。
- randomは予約2名のみ、privateは作成者P1を予約し、P2は最初の異なる有効playerRefに原子的に割り当てる。席は試合終了まで変わらない。未初期化・waitingで期限切れ・finishedへの対戦接続は拒否する。expiresAtはwaitingの初回入室だけに適用する。in_progressの本人再接続は固定席・接続世代・disconnectDeadlineで判定する。チケット発行時も同じ規則とする。
- randomのwaitingは観戦一覧候補、privateのwaitingは一覧非公開。期限切れ／waiting中の参加者切断／作成者キャンセルはfinishedを永続化し、`ROOM_DISBANDED`通知後closeする。
- `in_progress`: 予約された2名が接続した瞬間に開始。startedAtを一度だけ保存、全員へbattle_startを配信し一覧候補をin_progressへ更新する。
- `settling`: 最初のgame_over受信から100msの判定待ち。DOの受信時刻で期限以下の相手game_overも受信した場合だけ引き分け。その他の新規ゲーム操作を止め、期限にalarmで一度だけ決着する。遅れて届いたgame_overは結果を変えない。
- `finished`: 不変のendedAt・結果・未保存状態を永続化してからbattle_endを送信。KV削除後のcloseでも一覧を再作成しない。結果保存は成功までalarmで再試行する。解散した未開始ルームは戦績を作らない。

DO alarmは一つなので、入室期限・heartbeat期限・再接続期限・決着待ち・KV更新・保存再試行の最短期限を設定する。alarmでは期限到来処理を全て実行し再設定する。Hibernation復帰時にはstorageとsocket attachmentから復元し、メモリ内timerだけに依存しない。

## API

### WS /ws/rooms/:roomId?role=player|spectator&ticket=<ticket>
- 認証: DESIGN.md横断規約「WebSocket認証(チケット方式)」参照。`role=player`は`ticket`必須(`scope: "room"`、DOが`playerRef`と`displayName`を検証・解決する)。検証失敗(署名不正・期限切れ・scope不一致・使用済み)はHTTP 401でアップグレードを拒否する。`role=spectator`は`ticket`省略可(匿名観戦を許容する。省略時、または`ticket`を付けても`displayName`は常に固定値`"観戦者"`として扱う)
- `role=player`は最大2接続(3人目以降は**WebSocketアップグレード自体をHTTP 409で拒否**する。DESIGN.md横断規約「WebSocket接続の役割分離」参照)。加えて、同一`playerRef.id`が既に`role=player`で接続中の場合、2本目の接続はHTTP 409で拒否する(ルーム対戦での自己対戦・二重枠占有の防止)
- roomId・DO期限・タイムアウトの判定は上記「BattleRoomの状態機械」参照
- クライアント→サーバ(playerのみ):
  - `{ "type": "input", "action": "moveLeft"|"moveRight"|"rotate"|"softDrop"|"hardDrop"|"hold" }`
  - `{ "type": "board_update", "seq": number, "board": <盤面スナップショット(クライアント→サーバ形式)> }`
  - `{ "type": "piece_locked", "lockSeq": number, "count": number, "board": <消去後・おじゃま適用前>, "resumeState": <完全なエンジン状態> }`
  - `{ "type": "game_over" }`(自分の盤面が最上段まで積み上がった)
- サーバ→クライアント(player + spectator):
  - `{ "type": "joined", "role": "player"|"spectator", "side": "P1"|"P2"|null, "peers": [{ "side": "P1"|"P2", "displayName": string }] }`(`side`は`role=player`のとき自分の確定値、`spectator`はnull。`peers`は接続時点で既にいる対戦者(自分を含む、最大2件)の一覧)
  - `{ "type": "battle_start", "peers": [{ "side": "P1"|"P2", "displayName": string }] }`(2人揃った瞬間に全員へ配信)
  - `{ "type": "board_update", "seq": number, "from": "P1"|"P2", "board": <盤面スナップショット(サーバ→クライアント形式、pendingAttack込み)> }`(中継。観戦者が途中参加した場合は`joined`直後に両者の最新スナップショットを個別送信する)
  - `{ "type": "garbage_incoming", "amount": number, "target": "P1"|"P2" }`
  - `{ "type": "opponent_disconnected" }` / `{ "type": "opponent_reconnected" }`
  - `{ "type": "battle_end", "winner": "P1"|"P2"|null, "reason": "normal"|"forfeit_timeout"|"draw_timeout", "summary": { "P1": { "linesCleared": number, "linesSent": number }, "P2": { "linesCleared": number, "linesSent": number }, "durationMs": number } }`(`winner`は引き分け時null。`summary.P1`/`summary.P2`は消去/送出ラインのみを持つ。`durationMs`は両者共通の値としてトップレベルに置く。フロントは結果表示時、消去/送出ラインは`summary[自分のside]`から、対戦時間は`summary.durationMs`から読む)
  - WebSocketエラー形式: DESIGN.md横断規約「WebSocketエラー応答形式」参照(`ROOM_DISBANDED`等)
- 決着後、BattleRoomはservice binding(`env.API`、DESIGN.md「インフラ」参照)経由で`POST https://internal/api/battle-results`(内部専用)を呼びD1へ永続化する。リクエストヘッダ`X-Internal-Secret: <BATTLE_RESULTS_INTERNAL_SECRET環境変数>`で保護し、Hono側はこのヘッダを検証する(JWT/Cookieによる認証は行わない、DOはユーザーの資格情報を持たないため)。リクエストボディ: `{ "roomId": string, "players": [{ "playerRef": <両者分。登録・ゲストとも含む>, "side": "P1"|"P2", "displayName": string, "result": "win"|"lose"|"draw", "linesCleared": number, "linesSent": number }], "durationMs": number, "endedReason": "normal"|"forfeit_timeout"|"draw_timeout", "endedAt": <DOで確定したUTC時刻>, "replayManifest": <下記version1形式> }`。Hono側の挙動:
  - `players`のうち`playerRef.kind: "registered"`かつ下記「保存と退会の競合」の有効性条件を満たす要素についてのみ`battle_results`行を作成する(ゲストの要素は`battle_results`を作らないが、相手の`opponent_label`の出所として使う)。各行の`opponent_label`は、もう一方が退会中・退会済みなら「退会済みプレイヤー」、それ以外はその`displayName`を使う(相手がゲストでも`displayName`は必ず入っているため問題ない)。`opponent_id`は相手が`kind: "registered"`かつ保存時有効性条件を満たすときだけそのCognito subを設定し、ゲストならnullのままにする
  - `battle_results`行を1件も作らない場合(両者ともゲスト)は`replays`行も作成しない(参照元が無いため)
  - `battle_results`行を作成するたびに、**それぞれに1件ずつ**`replays`行を作成する(リプレイの内容は両者で同じスナップショット列。`replays.id`を複数の`battle_results`行で共有しない。理由: `replays.battle_result_id`は単一行を指すFKであり、共有すると一方の`battle_results.id`からの`GET /api/battle-results/:id/replay`が解決できなくなるため)、`created_at`は受信時刻でなく`endedAt`、`expires_at`は不変の`endedAt + 30日`で計算する。保存時点で期限を過ぎていればサマリーのみ保存する

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
- **エラー時に見えるもの**: WebSocket接続確立に失敗した場合はネットワークエラー画面。3人目以降・二重枠の`role=player`接続は409でアップグレード自体が拒否されるため、ブラウザーはupgrade失敗のHTTPステータスを直接取得できない。具体的な表示はチケット発行のRESTエラーで行い、発行後の競合・接続失敗は共通の接続失敗表示と再試行導線にする。404/409をWebSocket.onerrorから読み取れると仮定しない
- **並行操作・二重実行**: 上記settlingの100ms判定窓内に双方の`game_over`を受信した場合のみ引き分け(`winner: null`, `reason: "normal"`)として扱う
- **再表示時の整合**: `board_update`はプレイヤーごとの`seq`(単調増加)を持たせ、観戦者が途中から接続した場合は`joined`直後に現在の盤面スナップショットを個別送信して同期させる(差分ではなくフルスナップショットを都度送るため、途中参加でも整合する)

## テスト方針

- 単体: `garbageCalculator`(対応表・相殺ロジックの境界値: 1/2/3/4ライン、pendingAttackが0の場合とある場合)
- 結合: WebSocket経由での2クライアント間中継、3人目・二重枠`role=player`接続の409拒否、不正roomId/期限切れの404拒否、`waiting`状態でのタイムアウト(26文字roomIdで15秒、8文字roomIdで10分)による解散、切断検知〜猶予15秒〜タイムアウトのタイマー動作(強制切断・正常切断・猶予内再接続の3パターン。docs/design/FEASIBILITY.md PoC-1で検証したシナリオに準ずる)
- E2E(golden path): マッチング成立 → 双方が操作 → 片方がゲームオーバーになり結果画面へ遷移する一連(Playwright、2ブラウザコンテキスト)

## 着地・再開・記録の通信契約

### 着地の確定
移動は即時描画するが、自然落下を含む全着地でpiece_lockedを送信し、lock_result確定まで次のミノの進行を止める。countは0〜4の整数、lockSeqはsideごとに1から増加。別のlines_clearedイベントは設けない。
DOはlastLockSeq、直前のlock_result、完全なresumeStateを保存する。同じlockSeqの再送には保存済み応答を返して攻撃を二重加算しない。欠番・古い世代のsocketからの操作は拒否しresumeを要求する。countから消去数累計、相殺後の実送出からlinesSentを増やす。

応答は `{type:"lock_result", lockSeq, appliedRows:number, holeColumn:0..9, pendingAttack:number}`。穴の列はDOで着地ごとに一度生成し同じ応答では固定。appliedRowsが盤面高以上なら盤面が溢れるとしてgame_overを送るため、巨大な行配列を作らない。DOはこの応答までを記録してから送信する。新たな攻撃は次の着地用pendingAttackに積む。
クライアントは一度だけ適用し、`{type:"lock_applied",lockSeq,board,resumeState}`を返す。DOは未確認の該当lockSeqだけ適用後状態と確認済み境界を一度保存する。確認済みlock_appliedの再送では状態・集計・録画を変更しない。board_updateでも確認済みlockSeqを巻き戻す状態を拒否する。適用確認が欠けても再接続時に適用前状態と同じlock_resultを返すため、ブラウザー側の推測で二重適用しない。未確認のまま次のpiece_lockedは受け付けない。

### 復帰と切断
roomIdは対戦URLに保持し、再読込でも同じ部屋を対象にする。sideは予約時の割当を維持する。
`board_update`には内部保存用resumeStateも添付する。resumeStateはengineVersion、cells、current、next、hold、holdUsed、bag残り順序、乱数状態、落下・固定タイマー残量、確定lockSeqを含む。観戦配信には内部乱数状態を含めない。seqはside単位で単調増加し、DOの保存済みseq以下は破棄する。
接続時に `{type:"resume",side,state,startedAt,lastBoardSeq,lastLockSeq,acknowledgedLockSeq,board,resumeState,pendingLockResult,pendingAttack,disconnectDeadline}` を送信する。pendingLockResultがあるときは返却された適用前resumeStateから一度適用してlock_appliedを送り直す。
切断した側のシミュレーションは停止し、復帰時は最後にDOが保存した状態から再開する(未送信入力は失われる)。接続中の相手は継続し、切断側への攻撃はDOに蓄積する。

アプリheartbeatは5秒ごとにping/pong。DOで最後の受信から15秒無通信なら切断と判定し、そこから再接続猶予15秒を開始する。close/errorなら即座に猶予開始。同じ参加者の古いsocketのcloseを現接続の切断として処理しない。
最初の再接続期限に一方が接続中なら他方の敗北、両方不在ならdraw_timeoutで引き分けとし、片方だけを勝者にしない。期限ちょうどは失効扱い。settling開始済みならgame_over判定を優先する。

### リプレイ
入力の再シミュレーションではなく、DOが受理した両sideの描画スナップショットを時系列再生する。version1は `{version:1,roomId,startedAt,endedAt,events:[<以下で定義するsnapshot/endイベント>]}`。tMsはDO受信時刻-startedAt、indexは全体の通番。開始時の両盤面、受理したboard_update、lock_applied、決着イベントを記録する。inputだけから乱数や自然落下を再現しない。シークは指定時点以下の各sideの最後の完全盤面を選ぶ。通報名・guestId・Cognito sub・秘密値は記録に含めない。

長時間対戦を一つのD1行に保存しない。eventsをイベント境界で分割し、JSON配列のUTF-8表現が最大256KiBのチャンクにし、DO SQLiteに逐次保存する。終了時はチャンクを内部APIで冪等に転送後、manifestと結果を確定する。チャンク転送は `PUT /api/internal/replays/:roomId/:chunkIndex`、内部POSTと同じservice binding・X-Internal-Secretで保護し、本文 `{endedAt,hash,data}`、応答204。同じキー・hashなら再送成功、異なる内容は409。最終POST battle-resultsでは `replayManifest:{version:1,chunkCount,startedAt,endedAt}` を渡し、全チャンクの存在・個別hash確認後だけ結果を公開する。保存失敗時はDOに未送信データを残しalarm再試行する。
D1のreplay_chunksはroom_id/chunk_index複合主キー、data、hash、expires_atを持つ。replaysはroom_id、version、chunk_count、started_at、ended_atと本人用の取得権限を持つ。本人退会でreplaysを消し、同じroom_idの参照が残らなければ共有チャンクも削除する。期限切れ・未完了アップロードはendedAt+30日で削除する。保存リトライで保持期間を延長しない。

### 追加の受け入れ試験
0ライン自然着地、着地と攻撃の同時受信、piece_locked/lock_applied応答消失・重複・欠番、再読込での元side復帰、穴の再生成防止、Hibernation後の全期限復元を確認する。game_overの99/100/101ms境界、両者切断、silent切断も含める。
異なるミノ順・自然落下・おじゃま適用を含む対戦の録画とシークを照合する。256KiB境界、複数チャンク、転送途中失敗・再試行・退会と遅延保存の競合を確認する。

### 保存と退会の競合
保存時の有効な登録者はusersに存在し、account_deletionsに存在しない者。この条件は事前SELECTだけでなくD1 batch内のINSERT ... SELECTとNOT EXISTSへ含める。本人が対象外なら履歴・replays参照を作らない。相手が削除中／不在ならopponent_id=null、opponent_label="退会済みプレイヤー"。対象0名も正常成功とする。既存UNIQUE(room_id,player_id)行は再送で更新せず、退会で匿名化した名前を復元しない。
退会側は同じbatchで相手履歴のラベル匿名化、本人replays・履歴・設定・users削除を行う。保存先行なら退会が消し、退会先行なら条件付きINSERTが抑止する。

replay_uploads(room_id PK, status:staging|finalized|discarded, ended_at, expires_at)で転送状態を持つ。初回PUTのstaging確保とチャンク条件付き書き込みは同じbatch。finalizedでは既存index・同じhashの再送だけ成功、新規indexは禁止。discarded／期限切れは書き込まず410 REPLAY_DISCARDED(呼出側は再試行しない)。
最終結果batchは参照作成対象ありならfinalized、対象0名ならdiscarded化してチャンク削除。stagingは参照0件でも掃除せず、endedAt+30日の期限で削除する。退会で最後の参照が消えるroom_idだけ同じbatchでdiscarded化・チャンク削除する。discarded記録はendedAt+30日まで保持し遅延PUTによる再生成を防ぐ。期限後PUTは不変endedAtで拒否する。

### チャンクの型・確定・削除
snapshotイベントは {index,tMs,type:"snapshot",side:"P1"|"P2",board}、endイベントは {index,tMs,type:"end",winner,reason,summary}。endのwinner/reason/summaryはbattle_endと同じでside/boardは持たない。
dataはDOで一度JSON.stringifyしたイベント配列のJSON文字列。hashはそのUTF-8バイト列に対するSHA-256小文字hex。転送・保存・取得時は再シリアライズせず同じdata文字列を扱い、受信側はhash検証後にJSON.parseする。1イベントが256KiBを超える入力は拒否する。chunkIndexは0から連続、event indexは全体で連続とする。
各チャンクは不変。最終確定batchではstaging・未期限切れ・期待chunk_countと実件数および連続indexの一致を条件にreplays参照を作りfinalizedへ遷移する。期限切れ削除やdiscarded遷移と競合して欠損データを公開しない。期限切れならサマリーだけ保存する。
DO録画は最終POST成功／410で削除する。未成功でもendedAt+30日で録画を削除し、サマリー保存の再試行は別に継続する。両者ゲストならD1へ転送せず録画を終了時削除する。

### 終了結果の再取得
joinedで各参加者へ128bit以上のランダムなresultReceiptを個別送信し、DOにはhashとsideを保存する。クライアントはsessionStorageに保持しURLやログに載せない。対戦再開の権限には使えない。
GET /api/rooms/:roomId/resultはX-Room-Result-Receiptで元参加者の結果読取資格だけを検証する。finishedなら200でbattle_end内容、未決着なら202、無効／未開始解散／期限切れは404。別sideは指定不可。登録者は削除要求中・退会後に拒否する。Cookie失効後のゲストもreceiptで結果を読めるが対戦へ再接続はできない。
応答はCache-Control: no-store。DOは終了後24時間だけ結果再取得用サマリーとreceipt hashを保持してから削除する。一試合の終了通知回復用で、ゲスト履歴索引は作らない。D1保存の未完了リトライ状態とは分離する。battle_end受信前に切れたクライアントはこのAPIから結果画面を回復する。

追加試験: 初回入室期限後の猶予内復帰、lock_applied重複と巻戻し、退会と保存の前後順、転送途中の清掃、最終公開と期限切れ削除の競合、遅延PUT、結果再取得資格・24時間境界、DO録画削除を確認する。
