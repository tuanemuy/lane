# RecoveryRequestRepository

「上限」は、その節のメソッドの契約が定める、一度に渡せる件数・個数、または `limit` の上限を指す。値は、ポートを定めるドメインのファイルが持つ。

`RecoveryRequestRepository` の契約（[../../domains/identity.md](../../domains/identity.md)）と、`TransactionalRepository` の基底の契約（[../../domains/common.md](../../domains/common.md)）を、すべてのアダプターが通すケースで定める。

- 書き込みは `UnitOfWorkProvider.run` の UnitOfWork の中で呼ぶ。`WriteScope` は `{ kind: "installation" }`。「確定する」は、`fn` が正常に終わり `run` が値を返すことを指す。「失敗する」は、`run` がそのエラーを投げることを指す
- 読み取りは、断りが無ければ UnitOfWork の外で呼ぶ
- 「並行した二つの UnitOfWork」は、並行に実行する二つの `run` を指す。楽観ロックの `expectedVersion` は、二つの `run` の前に読む
- 回復依頼は `RecoveryRequest.issue`、`redeem`、`revoke` で作る。`expiresAt` は、渡す `now` と `channel` で決める。ID は `IdGenerator.generate` の値を使う
- 回復依頼のアカウントは、断りが無ければ、`Account.create` で作り、同じ設置の区画の `accountRepository.insert` で確定済みとする
- 断りの無い回復依頼は、`channel` が `email`
- `deleteLapsed` は、一回の呼び出しごとに一つの UnitOfWork で確定する

