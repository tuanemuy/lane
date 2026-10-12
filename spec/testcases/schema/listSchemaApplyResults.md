# 反映の結果を一覧する（listSchemaApplyResults）

断りが無い限り、実行者はスキーマ反映の許可（`schema_apply`）を持つトークン。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-schema-listSchemaApplyResults-01 | 終えた順に、確定した反映、`blocked`（一意性の違反）で失敗した反映、確定した直接反映がある。`checked` の事前確認のジョブもある | 一覧する | 対象の `WorkspaceId`、三件の反映のジョブを、終えた日時の新しいものから、総件数 3 を返す。事前確認のジョブを含まない。ジョブごとに、ジョブの ID、操作の求め方、実行者と受け付けた時点の名称（トークンの表示用の名称と発行者の名前）、要求と終了の日時、成功か失敗かを持つ。`succeeded` は `ApplySuccess`、`failed` は `blocked` と、該当するコンテンツと項目を持つ。保存方式に依存する情報とトークンの値を含まない |
| TC-schema-listSchemaApplyResults-03 | 確定した反映が一件ある。その後に、スキーマの不整合で拒否された要求と、差分が無いことを返した要求がある | 一覧する | `succeeded` の一件だけと、総件数 1 を返す。拒否された要求と差分の無い要求を含まない |
| TC-schema-listSchemaApplyResults-04 | 確定した反映が一件ある。ワークスペースは、別の反映 J で `applying` | 一覧する | `succeeded` の一件だけと、総件数 1 を返す。適用中の反映 J を含まない |
| TC-schema-listSchemaApplyResults-06 | 反映を一度も要求していない | 一覧する | 空の一覧と総件数 0 を返す |
| TC-schema-listSchemaApplyResults-08 | 実行者のメンバーは `schema:change` を持たない | 一覧する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限を添える |
| TC-schema-listSchemaApplyResults-09 | 反映を要求したトークンの発行者 A は、その後に利用を解除されている | 一覧する | そのジョブは、受け付けた時点のトークンの表示用の名称と A の名前を持つ |
