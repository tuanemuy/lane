# UnitOfWorkProvider

`UnitOfWorkProvider` の契約（[../../domains/common.md](../../domains/common.md) の「WriteScope」「UnitOfWorkProvider」）を、すべてのアダプターが通すケースで定める。

- 「確定する」は、`fn` が正常に終わり `run` が値を返すことを指す。「失敗する」は、`run` がそのエラーを投げることを指す
- 「`fn` は一回だけ呼ばれている」は、`run` が返るか投げるまでに、`run` が `fn` を呼んだ回数が一回であることを指す（[../../domains/common.md](../../domains/common.md) の「TransactionalRepository」の「やり直さない」）
- 「設置の UnitOfWork」は、`{ kind: "installation" }` を渡した `run` を指す。「W の UnitOfWork」は、`{ kind: "workspace", workspaceId: W, condition }` を渡した `run` を指す。条件は括弧に書く（`active`、`stableSchema` と `schemaVersion` はスキーマの版も書く）。条件を書かない W の UnitOfWork は `none`
- 設置の UnitOfWork の書き込みには `accountRepository` と `sessionRepository` を、W の UnitOfWork の書き込みには `memberRepository` と `invitationRepository` を使う
- 断りの無い読み取りは、UnitOfWork の外で、`PartitionReader` から得たリポジトリで呼ぶ
- ワークスペースと `WorkspaceSchema` の前提は、そのワークスペースの UnitOfWork（`none`）で `workspaceRepository` と `workspaceSchemaRepository` の書き込みを確定して作る。「W が `closed`」は、`closing` の W に `Workspace.close` を適用して `save` し、確定していることを指す。「W が存在しない」は、W の区画に何も確定していないことを指す
- メンバーと招待は、同じ値を、どのワークスペースの UnitOfWork にも渡せる
- メンバーは、互いに違うアカウントの ID で作る。アカウントを設置の区画に確定しておく必要は無い
- セッション S は、アカウント A のセッション。A は、そのケースで確定済みか、S と同じ UnitOfWork で `insert` する
- 「並行した二つの UnitOfWork」は、並行に実行する二つの `run` を指す。楽観ロックの `expectedVersion` は、二つの `run` の前に読む
- 「別の UnitOfWork が X を `save` して確定している」は、`version` を進める振る舞いを適用した X の `save` が確定していることを指す
- ID は `IdGenerator.generate` の値を使う
- 復旧の置き換えと重なった UnitOfWork が `ConflictError`（理由 `RESTORE_IN_PROGRESS`）で確定しないケースは、[backupStore.md](backupStore.md) が持つ
- `fn` の中から `PartitionReader` を呼べるケースは、[partitionReader.md](partitionReader.md) が持つ
- 文脈が区画の外のリポジトリを公開しないこと、文脈のリポジトリが UnitOfWork の外の記録のメソッドを持たないこと、設置の UnitOfWork の文脈が `collectEvents` を持たないことは、型が定める。ケースにしない

## 確定

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-unitOfWorkProvider-01 | ワークスペース W が確定済み | 設置の UnitOfWork と W の UnitOfWork で、それぞれ書き込みを行わず値を返す `fn` で `run` を呼ぶ | どちらの `run` も、`fn` の返した値を返す |
| TC-ports-unitOfWorkProvider-02 | ワークスペース W が確定済み | 設置の UnitOfWork で `accountRepository.insert(A)` と `sessionRepository.insert(S)` を呼んで確定し、W の UnitOfWork で `memberRepository.insert(M)` と `invitationRepository.insert(I)` を呼んで確定する | `findById` が A、S を、W の区画の `findById` が M、I を返す |
| TC-ports-unitOfWorkProvider-04 | アカウント A と、A のセッション S が確定済み。ワークスペース W にメンバー M、N が確定済み | 設置の UnitOfWork で A を `save`、S を読んだ `expectedVersion` で `delete`、新しいアカウント C を `insert` して確定する。W の UnitOfWork で M を `save`、N を `delete`、新しいメンバー P を `insert` して確定する | `findById` が、A と M は変更後の内容、S と N は `null`、C と P はそれぞれを返す |

