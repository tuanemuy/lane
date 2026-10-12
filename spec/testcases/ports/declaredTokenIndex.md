# DeclaredTokenIndex

`DeclaredTokenIndex` の契約（[../../domains/token.md](../../domains/token.md)）を、すべてのアダプターが通すケースで定める。

- 索引は、設置の区画にある。書き込み（`register`、`removeByWorkspace`）は `UnitOfWorkProvider.run` の UnitOfWork の中で呼ぶ。`WriteScope` は `{ kind: "installation" }`。「確定する」は、`fn` が正常に終わり `run` が値を返すことを指す。「失敗する」は、`run` がそのエラーを投げることを指す
- 読み取りは、断りが無ければ UnitOfWork の外で、`PartitionReader.installation()` の `declaredTokenIndex` で呼ぶ
- 「並行した二つの UnitOfWork」は、並行に実行する二つの `run` を指す
- W、V は、互いに違うワークスペースの ID。`IdGenerator.generate` の値から作る。ワークスペースの区画に何も確定していなくてよい。索引は、区画のトークンを読まない
- d1、d2、d3 は、互いに違う照合値。`TokenValueDigest.create` で作る。「(d, W)」は、照合値 d とワークスペース W の項目を指す
- 同じ区画のほかの書き込みには、`accountRepository` を使う
- 各ケースは、索引に項目が無い状態から始める

## register

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-declaredTokenIndex-01 | なし | `register([(d1, W), (d2, W), (d3, V)])` を一回呼び、確定する | `findByDigest` が、d1 と d2 に W、d3 に V を返す |
| TC-ports-declaredTokenIndex-03 | なし | `register([])` を呼び、確定する | `run` が確定する。何も登録しない |
| TC-ports-declaredTokenIndex-04 | なし | `register([(d1, W)])` を呼んだ後、`fn` が例外を投げる | `findByDigest(d1)` が `null` を返す |
| TC-ports-declaredTokenIndex-05 | (d1, W) が登録済み | `register([(d2, V), (d1, V)])` を呼ぶ | `run` が `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）で失敗する。`findByDigest(d1)` が W を、`findByDigest(d2)` が `null` を返す |
| TC-ports-declaredTokenIndex-07 | なし | `register([(d1, W), (d1, V)])` を呼ぶ | `run` が `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）で失敗する。`findByDigest(d1)` が `null` を返す |
| TC-ports-declaredTokenIndex-08 | なし | 並行した二つの UnitOfWork が、`register([(d1, W)])` と `register([(d1, V)])` をそれぞれ呼ぶ | 一方の `run` だけが確定し、他方が `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）で失敗する。`findByDigest(d1)` が、確定した側のワークスペースを返す |
| TC-ports-declaredTokenIndex-09 | (d1, W) が登録済み | `register([(d1, W), (d2, W)])` を呼び、確定する | `run` が確定する。`findByDigest` が、d1 と d2 に W を返す |
| TC-ports-declaredTokenIndex-10 | なし | `register([(d1, W), (d1, W)])` を呼び、確定する | `run` が確定する。`findByDigest(d1)` が W を返す。`removeByWorkspace(W)` は 1 を返し、項目は一つ |
| TC-ports-declaredTokenIndex-11 | なし | 並行した二つの UnitOfWork が、どちらも `register([(d1, W)])` を呼ぶ | 二つの `run` がどちらも確定する。`findByDigest(d1)` が W を返す。`removeByWorkspace(W)` は 1 を返し、項目は一つ |
| TC-ports-declaredTokenIndex-12 | (d1, W) を登録し、`removeByWorkspace(W)` を確定している | `register([(d1, V)])` を呼び、確定する | `run` が確定する。`findByDigest(d1)` が V を返す |
| TC-ports-declaredTokenIndex-13 | なし | 一つの UnitOfWork で、`register([(d1, W)])` と、アカウント X の `accountRepository.insert` を呼び、確定する | `findByDigest(d1)` が W を、`accountRepository.findById(X.id)` が X を返す |
| TC-ports-declaredTokenIndex-14 | (d1, W) が登録済み | 一つの UnitOfWork で、アカウント X の `accountRepository.insert` と、`register([(d1, V)])` を呼ぶ | `run` が `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）で失敗する。`accountRepository.findById(X.id)` が `null` を返す |
| TC-ports-declaredTokenIndex-15 | なし | 照合値が互いに違う 2,000 件の項目を、一回の `register` に渡し、確定する | `run` が確定する。どの照合値の `findByDigest` も、渡したワークスペースを返す |

## findByDigest

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-declaredTokenIndex-16 | (d1, W) と (d2, V) が登録済み | `findByDigest(d2)` と、登録していない照合値 d3 の `findByDigest` を呼ぶ | d2 に V、d3 に `null` を返す |
| TC-ports-declaredTokenIndex-18 | W の区画に何も確定していない。(d1, W) が登録済み | `findByDigest(d1)` を呼ぶ | W を返す |
| TC-ports-declaredTokenIndex-19 | UnitOfWork で `register([(d1, W)])` を確定した直後 | `findByDigest(d1)` を呼ぶ | W を返す |

## removeByWorkspace

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-declaredTokenIndex-20 | W の項目が無い。または、(d1, W) を登録し、`removeByWorkspace(W)` を確定している | `removeByWorkspace(W)` を呼び、確定する | 0 を返す |
| TC-ports-declaredTokenIndex-21 | (d1, W)、(d2, W)、(d3, V) が登録済み | `removeByWorkspace(W)` を呼び、確定する | 2 を返す。`findByDigest` が、d1 と d2 に `null`、d3 に V を返す |
| TC-ports-declaredTokenIndex-23 | (d1, W) が登録済み | 一つの UnitOfWork で、`removeByWorkspace(W)` と、アカウント X の `accountRepository.insert` を呼んだ後、`fn` が例外を投げる | `findByDigest(d1)` が W を、`accountRepository.findById(X.id)` が `null` を返す |
| TC-ports-declaredTokenIndex-24 | W の項目が 2,000 件登録済み | `removeByWorkspace(W)` を一度呼び、確定する | 2,000 を返す |
| TC-ports-declaredTokenIndex-25 | (d1, W) が登録済み | 一つの UnitOfWork で、`removeByWorkspace(W)` と、アカウント X の `accountRepository.insert` を呼び、確定する | `findByDigest(d1)` が `null` を、`accountRepository.findById(X.id)` が X を返す |
