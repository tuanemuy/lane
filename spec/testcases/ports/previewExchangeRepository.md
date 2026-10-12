# PreviewExchangeRepository

契約は [token.md](../../domains/token.md) の「ポート」と「トランザクション境界」、[common.md](../../domains/common.md) の `TransactionalRepository` と `UnitOfWorkProvider` が定める。

## 前提

- 引き換えは `PreviewExchange.create` で作り、`PreviewExchange.redeem` で変える
- 書き込み（`insert` / `save` / `deleteLapsed` / `deleteAll`）は `UnitOfWorkProvider.run` の中で、文脈のリポジトリで呼ぶ。`WriteScope` は、そのワークスペースの `workspace` で、条件は `none`。断りが無ければ、一つの操作を一つの UnitOfWork で確定し、読み取りは UnitOfWork の外で呼ぶ
- 「保存済み」は、`insert` を確定した状態を指す。`v` は、直前の読み取りが返した `expectedVersion` を指す
- 「並行して」は、二つの UnitOfWork を同時に始め、どちらも確定まで進めることを指す
- 競合と違反は、メソッドの呼び出しか確定のどちらかで現れる。`run` が `ConflictError` を投げることと、その UnitOfWork の書き込みが一つも反映されないことで確かめる
- ID の順は、文字列のコードポイントの昇順
- リポジトリは、ワークスペースの区画に結び付き、メソッドはワークスペースの ID を引数に取らない。W1、W2 は別のワークスペース。引き換えは、ワークスペースの ID のフィールドを持たない。「W1 の X」は、W1 の区画に保存した X を指す。断りの無い呼び出しは、W1 の区画のリポジトリに対して行う。W2 の引き換えの書き込みと、「W2 の区画」と書いた呼び出しは、W2 の区画のリポジトリに対して行う
- e1 < e2 < e3 は、固定の日時
- 「削除してある」は、`deleteLapsed` または `deleteAll` を確定して、その引き換えを削除した状態を指す
- トークンの `insert` と読み取りは、同じ区画の `TokenRepository` で行う

