# アカウント管理
<!-- 変更履歴 [2026-09-18]: 猶予なしの削除開始、失敗復旧、旧JWT拒否を具体化。 -->

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
| 確認コード検証成功 | 続けてサインインしJWTを取得し、`POST /api/account/provision` を自動的に呼び、成功後、未読ならチュートリアル画面(UC-011)へ遷移、既読ならトップへ遷移 |
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
- **MVPスコープの注記**: US-025はCould(あると嬉しい)のため、**MVPではこのAPIとゲスト向けlocalStorage(`vstetris.guestAccessibility`、features/battle.md)による保存機構と、対戦開始時の読み込み・適用までを実装する。キー割当を変更する設定UIは後続フェーズ**とする(UC-004代替C参照)。色覚サポートの切替は対戦画面のトグル(features/battle.md)としてMVPに含める
- リクエスト: `{ "colorSupportDefault"?: boolean, "keyBindings"?: { "moveLeft": string, "moveRight": string, "rotate": string, "softDrop": string, "hardDrop": string, "hold": string } }`(部分更新)
- レスポンス(200): 更新後の設定全体
- エラー: 不正なキー値(空文字・重複割当) → 400 `INVALID_KEY_BINDING`

### DELETE /api/account
- 認証: 登録プレイヤー本人。確認モーダルで確定後に実行。
- レスポンス: 両ストアで削除完了なら204。削除要求を永続化できたが処理中なら202 `{ "status": "deleting" }`。要求記録自体が失敗した場合503 `ACCOUNT_DELETE_UNAVAILABLE`。
- 猶予期間は設けない。DESIGN.md「保存の制約とトランザクション境界」に従って削除要求(`account_deletions`行)を先に記録し、通常API・プロフィール再作成・新しいチケット発行を即時拒否する。
- Cognito削除、D1の本人履歴・リプレイ・設定・usersの削除を`account_deletions.phase`に段階記録する。失敗時はCronで再試行し(DESIGN.md「インフラ」の定期実行②、`*/15 * * * *`)、Cognitoの既削除は成功扱いとする。D1の削除はbatchで行う。

  | `phase` | 意味 | 次にCron②が行うこと |
  |---|---|---|
  | `requested` | 削除要求を記録した直後。通常API・provision・チケット発行は既に403 `ACCOUNT_DELETING`で拒否している | CognitoのAdminDeleteUserを呼ぶ。成功、または`UserNotFoundException`(既削除)なら`cognito_deleted`へ |
  | `cognito_deleted` | Cognito側の削除が完了 | 下記の固定順D1 batchを実行する。成功なら`d1_deleted`へ |
  | `d1_deleted` | D1側の削除が完了 | `phase='completed'`、`retry_at = 完了時刻 + 60分`を書く |
  | `completed` | 両ストアの削除が完了。旧JWTの失効判定のためだけに行が残っている | `retry_at`が到来したら`account_deletions`の行自体を削除する |

  `retry_at`は失敗時に指数バックオフで先送りする。`last_error_code`に直近の失敗理由を残す。`completed`の`retry_at`に足す**60分はCognito app clientのアクセストークン有効期限**であり、セットアップissueでTerraform/IaCに60分固定として明示する(DESIGN.md「セットアップissue完了後に確定する手順」)。この期間は、署名だけ有効な旧JWTが提示されても`account_deletions`行の存在により拒否できる。
