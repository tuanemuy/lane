# SessionRepository

「上限」は、その節のメソッドの契約が定める、一度に渡せる件数・個数、または `limit` の上限を指す。値は、ポートを定めるドメインのファイルが持つ。

`SessionRepository` の契約（[../../domains/identity.md](../../domains/identity.md)）と、`TransactionalRepository` の基底の契約（[../../domains/common.md](../../domains/common.md)）を、すべてのアダプターが通すケースで定める。

- 書き込みは `UnitOfWorkProvider.run` の UnitOfWork の中で呼ぶ。`WriteScope` は `{ kind: "installation" }`。「確定する」は、`fn` が正常に終わり `run` が値を返すことを指す。「失敗する」は、`run` がそのエラーを投げることを指す
- 読み取りは、断りが無ければ UnitOfWork の外で呼ぶ
- 「並行した二つの UnitOfWork」は、並行に実行する二つの `run` を指す。楽観ロックの `expectedVersion` は、二つの `run` の前に読む
- セッションは `Session.start`（`active`）、`Session.startPendingMfa`（`pending_mfa`）、`Session.completeMfa` で作る。`expiresAt` は、渡す `now` で決める。ID は `IdGenerator.generate` の値を使う
- セッションのアカウントは、断りが無ければ、`Account.create` で作り、同じ設置の区画の `accountRepository.insert` で確定済みとする
- `deleteLapsed` は、一回の呼び出しごとに一つの UnitOfWork で確定する

## insert

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-sessionRepository-01 | なし | `active` のセッション S を `insert` し、確定する | `findById(S.id)` と `findByTokenDigest(S.tokenDigest)` が、S と全フィールドが一致するセッションを返す |
| TC-ports-sessionRepository-02 | なし | `pending_mfa` のセッション S を `insert` し、確定する | `findById(S.id)` が、`status` が `pending_mfa` で、ほかのフィールドも S と一致するセッションを返す |
| TC-ports-sessionRepository-03 | なし | S を `insert` した後、`fn` が例外を投げる | `findById(S.id)` と `findByTokenDigest(S.tokenDigest)` が `null` を返す |
| TC-ports-sessionRepository-04 | S が確定済み | S と同じ ID で、照合値が違うセッション T を `insert` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する。`findByTokenDigest(T.tokenDigest)` が `null` を返す |
| TC-ports-sessionRepository-05 | S が確定済み | S と ID が違い、照合値が同じセッション T を `insert` する | `run` が `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）で失敗する。`findById(T.id)` が `null` を返す |
| TC-ports-sessionRepository-06 | なし | 並行した二つの UnitOfWork が、照合値が同じセッション S、T をそれぞれ `insert` する | 一方の `run` だけが確定し、他方が `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）で失敗する |
| TC-ports-sessionRepository-07 | 同じアカウントのセッション S が確定済み | 同じアカウントで、照合値が違うセッション T を `insert` し、確定する | `findById(S.id)` が S を、`findById(T.id)` が T を返す |
| TC-ports-sessionRepository-08 | D は、`accountRepository` に確定していないアカウントの ID | `accountId` が D のセッション S を `insert` する | `run` が `ConflictError`（理由 `ACCOUNT_MISSING`）で失敗する。`findById(S.id)` と `findByTokenDigest(S.tokenDigest)` が `null` を返す |
| TC-ports-sessionRepository-09 | なし | 一つの UnitOfWork で、`accountRepository.insert` でアカウント E を保存し、`accountId` が E のセッション S を `insert` して、確定する | `run` が確定する。`findById(S.id)` が S を返す |

## findById

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-sessionRepository-10 | なし | 保存していない ID で `findById` を呼ぶ | `null` を返す |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-sessionRepository-11 | `pending_mfa` の S が確定済みで、`findByTokenDigest` で `expectedVersion` v を得ている | `Session.completeMfa` を適用した S を v で `save` し、確定する | `findById(S.id)` が、`status` が `active` で、`authenticatedAt` と `expiresAt` が保存した値と一致するセッションを返す |
| TC-ports-sessionRepository-12 | S を読んで v を得た後、別の UnitOfWork が S を `save` して確定している | 変更した S を v で `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）で失敗する。`findById(S.id)` が、別の UnitOfWork が保存した内容を返す |
| TC-ports-sessionRepository-13 | `pending_mfa` の S が確定済み | 並行した二つの UnitOfWork が、同じ `expectedVersion` で、`Session.completeMfa` を適用した S をそれぞれ `save` する | 一方の `run` だけが確定し、他方が `ConflictError`（理由 `STALE_VERSION`）で失敗する |
| TC-ports-sessionRepository-14 | `pending_mfa` の S が確定済みで、v を得ている | 変更した S を v で `save` した後、`fn` が例外を投げる | `findById(S.id)` が `pending_mfa` の S を返す |
| TC-ports-sessionRepository-15 | S を読んで v を得た後、別の UnitOfWork が S を `delete` して確定している | 変更した S を v で `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）で失敗する。`findById(S.id)` が `null` を返す |

## delete

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-sessionRepository-16 | S が確定済みで、`findByTokenDigest` で v を得ている | `delete(S.id, v)` を呼び、確定する | `findByTokenDigest(S.tokenDigest)` と `findById(S.id)` が `null` を返す |
| TC-ports-sessionRepository-17 | S を読んで v を得た後、別の UnitOfWork が S を `save` して確定している | `delete(S.id, v)` を呼ぶ | `run` が `ConflictError`（理由 `STALE_VERSION`）で失敗する。`findById(S.id)` が S を返す |
| TC-ports-sessionRepository-18 | S が確定済みで、v を得ている | `delete(S.id, v)` を呼んだ後、`fn` が例外を投げる | `findByTokenDigest(S.tokenDigest)` が S を返す |
| TC-ports-sessionRepository-19 | S、T が確定済み | S を `delete` し、確定する | `findById(T.id)` が T を返す |
| TC-ports-sessionRepository-20 | S を読んで v を得た後、別の UnitOfWork が S を `delete` して確定している | `delete(S.id, v)` を呼ぶ | `run` が `ConflictError`（理由 `STALE_VERSION`）で失敗する |

