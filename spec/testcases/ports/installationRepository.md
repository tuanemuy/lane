# InstallationRepository

`spec/domains/installation.md` の `InstallationRepository` の契約と、`spec/domains/common.md` のリポジトリに共通する規則を、すべてのアダプターが通すケースで定める。

- 「UoW」は、`WriteScope` に `{ kind: "installation" }` を渡した `UnitOfWorkProvider.run` の `fn`。書き込みは、`fn` が受ける `InstallationContext` の `installationRepository` で行う。「確定」は `fn` の正常終了を指す
- 前提条件の書き込みは、別の UoW で確定済み。期待結果の読み取りは、UoW の外で、`PartitionReader.installation()` の `installationRepository` で行う
- 「空の設置」は、`find` が `null` を返す設置
- `Installation` の値は、`Installation.initialize`、`completeInitialConfiguration`、`beginUpdate`、`completeUpdate` の結果
- 「ほかの集約」は、同じ設置の区画のアカウント。`accountRepository` で書く

## find

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-installationRepository-01 | 空の設置 | `find` を呼ぶ | `null` |
| TC-ports-installationRepository-02 | `Installation` がある | `find` を呼ぶ | その `Installation` と `expectedVersion` を返す |
| TC-ports-installationRepository-03 | `Installation` がある | UoW の外で `find` を呼ぶ | UoW を開かずに `Installation` を返す |
| TC-ports-installationRepository-04 | 空の設置。ほかの集約は設置の区画にある | `find` を呼ぶ | `null` |

## insert

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-installationRepository-05 | 空の設置 | UoW で、`initialConfiguration` が `loading` の `Installation` を `insert` して確定する | `find` が、渡した値と全フィールド（`initialConfiguration` の `status` と `declarationDigest`、`dataVersion`、`update`、`version`、`createdAt`、`updatedAt`）が等しい `entity` と `expectedVersion` を返す |
| TC-ports-installationRepository-06 | 空の設置 | UoW で、`initialConfiguration` が `settled` の `Installation` を `insert` して確定する | `find` が、`settled` の `Installation` を返す |
| TC-ports-installationRepository-07 | `Installation` がある | UoW で、`Installation` を `insert` する | `run` が `ConflictError`（理由 `INSTALLATION_EXISTS`）を投げる。`find` は元の `Installation` を返す |
| TC-ports-installationRepository-08 | `Installation` がある設置と、空の設置 | 一つの UoW で、ほかの集約の `insert` と、`Installation` の `insert`（`Installation` がある設置）、または任意の `expectedVersion` での `save`（空の設置）を行う | `run` が `ConflictError`（理由は、`insert` では `INSTALLATION_EXISTS`、`save` では `STALE_VERSION`）を投げる。ほかの集約の書き込みは反映されない |
| TC-ports-installationRepository-09 | 空の設置 | 二つの UoW が並行して、それぞれ `Installation` と、ほかの集約を `insert` する | 一方だけが確定する。他方の `run` は `ConflictError`（理由 `INSTALLATION_EXISTS`）を投げ、他方が書いたほかの集約は一件も残らない。`find` は確定した側の値を返す |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-installationRepository-10 | `loading` の `Installation` を `find` で読む | UoW で、`completeInitialConfiguration` の結果を、読んだ `expectedVersion` で `save` して確定する | `find` が、`initialConfiguration` が `settled` で、1 進んだ `version` の `Installation` を返す。`dataVersion` と `createdAt` は変わらない |
| TC-ports-installationRepository-11 | `initialConfiguration` が `settled` の `Installation` を `find` で読む | UoW で、`beginUpdate` の結果を、読んだ `expectedVersion` で `save` して確定する | `find` が、`update` が `updating` で `to` が `beginUpdate` に渡した版、1 進んだ `version` の `Installation` を返す。`dataVersion`、`initialConfiguration`、`createdAt` は変わらない |
| TC-ports-installationRepository-12 | `update` が `updating` の `Installation` を `find` で読む | UoW で、`completeUpdate` の結果を、読んだ `expectedVersion` で `save` して確定する | `find` が、`dataVersion` が更新先の版（保存前の `update.to`）、`update` が `idle` で、1 進んだ `version` の `Installation` を返す。`initialConfiguration` と `createdAt` は変わらない |
| TC-ports-installationRepository-13 | `Installation` を `save` して確定した | 保存の後に `find` で読んだ `expectedVersion` で、次の変更の結果を `save` して確定する | 成功する |
| TC-ports-installationRepository-14 | `initialConfiguration` が `settled` の `Installation`、または `loading` の `Installation` を `find` で読む | 二つの UoW が並行して、同じ `expectedVersion` で、`beginUpdate`（`settled`）または `completeInitialConfiguration`（`loading`）の結果を `save` する | 一方だけが確定する。他方の `run` は `ConflictError`（理由 `STALE_VERSION`）を投げる |
| TC-ports-installationRepository-15 | `Installation` を二度 `find` で読み、一方の変更を `save` して確定した | UoW で、他方の変更を、最初に読んだ `expectedVersion` で `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`find` は先に確定した内容を返す |
| TC-ports-installationRepository-17 | 空の設置 | UoW で、`Installation` を任意の `expectedVersion` で `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`find` は `null` |

## 復旧の後

前提条件の `restore` は、そのバックアップの `Recovery.begin` の結果（`restoring` の記録）を渡して呼ぶ。復旧の完了より前に始まった UoW が確定しないことは、`backupStore.md` が確かめる。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-installationRepository-19 | `initialConfiguration` が `settled` の `Installation` がある状態でバックアップを取得した。その後に `beginUpdate` の結果だけを保存した設置（`update` が `updating`）と、さらに `completeUpdate` の結果を保存した設置で、それぞれ `BackupStore.restore` が完了した | `find` を呼ぶ | どちらも、`initialConfiguration`、`dataVersion`、`update`（`idle`）、`version` が、取得の時点の値の `Installation` を返す |
| TC-ports-installationRepository-21 | 空の設置。`Installation` を含むバックアップがある。`BackupStore.restore` が完了した | `find` を呼ぶ | バックアップの中の `Installation` を返す |
| TC-ports-installationRepository-22 | `BackupStore.restore` が完了した | 復旧の後に始めた UoW の中で `find` を呼び、得た `expectedVersion` で、`beginUpdate` の結果を `save` して確定する | 成功する |

## 可視性と UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-installationRepository-23 | 空の設置 | UoW で `Installation` を `insert` し、確定の直後に `find` を呼ぶ | 待たずに、その `Installation` を返す |
| TC-ports-installationRepository-24 | 空の設置 | 一つの UoW で、`Installation` の `insert` と、ほかの集約の `insert` を行い、確定する | 両方が反映される |
| TC-ports-installationRepository-25 | 空の設置と、`Installation` がある設置 | 一つの UoW で、`Installation` の `insert`（空の設置）、または `beginUpdate` の結果の `save`（`Installation` がある設置）と、ほかの集約の `insert` を行い、`fn` が例外を投げる | `run` はその例外を投げる。`find` は、空の設置では `null`、`Installation` がある設置では元の `update`（`idle`）と `version` の `Installation` を返す。ほかの集約も残らない |
