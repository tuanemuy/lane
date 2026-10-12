# InvitationRepository

「上限」は、その節のメソッドの契約が定める、一度に渡せる件数・個数、または `limit` の上限を指す。値は、ポートを定めるドメインのファイルが持つ。

`InvitationRepository` の契約（[../../domains/workspace.md](../../domains/workspace.md)）と、`TransactionalRepository` の基底の契約、「期限つきの一度きりの値」の規則（[../../domains/common.md](../../domains/common.md)）を、すべてのアダプターが通すケースで定める。

- リポジトリは、ワークスペースの区画に結び付き、メソッドはワークスペースの ID を引数に取らない。区画は、`UnitOfWorkProvider.run` に渡す `WriteScope` の `workspaceId` と、`PartitionReader.workspace(workspaceId)` が決める。W、V は、互いに違うワークスペース。断りの無い呼び出しは、W の区画のリポジトリに対して行う。「V の区画」と書いた呼び出しは、V の区画のリポジトリに対して行う
- 書き込みは `UnitOfWorkProvider.run` の UnitOfWork の中で呼ぶ。`WriteScope` は、そのワークスペースの `workspace` で、条件は `none`。「確定する」は、`fn` が正常に終わり `run` が値を返すことを指す。「失敗する」は、`run` がそのエラーを投げることを指す
- 読み取りは、断りが無ければ UnitOfWork の外で、`PartitionReader.workspace` のリポジトリで呼ぶ
- 「並行した二つの UnitOfWork」は、並行に実行する二つの `run` を指す。楽観ロックの `expectedVersion` は、二つの `run` の前に読む
- 招待は `Invitation.issue`、`accept`、`revoke` で作る。`createdAt` と `expiresAt` は、渡す `now` で決める。ID は `IdGenerator.generate` の値を使う。「W の招待」は、W の区画に `insert` する招待を指す。V も同じ。断りの無い招待は、W の招待である
- メンバーのアカウントの ID は `IdGenerator.generate` の値から作る。設置の区画にアカウントを確定する必要は無い
- 「メンバーの招待」は `grant` が `member` の招待、「初期所有者の招待」は `grant` が `initial_owner` の招待を指す。断りの無い招待は、メンバーの招待である
- ワークスペース W、V は、それぞれの区画に `workspaceRepository.insert` で確定済みとする。「`closing` の W」は、`Workspace.beginClosing` を適用した W を `workspaceRepository.save` して確定したワークスペースを指す
- `deleteAll` と `deleteLapsed` は、一回の呼び出しごとに一つの UnitOfWork で確定する

