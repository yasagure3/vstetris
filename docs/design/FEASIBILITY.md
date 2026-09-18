# 技術検証（Feasibility Check）

## 対象機能
対戦テトリス(webapp)。Cloudflare Workers上に [fullstack-worker-template](https://github.com/skanehira/fullstack-worker-template) を使ってデプロイする。1vs1のリアルタイム対戦(UC-004)・観戦(UC-006)が本検証の主対象。

**前提調査**: fullstack-worker-template は React 19 + React Router v8 + Tailwind CSS v4(フロントエンド) / Hono + Cloudflare D1 + Drizzle ORM(バックエンド) / Amazon Cognito(認証) という構成で、**WebSocket・Durable Objectsは含まれていない**。リアルタイム対戦同期は本プロジェクトで新規に組み込む必要がある。

## 技術的不確実性

| ID | 不確実性 | 種類 | 影響度 | 発生確率 | リスクレベル |
|----|----------|------|--------|----------|-------------|
| R1 | Durable Objects + WebSocketでの1vs1盤面リアルタイム同期・切断検知 | 技術的実現性 | 致命的 | 中 | 🔴 最優先 |
| R2 | マッチメイキング待機列をDurable Objectでキュー管理し対戦部屋を生成する流れ | 技術的実現性 | 重大 | 中 | 🟡 優先 |
| R3 | テンプレートのCognito認証フローとゲストプレイヤー(認証不要)経路の共存 | 統合 | 重大 | 中 | 🟡 優先 |
| R4 | Durable Object上の対戦結果をCloudflare D1(Drizzle経由)へ書き戻す | 統合 | 中程度 | 低 | 🟢 通常 |
| R5 | 観戦者への対戦盤面ブロードキャスト(3人目以降のWebSocket配信)と進行中対戦一覧の取得 | 技術的実現性 | 重大 | 中 | 🟡 優先 |

## 検証項目

### R1: Durable Objects + WebSocketでの1vs1盤面リアルタイム同期・切断検知

**検証内容**:
- Cloudflare Durable ObjectsでWebSocket接続を持つ対戦ルーム(BattleRoom)を実装し、2クライアント間でメッセージを往復できるか確認する
- 片方のクライアントを意図的に切断し、DO側で切断検知〜再接続待ち〜タイムアウトの一連の挙動が実現できるか確認する

**成功基準**:
- ローカル(wrangler dev)環境で2クライアント間のメッセージ往復が動作し、送信時刻と受信時刻の差分(タイムスタンプ計測)の平均が200ms以内であること
- クライアント切断がDO側で検知され、再接続または一定時間後のタイムアウト処理が発火すること(UC-004のE2〜E4に対応)

**失敗した場合の代替案**:
- WebSocket Hibernation APIの利用や接続数上限の見直しなど実装方式の変更を検討する
- 同期頻度を下げる、または盤面全体ではなく差分のみを送るなど通信量を削減する

### R2: マッチメイキング待機列をDurable Objectでキュー管理し対戦部屋を生成する流れ

**検証内容**:
- MatchmakingQueueというDOクラスに複数クライアントを接続させ、キューに2人揃った時点で新しいBattleRoom DOのIDを払い出せるか確認する
- 払い出されたIDで両クライアントが同じBattleRoom DOへ再接続できるか確認する

**成功基準**:
- 2つのクライアントが同時に「対戦相手を探す」相当の操作をした際、双方が同じBattleRoom IDへ正しく誘導されること

**失敗した場合の代替案**:
- キューをシングルトンDOではなく定期実行のWorker(Cron Triggers)でマッチングする方式に変更する(Cron Triggersの最小実行間隔は1分のため、待機時間が最大60秒程度に悪化しうる点に留意する)
- ルーム対戦(UC-005)のみをMVPの対戦手段とし、ランダムマッチング(UC-004)の自動化範囲を縮小する(UC-004はコアユースケースのため要ユーザ判断)

### R3: テンプレートのCognito認証フローとゲストプレイヤー(認証不要)経路の共存

**検証内容**:
- テンプレートの認証ミドルウェア実装を確認し、ルート単位で認証必須/不要を切り分けられるか調査する
- 切り分けられない場合、認証不要ルート(ゲスト用マッチメイキング/対戦エンドポイント)を追加できるか実装して確認する

**成功基準**:
- 認証済みユーザー向けエンドポイントとは別に、認証不要でアクセスできる経路を`vp lint` / `vp check`(レイヤ境界のimport方向チェック)を壊さずに追加できることを確認する

**失敗した場合の代替案**:
- ゲストプレイヤーにも匿名Cognitoユーザー(Cognito Identity Pool等)を発行する方式に変更する
- MVPではゲストプレイ(US-002)を見送り、登録プレイヤーのみの対戦から始める(要ユーザ判断)

### R4: Durable Object上の対戦結果をCloudflare D1(Drizzle経由)へ書き戻す

**検証内容**:
- BattleRoom DOが対戦終了を検知した際、Worker経由でDrizzle ORMを使いD1へ結果行をINSERTできるか確認する

**成功基準**:
- 対戦終了トリガーからD1の対戦履歴テーブルへ1行が正しく保存されることを確認する

**失敗した場合の代替案**:
- DOから直接D1バインディングを持たせる、またはQueueを介した非同期書き込みに変更する

### R5: 観戦者への対戦盤面ブロードキャストと進行中対戦一覧の取得

**検証内容**:
- BattleRoom DOが対戦者2名に加えて観戦者(3人目以降)のWebSocket接続も受け入れ、対戦者間と同じ盤面更新イベントを観戦者にも配信できるか確認する
- Durable ObjectsにはインスタンスをAPIから直接列挙する機能がないため、稼働中のBattleRoom DOをD1やKV等の外部レジストリに登録し、「進行中の対戦一覧」(UC-006ステップ2)として取得できるか確認する

**成功基準**:
- 対戦者2名+観戦者2名が同時接続した状態で、全員が同じ盤面更新を受信すること
- 進行中の対戦一覧が、実際にアクティブなBattleRoom DOと一致すること

**失敗した場合の代替案**:
- リアルタイムプッシュ配信ではなく、一定間隔のポーリングで盤面を取得する方式に縮小する
- 進行中の対戦一覧を自動取得ではなく、ルームIDを知っている場合のみ観戦可能な方式に限定する(要ユーザ判断)

## PoC計画

### PoC-1: リアルタイム対戦同期(マッチメイキング〜対戦〜切断復帰)

**id**: realtime-battle-sync
**目的**: Durable Objects + WebSocketで、マッチメイキング〜1vs1対戦の盤面同期〜切断検知までの一連の流れが実現できるか検証する
**risk**: high
**blocker**: true

**スコープ**:
- 含む: MatchmakingQueue DO、BattleRoom DO、WebSocketでの2クライアント間メッセージ往復、意図的切断からの再接続/タイムアウト検知
- 含まない: 実際のテトリスゲームロジック全体、おじゃまラインの計算式、認証との統合、D1への永続化(PoC-4で扱う)、観戦者への配信(PoC-5で扱う)

**実装内容**:
1. wranglerで最小のCloudflare Workers + Durable Objectsプロジェクトを作成する
2. MatchmakingQueue DOを実装し、2クライアントが接続したらBattleRoom DOのIDを払い出す
3. BattleRoom DOでWebSocket接続を受け付け、片方から送ったメッセージがもう片方に転送されることを確認する
4. クライアントの一方を意図的に切断し、DO側の切断検知・再接続待ち・タイムアウト処理を確認する

**必要なリソース**:
- 技術: Cloudflare Workers, Durable Objects, WebSocket API, wrangler CLI
- 環境: wrangler devによるローカル実行環境(Cloudflareアカウントでのデプロイまでは不要)
- データ: なし(ダミーメッセージで検証)

**成功基準**:
- 2クライアント間のメッセージ往復が動作し、送受信タイムスタンプの差分の平均が200ms以内
- 切断検知・タイムアウト処理が期待通り発火する

**判断**:
- 成功 → 本実装(UC-004)のアーキテクチャ基盤として採用
- 失敗 → 代替案(R1の代替案)を検討し、スコープ縮小も含め再評価する

<!-- POC_NEEDED: id=realtime-battle-sync, scope=DO+WebSocketでの1vs1盤面リアルタイム同期・切断検知, risk=high, blocker=true -->
<!-- POC_STATUS: id=realtime-battle-sync, blocker=true, status=verified, confidence=0.92 -->

---

### PoC-2: Cognito認証とゲストプレイヤー経路の共存

**id**: cognito-guest-coexistence
**目的**: fullstack-worker-templateのCognito認証フローを壊さずに、認証不要のゲストプレイヤー経路を追加できるか検証する
**risk**: medium
**blocker**: true

**スコープ**:
- 含む: テンプレートの認証ミドルウェアの調査、認証不要ルートの追加実装、ゲスト用の一時識別子発行方式の検討
- 含まない: Cognitoの本番環境構築、ソーシャルログイン等の追加認証方式

**実装内容**:
1. fullstack-worker-templateをclone/セットアップし、`docker compose up -d` でmoto(Cognito IDPモック、localhost:5001)を起動する
2. `vp run cognito:setup` を実行し、Terraformでmoto上にUser Pool/Clientをプロビジョニングして `.dev.vars` / `.env.local` を生成する
3. `vp dev` で起動し、認証ミドルウェア(`src/server/modules/auth/adapter/` 配下の `authenticate` ミドルウェア、`verifyAccessToken.ts`)の実装箇所を確認する
4. ルート単位で認証必須/不要を切り分ける機構(例: `RequireAuth` 相当のコンポーネント/ミドルウェア)があるか調査する
5. なければ認証不要ルートを追加し、ゲスト用の一時識別子(セッションCookie等)を発行する最小実装を作る
6. 認証必須ルートと認証不要ルートが両方とも意図通りに動作することを確認する

**必要なリソース**:
- 技術: Hono, Amazon Cognito(ローカルはmoto+Terraformでのモック), Docker, fullstack-worker-templateのソース一式
- 環境: テンプレートのローカル開発環境(`vp dev`、事前に`docker compose up -d`と`vp run cognito:setup`が必要)
- データ: なし

**成功基準**:
- 認証必須ルートと認証不要ルートが共存し、`vp lint` / `vp check`(レイヤ境界のimport方向チェックを含む)がエラーなく通ること

**判断**:
- 成功 → DESIGN.mdの認証方式としてそのまま採用
- 失敗 → 代替案(匿名Cognitoユーザー発行 or ゲストプレイ見送り)をユーザに確認の上で採用

<!-- POC_NEEDED: id=cognito-guest-coexistence, scope=Cognito認証とゲストプレイヤー経路の共存, risk=medium, blocker=true -->
<!-- POC_STATUS: id=cognito-guest-coexistence, blocker=true, status=verified, confidence=0.85 -->

---

### PoC-3: マッチメイキングキューの多人数同時アクセス確認

**id**: matchmaking-queue-do
**目的**: MatchmakingQueue DOが3人以上の同時待機でも正しく2人ずつペアリングできるか検証する
**risk**: medium
**blocker**: false

**スコープ**:
- 含む: 3〜4クライアントが同時に待機した場合のペアリング順序・重複マッチの有無の確認
- 含まない: 数百人規模の負荷試験

**実装内容**:
1. PoC-1で作成したMatchmakingQueue DOに対し、3〜4個のクライアントをほぼ同時に接続させる
2. ペアリング結果が重複や取りこぼしなく2人1組になることを確認する

**必要なリソース**:
- 技術: PoC-1と同じ
- 環境: wrangler dev
- データ: なし

**成功基準**:
- 3〜4クライアント同時接続で、全員が過不足なくペアリングされる

**判断**:
- 成功 → そのままの設計で採用
- 失敗 → キューへのアクセスを直列化する実装(ロック相当の制御)を追加する

<!-- POC_NEEDED: id=matchmaking-queue-do, scope=マッチメイキングキューの多人数同時アクセス確認, risk=medium, blocker=false -->
<!-- POC_STATUS: id=matchmaking-queue-do, blocker=false, status=verified, confidence=0.88 -->

---

### PoC-4: 対戦結果のD1永続化

**id**: battle-result-persistence
**目的**: BattleRoom DOで検知した対戦終了イベントから、Cloudflare D1(Drizzle ORM経由)へ対戦結果を書き戻せるか検証する
**risk**: low
**blocker**: false

**スコープ**:
- 含む: DOからWorker API経由でのD1への対戦結果1行のINSERT
- 含まない: リプレイデータ全体の永続化、対戦履歴一覧APIの実装

**実装内容**:
1. fullstack-worker-templateのDrizzleスキーマ(`src/server/db/schema.ts`)に対戦結果テーブルの最小定義を追加する
2. drizzle-kitでマイグレーションファイルを生成し、`wrangler d1 migrations apply`(ローカルD1)でスキーマを適用する
3. PoC-1のBattleRoom DOに、対戦終了時にWorker側のAPIを呼び出す処理を追加する
4. D1に1行のINSERTが行われることを確認する

**必要なリソース**:
- 技術: Drizzle ORM, Cloudflare D1, fullstack-worker-templateのソース一式
- 環境: テンプレートのローカル開発環境
- データ: なし

**成功基準**:
- 対戦終了トリガーからD1テーブルへの1行保存が確認できる

**判断**:
- 成功 → 本実装(UC-004ステップ11、UC-007)の永続化方式として採用
- 失敗 → Queue経由の非同期書き込みなど代替方式を検討する

<!-- POC_NEEDED: id=battle-result-persistence, scope=DOからD1への対戦結果永続化, risk=low, blocker=false -->
<!-- POC_STATUS: id=battle-result-persistence, blocker=false, status=verified, confidence=0.92 -->

---

### PoC-5: 観戦機能(盤面ブロードキャストと進行中対戦一覧)

**id**: spectator-broadcast
**目的**: BattleRoom DOが対戦者2名に加えて観戦者にも盤面をリアルタイムブロードキャストできるか、また進行中の対戦一覧を提供できるかを検証する
**risk**: medium
**blocker**: false

**スコープ**:
- 含む: BattleRoom DOへの3人目以降のWebSocket接続(観戦者)の受け入れと盤面更新イベントの同時配信、稼働中BattleRoom DOの外部レジストリ(D1またはKV)への登録・一覧取得
- 含まない: リアクション(スタンプ)機能とそのレート制限(UC-006 BR-001)、観戦者数の上限制御、観戦一覧画面のUI実装

**実装内容**:
1. PoC-1で作成したBattleRoom DOに、観戦者用のWebSocket接続を受け付けるエンドポイント(例: `/rooms/:id/watch`)を追加する
2. 対戦者間で送受信している盤面更新メッセージを、接続中の観戦者全員にも同じタイミングでブロードキャストする実装を追加する
3. BattleRoom DO作成時にルームIDと対戦状態をD1(またはKV)の一時レジストリに登録し、対戦終了時に削除する処理を追加する
4. レジストリから「進行中の対戦一覧」を取得できることを確認する
5. 対戦者2クライアント+観戦者2クライアントを同時接続させ、全員に同じ盤面更新が配信されることを確認する

**必要なリソース**:
- 技術: PoC-1と同じ(Cloudflare Workers, Durable Objects, WebSocket API)に加えCloudflare D1またはKV(進行中対戦の登録用)
- 環境: wrangler devによるローカル実行環境
- データ: なし(ダミーメッセージで検証)

**成功基準**:
- 対戦者2名+観戦者2名が同時接続した状態で、全員の盤面表示が同じ更新内容を受け取ること
- 進行中の対戦一覧が、実際にアクティブなBattleRoom DOと一致すること

**判断**:
- 成功 → 本実装(UC-006)のアーキテクチャ基盤として採用
- 失敗 → 代替案(R5の代替案)を検討し、観戦機能のスコープ縮小も含め再評価する

<!-- POC_NEEDED: id=spectator-broadcast, scope=観戦者への盤面ブロードキャストと進行中対戦一覧取得, risk=medium, blocker=false -->
<!-- POC_STATUS: id=spectator-broadcast, blocker=false, status=verified, confidence=0.9 -->

---

## PoC 結果

### cognito-guest-coexistence — verified (confidence 0.85)
- 検証日: 2026-09-02
- 観測した事実: fullstack-worker-templateのソースを読解・実行して確認した結果、`authenticate`ミドルウェアはグローバル適用(`app.use("*", ...)`)されておらず、各ルートが`new Hono().get("/me", authenticate, handler)`のように個別にオプトインする方式だった。既存の`/api/health`が既に認証不要ルートとして`/api/me`と共存しており、ルート単位の切り分け機構は既に存在する。この方式に沿って認証不要ルート`/api/guest-ping`(HttpOnly Cookieでゲストid発行)を実装したところ、`vp lint`/`vp check`/workerテスト(16件)/フロントテスト(9件)/`vp build`が全てpass。domain層に意図的にhono importを注入してlintがエラーになる(EXIT=1)ことも逆検証し、レイヤ境界チェックが実際に機能していることを確認した。実HTTPでも`/api/guest-ping`が200+Cookie発行、`/api/me`はゲストCookieのみでは401のままであることを確認した。
- 結論: 成功基準(認証必須/不要ルートの共存、lint/checkが通ること)を満たす。ゲスト経路の追加に新しい仕組みは不要で、認証ミドルウェアを付けないルートを追加するだけでよい、という設計方針をDESIGN.mdに採用する。
- 未検証の範囲: Docker/terraformが検証環境になくmotoを起動できなかったため、有効なCognitoアクセストークンでの`/api/me`成功応答(200)は未確認(401側の異常系は実機確認済み)。ただしverifyAccessTokenの単体テストがjoseで自作署名トークンを使い正常系を含め検証済みであり、認証ロジック自体はオフラインで担保されている。
- 開発環境への申し送り: `git config core.autocrlf=true`のままcloneすると素の状態でも`vp check`がCRLFで落ちる。`core.autocrlf=false`でのcloneを推奨。

### matchmaking-queue-do — verified (confidence 0.88)
- 検証日: 2026-09-02
- 観測した事実: MatchmakingQueue DOを実装し、`wrangler dev`上で3〜4クライアントを同時接続させるテストを実施。ベースライン実装(read-modify-writeの間にstorage以外のawaitを挟まない)ではN=3/N=4を各5回、計10回すべてで重複・取りこぼし0のペアリングに成功。段階的到着(3人接続後に4人目が遅れて接続)でも正しくペアになった。一方、クリティカルセクション内にstorage以外のawait(setTimeout)を人為的に挟む対照実験では、N=4で3回中3回とも「lost update」(全員が自分だけをキューに積んで上書きし合い、誰もペアにならない)が発生することを確認。`state.blockConcurrencyWhile()`でクリティカルセクションを包むことでこの問題が解消することも実測で確認した。
- 結論: 成功基準を満たす。ただし付帯条件として「キュー更新のクリティカルセクションにstorage以外のawaitを入れない、または`blockConcurrencyWhile`で包む」という設計ルールを本実装(UC-004)の設計に明文化する必要がある。

### spectator-broadcast — verified (confidence 0.9)
- 検証日: 2026-09-02
- 観測した事実: WebSocket Hibernation API(`ctx.acceptWebSocket`/`getWebSockets`)を使いBattleRoom DOに対戦者2名+観戦者を接続できる実装を作成。3人目の対戦者接続は409で拒否されることを確認。対戦者2名+観戦者2名の計4クライアントで盤面更新を送ったところ、全員が完全一致する内容を受信(送信者へのack `delivered:4`)。観戦者を8名に増やしても`delivered:10`で全員一致。観戦者からの盤面更新送信はエラーで拒否されることも確認。KVレジストリ(進行中の対戦一覧)は接続・観戦者数の増減・対戦終了・全員切断のたびに正しく更新され、実際のDO接続状態と一致することを確認した。
- 結論: 成功基準(全員が同じ更新を受信、進行中対戦一覧が実DOと一致)を満たす。本実装(UC-006)の設計として採用する。
- 実装上の注意(申し送り): (1) `webSocketClose`内で`getWebSockets()`を呼ぶと切断中のソケットがまだ含まれ観戦者数カウントがずれるため、close対象を明示的に除外する必要がある。(2) 対戦終了(`battle_end`)でレジストリを削除した直後に`webSocketClose`のレジストリ同期処理が再作成してしまう競合があるため、DO storageに`finished`フラグを永続化し終了後は常に削除する実装にする必要がある。

### realtime-battle-sync — verified (confidence 0.92)
- 検証日: 2026-09-02
- 観測した事実: MatchmakingQueue DO + BattleRoom DOを実装し、`wrangler dev`上で実際にWebSocket通信を計測した。マッチング成立時、両クライアントが同一roomIdへ誘導されることを確認(R2の成功基準も同時に達成)。50通の盤面同期メッセージで片道平均1.58ms・往復平均2.58ms、20Hz×15秒(計600通)の持続負荷でも片道平均1.5ms台・p95=2ms・パケットロス0で、成功基準の200msに対して大きな余裕があった。切断検知は強制TCP切断(ws.terminate)で3〜8ms以内に検知、猶予時間内の再接続では正しくタイマーが解除され誤タイムアウトなし、再接続がない場合は猶予時間経過後(誤差20ms程度)にタイムアウト処理が発火し勝敗がDO storageに永続化されることを確認(UC-004のE2〜E4に相当する分岐を全て観測)。
- 結論: 成功基準(メッセージ往復平均200ms以内、切断検知・タイムアウト処理の発火)を満たす。本実装(UC-004)のアーキテクチャ基盤として採用する。
- 留意事項(本実装時に確認): (a) 計測はローカルループバックのため、実運用でのクライアント〜Cloudflareエッジ間のインターネットRTTは含まれていない。DOの中継処理自体のオーバーヘッドが約1〜2msでボトルネックにならないことが分かった、という位置づけで、本番デプロイ後に実RTTを含めた再計測を推奨する。(b) 本PoCは通常のWebSocket API(server.accept())を使用しており、長時間接続のコスト最適化に有効なWebSocket Hibernation APIは未検証。本実装時に再評価を推奨(切断検知ロジックはwebSocketCloseハンドラへの移植で同等に実現可能な見込み)。

### battle-result-persistence — verified (confidence 0.92)
- 検証日: 2026-09-02
- 観測した事実: fullstack-worker-templateをcloneし、Drizzleスキーマにbattle_resultsテーブルを追加、drizzle-kitでマイグレーション生成、`wrangler d1 migrations apply --local`でローカルD1に適用。BattleRoom Durable Object → self service binding → Hono API(`POST /api/battle-results`) → `drizzle(c.env.DB).insert(battleResults).values(...).returning()` という経路を実装し、`wrangler dev`起動後にPOSTでトリガーした結果、別プロセスの`wrangler d1 execute --local`によるSELECTでトリガー前0行→トリガー後1行(2回目実行後2行)を実測確認した。異常系(必須項目欠落)もzod検証でHTTP 400になることを確認。
- 結論: 成功基準(対戦終了トリガーからD1テーブルへの1行保存)を満たす。本実装の永続化方式として採用する。
- 開発環境への申し送り: Windows環境でMAX_PATH(260文字)制限により2回詰まった(pnpmの入れ子node_modulesでesbuild.exeがENOENT、miniflareのD1ファイルパス超過)。npmのフラットnode_modulesの利用と、`--persist-to`に短いパスを指定することで回避した。DESIGN.mdの開発環境注意事項に記載する。
- 追加の申し送り(マイグレーション生成順序): テンプレートの`migrations/0000_init.sql`は手書きで、drizzleのスナップショット(`migrations/meta/`)を持たない。この状態でいきなり新テーブルを`drizzle-kit generate`すると、`kv_example`を含む全量マイグレーションが生成されてしまう。そのため今回は先に`kv_example`のみでベースラインスナップショットを生成し、その後`battle_results`追加分を差分マイグレーション(0001)として生成する2段階の手順を踏んだ。本実装のセットアップissueでも同じ2段階手順を踏むこと。

---

## 技術スタック（案）

| レイヤー | 技術 | 選定理由 |
|----------|------|----------|
| フロントエンド | React 19 + React Router v8 + Tailwind CSS v4 | fullstack-worker-templateに準拠 |
| バックエンドAPI | Hono | fullstack-worker-templateに準拠 |
| リアルタイム対戦・観戦 | Cloudflare Durable Objects + WebSocket | テンプレートに無い機能。PoC-1(対戦同期)・PoC-5(観戦配信)で実現可能性を検証中 |
| 永続化(戦績・アカウント等) | Cloudflare D1 + Drizzle ORM | fullstack-worker-templateに準拠 |
| 認証(登録プレイヤー) | Amazon Cognito | fullstack-worker-templateに準拠。ゲスト経路との共存はPoC-2で検証中 |
| デプロイ | Cloudflare Workers(wrangler) | ユーザー指定 |

## 次のステップ

1. [x] PoC-1(realtime-battle-sync)を実施し、blocker=trueを解消する — verified (confidence 0.92)
2. [x] PoC-2(cognito-guest-coexistence)を実施し、blocker=trueを解消する — verified (confidence 0.85)
3. [x] PoC-3(matchmaking-queue-do)・PoC-4(battle-result-persistence)・PoC-5(spectator-broadcast)を実施する — いずれも verified (confidence 0.88 / 0.92 / 0.90)
4. [ ] 全PoC結果を踏まえてDESIGN.md(横断設計)に着手する
