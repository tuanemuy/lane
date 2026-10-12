# WebhookEndpointsRepository

契約は [webhook.md](../../domains/webhook.md) の「ポート」と「トランザクション境界」、[common.md](../../domains/common.md) の `SingletonRepository`、すべてのリポジトリに共通の規則、`UnitOfWorkProvider` が定める。

## 前提

- 集約は `WebhookEndpoints.create` に続けて `register` を適用して作り、`register` / `changeUrl` / `replaceSecret` / `remove` で変える。`changeUrl` / `replaceSecret` / `remove` の「結果」は、`found` が真の結果が持つ集約を指す。集約は区画に一つで、メソッドはワークスペースの ID を取らない。集約は `workspaceId` を持たない。「W1 の集約」は、W1 の UnitOfWork で保存する集約を指す
- `insert` / `save` / `deleteAll` は `UnitOfWorkProvider.run` の中で呼ぶ。断りが無ければ、一つの操作を一つの UnitOfWork で確定し、読み取りは UnitOfWork の外で呼ぶ
- UnitOfWork の `WriteScope` は、どのケースも、対象のワークスペースの `{ kind: "workspace", workspaceId, condition: { kind: "none" } }`。区画と条件は、`unitOfWorkProvider.md` が確かめる
- リポジトリは、ワークスペースの区画に結び付く。A1 の書き込みは W1 の UnitOfWork で、読み取りは `PartitionReader.workspace(W1)` のリポジトリで行う。A2 と W2 も同じ。断りが無ければ、`find()` と `deleteAll` は W1 の区画のリポジトリで呼ぶ
- 「保存済み」は、`insert` を確定した状態を指す。`v` は、直前の `find` が返した `expectedVersion` を指す
- 「並行して」は、二つの UnitOfWork を同時に始め、どちらも確定まで進めることを指す
- 競合と違反は、メソッドの呼び出しか確定のどちらかで現れる。`run` が `ConflictError` を投げることと、その UnitOfWork の書き込みが一つも反映されないことで確かめる
- W1、W2 は別の、確定済みのワークスペース。A1、A2 は、それぞれのワークスペースの集約。断りが無ければ、集約は宛先を 1 件持つ
- 通知の `insert` と読み取りは、同じ区画の `ChangeNotificationRepository` で行う。N は W1 の通知。一つの UnitOfWork の二つ目の書き込みには、N の `insert` を使う

