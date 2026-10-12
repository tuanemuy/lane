# 適用状況を確認する（getMigrationStatus）

断りが無い限り、実行者はスキーマ反映の許可（`schema_apply`）を持つトークン。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-schema-getMigrationStatus-01 | 現行スキーマの版は 3。適用済みの移行定義は、適用した M1 と、適用済みとして記録した M2。ワークスペースは `idle`。現行の構造は、M1、M2 による構造と一致する | 手元の列 M1、M2、M3 で確認する | 対象の `WorkspaceId`、版 3、スキーマ全体・モデルごと・フォームごとの識別子、適用済みの M1（`applied`）と M2（`recorded`）を返す。`comparison` は `consistent` で、未適用は M3。`drift` は空。適用中でない。現行スキーマ、版、適用済みの列、データは変わらない |
| TC-schema-getMigrationStatus-02 | 適用済みの移行定義は M1 | 操作の並びが同じで名前だけが異なる M1 を持つ列で確認する | `comparison` は `consistent`。食い違いを示さない |
| TC-schema-getMigrationStatus-03 | 適用済みの移行定義は M1 | 操作の並びを変えた M1 を持つ列で確認する | エラーにしない。`MigrationStatus` は `compared`。`comparison` は `diverged` で、`changed`（M1）を持ち、`recordable` を持たない |
| TC-schema-getMigrationStatus-04 | 適用済みの移行定義は M1 | M1 を持たない列で確認する | `comparison` は `diverged` で、`missingLocally`（M1）を持つ |
| TC-schema-getMigrationStatus-05 | 適用済みの移行定義は M1 | M0、M1 の列で確認する（M0 は未適用） | `comparison` は `diverged` で、`insertedBefore`（M1 と、前に入った M0）を持つ |
| TC-schema-getMigrationStatus-06 | 適用済みの移行定義は M1。直接反映で、現行スキーマに項目を加えてある。M2 は、その項目を加える | M1、M2 の列で確認する | エラーにしない。`comparison` は `consistent`。`drift` は食い違う定義を持つ。`comparison` の `recordable` は M2 を持つ |
| TC-schema-getMigrationStatus-07 | スキーマを一度も反映していない | M1、M2 の列で確認する | エラーにしない。版 0 と、空の適用済みの列を返す。`comparison` は `consistent` で、未適用は M1、M2 |
| TC-schema-getMigrationStatus-08 | ワークスペースは、`preparing` の反映 J で `applying` | 確認する | 適用中であることと、適用中の反映のジョブ（ジョブの ID J、実行者と受け付けた時点の名称、要求の日時、`status`、進行）を返す。成功として返さない。版と適用済みの列は、反映前のもの |
| TC-schema-getMigrationStatus-13 | ワークスペースを反映 J で `applying` と読んだ後、J のジョブを読む前に、J が終端になった。J のジョブが無い場合も同じ | 確認する | 適用中でないとして返す。適用中の反映のジョブを返さない |
| TC-schema-getMigrationStatus-09 | 実行者のトークンは `schema_apply` を持たない | 確認する | `ForbiddenError`（`GRANT_MISSING`）。必要な許可を示す |
| TC-schema-getMigrationStatus-10 | 適用済みの移行定義は M1。手元の M2 が、その時点の構造に無い項目のキーを指す | M1、M2 の列で確認する | エラーにしない。`MigrationStatus` は `invalidMigrations` で、M2 のその操作、`unknownKey`、無い項目を指す `missing` を持つ。内容は、同じ列の「スキーマと移行定義の食い違いを検出する」の不整合と等しい。版、識別子、適用済みの M1、適用中かどうかは返す。未適用の移行定義、列の比較、記録できる移行定義は返さない |