## insert

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-recoveryRequestRepository-01 | なし | `channel` が `email` の `pending` の回復依頼 R を `insert` し、確定する | `findById(R.id)`、`findByTokenDigest(R.tokenDigest)`、`findPendingByAccount(R.accountId, R.channel)` が、R と全フィールドが一致する回復依頼を返す |
| TC-ports-recoveryRequestRepository-02 | なし | `channel` が `operator` の `pending` の回復依頼 R を `insert` し、確定する | `findById(R.id)` が、`channel` が `operator` で、ほかのフィールドも R と一致する回復依頼を返す |
| TC-ports-recoveryRequestRepository-03 | なし | R を `insert` した後、`fn` が例外を投げる | `findById(R.id)`、`findByTokenDigest(R.tokenDigest)`、`findPendingByAccount(R.accountId, R.channel)` が `null` を返す |
| TC-ports-recoveryRequestRepository-04 | R が確定済み | R と同じ ID で、アカウントと照合値が違う回復依頼 Q を `insert` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する。`findByTokenDigest(Q.tokenDigest)` が `null` を返す |
| TC-ports-recoveryRequestRepository-05 | アカウント X の R が確定済み | 別のアカウント Y の、R と照合値が同じ回復依頼 Q を `insert` する | `run` が `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）で失敗する。`findById(Q.id)` が `null` を返す |
| TC-ports-recoveryRequestRepository-06 | なし | 並行した二つの UnitOfWork が、アカウントが違い、照合値が同じ回復依頼 R、Q をそれぞれ `insert` する | 一方の `run` だけが確定し、他方が `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）で失敗する |
| TC-ports-recoveryRequestRepository-07 | X の `revoked` の R が確定済み | X の、R と照合値が同じ `pending` の回復依頼 Q を `insert` する | `run` が `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）で失敗する |
| TC-ports-recoveryRequestRepository-08 | X の `pending` の R が確定済み | X の、照合値が違う `pending` の回復依頼 Q を `insert` する | `run` が `ConflictError`（理由 `PENDING_RECOVERY_EXISTS`）で失敗する。`findById(Q.id)` が `null` を返す。`findPendingByAccount(X, "email")` が R を返す |
| TC-ports-recoveryRequestRepository-09 | X の、`expiresAt` が現在時刻より前の `pending` の R が確定済み | X の `pending` の回復依頼 Q を `insert` する | `run` が `ConflictError`（理由 `PENDING_RECOVERY_EXISTS`）で失敗する |
| TC-ports-recoveryRequestRepository-10 | なし | 一つの UnitOfWork で、X の `pending` の回復依頼 R、Q を `insert` する | `run` が `ConflictError`（理由 `PENDING_RECOVERY_EXISTS`）で失敗する。`findById(R.id)` と `findById(Q.id)` が `null` を返す |
| TC-ports-recoveryRequestRepository-11 | X の回復依頼が無い | 並行した二つの UnitOfWork が、X の `pending` の回復依頼 R、Q をそれぞれ `insert` する | 一方の `run` だけが確定し、他方が `ConflictError`（理由 `PENDING_RECOVERY_EXISTS`）で失敗する。`findPendingByAccount(X, "email")` が、確定した側の回復依頼を返す |
| TC-ports-recoveryRequestRepository-12 | X の `pending` の R が確定済みで、`findPendingByAccount` で `expectedVersion` v を得ている | 一つの UnitOfWork で、`RecoveryRequest.revoke` を適用した R を v で `save` し、X の新しい `pending` の回復依頼 Q を `insert` して確定する | `run` が確定する。`findById(R.id)` が `revoked` の R を返す。`findPendingByAccount(X, "email")` が Q を返す |
| TC-ports-recoveryRequestRepository-13 | X の `used` の回復依頼と `revoked` の回復依頼が 1 件ずつ確定済み | X の `pending` の回復依頼 Q を `insert` し、確定する | `run` が確定する。`findPendingByAccount(X, "email")` が Q を返す |
| TC-ports-recoveryRequestRepository-14 | X の、`channel` が `email` の `pending` の R が確定済み | X の、`channel` が `operator` の `pending` の回復依頼 Q を `insert` し、確定する | `run` が確定する。一意性は `channel` ごと。`findPendingByAccount(X, "email")` が R を、`findPendingByAccount(X, "operator")` が Q を返す |
| TC-ports-recoveryRequestRepository-15 | X の、`channel` が `operator` の `pending` の R が確定済み | X の、`channel` が `email` の `pending` の回復依頼 Q を `insert` し、確定する | `run` が確定する。`findPendingByAccount(X, "operator")` が R を、`findPendingByAccount(X, "email")` が Q を返す |
| TC-ports-recoveryRequestRepository-16 | X の、`channel` が `operator` の `pending` の R が確定済み | X の、`channel` が `operator` で、照合値が違う `pending` の回復依頼 Q を `insert` する | `run` が `ConflictError`（理由 `PENDING_RECOVERY_EXISTS`）で失敗する |
| TC-ports-recoveryRequestRepository-17 | X の、`channel` が `email` の `pending` の R が確定済み | X の、`channel` が `operator` で、照合値が R と同じ `pending` の回復依頼 Q を `insert` する | `run` が `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）で失敗する |
| TC-ports-recoveryRequestRepository-18 | X の `pending` の R が確定済み | 別のアカウント Y の `pending` の回復依頼 Q を `insert` し、確定する | `run` が確定する。`findPendingByAccount(X, "email")` が R を、`findPendingByAccount(Y, "email")` が Q を返す |
| TC-ports-recoveryRequestRepository-19 | D は、`accountRepository` に確定していないアカウントの ID | `accountId` が D の `pending` の回復依頼 R を `insert` する | `run` が `ConflictError`（理由 `ACCOUNT_MISSING`）で失敗する。`findById(R.id)` と `findByTokenDigest(R.tokenDigest)` が `null` を返す |

## findById

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-recoveryRequestRepository-20 | なし | 保存していない ID で `findById` を呼ぶ | `null` を返す |
| TC-ports-recoveryRequestRepository-21 | `RecoveryRequest.redeem` を適用した R を `save` して確定している | `findById(R.id)` を呼ぶ | `status` が `used` で、`usedAt` が保存した値と一致する回復依頼を返す |
| TC-ports-recoveryRequestRepository-22 | `RecoveryRequest.revoke` を適用した R を `save` して確定している | `findById(R.id)` を呼ぶ | `status` が `revoked` で、`revokedAt` が保存した値と一致する回復依頼を返す |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-recoveryRequestRepository-23 | `pending` の R が確定済みで、`findByTokenDigest` で `expectedVersion` v を得ている | `RecoveryRequest.redeem` を適用した R を v で `save` し、確定する | `findByTokenDigest(R.tokenDigest)` が `used` の R を返す。`findPendingByAccount(R.accountId, R.channel)` が `null` を返す |
| TC-ports-recoveryRequestRepository-24 | R を読んで v を得た後、別の UnitOfWork が R を `save` して確定している | 変更した R を v で `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）で失敗する。`findById(R.id)` が、別の UnitOfWork が保存した内容を返す |
| TC-ports-recoveryRequestRepository-25 | `pending` の R が確定済み | 並行した二つの UnitOfWork が、同じ `expectedVersion` で、`RecoveryRequest.redeem` を適用した R をそれぞれ `save` する | 一方の `run` だけが確定し、他方が `ConflictError`（理由 `STALE_VERSION`）で失敗する。`findById(R.id)` が、確定した側の状態を返す |
| TC-ports-recoveryRequestRepository-27 | `pending` の R が確定済みで、v を得ている | `redeem` を適用した R を v で `save` した後、`fn` が例外を投げる | `findById(R.id)` が `pending` の R を返す |
| TC-ports-recoveryRequestRepository-28 | R を読んで v を得た後、別の UnitOfWork が、R を削除する `deleteLapsed` を確定している | 変更した R を v で `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）で失敗する。`findById(R.id)` が `null` を返す |