## insert

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookEndpointsRepository-01 | 集約が無い | 宛先 1 件の A1 を `insert` する | `find()` が、宛先の `id`、`url`、`secret`、`createdAt`、`updatedAt` を含めて A1 と等しいエンティティを返す |
| TC-ports-webhookEndpointsRepository-03 | 集約が無い | 宛先が `DeliveryPolicy.maxEndpoints` 件の A1 を `insert` する | `find()` の `endpoints` は、A1 と同じ順の、その件数 |
| TC-ports-webhookEndpointsRepository-04 | 集約が無い | URL が同じ宛先 2 件の A1 を `insert` する | `find()` の `endpoints` は、ID が別で URL が同じ 2 件 |
| TC-ports-webhookEndpointsRepository-05 | A1 が保存済み | W1 の集約をもう一つ `insert` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。`find()` は A1 を返す |
| TC-ports-webhookEndpointsRepository-06 | A1 が保存済み | W2 の UnitOfWork で A2 を `insert` する | 確定する。W1 の区画の `find()` は A1 を、W2 の区画の `find()` は A2 と `expectedVersion` を返す |
| TC-ports-webhookEndpointsRepository-07 | 集約が無い | 宛先が別の、W1 の二つの集約を、並行して `insert` する | 一方だけが確定し、他方の `run` は `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。`find()` は確定した方を返す |
| TC-ports-webhookEndpointsRepository-08 | A1 を保存し、`deleteAll(BoundedDeletion.maxLimit)` で削除してある | W1 の集約を `insert` する | 確定する。`find()` はその集約を返す |

## find

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookEndpointsRepository-09 | 集約が無い | `find()` を呼ぶ | `null` を返す |
| TC-ports-webhookEndpointsRepository-14 | 区画が無いワークスペース（存在しない ID） | その区画のリポジトリで `find()` を呼ぶ | `null` を返す |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookEndpointsRepository-15 | 宛先 E1 を持つ A1 が保存済み。`find` で A1 と `v` を得ている | 宛先 E2 の `register` の結果を `save` する | `find()` の `endpoints` は E1、E2 の順。`version` は A1 より 1 大きい |
| TC-ports-webhookEndpointsRepository-16 | 宛先 E1 と E2 を持つ A1 が保存済み。`find` で A1 と `v` を得ている | E1 の `changeUrl` の結果を `save` する | `find()` の E1 の `url` は変更後の URL。E1 の `secret` と、E2 は変わらない。順は E1、E2 のまま |
| TC-ports-webhookEndpointsRepository-17 | 宛先 E1 と E2 を持つ A1 が保存済み。`find` で A1 と `v` を得ている | E1 の `replaceSecret` の結果を `save` する | `find()` の E1 の `secret` は置き換え後の値。E1 の `url` と、E2 は変わらない |
| TC-ports-webhookEndpointsRepository-18 | 宛先 E1、E2、E3 を持つ A1 が保存済み。`find` で A1 と `v` を得ている | E2 の `remove` の結果を `save` する | `find()` の `endpoints` は E1、E3 の順 |
| TC-ports-webhookEndpointsRepository-19 | 宛先 E1 だけを持つ A1 が保存済み。`find` で A1 と `v` を得ている | E1 の `remove` の結果を `save` する | `find()` は、`endpoints` が空の集約を返す |
| TC-ports-webhookEndpointsRepository-21 | A1 と A2 が保存済み | A1 の `register` の結果を `save` する | W2 の区画の `find()` は、変わっていない A2 を返す |
| TC-ports-webhookEndpointsRepository-22 | A1 が保存済み。`save` を一度確定してある | 確定の後に `find` で得た `v` で、もう一度 `save` する | 確定する |
| TC-ports-webhookEndpointsRepository-23 | 宛先 E1 を持つ A1 が保存済み。`find` で A1 と `v` を得ている | 読んだ A1 をそのまま、`v` で `save` する | 確定する。`find()` は A1 と等しい集約と、`v` と等しい `expectedVersion` を返す |
| TC-ports-webhookEndpointsRepository-24 | 宛先 E1 と E2 を持つ A1 が保存済み。`find` で A1 と `v` を得た後、別の UnitOfWork が E2 の `remove` の結果を `save` して確定した | 読んだ A1 をそのまま、`v` で `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`find()` の `endpoints` は E1 だけ |
| TC-ports-webhookEndpointsRepository-26 | A1 が保存済み。二つの呼び出し側が、同じ `v` を得ている | 宛先の `register` の `save` と、E1 の `changeUrl` の `save` を、並行して呼ぶ | 一方だけが確定し、他方の `run` は `ConflictError`（理由 `STALE_VERSION`）を投げる。`find()` は確定した方の内容を返す |
| TC-ports-webhookEndpointsRepository-28 | A1 が保存済み。`find` で A1 と `v` を得た後、別の UnitOfWork が `deleteAll(BoundedDeletion.maxLimit)` を確定した | `v` で `register` の結果を `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`find()` は `null` |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookEndpointsRepository-29 | 集約が無い | `deleteAll(10)` を呼ぶ | 0 を返す |
| TC-ports-webhookEndpointsRepository-30 | 宛先が複数件の A1 が保存済み | `deleteAll(1)` を呼ぶ | 1 を返す。`find()` は `null` |
| TC-ports-webhookEndpointsRepository-32 | `endpoints` が空の A1 が保存済み | `deleteAll(10)` を呼ぶ | 1 を返す。`find()` は `null` |
| TC-ports-webhookEndpointsRepository-33 | A1 と A2 が保存済み | W1 の UnitOfWork で `deleteAll(10)` を呼ぶ | 1 を返す。W2 の区画の `find()` は A2 を返す |
| TC-ports-webhookEndpointsRepository-34 | A1 が保存済み。`find` で `v` を得た後、別の UnitOfWork が A1 の `save` を確定した | `deleteAll(10)` を呼ぶ | 1 を返す。`find()` は `null` |
| TC-ports-webhookEndpointsRepository-35 | A1 が保存済み。通知 N が `ChangeNotificationRepository` に、A1 の宛先を指す N の送信が `WebhookDeliveryRepository` に保存済み | `deleteAll(10)` を呼ぶ | 1 を返す。`ChangeNotificationRepository.findById(N.id)` は N を、`WebhookDeliveryRepository.findById` は、削除の前と等しい送信を返す |
| TC-ports-webhookEndpointsRepository-37 | A1 が保存済み | `deleteAll(BoundedDeletion.maxLimit)` を呼ぶ | 1 を返す。`find()` は `null` |
| TC-ports-webhookEndpointsRepository-38 | A1 が保存済み | `deleteAll(0)` と、`BoundedDeletion.maxLimit` を超える `limit` の `deleteAll` を、別の UnitOfWork で呼ぶ | どちらも `SystemError`（理由 `INVALID_QUERY`）になる。`find()` は A1 を返す |

## UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookEndpointsRepository-40 | A1 が保存済み | W1 の一つの UnitOfWork で、N の `insert` と、A1 の `register` の `save` を呼び、確定する | `ChangeNotificationRepository.findById(N.id)` は N、`find()` は宛先 2 件の A1 を返す |
| TC-ports-webhookEndpointsRepository-41 | A1 が保存済み | 一つの UnitOfWork で、同じ 2 つの書き込みを呼んだ後、`fn` が例外を投げる | `run` はその例外を投げる。`ChangeNotificationRepository.findById(N.id)` は `null`、`find()` は宛先 1 件の A1 を返す |
| TC-ports-webhookEndpointsRepository-42 | A1 が保存済み。`find` で `v` を得た後、別の UnitOfWork が A1 の `save` を確定した | W1 の一つの UnitOfWork で、N の `insert` と、`v` での A1 の `register` の `save` を呼ぶ | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`ChangeNotificationRepository.findById(N.id)` は `null` |
| TC-ports-webhookEndpointsRepository-44 | A1 が保存済み | 一つの UnitOfWork の中で、`deleteAll(10)` を呼んだ後、`fn` が例外を投げる | `find()` は A1 を返す |
| TC-ports-webhookEndpointsRepository-45 | 宛先 E1 を持つ A1 が保存済み | 一つの UnitOfWork で、通知 N の `ChangeNotificationRepository.insert` と、`find()` で読んだ宛先の ID を `endpointIds` にした `WebhookDelivery.beginAll` の結果の `WebhookDeliveryRepository.insertMany` を呼び、確定する | `WebhookDeliveryRepository.findByNotification(N.id)` は、E1 を指す送信の 1 件を返す |
