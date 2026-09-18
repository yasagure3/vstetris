# ルーム対戦

- 種別: 機能設計書
- 対象 UC: UC-005(ルームを作成して友人と対戦する)

## 何を作るか

ランダムマッチングを介さず、ルームIDを共有して特定の相手と対戦を始める機能。対戦本編そのものはfeatures/battle.mdのBattleRoomをそのまま使う(ルームはBattleRoomのIDを事前に払い出す入り口にすぎない)。

## 入出力と振る舞い

| 操作 | 画面に起きること |
|---|---|
| 「ルームを作成」を選ぶ | `POST /api/rooms`でルームID発行、続けて`POST /api/ws-tickets`(`scope: "room"`)でチケットを取得し自分自身が`/ws/rooms/:roomId?role=player&ticket=...`へ接続(作成者が1枠を確保する。この接続でBattleRoomが`waiting`状態になりKVレジストリ`match:<roomId>`が作成される)、画面にID表示+コピー導線 |
| ルームIDを相手に共有 | (アプリ外での共有。UI上はコピー操作のみ提供) |
| 相手がルームIDで入室 | 同様にチケットを取得し`/ws/rooms/:roomId?role=player&ticket=...`へ接続、2人目が揃った時点でbattle.mdのフローに合流(`battle_start`配信) |
| 3人目が同じIDで対戦者として入室しようとする | 満室エラー(features/battle.md「API」の409相当と同じ仕組み) |

UI_SKETCH.html「Room」画面に対応。

## API

### POST /api/rooms
- 認証: 不要(ゲスト可)
- リクエスト: なし
- レスポンス(201): `{ "roomId": string, "expiresAt": string }`
- 挙動: `roomId`はCrockford Base32で8文字(CSPRNGで生成、KVに既存キーがあれば再生成)。KVに`room_created:<roomId>`として作成時刻を記録し、有効期限10分(既定値、TTLで自動失効)を設定する
- エラー: なし

以後の接続は features/battle.md の `WS /ws/rooms/:roomId?role=player|spectator` をそのまま使う。

## 実装の配置

| 処理 | 層 | 実装先ファイル |
| --- | --- | --- |
| ルームID発行・有効期限管理(KV) | adapter | `src/server/modules/room/adapter/createRoom.ts` |
| ルームID表示・コピー・入室待ちUI | front | `src/front/pages/Room.tsx` |

## エッジケースの決定

- **空・最小データ**: 該当なし(入力を取らないエンドポイントのため)
- **上限・境界値**: ルームIDの有効期限は10分(既定値、`room_created:<roomId>`のKV TTLで作成時刻起点に自動失効)。features/battle.mdの`waiting`タイムアウト(8文字roomIdは10分)は最初のplayer接続時刻起点のため、`room_created:`のTTLとは起点が異なりうる(作成〜入室までに時間がかかった場合、ルームの実質的な生存時間が最大約20分になりうる)。実害は軽微(誰も専有・課金しないため)なため、MVPでは起点の統一は行わないTTL失効後は`room_created:<roomId>`キー自体がKVから消えるため、「存在しない」状態がマッチメイキング経由(そもそもキーを書かない)と区別できなくなる問題がある。これを**roomIdの形式で区別する**ことで解決する: room.mdが発行するroomIdはCrockford Base32の**8文字**(本節「POST /api/rooms」参照)、matchmaking.mdが発行するroomIdは**ULID(26文字)**とし、battle.mdのゲート判定は「roomIdが8文字なら`room_created:`の存在(未失効)を必須とし、26文字なら経路チェック自体を行わない」というroomIdの長さだけで機械的に判定する(features/battle.md「API」節参照)
- **エラー時に見えるもの**: 期限切れ・存在しないルームIDへのアクセスは「ルームが見つかりません」を表示しトップへ戻す導線を出す
- **並行操作・二重実行**: 作成者が「ルームを作成」を連打した場合、都度新しいルームIDが発行される(冪等にしない。MVPでは連打対策はUIのボタン無効化のみで十分とする)。作成者が接続する前に(ルームIDを推測または横取りした)他の2名が先に入室してしまうケースはMVPでは対策しない(ルームIDは第三者に知られない前提のためリスクは低いと判断。既知の制約として許容する)
- **ルーム解散(UC-005 E2)**: 能動的な解散APIは設けない。作成者が入室を待たずに離脱した場合はフロント側で作成状態を破棄するのみとし、KVの`room_created:<roomId>`は10分TTLで自然失効する。作成者が接続済み(=BattleRoomが`waiting`)の場合は、features/battle.mdの状態機械により8文字roomIdの`waiting`タイムアウト(10分、`room_created:`と同じ長さに揃えてある)で自動解散する。誰も専有・課金しないため実害はない
- **再表示時の整合**: 作成者がルーム作成後にページを再読み込みすると、発行済みのroomIdを保持していない限り再作成が必要になる(MVPではURLクエリ等での永続化はスコープ外)

## テスト方針

- 単体: ルームID生成の一意性、有効期限判定ロジック
- 結合: `POST /api/rooms`のレスポンス形式、期限切れルームへの入室拒否、2人目入室でbattle.mdのBattleRoomフローへ正しく合流すること
- E2E(golden path): ルーム作成 → 別ブラウザコンテキストでルームID入力 → 2人揃って対戦画面に遷移することをPlaywrightで確認
