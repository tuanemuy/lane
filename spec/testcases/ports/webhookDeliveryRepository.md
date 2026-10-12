# WebhookDeliveryRepository

「上限」は、その節のメソッドの契約が定める、一度に渡せる件数・個数、または `limit` の上限を指す。値は、ポートを定めるドメインのファイルが持つ。

契約は [webhook.md](../../domains/webhook.md) の「ポート」と「トランザクション境界」、[common.md](../../domains/common.md) の `TransactionalRepository`（`insert` と `delete` を除く）、すべてのリポジトリに共通の規則、`UnitOfWorkProvider` が定める。

## 前提

- 送信は `WebhookDelivery.beginAll`（通知、宛先の ID の列、作成の日時）が返す送信で作り、`record` / `requestRetry` / `markHandled` で変える。`requestRetry` と `markHandled` の「結果」は、受け付けた結果が持つ送信を指す
- `beginAll` に渡す日時を c とする。日時 t に `undelivered`（`timeout`）の失敗を `record` した最初の試行の、次の試行の時刻（`nextAttemptAt` と `overdueAt`）を「t の最初の間隔の後」と書く。送信の時刻の値は、[webhook.md](../../domains/webhook.md) の「WebhookDelivery」（`pending` と「試行の結果の記録」）と `DeliveryPolicy` が定める。ケースは、時刻の値を書かずに、この定数から求める
- `record` には、その送信が待っている試行（`dueAttempt` が返す、送信の `attempt` と、`round` として送信の `retryRound`）を渡す。`requestRetry` と `markHandled` には、確認した回として、その送信の今の `retryRound` を渡す
- 「送信成功」は、`{ status: "delivered" }` を `record` した送信を指す。「未対処の失敗」は、`notSent`（`endpoint_deleted`）の失敗を `record` して `failed` にした送信を指す。「対処済みの失敗」は、それに `markHandled` を適用した送信を指す
- `insertMany` / `save` / `deleteAll` は `UnitOfWorkProvider.run` の中で呼ぶ。断りが無ければ、一つの操作を一つの UnitOfWork で確定し、読み取りは UnitOfWork の外で呼ぶ
- UnitOfWork の `WriteScope` は、どのケースも、対象のワークスペースの `{ kind: "workspace", workspaceId, condition: { kind: "none" } }`。区画と条件は、`unitOfWorkProvider.md` が確かめる
- リポジトリは、ワークスペースの区画に結び付く。断りが無ければ、書き込みは W1 の UnitOfWork で、読み取りは `PartitionReader.workspace(W1)` のリポジトリで行う。メソッドは、ワークスペースの ID を取らない。送信は `workspaceId` を持たない
- 「保存済み」は、`insertMany` を確定した状態を指す。`v` は、直前の読み取りが返した `expectedVersion` を指す
- 「並行して」は、二つの UnitOfWork を同時に始め、どちらも確定まで進めることを指す
- 競合と違反は、メソッドの呼び出しか確定のどちらかで現れる。`run` が `ConflictError` を投げることと、その UnitOfWork の書き込みが一つも反映されないことで確かめる
- ID の順は、文字列のコードポイントの昇順
- W1、W2 は別の、確定済みのワークスペース。W0 は、区画が無いワークスペース（存在しない ID）。E1、E2、E3 は宛先の ID で、この順が ID の昇順。ポートは通知と宛先の存在を担保しないので、通知の保存と宛先の登録を前提にしない
- t1 < t2 < t3 は固定の日時。N、P、Q は通知。「N と E1」は、N の ID と E1 の組の送信を指す。断りが無ければ、送信は W1 のもので、通知ごとに E1 の一つ

