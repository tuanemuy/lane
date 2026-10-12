# `apps/cli` がスキーマの反映の要求に添える値

`apps/cli` のスキーマの反映のコマンドが、HTTP API の要求に添える `jobId` と `baseSchemaVersion` と、開発環境に保存する示した版。契約は [app-api.md](../../usecases/app-api.md) の「要求とユースケースの対応」のスキーマの反映と、[schema-migration.md](../../scenario/schema-migration.md) の「共通の前提」が定める。CMS の応答は、試験用の入口が返す。断りが無い限り、コマンドは同じ接続先と同じトークン T を使う。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-app-api-cliSchemaRequests-01 | 開発環境に、接続先と T の保存した版が無い。現行スキーマの版は 3 | 反映のコマンドを実行する | 要求の前に `GET /api/v1/schema` を呼び、版 3 を示してから、`baseSchemaVersion` 3 を添えて `POST /api/v1/schema/applies` を呼ぶ |
| TC-app-api-cliSchemaRequests-02 | 事前確認のコマンドが、版 3 を示して保存した。その後、別の反映が確定し、現行スキーマの版は 4 | 反映のコマンドを実行する | `baseSchemaVersion` 3 を添える。要求の直前に版を読み直さない |
| TC-app-api-cliSchemaRequests-03 | 事前確認のコマンドが版 3 を示した後、適用状況の確認のコマンドが、`data` の版 4 を示した | 反映のコマンドを実行する | `baseSchemaVersion` 4 を添える |
| TC-app-api-cliSchemaRequests-04 | 反映のコマンドの要求が受け付けられ、ジョブが `succeeded`（`success.schemaVersion` は 5）で終わった | 次の反映のコマンドを実行する | `baseSchemaVersion` 5 を添える |
| TC-app-api-cliSchemaRequests-05 | 保存した版は 3。反映のコマンドの要求が、版が変わったことを理由に拒否された。次の反映は、受け付けられた後に失敗で終えた | 次の反映のコマンドを実行する | `baseSchemaVersion` 3 を添える。保存した版は変わらない |
| TC-app-api-cliSchemaRequests-07 | 保存した版は 3。差分が無いことを返した反映の要求の `data` の現在の版は 3 | 適用済みとしての記録のコマンドを実行する | `baseSchemaVersion` 3 を添える |
| TC-app-api-cliSchemaRequests-08 | 同じ接続先で、T の保存した版は 3。別のトークン T2 の保存した版は 7 | T2 で反映のコマンドを実行する | `baseSchemaVersion` 7 を添える。T の保存した版は 3 のまま |
| TC-app-api-cliSchemaRequests-09 | 反映のコマンドの要求の応答を得られなかった | 同じ要求を再送する | 最初の要求と同じ `jobId` と同じ `baseSchemaVersion` を添える |
| TC-app-api-cliSchemaRequests-10 | 直接反映の要求が、`SCHEMA_LOSS_NOT_ACKNOWLEDGED` で拒否された。その要求の `baseSchemaVersion` は 3 | 開発者が確認し、`lossAcknowledged` を真にした要求を送る | `baseSchemaVersion` 3 と、前の要求と違う新しい `jobId` を添える |
