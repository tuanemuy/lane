# MemberRepository

「上限」は、その節のメソッドの契約が定める、一度に渡せる件数・個数、または `limit` の上限を指す。値は、ポートを定めるドメインのファイルが持つ。

`MemberRepository` の契約（[../../domains/workspace.md](../../domains/workspace.md)）と、`TransactionalRepository` の基底の契約（[../../domains/common.md](../../domains/common.md)）を、すべてのアダプターが通すケースで定める。

- リポジトリは、ワークスペースの区画に結び付き、メソッドはワークスペースの ID を引数に取らない。区画は、`UnitOfWorkProvider.run` に渡す `WriteScope` の `workspaceId` と、`PartitionReader.workspace(workspaceId)` が決める。W、V は、互いに違うワークスペース。断りの無い呼び出しは、W の区画のリポジトリに対して行う。「V の区画」と書いた呼び出しは、V の区画のリポジトリに対して行う
- 書き込みは `UnitOfWorkProvider.run` の UnitOfWork の中で呼ぶ。`WriteScope` は、そのワークスペースの `workspace` で、条件は `none`。「確定する」は、`fn` が正常に終わり `run` が値を返すことを指す。「失敗する」は、`run` がそのエラーを投げることを指す
- 読み取りは、断りが無ければ UnitOfWork の外で、`PartitionReader.workspace` のリポジトリで呼ぶ
- 「並行した二つの UnitOfWork」は、並行に実行する二つの `run` を指す。楽観ロックの `expectedVersion` は、二つの `run` の前に読む
- メンバーは `Member.create` と `Member.changePermissions` で作る。`createdAt` は、渡す `now` で決める。ID は `IdGenerator.generate` の値を使う。「W のメンバー」は、W の区画に `insert` するメンバーを指す。V も同じ
- アカウントの ID は `IdGenerator.generate` の値から作る。設置の区画にアカウントを確定する必要は無い。断りの無いメンバーは、互いに違うアカウントのメンバーである
- 「作成した W」は、一つの UnitOfWork で、`Workspace.create` の結果を `workspaceRepository.insert`、`AccessPolicy.create` の結果（所有者の指定が `vacant`）を `accessPolicyRepository.insert` で確定したワークスペースを指す
- 「`active` の W（所有者 M）」は、作成した W に対し、一つの UnitOfWork で W のメンバー M を `insert` し、`AccessPolicy.designateOwner` で M を所有者に指定した方針を `accessPolicyRepository.save` して確定したワークスペースを指す
- 「所有者がいない W」は、`active` の W（所有者 M）に対し、`AccessPolicy.clearOwner` を適用した方針を `accessPolicyRepository.save` して確定したワークスペースを指す。M は区画に残る
- 「`closing` の W」は、`Workspace.beginClosing` を適用した W を `workspaceRepository.save` して確定したワークスペースを指す
- `deleteAll` は、一回の呼び出しごとに一つの UnitOfWork で確定する

## insert

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-memberRepository-01 | `active` の W（所有者 M） | W の、別のアカウントのメンバー N を `insert` し、確定する | `findById(N.id)` と `findByAccount(N.accountId)` が、N と全フィールドが一致するメンバーを返す |
| TC-ports-memberRepository-02 | `active` の W | 権限の組が空のメンバー N を `insert` し、確定する | `findById(N.id)` が、`permissions` が空の組のメンバーを返す |
| TC-ports-memberRepository-03 | `active` の W | N を `insert` した後、`fn` が例外を投げる | `findById(N.id)` と `findByAccount(N.accountId)` が `null` を返す |
| TC-ports-memberRepository-04 | W の区画に何も確定していない | 一つの UnitOfWork で、ワークスペース W を `workspaceRepository.insert` し、W のメンバー M を `insert` して確定する | `run` が確定する。`findById(M.id)` が M を返す |
| TC-ports-memberRepository-05 | W のメンバー N が確定済み | N と同じ ID で、アカウントが違う W のメンバー P を `insert` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する。`findByAccount(P.accountId)` が `null` を返す |
| TC-ports-memberRepository-06 | W のメンバー N が確定済み | V の区画で、N と同じ ID の V のメンバー P を `insert` し、確定する | `run` が確定する。W の区画の `findById(N.id)` が N を、V の区画の `findById(N.id)` が P を返す |
| TC-ports-memberRepository-07 | W の、アカウント X のメンバー N が確定済み | W の、X のメンバー P を `insert` する | `run` が `ConflictError`（理由 `ALREADY_MEMBER`）で失敗する。`findById(P.id)` が `null` を返す。`findByAccount(X)` が N を返す |
| TC-ports-memberRepository-08 | `active` の W | 一つの UnitOfWork で、W の、同じアカウントのメンバー N、P を `insert` する | `run` が `ConflictError`（理由 `ALREADY_MEMBER`）で失敗する。`findById(N.id)` と `findById(P.id)` が `null` を返す |
| TC-ports-memberRepository-09 | `active` の W | 並行した二つの UnitOfWork が、W の、同じアカウントのメンバー N、P をそれぞれ `insert` する | 一方の `run` だけが確定し、他方が `ConflictError`（理由 `ALREADY_MEMBER`）で失敗する |
| TC-ports-memberRepository-10 | W の、アカウント X のメンバー N と、`active` の V が確定済み | V の区画で、V の、X のメンバー P を `insert` し、確定する | `run` が確定する。W の区画の `findByAccount(X)` が N を、V の区画の `findByAccount(X)` が P を返す |
| TC-ports-memberRepository-11 | W の、アカウント X のメンバー N を `delete` して確定している | W の、X のメンバー P を `insert` し、確定する | `run` が確定する。`findByAccount(X)` が P を返す |
| TC-ports-memberRepository-12 | `active` の W。設置の区画に確定していないアカウントの ID X | W の、X のメンバー N を `insert` し、確定する | `run` が確定する。`findById(N.id)` が N を返す |

