# SchemaVersionRecordRepository

契約は [schema.md](../../domains/schema.md) の「ポート」と、[common.md](../../domains/common.md) の「TransactionalRepository」「Pagination と PaginationResult」「WriteScope」「UnitOfWorkProvider」が定める。

## 記法

- W、W1、W2 は、互いに違うワークスペース。リポジトリは、ワークスペースの区画ごとにあり、メソッドはワークスペースの ID を取らない。断りの無い呼び出しは、W の区画のリポジトリに対して行う。「W1 の `findPage`」のように書いたワークスペースは、呼ぶリポジトリが結び付く区画を指す。記録は `SchemaVersionRecord` の振る舞いで作る。記録は、ワークスペースの ID を持たず、追記したリポジトリの区画に属する。「版 n の記録」は、`schemaVersion` が n の記録
- 前提条件の書き込みは、そのワークスペースの UnitOfWork（`UnitOfWorkProvider.run` に `{ kind: "workspace", workspaceId, condition }` を渡した UnitOfWork。[unitOfWorkProvider.md](unitOfWorkProvider.md)）の `WorkspaceContext` のリポジトリで呼び、確定済み。操作の書き込みは、新しい UnitOfWork で呼ぶ。`findPage` は UnitOfWork の外で、`PartitionReader.workspace` のリポジトリで呼ぶ。UnitOfWork の条件は `none`
- 「`ConflictError`（理由）になる」は、メソッドの呼び出しか確定のどちらかでエラーが現れ、`run` がそのエラーを投げ、その UnitOfWork の書き込みが一つも反映されないことを指す
- 引数を書かない `findPage` の `pagination` は `{ page: 1, limit: 10 }`。「上限」は、`findPage` では 1 ページの件数 10、`deleteAll` では `BoundedDeletion.maxLimit`

## append

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-schemaVersionRecordRepository-01 | W の記録が無い | `change.kind` が `applied` の、版 1 の記録を `append` し、確定の後に `findPage` を呼ぶ | `items` は、追記した値と全フィールドが等しい 1 件。`migrations` と `lossyOperations` の順序を保つ。`count` は 1 |
| TC-ports-schemaVersionRecordRepository-02 | W の記録が無い | `change.kind` が `reflected`、`direct`、`recorded` の記録を、別の版で `append` し、`findPage` を呼ぶ | それぞれ、追記した値と全フィールドが等しい記録を返す。`reflected` の記録は、`migrations` も `lossyOperations` も持たない |
| TC-ports-schemaVersionRecordRepository-03 | W の記録が無い | `actor` が `appToken` の記録と `memberToken` の記録を、別の版で `append` し、`findPage` を呼ぶ | それぞれ、追記した `actor` と `actorNames` を保つ。`appToken` は `tokenId` と `issuerMemberId` を、`memberToken` は `tokenId` と `memberId` を保つ |
| TC-ports-schemaVersionRecordRepository-04 | W の版 1 の記録がある | W の版 1 の、内容が異なる記録を `append` する | `ConflictError`（`ALREADY_EXISTS`）になる。`findPage` は、先に追記した記録だけを返す |
| TC-ports-schemaVersionRecordRepository-05 | W1 の版 1 の記録がある | W2 の版 1 の記録を、W2 のリポジトリで `append` する | 確定する。W1 の `findPage` と W2 の `findPage` は、それぞれの記録を返す |
| TC-ports-schemaVersionRecordRepository-06 | W の記録が無い | 並行する二つの UnitOfWork が、それぞれ W の版 1 の記録を `append` する | 一方だけが確定し、もう一方は `ConflictError`（`ALREADY_EXISTS`）になる。`findPage` は、確定した側の 1 件を返す |
| TC-ports-schemaVersionRecordRepository-07 | W の版 1 の記録を `append` し、`deleteAll` で消去済み | W の版 1 の記録を `append` する | 確定する。`findPage` は新しい記録を返す |

