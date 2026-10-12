# JoiningGuardRepository

「上限」は、その節のメソッドの契約が定める、一度に渡せる件数・個数、または `limit` の上限を指す。値は、ポートを定めるドメインのファイルが持つ。

`JoiningGuardRepository` の契約（[../../domains/workspace.md](../../domains/workspace.md)）と、`TransactionalRepository` の基底の契約（[../../domains/common.md](../../domains/common.md)）を、すべてのアダプターが通すケースで定める。

- リポジトリは、ワークスペースの区画に結び付き、メソッドはワークスペースの ID を引数に取らない。区画は、`UnitOfWorkProvider.run` に渡す `WriteScope` の `workspaceId` と、`PartitionReader.workspace(workspaceId)` が決める。W、V は、互いに違うワークスペース。断りの無い呼び出しは、W の区画のリポジトリに対して行う。「V の区画」と書いた呼び出しは、V の区画のリポジトリに対して行う
- 書き込みは `UnitOfWorkProvider.run` の UnitOfWork の中で呼ぶ。`WriteScope` は、そのワークスペースの `workspace` で、条件は `none`。「確定する」は、`fn` が正常に終わり `run` が値を返すことを指す。「失敗する」は、`run` がそのエラーを投げることを指す
- 読み取りは、断りが無ければ UnitOfWork の外で、`PartitionReader.workspace` のリポジトリで呼ぶ
- 「並行した二つの UnitOfWork」は、並行に実行する二つの `run` を指す。楽観ロックの `expectedVersion` は、二つの `run` の前に読む
- 囲いは `JoiningGuard.advance` で作る。A、B は、互いに違うアカウントの ID で、`IdGenerator.generate` の値から作る。設置の区画にアカウントを確定する必要は無い。「A の囲い G」は、`JoiningGuard.advance(null, A)` の結果を `insert` して確定した囲いを指す
- `deleteAll` は、一回の呼び出しごとに一つの UnitOfWork で確定する

## insert

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-joiningGuardRepository-01 | なし | `JoiningGuard.advance(null, A)` の結果 G を `insert` し、確定する | `findById(A)` が、`accountId` が A、`generation` が 1 の囲いと、`expectedVersion` を返す |
| TC-ports-joiningGuardRepository-02 | なし | G を `insert` した後、`fn` が例外を投げる | `findById(A)` が `null` を返す |
| TC-ports-joiningGuardRepository-03 | A の囲い G | A の囲いを、もう一度 `insert` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する。`findById(A)` が G を返す |
| TC-ports-joiningGuardRepository-04 | なし | 並行した二つの UnitOfWork が、A の囲いをそれぞれ `insert` する | 一方の `run` だけが確定し、他方が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する |
| TC-ports-joiningGuardRepository-06 | A の囲い G | V の区画で、A の囲いを `insert` し、確定する | `run` が確定する。W の区画と V の区画の `findById(A)` が、それぞれの囲いを返す |

## findById

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-joiningGuardRepository-07 | なし | `findById(A)` を呼ぶ | `null` を返す |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-joiningGuardRepository-09 | A の囲い G を読んで v を得ている | `JoiningGuard.advance` を適用した G を v で `save` し、確定する | `findById(A)` が、`generation` が 2 の囲いを返す |
| TC-ports-joiningGuardRepository-10 | G を読んで v を得た後、別の UnitOfWork が `advance` を適用した G を `save` して確定している | `advance` を適用した G を v で `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）で失敗する。`findById(A)` の `generation` は 2 |
| TC-ports-joiningGuardRepository-11 | A の囲い G | 並行した二つの UnitOfWork が、同じ `expectedVersion` で、`advance` を適用した G をそれぞれ `save` する | 一方の `run` だけが確定し、他方が `ConflictError`（理由 `STALE_VERSION`）で失敗する |
| TC-ports-joiningGuardRepository-12 | A の囲い G | 並行した二つの UnitOfWork が、一方は、同じ `expectedVersion` で `advance` を適用した G を `save` し、A のメンバー M を `memberRepository.insert` する。他方は、`advance` を適用した G を `save` する | 結果は次のどちらかになる。メンバーを作る側だけが確定し、M がある。メンバーを作らない側だけが確定し、M は無い。どちらの場合も、`findById(A)` の `generation` は 2 |
| TC-ports-joiningGuardRepository-13 | A の囲い G を読んで v を得ている | `advance` を適用した G を v で `save` した後、`fn` が例外を投げる | `findById(A)` の `generation` は 1 |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-joiningGuardRepository-14 | A と B の囲い。W は `closing` | `deleteAll(10)` を呼び、確定する | 2 を返す。`findById(A)` と `findById(B)` が `null` を返す |
| TC-ports-joiningGuardRepository-15 | A と B の囲い | `deleteAll(1)` を三回、それぞれ確定する | 順に 1、1、0 を返す |
| TC-ports-joiningGuardRepository-16 | W の A の囲いと、V の A の囲い | W の区画で `deleteAll(10)` を呼び、確定する | 1 を返す。V の区画の `findById(A)` が囲いを返す |
| TC-ports-joiningGuardRepository-17 | A の囲い G | `deleteAll(10)` を呼んだ後、`fn` が例外を投げる | `findById(A)` が G を返す |
| TC-ports-joiningGuardRepository-18 | 囲いを、上限の件数だけ持つ W | `deleteAll`（上限の `limit`）を呼び、確定する | 上限の件数を返す |
| TC-ports-joiningGuardRepository-19 | A の囲い G | `deleteAll(0)` と、上限より 1 大きい `limit` の `deleteAll` を、それぞれ別の UnitOfWork で呼ぶ | どちらの `run` も `SystemError`（理由 `INVALID_QUERY`）で失敗する。`findById(A)` が G を返す |
