# WorkspaceRepository

`WorkspaceRepository` の契約（[../../domains/workspace.md](../../domains/workspace.md)）と、`SingletonRepository` の基底の契約（[../../domains/common.md](../../domains/common.md)）を、すべてのアダプターが通すケースで定める。

- リポジトリは、ワークスペースの区画に結び付き、メソッドはワークスペースの ID を引数に取らない。区画は、`UnitOfWorkProvider.run` に渡す `WriteScope` の `workspaceId` と、`PartitionReader.workspace(workspaceId)` が決める。W、V は、互いに違うワークスペース。断りの無い呼び出しは、W の区画のリポジトリに対して行う。「V の区画」と書いた呼び出しは、V の区画のリポジトリに対して行う。「W」「V」と書いたワークスペースは、その区画に `insert` するワークスペースを指す。「W のメンバー」「W の招待」は、W の区画に `insert` するメンバーと招待を指す。V も同じ
- 書き込みは `UnitOfWorkProvider.run` の UnitOfWork の中で呼ぶ。`WriteScope` は、そのワークスペースの `workspace` で、条件は `none`。「確定する」は、`fn` が正常に終わり `run` が値を返すことを指す。「失敗する」は、`run` がそのエラーを投げることを指す
- 読み取りは、断りが無ければ UnitOfWork の外で、`PartitionReader.workspace` のリポジトリで呼ぶ
- 「並行した二つの UnitOfWork」は、並行に実行する二つの `run` を指す。楽観ロックの `expectedVersion` は、二つの `run` の前に読む
- 断りの無い `save` は、直前の `find()` が返した `expectedVersion` を渡す
- ワークスペース、方針、メンバー、招待、参加の囲いは、ドメインの振る舞い（`Workspace.create`、`AccessPolicy.create`、`Member.create`、`Invitation.issue`、`JoiningGuard.advance` など）で作る。`createdAt` は、渡す `now` で決める。メンバーと招待の ID は `IdGenerator.generate` の値を使う
- 「作成した W」は、`Workspace.create` の結果を、W の区画に `insert` して確定したワークスペースを指す
- メンバーのアカウントの ID は `IdGenerator.generate` の値から作る。設置の区画にアカウントと名簿を確定する必要は無い
- 「`closing` の W」は、`Workspace.beginClosing` を適用した W を `save` して確定したワークスペースを指す
- 「`closed` の W」は、方針、メンバー、招待、参加の囲いを持たない `closing` の W に、`Workspace.close` を適用して `save` して確定したワークスペースを指す
- 各リポジトリの `deleteAll` は、`closing` の W にだけ呼ぶ

## insert

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-workspaceRepository-01 | W の区画に何も確定していない | `Workspace.create` の結果のワークスペース W を `insert` し、確定する | `find()` が、W と全フィールドが一致するワークスペースを返す |
| TC-ports-workspaceRepository-02 | W の区画に何も確定していない | W を `insert` した後、`fn` が例外を投げる | `find()` が `null` を返す |
| TC-ports-workspaceRepository-03 | 作成した W | `createdAt` が違う `Workspace.create` の結果を、W の区画に `insert` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する。`find()` が、元の `createdAt` の W を返す |
| TC-ports-workspaceRepository-04 | 作成した W | V の区画で、`Workspace.create` の結果 V を `insert` し、確定する | `run` が確定する。W の区画の `find()` が W を、V の区画の `find()` が V を返す |
| TC-ports-workspaceRepository-05 | W の区画に何も確定していない | 並行した二つの UnitOfWork の一方が `Workspace.create` の結果を、他方が `Workspace.beginClosing(null, now)` の結果を、それぞれ `insert` する | 一方の `run` だけが確定し、他方が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する。`find()` が、確定した側のワークスペースを返す |
| TC-ports-workspaceRepository-06 | `closing` の W | `Workspace.create` の結果を、W の区画に `insert` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する。`find()` が、`closing` の W を返す |
| TC-ports-workspaceRepository-07 | `closed` の W | `Workspace.create` の結果を、W の区画に `insert` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する。`find()` が、`closed` の W を返す |
| TC-ports-workspaceRepository-08 | W の区画に何も確定していない | `Workspace.beginClosing(null, now)` の結果を `insert` し、確定する | `run` が確定する。`find()` が、`status` が `closing` のワークスペースを返す |

