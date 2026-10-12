# FormSettingsRepository

`spec/domains/form.md` の `FormSettingsRepository` の契約と、`spec/domains/common.md` のリポジトリに共通する規則を、すべてのアダプターが通すケースで定める。

- 「UoW」は `UnitOfWorkProvider.run` の `fn`。「確定」は `fn` の正常終了を指す
- UoW の `WriteScope` は、どのケースも、対象のワークスペースの `{ kind: "workspace", workspaceId, condition: { kind: "none" } }`。区画と条件は、`unitOfWorkProvider.md` が確かめる
- リポジトリは、ワークスペースの区画に結び付く。W1 の設定の書き込みは W1 の UoW で、読み取りは `PartitionReader.workspace(W1)` のリポジトリで行う。W2 も同じ。断りが無い限り、読み取りは W1 の区画のリポジトリで行う。メソッドは、ワークスペースの ID を取らない
- 前提条件の書き込みは、別の UoW で確定済み。期待結果の読み取りは、UoW の外で行う
- 設定は `workspaceId` を持たない。「W1・F1 の設定」は、W1 の区画に保存した、`formId` が F1 の設定を指す
- W1 と W2 は別の、確定済みのワークスペース。F1、F2、F3 は別の `FormId`（F1 < F2 < F3）。K1 と K2 は別の `FormKey`（K1 < K2）
- 紐付け待ちの設定は、`PendingFormSettings.declare`（`FormSettingsInput.validate` が返した値を渡す）の結果を、同じ区画の `PendingFormSettingsRepository.insertMany` で保存したもの。「W1・K1 の紐付け待ち」は、W1 の区画の、`formKey` が K1 の紐付け待ちの設定を指す。`active` の設定は、`createForNewForm` または `bind` の結果を、`applySchemaChange` の `created` または `bound` で保存したもの。`removed` の設定は、`active` の設定に `FormSettings.remove` を適用した結果を、`save` で保存したもの
- 「`findById(F1)`」は、`formId` が F1 の設定を引く。紐付け待ちの設定の有無は、`PendingFormSettingsRepository.findAll()` で確かめる

