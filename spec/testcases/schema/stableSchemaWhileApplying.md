# 適用中のワークスペースへの `stableSchema` の書き込み

[../../domains/schema.md](../../domains/schema.md) の「WriteScope とスキーマ」の、`stableSchema` を渡す、実行者が `system` でないユースケースが、読んだ `WorkspaceSchema` が適用中（`applying`）のときに行うことの代表ケース。どのユースケースにも同じ規則で当たるので、ユースケースごとのファイルに置かない。断りが無い限り、ワークスペースは利用中で、`WorkspaceSchema` は反映 J で `applying`。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-schema-stableSchemaWhileApplying-01 | 実行者は `content:edit` を持つメンバー。コンテンツ C がある | C の下書きを保存する（Content の `saveDraft`） | `UnavailableError`（`SCHEMA_APPLYING`）。C の下書き、履歴の版、メディアの使用箇所は変わらない |