## insertMany

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookDeliveryRepository-01 | 送信が無い | N の、E1〜E3 の送信を `insertMany` する | 3 つの組の `findById` が、`notificationId`、`endpointId`、`endpointOrder`（1、2、3）、`sequence`、`state`（`pending`、`attempt` が `first`、`nextAttemptAt`、`overdueAt`）、`retryRound` 0、`version` 0、`createdAt` を含めて、渡した送信と等しいエンティティを返す |
| TC-ports-webhookDeliveryRepository-02 | N と E1 が保存済み | N の、E1 と E2 の送信を `insertMany` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。N と E2 の `findById` は `null` |
| TC-ports-webhookDeliveryRepository-03 | 送信が無い | N と E1 の送信を二つ含む列を `insertMany` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。N と E1 の `findById` は `null` |
| TC-ports-webhookDeliveryRepository-04 | N と E1 が保存済み | P と E1 の送信を `insertMany` する | 確定する。同じ宛先の、別の通知の送信を保存できる |
| TC-ports-webhookDeliveryRepository-05 | W1 に N と E1 が保存済み | W2 の UnitOfWork で、N と同じ通知の ID と E1 の送信を `insertMany` する | 確定する。W2 の区画の `findById` が返す |
| TC-ports-webhookDeliveryRepository-06 | 送信が無い | 同じ組の送信の `insertMany` を、並行して呼ぶ | 一方だけが確定し、他方の `run` は `ConflictError`（理由 `ALREADY_EXISTS`）を投げる |
| TC-ports-webhookDeliveryRepository-07 | 送信が無い | 空の列を `insertMany` する | 何も書かない。`findPage({}, { page: 1, limit: 10 })` の `count` は 0 |
| TC-ports-webhookDeliveryRepository-08 | 送信が無い | 一つの UnitOfWork で、N の、`DeliveryPolicy.maxEndpoints` 件の宛先の送信の `insertMany` を一回と、互いに違う通知の `ChangeNotificationRepository.insert` を、合わせて `UnitOfWorkCapacity.workspaceWrites` 回になるまで呼ぶ | `run` が確定する。`findByNotification(N.id)` は、渡したすべての送信を返す。`insertMany` の書き込みの回数は、送信の件数によらず一回 |
| TC-ports-webhookDeliveryRepository-09 | 送信が無い | `ChangeNotificationRepository` に無い通知の ID と、`WebhookEndpointsRepository` に無い宛先の ID の送信を `insertMany` する | 確定する。通知と宛先の存在を確かめない |

## findById

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookDeliveryRepository-10 | N と E1 が保存済み | 未使用の組と、N と E2 の組で、`findById` を呼ぶ | どちらも `null` を返す |
| TC-ports-webhookDeliveryRepository-12 | W2 に N と E1 が保存済み | W1 の区画のリポジトリと、W2 の区画のリポジトリで、その組の `findById` を呼ぶ | W1 の区画は `null` を、W2 の区画は送信と `expectedVersion` を返す |
| TC-ports-webhookDeliveryRepository-13 | `WebhookEndpointsRepository` に保存した集約が持つ宛先 E1 を指す、N と E1 が保存済み。その後、E1 を `WebhookEndpoints.remove` で除いた集約の `save` を確定した | その組の `findById` を呼ぶ | 送信を返す |
| TC-ports-webhookDeliveryRepository-15 | W0 は区画が無い | W0 の区画のリポジトリで、`findById`、`findByRefs`、`findByNotification`、`findPage({}, { page: 1, limit: 10 })`、`countUnhandled()`、`countPending()`、`findDue(t2, { after: null, limit: 10 })` を呼ぶ | `findById` は `null`。`findByRefs` と `findByNotification` は空の列。`findPage` は、`items` が空で `count` が 0。`countUnhandled` と `countPending` は 0。`findDue` は、`refs` が空で `next` が `null` |

## save

