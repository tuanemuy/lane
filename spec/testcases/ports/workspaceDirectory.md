# WorkspaceDirectory

「上限」は、その節のメソッドの契約が定める、一度に渡せる件数・個数、または `limit` の上限を指す。値は、ポートを定めるドメインのファイルが持つ。

`WorkspaceDirectory` の契約（[../../domains/workspace.md](../../domains/workspace.md)）を、すべてのアダプターが通すケースで定める。

- 名簿は、設置の区画にある。書き込み（`register`、`changeStatus`、`finishCreation`、`remove`）は `UnitOfWorkProvider.run` の UnitOfWork の中で呼ぶ。`WriteScope` は `{ kind: "installation" }`。「確定する」は、`fn` が正常に終わり `run` が値を返すことを指す。「失敗する」は、`run` がそのエラーを投げることを指す
- 読み取りは、断りが無ければ UnitOfWork の外で、`PartitionReader.installation()` の `workspaceDirectory` で呼ぶ
- 「並行した二つの UnitOfWork」は、並行に実行する二つの `run` を指す
- 項目は、ワークスペースの ID と名称の組。断りが無ければ、`initialOwner` は `null`。e、f は、互いに違う `EmailAddress`。ID は `IdGenerator.generate` の値から、名称は `WorkspaceName.create` で作る。ワークスペースの区画に何も確定していなくてよい
- A、B、C は、ID と名称が互いに違う項目。「登録済み」は、`register` を確定した状態を指す。「開設済みの A」は、登録済みの A に `changeStatus(A.id, { status: "open" })` を確定した項目、「終了中の A」は、登録済みの A に `changeStatus(A.id, { status: "retiring" })` を確定した項目を指す
- 同じ区画のほかの書き込みには、`accountRepository` を使う
- 各ケースは、名簿に項目が無い状態から始める

## register

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-workspaceDirectory-01 | なし | `register([A])` を呼び、確定する | `findById(A.id)` が、ID と名称が A と一致し、`status` が `provisioning` の項目を返す。`findByName(A.name)` が、その項目だけを含む列を返す |
| TC-ports-workspaceDirectory-02 | なし | `register([A, B, C])` を一回呼び、確定する | 三つの項目が、どれも `provisioning` で `findById` から返る |
| TC-ports-workspaceDirectory-03 | なし | `register([])` を呼び、確定する | `run` が確定する。何も登録しない |
| TC-ports-workspaceDirectory-04 | なし | `register([A])` を呼んだ後、`fn` が例外を投げる | `findById(A.id)` が `null` を、`findByName(A.name)` が空の列を返す |
| TC-ports-workspaceDirectory-05 | A が登録済み | A と ID が違い、名称が同じ項目 D を `register` し、確定する | `run` が確定する。`findByName(A.name)` が A と D を ID の昇順で返す |
| TC-ports-workspaceDirectory-06 | 終了中の A | 同じ ID と同じ名称の A を、もう一度 `register` し、確定する | `run` が確定する。`findById(A.id)` が、`retiring` のままの A を返す |
| TC-ports-workspaceDirectory-07 | A が登録済み | A と同じ ID で、名称が違う項目を `register` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する。`findById(A.id)` が、元の名称の A を返す |
| TC-ports-workspaceDirectory-08 | なし | 並行した二つの UnitOfWork が、同じ ID で名称が違う二つの項目を、それぞれ `register` する | 一方の `run` だけが確定し、他方が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する。`findById` が、確定した側の項目を返す |
| TC-ports-workspaceDirectory-09 | なし | 初期所有者 e で A を `register` し、確定する | `findById(A.id)` が、`creation` の `initialOwner` が e、`finished` が偽の項目を返す |
| TC-ports-workspaceDirectory-10 | 初期所有者 e で A が登録済み | 同じ ID、同じ名称、初期所有者 f で `register` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する。`findById(A.id)` の `initialOwner` は e のまま |
| TC-ports-workspaceDirectory-11 | A が登録済み（`initialOwner` は `null`） | 同じ ID、同じ名称、初期所有者 e で `register` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する |
| TC-ports-workspaceDirectory-12 | 初期所有者 e で A を登録し、開設済みにし、`finishCreation(A.id)` を確定している | 同じ ID、同じ名称、初期所有者 e で `register` し、確定する | `run` が確定する。`findById(A.id)` が、`open` で `finished` が真のままの A を返す |
| TC-ports-workspaceDirectory-15 | なし | 一つの UnitOfWork で、`register([A])` と、アカウント X の `accountRepository.insert` を呼び、確定する | `findById(A.id)` が A を、`accountRepository.findById(X.id)` が X を返す |
| TC-ports-workspaceDirectory-16 | A が登録済み | 一つの UnitOfWork で、アカウント X の `accountRepository.insert` と、A と同じ ID で名称が違う項目の `register` を呼ぶ | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する。`accountRepository.findById(X.id)` が `null` を返す |
| TC-ports-workspaceDirectory-17 | なし | `register` の上限の件数の項目を、一回の `register` に渡し、確定する | `run` が確定する。`findPageByStatus(["provisioning"], { page: 1, limit: 1 })` の `count` は、渡した件数 |
| TC-ports-workspaceDirectory-18 | なし | 上限より 1 件多い項目を、一回の `register` に渡す | `SystemError`（理由 `INVALID_QUERY`）で失敗する。どの項目も登録しない |

