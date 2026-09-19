# ルーム対戦
<!-- 変更履歴 [2026-09-18]: DOによる存在・期限・予約席管理とキャンセルを統一。 -->

- 種別: 機能設計書
- 対象 UC: UC-005(ルームを作成して友人と対戦する)

## 何を作るか
ルームコードを共有して2人で対戦する。対戦本編はbattle.mdを使う。UI_SKETCH.html「Room」に対応する。

## 入出力と振る舞い
| 操作 | 画面に起きること |
|---|---|
| ルーム作成 | 有効な登録／ゲストセッションでPOST /api/rooms。P1予約済みroomIdを受け取り、roomチケットを取得して接続。コードとコピー導線を表示 |
| IDを共有 | アプリ外で共有 |
| 相手がコード入力 | roomチケットを取得し接続。P2を原子的に確保して2人が揃うとbattle_start |
| 満室・存在しない・期限切れ | チケット発行RESTの409/404で案内する。発行後に競合してWS接続が失敗した場合は共通の接続失敗と再試行導線 |
| 作成者がキャンセル | DELETE /api/rooms/:roomId。waitingなら即座に解散しトップへ |

## API
### POST /api/rooms
- 認証: 登録JWTまたは署名付きゲストCookie必須。guest-session.mdと同じ検証。登録不要だが身元未確定のリクエストは401。
- リクエスト: `{displayName?:string}`。ゲストは必須、共通nicknameFilterを適用。登録者はusers.nicknameを使う。
- レスポンス201: `{roomId:string,expiresAt:string}`。
- IDはCSPRNGによるCrockford Base32の8文字。対象DOの内部initialize RPCで未作成を原子的に確認し、衝突なら再生成する。KVによる重複判定は使わない。
- initializeはroomId、source:private、createdAt、expiresAt=createdAt+10分、ownerId、P1のplayerRefを永続化する。成功するまで201を返さない。外部からinitializeを呼べるルートは設けない。
- 不正名400、保存失敗503。連打はUIで抑止するが、別リクエストは別roomIdとなる。

### DELETE /api/rooms/:roomId
- 認証: 作成者本人のJWTまたは有効な署名付きゲストCookie。Cookie経路はOrigin検証。
- waitingをfinishedへ原子的に遷移して204。同じ所有者の再送も204。進行中は409 ROOM_ALREADY_STARTED。他人／未作成は404。
- キャンセルと2人目入室が競合した場合、DOで先に確定した状態を採用する。

### 入室
チケット発行時にDOの存在・期限・参加資格を確認し、接続時にも再確認する。privateはP1予約を奪わず、P2は最初の異なるplayerRefに確定する。コードを知る第三者がP2になる可能性は共有コード方式の制約として残る。randomの参加者予約はmatchmaking.mdに従う。形式が正しくても未初期化DOは入室不可。観戦者は席を消費せず、privateのwaitingは観戦不可。

## 実装の配置
| 処理 | 層 | 実装先 |
|---|---|---|
| 作成・解散ルート | adapter | src/server/modules/room/adapter/createRoom.ts, deleteRoom.ts |
| 期限・席・状態 | DO | src/server/battle/battleRoom.ts |
| コード表示・コピー・待機 | front | src/front/pages/Room.tsx |

## エッジケースの決定
- 期限は最初の接続時刻で延長しない。期限と同時刻は失効。開始後は入室期限による解散をしない。
- waiting中の作成者切断は即座に解散。接続前のキャンセルもDELETEで解散する。ブラウザー強制終了等で通知できなければ期限で解散する。
- roomIdはURLに保持し、再読込はDOの状態を確認する。waitingで切断によって解散済みなら再作成を案内し、進行中ならbattle.mdの再接続猶予を適用する。
- KVは進行中一覧の候補索引のみ。privateのwaitingは掲載せず、開始時に掲載する。終了フラグ確定後に削除し、closeで再掲載しない。

## テスト方針
- 単体: ID形式・衝突再生成・期限境界。
- 結合: 初期化失敗時に201を返さないこと、未初期化ULID/8文字IDの拒否、P1予約維持、P2同時入室競合、キャンセルと開始の競合、開始前離脱・期限解散。
- E2E: 作成→コード共有→別ブラウザーが入室→2人で開始。満室表示、キャンセル済みコード、再読込も検証。

入室expiresAtはwaitingの初回入室だけに適用する。進行中の本人復帰ではチケット発行時・接続時とも、固定席・接続世代・disconnectDeadlineで判定する。