## findByTokenDigest

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-recoveryRequestRepository-29 | なし | 保存していない照合値で `findByTokenDigest` を呼ぶ | `null` を返す |
| TC-ports-recoveryRequestRepository-30 | 照合値が違う R、Q が確定済み | `findByTokenDigest(R.tokenDigest)` を呼ぶ | R と `expectedVersion` を返す |
| TC-ports-recoveryRequestRepository-31 | `used` の R と `revoked` の Q が確定済み | `findByTokenDigest(R.tokenDigest)` と `findByTokenDigest(Q.tokenDigest)` を呼ぶ | `used` の R と `revoked` の Q を返す |
| TC-ports-recoveryRequestRepository-33 | `expiresAt` が現在時刻より前の `pending` の R が確定済み | `findByTokenDigest(R.tokenDigest)` を呼ぶ | `pending` の R を返す |

## findPendingByAccount

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-recoveryRequestRepository-34 | アカウント X の回復依頼が無い | `findPendingByAccount(X, "email")` を呼ぶ | `null` を返す |
| TC-ports-recoveryRequestRepository-35 | X の `pending` の R が確定済み | `findPendingByAccount(X, "email")` を呼ぶ | R と `expectedVersion` を返す |
| TC-ports-recoveryRequestRepository-36 | X の、`expiresAt` が現在時刻より前の `pending` の R が確定済み | `findPendingByAccount(X, "email")` を呼ぶ | R を返す |
| TC-ports-recoveryRequestRepository-37 | X の `used` の回復依頼と `revoked` の回復依頼だけが確定済み | `findPendingByAccount(X, "email")` を呼ぶ | `null` を返す |
| TC-ports-recoveryRequestRepository-38 | X の `used` の回復依頼、`revoked` の回復依頼、`pending` の R が確定済み | `findPendingByAccount(X, "email")` を呼ぶ | R を返す |
| TC-ports-recoveryRequestRepository-39 | 別のアカウント Y の `pending` の回復依頼だけが確定済み | `findPendingByAccount(X, "email")` を呼ぶ | `null` を返す |
| TC-ports-recoveryRequestRepository-40 | X の、`channel` が `operator` の `pending` の回復依頼だけが確定済み | `findPendingByAccount(X, "email")` を呼ぶ | `null` を返す。別の `channel` の回復依頼を返さない |
| TC-ports-recoveryRequestRepository-41 | X の、`channel` が `email` の `pending` の R と、`channel` が `operator` の `pending` の Q が確定済み | `findPendingByAccount(X, "operator")` を呼ぶ | Q を返す |

## deleteLapsed

