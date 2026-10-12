# 直接反映を要求する（requestDirectSchemaApply）

断りが無い限り、実行者はスキーマ反映の許可（`schema_apply`）を持つトークン、ワークスペースは `idle` で、未解消の失敗した反映（`failedApply`）を持たない。要求は現行スキーマの版と新しい `SchemaApplyJobId` J を添える。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-schema-requestDirectSchemaApply-01 | スキーマを一度も反映していない（版 0） | 前提とする版 0 で、モデルと項目を持つスキーマの直接反映を要求する | 受け付けたジョブ（ID は J、`mode` は `commit`、`status` は `preparing`）と、対象の `WorkspaceId` を返す。`plan.source` は `direct` で、`plan.operations` は、現行の構造との差分から求めた追加の操作を持つ。ワークスペースは J で `applying` になる。`schema.apply_requested`（`jobId` J）を記録する |
| TC-schema-requestDirectSchemaApply-02 | 現行スキーマはモデル article の項目 body を持つ | body を削除したスキーマを、データを失うことを確認せずに（`lossAcknowledged` は `false`）要求する | `BusinessRuleError`（`SCHEMA_LOSS_NOT_ACKNOWLEDGED`）。失う項目を表す `LossyOperation`（`removeField`、`loss` は `field`。移行定義の名前を持たない）を添える。J を受け付けない。ワークスペースは `idle` のまま、現行スキーマとデータは変わらず、イベントを記録しない |
| TC-schema-requestDirectSchemaApply-03 | 同上 | 同じスキーマを、データを失うことを確認して要求する | 受け付ける。`plan.lossyOperations` はその `removeField` を持つ |
| TC-schema-requestDirectSchemaApply-04 | 現行スキーマは項目 title を持つ | title のキーを heading に変えたスキーマを、確認して要求する | 受け付ける。`plan.operations` は、heading の `addField` と title の `removeField` を持ち、`renameField` を持たない |
| TC-schema-requestDirectSchemaApply-05 | 現行スキーマは数値の項目 price を持つ | price をテキストに変えたスキーマを、確認して要求する | 受け付ける。`plan.operations` は price の `changeFieldKind` を持つ |
| TC-schema-requestDirectSchemaApply-06 | 適用済みの移行定義がある | コンテンツの構造を変えるスキーマを、確認して要求する | 受け付ける。移行定義との一致を確かめない |
| TC-schema-requestDirectSchemaApply-07 | スキーマが、存在しないモデルを参照先に指定している | 直接反映を要求する | `BusinessRuleError`（`SCHEMA_INVALID`）。該当する定義と理由（`unknownModel`）を添える。J を受け付けない。現行スキーマ、データ、運用設定は変わらない |
| TC-schema-requestDirectSchemaApply-08 | ワークスペースは、別の反映 J0 で `applying`。スキーマは現行スキーマと同じ | 直接反映を要求する | `UnavailableError`（`SCHEMA_APPLYING`）。差分が無いこととして返さない。J を受け付けない。ワークスペースは J0 で `applying` のまま |
| TC-schema-requestDirectSchemaApply-09 | 現行スキーマの版は 3。版 3 を前提にした、データを失うことを確認しない要求が `SCHEMA_LOSS_NOT_ACKNOWLEDGED` で拒否された後、別の反映が確定して版が 4 になった | 同じスキーマを、データを失うことを確認して、前提とする版 3 のまま、新しい ID で要求する | `ConflictError`（`SCHEMA_VERSION_CHANGED`）。現行の版 4 を添える。受け付けない。現行スキーマは変わらない |
| TC-schema-requestDirectSchemaApply-10 | 実行者のトークンは `schema_apply` を持たない | 直接反映を要求する | `ForbiddenError`（`GRANT_MISSING`）。必要な許可を示す。ワークスペースは `idle` のまま |
| TC-schema-requestDirectSchemaApply-11 | 反映 J を受け付け済み | 同じ ID J と同じ内容で要求する | ジョブ J を、現在の状態のまま返す。何も書かず、イベントを新たに記録しない |
| TC-schema-requestDirectSchemaApply-12 | 反映 J を受け付け済み | 同じ ID J で、`lossAcknowledged` が異なる要求を送る | `ConflictError`（`SCHEMA_JOB_ID_CONFLICT`） |
| TC-schema-requestDirectSchemaApply-13 | スキーマは受け付けられる内容 | 並行する二つの要求が、異なる ID で直接反映を要求する | 一方だけを受け付ける。もう一方は `ConflictError`（`STALE_VERSION`）になり、イベントも残らない |
| TC-schema-requestDirectSchemaApply-14 | 現行スキーマの版は 3 | 現行スキーマと同じスキーマの直接反映を要求する | エラーにしない。差分が無いことと、現在の版 3 を返す。J を受け付けない。ワークスペースは `idle` のまま。何も書かない |
| TC-schema-requestDirectSchemaApply-15 | 最後に終えた反映 J0 は失敗した反映で、ワークスペースは J0 を未解消の失敗した反映として持つ。現行スキーマの版は 3。適用済みの移行定義がある | 現行スキーマと同じスキーマの直接反映を要求する | エラーにしない。差分が無いことと、現在の版 3 を返す。ワークスペースは、未解消の失敗した反映を持たなくなる。版は 3 のまま |
| TC-schema-requestDirectSchemaApply-16 | ID J の要求が、差分が無いことを返した後 | 同じ ID J と同じ内容で、直接反映を要求する | 再び、差分が無いことと現在の版 3 を返す。何も書かない |
| TC-schema-requestDirectSchemaApply-17 | 現行スキーマはフォーム contact と inquiry を持つ。contact は項目 mail を持つ | inquiry を削除し、contact の項目 mail のキーを email に変え、コンテンツの構造を変えないスキーマを、確認せずに（`lossAcknowledged` は `false`）要求する | `BusinessRuleError`（`SCHEMA_LOSS_NOT_ACKNOWLEDGED`）。`forms` は、inquiry を `removed` に、contact の `removedFields` に mail、`addedFields` に email を持つ。`lossyOperations` は空。J を受け付けない。ワークスペースは `idle` のまま、現行スキーマ、データ、運用設定は変わらない |
| TC-schema-requestDirectSchemaApply-18 | 現行スキーマのフォーム contact は、`email` が真のテキストの項目 reply を持つ。contact の通知定義は reply を返信先に使う | reply の `email` を偽にしたスキーマを、確認せずに要求する | `BusinessRuleError`（`SCHEMA_LOSS_NOT_ACKNOWLEDGED`）。`brokenNotificationReferences` は、contact と reply、理由 `kindUnsuitable` を持つ。`lossyOperations` は空で、`forms` は削除したフォームと項目を持たない。J を受け付けない |
| TC-schema-requestDirectSchemaApply-19 | 同上 | 同じスキーマを、確認して要求する | 受け付ける。運用設定の通知定義は変わらない |