## 失敗

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-unitOfWorkProvider-09 | アカウント A と、ワークスペース W のメンバー M が確定済み | 設置の UnitOfWork で A を `save` し、新しいアカウント C とセッション S を `insert` した後、`fn` が `NotFoundError` e を投げる。W の UnitOfWork で M を `save` し、新しいメンバー P と招待 I を `insert` した後、`fn` が例外を投げる | 一つ目の `run` が、e を別の種類へ変えずに投げる。二つ目の `run` が失敗する。A と M は変更前の内容を返す。C、S、P、I の `findById` が `null` を返す |
| TC-ports-unitOfWorkProvider-12 | アカウント A が確定済みで、`findById` で `expectedVersion` v を得ている | 設置の UnitOfWork で A を v で `save` し、新しいアカウント B を `insert` した後、`fn` が例外を投げる。次に、別の設置の UnitOfWork で A を同じ v で `save` し、同じ B を `insert` して確定する | 二つ目の `run` が確定する。`findById(A.id)` が二つ目の内容を、`findById(B.id)` が B を返す |
| TC-ports-unitOfWorkProvider-13 | アカウント A を読んで `expectedVersion` v を得た後、別の UnitOfWork が A を `save` して確定している。ワークスペース W のメンバー M についても同じ | 設置の UnitOfWork で新しいアカウント C を `insert` し、A を v で `save` する。W の UnitOfWork で招待 I を `insert` し、M を読んだ `expectedVersion` で `save` する | どちらの `run` も `ConflictError`（理由 `STALE_VERSION`）で失敗する。`fn` は一回だけ呼ばれている。C と I の `findById` が `null` を返す。A は別の UnitOfWork が保存した内容のまま |
| TC-ports-unitOfWorkProvider-14 | アカウント A が確定済みで、`findById` で `expectedVersion` v を得ている | 並行した二つの設置の UnitOfWork が、それぞれ振る舞いを適用した A を v で `save` する。それぞれの `fn` の呼び出しを数える | 一方の `run` だけが確定し、もう一方は `ConflictError`（理由 `STALE_VERSION`）で失敗する。どちらの `fn` も一回だけ呼ばれている |
| TC-ports-unitOfWorkProvider-15 | アカウント A が確定済み | 設置の UnitOfWork で `sessionRepository.insert(S)` と、A と同じメールアドレスのアカウント B の `insert` を呼ぶ。別の設置の UnitOfWork で `sessionRepository.insert(T)` と、A と同じ ID のアカウントの `insert` を呼ぶ | 一つ目の `run` が `ConflictError`（理由 `EMAIL_TAKEN`）で、二つ目の `run` が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する。どちらも `fn` は一回だけ呼ばれている。S、T、B の `findById` が `null` を返す |
| TC-ports-unitOfWorkProvider-17 | アカウント A が確定済みで、`findById` で `expectedVersion` v を得ている | 設置の UnitOfWork で、振る舞いを適用していない A を v で `save` して確定する。次に、別の設置の UnitOfWork で A を同じ v で `save` し、確定する | 二つの `run` がどちらも確定する。`findById(A.id)` の `expectedVersion` は v のまま |

## 区画

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-unitOfWorkProvider-21 | ワークスペース W、X が確定済み | W の UnitOfWork で `memberRepository.insert(M)` を呼び、確定する | W の区画の `memberRepository.findById(M.id)` が M を返す。X の区画の `memberRepository.findById(M.id)` が `null` を返す |
| TC-ports-unitOfWorkProvider-22 | ワークスペース W にメンバー M が確定済み。ワークスペース X が確定済み | X の UnitOfWork で、M と同じ ID のメンバーを `insert` し、確定する | `run` が確定する。W の区画の M は変わらない |
| TC-ports-unitOfWorkProvider-23 | ワークスペース W、X が確定済み | 並行した二つの UnitOfWork の一方が W の UnitOfWork で `memberRepository.insert(M)` を、もう一方が X の UnitOfWork で `memberRepository.insert(N)` を呼ぶ。設置の UnitOfWork の `accountRepository.insert(A)` と、W の UnitOfWork の `memberRepository.insert(P)` でも、同じに並行して呼ぶ | どの `run` も確定する |
| TC-ports-unitOfWorkProvider-25 | ワークスペース W が確定済み | 設置の UnitOfWork で `accountRepository.insert(A)` を確定した後、W の UnitOfWork で `memberRepository.insert(M)` を呼び、`fn` が例外を投げる | 二つ目の `run` が失敗する。`accountRepository.findById(A.id)` が A を返す |

