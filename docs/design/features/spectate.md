# 観戦
<!-- 変更履歴 [2026-09-18]: KVの結果整合性とローカルPoCの限界を明記。 -->

- 種別: 機能設計書
- 対象 UC: UC-006(対戦を観戦する)

## 何を作るか

進行中の対戦一覧を表示し、第三者視点で盤面をリアルタイム観戦、簡易リアクションを送れる機能。docs/design/FEASIBILITY.md PoC-5で対戦者2+観戦者8までのブロードキャストとローカル一覧レジストリの整合を検証済み。本番の即時整合性や無制限の収容能力は未検証。**観戦一覧はランダムマッチング(UC-004)・ルーム対戦(UC-005)を区別せず、進行中の全対戦を表示する**(UC-006 BR-002で確定)。

## 入出力と振る舞い

| 操作 | 画面に起きること |
|---|---|
| 「観戦する」を選ぶ | `GET /api/rooms/active`で進行中の対戦一覧を取得(ランダムマッチ・ルーム対戦を区別しない。ただし公開タイミングは経路により異なる。下記API節参照) |
| 対戦が1件もない | 「観戦できる対戦はありません」を表示 |
| 対戦を1つ選ぶ | `POST /api/ws-tickets`(`scope: "room"`, `role: "spectator"`, `roomId`を指定。匿名ならチケット取得自体を省略可)でチケット取得後、`/ws/rooms/:roomId?role=spectator&ticket=...`へ接続、両者の盤面をリアルタイム表示 |
| リアクション送信 | 観戦者・対戦者を含む全接続者に一時表示。5秒間に5回まで(スパム防止、下記確定値) |
| 制限超過でリアクション拒否 | トーストで「少し待ってから送ってください」を表示 |
| 「観戦をやめる」を選ぶ(対戦終了前) | WebSocket切断、観戦一覧へ戻る(観戦者数はKV上で自動的にデクリメントされる) |
| 対戦終了 | `battle_end`受信 → 結果表示 → 観戦一覧に戻れる |
| 観戦中に対戦が異常終了(通信断等) | 「対戦が中断されました」を表示してから一覧へ戻す |
| 「通報する」を選ぶ | ReportModalを開き、`reporterLabel`(自分がチケット取得済みならクライアントが保持する自分の`displayName`、匿名観戦なら固定値`"観戦者"`。いずれもサーバ側の観戦者識別とは独立したクライアント側の自己申告値)・`targetLabel`・`roomId`を渡して送信する(features/report.md参照) |

UI_SKETCH.html「SpectateList」「Spectate」画面に対応。

### リアクション送信制限(BR-001の確定値)

**5秒間に5回まで**。DO側でクライアントごとの直近タイムスタンプ配列を保持し、ウィンドウ超過分を拒否する(UI_SKETCH.htmlのプロトタイプで採用した値をそのまま正式採用)。

## API

### GET /api/rooms/active
- 認証: 不要
- レスポンス(200): `{ "matches": [{ "roomId": string, "status": "waiting"|"in_progress", "players": string[], "spectatorCount": number, "startedAt": string|null, "createdAt": string }] }`。`players`は接続中プレイヤーの`displayName`配列(内部ID・`playerRef.id`は公開しない)。`startedAt`は2名が揃った対戦開始時刻(ISO8601)、waitingではnull。createdAtはDO初期化時刻
- `status`の意味: `waiting`は初期化済み・未開始(0〜1名接続)(features/battle.md「BattleRoomの状態機械」参照)、`in_progress`は2名揃い対戦中
- **公開範囲**: DOに記録したsource=randomは`waiting`/`in_progress`とも一覧に含める。source=privateは**`in_progress`になるまで一覧に含めない**。この除外は**書き側(DOのKV更新)で行う**ため、privateの`waiting`はそもそもKVに存在せず、読み側(`activeRooms.ts`)には除外ロジックを置かない(正本はDESIGN.md横断規約「Workers KVのキー空間」。ルームIDを知る者だけが入室できる前提を、一覧での早期露出によって崩さないため)
- データソース: DESIGN.md横断規約「Workers KVのキー空間」で定義した`match:<roomId>`の**metadata JSON**(`{status, players, spectatorCount, startedAt, createdAt, source}`)。`list({ prefix: "match:" })`の**metadataだけ**でレスポンスを構成し、個別キーの`get`は行わない(KVの`list()`は値本体を返さないため)。`source`はmetadataには含まれるがレスポンスには出さない
- **返却上限**: 100件。超過分は`createdAt`の新しい順に残して切り捨てる(正本はDESIGN.md横断規約「Workers KVのキー空間」)
- **`waiting`行の表示規則**: `startedAt`がnullのため経過時間は表示せず、代わりに「開始待ち」と表示する。`players`が0件の場合はプレイヤー名欄に「対戦者を待っています」と表示する