`before` を日時 b とする。保持期間の起点（`lapsedAt`）は `expiresAt`。対象と順序は [../../domains/common.md](../../domains/common.md) の「期限つきの一度きりの値」が定める。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-recoveryRequestRepository-43 | 回復依頼が無い | `deleteLapsed(b, 10)` を呼び、確定する | 0 を返す |
| TC-ports-recoveryRequestRepository-44 | `expiresAt` が b の 1 ミリ秒前の回復依頼 R が確定済み | `deleteLapsed(b, 10)` を呼び、確定する | 1 を返す。`findById(R.id)`、`findByTokenDigest(R.tokenDigest)`、`findPendingByAccount(R.accountId, R.channel)` が `null` を返す |
| TC-ports-recoveryRequestRepository-45 | `expiresAt` が b と同じ回復依頼 R が確定済み | `deleteLapsed(b, 10)` を呼び、確定する | 1 を返す。`findById(R.id)` が `null` を返す |
| TC-ports-recoveryRequestRepository-46 | `expiresAt` が b の 1 ミリ秒後の回復依頼 R が確定済み | `deleteLapsed(b, 10)` を呼び、確定する | 0 を返す。`findById(R.id)` が R を返す |
| TC-ports-recoveryRequestRepository-47 | `expiresAt` が b より前の、`channel` が `email` の `pending`、`used`、`revoked` の回復依頼と、`channel` が `operator` の `pending` の回復依頼が 1 件ずつ、`expiresAt` が b より後の回復依頼が 1 件確定済み | `deleteLapsed(b, 10)` を呼び、確定する | 4 を返す。b より前の 4 件の `findById` が `null` を、b より後の 1 件がその回復依頼を返す |
| TC-ports-recoveryRequestRepository-48 | `expiresAt` が b より後で、`usedAt` が b より前の `used` の回復依頼 R と、`expiresAt` が b より後で、`revokedAt` が b より前の `revoked` の回復依頼 Q が確定済み | `deleteLapsed(b, 10)` を呼び、確定する | 0 を返す。`findById` が R と Q をそれぞれ返す |
| TC-ports-recoveryRequestRepository-50 | アカウント X の、`expiresAt` が b より前の `pending` の回復依頼 R を、`deleteLapsed(b, 10)` で削除して確定している | X の `pending` の回復依頼 Q を `insert` し、確定する | `run` が確定する。`findPendingByAccount(X, "email")` が Q を返す |
| TC-ports-recoveryRequestRepository-51 | `expiresAt` が b より前の回復依頼が 1 件確定済み | `deleteLapsed(b, 1)` を呼び、確定する | 1 を返す。その回復依頼の `findById` が `null` を返す |
| TC-ports-recoveryRequestRepository-52 | `expiresAt` が b より前の回復依頼が、上限の件数だけ確定済み | `deleteLapsed`（b、上限の `limit`）を呼び、確定する | 上限の件数を返す |
| TC-ports-recoveryRequestRepository-53 | `expiresAt` が互いに違い、どれも b より前の回復依頼が 3 件確定済み | `deleteLapsed(b, 2)` を呼び、確定する | 2 を返す。`expiresAt` の昇順で先頭の 2 件の `findById` が `null` を、3 件目がその回復依頼を返す |
| TC-ports-recoveryRequestRepository-54 | `expiresAt` が同じで、b より前の回復依頼が 3 件確定済み | `deleteLapsed(b, 2)` を呼び、確定する | 2 を返す。ID の昇順で先頭の 2 件の `findById` が `null` を、3 件目がその回復依頼を返す |
| TC-ports-recoveryRequestRepository-55 | `expiresAt` が b より前の回復依頼が 3 件確定済み | `deleteLapsed(b, 2)` を三回、それぞれ確定する | 順に 2、1、0 を返す。三回目の後、3 件の `findById` が `null` を返す |
| TC-ports-recoveryRequestRepository-58 | `expiresAt` が b より前の回復依頼が、上限より 1 件多く確定済み | `deleteLapsed`（b、上限の `limit`）を二回、それぞれ確定する | 順に、上限の件数、1 を返す |
| TC-ports-recoveryRequestRepository-59 | `expiresAt` が b より前の回復依頼 R が確定済み | `deleteLapsed(b, 0)` を呼ぶ | `run` が `SystemError`（理由 `INVALID_QUERY`）で失敗する。`findById(R.id)` が R を返す |
| TC-ports-recoveryRequestRepository-60 | `expiresAt` が b より前の回復依頼 R が確定済み | `deleteLapsed`（b、上限より 1 大きい `limit`）を呼ぶ | `run` が `SystemError`（理由 `INVALID_QUERY`）で失敗する。`findById(R.id)` が R を返す |
| TC-ports-recoveryRequestRepository-61 | `expiresAt` が b より前の回復依頼 R を読んで v を得た後、別の UnitOfWork が R を `save` して確定している | `deleteLapsed(b, 10)` を呼び、確定する | 1 を返す。`findById(R.id)` が `null` を返す |
| TC-ports-recoveryRequestRepository-62 | `expiresAt` が b より前の回復依頼 R が確定済み | `deleteLapsed(b, 10)` を呼んだ後、`fn` が例外を投げる | `findById(R.id)` が R を返す |
