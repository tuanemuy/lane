# 受け付けた反映・事前確認の進行と結果を取得する（getSchemaApplyJob）

断りが無い限り、実行者はスキーマ反映の許可（`schema_apply`）を持つトークンで、J は実行者のワークスペースで受け付けたジョブの ID。「取得する」は、J を入力に呼ぶことを指す。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-schema-getSchemaApplyJob-01 | ワークスペースは、`preparing` の反映 J で `applying`。変換を一部終えている | 取得する | 対象の `WorkspaceId` と、`preparing` のジョブ（`JobProgress` は変換を終えた件数と対象の総件数）を返す。成功として返さない |
| TC-schema-getSchemaApplyJob-02 | ワークスペースは、準備済み（`prepared`）の反映 J で `applying`。コンテンツと履歴の版の総数は 40 | 取得する | `prepared` のジョブ（処理済みの件数と総数がどちらも 40 の `JobProgress` と、`effects`）を返す。成功として返さない |
| TC-schema-getSchemaApplyJob-03 | 反映 J は確定済み | 取得する | `succeeded` のジョブと `ApplySuccess`（反映後の版と識別子、適用した移行定義、`ApplyEffects`）を返す |
| TC-schema-getSchemaApplyJob-04 | 反映 J は `blocked` で失敗して終わっている | 取得する | `failed` のジョブと、失敗の理由（`blocked`、該当するコンテンツと項目、`ApplyEffects`）を返す |
| TC-schema-getSchemaApplyJob-05 | 反映 J は `commitRejected`（`stagingMismatch`）で失敗して終わっている | 取得する | `failed` のジョブと、失敗の理由（`commitRejected`、`cause` は `stagingMismatch`、`ApplyEffects`）を返す |
| TC-schema-getSchemaApplyJob-06 | 反映 J は `interrupted` で失敗して終わっている。その後に、別の反映が確定している | 取得する | `failed` のジョブと、失敗の理由 `interrupted` を返す。成功として返さない |
| TC-schema-getSchemaApplyJob-07 | 事前確認のジョブ J は `checked`。失敗する公開内容がある。値を失う操作は無い | 取得する | 状態 `checked`、確認の時点の現行スキーマの版、`blockers`（該当するコンテンツと項目、件数）、`ApplyEffects` を返す。案内は空 |
| TC-schema-getSchemaApplyJob-08 | 事前確認のジョブ J は `checked`。`effects.losses` は、値を 3 件失う操作を持つ | 取得する | 状態 `checked` と `ApplyEffects` に、案内 `irreversibleLoss` を添えて返す |
| TC-schema-getSchemaApplyJob-09 | 事前確認のジョブ J は `failed`（`schemaChanged`） | 取得する | 状態 `failed` と、失敗の理由 `schemaChanged` を返す |
| TC-schema-getSchemaApplyJob-11 | ジョブ J は、ほかのワークスペースにだけある | 取得する | `NotFoundError`（対象の種類は `schema_apply_job`） |
| TC-schema-getSchemaApplyJob-14 | 実行者のメンバーは `schema:change` を持たない | 取得する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限を添える |
