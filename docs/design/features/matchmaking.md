# マッチメイキング

- 種別: 機能設計書
- 対象 UC: UC-004(ステップ1〜4, E1のみ。対戦本編はfeatures/battle.md)

## 何を作るか

ランダムな対戦相手を探し、2人揃った時点で対戦部屋(BattleRoom)へ誘導する。docs/design/FEASIBILITY.md PoC-1/PoC-3で実現可能性・多人数同時アクセス耐性を検証済み。

## 入出力と振る舞い

| 操作 | 画面に起きること |
|---|---|
| 「対戦相手を探す」を選ぶ | `WS /ws/matchmake`へ接続。`queued`メッセージ受信でスピナー表示 |
| 相手が見つかる | `matched`メッセージ受信 → `POST /api/ws-tickets`(`scope: "room"`, `roomId`に受け取った値を指定)で新規チケットを取得 → 対戦画面(`/ws/rooms/:roomId?role=player&ticket=...`)へ自動遷移 |
| 30秒経過しても不成立 | 「待機継続」「キャンセル」を提示(UC-004 E1) |
| キャンセル選択 | WebSocket切断、トップ画面へ |

UI_SKETCH.html「Matchmaking」画面に対応。

## API

### WS /ws/matchmake?ticket=<ticket>
- 認証: DESIGN.md横断規約「WebSocket認証(チケット方式)」参照。`ticket`は接続前に`POST /api/ws-tickets`(`scope: "matchmake"`)で取得する。DOは`ticket`を検証して`playerRef`(サーバ解決済み、クライアントの自己申告ではない)と`displayName`を得る。検証失敗(期限切れ・署名不正・scope不一致・使用済み)は接続をHTTP 401で拒否する
- サーバ→クライアント:
  - `{ "type": "queued", "position": number }`(キュー投入直後)
  - `{ "type": "matched", "roomId": string, "opponent": string }`(マッチ成立、`roomId`はULID(26文字。features/room.mdの8文字ルームコードと区別するための形式)、`opponent`は相手の`displayName`。**`side`はここでは確定しない**(BattleRoom DOが接続順に確定する権威値。features/battle.mdの`joined`メッセージで受け取る)。以後`POST /api/ws-tickets`で`scope: "room"`の新規チケットを取得し、`/ws/rooms/:roomId?role=player&ticket=<新規チケット>`へ接続する)
- 実装上の必須事項:
  - キューへのread-modify-writeは`state.blockConcurrencyWhile()`で直列化すること(DESIGN.md「既知の制約」参照。守らないとlost updateで誰もマッチしなくなる)
  - 同一`playerRef.id`がキューに既にいる場合は自分自身とペアにしない(自己対戦の防止。同一プレイヤーの2つ目の接続は古い接続を切断してキューを差し替える)

## 実装の配置

| 処理 | 層 | 実装先ファイル |
| --- | --- | --- |
| MatchmakingQueue Durable Object本体 | DO | `src/server/matchmaking/matchmakingQueue.ts` |
| WebSocketアップグレードのルーティング | adapter | `src/server/modules/matchmaking/adapter/wsRoute.ts` |
| 待機画面・30秒タイムアウトのUI状態管理 | front | `src/front/pages/Matchmaking.tsx` |

## エッジケースの決定

- **空・最小データ**: キューが空の状態で1人だけ接続した場合は`queued`のまま待機し続ける(タイムアウトはフロント側で計測・提示する。UC-004 E1)
- **上限・境界値**: 同時待機人数に上限は設けない(PoC-3で3〜4人同時接続まで検証済み。それ以上は実運用での観測に基づき見直す)
- **エラー時に見えるもの**: WebSocket接続自体が失敗した場合はネットワークエラー画面(DESIGN.md横断規約)を表示する。チケット検証失敗時は再度`POST /api/ws-tickets`からやり直す
- **並行操作・二重実行**: 同一`playerRef.id`が2つのタブから同時に`/ws/matchmake`へ接続した場合、古い接続を切断し新しい接続のみをキューに残す(上記「実装上の必須事項」参照。自己対戦の防止と二重登録の防止を同じ仕組みで扱う)
- **再表示時の整合**: マッチ成立後にクライアントが`matched`メッセージを取りこぼした場合(通信瞬断等)、`/ws/rooms/:roomId`への接続はBattleRoom側で`role=player`かつ空き枠がある限り受け付けるため、再接続すれば復帰できる。ただしBattleRoomは最初のplayer接続から15秒以内に2人目が来ないと不成立で解散する(features/battle.md「BattleRoomの状態」参照)ため、取りこぼしたまま15秒を超えると復帰できず、フロントは再度マッチメイキングをやり直す

## テスト方針

- 単体: キューのpush/pairingロジック(`blockConcurrencyWhile`ありでの直列化を含む)
- 結合: 2クライアント同時接続でのペアリング成立、3クライアント同時接続での1ペア+1待機、既存の認証必須ルートと`/ws/matchmake`(認証不要)が共存すること
- E2E(golden path): 2つのブラウザコンテキストで「対戦相手を探す」→両者が同じ対戦画面に遷移することをPlaywrightで確認
