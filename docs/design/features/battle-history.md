# 対戦結果・履歴・リプレイ
<!-- 変更履歴 [2026-09-18]: 保存期間を確定し取得時の期限検証を追加。 -->

- 種別: 機能設計書
- 対象 UC: UC-007(対戦結果・履歴を確認する)

## 何を作るか

対戦直後の結果サマリー表示、登録プレイヤーの過去対戦一覧、リプレイ再生を扱う。ゲストプレイヤーの対戦は保存しない(docs/design/USECASES.md UC-004 BR-004)。

## 入出力と振る舞い

| 操作 | 画面に起きること |
|---|---|
| 対戦終了直後(`battle_end`受信済み) | features/battle.mdの`battle_end`メッセージの`summary[side]`(消去/送出ライン)と`summary.durationMs`(対戦時間)を結果画面に表示する(追加API呼び出し不要)。`side`は対戦開始時の`joined`メッセージ(再接続した場合は`resume`)で受け取った値を結果画面まで保持して使う |
| 対戦終了直後(`battle_end`未受信のまま結果画面に来た) | sessionStorageに保持した`resultReceipt`を`X-Room-Result-Receipt`に載せて`GET /api/rooms/:roomId/result`を呼ぶ。**202(未決着)は3秒間隔で最大10回ポーリング**、200なら通常どおり結果を表示、**404は「結果を取得できませんでした」を表示してトップへ戻す**(10回を超えても200にならない場合も同じ扱い)。receiptが無い場合はAPIを呼ばず同じ表示にする。**この回復経路では`joined`/`resume`のsideを保持できていない場合があるため、応答の`side`フィールドを使って`summary[side]`を選ぶ**。数値と挙動の正本はfeatures/battle.md「終了結果の再取得」 |
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
- レスポンス(200): `{ "record": { "wins": number, "losses": number, "draws": number }, "items": [{ "id": string, "opponentLabel": string, "side": "P1"|"P2", "result": "win"|"lose"|"draw", "linesCleared": number, "linesSent": number, "durationMs": number, "endedReason": "normal"|"forfeit_timeout"|"draw_timeout", "createdAt": string }], "nextCursor": string|null }`(`record`はUS-013「通算勝敗」に対応する集計。ページネーションと独立して常に全期間の合計を返す。`side`は`battle_results.side`で、リプレイ再生時に「本人の盤面」がどちらかを判定するために使う)

### GET /api/battle-results/:id/replay
- 認証: 必須(かつ本人の`battle_results`行のみ取得可。他人のidを指定した場合404)
- レスポンス(200、manifest): `{ "battleResultId": string, "version": 1, "side": "P1"|"P2", "chunkCount": number, "chunks": [{ "index": number, "startTMs": number, "endTMs": number }], "startedAt": string, "endedAt": string, "createdAt": string }`(`battleResultId`は`battle_results.id`。`replays.id`自体はクライアントに公開しない。`side`は当該`battle_results.side`で、再生画面はこの値で本人の盤面を判定する。`chunks`は`replays.chunks`に保存した時刻索引で、シークに使う。定義はfeatures/battle.md「リプレイ」と一致させる)
- エラー: 対象が存在しない/他人のもの → 404 `REPLAY_NOT_FOUND`。`replays`テーブルのexpires_at <= サーバー現在時刻の場合はCron削除前でも同じ404。本体が既に無い場合も同じ404

### GET /api/battle-results/:id/replay/chunks/:index
認証・本人確認・期限判定はmanifest取得と同じ。indexは0以上chunkCount未満の整数。応答200は `{index,hash,data:<version1イベント配列のJSON文字列>}`。対象なし・期限切れ・他人は404 REPLAY_NOT_FOUND。欠落／hash不整合は再生不可と表示する。manifestと各チャンク応答はCache-Control: no-store。
フロントはチャンクを順次読み、**シーク時はmanifestの`chunks`から対象時刻を含むチャンク(`startTMs <= tMs <= endTMs`)を取得し、そのチャンク内の指定時点以下の最後のsnapshotで各sideの盤面を復元する**。指定時点以下のsnapshotが無いsideは前方のチャンクを順に遡る。**取得済みチャンクはメモリに保持して再取得しない**。全ログを一つのHTTPレスポンスやD1行にまとめない。初期両盤面・おじゃま・自然落下を含む録画と再生の一致、チャンク境界とシーク、取得途中の期限切れを検証する。

