# ChangeNotificationRepository

「上限」は、その節のメソッドの契約が定める、一度に渡せる件数・個数、または `limit` の上限を指す。値は、ポートを定めるドメインのファイルが持つ。

契約は [webhook.md](../../domains/webhook.md) の「ポート」と「トランザクション境界」、[common.md](../../domains/common.md) の、すべてのリポジトリに共通の規則、`UnitOfWorkProvider` が定める。

## 前提

- 通知は `ChangeNotification.create`（`id`、`publication` と、作成の日時）で作る
- `insert` / `deleteAll` は `UnitOfWorkProvider.run` の中で呼ぶ。断りが無ければ、一つの操作を一つの UnitOfWork で確定し、読み取りは UnitOfWork の外で呼ぶ
- UnitOfWork の `WriteScope` は、どのケースも、対象のワークスペースの `{ kind: "workspace", workspaceId, condition: { kind: "none" } }`。区画と条件は、`unitOfWorkProvider.md` が確かめる
- リポジトリは、ワークスペースの区画に結び付く。断りが無ければ、書き込みは W1 の UnitOfWork で、読み取りは `PartitionReader.workspace(W1)` のリポジトリで行う。W2 の通知の書き込みは W2 の UnitOfWork で、読み取りは W2 の区画のリポジトリで行う。メソッドは、ワークスペースの ID を取らない。通知は `workspaceId` を持たない。「W1 の通知」は、W1 の UnitOfWork で保存する通知を指す
- 「保存済み」は、`insert` を確定した状態を指す
- ID の順は、文字列のコードポイントの昇順
- 「並行して」は、二つの UnitOfWork を同時に始め、どちらも確定まで進めることを指す
- 競合と違反は、メソッドの呼び出しか確定のどちらかで現れる。`run` が `ConflictError` を投げることと、その UnitOfWork の書き込みが一つも反映されないことで確かめる
- W1、W2 は別の、確定済みのワークスペース。W0 は、区画が無いワークスペース（存在しない ID）
- 断りが無ければ、通知は W1 のもので、通し番号と操作の ID は通知ごとに別の値。ポートは送信の存在を担保しないので、`WebhookDeliveryRepository` への保存を前提にしない