## ワークスペースの条件

`none` と `active`。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-unitOfWorkProvider-27 | W が存在しない、`closing`、`closed` の三つの場合 | W の UnitOfWork（`none`）で、存在しない場合は `workspaceRepository.insert(W)` を、`closing` では `memberRepository.insert(M)` を、`closed` では書き込みを行わず値を返す `fn` を呼ぶ | どの `run` も確定する。W の区画の `find()` が W を、`findById(M.id)` が M を返し、`closed` の `run` は `fn` の返した値を返す |
| TC-ports-unitOfWorkProvider-30 | ワークスペース W が `active` | W の UnitOfWork（`active`）で `memberRepository.insert(M)` を呼ぶ。別の W の UnitOfWork（`active`）で、書き込みを行わず値を返す | どちらの `run` も確定する。`findById(M.id)` が M を返し、二つ目の `run` は `fn` の返した値を返す |
| TC-ports-unitOfWorkProvider-32 | W が `closing`、`closed`、存在しないの三つの場合 | W の UnitOfWork（`active`）で `memberRepository.insert(M)` と、ドラフト 1 件の `collectEvents` を呼ぶ | どの `run` も `ConflictError`（理由 `WORKSPACE_NOT_ACTIVE`）で失敗する。`fn` は一回だけ呼ばれている。`findById(M.id)` が `null` を返す。そのイベントが試験用の消費者に届かない |
| TC-ports-unitOfWorkProvider-35 | ワークスペース W が `closing` | W の UnitOfWork（`active`）で、書き込みを行わず値を返す | `run` が `ConflictError`（理由 `WORKSPACE_NOT_ACTIVE`）で失敗する |
| TC-ports-unitOfWorkProvider-36 | ワークスペース W が `active` で、`workspaceRepository.find` で `expectedVersion` v を得た後、別の UnitOfWork（`none`）が W を `closing` にして `save` し、確定している | W の UnitOfWork（`active`）で、読んだ W を v で `workspaceRepository.save` する | `run` が `ConflictError`（理由 `WORKSPACE_NOT_ACTIVE`）で失敗する。W は `closing` のまま |
| TC-ports-unitOfWorkProvider-37 | ワークスペース W が `active`。`workspaceRepository.find` で `expectedVersion` v を得ている | 並行した二つの UnitOfWork の一方が、W の UnitOfWork（`active`）で、読んだ W（振る舞いを適用しない）を v で `save` し、もう一方が、W の UnitOfWork（`none`）で W を `closing` にして v で `save` する | `closing` にする UnitOfWork が先に確定した場合、もう一方は `ConflictError`（理由 `WORKSPACE_NOT_ACTIVE`）で失敗する。`active` の UnitOfWork が先に確定した場合、W の `version` は v のままなので、`closing` にする UnitOfWork も確定し、W は `closing` になる |
| TC-ports-unitOfWorkProvider-38 | ワークスペース W が `active` で、`workspaceRepository.find` で `expectedVersion` v を得ている | W の UnitOfWork（`active`）で `memberRepository.insert(M)` を呼び、確定する。次に、別の W の UnitOfWork（`none`）で W を v で `save` する | 二つ目の `run` が確定する |
| TC-ports-unitOfWorkProvider-39 | ワークスペース W が `active` | 並行した二つの W の UnitOfWork（`active`）で、それぞれ `memberRepository.insert(M)` と `memberRepository.insert(N)` を呼ぶ | 二つの `run` がどちらも確定する。`findById` が M と N を返す |
| TC-ports-unitOfWorkProvider-40 | ワークスペース W が `active`。W の `WorkspaceSchema` は、スキーマの版が 3 で適用中でない | 並行した二つの UnitOfWork の一方が、W の UnitOfWork（`active`）で `memberRepository.insert(M)` を呼び、もう一方が、W の UnitOfWork（`none`）で W を `closing` にして `save` する。一方の条件を（`stableSchema`、3）にして、同じに行う | W を `closing` にする UnitOfWork は確定する。もう一方は、確定するか、`ConflictError`（理由 `WORKSPACE_NOT_ACTIVE`）で失敗するかのどちらかになる。失敗した場合、`findById(M.id)` が `null` を返す |
| TC-ports-unitOfWorkProvider-41 | ワークスペース W が `active` で、区画に `AccessPolicy` が無い。名簿の W の項目は `retiring`。W の `WorkspaceSchema` は、スキーマの版が 3 で適用中でない。もう一つのワークスペース X は `closing` で、X の `WorkspaceSchema` のスキーマの版は 5 | W の UnitOfWork（`active`）と、W の UnitOfWork（`stableSchema`、3）で、それぞれ `memberRepository.insert` を呼ぶ | どちらの `run` も確定する |
| TC-ports-unitOfWorkProvider-44 | ワークスペース W が `closing` | 設置の UnitOfWork で `accountRepository.insert(A)` を呼ぶ | `run` が確定する |

