# AuditLogRepository

`AuditLogRepository` の契約（[../../domains/audit.md](../../domains/audit.md)）を、すべてのアダプターが通すケースで定める。

- リポジトリは、ワークスペースの区画に結び付き、メソッドはワークスペースの ID を引数に取らない。記録は、ワークスペースの ID のフィールドを持たない。「W の記録」は、W の区画に追記した記録を指す。断りの無い呼び出しは、W の区画のリポジトリに対して行う。「V の区画」と書いた呼び出しと、V の記録の書き込みは、V の区画のリポジトリに対して行う
- 書き込み（`append`、`appendMany`、`deleteAll`）は `UnitOfWorkProvider.run` の UnitOfWork の中で、文脈のリポジトリで呼ぶ。`WriteScope` は、そのワークスペースの `workspace` で、条件は `none`。「確定する」は、`fn` が正常に終わり `run` が値を返すことを指す。「失敗する」は、`run` がそのエラーを投げることを指す
- 読み取りは、断りが無ければ UnitOfWork の外で呼ぶ
- 「並行した二つの UnitOfWork」は、並行に実行する二つの `run` を指す。楽観ロックの `expectedVersion` は、二つの `run` の前に読む
- 記録は `AuditLogEntry.record` で作る。`occurredAt` は、渡す `now` で決める。絞り込みの条件は `AuditLogFilter.create` で作る。ID は `IdGenerator.generate` の値を使う
- 「条件なし」は、`AuditLogFilter.create({})` の結果を指す
- メンバーのアカウントの ID は `IdGenerator.generate` の値から作る。設置の区画にアカウントを確定する必要は無い
- ワークスペース W、V は、それぞれの区画に `workspaceRepository.insert` で確定済みとする。「`closing` の W」は、`Workspace.beginClosing` を適用した W を `workspaceRepository.save` して確定したワークスペースを指す。記録を持つ `closing` の W は、記録を追記して確定した後に `closing` にする
- `deleteAll` は、一回の呼び出しごとに一つの UnitOfWork で確定する

## append

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-auditLogRepository-01 | なし | W の記録 E を `append` し、確定する | `findById(E.id)` が、E と全フィールドが一致する記録を返す |
| TC-ports-auditLogRepository-02 | なし | E を `append` した後、`fn` が例外を投げる | `findById(E.id)` が `null` を返す。条件なしの `findPage` の `count` が 0 |
| TC-ports-auditLogRepository-03 | なし | 一つの UnitOfWork で、W の記録 E1、E2、E3 を `append` し、確定する | 3 件すべてが `findById` で返る。条件なしの `findPage` の `count` が 3 |
| TC-ports-auditLogRepository-04 | W のメンバー N が確定済み | 一つの UnitOfWork で、権限の組を変えた N の `memberRepository.save` と、E の `append` を呼び、確定する | `memberRepository.findById(N.id)` が変更後の内容を、`findById(E.id)` が E を返す |
| TC-ports-auditLogRepository-05 | W のメンバー N を読んで `expectedVersion` v を得た後、別の UnitOfWork が N を `save` して確定している | 一つの UnitOfWork で、N の v での `memberRepository.save` と、E の `append` を呼ぶ | `run` が `ConflictError`（理由 `STALE_VERSION`）で失敗する。`findById(E.id)` が `null` を返す |
| TC-ports-auditLogRepository-06 | W のメンバー N が確定済み | 一つの UnitOfWork で、権限の組を変えた N の `memberRepository.save` と、E の `append` を呼んだ後、`fn` が例外を投げる | `memberRepository.findById(N.id)` が変更前の内容を、`findById(E.id)` が `null` を返す |
| TC-ports-auditLogRepository-07 | E が確定済み | E と同じ ID で、内容が違う W の記録 F を `append` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する。`findById(E.id)` が E の内容を返す |
| TC-ports-auditLogRepository-08 | W の E が確定済み | V の区画で、E と同じ ID の、V の記録 F を `append` し、確定する | `run` が確定する。W の区画の `findById(E.id)` が E を、V の区画の `findById(F.id)` が F を返す |
| TC-ports-auditLogRepository-09 | W のメンバー N が確定済み | 一つの UnitOfWork で、N の `memberRepository.save` と、確定済みの記録と同じ ID の記録の `append` を呼ぶ | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する。`memberRepository.findById(N.id)` が変更前の内容を返す |
| TC-ports-auditLogRepository-10 | なし | 並行した二つの UnitOfWork が、W の、ID が違う記録 E1、E2 をそれぞれ `append` する | 二つの `run` がどちらも確定する。E1 と E2 が `findById` で返る |
| TC-ports-auditLogRepository-11 | なし | E を `append` し、確定した直後に `findById(E.id)` と条件なしの `findPage` を呼ぶ | `findById` が E を返し、`findPage` の `items` が E を含む |

