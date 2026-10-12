# ContentRetentionPolicyRepository

「上限」は、その節のメソッドの契約が定める、一度に渡せる件数・個数、または `limit` の上限を指す。値は、ポートを定めるドメインのファイルが持つ。

契約は [content.md](../../domains/content.md) の「ContentRetentionPolicy」「ポート」と、[common.md](../../domains/common.md) の「SingletonRepository」・「TransactionalRepository」の共通の規則・「UnitOfWorkProvider」が定める。

## 記法

- リポジトリは、ワークスペース `W1` の区画に結び付く。メソッドは、ワークスペースの ID を引数に取らない。`W0` は、区画に何も確定していないワークスペース
- `P` は、`ContentRetentionPolicy.create` で作った `W1` の `ContentRetentionPolicy`。「`P` がある」は、`P` の `insert` を確定した状態
- UnitOfWork は、`{ kind: "workspace", workspaceId: W1, condition: { kind: "none" } }` を渡した `UnitOfWorkProvider.run` を指す。書き込みは、その `WorkspaceContext` のリポジトリで呼ぶ。UnitOfWork の外の読み取りは、`PartitionReader.workspace(W1)` のリポジトリで呼ぶ
- 前提条件の書き込みは、UnitOfWork で呼び、確定済み。操作の書き込みは、新しい UnitOfWork で呼び、UnitOfWork を確定する。読み取りは、UnitOfWork の外で呼ぶ
- 「`ConflictError` になる」は、メソッドの呼び出しか UnitOfWork の確定のどちらかでエラーが現れ、`run` がそのエラーを投げ、その UnitOfWork の書き込みが 1 件も反映されないことを指す
- 「`SystemError`（`INVALID_QUERY`）になる」は、`run` がそのエラーを投げ、その UnitOfWork の書き込みが 1 件も反映されないことを指す

## insert と find

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRetentionPolicyRepository-01 | `ContentRetentionPolicy` が無い | `find()` を呼ぶ | `null` を返す |
| TC-ports-contentRetentionPolicyRepository-02 | `W1` に `P` がある | `PartitionReader.workspace(W0)` のリポジトリで `find()` を呼ぶ | `null` を返す |
| TC-ports-contentRetentionPolicyRepository-03 | `ContentRetentionPolicy` が無い | `P` を `insert` する | `find()` が、`revisionDays`、`trashDays`、`version`、`updatedAt` が渡した `P` と等しい値と、`expectedVersion` を返す |
| TC-ports-contentRetentionPolicyRepository-05 | `P` がある | 日数が `P` と異なる、`W1` の `ContentRetentionPolicy` を `insert` する | `ConflictError`（`ALREADY_EXISTS`）になる。`find()` は前提条件の `P` を返す |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRetentionPolicyRepository-06 | `P` がある | `find()` の `expectedVersion` で、`change`（`revisionDays` 10、`trashDays` 20）した `P` を `save` する | `find()` が、`revisionDays` が 10、`trashDays` が 20、`version` が 1 で、`updatedAt` が渡した値と等しい `P` を返す |
| TC-ports-contentRetentionPolicyRepository-08 | `P` の `expectedVersion` を読んだ後に、`P` を一度 `save` した | 先に読んだ `expectedVersion` で、`P` を `save` する | `ConflictError`（`STALE_VERSION`）になる。`find()` は一度目の `save` の内容を返す |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRetentionPolicyRepository-09 | `ContentRetentionPolicy` が無い | `deleteAll`（10）を呼ぶ | 0 を返す |
| TC-ports-contentRetentionPolicyRepository-10 | `P` がある | `deleteAll`（10）を呼ぶ | 1 を返す。`find()` は `null` を返す |
| TC-ports-contentRetentionPolicyRepository-12 | `P` がある | `deleteAll`（1）を呼ぶ | 1 を返す。`find()` は `null` を返す |
| TC-ports-contentRetentionPolicyRepository-13 | `P` がある。`P` を 2 回 `save` した | `expectedVersion` を読まずに、`deleteAll`（10）を呼ぶ | 1 を返す。`find()` は `null` を返す |
| TC-ports-contentRetentionPolicyRepository-14 | `P` を `deleteAll` で削除済み | `W1` の `ContentRetentionPolicy` を `insert` する | 成功する。`find()` が、新しい `ContentRetentionPolicy` を返す |
| TC-ports-contentRetentionPolicyRepository-15 | `P` を `deleteAll` で削除済み。削除の前に `expectedVersion` を読んだ | その `expectedVersion` で、`P` を `save` する | `ConflictError`（`STALE_VERSION`）になる。`find()` は `null` を返す |
| TC-ports-contentRetentionPolicyRepository-16 | `P` がある | `deleteAll`（0）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる。`find()` は `P` を返す |
| TC-ports-contentRetentionPolicyRepository-17 | `P` がある | `deleteAll`（上限より 1 大きい `limit`）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる。`find()` は `P` を返す |
| TC-ports-contentRetentionPolicyRepository-18 | `P` がある | UnitOfWork の中で `deleteAll`（10）を呼んだ後、`fn` が例外を投げる | `find()` は `P` を返す |

## 並行性

二つの UnitOfWork を並行に実行する。どちらの `run` も、相手の結果を待たずに呼ぶ。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRetentionPolicyRepository-19 | `P` がある。二つの呼び出し元が、同じ `expectedVersion` を読んだ | 二つの UnitOfWork が、それぞれ別の日数に `change` した `P` を `save` する | 一方だけが確定する。他方は `ConflictError`（`STALE_VERSION`）になる。`find()` は、確定した側の日数を返す |
| TC-ports-contentRetentionPolicyRepository-20 | `ContentRetentionPolicy` が無い | 二つの UnitOfWork が、それぞれ日数が異なる `W1` の `ContentRetentionPolicy` を `insert` する | 一方だけが確定する。他方は `ConflictError`（`ALREADY_EXISTS`）になる。`find()` は、確定した側の日数を返す |

## 可視性と UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRetentionPolicyRepository-21 | `P` がある | `change` した `P` を `save` して確定した直後に、`find()` を呼ぶ | 変更後の日数を返す |
| TC-ports-contentRetentionPolicyRepository-22 | `P` がある | UnitOfWork の中で `change` した `P` を `save` した後、`fn` が例外を投げる | `run` はその例外を投げる。`find()` は、変更前の日数と `version` を返す |
| TC-ports-contentRetentionPolicyRepository-23 | `ContentRetentionPolicy` が無い | UnitOfWork の中で `P` を `insert` した後、`fn` が例外を投げる | `find()` は `null` を返す |
| TC-ports-contentRetentionPolicyRepository-24 | `ContentRetentionPolicy` と `PublicationLedger` が無い | 一つの UnitOfWork で、`P` の `insert` と、`PublicationLedgerRepository.insert` を呼び、確定する | `find()` は `P` を返す。続けて、新しい UnitOfWork で `PublicationLedgerRepository.append` に `PublicationRecordDraft` を渡すと、確定する |
| TC-ports-contentRetentionPolicyRepository-25 | `ContentRetentionPolicy` が無い。`PublicationLedger` はある | 一つの UnitOfWork で、`P` の `insert` と、`PublicationLedgerRepository.insert` を呼ぶ | `ConflictError`（`ALREADY_EXISTS`）になる。`find()` は `null` を返す |