## insert

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-invitationRepository-01 | なし | W の、宛先 e の `pending` のメンバーの招待 I を `insert` し、確定する | `findById(I.id)`、`findByTokenDigest(I.tokenDigest)`、`findPendingMemberByEmail(e)` が、I と全フィールド（`grant` の権限の組を含む）が一致する招待を返す |
| TC-ports-invitationRepository-02 | なし | W の `pending` の初期所有者の招待 I を `insert` し、確定する | `findById(I.id)` と `findPendingInitialOwner()` が、`grant.kind` が `initial_owner` で、ほかのフィールドも I と一致する招待を返す |
| TC-ports-invitationRepository-03 | なし | I を `insert` した後、`fn` が例外を投げる | `findById(I.id)`、`findByTokenDigest(I.tokenDigest)`、`findPendingMemberByEmail(I.email)` が `null` を返す |
| TC-ports-invitationRepository-04 | I が確定済み | I と同じ ID で、宛先と照合値が違う招待 J を `insert` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する。`findByTokenDigest(J.tokenDigest)` が `null` を返す |
| TC-ports-invitationRepository-05 | W の I が確定済み | W の、宛先が違い、照合値が I と同じ招待 J を `insert` する | `run` が `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）で失敗する。`findById(J.id)` が `null` を返す |
| TC-ports-invitationRepository-06 | なし | 並行した二つの UnitOfWork が、W の、宛先が違い、照合値が同じ招待 I、J をそれぞれ `insert` する | 一方の `run` だけが確定し、他方が `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）で失敗する |
| TC-ports-invitationRepository-07 | W の I が確定済み | V の区画で、I と同じ ID、同じ照合値の、V の招待 J を `insert` し、確定する | `run` が確定する。W の区画の `findByTokenDigest` が I を、V の区画の `findByTokenDigest` が J を返す |
| TC-ports-invitationRepository-08 | W の、宛先 e の `pending` の I が確定済み | W の、宛先 e の、照合値が違う `pending` の招待 J を `insert` する | `run` が `ConflictError`（理由 `PENDING_INVITATION_EXISTS`）で失敗する。`findById(J.id)` が `null` を返す。`findPendingMemberByEmail(e)` が I を返す |
| TC-ports-invitationRepository-09 | W の、宛先 e の `pending` の I が確定済み。I の `expiresAt` より後の日時 n | `Invitation.issue` に n を渡して作った、W の、宛先 e の `pending` の招待 J を `insert` する | `run` が `ConflictError`（理由 `PENDING_INVITATION_EXISTS`）で失敗する |
| TC-ports-invitationRepository-10 | なし | 一つの UnitOfWork で、W の、宛先 e の `pending` の招待 I、J を `insert` する | `run` が `ConflictError`（理由 `PENDING_INVITATION_EXISTS`）で失敗する。`findById(I.id)` と `findById(J.id)` が `null` を返す |
| TC-ports-invitationRepository-11 | W の、宛先 e の招待が無い | 並行した二つの UnitOfWork が、W の、宛先 e の `pending` の招待 I、J をそれぞれ `insert` する | 一方の `run` だけが確定し、他方が `ConflictError`（理由 `PENDING_INVITATION_EXISTS`）で失敗する。`findPendingMemberByEmail(e)` が、確定した側の招待を返す |
| TC-ports-invitationRepository-12 | W の `pending` の初期所有者の招待 I が確定済み | W の、宛先が違う `pending` の初期所有者の招待 J を `insert` する | `run` が `ConflictError`（理由 `PENDING_INVITATION_EXISTS`）で失敗する。`findPendingInitialOwner()` が I を返す |
| TC-ports-invitationRepository-13 | W の初期所有者の招待が無い | 並行した二つの UnitOfWork が、W の、宛先が違う `pending` の初期所有者の招待 I、J をそれぞれ `insert` する | 一方の `run` だけが確定し、他方が `ConflictError`（理由 `PENDING_INVITATION_EXISTS`）で失敗する |
| TC-ports-invitationRepository-14 | W の、宛先 e の `pending` の初期所有者の招待 I が確定済み | W の、宛先 e の `pending` のメンバーの招待 J を `insert` し、確定する | `run` が確定する。`findPendingInitialOwner()` が I を、`findPendingMemberByEmail(e)` が J を返す |
| TC-ports-invitationRepository-15 | W の、宛先 e の `pending` のメンバーの招待 I が確定済み | W の、宛先 e の `pending` の初期所有者の招待 J を `insert` し、確定する | `run` が確定する。`findPendingMemberByEmail(e)` が I を、`findPendingInitialOwner()` が J を返す |
| TC-ports-invitationRepository-16 | W の、宛先 e の `pending` のメンバーの招待 I と、宛先 e の `pending` の初期所有者の招待 K が確定済み。I を読んで v を得ている | 一つの UnitOfWork で、`revoke` を適用した I を v で `save` し、W の、宛先 e の新しい `pending` のメンバーの招待 J を `insert` して確定する | `run` が確定する。`findById(I.id)` が `revoked` の I を返す。`findPendingMemberByEmail(e)` が J を、`findPendingInitialOwner()` が、`version` の変わらない K を返す |
| TC-ports-invitationRepository-18 | W の、宛先 e の `pending` のメンバーの招待 I が確定済み | W の、宛先 e の、照合値が I と同じ `pending` の初期所有者の招待 J を `insert` する | `run` が `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）で失敗する |
| TC-ports-invitationRepository-20 | W の `pending` の初期所有者の招待 I を読んで v を得ている | 一つの UnitOfWork で、`revoke` を適用した I を v で `save` し、W の新しい `pending` の初期所有者の招待 J を `insert` して確定する | `run` が確定する。`findPendingInitialOwner()` が J を返す |
| TC-ports-invitationRepository-21 | W の、宛先 e の `accepted` の招待と `revoked` の招待が 1 件ずつ確定済み | W の、宛先 e の `pending` の招待 J を `insert` し、確定する | `run` が確定する。`findPendingMemberByEmail(e)` が J を返す |
| TC-ports-invitationRepository-22 | W の、宛先 e の `pending` の I を、`deleteLapsed(I.expiresAt, 10)` で削除して確定している | W の、宛先 e の、照合値が I と同じ `pending` の招待 J を `insert` し、確定する | `run` が確定する。`findPendingMemberByEmail(e)` と `findByTokenDigest(J.tokenDigest)` が J を返す |
| TC-ports-invitationRepository-23 | W の、宛先 e の `pending` の I が確定済み | V の区画で、V の、宛先 e の `pending` の招待 J を `insert` し、確定する | `run` が確定する。W の区画の `findPendingMemberByEmail(e)` が I を、V の区画の `findPendingMemberByEmail(e)` が J を返す |

## findById

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-invitationRepository-26 | なし | 保存していない ID で `findById` を呼ぶ | `null` を返す |
| TC-ports-invitationRepository-28 | W の I と、V の J が確定済み | W の区画で `findById(J.id)` を呼ぶ | `null` を返す |
| TC-ports-invitationRepository-30 | 区画に何も確定していないワークスペースの ID Z と、W の、宛先 e の `pending` の I が確定済み | Z の区画で `findById(I.id)`、`findByTokenDigest(I.tokenDigest)`、`findPendingMemberByEmail(e)`、`findPendingInitialOwner()` を呼ぶ | どれも `null` を返す |
| TC-ports-invitationRepository-31 | `Invitation.accept` を適用した I を `save` して確定している | `findById(I.id)` を呼ぶ | `status` が `accepted` で、`acceptedAt` と `memberId` が保存した値と一致する招待を返す |
| TC-ports-invitationRepository-32 | `Invitation.revoke` を適用した I を `save` して確定している | `findById(I.id)` を呼ぶ | `status` が `revoked` で、`revokedAt` が保存した値と一致する招待を返す |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-invitationRepository-33 | `pending` の I を `findByTokenDigest` で読んで v を得ている | `Invitation.accept` を適用した I を v で `save` し、確定する | `findByTokenDigest(I.tokenDigest)` が `accepted` の I を返す。`findPendingMemberByEmail(I.email)` が `null` を返す |
| TC-ports-invitationRepository-34 | I を読んで v を得た後、別の UnitOfWork が I を `save` して確定している | 変更した I を v で `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）で失敗する。`findById(I.id)` が、別の UnitOfWork が保存した内容を返す |
| TC-ports-invitationRepository-35 | `pending` の I が確定済み | 並行した二つの UnitOfWork が、同じ `expectedVersion` で、一方は `accept`、他方は `revoke` を適用した I を `save` する | 一方の `run` だけが確定し、他方が `ConflictError`（理由 `STALE_VERSION`）で失敗する。`findById(I.id)` が、確定した側の状態を返す |
| TC-ports-invitationRepository-37 | `pending` の I を読んで v を得ている | `revoke` を適用した I を v で `save` した後、`fn` が例外を投げる | `findById(I.id)` が `pending` の I を返す |
| TC-ports-invitationRepository-38 | I を読んで v を得た後、別の UnitOfWork が `deleteLapsed(I.expiresAt, 10)` で I を削除して確定している | 変更した I を v で `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）で失敗する。`findById(I.id)` が `null` を返す |

## findByTokenDigest

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-invitationRepository-39 | なし | 保存していない照合値で `findByTokenDigest` を呼ぶ | `null` を返す |
| TC-ports-invitationRepository-41 | W の I と、V の J が確定済み | W の区画で `findByTokenDigest(J.tokenDigest)` を呼ぶ | `null` を返す |
| TC-ports-invitationRepository-43 | `accepted` の I と `revoked` の J が確定済み | `findByTokenDigest(I.tokenDigest)` と `findByTokenDigest(J.tokenDigest)` を呼ぶ | `accepted` の I と `revoked` の J を返す |
| TC-ports-invitationRepository-45 | `expiresAt` が現在時刻より前の `pending` の I が確定済み | `findByTokenDigest(I.tokenDigest)` を呼ぶ | `pending` の I を返す |
| TC-ports-invitationRepository-46 | `pending` の初期所有者の招待 I が確定済み | `findByTokenDigest(I.tokenDigest)` を呼ぶ | I を返す |
| TC-ports-invitationRepository-47 | `closing` の W の招待 I が確定済み | `findByTokenDigest(I.tokenDigest)` を呼ぶ | I を返す |

## findPendingMemberByEmail

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-invitationRepository-48 | W の招待が無い | `findPendingMemberByEmail(e)` を呼ぶ | `null` を返す |
| TC-ports-invitationRepository-50 | W の、宛先 e の、`expiresAt` が現在時刻より前の `pending` の I が確定済み | `findPendingMemberByEmail(e)` を呼ぶ | I を返す |
| TC-ports-invitationRepository-51 | W の、宛先 e の `accepted` の招待と `revoked` の招待だけが確定済み | `findPendingMemberByEmail(e)` を呼ぶ | `null` を返す |
| TC-ports-invitationRepository-53 | V の、宛先 e の `pending` の招待だけが確定済み | W の区画で `findPendingMemberByEmail(e)` を呼ぶ | `null` を返す |
| TC-ports-invitationRepository-54 | W の、別の宛先の `pending` の招待だけが確定済み | `findPendingMemberByEmail(e)` を呼ぶ | `null` を返す |
| TC-ports-invitationRepository-55 | W の、宛先 e の `pending` の初期所有者の招待 I だけが確定済み | `findPendingMemberByEmail(e)` を呼ぶ | `null` を返す |
| TC-ports-invitationRepository-56 | W の、宛先 e の `pending` の初期所有者の招待 I と、宛先 e の `pending` のメンバーの招待 J が確定済み | `findPendingMemberByEmail(e)` を呼ぶ | J を返す |

## findPendingInitialOwner

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-invitationRepository-57 | W の招待が無い | `findPendingInitialOwner()` を呼ぶ | `null` を返す |
| TC-ports-invitationRepository-59 | W の、`expiresAt` が現在時刻より前の `pending` の初期所有者の招待 I が確定済み | `findPendingInitialOwner()` を呼ぶ | I を返す |
| TC-ports-invitationRepository-60 | W の `pending` のメンバーの招待だけが確定済み | `findPendingInitialOwner()` を呼ぶ | `null` を返す |
| TC-ports-invitationRepository-62 | W の `accepted` の初期所有者の招待だけが確定済み | `findPendingInitialOwner()` を呼ぶ | `null` を返す |
| TC-ports-invitationRepository-64 | V の `pending` の初期所有者の招待だけが確定済み | W の区画で `findPendingInitialOwner()` を呼ぶ | `null` を返す |

## findPendingPage

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-invitationRepository-65 | W の招待が無い | `findPendingPage({ page: 1, limit: 10 })` を呼ぶ | `items` が空の列、`count` が 0 |
| TC-ports-invitationRepository-66 | W の、宛先が違う `pending` の I1、I2、I3 が `createdAt` t1 < t2 < t3 で確定済み。`insert` の順は I2、I3、I1 | `findPendingPage({ page: 1, limit: 10 })` を呼ぶ | `items` が I3、I2、I1 の順、`count` が 3 |
| TC-ports-invitationRepository-68 | W の、宛先が違う `pending` の招待 3 件が、同じ `createdAt` で確定済み | `findPendingPage({ page: 1, limit: 10 })` を呼ぶ | `items` が ID の昇順、`count` が 3 |
| TC-ports-invitationRepository-70 | W の、`pending` の I、`accepted` の招待、`revoked` の招待が確定済み | `findPendingPage({ page: 1, limit: 10 })` を呼ぶ | `items` が I だけ、`count` が 1 |
| TC-ports-invitationRepository-71 | W の、`expiresAt` が現在時刻より前の `pending` の I が確定済み | `findPendingPage({ page: 1, limit: 10 })` を呼ぶ | `items` が I だけ、`count` が 1 |
| TC-ports-invitationRepository-72 | W の、`pending` のメンバーの招待 I と、宛先が違う `pending` の初期所有者の招待が確定済み | `findPendingPage({ page: 1, limit: 10 })` を呼ぶ | `items` が I だけ、`count` が 1 |
| TC-ports-invitationRepository-73 | W の `pending` の招待 2 件と、V の `pending` の招待 3 件が確定済み | W の区画と V の区画で、それぞれ `findPendingPage({ page: 1, limit: 10 })` を呼ぶ | W の区画は、`items` が W の 2 件だけ、`count` が 2。V の区画は、`items` が V の 3 件だけ、`count` が 3 |
| TC-ports-invitationRepository-74 | W の、宛先が違う `pending` の I1、I2、I3 が `createdAt` t1 < t2 < t3 で確定済み | `findPendingPage({ page: 1, limit: 3 })` を呼ぶ | `items` が I3、I2、I1、`count` が 3 |
| TC-ports-invitationRepository-75 | 同上 | `findPendingPage({ page: 1, limit: 2 })` を呼ぶ | `items` が I3、I2、`count` が 3 |
| TC-ports-invitationRepository-76 | 同上 | `findPendingPage({ page: 2, limit: 2 })` を呼ぶ | `items` が I1 だけ、`count` が 3 |
| TC-ports-invitationRepository-77 | 同上 | `findPendingPage({ page: 3, limit: 2 })` を呼ぶ | `items` が空の列、`count` が 3 |
| TC-ports-invitationRepository-78 | W の、宛先が違う `pending` の招待が 101 件確定済み | `page` が 1 と 2、`limit` が 100 で `findPendingPage` を呼ぶ | 一つ目の `items` が 100 件、二つ目の `items` が残りの 1 件。どちらも `count` は 101。二つの `items` に重複が無い |
| TC-ports-invitationRepository-80 | 区画に何も確定していないワークスペースの ID Z | Z の区画で `findPendingPage({ page: 1, limit: 10 })` を呼ぶ | `items` が空の列、`count` が 0 |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-invitationRepository-81 | 招待を持たない `closing` の W | `deleteAll(10)` を呼び、確定する | 0 を返す |
| TC-ports-invitationRepository-82 | 招待 1 件を持つ `closing` の W | `deleteAll(1)` を呼び、確定する | 1 を返す。その招待の `findById` が `null` を返す |
| TC-ports-invitationRepository-83 | `pending`、`accepted`、`revoked` の招待と、`pending` の初期所有者の招待を 1 件ずつ持つ `closing` の W | `deleteAll(10)` を呼び、確定する | 4 を返す。4 件の `findById` が `null` を返す |
| TC-ports-invitationRepository-84 | 招待を、上限の件数だけ持つ `closing` の W | `deleteAll`（上限の `limit`）を呼び、確定する | 上限の件数を返す |
| TC-ports-invitationRepository-85 | 招待 3 件を持つ `closing` の W | `deleteAll(2)` を三回、それぞれ確定する | 順に 2、1、0 を返す。一回目の後、3 件のうち 1 件だけが `findById` で返る。三回目の後、3 件の `findById` が `null` を返す |
| TC-ports-invitationRepository-87 | 招待を、上限より 1 件多く持つ `closing` の W | `deleteAll`（上限の `limit`）を二回、それぞれ確定する | 順に、上限の件数、1 を返す |
| TC-ports-invitationRepository-88 | 招待 2 件を持つ `closing` の W と、招待 J を持つ V | W の区画で `deleteAll(10)` を呼び、確定する | 2 を返す。V の区画の `findById(J.id)` が J を返す |
| TC-ports-invitationRepository-89 | 招待 I を持つ `closing` の W | `deleteAll(0)`、`deleteAll(-1)`、`deleteAll(1.5)` を、それぞれ別の UnitOfWork で呼ぶ | どの `run` も `SystemError`（理由 `INVALID_QUERY`）で失敗する。`findById(I.id)` が I を返す |
| TC-ports-invitationRepository-90 | 招待 I を持つ `closing` の W | `deleteAll`（上限より 1 大きい `limit`）を呼ぶ | `run` が `SystemError`（理由 `INVALID_QUERY`）で失敗する。`findById(I.id)` が I を返す |
| TC-ports-invitationRepository-91 | `closing` の W の招待 I を読んで v を得た後、別の UnitOfWork が I を `save` して確定している | `deleteAll(10)` を呼び、確定する | `run` が確定する。`findById(I.id)` が `null` を返す |
| TC-ports-invitationRepository-92 | 招待 2 件を持つ `closing` の W | `deleteAll(10)` を呼んだ後、`fn` が例外を投げる | 2 件の `findById` がそれぞれの招待を返す |
| TC-ports-invitationRepository-93 | 招待 2 件を持つ `closing` の W で、`deleteAll(10)` と、`Workspace.close` を適用した W の `workspaceRepository.save` を確定している（W は `closed`） | `deleteAll(10)` を呼び、確定する | 0 を返す |

## deleteLapsed

`before` に渡す日時を b とする。招待の `lapsedAt` は `expiresAt`（`Invitation.lapsedAt`）で、`Invitation.issue` に渡す `now` で決まる。「対象の招待」は、`expiresAt` が b 以前（同じ日時を含む）の招待を指す。断りの無い招待は、宛先が互いに違う。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-invitationRepository-96 | 招待が無い | `deleteLapsed(b, 10)` を呼び、確定する | 0 を返す |
| TC-ports-invitationRepository-97 | `expiresAt` が b の 1 ミリ秒前の `pending` の招待 I が確定済み | `deleteLapsed(b, 10)` を呼び、確定する | 1 を返す。`findById(I.id)`、`findByTokenDigest(I.tokenDigest)`、`findPendingMemberByEmail(I.email)` が `null` を返す |
| TC-ports-invitationRepository-98 | `expiresAt` が b と同じ招待 X が確定済み | `deleteLapsed(b, 10)` を呼び、確定する | 1 を返す。`findById(X.id)` が `null` を返す |
| TC-ports-invitationRepository-99 | `expiresAt` が b の 1 ミリ秒後の招待 X が確定済み | `deleteLapsed(b, 10)` を呼び、確定する | 0 を返す。`findById(X.id)` が X を返す |
| TC-ports-invitationRepository-100 | 対象の `pending`、`accepted`、`revoked` の招待が 1 件ずつ、`expiresAt` が b より後の招待が 1 件確定済み | `deleteLapsed(b, 10)` を呼び、確定する | 3 を返す。対象の 3 件の `findById` が `null` を、`expiresAt` が b より後の 1 件がその招待を返す |
| TC-ports-invitationRepository-101 | `expiresAt` が b より後で、`acceptedAt` が b より前の `accepted` の招待 X と、`expiresAt` が b より後で、`revokedAt` が b より前の `revoked` の招待 Y が確定済み | `deleteLapsed(b, 10)` を呼び、確定する | 0 を返す。`findById` が X と Y を返す |
| TC-ports-invitationRepository-102 | 対象の招待が、W と V に 1 件ずつ確定済み | W の区画で `deleteLapsed(b, 10)` を呼び、確定する | 1 を返す。W の招待の `findById` が `null` を返す。V の区画の `findById` が、V の招待を返す |
| TC-ports-invitationRepository-103 | 対象の招待を持つ `closing` の W | `deleteLapsed(b, 10)` を呼び、確定する | 1 を返す |
| TC-ports-invitationRepository-104 | 区画に何も確定していないワークスペースの ID Z | Z の区画で `deleteLapsed(b, 10)` を呼び、確定する | `run` が確定し、0 を返す |
| TC-ports-invitationRepository-105 | 対象の、`pending` の初期所有者の招待 I が確定済み | `deleteLapsed(b, 10)` を呼び、確定する | 1 を返す。`findPendingInitialOwner()` が `null` を返す |
| TC-ports-invitationRepository-106 | 対象の `accepted` の招待 I と、その `memberId` が指すメンバー M が確定済み | `deleteLapsed(b, 10)` を呼び、確定する | 1 を返す。`memberRepository.findById(M.id)` が、変わらない M と、変わらない `expectedVersion` を返す |
| TC-ports-invitationRepository-107 | 対象の招待が 1 件確定済み | `deleteLapsed(b, 1)` を呼び、確定する | 1 を返す。その招待の `findById` が `null` を返す |
| TC-ports-invitationRepository-108 | 対象の招待が、上限の件数だけ確定済み | `deleteLapsed`（b、上限の `limit`）を呼び、確定する | 上限の件数を返す |
| TC-ports-invitationRepository-109 | 対象の招待が 3 件確定済み | `deleteLapsed(b, 2)` を三回、それぞれ確定する | 順に 2、1、0 を返す。一回目の後、3 件のうち 1 件だけが `findById` で返る。三回目の後、3 件の `findById` が `null` を返す |
| TC-ports-invitationRepository-110 | 対象の招待 I1、I2、I3 が `expiresAt` e1 < e2 < e3 で確定済み。`insert` の順は I3、I1、I2 | `deleteLapsed(b, 2)` を呼び、確定する | 2 を返す。`findById` が、I1 と I2 に `null` を、I3 にその招待を返す |
| TC-ports-invitationRepository-111 | 対象の招待 3 件が、同じ `expiresAt` で確定済み | `deleteLapsed(b, 2)` を呼び、確定する | 2 を返す。ID の昇順で先頭の 2 件の `findById` が `null` を、残りの 1 件がその招待を返す |
| TC-ports-invitationRepository-113 | 対象の招待が、上限より 1 件多く確定済み | `deleteLapsed`（b、上限の `limit`）を二回、それぞれ確定する | 順に、上限の件数、1 を返す |
| TC-ports-invitationRepository-114 | 対象の招待 I が確定済み | `deleteLapsed(b, 0)`、`deleteLapsed(b, -1)`、`deleteLapsed(b, 1.5)` を、それぞれ別の UnitOfWork で呼ぶ | どの `run` も `SystemError`（理由 `INVALID_QUERY`）で失敗する。`findById(I.id)` が I を返す |
| TC-ports-invitationRepository-115 | 対象の招待 I が確定済み | `deleteLapsed`（b、上限より 1 大きい `limit`）を呼ぶ | `run` が `SystemError`（理由 `INVALID_QUERY`）で失敗する。`findById(I.id)` が I を返す |
| TC-ports-invitationRepository-118 | 対象の招待 I を読んで v を得た後、別の UnitOfWork が I を `save` して確定している | `deleteLapsed(b, 10)` を呼び、確定する | 1 を返す。`findById(I.id)` が `null` を返す |
| TC-ports-invitationRepository-119 | 対象の招待 I が確定済み | `deleteLapsed(b, 10)` を呼んだ後、`fn` が例外を投げる | `findById(I.id)` が I を返す |
| TC-ports-invitationRepository-120 | V の区画に何も確定していない | V の区画で `deleteLapsed(b, 10)` を呼び、確定する | 0 を返す |
