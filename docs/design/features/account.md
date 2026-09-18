# アカウント管理

- 種別: 機能設計書
- 対象 UC: UC-001(アカウント登録), UC-002(ログイン), UC-008(アカウント管理)。UC-004代替B/C(アクセシビリティ設定)の登録プレイヤー向け実装もここに含む

## 何を作るか

登録プレイヤーのアカウントのライフサイクル全体(新規登録・ログイン・パスワード再設定・アクセシビリティ設定・退会)を扱う。認証情報(パスワード等)はAmazon Cognitoが保持し、本アプリのD1にはニックネームや設定などアプリ固有のプロフィールのみを持つ。

## 入出力と振る舞い

### 新規登録(UC-001)
1. フロントエンドがCognito SDK(Hosted UI or Amplify等)で直接SignUpを呼び出す
2. Cognitoの既定フローに従い、確認コード入力画面を表示する(Cognito User PoolはEメール確認必須の設定とする。未確認ユーザーはログイン不可のため、この手順を省略できない)
3. 確認コード検証(ConfirmSignUp)を行う。**ConfirmSignUpはJWTを返さないため、続けてフロントエンドが同じメールアドレス・パスワードでサインイン(InitiateAuth)を行いJWTを取得する**(Amplify SDK利用時は`autoSignIn`オプションで3を自動化できる。いずれかの方式をセットアップissueで確定する)
4. 取得したJWTを使い、フロントエンドが `POST /api/account/provision` を呼び、D1に `users` 行(id=Cognitoのsub、nickname、terms_agreed_at)を作成する

| 操作 | 画面に起きること |
|---|---|
| 規約に同意せず送信 | 送信ボタン無効化 or エラー表示(UI_SKETCH.html「新規登録」画面のバリデーション参照) |
| SignUp成功 | 確認コード入力画面を表示 |
| 確認コード検証成功 | 続けてサインインしJWTを取得し、`POST /api/account/provision` を自動的に呼び、成功後チュートリアル画面(UC-011)へ遷移 |
| メールアドレス重複・確認コード誤り(Cognito側で検知) | Cognitoのエラーをそのまま表示(パスワード誤り等の存在推測につながらないエラー種別のみ) |

### ログイン(UC-002)
フロントエンドがCognitoでログインしJWTを取得、以後 `Authorization: Bearer <JWT>` を付けてAPIを呼ぶ。`next`パラメータの保持・復帰はフロントエンドのルーティング層(React Router)で行い、バックエンドAPIは関与しない。**BR-001(認証失敗理由を区別しない)に従い、`UserNotFoundException`と`NotAuthorizedException`は同一の「メールアドレスまたはパスワードが正しくありません」に丸めて表示する**(下記「エラー時に見えるもの」参照。アカウント未確認・試行回数超過など存在推測につながらない種別は個別に表示してよい)。

### アカウント設定・退会(UC-004代替B/C, UC-008)
アクセシビリティ設定の更新(UC-004代替B/C、登録プレイヤー分の保存先としてここに実装を置く)、パスワード再設定(Cognito ForgotPassword/ConfirmForgotPasswordをフロントから直接呼ぶ、UC-008)、退会(`DELETE /api/account`、UC-008)を扱う。退会は D1 の `users` 行削除に加え、**Cognito側のユーザーもAdminDeleteUserで削除する**(DESIGN.md横断規約「データ保持期間」参照。これにより同一メールアドレスでの再登録が制限なく可能になる)。

## API

DESIGN.md「API一覧」「横断規約」参照(認証方式・エラー形式は共通)。

### POST /api/account/provision
Cognito確認コード検証(ConfirmSignUp)成功後にフロントエンドが呼ぶ。認証必須(Cognitoから得た直後のJWTを使用)。

- リクエスト: `{ "nickname": string, "termsAgreed": true }`
- レスポンス(201): `{ "id": string, "nickname": string, "createdAt": string }`
- 挙動: `nickname`に共通の`nicknameFilter`(`src/server/modules/shared/domain/nicknameFilter.ts`。features/guest-session.mdと共有、NGワード判定・1〜20文字)を適用する
- エラー: `termsAgreed`がfalse/欠落 → 400 `TERMS_NOT_AGREED`。ニックネーム不正 → 400 `NICKNAME_REJECTED`/`NICKNAME_TOO_LONG`。既に`users`行が存在(二重呼び出し) → 200で既存行を返す(冪等)

### GET /api/account/settings
- レスポンス(200): `{ "colorSupportDefault": boolean, "keyBindings": { "moveLeft": string, "moveRight": string, "rotate": string, "softDrop": string, "hardDrop": string, "hold": string } }`(未設定時は既定値: `colorSupportDefault=false`、`keyBindings`はUI_SKETCH.htmlのデフォルト割当と同じ)
- features/battle.mdが対戦開始時に呼ぶほか、アカウント設定画面の初期表示にも使う

