# 通報

- 種別: 機能設計書
- 対象 UC: UC-009(不適切な行為を通報する), UC-010(通報を確認する(運営者))

## 何を作るか

対戦中・結果画面・観戦中から不適切な行為を通報する機能と、運営者権限を持つアカウントが通報一覧を確認する機能。UC-009(作成)とUC-010(閲覧)は同じデータモデルを共有するため1ファイルにまとめる。

## 入出力と振る舞い

| 操作 | 画面に起きること |
|---|---|
| 対戦/結果/観戦画面で「通報する」 | 通報モーダル表示、理由選択+任意詳細入力 |
| 理由未選択で送信 | 送信不可・エラー表示(UC-009 E1) |
| 送信 | `POST /api/reports`、成功トースト表示 |
| 運営者が「通報一覧」を選ぶ | `GET /api/reports`で一覧取得。運営者権限が無い場合は「権限なし」画面(UC-010 E1) |
| 通報が0件 | 「通報はありません」を表示(UC-010代替フローA) |

UI_SKETCH.html「ReportModal」「AdminReports」画面に対応。

## API

### POST /api/reports
- 認証: 不要(ゲスト可)
- リクエスト: `{ "reporterLabel": string, "targetLabel": string, "reason": "inappropriate_nickname"|"cheating_suspected"|"other", "detail"?: string, "roomId"?: string }`。値の出所(呼び出し画面ごとにフロントが自動的に詰める。通報者の身元確認は行わない=MVPでは自己申告のまま記録する):
  - 対戦中・結果画面(features/battle.md, features/battle-history.md): `reporterLabel`=自分の`displayName`(`joined`メッセージで受け取った値)、`targetLabel`=相手の`displayName`(**`battle_start.peers`、再接続後は`resume.peers`から取得**。`joined.peers`は自分1件しか含まれない場合があるため使わない)
  - 観戦中(features/spectate.md): `reporterLabel`=**常に固定値`"観戦者"`**(観戦接続はチケットを使わず表示名を持たないため)。`targetLabel`=通報対象を選択させるUIが無いため、`battle_start`/`joined`で得た対戦者いずれかの`displayName`(実装時にUIで選択させる)
- レスポンス(201): `{ "id": string, "createdAt": string }`
- エラー: `reason`/`reporterLabel`/`targetLabel`欠落 → 400 `REASON_REQUIRED`/`REPORTER_LABEL_REQUIRED`/`TARGET_LABEL_REQUIRED`。`reporterLabel`/`targetLabel`が21文字以上 → 400 `LABEL_TOO_LONG`。`detail`が500文字超 → 400 `DETAIL_TOO_LONG`
- `reporterLabel`/`targetLabel`は**1〜20文字**(`nicknameFilter`と同じ文字数範囲。表示名の出所がニックネームまたは固定値「観戦者」であるため)
- **レート制限**: アプリ内では実装しない。Cloudflare側のrate limiting rule(インフラ設定)で`POST /api/reports`に対して行う。具体的な閾値は運用設定で確定する(DESIGN.md「未解決の論点」)

### GET /api/reports
- 認証: 必須+運営者権限。Cognitoグループ名は`admins`で、JWTの`cognito:groups`クレームに`admins`が含まれることを判定条件とする(DESIGN.md横断規約「運営者権限」が正本)
- クエリ: `?limit=20&cursor=<string>`(battle-history.mdと同じ複合カーソル方式)
- レスポンス(200): `{ "items": [{ "id": string, "reporterLabel": string, "targetLabel": string, "reason": string, "detail": string|null, "roomId": string|null, "createdAt": string }], "nextCursor": string|null }`
- エラー: 未認証 → 401。認証済みだが運営者権限なし → 403 `FORBIDDEN`(フロントは「権限なし」画面へ)

## 実装の配置

| 処理 | 層 | 実装先ファイル |
| --- | --- | --- |
| 通報の作成(reports INSERT) | adapter | `src/server/modules/report/adapter/createReport.ts` |
| 通報一覧取得(運営者権限チェック含む) | adapter | `src/server/modules/report/adapter/listReports.ts` |
| 運営者権限判定(JWTの`cognito:groups`クレームに`admins`が含まれるかを判定) | domain | `src/server/modules/report/domain/isAdmin.ts`(DESIGN.md横断規約「運営者権限」を実装、他機能から再利用可能な形にする) |
| 通報モーダル | front | `src/front/components/ReportModal.tsx` |
| 通報一覧画面 | front | `src/front/pages/AdminReports.tsx` |

## エッジケースの決定

- **空・最小データ**: `detail`は任意(未入力で送信可)
- **上限・境界値**: `reporterLabel`/`targetLabel`は1〜20文字(超過は400 `LABEL_TOO_LONG`)。`detail`は最大500文字(超過は400 `DETAIL_TOO_LONG`)。一覧は20件ずつカーソルページネーション
- **保持期間**: `reports`は**MVPでは無期限保持とし、退会時にも削除しない**(`reporter_label`/`target_label`は自己申告名のみで、確実な本人照合ができないため)。公開前に保持方針を再確定する(DESIGN.md横断規約「データ保持期間」「未解決の論点」)
- **エラー時に見えるもの**: 権限なし(403)はUI_SKETCH.htmlの「権限なし」専用画面を表示し、「ホームへ戻る」導線のみを出す
- **並行操作・二重実行**: 同一対戦・同一相手への重複通報は制限しない(docs/design/USECASES.md UC-009 BR-001の通り、MVPではすべて記録する)
- **再表示時の整合**: 通報一覧は表示のみ(既読/対応ステータスの管理はスコープ外、US-020 Won't相当)であるため、再表示時に特別な整合性考慮は不要

## テスト方針

- 単体: `isAdmin`判定(`cognito:groups`クレームの有無・`admins`以外のみ所属・複数グループ混在時)。`reporterLabel`/`targetLabel`の文字数境界(0/1/20/21文字)
- 結合: `POST /api/reports`が認証不要ルートとして機能すること、`GET /api/reports`の権限ガード(未認証401・非運営者403・運営者200)
- E2E(golden path): 対戦中に通報 → トースト表示。別途、運営者アカウントでログインし通報一覧に当該通報が表示されることをPlaywrightで確認