## changeStatus

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-workspaceDirectory-19 | A が登録済み | `changeStatus(A.id, { status: "open" })` を呼び、確定する | `findById(A.id)` が、`status` が `open` の A を返す |
| TC-ports-workspaceDirectory-20 | 開設済みの A | `changeStatus(A.id, { status: "retiring" })` を呼び、確定する | `findById(A.id)` が、`status` が `retiring` の A を返す |
| TC-ports-workspaceDirectory-21 | A が登録済み | `changeStatus(A.id, { status: "retiring" })` を呼び、確定する | `findById(A.id)` が、`status` が `retiring` の A を返す |
| TC-ports-workspaceDirectory-22 | 終了中の A | `changeStatus(A.id, { status: "retiring" })` を呼び、確定する | `run` が確定する。`findById(A.id)` が、`retiring` の A を返す |
| TC-ports-workspaceDirectory-24 | 終了中の A | `changeStatus(A.id, { status: "open" })` を呼び、確定する | `run` が確定する。`findById(A.id)` が、`retiring` のままの A を返す |
| TC-ports-workspaceDirectory-25 | なし | 登録していない ID で `changeStatus` を呼び、確定する | `run` が確定する。その ID の `findById` が `null` を返す |
| TC-ports-workspaceDirectory-26 | A が登録済み | `changeStatus(A.id, { status: "open" })` を呼んだ後、`fn` が例外を投げる | `findById(A.id)` が、`provisioning` の A を返す |
| TC-ports-workspaceDirectory-27 | A が登録済み | 並行した二つの UnitOfWork の一方が `{ status: "open" }` を、もう一方が `{ status: "retiring" }` を、A に `changeStatus` する | 二つの `run` がどちらも確定する。`findById(A.id)` が、`retiring` の A を返す |
| TC-ports-workspaceDirectory-28 | A、B が登録済み | `changeStatus(A.id, { status: "open" })` を呼び、確定する | `findById(B.id)` が、`provisioning` のままの B を返す |
| TC-ports-workspaceDirectory-29 | 開設済みの A | `changeStatus(A.id, { status: "retiring" })` を確定した後、`findByName(A.name)` を呼ぶ | `retiring` の A だけを含む列を返す。名称は変わらない |

## finishCreation

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-workspaceDirectory-30 | 初期所有者 e で A を登録し、開設済みにしている | `finishCreation(A.id)` を呼び、確定する | `findById(A.id)` が、`open` で、`creation` の `initialOwner` が e、`finished` が真の A を返す |
| TC-ports-workspaceDirectory-31 | 初期所有者 e で A が登録済み（`provisioning`） | `finishCreation(A.id)` を呼び、確定する | `run` が確定する。`finished` は偽のまま。状態は `provisioning` のまま |
| TC-ports-workspaceDirectory-32 | 初期所有者 e で A を登録し、開設済みにし、`finishCreation(A.id)` を確定している | もう一度 `finishCreation(A.id)` を呼び、確定する | `run` が確定する。`finished` は真のまま |
| TC-ports-workspaceDirectory-33 | A が登録済み（`initialOwner` は `null`） | `finishCreation(A.id)` を呼び、確定する | `run` が確定する。`creation` は `null` のまま |
| TC-ports-workspaceDirectory-34 | なし | 登録していない ID で `finishCreation` を呼び、確定する | `run` が確定する。その ID の `findById` が `null` を返す |
| TC-ports-workspaceDirectory-35 | 初期所有者 e で A を登録し、開設済みにしている | `finishCreation(A.id)` を呼んだ後、`fn` が例外を投げる | `finished` は偽のまま |

## remove

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-workspaceDirectory-36 | 終了中の A | `remove(A.id)` を呼び、確定する | `findById(A.id)` が `null` を返す。`findByName(A.name)` と `findByIds([A.id])` が空の列を返す |
| TC-ports-workspaceDirectory-37 | A が登録済み（`provisioning`） | `remove(A.id)` を呼び、確定する | `run` が確定する。`findById(A.id)` が `null` を返す |
| TC-ports-workspaceDirectory-39 | なし | 登録していない ID で `remove` を呼び、確定する | `run` が確定する |
| TC-ports-workspaceDirectory-41 | 終了中の A | `remove(A.id)` を呼んだ後、`fn` が例外を投げる | `findById(A.id)` が、`retiring` の A を返す |
| TC-ports-workspaceDirectory-42 | 終了中の A と、開設済みの B | `remove(A.id)` を呼び、確定する | `findById(B.id)` が、`open` の B を返す |
| TC-ports-workspaceDirectory-43 | 終了中の A。A の区画に `Workspace` を `workspaceRepository.insert` で確定している | `remove(A.id)` を呼び、確定する | `run` が確定する。`findById(A.id)` が `null` を返す。A の区画の `workspaceRepository.find()` が、その `Workspace` を返す |
| TC-ports-workspaceDirectory-44 | 終了中の A | 一つの UnitOfWork で、`remove(A.id)` と、アカウント X の `accountRepository.insert` を呼び、確定する | `findById(A.id)` が `null` を、`accountRepository.findById(X.id)` が X を返す |
| TC-ports-workspaceDirectory-45 | 終了中の A | 一つの UnitOfWork で、`remove(A.id)` と、アカウント X の `accountRepository.insert` を呼んだ後、`fn` が例外を投げる | `findById(A.id)` が `retiring` の A を返す。`accountRepository.findById(X.id)` が `null` を返す |

