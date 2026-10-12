# 反映を要求する（requestSchemaApply）

断りが無い限り、実行者はスキーマ反映の許可（`schema_apply`）を持つトークン、ワークスペースは `idle` で、未解消の失敗した反映（`failedApply`）を持たない。現行スキーマの版は 3、適用済みの移行定義は M1、要求は前提とする版 3 と新しい `SchemaApplyJobId` J を添える。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-schema-requestSchemaApply-01 | 手元の列は M1、M2、M3。スキーマのコンテンツの構造は、現行の構造へ M2、M3 を順に適用した結果と一致する | 反映を要求する | 受け付けたジョブ（ID は J、`mode` は `commit`、`status` は `preparing`、`Preparation.progress` は `{ processed: 0, total: null }`）と、対象の `WorkspaceId` を返す。`plan.source` は `migrations` で、`plan.migrations` と `plan.units` は M2、M3 の順。ジョブは、実行者と、受け付けた時点のトークンの表示用の名称と発行者の名前を持つ。ワークスペースは J で `applying` になり、現行スキーマ、版、適用済みの列は変わらない。`schema.apply_requested`（`jobId` J）を記録する。作るジョブは J の一件だけ |
| TC-schema-requestSchemaApply-03 | 手元の列は M1。スキーマは、現行スキーマから、文字数の制約、表示名称、フォーム contact の項目のキー（mail から email）だけを変えたもの | 反映を要求する | 受け付ける。`plan.migrations` は空。`plan.forms.changed` は、contact の `removedFields` に mail、`addedFields` に email を持つ。`plan.forms` の `added` と `removed` は空 |
| TC-schema-requestSchemaApply-04 | M2 は項目の削除（`removeField`、`lossConfirmed`）を含む。要求は、データを失うことの確認を持たない | 反映を要求する | 受け付ける。`plan.lossyOperations` は、M2 のその操作（`loss` は `field`、`migration` は M2）を持つ |
| TC-schema-requestSchemaApply-05 | M1 は項目 a を加え、M2 は a を削除する。スキーマは a を持たない。M1 は a を扱う記述のまま | 反映を要求する | 受け付ける。M1 は不整合にならない |
| TC-schema-requestSchemaApply-06 | 手元の列は M1、M2。手元の M1 は、適用済みの M1 と操作の並びが同じで、名前だけが異なる。スキーマの構造は M2 を適用した結果と一致する | 反映を要求する | 受け付ける。列の食い違いとして扱わない。`plan.migrations` は M2 だけを持つ |
| TC-schema-requestSchemaApply-07 | 実行者のトークンは `schema_apply` を持たない。スキーマは、同じモデルに同じキーの項目を二つ持つ | 反映を要求する | `ForbiddenError`（`GRANT_MISSING`）。必要な許可を示す。`SCHEMA_INVALID` を返さない。ワークスペースは `idle` のまま、イベントを記録しない |
| TC-schema-requestSchemaApply-09 | 入力の移行定義が、`MigrationOperation` に無い操作を含む | 反映を要求する | `ValidationError`（`INVALID_DEFINITION`）。移行定義の中の位置を示す。ワークスペースは `idle` のまま |
| TC-schema-requestSchemaApply-10 | ワークスペースは、別の反映 J0 で `applying`。スキーマは現行スキーマと同じで、未適用の移行定義は無い | 反映を要求する | `UnavailableError`（`SCHEMA_APPLYING`）。適用中の反映のジョブの ID J0 を添える。差分が無いこととして返さない。J を受け付けない。ワークスペースは J0 で `applying` のまま。イベントを記録しない |
| TC-schema-requestSchemaApply-11 | 要求が前提とする版は 2 | 反映を要求する | `ConflictError`（`SCHEMA_VERSION_CHANGED`）。現行の版 3 を添える。J を受け付けない。現行スキーマ、版、適用済みの列は変わらず、ワークスペースは `idle` のまま |
| TC-schema-requestSchemaApply-12 | スキーマが、同じモデルに同じキーの項目を二つ持つ | 反映を要求する | `BusinessRuleError`（`SCHEMA_INVALID`）。該当する定義と理由（`SchemaIssue`）を添える。J を受け付けない。ワークスペースは `idle` のまま |
| TC-schema-requestSchemaApply-13 | 手元の M1 の操作の並びが、適用済みの M1 と異なる | 反映を要求する | `BusinessRuleError`（`MIGRATION_SEQUENCE_DIVERGED`）。`changed` の食い違い（位置と M1）を添える。J を受け付けない。現行スキーマ、データは変わらない |
| TC-schema-requestSchemaApply-14 | 未適用の M2 が、その時点の構造に無い項目のキーを指す | 反映を要求する | `BusinessRuleError`（`MIGRATIONS_INVALID`）。該当する操作（`OperationRef`）、理由 `unknownKey`、その時点の構造に無い項目（`missing`）を添える。J を受け付けない |
| TC-schema-requestSchemaApply-15 | 直接反映によって、現行スキーマのコンテンツの構造が、適用済みの移行定義による構造と一致しない。手元の列は M1、M2 | 反映を要求する | `BusinessRuleError`（`SCHEMA_DRIFT`）。食い違う定義と、適用済みとして記録できる移行定義を添える。J を受け付けない |
| TC-schema-requestSchemaApply-16 | 直接反映によって、現行スキーマのコンテンツの構造が、適用済みの移行定義による構造と一致しない。手元の列は M1 で、未適用の移行定義は無い。スキーマは、現行スキーマから文字数の制約だけを変えたもの | 反映を要求する | 受け付ける。`SCHEMA_DRIFT` で拒否しない。`plan.migrations` は空 |
| TC-schema-requestSchemaApply-17 | 直接反映で食い違った後、直接反映で、適用済みの移行定義による構造と一致する構造に戻してある。手元の列は M1、M2。スキーマの構造は M2 を適用した結果と一致する | 現行の版を前提に、反映を要求する | 受け付ける。`preparing` の反映を返す |
| TC-schema-requestSchemaApply-18 | 手元の列は M1。スキーマは、移行定義に記録していない項目を加えている | 反映を要求する | `BusinessRuleError`（`SCHEMA_STRUCTURE_MISMATCH`）。食い違う定義（`StructureDifference`）を添える。J を受け付けない。ワークスペースは `idle` のまま |
| TC-schema-requestSchemaApply-19 | 反映 J を受け付け済みで、確定している | 同じ ID J と同じ内容で、反映を要求する | `succeeded` のジョブ J を返す。何も書かず、`schema.apply_requested` を新たに記録しない。ワークスペースは `idle` のまま |
| TC-schema-requestSchemaApply-21 | ID J の要求が `SCHEMA_STRUCTURE_MISMATCH` で拒否された後 | 同じ ID J で、移行定義を直した内容の反映を要求する | その時点の `WorkspaceSchema` で検証し直し、受け付ける。`preparing` の反映 J を返す。`SCHEMA_JOB_ID_CONFLICT` にしない |
| TC-schema-requestSchemaApply-23 | 事前確認のジョブを ID J で受け付け済み | 同じ ID J と同じスキーマ・移行定義で、反映を要求する | `ConflictError`（`SCHEMA_JOB_ID_CONFLICT`）。ジョブ J は変わらない |
| TC-schema-requestSchemaApply-25 | 手元の列とスキーマは受け付けられる内容 | 並行する二つの要求が、異なる ID J1、J2 で反映を要求する | 一方だけを受け付ける。もう一方は `ConflictError`（`STALE_VERSION`）になり、イベントも残らない。ワークスペースは、受け付けた反映で `applying` |
| TC-schema-requestSchemaApply-27 | 手元の列は M1 で、未適用の移行定義は無い。スキーマは現行スキーマと同じ | 反映を要求する | エラーにしない。差分が無いことと、現在の版 3 を返す。J を受け付けない。ワークスペースは `idle` のまま。何も書かない |
| TC-schema-requestSchemaApply-28 | 最後に終えた反映 J0 は失敗した反映で、ワークスペースは J0 を未解消の失敗した反映として持つ。手元の列は M1 で、未適用の移行定義は無い。スキーマは現行スキーマと同じ | 反映を要求する | エラーにしない。差分が無いことと、現在の版 3 を返す。J を受け付けない。ワークスペースは `idle` のまま、版は 3 のままで、未解消の失敗した反映を持たなくなる。`SchemaStatus.of` の `unresolved_failed_apply` の件数は 0 になる。ジョブ J0 は `failed` のまま変わらず、反映の結果の一覧に残る。`schema.apply_requested` を記録せず、版の記録を追記せず、操作履歴に残さない |
| TC-schema-requestSchemaApply-29 | 同上。手元の列は M1、M2 で、スキーマの構造は M2 を適用した結果と一致する | 反映を要求する | 受け付ける。ワークスペースは J で `applying` になり、J0 を未解消の失敗した反映として持ったまま。`SchemaStatus.of` の `unresolved_failed_apply` の件数は 1、`apply_in_progress` の件数は 1 |
| TC-schema-requestSchemaApply-30 | 同上。手元の列は M1 で、未適用の移行定義は無い。スキーマは現行スキーマと同じ | 反映を要求し、未解消の失敗した反映を外す保存の UnitOfWork が確定しない | エラーを返す。ワークスペースは J0 を未解消の失敗した反映として持ったまま。同じ内容で送り直すと、差分が無いことと現在の版 3 を返し、未解消の失敗した反映を持たなくなる |
| TC-schema-requestSchemaApply-33 | 手元の列は M1、M2。M2 は未適用。スキーマは現行スキーマと同じで、M2 を適用した構造と一致しない | 反映を要求する | `BusinessRuleError`（`SCHEMA_STRUCTURE_MISMATCH`）。J を受け付けない。差分が無いこととして返さない |
| TC-schema-requestSchemaApply-34 | 手元の列は M1、M2、M2（同じ名前の移行定義が二つある） | 反映を要求する | `BusinessRuleError`（`MIGRATIONS_INVALID`）。その名前と位置を示す `duplicateName` の不整合を添える。J を受け付けない |