### WS /ws/rooms/:roomId?role=spectator の追加メッセージ(features/battle.mdの共通部分を除く)
- クライアント→サーバ: `{ "type": "reaction", "emoji": string }`(**観戦者(`role=spectator`)のみ送信可能**。features/battle.md「WebSocketメッセージ一覧(正本)」にも参照として1行載せてあり、対戦者(`role=player`)から送られた場合は無視する)
- サーバ→クライアント: `{ "type": "reaction", "emoji": string, "from": "spectator" }`(対戦者・観戦者を含む全接続者へブロードキャストする。送信元は観戦者のみのため`from`は固定値。対戦のプレイ妨害を避けるため、対戦者クライアントのUIはリアクションを画面端に小さく表示するに留める)、制限超過時は送信者のみに`{ "type": "error", "code": "REACTION_RATE_LIMITED", "message": "少し待ってから送ってください" }`(DESIGN.md横断規約「WebSocketエラー応答形式」準拠)

## 実装の配置

| 処理 | 層 | 実装先ファイル |
| --- | --- | --- |
| 進行中対戦一覧の取得(KVの`list({prefix:"match:"})`とmetadataの整形、100件への切り詰め。**privateの`waiting`除外は書き側で済んでいるため読み側では行わない**) | adapter | `src/server/modules/spectate/adapter/activeRooms.ts` |
| KVレジストリ(`match:<roomId>`)の読み書き関数(Honoルートではなく純粋なKVアクセス関数。metadataの組み立て・TTL・集約更新・**privateの`waiting`を書かない判定**もここ) | adapter | `src/server/modules/spectate/adapter/matchRegistry.ts`(features/battle.mdのBattleRoom DOから呼ばれる) |
| リアクションのレート制限・ブロードキャスト | DO | `src/server/battle/battleRoom.ts`(features/battle.mdと共通のDOに追加) |
| 通報導線(「通報する」→ReportModal) | front | `src/front/pages/Spectate.tsx`(features/report.md参照) |
| 観戦一覧・観戦画面・リアクションUI・「観戦をやめる」導線 | front | `src/front/pages/SpectateList.tsx`, `src/front/pages/Spectate.tsx` |

## エッジケースの決定

- **空・最小データ**: 進行中の対戦が0件の場合の表示はUC-006代替フローA(空状態表示)の通り
- **上限・境界値**: 観戦者数に上限は設けない(PoC-5で8名まで動作確認済み。上限が必要になった場合は`role=spectator`接続時に人数チェックを追加する形で拡張できる設計にする)
- **エラー時に見えるもの**: 観戦中に対戦が異常終了(通信断等)した場合、`battle_end`が届かないまま接続が切れることがある。この場合フロントは「対戦が中断されました」を表示してから一覧に戻す(UC-006 E2)
- **並行操作・二重実行**: 同一観戦者が同じルームに複数タブで接続することを禁止しない(複数タブを許容する。ただし収容能力とリソース上限は別途負荷検証する)
- **再表示時の整合**: 観戦一覧は`GET /api/rooms/active`の**5秒間隔**ポーリングで更新する(既定値)。リアルタイムプッシュ配信は一覧自体には行わない(個々の観戦画面内の盤面のみWebSocketでリアルタイム)

## テスト方針

- 単体: リアクションのレート制限ロジック(5秒5回の境界値)
- 結合: `GET /api/rooms/active`がKVレジストリと実際のBattleRoom接続状態(観戦者増減・対戦終了)に追従すること、ランダムマッチ・ルーム対戦の両方が一覧に含まれること
- E2E(golden path): 対戦中のルームに観戦者として接続 → 両者の盤面が更新されること → リアクション送信 → 対戦終了で結果表示に切り替わることをPlaywrightで確認

## 一覧の更新遅延

KVの結果整合性により新しい対戦がすぐ表示されない、終了済み対戦が残る場合がある。5秒ポーリングは反映期限の保証ではない。接続先DOで状態を再確認し、終了済みなら案内を出して一覧へ戻す。観戦者数も概数として扱う。キー更新の集約・再試行・TTLはDESIGN.md「KVレジストリの整合性」に従う。

結合試験にはKVが古い状態／キー欠落／更新失敗を返すケース、終了後のcloseイベント、再起動後のfinished復元を含める。

接続前の一覧は候補のみであり、private waiting／未初期化／終了済みのDOは観戦不可。チケットを取得する場合はrole:spectatorを明記する。匿名接続のupgrade失敗ではHTTP状態をブラウザーから取得できないため「接続できません。対戦が終了した可能性があります」と表示し一覧へ戻す。観戦者への通報用表示名は従来どおり固定値「観戦者」。録画はbattle.mdの盤面スナップショット契約に従う。