## findById

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-memberRepository-13 | なし | 保存していない ID で `findById` を呼ぶ | `null` を返す |
| TC-ports-memberRepository-15 | W のメンバー N と、V のメンバー P が確定済み | W の区画で `findById(P.id)` を呼ぶ | `null` を返す |
| TC-ports-memberRepository-17 | 区画に何も確定していないワークスペースの ID Z と、W の、アカウント X のメンバー N が確定済み | Z の区画で `findById(N.id)` と `findByAccount(X)` を呼ぶ | どちらも `null` を返す |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-memberRepository-18 | W のメンバー N を読んで `expectedVersion` v を得ている | `Member.changePermissions` で権限の組を変えた N を v で `save` し、確定する | `findById(N.id)` が、変更後の権限の組を、語彙の定義順で持つメンバーを返す |
| TC-ports-memberRepository-19 | N を読んで v を得た後、別の UnitOfWork が N を `save` して確定している | 変更した N を v で `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）で失敗する。`findById(N.id)` が、別の UnitOfWork が保存した内容を返す |
| TC-ports-memberRepository-20 | W のメンバー N が確定済み | 並行した二つの UnitOfWork が、同じ `expectedVersion` で、違う権限の組にした N をそれぞれ `save` する | 一方の `run` だけが確定し、他方が `ConflictError`（理由 `STALE_VERSION`）で失敗する。`findById(N.id)` が、確定した側の権限の組を返す |
| TC-ports-memberRepository-21 | N を読んで v を得ている | 変更した N を v で `save` した後、`fn` が例外を投げる | `findById(N.id)` が変更前の内容を返す |
| TC-ports-memberRepository-22 | N を読んで v を得た後、別の UnitOfWork が N を `delete` して確定している | 変更した N を v で `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）で失敗する。`findById(N.id)` が `null` を返す |

