# スキーマの版の記録を一覧する（listSchemaVersionRecords）

断りが無い限り、実行者はスキーマ反映の許可（`schema_apply`）を持つトークン。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-schema-listSchemaVersionRecords-01 | 版 1 は項目を加える直接反映、版 2 は項目を削除する直接反映、版 3 は適用済みとしての記録（M1）、版 4 は移行定義の適用（M2。データを失い得る操作を含む）、版 5 は移行定義を必要としない定義だけの反映。どの版も、メンバー A が発行したトークン T が要求した | 一覧する | 対象の `WorkspaceId`、版 5、4、3、2、1 の順の記録、総件数 5 を返す。版ごとに、日時と、実行者（記録の時点の T の表示用の名称と A の名前）を持つ。版 5 は `reflected`、版 4 は `applied` で M2 とデータを失い得る操作、版 3 は `recorded` で M1、版 2 は `direct` で失った項目を表すデータを失い得る操作、版 1 は `direct` で空のデータを失い得る操作を持つ。トークンの値を含まない |
| TC-schema-listSchemaVersionRecords-03 | スキーマを一度も反映していない | 一覧する | エラーにしない。空の一覧と総件数 0 を返す |
| TC-schema-listSchemaVersionRecords-04 | 版 1 の反映の後に、T の発行者 A の名前が変わり、A は利用を解除され、T は失効して一覧から外れている | 一覧する | エラーにしない。版 1 の記録は、発行者の `MemberId` と、記録の時点の T の表示用の名称と A の名前を持つ |
| TC-schema-listSchemaVersionRecords-06 | 実行者のメンバーは `schema:change` を持たない | 一覧する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限を添える |
