# マッチメイキング

- 種別: 機能設計書
- 対象 UC: UC-004(ステップ1〜4, E1のみ。対戦本編はfeatures/battle.md)

## 何を作るか

ランダムな対戦相手を探し、2人揃った時点で対戦部屋(BattleRoom)へ誘導する。docs/design/FEASIBILITY.md PoC-1/PoC-3で限定的なローカル同時接続を検証済み。本番の収容能力は未検証。

## 入出力と振る舞い

| 操作 | 画面に起きること |
|---|---|
| 「対戦相手を探す」を選ぶ | `WS /ws/matchmake`へ接続。`queued`メッセージ受信でスピナー表示 |
| 相手が見つかる | `matched`メッセージ受信 → `POST /api/ws-tickets`(`scope: "room"`, `roomId`に受け取った値を指定)で新規チケットを取得 → 対戦画面(`/ws/rooms/:roomId?role=player&ticket=...`)へ自動遷移 |
| 30秒経過しても不成立 | 「待機継続」「キャンセル」を提示(UC-004 E1) |
| キャンセル選択 | `{type:"cancel"}`送信、取消応答後に切断してトップへ |

UI_SKETCH.html「Matchmaking」画面に対応。

## API

### WS /ws/matchmake?ticket=<ticket>
- 認証: DESIGN.md横断規約「WebSocket認証(チケット方式)」参照。`ticket`は接続前に`POST /api/ws-tickets`(`scope: "matchmake"`)で取得する。DOは`ticket`を検証して`playerRef`(サーバ解決済み、クライアントの自己申告ではない)と`displayName`を得る。検証失敗(期限切れ・署名不正・scope不一致・使用済み)は接続をHTTP 401で拒否する
- クライアント→サーバ:
  - `{ "type": "cancel" }`(待機の取消。下記「予約・取消の直列化」の通り、予約後の取消は明示的な`cancel`だけで行う)
  - `{ "type": "ping" }`(アプリheartbeat。5秒ごと。サーバは`pong`を返す。最終受信から15秒(`heartbeatTimeout`)無通信の接続はペア候補から除く)
- サーバ→クライアント:
  - `{ "type": "queued", "position": number }`(キュー投入直後)
  - `{ "type": "matched", "roomId": string, "opponent": string }`(マッチ成立、`roomId`はULID(26文字。features/room.mdの8文字ルームコードと区別するための形式)、`opponent`は相手の`displayName`。**`side`はペア順のP1/P2をBattleRoom初期化時に予約する**(クライアントはbattle.mdのjoinedで自分の確定値を受け取る)。以後`POST /api/ws-tickets`で`scope: "room"`の新規チケットを取得し、`/ws/rooms/:roomId?role=player&ticket=<新規チケット>`へ接続する)
  - `{ "type": "cancelled" }`(取消の確定応答。クライアントはこれを受けてから切断しトップへ戻る)
  - `{ "type": "pong" }`(アプリheartbeatの応答)
  - `{ "type": "error", "code": "ROOM_DISBANDED", "message": <日本語メッセージ> }`(予約済みの部屋が`enterDeadline`超過・相手の取消などで解散した場合。DESIGN.md横断規約「WebSocketエラー応答形式」準拠。クライアントは新しいキューへの参加を選べる)
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
- **再表示時の整合**: Queue DOはplayerRefごとに未期限切れの割当(roomId、side、opponent、expiresAt)を永続化する。matchedを取りこぼしたクライアントは新しいmatchmakeチケットでQueueへ再接続し、同じmatchedを再受信する。既存割当がある間は新規ペアに入れない。BattleRoomの入室期限(`enterDeadline`)は初期化から15秒(DESIGN.md「承認済みの運用方針」「期限・猶予の用語」)。期限後はROOM_DISBANDEDを示し、新規キューへの参加を選べる。


## テスト方針

- 単体: キューのpush/pairingロジック(`blockConcurrencyWhile`ありでの直列化を含む)
- 結合: 2クライアント同時接続でのペアリング成立、3クライアント同時接続での1ペア+1待機、既存の認証必須ルートと`/ws/matchmake`(認証不要)が共存すること
- E2E(golden path): 2つのブラウザコンテキストで「対戦相手を探す」→両者が同じ対戦画面に遷移することをPlaywrightで確認

## 予約・取消の直列化
キュー項目にはplayerRefと接続ごとのconnectionIdを持つ。cancel/close/errorは現在項目とconnectionIdが一致するときだけ除去する。置換済みの古いsocketの遅延closeは新項目を消さない。heartbeatの5秒ping/pong・15秒無受信(`heartbeatTimeout`)をキューにも適用し、死んだ接続をペア候補から除く。

短いstorage更新だけをblockConcurrencyWhileで保護する。外部RPCを待ちながらロックを保持しない。2名を選んだら同一トランザクションでqueueからreservedへ移し、roomIdと割当準備レコードを保存する。その後BattleRoom.initializeを呼ぶ。initializeは同じroomId・参加者なら冪等。成功後にreadyを保存してmatchedを送る。応答消失時は同じroomIdで再試行し、別の対戦を作らない。再起動時は準備レコードを回復する。
予約後のキャンセルはキューから勝手に戻さず、BattleRoomへ同じplayerRefによるwaiting解散を要求する。開始済みなら通常の切断規則へ移る。解散確定時だけ割当を終端化し、相手へROOM_DISBANDEDを通知する。15秒期限を過ぎた割当の掃除もroom状態を確認し、既にin_progressなら保持する。結果終了はBattleRoomからQueueへ冪等通知し、通知失敗は再試行する。Queue再接続時にもroomを照会して復旧する。

追加試験: matched応答消失、初期化RPC応答消失、Hibernation復帰、古いclose、予約中cancel、開始済み割当の再キュー防止、同一人物の二重タブを検証する。

予約前はclose/errorで項目を除去するが、reserved/ready以降はQueue socketのcloseだけで予約を取り消さない。matched後の正常な画面遷移ではcloseしてよい。取消は明示的cancelのみ。
cancelは準備レコードへcancelRequestedを保存する。initialize前の解散RPC失敗を取消完了としない。initialize成功後に再確認し、取消ならBattleRoom解散確定後にcancelled/ROOM_DISBANDEDを通知してmatchedを送らない。readyかつcancelRequested=falseの確認は直列化する。送信とcancelが競合してmatchedが届いた場合もDOの解散確定を優先する。初期化前cancel、部屋への通常移行close、通知消失後の復旧を試験する。
