# AccessPolicyRepository

「上限」は、その節のメソッドの契約が定める、一度に渡せる件数・個数、または `limit` の上限を指す。値は、ポートを定めるドメインのファイルが持つ。

`AccessPolicyRepository` の契約（[../../domains/workspace.md](../../domains/workspace.md)）と、`SingletonRepository` の基底の契約（[../../domains/common.md](../../domains/common.md)）を、すべてのアダプターが通すケースで定める。

- リポジトリは、ワークスペースの区画に結び付き、メソッドはワークスペースの ID を引数に取らない。区画は、`UnitOfWorkProvider.run` に渡す `WriteScope` の `workspaceId` と、`PartitionReader.workspace(workspaceId)` が決める。W、V は、互いに違うワークスペース。断りの無い呼び出しは、W の区画のリポジトリに対して行う。「V の区画」と書いた呼び出しは、V の区画のリポジトリに対して行う
- 書き込みは `UnitOfWorkProvider.run` の UnitOfWork の中で呼ぶ。`WriteScope` は、そのワークスペースの `workspace` で、条件は `none`。「確定する」は、`fn` が正常に終わり `run` が値を返すことを指す。「失敗する」は、`run` がそのエラーを投げることを指す
- 読み取りは、断りが無ければ UnitOfWork の外で、`PartitionReader.workspace` のリポジトリで呼ぶ
- 「並行した二つの UnitOfWork」は、並行に実行する二つの `run` を指す。楽観ロックの `expectedVersion` は、二つの `run` の前に読む
- 断りの無い `save` は、直前の `find()` が返した `expectedVersion` を渡す
- 方針は `AccessPolicy` の振る舞いで、メンバーは `Member.create` で作る。メンバーのアカウントの ID は `IdGenerator.generate` の値から作る。設置の区画にアカウントを確定する必要は無い。「W のメンバー」は、W の区画に `memberRepository.insert` するメンバーを指す
- 「W の方針 P」は、`AccessPolicy.create` の結果を、W の区画に `insert` して確定した方針を指す。「所有者 M の P」は、一つの UnitOfWork で W のメンバー M を `memberRepository.insert` し、`AccessPolicy.designateOwner` で M を所有者に指定した P を `save` して確定した方針を指す
- `deleteAll` は、一回の呼び出しごとに一つの UnitOfWork で確定する

## insert

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-accessPolicyRepository-01 | W の区画に何も確定していない | `AccessPolicy.create` の結果 P を `insert` し、確定する | `find()` が、P と全フィールドが一致する方針を返す |
| TC-ports-accessPolicyRepository-02 | W の区画に何も確定していない | P を `insert` した後、`fn` が例外を投げる | `find()` が `null` を返す |
| TC-ports-accessPolicyRepository-03 | W の方針 P | `AccessPolicy.create` の結果を、W の区画に `insert` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する |
| TC-ports-accessPolicyRepository-04 | W の区画に何も確定していない | 並行した二つの UnitOfWork が、それぞれ方針を `insert` する | 一方の `run` だけが確定し、他方が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する |
| TC-ports-accessPolicyRepository-05 | W の方針 P | V の区画で、方針を `insert` し、確定する | `run` が確定する。W の区画と V の区画の `find()` が、それぞれの方針を返す |
| TC-ports-accessPolicyRepository-06 | W の区画に何も確定していない | 一つの UnitOfWork で、`AccessPolicy.create` の結果に `MemberAdministration.designateInitialOwner`（`existing` は `null`）を適用し、結果の方針を `insert` し、`owner` の M を `memberRepository.insert` して確定する | `run` が確定する。`find()` が、`owner.status` が `designated`、`owner.memberId` が M の ID の方針を返す |
| TC-ports-accessPolicyRepository-07 | W の区画に何も確定していない | `AccessPolicy.create` の結果に `MemberAdministration.designateInitialOwner`（`existing` は `null`）を適用し、結果の方針だけを `insert` する。`owner` のメンバーを保存しない | `run` が `ConflictError`（理由 `OWNER_NOT_MEMBER`）で失敗する。`find()` が `null` を返す |
| TC-ports-accessPolicyRepository-08 | メンバー N が、V の区画に確定済み。W の区画に何も確定していない | `AccessPolicy.designateOwner` で N を所有者に指定した方針を、W の区画に、メンバーを保存せずに `insert` する | `run` が `ConflictError`（理由 `OWNER_NOT_MEMBER`）で失敗する |