## findById

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-formSettingsRepository-01 | 区画が無いワークスペース（存在しない ID） | その区画のリポジトリで `findById(F1)` と `findAll()` を呼び、その区画の UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼んで確定する | `findById` は `null`、`findAll` は空の列、`deleteAll` は 0 を返す |
| TC-ports-formSettingsRepository-02 | 設定が無い。または W1・F1 の `active` だけがある | `findById(F2)` を呼ぶ | `null` |
| TC-ports-formSettingsRepository-03 | W1・F1 の `active` がある | `findById(F1)` を呼ぶ | `active` の設定と `expectedVersion` を返す。保存した設定と全フィールドが等しい |
| TC-ports-formSettingsRepository-04 | W1・F1 の `active` が、通知先を 3 件（有効と無効を含む）持つ | `findById(F1)` を呼ぶ | `recipients` は、保存した順序と有効・無効を保つ |
| TC-ports-formSettingsRepository-05 | W1・F1 の `removed` がある | `findById(F1)` を呼ぶ | `removed` の設定と `expectedVersion` を返す。`formKey`、`formLabel`、`removedAt` は保存した値 |
| TC-ports-formSettingsRepository-06 | W1 に F1 の `active` を含む 2 件、W2 に F3 の `active` が 1 件ある | W2 の区画のリポジトリで `findById(F1)` を、W1 の区画のリポジトリで `findAll()` を呼ぶ | `findById` は `null`。`findAll` は W1 の 2 件だけ。ほかのワークスペースの区画を読まない |
| TC-ports-formSettingsRepository-08 | W1・F1 の `active` がある | UoW の外で `findById(F1)` を呼ぶ | UoW を開かずに設定を返す |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-formSettingsRepository-09 | W1・F1 の `active` を `findById` で読む | UoW で、`FormSettings.addRecipient`、`open`、`changeNotification`、`changeRetention` を順に適用した結果（変更の振る舞いは、どれも `changed` の `value`）を、読んだ `expectedVersion` で `save` して確定する | `findById` が、変更後の `recipients`、`reception`、`notification`、`retention`、`version`、`updatedAt` を返す |
| TC-ports-formSettingsRepository-11 | W1・F1 の `active` を `findById` で読む | UoW で、`FormSettings.remove` の結果を `save` して確定する | `findById(F1)` が `removed` を返す。`recipients` は引き継いだ値。`findAll()` は 1 件 |
| TC-ports-formSettingsRepository-12 | W1・F1 の `removed` を `findById` で読む | UoW で、`FormSettings.removeRecipient` の結果を `save` して確定する | `findById` が、変更後の `recipients` を持つ `removed` を返す |
| TC-ports-formSettingsRepository-13 | W1・F1 の `active` を `save` して確定した | 保存の後に `findById` で読んだ `expectedVersion` で、次の変更を `save` して確定する | 成功する |
| TC-ports-formSettingsRepository-14 | W1・F1 の `active` を二度 `findById` で読み、一方の変更を `save` して確定した | UoW で、他方の変更を、最初に読んだ `expectedVersion` で `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById` は先に確定した内容を返す |
| TC-ports-formSettingsRepository-15 | W1・F1 の `active` を `findById` で読む | 二つの UoW が並行して、同じ `expectedVersion` で別の変更を `save` する | 一方だけが確定する。他方の `run` は `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById` は確定した側の内容を返す |
| TC-ports-formSettingsRepository-16 | W1 に F1 と F2 の `active` がある | 二つの UoW が並行して、F1 と F2 をそれぞれ `save` する | 両方が確定する |
| TC-ports-formSettingsRepository-17 | W1・F1 の設定を `findById` で読み、その後に `delete` または `deleteAll` で削除して確定した | UoW で、読んだ `expectedVersion` で変更を `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById(F1)` は `null` |
| TC-ports-formSettingsRepository-19 | W1・F1 の `active` を `save` で変更して確定した。`findAll` で読む | `findById(F1)` を呼び、続けて UoW で、`findAll` が返した `expectedVersion` で変更を `save` して確定する | `findAll` と `findById` が返す `entity` と `expectedVersion` は等しい。`save` は成功する |

