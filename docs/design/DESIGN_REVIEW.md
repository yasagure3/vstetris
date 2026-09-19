# 設計チェック記録

実施日: 2026-09-18〜2026-09-19。対象: USER_STORIES.md、USECASES.md、DESIGN.md、features全9件。

## 状態
独立した2名が分担して2回の全文レビューを実施。第1回はhigh6件・medium9件、第2回は重複を除きhigh3件・medium8件。第2回の修正案まで反映したが、最終修正の独立再確認は未実施。

第3回(2026-09-19)として、第2回までの修正を反映した更新後docsを対象に独立再確認を2グループで実施し、high10件・medium14件前後・low数件の指摘を得た。指摘は下記「第3回の指摘と修正」の通りdocsへ反映済みである。**ただしこの第3回修正に対する最終の独立再確認はまだ実施していないため、引き続き設計チェック合格とは扱わない。**

## 第2回の指摘と修正案
| 重要度 | 指摘 | 修正 |
|---|---|---|
| high | 退会後の結果保存が外部キー違反 | D1 batch内の条件付きINSERT、相手匿名化、対象0名も成功 |
| high | 参照0件の清掃が転送中チャンクを消す | staging/finalized/discardedの管理、公開と削除の原子的遷移 |
| high | 初回入室期限で進行中の再接続を拒否 | waiting期限と再接続deadlineの分離 |
| medium | ゲスト期限後も発行済みチケットが有効 | guest.expiresAtをチケット期限の上限にする |
| medium | lock_applied再送で巻戻し | 確認済みseqは副作用なし、board_updateも境界検証 |
| medium | initialize前cancelが消える | cancelRequested永続化と初期化後再確認 |
| medium | 正常移行closeと取消の混同 | 予約後は明示的cancelだけで取消 |
| medium | リプレイ型・hash未定義 | snapshot/end、保存UTF-8のSHA-256、連続index |
| medium | 切断側が結果を回復できない | 一試合限定resultReceipt、終了後24時間の回復API |
| medium | DO録画に削除期限なし | 転送成功／破棄／終了後30日で削除 |
| medium | 観戦waitingとstartedAtが旧仕様 | 未開始はstartedAt:null、createdAt分離 |