D は、断りが無ければ、t1 に作成した N と E1 の送信。N と E2 の送信も保存済み。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookDeliveryRepository-16 | `findById` で D と `v` を得ている | 日時 t2 に `delivered` を `record` した結果を `save` する | `findById` が、`succeeded`（`succeededAt` は t2）で、`version` が D より 1 大きい送信を返す。N と E2 の送信は `pending` のまま、`version` も変わらない |
| TC-ports-webhookDeliveryRepository-17 | `findById` で D と `v` を得ている | 日時 t2 に `undelivered`（`rejected`、`statusCode` 500）の失敗を `record` した結果を `save` する | `findById` の送信は、`pending`、`attempt` が `automatic` の 2 回目、`previous` が t2 と `undelivered`（`rejected`、500）、`nextAttemptAt` と `overdueAt` が t2 の最初の間隔の後 |
| TC-ports-webhookDeliveryRepository-18 | `findById` で D と `v` を、N と E2 の送信の `v` を得ている | 日時 t2 に、D には `notSent`（`endpoint_deleted`）、N と E2 の送信には `notSent`（`secret_unavailable`）の失敗を `record` した結果を、それぞれ `save` する | どちらの `findById` の送信も、`failed`、`failedAt` が t2、`failure` が `record` した失敗、`handling` が `unhandled` |
| TC-ports-webhookDeliveryRepository-20 | D は未対処の失敗。`findById` で D と `v` を得ている | `markHandled(D, 0, by, t3)` の結果を `save` する | `findById` の送信の `handling` は、`handled`、`by` が渡したメンバー、`at` が t3 |
| TC-ports-webhookDeliveryRepository-21 | D は未対処の失敗（`failedAt` は t2）。`findById` で D と `v` を得ている | `requestRetry(D, 0, t3)` の結果を `save` する | `findById` の送信は、`pending`、`attempt` が `manual`、`previous` が t2 と `notSent`（`endpoint_deleted`）、`nextAttemptAt` が t3、`overdueAt` が t3 に `DeliveryPolicy.requestGraceMs` を足した時刻、`retryRound` が 1 |
| TC-ports-webhookDeliveryRepository-23 | `save` を一度確定してある | 確定の後に `findById` で得た `v` で、もう一度 `save` する | 確定する |
| TC-ports-webhookDeliveryRepository-24 | `findById` で D と `v` を得た後、別の UnitOfWork が D の `record` の結果を `save` して確定した | `v` で、D の別の `record` の結果を `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById` は、先に確定した内容を返す |
| TC-ports-webhookDeliveryRepository-25 | 二つの呼び出し側が、D の同じ `v` を得ている | `delivered` の `record` の `save` と、失敗の `record` の `save` を、並行して呼ぶ | 一方だけが確定し、他方の `run` は `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById` は確定した方の内容を返す |
| TC-ports-webhookDeliveryRepository-27 | N と E1、N と E2 の送信の `v` を、それぞれ得ている | N と E1 の `record` の `save` と、N と E2 の `record` の `save` を、並行して呼ぶ | どちらも確定する。同じ通知の別の宛先の送信は競合しない |
| TC-ports-webhookDeliveryRepository-28 | `findById` で D と `v` を得た後、別の UnitOfWork が `deleteAll` で D を削除して確定した | `v` で `record` の結果を `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById` は `null` |

## findByRefs

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookDeliveryRepository-29 | N の E1〜E3 と、P の E1 の送信が保存済み。N の ID は P の ID より小さい | P と E1、N と E3、N と E1 の組で `findByRefs` を呼ぶ | N と E1、N と E3、P と E1 の順（通知の ID、宛先の ID の昇順）で、それぞれ `expectedVersion` とともに返す |
| TC-ports-webhookDeliveryRepository-30 | N と E1 が保存済み | N と E1 の組を二つと、N と E2 の組を並べて `findByRefs` を呼ぶ | N と E1 の 1 件を返す。重複した組を一つとして扱い、存在しない組を結果に含めない |
| TC-ports-webhookDeliveryRepository-32 | W1 に N と E1、W2 に P と E1 が保存済み | W1 の区画のリポジトリで、二つの組の `findByRefs` を呼ぶ | N と E1 の 1 件を返す |
| TC-ports-webhookDeliveryRepository-33 | N と E1 が保存済み | `findByRefs` が返した `expectedVersion` で、`record` の結果を `save` する | 確定する |
| TC-ports-webhookDeliveryRepository-34 | N と E1 が保存済み | 空の列で `findByRefs` を呼ぶ | 空の列を返す |
| TC-ports-webhookDeliveryRepository-35 | N と E1 が保存済み | 重複を除いて `DeliveryBatch.maxTargets` 件の組で `findByRefs` を呼ぶ | 存在する送信だけを返す |
| TC-ports-webhookDeliveryRepository-36 | N と E1 が保存済み | 重複を除いて `DeliveryBatch.maxTargets` 件を超える組で `findByRefs` を呼ぶ | `SystemError`（理由 `INVALID_QUERY`）になる |

## findByNotification

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookDeliveryRepository-38 | N の E1〜E3 と、P の E1 の送信が保存済み | `findByNotification(N.id)` を呼ぶ | N と E1、N と E2、N と E3 の順（宛先の ID の昇順）で返す。P の送信を返さない。`version` を返さない |
| TC-ports-webhookDeliveryRepository-40 | N と E1 が保存済み | 送信の無い通知の ID で `findByNotification` を呼ぶ | 空の列を返す |
| TC-ports-webhookDeliveryRepository-41 | W1 に N と E1、W2 に、同じ通知の ID の送信が保存済み | W1 の区画のリポジトリで `findByNotification(N.id)` を呼ぶ | W1 の 1 件だけを返す |