## delete

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-formSettingsRepository-20 | W1・F1 の `removed` を `findById` または `findAll` で読む | UoW で、読んだ `expectedVersion` で `delete(F1, expectedVersion)` を呼び、確定する | `findById(F1)` は `null`。`findAll()` は含まない |
| TC-ports-formSettingsRepository-22 | W1・F1 の `removed` があり、同じ区画の `SubmissionRepository` に、`formId` が F1 の回答が保存済み。`removed` を `findById` で読む | UoW で、読んだ `expectedVersion` で `delete` して確定する | 成功する。回答の有無を確かめない。`findById(F1)` は `null`。`SubmissionRepository.findById` は、その回答を返す |
| TC-ports-formSettingsRepository-23 | W1・F1 の `removed` を `findById` で読み、その後に、別の変更を `save` して、または `delete` して確定した | UoW で、最初に読んだ `expectedVersion` で `delete` する | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById` は、`save` 後の設定、または `null` を返す |
| TC-ports-formSettingsRepository-24 | W1・F1 の `removed` を `findById` で読む | 二つの UoW が並行して、同じ `expectedVersion` で `save` と `delete` を行う | 一方だけが確定する。他方の `run` は `ConflictError`（理由 `STALE_VERSION`）を投げる |
| TC-ports-formSettingsRepository-26 | W1 と W2 に、それぞれ F1 の `removed` と F2 の `active` がある。W1 の F1 を読む | W1 の UoW で、読んだ `expectedVersion` で F1 を `delete` して確定し、続けて別の W1 の UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼んで確定する | W2 の区画の `findAll()` は、F1 と F2 の 2 件のまま |

## applySchemaChange

引数は `FormSettingsSchemaChange`。`FormSettings.planSchemaChange` の結果と同じ形で、`created` は `createForNewForm` の結果、`bound` は `bind` の結果と、紐付け元の紐付け待ちの設定（`PendingFormSettingsRepository.findAll` で読んだ値と `expectedVersion`）の組（紐付け待ちの設定の削除は `pendingFormSettingsRepository.md` の `deleteMany` が確かめる）、`removed` は `remove` の結果と、元の `active` を読んだ `expectedVersion` の組。書かない列は空。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-formSettingsRepository-27 | 設定が無い | UoW で、`created` に F1 と F2 の `active` を渡して確定する | `findById(F1)` と `findById(F2)` が、渡した設定と全フィールドが等しい `entity` を返す |
| TC-ports-formSettingsRepository-28 | W1・K1 の紐付け待ちを読む | UoW で、`bound` に、読んだ紐付け待ちと `bind` の結果（F1 の `active`）の組を渡して確定する | `findById(F1)` は、引き継いだ通知先・受付状態・保持方針を持つ `version` 0 の `active` を返す。`findAll()` は、その 1 件だけ。`PendingFormSettingsRepository.findAll()` は K1 を返す。紐付け待ちの設定を書かない |
| TC-ports-formSettingsRepository-29 | W1・F1 の `active` を読む | UoW で、`removed` に、`remove` の結果と読んだ `expectedVersion` の組を渡して確定する | `findById(F1)` が `removed` を返す。`recipients` は引き継いだ値 |
| TC-ports-formSettingsRepository-30 | W1 に、K1 の紐付け待ちと F2 の `active` を読む | 一つの UoW で、`created` に F3 の `active`、`bound` に K1 の組（F1 の `active`）、`removed` に F2 の組を、一回の呼び出しで渡して確定する | F3 は `active`、F1 は `active`、F2 は `removed`。`findAll()` は 3 件 |
| TC-ports-formSettingsRepository-31 | 設定が無い | UoW で、三つの列がすべて空の `applySchemaChange` を呼び、確定する | 成功する。`findAll()` は空 |
| TC-ports-formSettingsRepository-32 | W1・F1 の `active` または `removed` がある | UoW で、`created` に F2 と F1 の `active` を渡す | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。`findById(F2)` は `null`。F1 は元の設定 |
| TC-ports-formSettingsRepository-34 | W1・F1 の `active` と、K1 の紐付け待ちがある。紐付け待ちを読む | UoW で、`bound` に、読んだ紐付け待ちと F1 の `active` の組を渡す | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。K1 の紐付け待ちは残り、F1 は元の設定 |
| TC-ports-formSettingsRepository-35 | W1・F2 の `active` を読み、その後に別の変更を `save` して確定した | UoW で、`created` に F3 の `active`、`removed` に、最初に読んだ `expectedVersion` と F2 の `remove` の結果の組を渡す | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById(F3)` は `null`。F2 は `save` 後の `active` |
| TC-ports-formSettingsRepository-36 | W1・K1 の紐付け待ちと F2 の `active` を読む | UoW で、`bound` と `removed` を渡して `applySchemaChange` を呼んだ後、`fn` が例外を投げる | `findById(F1)` は `null`、F2 は元の `active` |
| TC-ports-formSettingsRepository-37 | W1・F2 の `active` を読む | 二つの UoW が並行して、一方は `removed` に F2 の組を渡して `applySchemaChange` を呼び、他方は同じ `expectedVersion` で F2 の通知先の変更を `save` する | 一方だけが確定する。他方の `run` は `ConflictError`（理由 `STALE_VERSION`）を投げる |
| TC-ports-formSettingsRepository-38 | 設定が無い | 二つの UoW が並行して、`created` に F1 の `active` を渡す | 一方だけが確定する。他方の `run` は `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。`findAll()` は 1 件 |
| TC-ports-formSettingsRepository-39 | W1 に、紐付け待ちの設定がある。`created` と `bound` に渡す件数の合計は、`UnitOfWorkCapacity.workspaceWrites` を超える | 一つの UoW で、`created` に `active` の設定、`bound` に、読んだ紐付け待ちの組を、一回の `applySchemaChange` で渡して確定する | 成功する。`findAll()` は、渡した全件を `active` で返す。件数が、ワークスペースの区画の UnitOfWork の書き込みの回数（`UnitOfWorkCapacity.workspaceWrites`）を超えても、一回の書き込みとして確定する |

## findAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-formSettingsRepository-40 | W1 に設定が無い | `findAll()` を呼ぶ | 空の列 |
| TC-ports-formSettingsRepository-42 | W1 に `active` が 1 件ある | `findAll()` を呼ぶ | その 1 件の `entity` と `expectedVersion` |
| TC-ports-formSettingsRepository-44 | W1 に、`createdAt` が互いに異なる `active` 2 件と `removed` 1 件を、`createdAt` の降順に保存した | `findAll()` を呼ぶ | 保存の順と状態によらず、3 件を `createdAt` の昇順で返す |
| TC-ports-formSettingsRepository-46 | W1 に、`createdAt` が同じ `active`（F2）と `removed`（F1）と、それより新しい 1 件がある | `findAll()` を呼ぶ | F1、F2、新しい 1 件の順で返す。同じ日時は、状態によらず `formId` の昇順 |
| TC-ports-formSettingsRepository-49 | W1 に `UnitOfWorkCapacity.workspaceWrites` 件より多くある | `findAll()` を呼ぶ | すべてを一度に返す |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-formSettingsRepository-50 | W1 に設定が無い | UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼び、確定する | 0 を返す |
| TC-ports-formSettingsRepository-51 | W1 に `active` が 2 件、`removed` が 1 件ある | UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼び、確定する | 3 を返す。`findAll()` は空 |
| TC-ports-formSettingsRepository-53 | W1 に 3 件ある | UoW で `deleteAll(2)` を呼び、確定する | 2 を返す。`findAll()` は 1 件 |
| TC-ports-formSettingsRepository-55 | W1 に 5 件ある | 呼び出しごとに別の UoW で、`deleteAll(2)` を、返る件数が 2 より少なくなるまで繰り返す | 2、2、1 の順に返す。`findAll()` は空 |
| TC-ports-formSettingsRepository-57 | W1・F1 の `active` を `save` で 3 回変更して確定した | UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼び、確定する | `expectedVersion` を渡さずに 1 を返す。`findById(F1)`は `null` |
| TC-ports-formSettingsRepository-58 | W1 の区画の `Workspace` は `active`。W1 に 2 件ある | UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼び、確定する | ワークスペースの状態を確かめずに 2 を返す |
| TC-ports-formSettingsRepository-59 | W1 に 2 件ある | UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼び、`fn` が例外を投げる | `findAll()` は 2 件のまま |
| TC-ports-formSettingsRepository-60 | W1 に 2 件ある | UoW で `deleteAll(0)` を呼ぶ。別の UoW で、`BoundedDeletion.maxLimit` を超える `limit` で `deleteAll` を呼ぶ | どちらも `run` が `SystemError`（理由 `INVALID_QUERY`）で失敗する。`findAll()` は 2 件のまま |

## 可視性と UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-formSettingsRepository-63 | 設定が無い | UoW で、`created` に F1 の `active` を渡して `applySchemaChange` を呼び、確定の直後に `findById` と `findAll` を呼ぶ | 待たずに、どちらもその設定を返す |
| TC-ports-formSettingsRepository-64 | W1・F1 の `active` と W1・F2 の `removed` がある | 一つの UoW で、F3 の `active` の `applySchemaChange`、F1 の `save`、F2 の `delete` を行い、確定する | 三つの書き込みがすべて反映される |
| TC-ports-formSettingsRepository-65 | W1・F1 の `active` と W1・F2 の `removed` がある | 一つの UoW で、F3 の `active` の `applySchemaChange`、F1 の `save`、F2 の `delete` を行い、`fn` が例外を投げる | `run` はその例外を投げる。`findAll()` は F3 を含まず、F1 は変更前、F2 は元の `removed` |
| TC-ports-formSettingsRepository-66 | W1・F1 の `active` を `findById` で読み、その後に別の変更を `save` して確定した | 一つの UoW で、F3 の `active` の `applySchemaChange` と、最初に読んだ `expectedVersion` での F1 の `save` を行う | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`findAll()` は F3 を含まない |