## appendMany

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-auditLogRepository-12 | なし | `appendMany([])` を呼び、確定する | `run` が確定する。条件なしの `findPage` の `count` が 0 |
| TC-ports-auditLogRepository-13 | なし | W の記録 E1、E2、E3 を、一回の `appendMany` で追記し、確定する | 3 件すべてが `findById` で、全フィールドが一致して返る。条件なしの `findPage` の `count` が 3 |
| TC-ports-auditLogRepository-14 | なし | E1、E2 を `appendMany` した後、`fn` が例外を投げる | E1 と E2 の `findById` が `null` を返す |
| TC-ports-auditLogRepository-15 | E が確定済み | E と同じ ID の記録と、新しい記録 F を、一回の `appendMany` に渡す | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する。`findById(F.id)` が `null` を返す。E の内容は変わらない |
| TC-ports-auditLogRepository-16 | なし | 同じ ID の二つの記録を、一回の `appendMany` に渡す | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）で失敗する。どちらも追記しない |
| TC-ports-auditLogRepository-17 | W のメンバー N が確定済み | 一つの UnitOfWork で、権限の組を変えた N の `memberRepository.save` と、E1、E2 の `appendMany` を呼んだ後、`fn` が例外を投げる | `memberRepository.findById(N.id)` が変更前の内容を返す。E1 と E2 の `findById` が `null` を返す |
| TC-ports-auditLogRepository-18 | なし | 一つの UnitOfWork で、E1 の `append` と、E2、E3 の `appendMany` を呼び、確定する | 3 件すべてが `findById` で返る |
| TC-ports-auditLogRepository-19 | なし | 1,000 件の記録を、一回の `appendMany` で追記し、確定する | `run` が確定する。条件なしの `findPage` の `count` が 1,000 |

## findById

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-auditLogRepository-20 | なし | 保存していない ID で `findById` を呼ぶ | `null` を返す |
| TC-ports-auditLogRepository-21 | W の E と、V の F が確定済み | W の区画と V の区画で、それぞれ `findById(F.id)` を呼ぶ | W の区画は `null` を返し、ほかのワークスペースの記録を返さない。V の区画は F を返す |
| TC-ports-auditLogRepository-23 | `actor` が `member`（`memberId` と `name`）で、`operation` が `changes` を持たない種類の E が確定済み | `findById(E.id)` を呼ぶ | `actor` の `kind`、`memberId`、`name` が E と一致し、`changes` を持たない記録を返す |
| TC-ports-auditLogRepository-24 | `actor` が `appToken`（`tokenId`、`binding` が `app` の `token`、`issuerMemberId`、`issuerName`）の E が確定済み | `findById(E.id)` を呼ぶ | `actor` の五つのフィールドが E と一致する記録を返す |
| TC-ports-auditLogRepository-25 | `actor` が `memberToken`（`tokenId`、`binding` が `member` の `token`、`memberId`、`memberName`）の E が確定済み | `findById(E.id)` を呼ぶ | `actor` の五つのフィールドが E と一致する記録を返す |
| TC-ports-auditLogRepository-26 | `actor` が `operator` の E が確定済み | `findById(E.id)` を呼ぶ | `actor.kind` が `operator` の記録を返す |
| TC-ports-auditLogRepository-27 | `operation` の対象が `MemberTarget`、`InvitationTarget`、`TokenTarget`（`token` の `binding` が `app` と `member` のそれぞれ）、`SchemaTarget`、`FormTarget`、`WebhookEndpointTarget`、`WorkspaceTarget` の記録が 1 件ずつ確定済み | それぞれ `findById` を呼ぶ | どの記録も、`operation.type` と、対象の全フィールドが保存した値と一致する |
| TC-ports-auditLogRepository-28 | `operation` の `changes` が、`before` が `null` で `after` が文字列の項目、`before` と `after` が文字列の列の項目、`before` が `null` で `after` が数の項目、`after` が空の列の項目、`SecretChange` の項目を持つ E が確定済み | `findById(E.id)` を呼ぶ | `changes` のすべての項目が、`field` と値を含めて E と一致する記録を返す。空の列は空の列のまま返る。`SecretChange` の項目は変えた事実だけを持ち、値を持たない |
| TC-ports-auditLogRepository-31 | `occurredAt` がミリ秒の端数を持つ E が確定済み | `findById(E.id)` を呼ぶ | `occurredAt` が、ミリ秒まで E と一致する |

## findPage の絞り込み

W に、次の記録が確定済みとする。`occurredAt` は t1 < t2 < t3 < t4 < t5。