### PATCH /api/account/settings
- リクエスト: `{ "colorSupportDefault"?: boolean, "keyBindings"?: { "moveLeft": string, "moveRight": string, "rotate": string, "softDrop": string, "hardDrop": string, "hold": string } }`(部分更新)
- レスポンス(200): 更新後の設定全体
- エラー: 不正なキー値(空文字・重複割当) → 400 `INVALID_KEY_BINDING`

### DELETE /api/account
- レスポンス(204)
- 挙動: `users`行を即時削除(DESIGN.md横断規約「データ保持期間: 退会は即時削除」に従う)。関連する`battle_results`(自分が`player_id`の行)/`account_settings`/紐づく`replays`も併せて削除する。他人が`player_id`の`battle_results`行で自分が`opponent_id`の場合は`opponent_id`をnull化する(`opponent_label`は残す。DESIGN.md「データスキーマ」参照)。`reports`は`reporter_label`が自己申告の表示名のみで`users`行を参照しないため、更新不要(通報記録はそのまま残る)。**続けてCognito AdminDeleteUserを呼び、Cognito側のユーザーも削除する**
- エラー: なし(未認証は横断規約の共通401)

## 実装の配置

| 処理 | 層 | 実装先ファイル |
| --- | --- | --- |
| アカウント作成(users INSERT) | adapter/domain | `src/server/modules/account/adapter/provision.ts`, `src/server/modules/account/domain/provisionAccount.ts` |
| 設定取得・更新(account_settings SELECT/UPSERT、部分更新マージロジック) | adapter/domain | `src/server/modules/account/adapter/settings.ts`, `src/server/modules/account/domain/mergeSettings.ts` |
| 退会(users/battle_results/replays/account_settings DELETE、Cognito AdminDeleteUser呼び出し) | adapter | `src/server/modules/account/adapter/deleteAccount.ts`(D1操作とCognito管理APIの呼び出しを含むため、純粋domainには分離しない) |
| 新規登録フォーム・規約同意チェック | front | `src/front/pages/Signup.tsx` |
| ログインフォーム・next復帰 | front | `src/front/pages/Login.tsx`, `src/front/components/RequireAuth.tsx`(セットアップissueでテンプレートに実在するか確認する。無ければ新規作成) |
| アカウント設定画面・退会確認モーダル | front | `src/front/pages/AccountSettings.tsx` |

## エッジケースの決定

- **空・最小データ**: ニックネーム未入力はCognito SignUp前のフロント側バリデーションで送信不可にする
- **上限・境界値**: パスワード強度はCognito User Poolのデフォルトパスワードポリシー(最低8文字、大小英字+数字を含む)をそのまま採用し、アプリ側では独自検証しない
- **エラー時に見えるもの**: Cognitoからのエラーは日本語ラベルにマッピングして表示する。少なくとも次の4種は個別に定義すること: `UsernameExistsException`(メール重複)、`InvalidPasswordException`(パスワード強度不足)、`CodeMismatchException`(確認コード誤り)、ログイン時の`UserNotFoundException`/`NotAuthorizedException`(BR-001に従い単一メッセージに統合)。マッピング表本体は実装時にfeatures外の定数ファイルに定義してよい
- **並行操作・二重実行**: `POST /api/account/provision`はネットワーク再送等での二重呼び出しに対して冪等(既存行があれば200で返す)
- **再表示時の整合**: 退会時にCognito側のユーザーもAdminDeleteUserで削除するため、既存のアクセストークンはCognito側の失効処理により早期に無効化される想定だが、トークンの有効期限内は理論上API疎通してしまう可能性がある。その場合も`users`行が存在しないため、`authenticate`ミドルウェア通過後の各ハンドラは`users`行の有無を確認し、無ければ404 `USER_NOT_FOUND`を返す(この挙動をテンプレート既存の`GET /api/me`にも適用する。セットアップissueで現状の挙動を確認し、必要なら修正する)。フロントは404 `USER_NOT_FOUND`を検知して強制ログアウトしトップへ遷移する

## テスト方針

- 単体: `provisionAccount`(冪等性)、`deleteAccount`(関連テーブルの削除、opponent_idのnull化)、`account_settings`の部分更新マージロジック
- 結合: `PATCH /api/account/settings`の認証ガード(未認証401)、`DELETE /api/account`後に`GET /api/me`的な情報取得が失敗すること
- E2E(golden path): 新規登録 → チュートリアル表示 → ログアウト → ログイン → アカウント設定でキー割当変更 → 退会確認モーダルで確定 → トップに戻る、の一連
