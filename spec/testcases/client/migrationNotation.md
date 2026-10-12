# 移行定義の記法（@repo/client/migration）

`packages/client` の `@repo/client/migration` の関数。契約は [app-api.md](../../usecases/app-api.md) の「移行定義の記法」、返す値の型は [schema.md](../../domains/schema.md) の「Migration」が定める。CMS へ接続しない。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-client-migrationNotation-01 | なし | `defineMigration` に、名前 m1 と、`addField`（階層は `level("article")`、項目は lead、任意のテキスト、補完値の指定なし）を渡す | `name` が m1 で、`operations` が、`op` が `addField`、`fill` が `null` の操作一つの `Migration` を返す |
| TC-client-migrationNotation-02 | なし | `level("article", "sections", blockType("hero"))` を呼ぶ | `model` が article で、`steps` が、`field`（sections）、`blockType`（hero）の順の `LevelPath` を返す |
| TC-client-migrationNotation-03 | なし | `removeField` を、`lossConfirmed: true` を渡さずに呼ぶ | 型の検査が受け付けない |
| TC-client-migrationNotation-04 | なし | `removeField` を、`lossConfirmed: true` を渡して呼ぶ | `op` が `removeField`、`lossConfirmed` が真の操作を返す |
| TC-client-migrationNotation-05 | m1、m2 は `defineMigration` の結果 | `defineMigrations([m1, m2])` を呼ぶ | m1、m2 の順の列を返す |
| TC-client-migrationNotation-06 | `generateMigration` が、`generated` の操作の並びを返している | その結果から出力した記法のコードを評価する | `operations` が、`generated` の操作の並びと等しい `Migration` になる |
