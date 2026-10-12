# スキーマと移行定義の食い違いを検出する（checkMigrations）

CMS に接続せず、認可を持たない。出力は、食い違う定義、移行定義の不整合、データを失い得る操作の三つ。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-schema-checkMigrations-01 | 移行定義の列を先頭から順に適用した構造が、スキーマのコンテンツの構造と一致する。データを失い得る操作は無い | 検出する | 三つとも空を返す |
| TC-schema-checkMigrations-02 | 同じく一致する。M2 は項目の削除と、置き換え先を未設定にする選択肢の削除を含む | 検出する | 食い違いと不整合は空。データを失い得る操作は、M2 の名前とともに、`removeField`（`loss` は `field`）と `removeOption`（`loss` は `optionValues`）を、列の順に持つ |
| TC-schema-checkMigrations-03 | スキーマは、移行定義に記録していない項目を持つ | 検出する | エラーにしない。食い違う定義は、その項目を指す `unexpected` を持つ。不整合は空 |
| TC-schema-checkMigrations-04 | 移行定義は、スキーマに無い項目を加えている | 検出する | 食い違う定義は、その項目を指す `missing` を持つ |
| TC-schema-checkMigrations-05 | M1 は項目 a を加え、M2 は a を削除する。スキーマは a を持たない | 検出する | 食い違いも不整合も空 |
| TC-schema-checkMigrations-06 | M2 が、その時点の構造に無い項目のキーを指す | 検出する | 不整合は、M2 のその操作（`OperationRef`）、`unknownKey`、その階層のその項目を指す `missing` を持つ。食い違う定義は空（求めない） |
| TC-schema-checkMigrations-07 | M2 が、リッチテキストの項目から数値の項目への値の複写を含む | 検出する | 不整合は、その操作、`noConversionRule`、複写元の値の種類（`from` はリッチテキスト）と複写先の値の種類（`to` は数値）を持つ |
| TC-schema-checkMigrations-08 | M2 が、モデル直下の階層を指して、繰り返しの中にだけある項目を複写元にする値の複写を含む | 検出する | 不整合は、その操作、`unknownKey`、モデル直下の階層の複写元の項目を指す `missing` を持つ |
| TC-schema-checkMigrations-10 | M2 が、選択肢 x、y を持つ必須の単一選択の項目を、補完値 z で加える | 検出する | 不整合は、その操作と `fillMismatch` を持つ |
| TC-schema-checkMigrations-11 | M2 が、単一選択の項目を、別の選択肢を持つ単一選択に変える値の種類の変更を含む | 検出する | 不整合は、その操作と `notApplicable` を持つ |
| TC-schema-checkMigrations-12 | 入力の移行定義が、`MigrationOperation` に無い操作、または関数の記述を含む | 検出する | `ValidationError`（`INVALID_DEFINITION`）。該当する操作の位置を示す |
| TC-schema-checkMigrations-13 | 列に、同じ名前の移行定義が二つある | 検出する | エラーにしない。不整合は、その名前と位置を示す `duplicateName` を持つ。食い違う定義は空（求めない） |