## findById と findByName

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-workspaceDirectory-46 | なし | 登録していない ID で `findById` を呼ぶ | `null` を返す |
| TC-ports-workspaceDirectory-47 | なし | 登録していない名称で `findByName` を呼ぶ | 空の列を返す |
| TC-ports-workspaceDirectory-48 | A、B が登録済み | `findByName(A.name)` を呼ぶ | A だけを含む列を返す |
| TC-ports-workspaceDirectory-50 | A が登録済み | A の名称の前後に空白を加えた文字列から `WorkspaceName.create` で作った値で `findByName` を呼ぶ | A だけを含む列を返す |

## findByIds

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-workspaceDirectory-53 | A が登録済み | `findByIds([])` を呼ぶ | 空の列を返す |
| TC-ports-workspaceDirectory-54 | 登録済みの A、開設済みの B、終了中の C | 三つの ID で `findByIds` を呼ぶ | A、B、C を、それぞれの状態で、ID の昇順で返す |
| TC-ports-workspaceDirectory-56 | A が登録済み | A の ID と、登録していない ID で `findByIds` を呼ぶ | A だけを含む列を返す |
| TC-ports-workspaceDirectory-57 | A が登録済み | `findByIds([A.id, A.id])` を呼ぶ | A を 1 件だけ含む列を返す |
| TC-ports-workspaceDirectory-58 | 上限の件数の項目が登録済み | その全件の ID を、ID の降順に並べて `findByIds` を呼ぶ | 上限の件数を、ID の昇順（文字列のコードポイントの昇順）で返す |
| TC-ports-workspaceDirectory-59 | 上限の件数の項目が登録済み | その全件の ID に、そのうち 1 件の ID をもう一度加えた上限より 1 個多い ID で `findByIds` を呼ぶ | 失敗せず、上限の件数を返す |
| TC-ports-workspaceDirectory-60 | なし | 登録していない、互いに違う上限より 1 個多い ID で `findByIds` を呼ぶ | `SystemError`（理由 `INVALID_QUERY`）で失敗する |

## findPageByStatus

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-workspaceDirectory-61 | なし | `findPageByStatus(["open"], { page: 1, limit: 10 })` を呼ぶ | `items` が空の列、`count` が 0 |
| TC-ports-workspaceDirectory-62 | 登録済みの A、開設済みの B、終了中の C | `findPageByStatus(["open"], { page: 1, limit: 10 })` を呼ぶ | `items` が B だけ、`count` が 1 |
| TC-ports-workspaceDirectory-63 | 同上 | `findPageByStatus(["open", "retiring"], { page: 1, limit: 10 })` を呼ぶ | `items` が B と C、`count` が 2 |
| TC-ports-workspaceDirectory-64 | 同上 | `findPageByStatus(["provisioning", "open", "retiring"], { page: 1, limit: 10 })` を呼ぶ | `items` が A、B、C、`count` が 3 |
| TC-ports-workspaceDirectory-65 | 同上 | `findPageByStatus([], { page: 1, limit: 10 })` を呼ぶ | `items` が空の列、`count` が 0 |
| TC-ports-workspaceDirectory-66 | 同上 | `findPageByStatus(["open", "open"], { page: 1, limit: 10 })` を呼ぶ | `items` が B だけ、`count` が 1 |
| TC-ports-workspaceDirectory-67 | 開設済みの項目が 3 件 | `findPageByStatus(["open"], { page: 1, limit: 10 })` を呼ぶ | `items` が ID の昇順、`count` が 3 |
| TC-ports-workspaceDirectory-68 | 開設済みの項目が 3 件 | `page` が 1、2、3 で、`limit` が 2 の `findPageByStatus(["open"], …)` を呼ぶ | 順に、ID の昇順で先頭の 2 件、残りの 1 件、空の列を返す。どれも `count` が 3 |
| TC-ports-workspaceDirectory-69 | 開設済みの項目が 101 件 | `page` が 1 と 2、`limit` が 100 で呼ぶ | 一つ目の `items` が 100 件、二つ目の `items` が残りの 1 件。どちらも `count` は 101。二つの `items` に重複が無い |
