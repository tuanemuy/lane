# PendingFormSettingsRepository

`spec/domains/form.md` の `PendingFormSettingsRepository` の契約と、`spec/domains/common.md` のリポジトリに共通する規則を、すべてのアダプターが通すケースで定める。

- 「UoW」は `UnitOfWorkProvider.run` の `fn`。「確定」は `fn` の正常終了を指す
- UoW の `WriteScope` は、どのケースも、対象のワークスペースの `{ kind: "workspace", workspaceId, condition: { kind: "none" } }`。区画と条件は、`unitOfWorkProvider.md` が確かめる
- リポジトリは、ワークスペースの区画に結び付く。W1 の書き込みは W1 の UoW で、読み取りは `PartitionReader.workspace(W1)` のリポジトリで行う。W2 も同じ。断りが無い限り、読み取りは W1 の区画のリポジトリで行う。メソッドは、ワークスペースの ID を取らない
- 前提条件の書き込みは、別の UoW で確定済み。期待結果の読み取りは、UoW の外で行う
- W1 と W2 は別の、確定済みのワークスペース。K1 と K2 は別の `FormKey`（K1 < K2）。F1 は `FormId`
- 紐付け待ちの設定は、`PendingFormSettings.declare`（`FormSettingsInput.validate` が返した値を渡す）の結果。「W1・K1 の紐付け待ち」は、W1 の区画に保存した、`formKey` が K1 の紐付け待ちの設定を指す
- `deleteMany` に渡す要素は、`findAll` で読んだ値と `expectedVersion` の組

