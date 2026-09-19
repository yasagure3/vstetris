# 設計チェック記録

実施日: 2026-09-18〜2026-09-19。対象: USECASES.md、DESIGN.md、features全9件。

## 状態
独立した2名が分担して2回の全文レビューを実施。第1回はhigh6件・medium9件、第2回は重複を除きhigh3件・medium8件。第2回の修正案まで反映したが、最終修正の独立再確認は未実施。設計チェック合格とは扱わない。

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

## UC対応
UC-001/002/008はaccount、UC-003はguest-session、UC-004はmatchmaking/battle、UC-005はroom/battle、UC-006はspectate、UC-007はbattle-history、UC-009/010はreport、UC-011はtutorialに対応。

## 検証範囲
本体は未scaffoldでpackage.jsonなし。アプリテスト・実Cloudflareデプロイは未実施。PoC報告を本体のテスト結果として扱わない。今回の確認は文書照合とgit diff --check。既存UI_SKETCH.htmlの差分は今回変更していない。

## 残る確認
通報本文・自己申告名の保持期間と退会時削除範囲は公開前に確定する。新しい着地ACK・復帰・チャンク転送・退会競合はPoC未検証で、実装時に結合試験する。Cognito成功系、実RTT、Hibernationとalarm、silent切断も未検証。

## 次工程の判断
dev-specフェーズ8の「2周しても残るhighは人間に提示して判断を仰ぐ」に従い、上記high3件への具体的修正案を確認してからissueドラフトへ進む。GitHub issue作成・コミット・push・本体実装は未実施。