## insert

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-previewExchangeRepository-01 | 引き換えが無い | `pending` の引き換え X を `insert` する | `findById(X.id)` が X と等しいエンティティを返す |
| TC-ports-previewExchangeRepository-02 | X が保存済み | X と同じ ID で、照合値が別の引き換えを `insert` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。`findById(X.id)` は X を返す |
| TC-ports-previewExchangeRepository-03 | W1 の X が保存済み（`redeem` して保存した X でも同じ） | X と同じ照合値で、ID が別の引き換え Y を `insert` する | `run` が `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）を投げる。`findById(Y.id)` は `null` |
| TC-ports-previewExchangeRepository-04 | W1 の X が保存済み | W2 の区画で、X と同じ照合値で、ID が別の引き換え Y を `insert` する | `run` が確定する。W1 の区画の `findByCodeDigest` は X を、W2 の区画の `findByCodeDigest` は Y を返す |
| TC-ports-previewExchangeRepository-06 | X を保存し、削除してある | X と同じ照合値で、ID が別の引き換え Y を `insert` する | 確定する。`findByCodeDigest` が Y を返す |
| TC-ports-previewExchangeRepository-07 | 引き換えが無い | 照合値が同じで ID が別の X と Y を、並行して `insert` する | 一方だけが確定し、他方の `run` は `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）を投げる。`findByCodeDigest` は確定した方を返す |
| TC-ports-previewExchangeRepository-08 | 引き換えが無い | 一つの UnitOfWork で、照合値が同じで ID が別の X と Y を `insert` する | `run` が `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）を投げる。`findById(X.id)` と `findById(Y.id)` は `null` |
| TC-ports-previewExchangeRepository-09 | W1 の X が保存済み | W2 の区画で、X と同じ ID で、照合値が別の引き換え Y を `insert` する | `run` が確定する。W1 の区画の `findById(X.id)` は X を、W2 の区画の `findById(X.id)` は Y を返す |

## findById

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-previewExchangeRepository-10 | 引き換えが無い | 未使用の ID で `findById` を呼ぶ | `null` を返す |
| TC-ports-previewExchangeRepository-11 | X を `redeem` して保存してある | `findById(X.id)` を呼ぶ | `status` が `redeemed` で、`redeemedAt` と `tokenId` を持つエンティティを返す |
| TC-ports-previewExchangeRepository-12 | X を保存し、削除してある | `findById(X.id)` と `findByCodeDigest(X.codeDigest)` を呼ぶ | どちらも `null` を返す |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-previewExchangeRepository-13 | X が保存済み。`findById` で X と `v` を得ている | `save(PreviewExchange.redeem(X, tokenId, now), v)` を呼ぶ | `findById(X.id)` が、`status` が `redeemed`、`redeemedAt` が `now`、`tokenId` が渡した ID のエンティティを返す |
| TC-ports-previewExchangeRepository-14 | X が保存済み。`findById` で X と `v` を得た後、別の UnitOfWork が X の `redeem` を `save` して確定した | 同じ `v` で、別の `tokenId` の `redeem` を `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById(X.id)` の `tokenId` は、先に確定した方 |
| TC-ports-previewExchangeRepository-15 | X が保存済み。二つの呼び出し側が、同じ `v` を得ている | それぞれの UnitOfWork で、`redeem` の `save` と、発行したトークンの `insert` を、並行して呼ぶ | 一方だけが確定し、他方の `run` は `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById(X.id)` の `tokenId` は確定した方のトークン。確定しなかった方のトークンは、`TokenRepository.findById` が `null` |
| TC-ports-previewExchangeRepository-16 | X が保存済み。`findById` で X と `v` を得た後、別の UnitOfWork が `deleteAll(10)` を確定した | `v` で `redeem` の結果を `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById(X.id)` は `null` |

## findByCodeDigest

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-previewExchangeRepository-17 | 引き換えが無い | 未使用の照合値で `findByCodeDigest` を呼ぶ | `null` を返す |
| TC-ports-previewExchangeRepository-18 | `pending` の X と、照合値が別の Y が保存済み | `findByCodeDigest(X.codeDigest)` を呼ぶ | X と `expectedVersion` を返す |
| TC-ports-previewExchangeRepository-20 | `redeem` して保存した X と、`expiresAt` が過去の日時になる Y が保存済み | それぞれの照合値で `findByCodeDigest` を呼ぶ | どちらも、その状態のまま引き換えを返す |
| TC-ports-previewExchangeRepository-22 | W2 の X が保存済み | W1 の区画で `findByCodeDigest(X.codeDigest)` を呼ぶ | `null` を返す。ほかのワークスペースの引き換えを返さない |
| TC-ports-previewExchangeRepository-23 | X が保存済み | `findByCodeDigest` が返した `expectedVersion` で、`redeem` の結果を `save` する | 確定する。`findById(X.id)` は `redeemed` のエンティティを返す |

## deleteLapsed

保持期間の起点は `PreviewExchange.lapsedAt`。対象と順序は [common.md](../../domains/common.md) の「期限つきの一度きりの値」が定める。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-previewExchangeRepository-24 | 引き換えが無い | `deleteLapsed(e2, 10)` を呼ぶ | 0 を返す |
| TC-ports-previewExchangeRepository-25 | `expiresAt` が e1、e2、e3 の X、Y、Z | `deleteLapsed(e2, 10)` を呼ぶ | 2 を返す。`findById` は X と Y に `null`、Z に Z を返す |
| TC-ports-previewExchangeRepository-28 | `expiresAt` が e1 の、`pending` の X と `redeemed` の Y | `deleteLapsed(e2, 10)` を呼ぶ | 2 を返す。どちらも `findById` は `null` |
| TC-ports-previewExchangeRepository-29 | `expiresAt` が e1 の、W1 の X と W2 の Y | W1 の区画で `deleteLapsed(e2, 10)` を呼ぶ | 1 を返す。W1 の区画の `findById(X.id)` は `null`。W2 の区画の `findById(Y.id)` は Y を返す |
| TC-ports-previewExchangeRepository-31 | `expiresAt` が e1、e2、e3 の X、Y、Z | `deleteLapsed(e3, 2)` を呼ぶ | 2 を返す。`findById` は X と Y に `null`、Z に Z を返す |
| TC-ports-previewExchangeRepository-32 | `expiresAt` が e1 の引き換えが 3 件 | `deleteLapsed(e2, 2)` を呼ぶ | 2 を返す。`findById` は、ID の昇順の先頭 2 件に `null`、3 件目に引き換えを返す |
| TC-ports-previewExchangeRepository-33 | `expiresAt` が e3 以前の引き換えが 4 件 | `deleteLapsed(e3, 3)` を三度続けて、別の UnitOfWork で呼ぶ | 順に 3、1、0 を返す。4 件とも `findById` は `null` |
| TC-ports-previewExchangeRepository-35 | `expiresAt` が e1 の X を `redeem` し、発行したトークン T を同じ UnitOfWork で `insert` してある | `deleteLapsed(e2, 10)` を呼ぶ | 1 を返す。`TokenRepository.findById(T.id)` は T を返す |
| TC-ports-previewExchangeRepository-37 | `expiresAt` が e1 の引き換えが 2 件 | `deleteLapsed(e2, BoundedDeletion.maxLimit)` を呼ぶ | 2 を返す |
| TC-ports-previewExchangeRepository-38 | `expiresAt` が e1 の X | `deleteLapsed(e2, 0)` と、`BoundedDeletion.maxLimit` を超える `limit` の `deleteLapsed(e2, limit)` を、それぞれ呼ぶ | どちらも `SystemError`（理由 `INVALID_QUERY`）になる。`findById(X.id)` は X を返す |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-previewExchangeRepository-40 | W1 に引き換えが無い | `deleteAll(10)` を呼ぶ | 0 を返す |
| TC-ports-previewExchangeRepository-41 | W1 に、`pending` の 1 件と `redeemed` の 1 件 | `deleteAll(10)` を呼ぶ | 2 を返す。どちらも `findById` は `null` |
| TC-ports-previewExchangeRepository-42 | W1 に 4 件 | `deleteAll(3)` を三度続けて、別の UnitOfWork で呼ぶ | 順に 3、1、0 を返す。4 件とも `findById` は `null` |
| TC-ports-previewExchangeRepository-44 | W1 に 2 件、W2 に 1 件 | `deleteAll(10)` を呼ぶ | 2 を返す。W2 の 1 件は、W2 の区画の `findById` が返る |
| TC-ports-previewExchangeRepository-46 | W1 に 2 件 | `deleteAll(BoundedDeletion.maxLimit)` を呼ぶ | 2 を返す |
| TC-ports-previewExchangeRepository-47 | W1 に 2 件 | `deleteAll(0)` と、`BoundedDeletion.maxLimit` を超える `limit` の `deleteAll` を、それぞれ呼ぶ | どちらも `SystemError`（理由 `INVALID_QUERY`）になる。2 件とも `findById` が返る |

## UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-previewExchangeRepository-49 | X が保存済み | 一つの UnitOfWork で、X の `redeem` の `save` と、発行したトークン T の `TokenRepository.insert` を呼び、確定する | `findById(X.id)` は `redeemed` の X、`TokenRepository.findById(T.id)` は T を返す |
| TC-ports-previewExchangeRepository-50 | X が保存済み | 一つの UnitOfWork で、同じ 2 つの書き込みを呼んだ後、`fn` が例外を投げる | `findById(X.id)` は `pending` の X、`TokenRepository.findById(T.id)` は `null` を返す |
| TC-ports-previewExchangeRepository-51 | X が保存済み。`findById` で `v` を得た後、別の UnitOfWork が X の `redeem` を確定した | 一つの UnitOfWork で、新しい引き換え Y の `insert` と、`v` での X の `save` を呼ぶ | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById(Y.id)` は `null` |
| TC-ports-previewExchangeRepository-52 | `expiresAt` が e1 の引き換えが 2 件 | 一つの UnitOfWork の中で、`deleteLapsed(e2, 10)` を呼んだ後、`fn` が例外を投げる | 2 件とも `findById` が返る |
| TC-ports-previewExchangeRepository-53 | W1 に 2 件 | 一つの UnitOfWork の中で、`deleteAll(10)` を呼んだ後、`fn` が例外を投げる | 2 件とも `findById` が返る |