## findPage

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-schemaVersionRecordRepository-08 | W の記録が無い | `findPage({ page: 1, limit: 10 })` を呼ぶ | `items` は空、`count` は 0 |
| TC-ports-schemaVersionRecordRepository-09 | W の `WorkspaceSchema` も記録も無い | `findPage({ page: 1, limit: 10 })` を呼ぶ | `items` は空、`count` は 0 |
| TC-ports-schemaVersionRecordRepository-11 | W の版 1 から 5 の記録を、3、1、5、2、4 の順に `append` 済み | `findPage({ page: 1, limit: 10 })` を呼ぶ | `items` は版が 5、4、3、2、1 の順、`count` は 5 |
| TC-ports-schemaVersionRecordRepository-12 | W の版 1 から 5 の記録がある | `findPage({ page: 1, limit: 2 })`、`page: 2`、`page: 3` を順に呼ぶ | `items` は順に、5 と 4、3 と 2、1。`count` はどれも 5 |
| TC-ports-schemaVersionRecordRepository-13 | W の版 1 から 5 の記録がある | `findPage({ page: 4, limit: 2 })` を呼ぶ | `items` は空、`count` は 5 |
| TC-ports-schemaVersionRecordRepository-15 | W の記録が、上限より 1 件多くある | `limit` を上限にして、`page` が 1 と 2 を呼ぶ | 1 ページ目は、版の降順で先頭から上限の件数、2 ページ目は版 1 の 1 件。`count` はどちらも、その件数 |
| TC-ports-schemaVersionRecordRepository-17 | W1 の記録が 2 件、W2 の記録が 3 件ある | W1 の `findPage({ page: 1, limit: 10 })` を呼ぶ | `items` は W1 の 2 件だけ、`count` は 2 |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-schemaVersionRecordRepository-18 | W の記録が無い | `deleteAll(10)` を呼ぶ | 0 を返す |
| TC-ports-schemaVersionRecordRepository-19 | W の記録が 3 件ある | `deleteAll(10)` を呼び、確定の後に `findPage` を呼ぶ | `limit` より少ない 3 を返す。`items` は空、`count` は 0 |
| TC-ports-schemaVersionRecordRepository-21 | W の記録が 5 件ある | `deleteAll(2)` を、呼び出しごとに別の UnitOfWork で、返す件数が 2 より少なくなるまで繰り返す | 返す値は順に 2、2、1。1 回目の確定の後の `count` は 3、最後は 0 |
| TC-ports-schemaVersionRecordRepository-22 | W1 の記録が 2 件、W2 の記録が 3 件ある | W1 の `deleteAll(10)` を呼び、確定の後に W2 の `findPage` を呼ぶ | 2 を返す。W2 の `count` は 3 |
| TC-ports-schemaVersionRecordRepository-24 | W の記録が 3 件ある | `limit` を上限にして `deleteAll` を呼ぶ | 3 を返す |
| TC-ports-schemaVersionRecordRepository-25 | W の記録が 3 件ある | `deleteAll(0)` を呼ぶ | `SystemError`（`INVALID_QUERY`）を投げる。`findPage` の `count` は 3 |
| TC-ports-schemaVersionRecordRepository-26 | W の記録が 3 件ある | `limit` を、上限より 1 大きい値にして `deleteAll` を呼ぶ | `SystemError`（`INVALID_QUERY`）を投げる。`findPage` の `count` は 3 |

## UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-schemaVersionRecordRepository-27 | W の `WorkspaceSchema` が `WorkspaceSchemaRepository` にある。W の記録が無い | 一つの UnitOfWork で、`WorkspaceSchemaRepository.save` と `append` を呼び、正常に終える | `WorkspaceSchemaRepository.find` は保存した値を、`findPage` は追記した記録を返す |
| TC-ports-schemaVersionRecordRepository-28 | 同上 | 一つの UnitOfWork で、`WorkspaceSchemaRepository.save` と `append` を呼んだ後、`fn` が例外を投げる | `run` はその例外を投げる。`findPage` の `count` は 0。`WorkspaceSchemaRepository.find` は保存の前の値を返す |
| TC-ports-schemaVersionRecordRepository-29 | W の `WorkspaceSchema` と、W の版 1 の記録がある | 一つの UnitOfWork で、`WorkspaceSchemaRepository.save` と、W の版 1 の記録の `append` を呼ぶ | `ConflictError`（`ALREADY_EXISTS`）になる。`WorkspaceSchemaRepository.find` は保存の前の値を返す |
| TC-ports-schemaVersionRecordRepository-30 | W の `WorkspaceSchema` がある。その `expectedVersion` を得た後、別の UnitOfWork が `WorkspaceSchemaRepository.save` を確定した | 一つの UnitOfWork で、古い `expectedVersion` での `WorkspaceSchemaRepository.save` と、`append` を呼ぶ | `ConflictError`（`STALE_VERSION`）になる。`findPage` の `count` は 0 |
| TC-ports-schemaVersionRecordRepository-31 | W の記録が 3 件ある | `deleteAll(10)` を呼んだ後、`fn` が例外を投げる | `findPage` の `count` は 3 |