- **D1 batchは固定順・固定文数**とする。順序の根拠は2つある。(a)`battle_results.opponent_id`はON DELETE SET NULLのため、`users`を先に削除すると相手行の`opponent_id`がnull化され**匿名化すべき行を特定できなくなる**。(b)共有チャンクの削除対象`room_id`は本人の`replays`行からしか辿れないため、**`replays`を削除する前**に②-a/②-bを実行しなければ`room_id`が特定不能になる。

  以下のSQLの`?name`は可読性のための表記で、実装では**出現順の位置指定`?`に展開**し同じ値も都度バインドする(SQLiteの名前付きパラメータは`:name`/`@name`/`$name`。`?name`は`WHERE`等では構文エラーだが、SELECT句内では`? AS name`として黙って受理されるため、展開を省略しない。features/battle.md「確定batchの固定SQL文列」の記法注記と同じ)。

  | # | 文 | 内容 |
  |---|---|---|
  | ① | `UPDATE battle_results SET opponent_label = '退会済みプレイヤー', opponent_id = NULL WHERE opponent_id = ?userId` | 相手の履歴に残る本人の表示名を匿名化する。**必ず最初に実行する** |
  | ②-a | 下記 | 本人の`replays`が指す`room_id`のうち、**他の`player_id`の`replays`参照が無い**ものの`replay_chunks`を削除する。**`replays`削除より前** |
  | ②-b | 下記 | 同じ条件の`replay_uploads`を`discarded`にする(遅延PUTによる再生成を防ぐ)。**`replays`削除より前** |
  | ②-c | `DELETE FROM replays WHERE battle_result_id IN (SELECT id FROM battle_results WHERE player_id = ?userId)` / `DELETE FROM battle_results WHERE player_id = ?userId` / `DELETE FROM account_settings WHERE user_id = ?userId` | 本人のリプレイ参照・履歴・設定を削除する(FKのCASCADEに依存せず明示的に削除し、削除順を決定的にする) |
  | ③ | `DELETE FROM users WHERE id = ?userId` | 本人行を削除する。`account_deletions`はFKを持たないため残り、再試行と旧JWT失効判定に使う |

  ②-a(共有チャンクの削除):

  ```sql
  DELETE FROM replay_chunks
  WHERE room_id IN (
    SELECT r.room_id FROM replays r
    JOIN battle_results br ON br.id = r.battle_result_id
    WHERE br.player_id = ?userId
      AND NOT EXISTS (
        SELECT 1 FROM replays r2
        JOIN battle_results br2 ON br2.id = r2.battle_result_id
        WHERE r2.room_id = r.room_id AND br2.player_id <> ?userId
      )
  );
  ```

  ②-b(`replay_uploads`のdiscarded化。②-aと同じ副問い合わせを使う):

  ```sql
  UPDATE replay_uploads SET status = 'discarded'
  WHERE room_id IN (
    SELECT r.room_id FROM replays r
    JOIN battle_results br ON br.id = r.battle_result_id
    WHERE br.player_id = ?userId
      AND NOT EXISTS (
        SELECT 1 FROM replays r2
        JOIN battle_results br2 ON br2.id = r2.battle_result_id
        WHERE r2.room_id = r.room_id AND br2.player_id <> ?userId
      )
  );
  ```

  相手も登録者でその`replays`行が残っている`room_id`は`NOT EXISTS`で除外されるため、チャンクは削除されず相手のリプレイは再生できる。未完了の`staging`を「参照0件」という理由で全件掃除することはしない(期限での削除はCron①に任せる)。
- 他人の履歴内の本人参照はnullにし、opponent_labelは「退会済みプレイヤー」に置き換える(①)。通報(`reports`)は現行スキーマでは自己申告名のみのため確実な本人照合ができず、**MVPでは退会時にも削除しない**(DESIGN.md横断規約「データ保持期間」)。この点はDESIGN.mdの未解決論点として公開前に保持方針を確定する。
- 202の場合は「退会を受け付けました。削除処理中です」と表示してローカルの認証情報を破棄する。失敗を削除完了とは表示しない。重複要求で削除ジョブを増やさない。

## 実装の配置

