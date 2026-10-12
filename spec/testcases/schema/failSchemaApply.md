# 反映・事前確認を失敗させる（failSchemaApply）

断りが無い限り、ワークスペースは利用中。「反映 J」は、`mode` が `commit` の進行中のジョブ J で、`WorkspaceSchema` は J で `applying`。「失敗させる」は、J を入力に呼ぶことを指す。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-schema-failSchemaApply-01 | 反映 J は `preparing`。Content と Media の準備領域に、変換結果と使用箇所がある | `capacityExceeded` で失敗させる | `failed`（`capacityExceeded`）のジョブを返す。`preparation` は準備の途中のままで、失敗の理由は `effects` を持たない。ジョブは、J の実行者、名称、要求の日時を保つ。ワークスペースは `idle` に戻り、J を未解消の失敗した反映（`failedApply`）として持つ。`SchemaStatus.of` の `unresolved_failed_apply` の件数は 1、`apply_in_progress` の件数は 0 になる。現行スキーマ、版、適用済みの列は変わらない。準備領域は残らない。コンテンツ、履歴の版、メディアの使用箇所、運用設定は変わらない。版の記録と操作履歴を作らず、イベントを記録しない |
| TC-schema-failSchemaApply-02 | 反映 J は準備済み（`prepared`） | `commitRejected`（`stagingMismatch`）で失敗させる | `failed`（`commitRejected`、`stagingMismatch`）のジョブを返す。失敗の理由の `effects` は、準備済みの反映のものを引き継ぐ。ワークスペースは `idle` に戻り、準備領域は残らない。現行スキーマとデータは変わらない |
| TC-schema-failSchemaApply-03 | 反映 J は準備済み（`prepared`） | `capacityExceeded` で失敗させる | `failed`（`capacityExceeded`）のジョブを返す。`preparation` は準備を終えたもので、失敗の理由は準備済みの反映の `effects` を引き継ぐ。ワークスペースは `idle` に戻り、準備領域は残らない |
| TC-schema-failSchemaApply-04 | 事前確認のジョブ D は `preparing`。ワークスペースは、別の反映 J0 で `applying` | D を `capacityExceeded` で失敗させる | `failed`（`capacityExceeded`）のジョブを返す。D の準備領域は残らない。ワークスペースは J0 で `applying` のまま。`WorkspaceSchema` は変わらない |
| TC-schema-failSchemaApply-05 | 反映 J を失敗させた後 | 同じワークスペースへ、現行の版を前提に、新しい ID で反映を要求する | 受け付ける。ワークスペースは、新しい反映で `applying` になり、J を未解消の失敗した反映として持ったまま |
| TC-schema-failSchemaApply-06 | 反映 J は確定済み（`succeeded`） | `capacityExceeded` で失敗させる | 何も書かず、`succeeded` のジョブを返す |
| TC-schema-failSchemaApply-08 | 反映 J は準備済み（`prepared`） | 確定と、`capacityExceeded` での失敗を並行して行う | 一方だけが確定する。ジョブ J は、`succeeded` または `failed` のどちらかになり、競合した UnitOfWork は `ConflictError`（`STALE_VERSION`）で何も反映しない。`succeeded` なら版は 1 進み、`failed` なら現行スキーマとデータは変わらない |