## insertMany

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-pendingFormSettingsRepository-01 | 紐付け待ちが無い | UoW で、W1 の K1 と K2 の紐付け待ちを `insertMany` で渡して確定する | `findAll()` が、渡した値と全フィールドが等しい `entity` を 2 件返す |
| TC-ports-pendingFormSettingsRepository-02 | 紐付け待ちが無い | `notification` が `null` の紐付け待ちを `insertMany` で渡して確定する | `findAll()` が返す `notification` は `null` |
| TC-ports-pendingFormSettingsRepository-03 | 紐付け待ちが無い | UoW で、空の列を `insertMany` で渡して確定する | 成功する。`findAll()` は空 |
| TC-ports-pendingFormSettingsRepository-04 | W1・K1 の紐付け待ちがある | UoW で、K2 と K1 の紐付け待ちを `insertMany` で渡す | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。`findAll()` は、元の K1 の 1 件 |
| TC-ports-pendingFormSettingsRepository-05 | 紐付け待ちが無い | UoW で、`formKey` が同じ K1 の紐付け待ちを二つ `insertMany` で渡す | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。`findAll()` は空 |
| TC-ports-pendingFormSettingsRepository-06 | W1・K1 の紐付け待ちがある | W2 の UoW で、K1 の紐付け待ちを `insertMany` で渡して確定する | 成功する。W1 の区画と W2 の区画の `findAll()` が、それぞれの値を返す |
| TC-ports-pendingFormSettingsRepository-07 | W1 の `FormSettingsRepository` に、`formKey` が K1 の `removed`（F1）がある | UoW で、W1・K1 の紐付け待ちを `insertMany` で渡して確定する | 成功する。`FormSettingsRepository.findById(F1)` は `removed` を返し、`findAll()` は K1 の 1 件を返す |
| TC-ports-pendingFormSettingsRepository-08 | 紐付け待ちが無い | 一つの UoW で、`UnitOfWorkCapacity.workspaceWrites` を超える件数の紐付け待ちを、一回の `insertMany` で渡して確定する | 成功する。`findAll()` は、渡した全件を返す。件数が、ワークスペースの区画の UnitOfWork の書き込みの回数（`UnitOfWorkCapacity.workspaceWrites`）を超えても、一回の書き込みとして確定する |
| TC-ports-pendingFormSettingsRepository-09 | 紐付け待ちが無い | UoW で、2 件の紐付け待ちを `insertMany` で渡し、`fn` が例外を投げる | `findAll()` は空 |
| TC-ports-pendingFormSettingsRepository-10 | 紐付け待ちが無い | 二つの UoW が並行して、W1・K1 の紐付け待ちを含む列を `insertMany` で渡す | 一方だけが確定する。他方の `run` は `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。`findAll()` は 1 件 |

## findAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-pendingFormSettingsRepository-11 | W1 に紐付け待ちが無い | `findAll()` を呼ぶ | 空の列 |
| TC-ports-pendingFormSettingsRepository-12 | 区画に何も確定していないワークスペース | その区画のリポジトリで `findAll()` を呼び、その区画の UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼んで確定する | `findAll` は空の列、`deleteAll` は 0 を返す |
| TC-ports-pendingFormSettingsRepository-13 | W1 に、K2、K1 の順に保存した紐付け待ちがある | `findAll()` を呼ぶ | K1、K2 の順に、`entity` と `expectedVersion` を返す |
| TC-ports-pendingFormSettingsRepository-14 | W1 に 2 件、W2 に 1 件ある | W1 の区画のリポジトリで `findAll()` を呼ぶ | W1 の 2 件だけ |
| TC-ports-pendingFormSettingsRepository-15 | W1 に `UnitOfWorkCapacity.workspaceWrites` 件より多くある | `findAll()` を呼ぶ | すべてを一度に返す |
| TC-ports-pendingFormSettingsRepository-16 | W1・K1 の紐付け待ちがある | UoW の外で `findAll()` を呼ぶ | UoW を開かずに返す |

## deleteMany

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-pendingFormSettingsRepository-17 | W1・K1 と W1・K2 の紐付け待ちを読む | UoW で、K1 の組を `deleteMany` で渡して確定する | `findAll()` は K2 の 1 件だけ |
| TC-ports-pendingFormSettingsRepository-18 | W1・K1 と W1・K2 の紐付け待ちを読む | UoW で、K1 と K2 の組を一回の `deleteMany` で渡して確定する | `findAll()` は空 |
| TC-ports-pendingFormSettingsRepository-19 | W1・K1 の紐付け待ちがある | UoW で、空の列を `deleteMany` で渡して確定する | 成功する。`findAll()` は K1 の 1 件 |
| TC-ports-pendingFormSettingsRepository-20 | W1・K1 と W1・K2 の紐付け待ちを読み、その後に K2 を `deleteMany` で削除して確定した | UoW で、最初に読んだ K1 と K2 の組を一回の `deleteMany` で渡す | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`findAll()` は K1 の 1 件 |
| TC-ports-pendingFormSettingsRepository-22 | W1・K1 の紐付け待ちを読む | 二つの UoW が並行して、読んだ K1 の組を `deleteMany` で渡す | 一方だけが確定する。他方の `run` は `ConflictError`（理由 `STALE_VERSION`）を投げる。`findAll()` は空 |
| TC-ports-pendingFormSettingsRepository-23 | W1・K1、W1・K2 と W2・K1、W2・K2 の紐付け待ちがある。W1 の K1 を読む | W1 の UoW で、読んだ組を `deleteMany` で渡して確定し、続けて別の W1 の UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼んで確定する | W2 の区画の `findAll()` は K1 と K2 の 2 件のまま |
| TC-ports-pendingFormSettingsRepository-24 | W1・K1 と W1・K2 の紐付け待ちを読む | UoW で、K1 の組を二度と K2 の組を含む列を `deleteMany` で渡す | `run` が `SystemError`（理由 `INVALID_QUERY`）で失敗する。`findAll()` は K1 と K2 の 2 件のまま |
| TC-ports-pendingFormSettingsRepository-25 | W1・K1 の紐付け待ちを読む | UoW で、読んだ組を `deleteMany` で渡し、`fn` が例外を投げる | `findAll()` は K1 の 1 件のまま |
| TC-ports-pendingFormSettingsRepository-26 | W1 に `UnitOfWorkCapacity.workspaceWrites` 件より多くの紐付け待ちがあり、すべてを読む | 一つの UoW で、読んだ全件の組を一回の `deleteMany` で渡して確定する | 成功する。`findAll()` は空。件数が、ワークスペースの区画の UnitOfWork の書き込みの回数（`UnitOfWorkCapacity.workspaceWrites`）を超えても、一回の書き込みとして確定する |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-pendingFormSettingsRepository-27 | W1 に紐付け待ちが無い | UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼び、確定する | 0 を返す |
| TC-ports-pendingFormSettingsRepository-29 | W1 に 5 件ある | 呼び出しごとに別の UoW で、`deleteAll(2)` を、返る件数が 2 より少なくなるまで繰り返す | 2、2、1 の順に返す。`findAll()` は空 |
| TC-ports-pendingFormSettingsRepository-31 | W1 の区画の `Workspace` は `active`。W1 に 2 件ある | UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼び、確定する | ワークスペースの状態を確かめずに 2 を返す |
| TC-ports-pendingFormSettingsRepository-32 | W1 に 2 件ある | UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼び、`fn` が例外を投げる | `findAll()` は 2 件のまま |
| TC-ports-pendingFormSettingsRepository-33 | W1 に 2 件ある | UoW で `deleteAll(0)` を呼ぶ。別の UoW で、`BoundedDeletion.maxLimit` を超える `limit` で `deleteAll` を呼ぶ | どちらも `run` が `SystemError`（理由 `INVALID_QUERY`）で失敗する。`findAll()` は 2 件のまま |

## 可視性と UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-pendingFormSettingsRepository-36 | 紐付け待ちが無い | UoW で K1 を `insertMany` で渡し、確定の直後に `findAll()` を呼ぶ | 待たずに K1 を返す |
| TC-ports-pendingFormSettingsRepository-37 | W1 の `FormSettingsRepository` に F1 の `active` がある | 一つの UoW で、K1 の紐付け待ちの `insertMany` と、F1 の変更の `save` を行い、`fn` が例外を投げる | `run` はその例外を投げる。`findAll()` は空で、F1 は変更前 |
| TC-ports-pendingFormSettingsRepository-38 | W1・K1 の紐付け待ちを読む | 一つの UoW で、`FormSettingsRepository.applySchemaChange`（`bound` に、読んだ K1 の組と `bind` の結果の F1 の `active`）と、その組の `pending` の `deleteMany` を行い、確定する | `findAll()` は空。`FormSettingsRepository.findById(F1)` は `active` を返す |
| TC-ports-pendingFormSettingsRepository-39 | W1・K1 の紐付け待ちを読む | 一つの UoW で、上と同じ二つの書き込みを行い、`fn` が例外を投げる | `findAll()` は K1 の 1 件のまま。`FormSettingsRepository.findById(F1)` は `null` |