### GET /api/rooms/:roomId/result
`battle_end`を受け取れなかった結果画面の回復に使う。仕様・ポーリング値の正本はfeatures/battle.md「終了結果の再取得」(実装はbattle側の`src/server/modules/battle/adapter/roomResult.ts`)。本ファイルは呼び出し側(結果画面)の挙動だけを定める。応答(200)は `{ "side": "P1"|"P2", "winner": "P1"|"P2"|null, "reason": "normal"|"forfeit_timeout"|"draw_timeout", "summary": { "P1": {...}, "P2": {...}, "durationMs": number }, "endedAt": string }` で、`side`はreceiptから解決された呼び出し元自身の席。

## 実装の配置

| 処理 | 層 | 実装先ファイル |
| --- | --- | --- |
| 履歴一覧取得(battle_results SELECT、カーソルページネーション) | adapter | `src/server/modules/history/adapter/listBattleResults.ts` |
| リプレイmanifest取得(replays SELECT、本人チェック) | adapter | `src/server/modules/history/adapter/getReplay.ts` |
| リプレイチャンク取得(`GET /api/battle-results/:id/replay/chunks/:index`、本人チェック・期限判定) | adapter | `src/server/modules/history/adapter/getReplayChunk.ts` |
| リプレイの定期削除(Cron Triggers。`replays`・`replay_chunks`・`replay_uploads`の**3テーブルすべて**を担当する) | adapter | `src/server/modules/history/adapter/purgeExpiredReplays.ts`(DESIGN.md「インフラ」の定期実行①) |
| 結果画面(battle.mdのsummary表示。`battle_end`未受信時は`GET /api/rooms/:roomId/result`で回復) | front | `src/front/pages/Result.tsx` |
| 終了結果の再取得ルート(`GET /api/rooms/:roomId/result`、receipt検証とBattleRoom DO照会) | adapter | `src/server/modules/battle/adapter/roomResult.ts`(features/battle.mdと共通。battle側に配置する) |
| 履歴一覧・ゲスト向け案内 | front | `src/front/pages/History.tsx` |
| リプレイ再生(再生/一時停止/シーク) | front | `src/front/pages/Replay.tsx`, `src/front/game/replayPlayer.ts` |

## エッジケースの決定

- **空・最小データ**: 履歴0件の場合は「まだ対戦履歴がありません」を表示(UI_SKETCH.htmlの空状態パターンに準ずる)
- **上限・境界値**: 一覧は20件ずつカーソルページネーション。リプレイ保存期間30日(DESIGN.md横断規約)を過ぎたものは`replays`・`replay_chunks`の行を削除するが、`battle_results`行(サマリー)自体は無期限保持のため一覧には残り続け、選択時にのみ404となる。録画量の上限(チャンク40)超過で`replays`行が作られなかった対戦も同様に一覧には残り、選択時に404となる(features/battle.md「リプレイ」)
- **エラー時に見えるもの**: 他人の`battle_results.id`を直接叩いた場合も404(存在しないケースと区別しない。IDOR対策)
- **並行操作・二重実行**: 該当なし(読み取り専用API)
- **再表示時の整合**: 対戦直後の結果画面はWebSocketメッセージの`summary`をそのまま使うため、この時点ではまだD1へのINSERTと競合しない(表示用データと永続化が独立している)。履歴一覧に反映されるのはD1書き込み完了後(通常は対戦終了とほぼ同時)

## テスト方針

- 単体: カーソルページネーションの境界(0件、ちょうど20件、21件目)
- 結合: 他人の`battle_results.id`指定時の404、ゲストの`battle_results`が作成されないこと(battle.mdのINSERT処理と合わせて確認)、期限切れリプレイの404
- E2E(golden path): 対戦終了 → 結果画面表示 → 履歴ページで該当行を確認 → クリックしてリプレイが再生されることをPlaywrightで確認

保存方針はDESIGN.md「承認済みの運用方針」を参照。期限直前・期限と同時刻・期限直後と、Cronが未実行のケースを取得APIの結合試験で確認する。無期限保持は退会による本人データ削除を妨げない。