`stableSchema` と `schemaVersion`。スキーマの版と適用中の条件のケースは、[workspaceSchemaRepository.md](workspaceSchemaRepository.md) が持つ。ここでは、ワークスペースの状態との組み合わせと、条件を満たさない UnitOfWork がほかの `ConflictError` の理由にも当たる場合の理由を確かめる。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-unitOfWorkProvider-45 | ワークスペース W が `closing` または `closed`。W の `WorkspaceSchema` は、スキーマの版が 3 で適用中でない | W の UnitOfWork（`stableSchema`、3）と、W の UnitOfWork（`schemaVersion`、3）で、それぞれ `memberRepository.insert(M)` を呼ぶ | どちらの `run` も `ConflictError`（理由 `WORKSPACE_NOT_ACTIVE`）で失敗する。`findById(M.id)` が `null` を返す |
| TC-ports-unitOfWorkProvider-46 | ワークスペース W が `closing` で、W の `WorkspaceSchema` のスキーマの版が 3。または、W が存在しない | W の UnitOfWork（`stableSchema`、2）と、W の UnitOfWork（`schemaVersion`、2）で、それぞれ `memberRepository.insert(M)` を呼ぶ | どちらの `run` も `ConflictError`（理由 `WORKSPACE_NOT_ACTIVE`）で失敗する。`findById(M.id)` が `null` を返す |
| TC-ports-unitOfWorkProvider-85 | ワークスペース W にメンバー M が確定済み。W の `WorkspaceSchema` のスキーマの版は 3 で、適用中でない。その後、W を `closing` にして確定する場合と、W を `active` のまま、スキーマの版を 4 にして確定する場合 | W の UnitOfWork（`schemaVersion`、3）で、M と同じ ID のメンバーを `insert` する | `closing` の場合は `ConflictError`（理由 `WORKSPACE_NOT_ACTIVE`）、版が 4 の場合は `ConflictError`（理由 `SCHEMA_CHANGED`）で失敗する。どちらも `ALREADY_EXISTS` にしない。`fn` は一回だけ呼ばれている。M は変わらない |

## 書き込みの量

回数は、文脈のリポジトリの書き込みのメソッドの呼び出しと、`collectEvents` に渡したイベントを、それぞれ一回に数える。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-unitOfWorkProvider-53 | ワークスペース W が `active` | W の UnitOfWork（`active`）で、`UnitOfWorkCapacity.workspaceWrites` 回の半分を互いに違うメンバーの `insert` に、残りの半分を、一度の `collectEvents` に渡すドラフトの件数に充てる | `run` が確定する。`insert` したすべてのメンバーが、UnitOfWork の外の読み取りに現れ、渡したすべてのイベントが試験用の消費者に届く |
| TC-ports-unitOfWorkProvider-55 | なし | 設置の UnitOfWork で、`accountRepository.insert` を、互いに違うアカウントで `UnitOfWorkCapacity.installationWrites` 回呼ぶ | `run` が確定する。`insert` したすべてのアカウントが、UnitOfWork の外の読み取りに現れる |
| TC-ports-unitOfWorkProvider-56 | ワークスペース W が `active` | W の UnitOfWork（`active`）で、`memberRepository.insert` を、互いに違うメンバーで `UnitOfWorkCapacity.workspaceWrites` 回呼んだ後、`fn` が例外を投げる | `run` が失敗する。どのメンバーも、UnitOfWork の外の読み取りに現れない |

