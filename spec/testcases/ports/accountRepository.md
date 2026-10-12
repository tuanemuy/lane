# AccountRepository

「上限」は、その節のメソッドの契約が定める、一度に渡せる件数・個数、または `limit` の上限を指す。値は、ポートを定めるドメインのファイルが持つ。

`AccountRepository` の契約（[../../domains/identity.md](../../domains/identity.md)）と、`TransactionalRepository` の基底の契約（[../../domains/common.md](../../domains/common.md)）を、すべてのアダプターが通すケースで定める。

- 書き込みは `UnitOfWorkProvider.run` の UnitOfWork の中で呼ぶ。`WriteScope` は `{ kind: "installation" }`。「確定する」は、`fn` が正常に終わり `run` が値を返すことを指す。「失敗する」は、`run` がそのエラーを投げることを指す
- 読み取りは、断りが無ければ UnitOfWork の外で呼ぶ
- 「並行した二つの UnitOfWork」は、並行に実行する二つの `run` を指す。楽観ロックの `expectedVersion` は、二つの `run` の前に読む
- アカウントは `Account.create` と `Account` の振る舞いで作る。ID は `IdGenerator.generate` の値を使う
- ケースは、所属の索引に項目を置かないアカウントで組む。所属の索引との整合性は [membershipIndex.md](membershipIndex.md) が定める

## insert

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-accountRepository-01 | なし | アカウント A を `insert` し、確定する | `findById(A.id)` が、A と全フィールドが一致するアカウントを返す。`findByEmail(A.email)` が A を返す |
| TC-ports-accountRepository-02 | なし | A を `insert` した後、`fn` が例外を投げる | `findById(A.id)` と `findByEmail(A.email)` が `null` を返す |
| TC-ports-accountRepository-03 | A が確定済み | A と同じ ID で、メールアドレスが違うアカウント B を `insert` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する。`findById(A.id)` が A を返す。`findByEmail(B.email)` が `null` を返す |
| TC-ports-accountRepository-04 | A が確定済み | A と ID が違い、メールアドレスが同じアカウント B を `insert` する | `run` が `ConflictError`（理由 `EMAIL_TAKEN`）で失敗する。`findById(B.id)` が `null` を返す。`findByEmail(A.email)` が A を返す |
| TC-ports-accountRepository-05 | なし | 一つの UnitOfWork で、メールアドレスが同じアカウント A、B を `insert` する | `run` が `ConflictError`（理由 `EMAIL_TAKEN`）で失敗する。`findById(A.id)` と `findById(B.id)` が `null` を返す |
| TC-ports-accountRepository-06 | なし | 並行した二つの UnitOfWork が、メールアドレスが同じアカウント A、B をそれぞれ `insert` する | 一方の `run` だけが確定し、他方が `ConflictError`（理由 `EMAIL_TAKEN`）で失敗する。`findByEmail` が、確定した側のアカウントを返す |