- E1: `occurredAt` が t1、`actor` が `member`（M1）、`operation.type` が `member.invited`
- E2: `occurredAt` が t2、`actor` が `member`（M2）、`operation.type` が `member.removed`
- E3: `occurredAt` が t3、`actor` が `appToken`（T1、発行者 M1）、`operation.type` が `schema.reflected`
- E4: `occurredAt` が t4、`actor` が `operator`、`operation.type` が `invitation.created`
- E5: `occurredAt` が t5、`actor` が `appToken`（T3、発行者 M2）、`operation.type` が `bootstrap.setting_bound`

V に、`occurredAt` が t3、`actor` が `member`（M1）、`operation.type` が `member.invited` の記録 F が確定済みとする。操作は `findPage(条件, { page: 1, limit: 10 })`。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-auditLogRepository-32 | 上の記録 | 条件なし | `items` が E5、E4、E3、E2、E1 の順、`count` が 5。F を含まない |
| TC-ports-auditLogRepository-33 | 上の記録 | `from` が t3。続けて、`from` が t3 の 1 ミリ秒後 | 一つ目の `items` が E5、E4、E3、`count` が 3。二つ目の `items` が E5、E4、`count` が 2 |
| TC-ports-auditLogRepository-35 | 上の記録 | `to` が t3。続けて、`to` が t3 の 1 ミリ秒後 | 一つ目の `items` が E2、E1、`count` が 2。二つ目の `items` が E3、E2、E1、`count` が 3 |
| TC-ports-auditLogRepository-37 | 上の記録 | `from` が t2、`to` が t4 | `items` が E3、E2、`count` が 2 |
| TC-ports-auditLogRepository-38 | 上の記録 | `from` と `to` がどちらも t3 | `items` が空の列、`count` が 0 |
| TC-ports-auditLogRepository-39 | 上の記録 | `from` が t5 の 1 ミリ秒後 | `items` が空の列、`count` が 0 |
| TC-ports-auditLogRepository-40 | 上の記録 | `operationTypes` が `["member.invited"]` | `items` が E1 だけ、`count` が 1 |
| TC-ports-auditLogRepository-41 | 上の記録 | `operationTypes` が `["member.invited", "schema.reflected"]` | `items` が E3、E1、`count` が 2 |
| TC-ports-auditLogRepository-42 | 上の記録 | `operationTypes` が `["token.issued"]` | `items` が空の列、`count` が 0 |
| TC-ports-auditLogRepository-43 | 上の記録 | `operationTypes` が空の列。続けて、`operationTypes` が空の列で、`actor` が `member`（M1） | どちらも `items` が空の列、`count` が 0 |
| TC-ports-auditLogRepository-44 | 上の記録 | `operationTypes` が `null` | `items` が E5、E4、E3、E2、E1、`count` が 5 |
| TC-ports-auditLogRepository-45 | 上の記録 | `operationTypes` が `["member.invited", "member.invited"]` | `items` が E1 だけ、`count` が 1 |
| TC-ports-auditLogRepository-47 | 上の記録 | `actor` が `member`（M1）、`member`（M2）、`member`（記録に無いメンバー）のそれぞれ | M1 は `items` が E1 だけで、M1 が発行したトークンが実行者の E3 を含まない。M2 は `items` が E2 だけ。記録に無いメンバーは `items` が空の列、`count` が 0 |
| TC-ports-auditLogRepository-50 | 上の記録 | `actor` が `token`（T1）。続けて、`actor` が `token`（記録に無いトークン） | 一つ目の `items` が E3 だけ、`count` が 1。二つ目の `items` が空の列、`count` が 0 |
| TC-ports-auditLogRepository-52 | 上の記録に加えて、`actor` が `memberToken`（T2、紐付くメンバー M2）の E6 が確定済み | `actor` が `token`（T2）、続けて `actor` が `member`（M2） | 一つ目の `items` が E6 だけ。二つ目の `items` が E2 だけで、E6 を含まない |
| TC-ports-auditLogRepository-53 | 上の記録 | `actor` が `operator` | `items` が E4 だけ、`count` が 1 |
| TC-ports-auditLogRepository-54 | 上の記録 | `from` が t1、`to` が t3、`operationTypes` が `["member.invited", "member.removed", "schema.reflected"]`、`actor` が `member`（M2）。続けて、`operationTypes` が `["schema.reflected"]`、`actor` が `member`（M1） | 一つ目の `items` が E2 だけ、`count` が 1。二つ目の `items` が空の列、`count` が 0 |
| TC-ports-auditLogRepository-56 | 上の記録 | 上の各条件で返った `items` | すべての記録で `AuditLogEntry.matches(記録, 条件)` が真。W の記録のうち `items` に無いものは、`matches` が偽 |
| TC-ports-auditLogRepository-57 | 上の記録に加え、W に `operation.type` が `member.joined` の E6 と `schema.reflected` の E7 が、t5 より後に確定済み | `operationTypes` が `["member.joined"]`。続けて、`operationTypes` が `["schema.reflected"]` | 一つ目の `items` が E6 だけ、`count` が 1。二つ目の `items` が E7、E3、`count` が 2 |