## ネスト

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-unitOfWorkProvider-57 | ワークスペース W が確定済み | 設置の UnitOfWork で `accountRepository.insert(A)` を呼んだ後、`fn` の中で設置の UnitOfWork の `run` を呼ぶ。同じに、外側を設置・内側を W、外側を W・内側を設置の UnitOfWork にして、内側の `fn` で書き込みを呼ぶ | どの外側の `run` も `SystemError`（理由 `NESTED_UNIT_OF_WORK`）で失敗する。外側と内側で書いた集約の `findById` が `null` を返す |
| TC-ports-unitOfWorkProvider-60 | なし | 設置の UnitOfWork で `accountRepository.insert(A)` を呼んだ後、`fn` の中で `run` を呼び、投げられた `SystemError` を `fn` が捕まえて正常に終わる | 外側の `run` が、同じ `SystemError`（理由 `NESTED_UNIT_OF_WORK`）で失敗する。`findById(A.id)` が `null` を返す |
| TC-ports-unitOfWorkProvider-61 | ワークスペース W が確定済み。ネストした `run` が `SystemError`（理由 `NESTED_UNIT_OF_WORK`）で失敗している | 新しい設置の UnitOfWork で `accountRepository.insert(A)` を呼んで確定し、その後に W の UnitOfWork の `run` を呼ぶ | 二つの `run` がどちらも確定する。`findById(A.id)` が A を返す |

## イベントの記録

配送は、試験用の消費者にイベントが届くことで確かめる。UnitOfWork は、確定済みのワークスペース W の UnitOfWork。「上限を超えるドラフト」は、ペイロードを JSON にした UTF-8 のバイト列が `EventPayloadLimit.maxBytes` より長いドラフトを、「上限以下のドラフト」は、それ以下のドラフトを指す。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-unitOfWorkProvider-63 | なし | UnitOfWork で `memberRepository.insert(M)` を呼び、`collectEvents` を二回呼んで、それぞれドラフト 2 件と 1 件を渡し、確定する。別の UnitOfWork で、書き込みを行わず、ドラフト 1 件の `collectEvents` を呼び、確定する | `findById(M.id)` が M を返す。4 件のイベントがすべて試験用の消費者に届く |
| TC-ports-unitOfWorkProvider-65 | もう一つの確定済みのワークスペース X | W の UnitOfWork と X の UnitOfWork で、同じ内容のドラフトを 1 件ずつ `collectEvents` に渡し、確定する | 試験用の消費者に 2 件が届く。W の UnitOfWork のイベントの `workspaceId` は W、X の UnitOfWork のイベントの `workspaceId` は X |
| TC-ports-unitOfWorkProvider-66 | なし | UnitOfWork で、`aggregateId` を持たないドラフト 1 件の `collectEvents` を呼び、確定する | 届いたイベントの `aggregateId` と `workspaceId` が、どちらも W |
| TC-ports-unitOfWorkProvider-68 | なし | UnitOfWork で空の列を `collectEvents` に渡し、確定する | `run` が確定する。試験用の消費者にイベントが届かない |
| TC-ports-unitOfWorkProvider-69 | メンバー N を読んで `expectedVersion` v を得た後、別の UnitOfWork が N を `save` して確定している | ドラフト 1 件の `collectEvents` を呼んだ UnitOfWork を、三つの場合で終える。`memberRepository.insert(M)` の後に `fn` が例外を投げる。N を v で `save` する。`fn` の中で `run` を呼ぶ | 三つの `run` が、それぞれ例外、`ConflictError`（理由 `STALE_VERSION`）、`SystemError`（理由 `NESTED_UNIT_OF_WORK`）で失敗する。`findById(M.id)` が `null` を返す。どのイベントも試験用の消費者に届かない |
| TC-ports-unitOfWorkProvider-72 | なし | UnitOfWork で `memberRepository.insert(M)` を呼んだ後、上限以下のドラフト 1 件と、上限を超えるドラフト 1 件を、一度の `collectEvents` に渡す | `collectEvents` が `SystemError`（理由 `EVENT_PAYLOAD_TOO_LARGE`）を投げ、`run` が同じ `SystemError` で失敗する。`findById(M.id)` が `null` を返す。どちらのイベントも試験用の消費者に届かない |
| TC-ports-unitOfWorkProvider-73 | なし | UnitOfWork で `memberRepository.insert(M)` を呼んだ後、上限を超えるドラフト 1 件の `collectEvents` を呼び、投げられた `SystemError` を `fn` が捕まえて正常に終わる | `run` が、同じ `SystemError`（理由 `EVENT_PAYLOAD_TOO_LARGE`）で失敗する。`findById(M.id)` が `null` を返す |

