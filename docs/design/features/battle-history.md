# 対戦結果・履歴・リプレイ
<!-- 変更履歴 [2026-09-18]: 保存期間を確定し取得時の期限検証を追加。 -->

- 種別: 機能設計書
- 対象 UC: UC-007(対戦結果・履歴を確認する)

## 何を作るか

対戦直後の結果サマリー表示、登録プレイヤーの過去対戦一覧、リプレイ再生を扱う。ゲストプレイヤーの対戦は保存しない(docs/design/USECASES.md UC-004 BR-004)。

## 入出力と振る舞い

| 操作 | 画面に起きること |
|---|---|
| 対戦終了直後 | features/battle.mdの`battle_end`メッセージの`summary[side]`(消去/送出ライン)と`summary.durationMs`(対戦時間)を結果画面に表示する(追加API呼び出し不要)。`side`は対戦開始時の`joined`メッセージ(features/battle.md)で受け取った値を結果画面まで保持して使う |
| 結果画面で「通報する」を選ぶ | ReportModalを開き送信する(features/report.md参照) |
| 登録プレイヤーが「履歴」を選ぶ | `GET /api/battle-results`で新しい順一覧を取得 |
| ゲスト/未ログインが履歴ページへ | ログイン強制はせず「履歴は保存されません」の案内を表示(UC-007代替フローA) |
| 履歴の1件を選ぶ | `GET /api/battle-results/:id/replay`でリプレイ取得、再生画面へ |
| リプレイデータが無い/破損 | 「リプレイを再生できません」を表示(UC-007 E1) |

UI_SKETCH.html「Result」「History」「Replay」画面に対応。

## API

### GET /api/battle-results
- 認証: 必須
- クエリ: `?limit=20&cursor=<string>`(カーソルページネーション、既定20件。`cursor`は`created_at`と`id`の複合値をbase64エンコードしたもの。同一ミリ秒内の複数行があっても欠落・重複しない)
- レスポンス(200): `{ "record": { "wins": number, "losses": number, "draws": number }, "items": [{ "id": string, "opponentLabel": string, "result": "win"|"lose"|"draw", "linesCleared": number, "linesSent": number, "durationMs": number, "endedReason": "normal"|"forfeit_timeout"|"draw_timeout", "createdAt": string }], "nextCursor": string|null }`(`record`はUS-013「通算勝敗」に対応する集計。ページネーションと独立して常に全期間の合計を返す)

### GET /api/battle-results/:id/replay
- 認証: 必須(かつ本人の`battle_results`行のみ取得可。他人のidを指定した場合404)
- レスポンス(200): `{ "battleResultId": string, "version": 1, "chunkCount": number, "startedAt": string, "endedAt": string, "createdAt": string }`(`battleResultId`は`battle_results.id`。`replays.id`自体はクライアントに公開しない)
- エラー: 対象が存在しない/他人のもの → 404 `REPLAY_NOT_FOUND`。`replays`テーブルのexpires_at <= サーバー現在時刻の場合はCron削除前でも同じ404。本体が既に無い場合も同じ404

## 実装の配置

| 処理 | 層 | 実装先ファイル |
| --- | --- | --- |
| 履歴一覧取得(battle_results SELECT、カーソルページネーション) | adapter | `src/server/modules/history/adapter/listBattleResults.ts` |
| リプレイ取得(replays SELECT、本人チェック) | adapter | `src/server/modules/history/adapter/getReplay.ts` |
| リプレイの定期削除(Cron Triggers) | adapter | `src/server/modules/history/adapter/purgeExpiredReplays.ts`(DESIGN.md「インフラ」の定期実行) |
| 結果画面(battle.mdのsummaryをそのまま表示) | front | `src/front/pages/Result.tsx` |
| 履歴一覧・ゲスト向け案内 | front | `src/front/pages/History.tsx` |
| リプレイ再生(再生/一時停止/シーク) | front | `src/front/pages/Replay.tsx`, `src/front/game/replayPlayer.ts` |

## エッジケースの決定

- **空・最小データ**: 履歴0件の場合は「まだ対戦履歴がありません」を表示(UI_SKETCH.htmlの空状態パターンに準ずる)
- **上限・境界値**: 一覧は20件ずつカーソルページネーション。リプレイ保存期間30日(DESIGN.md横断規約)を過ぎたものは本体データ(`replay_chunks.data`)を削除するが、`battle_results`行(サマリー)自体は無期限保持のため一覧には残り続け、選択時にのみ404となる
- **エラー時に見えるもの**: 他人の`battle_results.id`を直接叩いた場合も404(存在しないケースと区別しない。IDOR対策)
- **並行操作・二重実行**: 該当なし(読み取り専用API)
- **再表示時の整合**: 対戦直後の結果画面はWebSocketメッセージの`summary`をそのまま使うため、この時点ではまだD1へのINSERTと競合しない(表示用データと永続化が独立している)。履歴一覧に反映されるのはD1書き込み完了後(通常は対戦終了とほぼ同時)

## テスト方針

- 単体: カーソルページネーションの境界(0件、ちょうど20件、21件目)
- 結合: 他人の`battle_results.id`指定時の404、ゲストの`battle_results`が作成されないこと(battle.mdのINSERT処理と合わせて確認)、期限切れリプレイの404
- E2E(golden path): 対戦終了 → 結果画面表示 → 履歴ページで該当行を確認 → クリックしてリプレイが再生されることをPlaywrightで確認

保存方針はDESIGN.md「承認済みの運用方針」を参照。期限直前・期限と同時刻・期限直後と、Cronが未実行のケースを取得APIの結合試験で確認する。無期限保持は退会による本人データ削除を妨げない。

### GET /api/battle-results/:id/replay/chunks/:index
認証・本人確認・期限判定はmanifest取得と同じ。indexは0以上chunkCount未満の整数。応答200は `{index,hash,data:<version1イベント配列のJSON文字列>}`。対象なし・期限切れ・他人は404 REPLAY_NOT_FOUND。欠落／hash不整合は再生不可と表示する。manifestと各チャンク応答はCache-Control: no-store。
フロントはチャンクを順次読み、シーク時は必要位置まで取得して各sideの直近盤面を採用する。全ログを一つのHTTPレスポンスやD1行にまとめない。初期両盤面・おじゃま・自然落下を含む録画と再生の一致、チャンク境界とシーク、取得途中の期限切れを検証する。
