# ゲストセッション

- 種別: 機能設計書
- 対象 UC: UC-003(ゲストとして対戦を始める)

## 何を作るか

登録せずにニックネームだけで対戦を始められるゲストプレイヤーの識別機構。認証不要ルートとして実装し、Cognitoには一切依存しない(docs/design/FEASIBILITY.md PoC-2で検証済みの「ルート単位オプトイン」方式)。

## 入出力と振る舞い

| 操作 | 画面に起きること |
|---|---|
| トップで「ゲストで遊ぶ」→ニックネーム入力→「この名前で始める」 | ①`POST /api/guest/session`で`guest_id` Cookie発行(初回のみ) ②`POST /api/ws-tickets`にニックネームを渡しチケット取得 ③`/ws/matchmake?ticket=...`へ接続してマッチメイキング画面(UC-004)へ遷移 |
| ニックネーム未入力のまま送信 | 送信不可・エラー表示(UI_SKETCH.html Home画面参照) |
| 不適切な語を含むニックネーム | `POST /api/ws-tickets`が拒否(UC-003 E2の判定はニックネーム送信時=チケット発行時に行う)、登録拒否・再入力を促す |

UI_SKETCH.html「Home」画面のゲスト入力モーダルに対応。ニックネームのNGワード判定はDESIGN.md「横断規約」の`POST /api/ws-tickets`で行う(本機能はゲストの識別Cookie発行のみを担当する)。

入力されたゲスト名は**sessionStorage(キー`vstetris.guestName`)**にだけ保持する。localStorageに置かないのはUC-003 BR-002(ニックネームは次回訪問時に引き継がれない)のため。チュートリアル(features/tutorial.md)・マッチメイキング(features/matchmaking.md)はこの値を読むだけで、`POST /api/ws-tickets`へ渡す`displayName`の出所とする。

## API

DESIGN.md「API一覧」参照。

### POST /api/guest/session
- 認証: 不要
- リクエスト: なし
- 挙動: 既存の`guest_id` Cookieの署名・用途・有効期限をサーバーで検証できた場合のみそれを再利用し`issued: false`、無い／不正／期限切れなら新規発行して`issued: true`。**Cookieの有無に関わらず、再訪問のたびにニックネームは`POST /api/ws-tickets`で毎回指定し直す**(UC-003 BR-002: ニックネームは引き継がれない)
- レスポンス(200): `{ "guestId": string, "issued": boolean, "expiresAt": string }`
- Cookie: `guest_id=<署名付きトークン>; Path=/api; HttpOnly; SameSite=Lax; Max-Age=86400`(DESIGN.md横断規約: 有効期限24時間。`Secure`属性は本番のみ付与し、開発環境では環境変数`COOKIE_SECURE=false`で無効化する)
- 署名対象: `{ version: 1, purpose: "guest-session", guestId: <ULID>, issuedAt, expiresAt }`。専用secretでHMAC署名し、発行時刻＋24時間を固定期限とする。再利用時は期限を延長しない。`POST /api/ws-tickets`でも署名・用途・期限を検証し、不正／期限切れは401 `GUEST_SESSION_EXPIRED`。Cookie中の生ULIDを資格情報として受け入れない。`playerRef.id`には検証後のguestIdのみを使う。
- エラー: 発行処理失敗は503。Cookieを新規発行できたようには表示しない。Cookie認証の変更系APIは同一Originを検証する。
- 期限と再接続: 期限は画面に表示し、残り60秒以下なら開始前に「期限後は再接続できません」と表示する。既に接続した対戦は期限後も継続するが、新規チケットは取得不可。期限後の切断では同じIDの復帰を許可せず、通常の切断猶予15秒で決着する。再発行したIDで既存席を引き継ぐことはできない。

## 実装の配置

| 処理 | 層 | 実装先ファイル |
| --- | --- | --- |
| ゲストIDの発行・Cookie設定(認証ミドルウェアなし) | adapter | `src/server/modules/guest/adapter/guestSession.ts` |
| NGワード判定(account.mdの新規登録、`POST /api/ws-tickets`からも共通利用するため`shared`配下に置く) | domain | `src/server/modules/shared/domain/nicknameFilter.ts` |
| ゲスト名の読み書き(sessionStorage、キー`vstetris.guestName`。**書き込みはこのHome側が正本**、features/tutorial.md・features/matchmaking.mdは読み取りのみ) | front | `src/front/lib/guestName.ts` |
| ゲスト入力モーダル | front | `src/front/pages/Home.tsx` |

## エッジケースの決定

- **空・最小データ**: ニックネーム空文字・空白のみは`POST /api/ws-tickets`側で400(DESIGN.md参照)
- **上限・境界値**: `nicknameFilter`はニックネーム1〜20文字(全角/半角問わず文字数カウント)を許容範囲とする。超過は400 `NICKNAME_TOO_LONG`
- **エラー時に見えるもの**: NGワード判定に該当した場合、具体的にどの語が引っかかったかは返さず「ニックネームを変更してください」とだけ表示する(判定ロジックの推測を防ぐ)
- **並行操作・二重実行**: 同一ブラウザで複数タブから同時に`POST /api/guest/session`を呼んでも、Cookieが未設定の間は複数回発行されうる(最後に設定されたCookieが有効になる)。ゲスト機能はセッション永続性が要件でないため許容する
- **再表示時の整合**: ゲストCookieを保持したままログイン(UC-002)した場合、以後は登録プレイヤーとして扱い`guest_id`は無視する(削除はしない、単に参照しなくなる)。対戦継続中の`playerRef`の扱いはDESIGN.md横断規約「ゲストの識別」参照

## テスト方針

- 単体: `nicknameFilter`(NGワード判定、文字数境界値)
- 結合: `POST /api/guest/session`が認証ミドルウェアを経由しないこと(既存の認証必須ルートと共存すること)、有効な既存Cookieを再発行・延長しないこと、署名改変・期限境界の拒否、Pathによりws-ticketsへCookieが届くこと、期限直前開始後の再接続成功／期限後拒否
- E2E(golden path): トップ→「ゲストで遊ぶ」→ニックネーム入力→マッチメイキング画面への遷移

ゲスト用WSチケットのexpiresAtはmin(発行時刻＋60秒, guest.expiresAt)。DOでも同じ期限を検証し、期限と同時刻は拒否する。期限直前の発行でも失効後の復帰はできない。
