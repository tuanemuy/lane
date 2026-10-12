# スキーマを検証する（validateSchema）

CMS に接続せず、認可を持たない。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-schema-validateSchema-01 | モデル、フォーム、派生形式を持つ、不整合の無いスキーマ | 検証する | 空の列を返す |
| TC-schema-validateSchema-02 | 同じモデルに、同じキーの項目が二つある | 検証する | エラーにしない。その項目を指す `duplicateKey` の不整合（分類は `duplicateKey`）を返す。案内は空 |
| TC-schema-validateSchema-03 | コンテンツ参照の項目が、スキーマに無いモデルを許可している | 検証する | その項目を指す `unknownModel` の不整合（分類は `unknownReference`）を返す |
| TC-schema-validateSchema-04 | モデルの一覧表示用の項目が、そのモデルに無い項目のキーを指す | 検証する | `unknownField` の不整合（分類は `unknownReference`）を返す |
| TC-schema-validateSchema-05 | テキストの項目の最小文字数が、最大文字数より大きい | 検証する | その項目を指す `invalidRange` の不整合（分類は `inapplicableConstraint`）を返す |
| TC-schema-validateSchema-06 | 単一選択の項目の初期値が、選択肢に無いキー | 検証する | その項目を指す `defaultNotInOptions` の不整合（分類は `defaultMismatch`）を返す |
| TC-schema-validateSchema-07 | モデルの一覧表示用の項目が、リッチテキストの項目を指す | 検証する | `listFieldKind` の不整合（分類は `displayFieldMismatch`）を返す |
| TC-schema-validateSchema-08 | モデルのプレビュー先の経路の `literal` が `?` を含む | 検証する | そのモデルを指す `previewPathShape` の不整合（分類は `displayFieldMismatch`）を返す |
| TC-schema-validateSchema-09 | 必須の項目に、非推奨が付いている | 検証する | その項目を指す `deprecatedRequired` の不整合を、案内 `optionalizeWithDeprecation` とともに返す |
| TC-schema-validateSchema-10 | キーの重複と、必須の項目への非推奨の指定が、別々の定義にある | 検証する | 二つの不整合を、どちらも返す |
| TC-schema-validateSchema-11 | 入力のスキーマが、値の種類に無い制約（真偽値の項目に文字数）を持つ | 検証する | `ValidationError`（`INVALID_DEFINITION`）。スキーマの中の位置を示す |
