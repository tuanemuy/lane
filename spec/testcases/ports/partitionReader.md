# PartitionReader

`PartitionReader` の契約（[../../domains/common.md](../../domains/common.md) の「UnitOfWorkProvider」）を、すべてのアダプターが通すケースで定める。

- 書き込みは `UnitOfWorkProvider.run` の UnitOfWork の中で呼ぶ。「設置の UnitOfWork」「W の UnitOfWork」の書き方は [unitOfWorkProvider.md](unitOfWorkProvider.md) に従う。条件を書かない W の UnitOfWork は `none`
- 読み取りには、設置の区画は `accountRepository`、ワークスペースの区画は `workspaceRepository` と `memberRepository` を使う
- W、X は、互いに違う確定済みのワークスペース。メンバーは、互いに違うアカウントの ID で作る
- ID は `IdGenerator.generate` の値を使う
- 返るリポジトリが書き込みのメソッドを持たないことは、型が定める。ケースにしない
- `workspace(W)` が公開する `quarantinedEventQuery` のケースは、[quarantinedEventQuery.md](quarantinedEventQuery.md) が持つ

## installation

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-partitionReader-01 | なし | `installation()` の `accountRepository.findById` を、保存していない ID で呼ぶ | `null` を返す |
| TC-ports-partitionReader-02 | 設置の UnitOfWork で `accountRepository.insert(A)` を確定している | `installation()` の `accountRepository.findById(A.id)` を呼ぶ | A と `expectedVersion` を返す |
| TC-ports-partitionReader-03 | 設置の UnitOfWork で `accountRepository.insert(A)` を呼んだ後、`fn` が例外を投げている | `installation()` の `accountRepository.findById(A.id)` を呼ぶ | `null` を返す |
| TC-ports-partitionReader-04 | `installation()` を呼んでリポジトリを得た後に、設置の UnitOfWork で `accountRepository.insert(A)` を確定している | 先に得たリポジトリの `findById(A.id)` を呼ぶ | A を返す |
| TC-ports-partitionReader-05 | アカウント A が確定済みで、`installation()` の `findById` で `expectedVersion` v を得ている | 設置の UnitOfWork で A を v で `save` し、確定する | `run` が確定する |

## workspace

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-partitionReader-06 | ワークスペース W、X が確定済み | `workspace(W)` の `workspaceRepository.find()` と、`workspace(X)` の `workspaceRepository.find()` を呼ぶ | W と X を、それぞれの `expectedVersion` とともに返す |
| TC-ports-partitionReader-08 | W の UnitOfWork で `memberRepository.insert(M)` を確定している | `workspace(W)` の `memberRepository.findById(M.id)` を呼ぶ | M を返す |
| TC-ports-partitionReader-09 | W の UnitOfWork で `memberRepository.insert(M)` を確定している | `workspace(X)` の `memberRepository.findById(M.id)` と `findPage` を呼ぶ | `findById` が `null` を、`findPage` が空の結果と件数 0 を返す |
| TC-ports-partitionReader-11 | W の UnitOfWork で `memberRepository.insert(M)` を呼んだ後、`fn` が例外を投げている | `workspace(W)` の `memberRepository.findById(M.id)` を呼ぶ | `null` を返す |
| TC-ports-partitionReader-12 | 区画に何も確定していないワークスペースの ID v | `workspace(v)` を呼び、`workspaceRepository.find()` と `memberRepository.findPage` を呼ぶ | `workspace(v)` はエラーにならない。`find` が `null` を、一覧が空の結果と件数 0 を返す |
| TC-ports-partitionReader-13 | W のメンバー M が確定済みで、`workspace(W)` の `findById` で `expectedVersion` v を得ている | W の UnitOfWork で M を v で `save` し、確定する | `run` が確定する |

## UnitOfWork の中の読み取り

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-partitionReader-15 | アカウント A と、W のメンバー M が確定済み | W の UnitOfWork の `fn` の中で `installation()` の `accountRepository.findById(A.id)` を呼ぶ。設置の UnitOfWork と X の UnitOfWork の `fn` の中で、それぞれ `workspace(W)` の `memberRepository.findById(M.id)` を呼ぶ | それぞれ A と M を返す。どの `run` も確定する |