## findPage の並び順とページング

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-auditLogRepository-59 | W の記録が無い。V の記録だけが確定済み | 条件なしで `findPage(..., { page: 1, limit: 10 })` を呼ぶ | `items` が空の列、`count` が 0 |
| TC-ports-auditLogRepository-61 | 区画に何も確定していないワークスペースの ID Z | Z の区画で、条件なしで `findPage(..., { page: 1, limit: 10 })` を呼ぶ | `items` が空の列、`count` が 0 |
| TC-ports-auditLogRepository-62 | W の記録 E が 1 件確定済み | 条件なしで `{ page: 1, limit: 10 }` と `{ page: 1, limit: 1 }` を呼ぶ | どちらも `items` が E だけ、`count` が 1 |
| TC-ports-auditLogRepository-63 | `occurredAt` が t1 < t2 < t3 の E1、E2、E3 を、E2、E3、E1 の順に `append` して確定している | 条件なしで `findPage(..., { page: 1, limit: 10 })` を呼ぶ | `items` が E3、E2、E1 の順 |
| TC-ports-auditLogRepository-64 | E1、E2 が `occurredAt` t1 で、E3 が t2（t1 < t2）で確定済み | 条件なしで `findPage(..., { page: 1, limit: 10 })` を呼ぶ | `items` の先頭が E3。続く E1、E2 が ID の昇順。`count` が 3 |
| TC-ports-auditLogRepository-66 | `occurredAt` が同じ 3 件が確定済み | 条件なしで `{ page: 1, limit: 2 }` と `{ page: 2, limit: 2 }` | 一つ目の `items` が ID の小さい 2 件、二つ目が残りの 1 件 |
| TC-ports-auditLogRepository-67 | `occurredAt` が t1 < t2 < t3 の E1、E2、E3 が確定済み | 条件なしで `{ page: 1, limit: 3 }` | `items` が E3、E2、E1、`count` が 3 |
| TC-ports-auditLogRepository-68 | `occurredAt` が t1 < t2 < t3 の E1、E2、E3 が確定済み | 条件なしで `{ page: 1, limit: 2 }`、`{ page: 2, limit: 2 }`、`{ page: 3, limit: 1 }` を、それぞれ呼ぶ | 一つ目の `items` が E3、E2、二つ目と三つ目が E1 だけ。どれも `count` が 3 |
| TC-ports-auditLogRepository-70 | 同上 | 条件なしで `{ page: 3, limit: 2 }` | `items` が空の列、`count` が 3 |
| TC-ports-auditLogRepository-72 | `operation.type` が `member.invited` の記録 3 件と、`member.removed` の記録 2 件が確定済み | `operationTypes` が `["member.invited"]` で `{ page: 1, limit: 2 }` | `items` が、`member.invited` の記録のうち `occurredAt` が新しい 2 件。`count` が 3 |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-auditLogRepository-74 | 記録を持たない `closing` の W | `deleteAll(10)` を呼び、確定する | 0 を返す |
| TC-ports-auditLogRepository-75 | 実行者と操作の種類が違う記録 3 件を持つ `closing` の W | `deleteAll(3)` を呼び、確定する | 3 を返す。3 件とも `findById` が `null` を返す。条件なしの `findPage` の `count` が 0 |
| TC-ports-auditLogRepository-77 | 記録 3 件を持つ `closing` の W | `deleteAll(2)` を三回、それぞれ確定する | 順に 2、1、0 を返す。一回目の後、条件なしの `findPage` の `count` が 1。三回目の後、`count` が 0 |
| TC-ports-auditLogRepository-78 | 記録 2 件を持つ `closing` の W | `deleteAll(BoundedDeletion.maxLimit)` を呼び、確定する | 2 を返す |
| TC-ports-auditLogRepository-79 | 記録 2 件を持つ `closing` の W と、記録 F を持つ V | `deleteAll(10)` を呼び、確定する | 2 を返す。V の区画の `findById(F.id)` が F を返す |
| TC-ports-auditLogRepository-81 | 記録 E を持つ `closing` の W | `deleteAll` を、`limit` が 0、-1、1.5、`BoundedDeletion.maxLimit` を超える値で、それぞれ呼ぶ | どれも `run` が `SystemError`（理由 `INVALID_QUERY`）で失敗する。`findById(E.id)` が E を返す |
| TC-ports-auditLogRepository-83 | 記録 2 件を持つ `closing` の W | `deleteAll(10)` を呼んだ後、`fn` が例外を投げる | 条件なしの `findPage` の `count` が 2 |
| TC-ports-auditLogRepository-84 | 区画に何も確定していないワークスペースの ID Z | Z の区画で `deleteAll(10)` を呼び、確定する | 0 を返す |