## イベントの消去

`deleteEvents` は、W の UnitOfWork（`none`）の文脈で呼ぶ。`count(W)` は、`PartitionReader.workspace(W)` の `quarantinedEventQuery.count()` を指す。「常に失敗する消費者」「成功する消費者」と「隔離を待つ」の書き方は [quarantinedEventQuery.md](quarantinedEventQuery.md) に従う。断りの無い `limit` は `BoundedDeletion.maxLimit`。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-unitOfWorkProvider-75 | W にイベントを一件も記録していない。区画に何も確定していないワークスペースの ID v | W の UnitOfWork と v の UnitOfWork で、それぞれ `deleteEvents` を呼び、確定する | どちらも 0 を返す。どちらの `run` も確定する |
| TC-ports-unitOfWorkProvider-76 | 常に失敗する消費者に対して、W のイベント 2 件を確定し、隔離を待つ | UnitOfWork で `deleteEvents` を呼び、確定する | 2 を返す。`count(W)` が 0 を返す |
| TC-ports-unitOfWorkProvider-77 | 常に失敗する消費者に対して、W のイベント 3 件を確定し、隔離を待つ | UnitOfWork で `deleteEvents(2)` を呼んで確定し、別の UnitOfWork でもう一度 `deleteEvents(2)` を呼んで確定する | 一度目が 2 を、二度目が 1 を返す。`count(W)` が、一度目の後に 1 を、二度目の後に 0 を返す |
| TC-ports-unitOfWorkProvider-78 | 常に失敗する消費者に対して、W のイベント 1 件と、もう一つのワークスペース X のイベント 1 件を確定し、隔離を待つ | W の UnitOfWork で `deleteEvents` を呼び、確定する | 1 を返す。`count(W)` が 0 を、`count(X)` が 1 を返す |
| TC-ports-unitOfWorkProvider-79 | 常に失敗する消費者に対して、W のイベント 1 件を確定し、隔離を待つ | UnitOfWork で `deleteEvents` を呼んだ後、`fn` が例外を投げる | `run` が失敗する。`count(W)` が 1 を返す |
| TC-ports-unitOfWorkProvider-80 | 試験用の消費者が配送を受け始める前に、W のイベント 1 件を確定している（未配送で、配送を始めていない） | UnitOfWork で `deleteEvents` を呼び、確定する。その後に、成功する消費者が配送を受け始める | 1 を返す。そのイベントは消費者に届かない |
| TC-ports-unitOfWorkProvider-81 | W の `Workspace` が `closing` で、常に失敗する消費者に対して記録した W のイベント 1 件が隔離されている | UnitOfWork で `deleteEvents` を呼び、確定する | 1 を返す |
| TC-ports-unitOfWorkProvider-83 | W にイベントを一件も記録していない | UnitOfWork で、ドラフト 1 件の `collectEvents` と `deleteEvents` を呼び、確定する | `deleteEvents` が 0 を返す。そのイベントが試験用の消費者に届く |
| TC-ports-unitOfWorkProvider-84 | なし | UnitOfWork で `deleteEvents(0)` を呼ぶ。別の UnitOfWork で、`limit` に `BoundedDeletion.maxLimit` より 1 大きい値を渡して `deleteEvents` を呼ぶ | どちらも `SystemError`（理由 `INVALID_QUERY`）で失敗する |