## delete

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-memberRepository-23 | `active` の W（所有者 M）と、W のメンバー N が確定済みで、N の v を得ている | `delete(N.id, v)` を呼び、確定する | `findById(N.id)` と `findByAccount(N.accountId)` が `null` を返す。`findById(M.id)` が M を返す |
| TC-ports-memberRepository-24 | `active` の W（所有者 M） | M を `delete` する | `run` が `ConflictError`（理由 `MEMBER_IS_OWNER`）で失敗する。`findById(M.id)` が M を返す |
| TC-ports-memberRepository-25 | `active` の W（所有者 M）と、W のメンバー N が確定済みで、所有者を N にした方針を `accessPolicyRepository.save` して確定している | M を `delete` し、確定する | `run` が確定する。`findById(M.id)` が `null` を返す |
| TC-ports-memberRepository-27 | `active` の W（所有者 M）と、W のメンバー N が確定済み | 一つの UnitOfWork で、所有者を N にした方針を `accessPolicyRepository.save` し、M を `delete` して確定する | `run` が確定する。`findById(M.id)` が `null` を返す。`accessPolicyRepository.find()` の `owner.memberId` が N の ID |
| TC-ports-memberRepository-28 | 同上 | 並行した二つの UnitOfWork の一方が、所有者を N にした方針を `accessPolicyRepository.save` し、他方が N を `delete` する | 一方の `run` だけが確定し、他方が `ConflictError` で失敗する。確定後、`accessPolicyRepository.find()` の `owner.memberId` が指すメンバーが `findById` で返る |
| TC-ports-memberRepository-29 | `active` の W（所有者 M） | 一つの UnitOfWork で、`AccessPolicy.clearOwner` を適用した方針を `accessPolicyRepository.save` し、M を `delete` して確定する | `run` が確定する。`findById(M.id)` が `null` を返す。`accessPolicyRepository.find()` が、`owner.status` が `vacant` の方針を返す |
| TC-ports-memberRepository-31 | 所有者がいない W と、W のメンバー N が確定済み | 所有者に指定されていた M と N を、それぞれ別の UnitOfWork で `delete` し、確定する | どちらの `run` も確定する。`findById(M.id)` と `findById(N.id)` が `null` を返す |
| TC-ports-memberRepository-33 | `active` の W（所有者 M）を `closing` にしている。方針は区画に残っている | M を `delete` する | `run` が `ConflictError`（理由 `MEMBER_IS_OWNER`）で失敗する |
| TC-ports-memberRepository-34 | 方針が無い区画（`Workspace` だけを確定した W）のメンバー N が確定済み | N を `delete` し、確定する | `run` が確定する |
| TC-ports-memberRepository-35 | N を読んで v を得た後、別の UnitOfWork が N を `save` して確定している | `delete(N.id, v)` を呼ぶ | `run` が `ConflictError`（理由 `STALE_VERSION`）で失敗する。`findById(N.id)` が N を返す |
| TC-ports-memberRepository-36 | N を読んで v を得ている | `delete(N.id, v)` を呼んだ後、`fn` が例外を投げる | `findById(N.id)` が N を返す |
| TC-ports-memberRepository-37 | N を読んで v を得た後、別の UnitOfWork が N を `delete` して確定している | `delete(N.id, v)` を呼ぶ | `run` が `ConflictError`（理由 `STALE_VERSION`）で失敗する |

## findByAccount

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-memberRepository-38 | `active` の W | メンバーを持たないアカウント X で `findByAccount(X)` を呼ぶ | `null` を返す |
| TC-ports-memberRepository-41 | V の、X のメンバー P だけが確定済み | W の区画で `findByAccount(X)` を呼ぶ | `null` を返す |

## findByIds

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-memberRepository-43 | W のメンバー N が確定済み | `findByIds([])` を呼ぶ | 空の列を返す |
| TC-ports-memberRepository-44 | W のメンバー N、P、Q が確定済み | `findByIds([N.id, Q.id])` を呼ぶ | N と Q だけを、ID の昇順で返す |
| TC-ports-memberRepository-46 | W のメンバー N と、V のメンバー P が確定済み | `findByIds([N.id, P.id])` を呼ぶ | N だけを含む列を返す |
| TC-ports-memberRepository-48 | W のメンバー N が確定済み | N の ID と、保存していない ID で `findByIds` を呼ぶ | N だけを含む列を返す |
| TC-ports-memberRepository-49 | W のメンバーが、上限の件数だけ確定済み | その全件の ID を、ID の降順に並べて `findByIds` を呼ぶ | 上限の件数を、ID の昇順（文字列のコードポイントの昇順）で返す |
| TC-ports-memberRepository-51 | W のメンバー N が確定済み | `findByIds([N.id, N.id])` を呼ぶ | N を 1 件だけ含む列を返す |
| TC-ports-memberRepository-52 | W のメンバーが、上限の件数だけ確定済み | その全件の ID に、そのうち 1 件の ID をもう一度加えた上限より 1 個多い ID で `findByIds` を呼ぶ | 失敗せず、上限の件数を返す |
| TC-ports-memberRepository-53 | なし | 保存していない、互いに違う上限より 1 個多い ID で `findByIds` を呼ぶ | `SystemError`（理由 `INVALID_QUERY`）で失敗する |
| TC-ports-memberRepository-55 | 区画に何も確定していないワークスペースの ID Z と、W のメンバー N が確定済み | Z の区画で `findByIds([N.id])` を呼ぶ | 空の列を返す |