## findPage

「5 組の状態」は、W1 に次の送信がある状態を指す。N1（通し番号 1）は、E1 が `pending`、E2 が送信成功、E3 が未対処の失敗。N2（通し番号 2）は、E1 が対処済みの失敗、E2 が未対処の失敗。`pagination` は、断りが無ければ `{ page: 1, limit: 10 }`。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookDeliveryRepository-43 | W1 に送信が無い | `findPage({}, pagination)` を呼ぶ | `items` は空、`count` は 0 |
| TC-ports-webhookDeliveryRepository-45 | 5 組の状態 | `filter` を `{}` にして呼ぶ | `items` は、N2 と E1、N2 と E2、N1 と E1、N1 と E2、N1 と E3 の順の 5 件。`count` は 5 |
| TC-ports-webhookDeliveryRepository-46 | W1 に、通し番号がどちらも 5 の通知 N と P の、それぞれ E1 と E2 の送信がある。N の ID は P の ID より小さい | `filter` を `{}` にして呼ぶ | `items` は、N と E1、N と E2、P と E1、P と E2 の順の 4 件（同じ通し番号の中は通知の ID の昇順、同じ通知の中は宛先の ID の昇順） |
| TC-ports-webhookDeliveryRepository-47 | 上と同じ | `{ page: 1, limit: 2 }` と `{ page: 2, limit: 2 }` で呼ぶ | 1 ページ目は N と E1、N と E2。2 ページ目は P と E1、P と E2。どの組も、二つのページのどちらか一方にだけ現れる |
| TC-ports-webhookDeliveryRepository-48 | 5 組の状態 | `handling: "unhandled"` で呼ぶ | `items` は、N2 と E2、N1 と E3 の順の 2 件。`count` は 2 |
| TC-ports-webhookDeliveryRepository-49 | 5 組の状態 | `handling: "handled"` で呼ぶ | `items` は、N2 と E1 の 1 件。`count` は 1 |
| TC-ports-webhookDeliveryRepository-51 | 5 組の状態。W2 に、未対処の失敗が 1 件 | `findPage({ handling: "unhandled" }, pagination)` を呼ぶ | `items` は W1 の 2 件だけ。`count` は 2 |
| TC-ports-webhookDeliveryRepository-52 | 5 組の状態 | `{ page: 1, limit: 4 }` と `{ page: 2, limit: 4 }` で呼ぶ | 1 ページ目の `items` は、N2 と E1、N2 と E2、N1 と E1、N1 と E2 の順の 4 件。2 ページ目の `items` は、N1 と E3 の 1 件。どちらも `count` は 5 |
| TC-ports-webhookDeliveryRepository-54 | 5 組の状態 | `{ page: 2, limit: 5 }` で呼ぶ | `items` は空、`count` は 5 |
| TC-ports-webhookDeliveryRepository-55 | 5 組の状態 | `handling: "unhandled"`、`{ page: 2, limit: 1 }` で呼ぶ | `items` は、N1 と E3 の 1 件。`count` は 2 |
| TC-ports-webhookDeliveryRepository-57 | 5 組の状態。N1 と E3 に `requestRetry` を適用して `save` を確定した | `handling: "unhandled"` と `handling: "handled"` でそれぞれ呼ぶ | `unhandled` は N2 と E2 の 1 件。`handled` は N2 と E1 の 1 件 |

## countUnhandled

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookDeliveryRepository-59 | W1 に、`pending`、送信成功、対処済みの失敗の送信だけがある | `countUnhandled()` を呼ぶ | 0 を返す |
| TC-ports-webhookDeliveryRepository-62 | 「findPage」の 5 組の状態 | `countUnhandled()` を呼ぶ | 2 を返す。`findPage({ handling: "unhandled" }, pagination)` の `count` と一致する |
| TC-ports-webhookDeliveryRepository-63 | W1 に未対処の失敗が 1 つ、W2 に未対処の失敗が 3 つ | `countUnhandled()` を呼ぶ | 1 を返す |
| TC-ports-webhookDeliveryRepository-65 | W1 に未対処の失敗が 1 つ。それに `requestRetry` を適用して `save` を確定した | `countUnhandled()` を呼ぶ | 0 を返す |

