# Pagination

[../../domains/common.md](../../domains/common.md) の「Pagination と PaginationResult」の代表ケース。保存を伴わない値オブジェクトなので、ユースケースを通さずに確かめる。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-common-Pagination-01 | なし | `create({ page: 2, limit: 20 }, 100)` を呼ぶ | `page` が 2、`limit` が 20 の値を返す |
| TC-common-Pagination-02 | なし | `create({ page: 0, limit: 101 }, 100)` を呼ぶ | `ValidationError`（`InvalidPagination`）。項目ごとの理由を `page` と `limit` に持つ |