## 第3回の指摘と修正
| ID | 重要度 | 指摘 | 修正 | 反映先 |
|---|---|---|---|---|
| H1 | high | replay_uploads/replay_chunks/account_deletionsがスキーマ表・図・索引に無い | 3テーブルを列定義・FK・ON DELETE つきで表に追加、Mermaidに追記、索引を追加(reportsは(created_at,id)へ変更) | DESIGN.md |
| H2 | high | WSチケットにroleが無く、エラー一覧が不足 | リクエストに`role`(省略時player)を追加しpayload・接続時照合を明記。404/409/GUEST_SESSION_EXPIRED/ACCOUNT_DELETINGをコード名・HTTPステータスつきで表化 | DESIGN.md |
| H3 | high | WSメッセージ一覧が不完全(lock_result/resume/lock_applied/ping・pong/reaction/resultReceipt) | battle.mdに「WebSocketメッセージ一覧(正本)」を新設し全メッセージを定義。resumeにもresultReceiptを含め同一試合中は同値を再送 | battle.md |
| H4 | high | D1 batchが前文の結果で分岐できない | 確定batchを固定順・固定文数のSQL文列(①条件付きINSERT×2 ②replays ③uploads ④chunks ⑤応答用SELECT)として明記。POST応答`{status,created}`を定義。「両者ゲスト」→「両者ゲスト、または有効性条件を満たす登録者が0名」 | battle.md |
| H5 | high | usersを先に消すとopponent_label匿名化の対象を特定できない | 退会batchを①匿名化UPDATE→②本人データ削除→③users削除の固定順に明記 | account.md, battle.md, DESIGN.md |
| H6 | high | battle_end未受信時の結果回復手順が無い | receiptによる`GET /api/rooms/:roomId/result`呼び出し、202は3秒×最大10回、404はトップへ。Result.tsx / roomResult.tsを配置に追加 | battle.md, battle-history.md |
| H7 | high | リプレイで本人の盤面を判定できない | `battle_results.side`(text, NOT NULL)を追加し、履歴一覧itemsとreplay manifestにsideを含める | DESIGN.md, battle-history.md, battle.md |
| H8 | high | private waiting除外が読み側と書き側で矛盾 | 書き側で除外に統一し、activeRooms.tsから除外ロジックの記述を削除 | spectate.md |
| H9 | high | KV値スキーマ未定義(list()は値を返さない) | `match:<roomId>`のmetadata JSONを定義。1024バイト制約の根拠、list()のmetadataのみで構成、返却上限100件を明記 | DESIGN.md, spectate.md |
| H10 | high | Cronが1種類しか書かれていない | ①期限切れリプレイ(3テーブル)②account_deletions再試行の2ジョブを実装先つきで表化 | DESIGN.md, battle-history.md, account.md |
| M2 | medium | /ws/matchmakeのメッセージが不足 | cancelled/error(ROOM_DISBANDED)/ping/pong、クライアント→サーバのcancel・pingを追記 | matchmaking.md |
| M3 | medium | upgrade拒否条件とテスト方針の表現がずれる | 404/409/401の条件を明記。「26文字roomIdで15秒」→「source=randomで15秒／source=privateで10分」 | battle.md |
| M4 | medium | 実装の配置に未記載のルートがある | roomResult.ts / replayChunks.ts / getReplayChunk.ts / roomTicketCheck.ts / purgeExpiredReplays.ts / retryAccountDeletions.tsを追加 | battle.md, battle-history.md, account.md |
| M5 | medium | リプレイ記録量に上限が無い | 同一sideの250ms以内の連続board_updateを間引き、チャンク上限40。超過時は録画中止・replays行を作らない(サマリーは保存) | battle.md, DESIGN.md |
| M6 | medium | snapshotのboard型が曖昧 | サーバ→クライアント形式(pendingAttack込み)と明記。resumeState・乱数状態は録画に含めない | battle.md |
| M7 | medium | シーク手順が未定義 | manifestに`chunks:[{index,startTMs,endTMs}]`を追加(`replays.chunks`に保存)、シーク規則と取得済みチャンクのメモリ保持を規定 | battle.md, battle-history.md, DESIGN.md |
| M8 | medium | 通報の保持期間・管理者判定・入力上限が未確定 | reportsはMVPで無期限保持・退会時も削除しない。グループ名`admins`と`cognito:groups`判定を確定。label 1〜20文字/LABEL_TOO_LONG、レート制限はCloudflare側 | report.md, DESIGN.md |
| M9 | medium | チュートリアルのゲスト名・チケット取得がUC-003 BR-002と不整合 | ゲスト名はsessionStorage(`vstetris.guestName`)、チケット取得はMatchmaking画面の責務。既定キー割当のみ表示 | tutorial.md, guest-session.md |
| M10 | medium | 観戦一覧のwaiting行の表示規則が無い | 経過時間の代わりに「開始待ち」、players 0件は「対戦者を待っています」 | spectate.md |
| M11 | medium | 「操作ログを再現」がリプレイ方式(スナップショット再生)と不一致 | UC-007ステップ7とUS-017の表現を盤面スナップショット再生に修正 | USECASES.md, USER_STORIES.md |
| M12 | medium | 3つの「15秒」が区別されていない | 承認済み方針にrandom入室期限15秒を追記し、enterDeadline/disconnectDeadline/heartbeatTimeoutの用語表を横断規約に新設 | DESIGN.md ほか全docs |
| L1 | low | 実在しない見出し「BattleRoomの状態」を参照 | 「BattleRoomの状態機械」へ修正 | spectate.md |
| L2 | low | GET /api/rooms/:roomId/resultの対応UCが不足 | UC-004, UC-005に修正 | DESIGN.md |
| L3 | low | replay chunksのAPIが節外に置かれ、削除対象の表現が不正確 | 「API」節へ移動。「replay_chunks.dataを削除」→「replay_chunks(とreplays)の行を削除」 | battle-history.md |
| L4 | low | reactionがbattle.mdのメッセージ一覧に無い | spectate.mdへの参照つきで1行追加(H3に含む) | battle.md, spectate.md |
| L5 | low | 「第2回設計レビューの補正」節が本文を上書きしている | 内容を本文へ畳み込み、節を「設計レビューによる変更履歴」に置き換え | DESIGN.md |

### 採用しなかった指摘(記録のみ)
- 「リプレイ(US-017)を後続フェーズへ分離する」「resultReceipt方式をやめてresume+finished判定に一本化する」: **MVPとして重い旨の指摘があったが、ユーザー承認済みスコープとして維持する。**
- UI_SKETCH.htmlの通報一覧に通報者列を追加する指摘: 今回はUI_SKETCH.htmlを変更しない(記録のみ)。

## UC対応
UC-001/002/008はaccount、UC-003はguest-session、UC-004はmatchmaking/battle、UC-005はroom/battle、UC-006はspectate、UC-007はbattle-history、UC-009/010はreport、UC-011はtutorialに対応。

## 検証範囲
本体は未scaffoldでpackage.jsonなし。アプリテスト・実Cloudflareデプロイは未実施。PoC報告を本体のテスト結果として扱わない。今回の確認は文書照合とgit diff --check。既存UI_SKETCH.htmlの差分は今回変更していない。

## 残る確認
通報本文・自己申告名の保持期間と退会時削除範囲は公開前に確定する。新しい着地ACK・復帰・チャンク転送・退会競合はPoC未検証で、実装時に結合試験する。Cognito成功系、実RTT、Hibernationとalarm、silent切断も未検証。

## 次工程の判断
dev-specフェーズ8の「2周しても残るhighは人間に提示して判断を仰ぐ」に従い、第2回high3件への具体的修正案を確認してからissueドラフトへ進む。第3回のhigh10件も同様に修正を反映済みだが独立再確認が未了のため、issueドラフトへ進む前に人間の確認を挟む。GitHub issue作成・コミット・push・本体実装は未実施。