## findById

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-accountRepository-08 | なし | 保存していない ID で `findById` を呼ぶ | `null` を返す |
| TC-ports-accountRepository-10 | `Account.startMfaEnrollment` を適用した A を `save` して確定している | `findById(A.id)` を呼ぶ | `mfa.status` が `enrolling` で、`secret` が保存した値と一致するアカウントを返す |
| TC-ports-accountRepository-11 | `Account.enableMfa` を適用した A を `save` して確定している | `findById(A.id)` を呼ぶ | `mfa.status` が `enabled` で、`secret` と `enabledAt` が保存した値と一致するアカウントを返す |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-accountRepository-12 | A が確定済みで、`findById` で `expectedVersion` v を得ている | `Account.startMfaEnrollment` を適用した A を v で `save` し、確定する | `findById(A.id)` と `findByEmail(A.email)` が、`mfa.status` が `enrolling` の、変更後の内容を返す |
| TC-ports-accountRepository-14 | A を読んで v を得た後、別の UnitOfWork が A を `save` して確定している | 変更した A を v で `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）で失敗する。`findById(A.id)` が、別の UnitOfWork が保存した内容を返す |
| TC-ports-accountRepository-15 | A が確定済み | 並行した二つの UnitOfWork が、同じ `expectedVersion` で A をそれぞれ `save` する | 一方の `run` だけが確定し、他方が `ConflictError`（理由 `STALE_VERSION`）で失敗する。`findById(A.id)` が、確定した側の内容を返す |
| TC-ports-accountRepository-16 | A が確定済みで、v を得ている | 変更した A を v で `save` した後、`fn` が例外を投げる | `findById(A.id)` が変更前の内容を返す |

## findByEmail

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-accountRepository-17 | なし | 保存していないメールアドレスで `findByEmail` を呼ぶ | `null` を返す |
| TC-ports-accountRepository-18 | メールアドレスが違う A、B が確定済み | `findByEmail(A.email)` を呼ぶ | A を返す |
| TC-ports-accountRepository-19 | A が確定済み | A のメールアドレスを、大文字と前後の空白を加えた文字列から `EmailAddress.create` で作り、`findByEmail` を呼ぶ | A を返す |

## findByIds

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-accountRepository-20 | A が確定済み | `findByIds([])` を呼ぶ | 空の列を返す |
| TC-ports-accountRepository-21 | A、B、C が確定済み | `findByIds([A.id, C.id])` を呼ぶ | A と C だけを、ID の昇順で返す |
| TC-ports-accountRepository-23 | A が確定済み | A の ID と、保存していない ID で `findByIds` を呼ぶ | A だけを含む列を返す |
| TC-ports-accountRepository-25 | 上限の件数のアカウントが確定済み | その全件の ID を、ID の降順に並べて `findByIds` を呼ぶ | 上限の件数を、ID の昇順（文字列のコードポイントの昇順）で返す |
| TC-ports-accountRepository-27 | A が確定済み | `findByIds([A.id, A.id])` を呼ぶ | A を 1 件だけ含む列を返す |
| TC-ports-accountRepository-28 | 上限の件数のアカウントが確定済み | その全件の ID に、そのうち 1 件の ID をもう一度加えた上限より 1 個多い ID で `findByIds` を呼ぶ | 失敗せず、上限の件数を返す |
| TC-ports-accountRepository-29 | なし | 保存していない、互いに違う上限より 1 個多い ID で `findByIds` を呼ぶ | `SystemError`（理由 `INVALID_QUERY`）で失敗する |

## insertMany

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-accountRepository-31 | なし | `insertMany([])` を呼び、確定する | `run` が確定する。何も保存しない |
| TC-ports-accountRepository-32 | なし | メールアドレスが互いに違う A、B、C を `insertMany` し、確定する | `findById` が、A、B、C をそれぞれ返す。`findByEmail` が、それぞれのアカウントを返す |
| TC-ports-accountRepository-33 | なし | A、B を `insertMany` した後、`fn` が例外を投げる | `findById(A.id)` と `findById(B.id)` が `null` を返す |
| TC-ports-accountRepository-34 | A が確定済み | A とメールアドレスが同じ B と、メールアドレスが違う C を `insertMany` する | `run` が `ConflictError`（理由 `EMAIL_TAKEN`）で失敗する。`findById(B.id)` と `findById(C.id)` が `null` を返す |
| TC-ports-accountRepository-35 | なし | メールアドレスが同じ A、B を、一回の `insertMany` に渡す | `run` が `ConflictError`（理由 `EMAIL_TAKEN`）で失敗する。どちらも保存しない |
| TC-ports-accountRepository-36 | A が確定済み | A と同じ ID で、メールアドレスが A と違うアカウントと、新しいアカウント C を `insertMany` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する。`findById(C.id)` が `null` を返す |
| TC-ports-accountRepository-37 | なし | メールアドレスが互いに違う上限の件数のアカウントを `insertMany` し、確定する | `run` が確定する。上限の件数が `findById` で返る |
| TC-ports-accountRepository-38 | なし | 上限より 1 件多いアカウントを `insertMany` する | `SystemError`（理由 `INVALID_QUERY`）で失敗する。どのアカウントも保存しない |
