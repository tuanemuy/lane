# 現行スキーマを取得する（getCurrentSchema）

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-schema-getCurrentSchema-01 | 現行スキーマの版は 3。実行者はスキーマ反映の許可（`schema_apply`）を持つトークン | 取得する | 対象の `WorkspaceId`、現行スキーマ、版 3、スキーマ全体・モデルごと・フォームごとの識別子を返す |
| TC-schema-getCurrentSchema-03 | スキーマを一度も反映していない | 取得する | エラーにしない。版 0 と、モデル・フォーム・派生形式が空のスキーマを返す |
| TC-schema-getCurrentSchema-04 | ワークスペースは、スキーマを変える反映で `applying`。現行スキーマの版は 3 | 取得する | 反映前のスキーマと版 3 を返す |
| TC-schema-getCurrentSchema-06 | 実行者のメンバーは `schema:change` を持たない | 取得する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限を添える |
