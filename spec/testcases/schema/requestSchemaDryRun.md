# 適用の事前確認を要求する（requestSchemaDryRun）

断りが無い限り、実行者はスキーマ反映の許可（`schema_apply`）を持つトークン、現行スキーマの版は 3、適用済みの移行定義は M1、手元の列は M1、M2、スキーマの構造は M2 を適用した結果と一致し、要求は前提とする版 3 と新しい `SchemaApplyJobId` J を添える。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-schema-requestSchemaDryRun-01 | ワークスペースは `idle` | 適用の事前確認を要求する | `mode` が `dryRun` の `preparing` のジョブ J と、対象の `WorkspaceId` を返す。ジョブの `plan.baseSchemaVersion` は 3。ワークスペースは `idle` のまま、`WorkspaceSchema` を保存しない。`schema.apply_requested`（`jobId` J）を記録する |
| TC-schema-requestSchemaDryRun-02 | ワークスペースは、反映 J0 で `applying` | 適用の事前確認を要求する | 受け付ける。`preparing` のジョブ J を返す。ワークスペースは J0 で `applying` のまま |
| TC-schema-requestSchemaDryRun-03 | ワークスペースは、準備済みの反映 J0 で `applying` | 検証の後、受け付けの確定の前に J0 の確定が版を 4 にした状態で、適用の事前確認を要求する | `ConflictError`（`SCHEMA_CHANGED`）。ジョブ J を作らず、イベントを記録しない |
| TC-schema-requestSchemaDryRun-04 | スキーマは、移行定義に記録していない項目を加えている | 適用の事前確認を要求する | `BusinessRuleError`（`SCHEMA_STRUCTURE_MISMATCH`）。反映と同じ拒否の理由で拒否する。ジョブ J を作らない。ワークスペースの状態、現行スキーマ、データは変わらず、イベントを記録しない |
| TC-schema-requestSchemaDryRun-05 | 実行者のトークンは `schema_apply` を持たない | 適用の事前確認を要求する | `ForbiddenError`（`GRANT_MISSING`）。必要な許可を示す。ジョブを作らない |
| TC-schema-requestSchemaDryRun-06 | ジョブ J を受け付け済み。ジョブは `checked` | 同じ ID J と同じ内容で要求する | `checked` のジョブ J を現在の状態のまま返す。何も書かず、イベントを新たに記録しない |
| TC-schema-requestSchemaDryRun-07 | 同じスキーマ・移行定義の反映を ID J で受け付け済みで、ワークスペースは J で `applying` | 同じ ID J と同じスキーマ・移行定義で、適用の事前確認を要求する | `ConflictError`（`SCHEMA_JOB_ID_CONFLICT`）。ジョブ J は変わらない |
| TC-schema-requestSchemaDryRun-09 | ワークスペースは `idle`。J の要求は無い | 並行する二つの要求が、同じ ID J で、一方は反映を、もう一方は適用の事前確認を要求する | 一方だけを受け付ける。もう一方は `ConflictError`（`ALREADY_EXISTS`）になり、何も残さない。J のジョブは一件だけで、反映か事前確認のどちらか一方 |
| TC-schema-requestSchemaDryRun-10 | 手元の列は M1 で、未適用の移行定義は無い。スキーマは現行スキーマと同じ | 適用の事前確認を要求する | 受け付ける。`preparing` のジョブ J を返し、`schema.apply_requested` を記録する。差分が無いこととして返さない |