| 処理 | 層 | 実装先ファイル |
| --- | --- | --- |
| アカウント作成(users INSERT) | adapter/domain | `src/server/modules/account/adapter/provision.ts`, `src/server/modules/account/domain/provisionAccount.ts` |
| 設定取得・更新(account_settings SELECT/UPSERT、部分更新マージロジック) | adapter/domain | `src/server/modules/account/adapter/settings.ts`, `src/server/modules/account/domain/mergeSettings.ts` |
| 退会(account_deletions記録、相手履歴の匿名化→本人replays/battle_results/account_settings→users DELETEの固定順batch、Cognito AdminDeleteUser呼び出し) | adapter | `src/server/modules/account/adapter/deleteAccount.ts`(D1操作とCognito管理APIの呼び出しを含むため、純粋domainには分離しない) |
| 退会の再試行(Cron Triggers。`account_deletions.retry_at`が到来した行の未完了段階を再実行する) | adapter | `src/server/modules/account/adapter/retryAccountDeletions.ts`(DESIGN.md「インフラ」の定期実行②) |
| 新規登録フォーム・規約同意チェック | front | `src/front/pages/Signup.tsx` |
| ログインフォーム・next復帰 | front | `src/front/pages/Login.tsx`, `src/front/components/RequireAuth.tsx`(セットアップissueでテンプレートに実在するか確認する。無ければ新規作成) |
| アカウント設定画面・退会確認モーダル | front | `src/front/pages/AccountSettings.tsx` |

## エッジケースの決定

- **空・最小データ**: ニックネーム未入力はCognito SignUp前のフロント側バリデーションで送信不可にする
- **上限・境界値**: パスワード強度はCognito User PoolのIaCで最低8文字・大小英字・数字必須を明示設定して採用し、アプリ側では独自検証しない
- **エラー時に見えるもの**: Cognitoからのエラーは日本語ラベルにマッピングして表示する。少なくとも次の4種は個別に定義すること: `UsernameExistsException`(メール重複)、`InvalidPasswordException`(パスワード強度不足)、`CodeMismatchException`(確認コード誤り)、ログイン時の`UserNotFoundException`/`NotAuthorizedException`(BR-001に従い単一メッセージに統合)。マッピング表本体は実装時にfeatures外の定数ファイルに定義してよい
- **並行操作・二重実行**: `POST /api/account/provision`はネットワーク再送等での二重呼び出しに対して冪等(既存行があれば200で返す)
- **再表示時の整合**: JWTの署名が有効でも、削除要求が存在するユーザーは認可しない。`POST /api/account/provision`も例外にせず旧JWTでの再作成を拒否する。削除要求がある通常APIは403 `ACCOUNT_DELETING`とし認証状態を破棄する。削除要求がなく有効JWTのusersが未作成ならGET /api/meは404 `PROFILE_NOT_PROVISIONED`を返し、JWTを保持してプロフィール入力・規約同意へ戻りprovisionを再試行する。削除済み識別子はこの復帰経路へ入れない。削除APIの同一要求再送だけはジョブ照合を許可する。

## テスト方針

- 単体: `provisionAccount`(冪等性)、`deleteAccount`(関連テーブルの削除、opponent_idのnull化、**匿名化UPDATEがusers削除より前に実行されること**)、`account_settings`の部分更新マージロジック
- 結合: `PATCH /api/account/settings`の認証ガード(未認証401)、`DELETE /api/account`後に`GET /api/me`的な情報取得が失敗すること
- E2E(golden path): 新規登録 → チュートリアル表示 → ログアウト → ログイン → アカウント設定でキー割当変更 → 退会確認モーダルで確定 → トップに戻る、の一連

- 削除結合試験: Cognito失敗／D1失敗／応答消失からの再試行、旧JWTでのprovision拒否、削除済み相手名の置換を確認する。

### 登録・再設定の復旧経路
Cognito登録後のニックネーム拒否、D1障害、provision応答消失は、再登録せず同じsubでプロフィール入力から再試行する。ログイン後も未provisionなら同じ経路へ進む。ログイン後のnextは同一オリジン内の許可した相対パスに限定する。

ForgotPasswordの成功とアカウント不在は、共通の「登録されている場合は確認コードを送信しました」を表示する。確認コード誤り・期限切れは再入力／再送を案内する。provisionの400・503・応答消失後の復旧、未登録メールでの共通表示を結合／E2E試験に含める。

退会で最後のreplays参照が消えるroom_idだけ、同じD1 batchでreplay_uploadsをdiscardedにして共有チャンクを削除する(上記②-a/②-b。**replays削除より前**に実行する)。未完了stagingを参照0件という理由で全件掃除しない。遅延結果との競合はbattle.md「保存と退会の競合」に従う。