## countPending

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookDeliveryRepository-67 | W1 に、送信成功と `failed` の送信だけがある | `countPending()` を呼ぶ | 0 を返す |
| TC-ports-webhookDeliveryRepository-68 | W1 に、作成したままの、N の E1〜E3 の送信がある | `countPending()` を呼ぶ | 3 を返す |
| TC-ports-webhookDeliveryRepository-71 | W1 に、`undelivered`（`timeout`）の失敗を一度 `record` した `pending` の送信と、`requestRetry` で `pending` に戻した送信が 1 つずつ | `countPending()` を呼ぶ | 2 を返す |
| TC-ports-webhookDeliveryRepository-73 | W1 に `pending` が 1 つ、W2 に `pending` が 3 つ | `countPending()` を呼ぶ | 1 を返す |

## findDue: 対象

`page` は `{ after: null, limit: 10 }` で呼ぶ。g は `DeliveryPolicy.requestGraceMs`。「t+g」は、日時 t に g を足した日時を指す。対象は、`WebhookDelivery.isOverdue(delivery, now)` が真の送信。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookDeliveryRepository-75 | 送信成功の送信と、`failed` の送信だけがある | `findDue(t3+g, page)` を呼ぶ | `refs` は空、`next` は `null` |
| TC-ports-webhookDeliveryRepository-76 | t1 に作成した N と E1（最初の試行を待つ） | t1+g より 1 ミリ秒前を `now` にして `findDue` を呼ぶ | `refs` は空、`next` は `null`。送信の要求の消費者に任せる |
| TC-ports-webhookDeliveryRepository-77 | 上と同じ | `findDue(t1+g, page)` を呼ぶ | `refs` は N と E1 の 1 件。`next` は `null` |
| TC-ports-webhookDeliveryRepository-80 | t1 に作成した N と E1。日時 t2 に `undelivered`（`timeout`）の失敗を `record` して `save` を確定した | t2 の最初の間隔の後より 1 ミリ秒前を `now` にして `findDue` を呼ぶ | `refs` は空 |
| TC-ports-webhookDeliveryRepository-81 | 上と同じ | t2 の最初の間隔の後を `now` にして `findDue` を呼ぶ | `refs` は N と E1 の 1 件。自動の再試行は、g を待たない |
| TC-ports-webhookDeliveryRepository-84 | t1 に作成した、W1 の N と E1、W2 の P と E1 | W1 の区画のリポジトリで `findDue(t1+g, page)` を呼ぶ | `refs` は N と E1 の 1 件だけ。ほかのワークスペースの送信を返さない |
| TC-ports-webhookDeliveryRepository-85 | t1 に作成した W1 の N と E1。`WorkspaceRepository` で W1 の区画を `closing` にして確定した | `findDue(t1+g, page)` を呼ぶ | `refs` は N と E1 の 1 件。ワークスペースの状態を確かめない |

## findDue: 並び順とページング