## find

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-workspaceRepository-10 | W の区画に何も確定していない | `find()` を呼ぶ | `null` を返す |
| TC-ports-workspaceRepository-12 | `closing` の W | `find()` を呼ぶ | `status` が `closing` のワークスペースを返す |
| TC-ports-workspaceRepository-13 | `closed` の W | `find()` を呼ぶ | `status` が `closed` のワークスペースと、`expectedVersion` を返す |
| TC-ports-workspaceRepository-14 | 作成した W。V の区画に何も確定していない | V の区画で `find()` を呼ぶ | `null` を返す |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-workspaceRepository-17 | 作成した W | `Workspace.beginClosing` を適用した W を `save` し、確定する | `run` が確定する。`find()` が `closing` のワークスペースを返す |
| TC-ports-workspaceRepository-18 | W を読んで v を得た後、別の UnitOfWork が `beginClosing` を適用した W を `save` して確定している | `beginClosing` を適用した W を v で `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）で失敗する。`find()` が、別の UnitOfWork が保存した内容を返す |
| TC-ports-workspaceRepository-19 | 作成した W | 並行した二つの UnitOfWork が、同じ `expectedVersion` で、`beginClosing` を適用した W をそれぞれ `save` する | 一方の `run` だけが確定し、他方が `ConflictError`（理由 `STALE_VERSION`）で失敗する |
| TC-ports-workspaceRepository-20 | 作成した W を読んで v を得ている | `beginClosing` を適用した W を v で `save` した後、`fn` が例外を投げる | `find()` が、`active` の W を返す |
| TC-ports-workspaceRepository-21 | 作成した V を読んで v を得ている。W の区画に何も確定していない | W の区画で、V を v で `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）で失敗する。W の区画の `find()` が `null` を返す |
| TC-ports-workspaceRepository-22 | 作成した W と、W の方針が確定済み | 方針に `AccessPolicy.requireMfa` を適用して `accessPolicyRepository.save` し、確定した後に、`find()` で W を読む | W の `expectedVersion` は、方針の保存の前と変わらない |

## save（`closed`）

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-workspaceRepository-23 | 方針、メンバー、招待、参加の囲いを持たない `closing` の W | `Workspace.close` を適用した W を `save` し、確定する | `run` が確定する。`find()` が `closed` のワークスペースを返す |
| TC-ports-workspaceRepository-24 | メンバーを持つ `closing` の W | `Workspace.close` を適用した W を `save` する | `run` が `ConflictError`（理由 `WORKSPACE_NOT_EMPTY`）で失敗する。`find()` が `closing` の W を返す |
| TC-ports-workspaceRepository-25 | `pending` の招待を持つ `closing` の W | `Workspace.close` を適用した W を `save` する | `run` が `ConflictError`（理由 `WORKSPACE_NOT_EMPTY`）で失敗する。`find()` が `closing` の W を返す |
| TC-ports-workspaceRepository-27 | 方針を持つ `closing` の W | `Workspace.close` を適用した W を `save` する | `run` が `ConflictError`（理由 `WORKSPACE_NOT_EMPTY`）で失敗する |
| TC-ports-workspaceRepository-28 | 参加の囲いを持つ `closing` の W | `Workspace.close` を適用した W を `save` する | `run` が `ConflictError`（理由 `WORKSPACE_NOT_EMPTY`）で失敗する |
| TC-ports-workspaceRepository-29 | 方針、メンバー 2 件、招待 2 件、参加の囲い 1 件を持つ `closing` の W で、四つのリポジトリの `deleteAll(1000)` を、それぞれ確定している | `Workspace.close` を適用した W を `save` し、確定する | `run` が確定する。`find()` が `closed` のワークスペースを返す |
| TC-ports-workspaceRepository-30 | メンバーと招待 2 件を持つ `closing` の W で、`memberRepository.deleteAll(1000)` だけを確定している | `Workspace.close` を適用した W を `save` する | `run` が `ConflictError`（理由 `WORKSPACE_NOT_EMPTY`）で失敗する。`find()` が `closing` の W を返す |
| TC-ports-workspaceRepository-31 | 方針、メンバー、招待、参加の囲いを持たない `closing` の W。V の区画に、ワークスペース、メンバー、招待が確定済み | `Workspace.close` を適用した W を `save` し、確定する | `run` が確定する。V の区画の `find()` が V を返す |
| TC-ports-workspaceRepository-32 | メンバー 2 件と招待 2 件を持つ `closing` の W | 一つの UnitOfWork で、`memberRepository.deleteAll(1000)`、`invitationRepository.deleteAll(1000)`、`Workspace.close` を適用した W の `save` を呼び、確定する | `run` が確定する。`find()` が `closed` のワークスペースを返す。メンバーと招待が、それぞれのリポジトリの `findById` で返らない |
| TC-ports-workspaceRepository-33 | メンバー 2 件と招待 2 件を持つ `closing` の W | 一つの UnitOfWork で、`memberRepository.deleteAll(1000)` と、`Workspace.close` を適用した W の `save` を呼ぶ | `run` が `ConflictError`（理由 `WORKSPACE_NOT_EMPTY`）で失敗する。`find()` が `closing` の W を返す。メンバー 2 件が `memberRepository.findById` で返る |
| TC-ports-workspaceRepository-35 | 方針、メンバー、招待、参加の囲いを持たない `closing` の W | 並行した二つの UnitOfWork が、同じ `expectedVersion` で、`Workspace.close` を適用した W をそれぞれ `save` する | 一方の `run` だけが確定し、他方が `ConflictError`（理由 `STALE_VERSION`）で失敗する。`find()` が `closed` のワークスペースを返す |
| TC-ports-workspaceRepository-36 | 方針、メンバー、招待、参加の囲いを持たない `closing` の W | `Workspace.close` を適用した W を `save` した後、`fn` が例外を投げる | `find()` が `closing` の W を返す |
| TC-ports-workspaceRepository-37 | `closed` の W | `memberRepository.findPage` と `invitationRepository.findPendingPage` を、それぞれ 1 ページ目で呼ぶ | どちらも、`items` が空の列、`count` が 0 |