## insert

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-changeNotificationRepository-01 | 通知が無い | 次の通知を `insert` する。`source` が `content` で、変更が 2 件（一意な値の変更を持つ）、参照元が 2 件の N1。`source` が `schemaApply`（`schemaVersion` は 7）で、参照元が 0 件、変更の `uniqueValues` が 0 件の N2。`actor` が `member`、`appToken`、`memberToken`、`operator`、`system` の通知を 1 件ずつ | それぞれの `findById` が、`sequence`、`operationId`、`source`（`kind` と `schemaVersion`）、`actor`（種類と ID）、`changeList`（すべての `changes` と `referrers`。空の列を含む）、`occurredAt`、`createdAt` を含めて、渡した通知と等しいエンティティを返す |
| TC-ports-changeNotificationRepository-02 | 通知が無い | 変更が 150 件、参照元が 600 件の通知 N を `insert` する | `findById(N.id)` が、150 件の `changes` と 600 件の `referrers` を、渡した順で持つ、N と等しいエンティティを返す。切り詰めない |
| TC-ports-changeNotificationRepository-06 | N が保存済み | N と同じ ID で、操作の ID が別の通知を `insert` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。`findById(N.id)` は N を返す |
| TC-ports-changeNotificationRepository-07 | W1 に、操作の ID が O の N が保存済み | W1 の、操作の ID が O で ID が別の通知 P を `insert` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。`findById(P.id)` は `null`。`findByOperationId(O)` は N を返す |
| TC-ports-changeNotificationRepository-08 | W1 に、操作の ID が O の N が保存済み | W2 の UnitOfWork で、W2 の、操作の ID が O の通知 P を `insert` する | 確定する。W2 の区画の `findById(P.id)` は P を返す |
| TC-ports-changeNotificationRepository-09 | W1 に通し番号 5 の N が保存済み | W1 の、通し番号 5 で、ID と操作の ID が別の通知 P を `insert` する | 確定する。`findById(P.id)` は P を返す。通し番号の一意性を確かめない |
| TC-ports-changeNotificationRepository-10 | 通知が無い | W1 の、操作の ID が同じで ID が別の N と P を、並行して `insert` する | 一方だけが確定し、他方の `run` は `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。`findByOperationId` は確定した方を返し、確定しなかった方の `findById` は `null` |
| TC-ports-changeNotificationRepository-12 | W1 に、操作の ID が O の N を保存し、`deleteAll(10)` で削除してある | W1 の、操作の ID が O で ID が別の通知を `insert` する | 確定する |

## findById

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-changeNotificationRepository-13 | 通知が無い | 未使用の ID で `findById` を呼ぶ | `null` を返す |
| TC-ports-changeNotificationRepository-14 | W2 の N が保存済み | W1 の区画のリポジトリと、W2 の区画のリポジトリで、`findById(N.id)` を呼ぶ | W1 の区画は `null` を、W2 の区画は N を返す。`version` を返さない |
| TC-ports-changeNotificationRepository-15 | `changeList` の変更と参照元が、同じ区画の `ContentRepository` に存在しないコンテンツID を指す N が保存済み | `findById(N.id)` を呼ぶ | すべての変更と参照元を含めて、N と等しいエンティティを返す。コンテンツの存在を確かめない |
| TC-ports-changeNotificationRepository-17 | W0 は区画が無い | W0 の区画のリポジトリで `findById`、`findByIds`、`findByOperationId` を呼ぶ | `findById` と `findByOperationId` は `null`、`findByIds` は空の列を返す |

## findByIds

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-changeNotificationRepository-18 | W1 に、通し番号 2、7、4 の N1、N2、N3 が保存済み | `findByIds([N1.id, N2.id, N3.id])` を呼ぶ | N2、N3、N1 の順（通し番号の降順）で返す |
| TC-ports-changeNotificationRepository-19 | W1 に、通し番号がどちらも 5 の N1 と N2 が保存済み | `findByIds([N1.id, N2.id])` を呼ぶ | 2 件を、通知の ID の昇順で返す |
| TC-ports-changeNotificationRepository-20 | W1 に N1 が保存済み | `findByIds([N1.id, N1.id, X])`（X は未使用の ID）を呼ぶ | N1 の 1 件を返す。重複した ID を一つとして扱い、存在しない ID を結果に含めない |
| TC-ports-changeNotificationRepository-22 | W1 に N1、W2 に P が保存済み | W1 の区画のリポジトリで `findByIds([N1.id, P.id])` を呼ぶ | N1 の 1 件を返す |
| TC-ports-changeNotificationRepository-23 | W1 に通知が 1 件 | 空の列で `findByIds` を呼ぶ | 空の列を返す |
| TC-ports-changeNotificationRepository-24 | W1 に通知が 1 件 | 重複を除いて上限の件数の ID で `findByIds` を呼ぶ | 存在する通知だけを返す |
| TC-ports-changeNotificationRepository-25 | W1 に通知が 1 件 | 重複を除いて上限より 1 件多い ID で `findByIds` を呼ぶ | `SystemError`（理由 `INVALID_QUERY`）になる |

## findByOperationId

O、O2 は別の操作の ID。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-changeNotificationRepository-27 | W1 に、操作の ID が O の N が保存済み | `findByOperationId(O2)` を呼ぶ | `null` を返す |
| TC-ports-changeNotificationRepository-28 | W1 に、操作の ID が O の N と、操作の ID が O2 の P が保存済み | `findByOperationId(O)` と `findByOperationId(O2)` を呼ぶ | O は N、O2 は P を返す |
| TC-ports-changeNotificationRepository-31 | W1 に、操作の ID が O の N が保存済み | W2 の区画のリポジトリで `findByOperationId(O)` を呼ぶ | `null` を返す |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-changeNotificationRepository-35 | W1 に 3 件 | `deleteAll(3)` を呼ぶ | 3 を返す。3 件とも `findById` は `null`。もう一度呼ぶと 0 を返す |
| TC-ports-changeNotificationRepository-36 | W1 に 4 件 | `deleteAll(3)` を三度続けて、別の UnitOfWork で呼ぶ | 順に 3、1、0 を返す。1 回目の後、4 件の ID を渡した `findByIds` は 1 件を返す |
| TC-ports-changeNotificationRepository-37 | W1 に 2 件、W2 に 1 件 | W1 の UnitOfWork で `deleteAll(10)` を呼ぶ | 2 を返す。W2 の 1 件は、W2 の区画の `findById` が返す |
| TC-ports-changeNotificationRepository-38 | W1 に N が 1 件。`WebhookDeliveryRepository` に、N の送信が 2 件保存済み | `deleteAll(10)` を呼ぶ | 1 を返す。`WebhookDeliveryRepository.findByNotification(N.id)` は 2 件を返す。送信を削除しない |
| TC-ports-changeNotificationRepository-39 | W1 に 2 件 | `deleteAll(BoundedDeletion.maxLimit)` を呼ぶ | 2 を返す |
| TC-ports-changeNotificationRepository-40 | W1 に 2 件 | `deleteAll(0)` と、`BoundedDeletion.maxLimit` を超える `limit` の `deleteAll` を、別の UnitOfWork で呼ぶ | どちらも `SystemError`（理由 `INVALID_QUERY`）になる。2 件とも `findById` が返す |

## UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-changeNotificationRepository-42 | 通知が無い | 一つの UnitOfWork で、N の `insert` と、N の送信 2 件の `WebhookDeliveryRepository.insertMany` を呼び、確定する | `findById(N.id)` は N を返し、`WebhookDeliveryRepository.findByNotification(N.id)` は 2 件を返す |
| TC-ports-changeNotificationRepository-43 | 通知が無い | 一つの UnitOfWork で、同じ 2 つの書き込みを呼んだ後、`fn` が例外を投げる | `run` はその例外を投げる。`findById(N.id)` は `null`、`WebhookDeliveryRepository.findByNotification(N.id)` は空 |
| TC-ports-changeNotificationRepository-44 | W1 に、操作の ID が O の P が保存済み | 一つの UnitOfWork で、操作の ID が別の N の `insert` と、操作の ID が O の R の `insert` を呼ぶ | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。`findById(N.id)` と `findById(R.id)` は `null` |
| TC-ports-changeNotificationRepository-45 | W1 に 2 件 | 一つの UnitOfWork の中で、`deleteAll(10)` を呼んだ後、`fn` が例外を投げる | 2 件とも `findById` が返す |
| TC-ports-changeNotificationRepository-46 | W1 に通知が無い | 操作の ID が O の P の `insert` を確定した直後に、UnitOfWork の外で `findById(P.id)`、`findByOperationId(O)`、`findByIds([P.id])` を呼ぶ | `findById` と `findByOperationId` は P を返し、`findByIds` は P の 1 件を返す |
