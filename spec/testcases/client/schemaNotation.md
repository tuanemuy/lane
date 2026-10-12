# スキーマの記法（@repo/client/schema）

`packages/client` の `@repo/client/schema` の関数。契約は [app-api.md](../../usecases/app-api.md) の「スキーマの記法」、返す値の型は [schema.md](../../domains/schema.md) が定める。CMS へ接続しない。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-client-schemaNotation-01 | なし | `defineSchema` に、`collection` で宣言したモデル article（`field.text` の項目 title）と、`single` で宣言したモデル site を渡す | `Schema` の `models` は、`key` が article で `cardinality` が `collection` の定義と、`key` が site で `cardinality` が `single` の定義を持つ。title の `type.kind` は `text` |
| TC-client-schemaNotation-02 | なし | プロパティを宣言せずに `field.text` で項目を宣言する | `required`・`unique`・`deprecated` は偽、`label`・`description`・`example`・`defaultValue` と制約は `null`、`input` は `singleLine` |
| TC-client-schemaNotation-03 | なし | `defaultValue` を宣言せずに `field.multiSelect` で項目を宣言する | `defaultValue` は空の組 |
| TC-client-schemaNotation-04 | なし | モデルの項目を、b、a の順のプロパティで宣言する | `fields` は b、a の順 |
| TC-client-schemaNotation-05 | なし | 項目に `deprecated: true` を宣言する | その `ContentField` の `deprecated` は真 |
| TC-client-schemaNotation-06 | なし | `previewPath("/articles/", uniqueValue("slug"))` を宣言する | `PreviewPath` は、`literal`（`/articles/`）、`uniqueValue`（`slug`）の順 |
| TC-client-schemaNotation-07 | なし | モデルのキーを `Article` にして宣言する | `ValidationError`（`INVALID_KEY`） |
| TC-client-schemaNotation-08 | なし | 参照の項目の `models` に、スキーマに無いモデルのキーを宣言する | 記法は受け付け、`Schema` を返す。`SchemaValidation.validate` が、`unknownModel` の不整合を返す |
| TC-client-schemaNotation-09 | なし | `field.boolean` に、文字数の制約を宣言する | 型の検査が受け付けない |