## find

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-accessPolicyRepository-09 | W の区画に何も確定していない | `find()` を呼ぶ | `null` を返す |
| TC-ports-accessPolicyRepository-12 | W の方針 P。V の区画に何も確定していない | V の区画で `find()` を呼ぶ | `null` を返す |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-accessPolicyRepository-13 | W の方針 P | 一つの UnitOfWork で、W のメンバー M を `memberRepository.insert` し、`AccessPolicy.designateOwner` で M を所有者に指定した P を `save` して確定する | `run` が確定する。`find()` の `owner.memberId` が M の ID |
| TC-ports-accessPolicyRepository-14 | 所有者 M の P と、W のメンバー N が確定済み | `AccessPolicy.changeOwner` で所有者を N にした P を `save` し、確定する | `run` が確定する。`find()` の `owner.memberId` が N の ID |
| TC-ports-accessPolicyRepository-15 | W の方針 P | 保存していないメンバーを `AccessPolicy.designateOwner` で所有者に指定した P を `save` する | `run` が `ConflictError`（理由 `OWNER_NOT_MEMBER`）で失敗する。`find()` が、`owner.status` が `vacant` の方針を返す |
| TC-ports-accessPolicyRepository-16 | 所有者 M の P と、V の区画のメンバー N が確定済み | `AccessPolicy.changeOwner` で N を所有者にした P を、W の区画で `save` する | `run` が `ConflictError`（理由 `OWNER_NOT_MEMBER`）で失敗する。`find()` の `owner.memberId` が M の ID |
| TC-ports-accessPolicyRepository-17 | 所有者 M の P と、W のメンバー N が確定済み | 並行した二つの UnitOfWork の一方が、所有者を N にした P を `save` し、他方が N を `memberRepository.delete` する | 一方の `run` だけが確定し、他方が `ConflictError` で失敗する。確定後、`find()` の `owner.memberId` が指すメンバーが `memberRepository.findById` で返る |
| TC-ports-accessPolicyRepository-18 | 所有者 M の P と、W のメンバー N が確定済み | 一つの UnitOfWork で、所有者を N にした P を `save` し、N を `memberRepository.delete` する | `run` が `ConflictError` で失敗する。理由は `OWNER_NOT_MEMBER` または `MEMBER_IS_OWNER` のどちらか。`find()` の `owner.memberId` が M の ID。`memberRepository.findById(N.id)` が N を返す |
| TC-ports-accessPolicyRepository-19 | 所有者 M の P | `AccessPolicy.clearOwner` を適用した P を `save` し、確定する | `run` が確定する。`find()` の `owner.status` が `vacant`。`memberRepository.findById(M.id)` が M を返す |
| TC-ports-accessPolicyRepository-20 | 所有者 M の P | 一つの UnitOfWork で、`AccessPolicy.clearOwner` を適用した P を `save` し、M を `memberRepository.delete` して確定する | `run` が確定する。`find()` の `owner.status` が `vacant`。`memberRepository.findById(M.id)` が `null` を返す |
| TC-ports-accessPolicyRepository-21 | 所有者 M の P | `AccessPolicy.requireMfa` を適用した P を `save` し、確定する | `find()` が、`mfaRequired` が `true` の方針を返す |
| TC-ports-accessPolicyRepository-24 | P を読んで v を得た後、別の UnitOfWork が `requireMfa` を適用した P を `save` して確定している | `liftMfaRequirement` を適用した P を v で `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）で失敗する。`find()` が、別の UnitOfWork が保存した内容を返す |
| TC-ports-accessPolicyRepository-25 | 所有者 M の P | 並行した二つの UnitOfWork が、同じ `expectedVersion` で、`requireMfa` を適用した P をそれぞれ `save` する | 一方の `run` だけが確定し、他方が `ConflictError`（理由 `STALE_VERSION`）で失敗する |
| TC-ports-accessPolicyRepository-26 | 所有者 M の P を読んで v を得ている | `requireMfa` を適用した P を v で `save` した後、`fn` が例外を投げる | `find()` が、`mfaRequired` が `false` の方針を返す |
| TC-ports-accessPolicyRepository-27 | 所有者 M の P。W が `active` | 並行した二つの UnitOfWork の一方が、`requireMfa` を適用した P を `save` し、他方が `Workspace.beginClosing` を適用した W を `workspaceRepository.save` する。どちらも条件は `none` | 両方の `run` が確定する |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-accessPolicyRepository-28 | 所有者 M の P。W は `closing` | `deleteAll(10)` を呼び、確定する | 1 を返す。`find()` が `null` を返す |
| TC-ports-accessPolicyRepository-29 | W の区画に方針が無い | `deleteAll(10)` を呼び、確定する | 0 を返す |
| TC-ports-accessPolicyRepository-30 | W の方針 P と、V の方針 Q | W の区画で `deleteAll(10)` を呼び、確定する | 1 を返す。V の区画の `find()` が Q を返す |
| TC-ports-accessPolicyRepository-31 | W の方針 P | `deleteAll(10)` を呼んだ後、`fn` が例外を投げる | `find()` が P を返す |
| TC-ports-accessPolicyRepository-32 | W の方針 P | `deleteAll(0)` と、上限より 1 大きい `limit` の `deleteAll` を、それぞれ別の UnitOfWork で呼ぶ | どちらの `run` も `SystemError`（理由 `INVALID_QUERY`）で失敗する。`find()` が P を返す |
| TC-ports-accessPolicyRepository-33 | 所有者 M の P を読んで v を得た後、別の UnitOfWork が P を `save` して確定している | `deleteAll(10)` を呼び、確定する | 1 を返す |
