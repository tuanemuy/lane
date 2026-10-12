# EmailAddress

[../../domains/common.md](../../domains/common.md) の「EmailAddress」の代表ケース。保存を伴わない値オブジェクトなので、ユースケースを通さずに確かめる。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-common-EmailAddress-01 | なし | `create(" Taro@Example.COM ")` を呼ぶ | `"taro@example.com"` を保持する値を返す。`create("taro@example.com")` の値と等しい |
| TC-common-EmailAddress-03 | なし | `@` の無い `"example.com"` で `isWellFormed` と `create` を呼ぶ | `isWellFormed` は偽を返す。`create` は `ValidationError`（`InvalidEmailAddress`） |