## findPage

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-memberRepository-56 | メンバーを持たない、作成した W | `findPage({ page: 1, limit: 10 })` を呼ぶ | `items` が空の列、`count` が 0 |
| TC-ports-memberRepository-57 | W のメンバー M1、M2、M3 が `createdAt` t1 < t2 < t3 で確定済み。`insert` の順は M3、M1、M2 | `findPage({ page: 1, limit: 10 })` を呼ぶ | `items` が M1、M2、M3 の順、`count` が 3 |
| TC-ports-memberRepository-59 | W のメンバー 3 件が、同じ `createdAt` で確定済み | `findPage({ page: 1, limit: 10 })` を呼ぶ | `items` が ID の昇順、`count` が 3 |
| TC-ports-memberRepository-60 | W のメンバー M1、M2、M3 が `createdAt` t1 < t2 < t3 で確定済み | `findPage({ page: 1, limit: 3 })` を呼ぶ | `items` が M1、M2、M3、`count` が 3 |
| TC-ports-memberRepository-61 | 同上 | `findPage({ page: 1, limit: 2 })` を呼ぶ | `items` が M1、M2、`count` が 3 |
| TC-ports-memberRepository-62 | 同上 | `findPage({ page: 2, limit: 2 })` を呼ぶ | `items` が M3 だけ、`count` が 3 |
| TC-ports-memberRepository-63 | 同上 | `findPage({ page: 3, limit: 2 })` を呼ぶ | `items` が空の列、`count` が 3 |
| TC-ports-memberRepository-64 | W のメンバーが 101 件確定済み | `page` が 1 と 2、`limit` が 100 で `findPage` を呼ぶ | 一つ目の `items` が 100 件、二つ目の `items` が残りの 1 件。どちらも `count` は 101。二つの `items` に重複が無い |
| TC-ports-memberRepository-65 | W のメンバー 2 件と、V のメンバー 3 件が確定済み | W の区画と V の区画で、それぞれ `findPage({ page: 1, limit: 10 })` を呼ぶ | W の区画は、`items` が W の 2 件だけ、`count` が 2。V の区画は、`items` が V の 3 件だけ、`count` が 3 |
| TC-ports-memberRepository-67 | 区画に何も確定していないワークスペースの ID Z | Z の区画で `findPage({ page: 1, limit: 10 })` を呼ぶ | `items` が空の列、`count` が 0 |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-memberRepository-68 | メンバーを持たない `closing` の W | `deleteAll(10)` を呼び、確定する | 0 を返す |
| TC-ports-memberRepository-69 | メンバー 1 件を持つ `closing` の W | `deleteAll(1)` を呼び、確定する | 1 を返す。そのメンバーの `findById` が `null` を返す |
| TC-ports-memberRepository-70 | メンバーを、上限の件数だけ持つ `closing` の W | `deleteAll`（上限の `limit`）を呼び、確定する | 上限の件数を返す |
| TC-ports-memberRepository-71 | メンバー 3 件を持つ `closing` の W | `deleteAll(2)` を三回、それぞれ確定する | 順に 2、1、0 を返す。一回目の後、`findPage` の `count` が 1。三回目の後、`count` が 0 |
| TC-ports-memberRepository-73 | メンバーを、上限より 1 件多く持つ `closing` の W | `deleteAll`（上限の `limit`）を二回、それぞれ確定する | 順に、上限の件数、1 を返す |
| TC-ports-memberRepository-74 | メンバー 2 件を持つ `closing` の W と、メンバー 2 件を持つ `active` の V | W の区画で `deleteAll(10)` を呼び、確定する | 2 を返す。V の区画の `findPage` の `count` が 2 |
| TC-ports-memberRepository-75 | メンバー 1 件を持つ `closing` の W | `deleteAll(0)`、`deleteAll(-1)`、`deleteAll(1.5)` を、それぞれ別の UnitOfWork で呼ぶ | どの `run` も `SystemError`（理由 `INVALID_QUERY`）で失敗する。メンバーの `findById` がそのメンバーを返す |
| TC-ports-memberRepository-76 | メンバー 1 件を持つ `closing` の W | `deleteAll`（上限より 1 大きい `limit`）を呼ぶ | `run` が `SystemError`（理由 `INVALID_QUERY`）で失敗する。メンバーの `findById` がそのメンバーを返す |
| TC-ports-memberRepository-77 | `closing` の W のメンバー N を読んで v を得た後、別の UnitOfWork が N を `save` して確定している | `deleteAll(10)` を呼び、確定する | `run` が確定する。`findById(N.id)` が `null` を返す |
| TC-ports-memberRepository-78 | メンバー 2 件を持つ `closing` の W | `deleteAll(10)` を呼んだ後、`fn` が例外を投げる | `findPage` の `count` が 2 |
| TC-ports-memberRepository-79 | メンバー 2 件を持つ `closing` の W で、`deleteAll(10)`、`accessPolicyRepository.deleteAll(10)`、`Workspace.close` を適用した W の `workspaceRepository.save` を確定している（W は `closed`） | `deleteAll(10)` を呼び、確定する | 0 を返す |
| TC-ports-memberRepository-82 | `active` の W（所有者 M）とメンバー N を、`closing` にしている | `deleteAll(10)` を呼び、確定する | 2 を返す。所有者だった M も `findById` で返らない |