「位置」は、`overdueAt`、通知の ID、宛先の ID の組（`DueCursor`）を指す。g と「t+g」は、前の節と同じ。「対象の送信」は、`now` の時点で `WebhookDelivery.isOverdue` が真の送信を指す。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookDeliveryRepository-87 | t3、t1、t2 に作成した送信を、この順に保存してある | `findDue(t3+g, { after: null, limit: 10 })` を呼ぶ | `refs` は、作成が t1、t2、t3 の送信の順 |
| TC-ports-webhookDeliveryRepository-88 | t1 に作成した、E1 と E2 の送信を持つ通知が 2 件 | `findDue(t1+g, { after: null, limit: 10 })` を呼ぶ | `refs` は 4 件。通知の ID の昇順で、同じ通知の中は E1、E2 の順 |
| TC-ports-webhookDeliveryRepository-89 | 対象の送信が、上限の件数だけある | 上限の `limit` で、`after` を `null` にして呼ぶ | `refs` は上限の件数、`next` は `null` |
| TC-ports-webhookDeliveryRepository-90 | `overdueAt` が別の、対象の送信が上限より 1 件多い | 上限の `limit` で、`after` を `null` にして呼ぶ | `refs` は `overdueAt` の早い方から上限の件数。`next` は、返した最後の送信の位置 |
| TC-ports-webhookDeliveryRepository-91 | 上と同じ | 上の `next` を `after` にして、上限の `limit` で呼ぶ | `refs` は最後の 1 件だけ、`next` は `null` |
| TC-ports-webhookDeliveryRepository-92 | 上と同じ | 最後の送信の位置を `after` にして呼ぶ | `refs` は空、`next` は `null` |
| TC-ports-webhookDeliveryRepository-93 | t1 に作成した、N の E1〜E3 | `findDue(t1+g, { after: null, limit: 2 })` を呼ぶ | `refs` は、N と E1、N と E2 の 2 件。`next` は、t1+g、N の ID、E2 |
| TC-ports-webhookDeliveryRepository-94 | 上と同じ | 上の `next` を `after` にして、`limit: 2` で呼ぶ | `refs` は N と E3 の 1 件、`next` は `null` |
| TC-ports-webhookDeliveryRepository-95 | t1 に作成した N と E1、t2 に作成した P と E1。`findDue(t3+g, { after: null, limit: 1 })` が N と E1 と `next` を返した。N と E1 に、日時 t3+g の `undelivered`（`timeout`）の失敗を `record` して `save` を確定した | その `next` を `after` にして、`findDue(t3+g, { after, limit: 1 })` を呼ぶ | `refs` は P と E1 だけ、`next` は `null` |
| TC-ports-webhookDeliveryRepository-96 | 上の前提で、N と E1 の代わりに、P と E1 に `delivered` を `record` して `save` を確定した | その `next` を `after` にして、`findDue(t3+g, { after, limit: 1 })` を呼ぶ | `refs` は空、`next` は `null` |
| TC-ports-webhookDeliveryRepository-100 | 対象の送信が 1 件 | `limit: 0` と、上限より 1 大きい `limit` で呼ぶ | どちらも `SystemError`（理由 `INVALID_QUERY`）になる |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookDeliveryRepository-103 | W1 に、`pending`、送信成功、未対処の失敗、対処済みの失敗の送信が 1 件ずつ | `deleteAll(10)` を呼ぶ | 4 を返す。4 件とも `findById` は `null`。`countPending()` と `countUnhandled()` は 0 |
| TC-ports-webhookDeliveryRepository-104 | W1 に 4 件 | `deleteAll(3)` を三度続けて、別の UnitOfWork で呼ぶ | 順に 3、1、0 を返す。1 回目の後、`findPage` の `count` は 1 |
| TC-ports-webhookDeliveryRepository-105 | W1 に 2 件、W2 に 1 件 | W1 の UnitOfWork で `deleteAll(10)` を呼ぶ | 2 を返す。W2 の 1 件は、W2 の区画の `findById` が返す |
| TC-ports-webhookDeliveryRepository-106 | W1 に 2 件 | `deleteAll(BoundedDeletion.maxLimit)` を呼ぶ | 2 を返す |
| TC-ports-webhookDeliveryRepository-107 | W1 に 2 件 | `deleteAll(0)` と、`BoundedDeletion.maxLimit` を超える `limit` の `deleteAll` を、別の UnitOfWork で呼ぶ | どちらも `SystemError`（理由 `INVALID_QUERY`）になる。`findPage` の `count` は 2 |

## UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookDeliveryRepository-109 | P と E1 が保存済み | 一つの UnitOfWork で、N と E1 の `insertMany` と、P と E1 の `record` の `save` を呼んだ後、`fn` が例外を投げる | `run` はその例外を投げる。N と E1 の `findById` は `null`。P と E1 は `pending` のまま |
| TC-ports-webhookDeliveryRepository-110 | P と E1、Q と E1 が保存済み。それぞれの `v` を得た後、別の UnitOfWork が Q と E1 の `record` の `save` を確定した | 一つの UnitOfWork で、P と E1 の `record` の `save` と、古い `v` での Q と E1 の `save` を呼ぶ | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。P と E1 は `pending` のまま |
| TC-ports-webhookDeliveryRepository-111 | W1 に 2 件 | 一つの UnitOfWork の中で、`deleteAll(10)` を呼んだ後、`fn` が例外を投げる | `findPage` の `count` は 2 |
| TC-ports-webhookDeliveryRepository-112 | W1 に送信が無い | t1 に作成した N と E1 の `insertMany` を確定した直後に、UnitOfWork の外で `findById`、`findPage({}, pagination)`、`countPending()`、`findDue(t1+g, page)` を呼ぶ（g は `DeliveryPolicy.requestGraceMs`） | `findById` は送信を返し、`findPage` の `count` は 1、`countPending` は 1、`findDue` の `refs` は N と E1 の 1 件 |