## findByTokenDigest

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-sessionRepository-21 | なし | 保存していない照合値で `findByTokenDigest` を呼ぶ | `null` を返す |
| TC-ports-sessionRepository-22 | 照合値が違う S、T が確定済み | `findByTokenDigest(S.tokenDigest)` を呼ぶ | S と `expectedVersion` を返す |
| TC-ports-sessionRepository-23 | `expiresAt` が現在時刻より前の `active` のセッション S と `pending_mfa` のセッション T が確定済み | `findByTokenDigest(S.tokenDigest)` と `findByTokenDigest(T.tokenDigest)` を呼ぶ | S と T を返す |

## deleteLapsed

`before` を日時 b とする。保持期間の起点（`lapsedAt`）は `expiresAt`。対象と順序は [../../domains/common.md](../../domains/common.md) の「期限つきの一度きりの値」が定める。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-sessionRepository-26 | セッションが無い | `deleteLapsed(b, 10)` を呼び、確定する | 0 を返す |
| TC-ports-sessionRepository-27 | `expiresAt` が b の 1 ミリ秒前のセッション S が確定済み | `deleteLapsed(b, 10)` を呼び、確定する | 1 を返す。`findById(S.id)` と `findByTokenDigest(S.tokenDigest)` が `null` を返す |
| TC-ports-sessionRepository-28 | `expiresAt` が b と同じセッション S が確定済み | `deleteLapsed(b, 10)` を呼び、確定する | 1 を返す。`findById(S.id)` が `null` を返す |
| TC-ports-sessionRepository-29 | `expiresAt` が b の 1 ミリ秒後のセッション S が確定済み | `deleteLapsed(b, 10)` を呼び、確定する | 0 を返す。`findById(S.id)` が S を返す |
| TC-ports-sessionRepository-30 | `expiresAt` が b より前の `active` のセッションと `pending_mfa` のセッションが 1 件ずつ、`expiresAt` が b より後のセッションが 1 件確定済み | `deleteLapsed(b, 10)` を呼び、確定する | 2 を返す。b より前の 2 件の `findById` が `null` を、b より後の 1 件がそのセッションを返す |
| TC-ports-sessionRepository-31 | `expiresAt` が b より前の `pending_mfa` のセッション S に、`Session.completeMfa` を適用して `save` し、確定している。`save` した S の `expiresAt` は b より後 | `deleteLapsed(b, 10)` を呼び、確定する | 0 を返す。`findById(S.id)` が `active` の S を返す |
| TC-ports-sessionRepository-33 | `expiresAt` が b より前のセッションが 1 件確定済み | `deleteLapsed(b, 1)` を呼び、確定する | 1 を返す。そのセッションの `findById` が `null` を返す |
| TC-ports-sessionRepository-34 | `expiresAt` が b より前のセッションが、上限の件数だけ確定済み | `deleteLapsed`（b、上限の `limit`）を呼び、確定する | 上限の件数を返す |
| TC-ports-sessionRepository-35 | `expiresAt` が互いに違い、どれも b より前のセッションが 3 件確定済み | `deleteLapsed(b, 2)` を呼び、確定する | 2 を返す。`expiresAt` の昇順で先頭の 2 件の `findById` が `null` を、3 件目がそのセッションを返す |
| TC-ports-sessionRepository-36 | `expiresAt` が同じで、b より前のセッションが 3 件確定済み | `deleteLapsed(b, 2)` を呼び、確定する | 2 を返す。ID の昇順で先頭の 2 件の `findById` が `null` を、3 件目がそのセッションを返す |
| TC-ports-sessionRepository-37 | `expiresAt` が b より前のセッションが 3 件確定済み | `deleteLapsed(b, 2)` を三回、それぞれ確定する | 順に 2、1、0 を返す。三回目の後、3 件の `findById` が `null` を返す |
| TC-ports-sessionRepository-40 | `expiresAt` が b より前のセッションが、上限より 1 件多く確定済み | `deleteLapsed`（b、上限の `limit`）を二回、それぞれ確定する | 順に、上限の件数、1 を返す |
| TC-ports-sessionRepository-41 | `expiresAt` が b より前のセッション S が確定済み | `deleteLapsed(b, 0)` を呼ぶ | `run` が `SystemError`（理由 `INVALID_QUERY`）で失敗する。`findById(S.id)` が S を返す |
| TC-ports-sessionRepository-42 | `expiresAt` が b より前のセッション S が確定済み | `deleteLapsed`（b、上限より 1 大きい `limit`）を呼ぶ | `run` が `SystemError`（理由 `INVALID_QUERY`）で失敗する。`findById(S.id)` が S を返す |
| TC-ports-sessionRepository-43 | `expiresAt` が b より前のセッション S を読んで v を得た後、別の UnitOfWork が S を `save` して確定している | `deleteLapsed(b, 10)` を呼び、確定する | 1 を返す。`findById(S.id)` が `null` を返す |
| TC-ports-sessionRepository-44 | `expiresAt` が b より前のセッション S が確定済み | `deleteLapsed(b, 10)` を呼んだ後、`fn` が例外を投げる | `findById(S.id)` が S を返す |
