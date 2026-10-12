# 反映の適用中かを確かめる（getSchemaApplyState）

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-schema-getSchemaApplyState-01 | ワークスペースは、`preparing` の反映 J で `applying`。実行者は `schema:change` を持たないメンバー | 確かめる | 対象の `WorkspaceId`、適用中であること、ジョブの ID J、反映の `JobProgress` を返す |
| TC-schema-getSchemaApplyState-02 | ワークスペースは `idle`。事前確認のジョブが `preparing`。実行者は権限を一つも持たないメンバー | 確かめる | 適用中でないことを返す |
| TC-schema-getSchemaApplyState-04 | ワークスペースは、準備済み（`prepared`）の反映 J で `applying`。コンテンツと履歴の版の総数は 40 | 確かめる | 適用中であること、ジョブの ID J、処理済みの件数と総数がどちらも 40 の `JobProgress` を返す |
| TC-schema-getSchemaApplyState-06 | ワークスペースを反映 J で `applying` と読んだ後、J のジョブを読む前に、J が終端になった。J のジョブが無い場合も同じ。実行者は権限を一つも持たないメンバー | 確かめる | 適用中でないことを返す |
