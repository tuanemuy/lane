# Webhook のユースケース

表示アプリへの通知の宛先の管理、通知の作成、送信、再試行、通知の状況の確認、送信失敗の対処の記録を定める。型・振る舞い・ポートは [../domains/webhook.md](../domains/webhook.md) が定める。ほかのドメインのユースケースとの分担は、[../domains/index.md](../domains/index.md)「ユースケースの所属」に従う。

見出しの括弧の中は、テストケース（`spec/testcases/webhook/`）のファイル名と同じ識別子である。各ユースケースの振る舞いの結果（結果の種類ごとの条件、エラー、途中で止まったときに残る状態）は、そのファイルが定める。

## 共通

- 認可: [../domains/index.md](../domains/index.md)「認可」に従う。各ユースケースは、必要な権限と、受け付ける実行者の種類だけを書く
- 変更の無い変更、競合、イベントの消費者と定期ジョブ、利用できないワークスペースは、[../domains/index.md](../domains/index.md) の「変更の無い変更」「競合の扱い」「イベントの消費者と定期ジョブ」「利用できるワークスペース」に従う
- UnitOfWork: このファイルのどの UnitOfWork も、`WriteScope` は `{ kind: "workspace", workspaceId, condition: { kind: "active" } }`。各ユースケースの「使うリポジトリ」は、この区画のもの
- 書き込みの量: [../domains/common.md](../domains/common.md)「UnitOfWorkProvider」に従う。件数に比例して書くユースケース（通知の作成、再試行、対処の記録）は、「トランザクション境界」に回数を書く
- 一度に扱う組: 通知と宛先の組の列を受け取るユースケース（再試行、対処の記録）は、列を、組の `DeliveryBatch.key` をキーにして、`BatchTargets.createBy`（上限は `DeliveryBatch.maxTargets`。[../domains/common.md](../domains/common.md)「BatchTargets」）に通してから、`WebhookDeliveryRepository.findByRefs` で送信を読み、組ごとに、読めた送信（読めなければ `null`）と確認した回を `ConfirmedDelivery`（[../domains/webhook.md](../domains/webhook.md) の「DeliveryBatch」）にして渡す
- 操作履歴: 宛先の登録・変更・削除と、秘密情報の置き換えは、[../domains/audit.md](../domains/audit.md)「トランザクション境界」に従って追記する。`changes` の項目は、`url`（宛先の URL の `ValueChange`）と、`secret`（登録と秘密情報の置き換えで渡す `SecretChange`）
- 未対処の失敗は [../domains/common.md](../domains/common.md)「FailureHandling」、自動の再試行の回数と間隔は [../domains/webhook.md](../domains/webhook.md)「DeliveryPolicy」に従う

### 送信の試行

送信の要求の消費者と定期ジョブが、対象のワークスペースの区画で、一つの送信（`DeliveryRef`）について行う手続き。宛先へ送るのは、この手続きだけである。`DeliveryDispatch.plan` を呼ぶ主体と、UnitOfWork の分け方は、ここだけが定める。途中で止まったときに残る状態は、[../testcases/webhook/sendRequestedDelivery.md](../testcases/webhook/sendRequestedDelivery.md) と [../testcases/webhook/retryDueDeliveries.md](../testcases/webhook/retryDueDeliveries.md) が定める。

- **入力**: `WorkspaceId` と `DeliveryRef`
- **出力**: 無し
- 実行者は `system`

呼ぶ主体と対象。

| 主体 | 対象の送信 |
| --- | --- |
| `sendRequestedDelivery`（`webhook.delivery_requested` の消費者） | イベントの組の送信 |
| `retryDueDeliveries`（定期ジョブ） | `findDue` が返した組ごと（`WebhookDelivery.isOverdue` が真の送信） |

使用するドメインの振る舞い・ポート。

- 名簿の項目（`PartitionReader.installation()` の `WorkspaceDirectory.findById`）、`WorkspaceRepository.find`、Workspace の `WorkspaceAvailability.of`（[../domains/index.md](../domains/index.md) の「利用できるワークスペース」）
- `WebhookDeliveryRepository.findById`、`ChangeNotificationRepository.findById`、`WebhookEndpointsRepository.find`
- `Clock.now`（手続きの一回ごとに、送る前の読み取りの後に一度だけ読む。その値を、`plan` の `now`、`WebhookSender.send` の `at`、`WebhookDelivery.record` の `now` に渡す）
- `DeliveryDispatch.plan`
- `WebhookSender.send`（`plan` が `send` を返した場合に、`message` を一度渡す）
- `WebhookDelivery.record`（`plan` が返した `attempt` と、結果。`send` では `WebhookSender.send` の結果、`fail` では `{ status: "failed", failure }`）、`WebhookDeliveryRepository.save`

トランザクション境界。

- 送る前の読み取り（名簿の項目、`Workspace`、送信、通知、宛先）は、UnitOfWork の外で行う
- 送信（`WebhookSender.send`）は、UnitOfWork の外で行う
- 結果の記録は、一つの UnitOfWork。送信の読み直し（`findById`）、`record`、`save` を行う。書き込みは、送信ごとに `save` の一回
- 使うリポジトリ: `webhookDeliveryRepository`（書き込み）。読み取りは `changeNotificationRepository`、`webhookEndpointsRepository`、`workspaceRepository`、設置の区画の `WorkspaceDirectory`

## 通知の宛先を登録する（`registerWebhookEndpoint`）

通知の宛先と秘密情報を登録する。

- **シナリオ**: WS-09
- **認可**: アプリ連携の権限（`integration:manage`）。実行者は `member` だけ
- **操作履歴**: `webhook.settings_changed`

### 入出力

- **入力**: 宛先の URL、秘密情報
- **出力**: 登録した宛先

### 使用するドメインの振る舞い・ポート

- `WebhookEndpointInput.validate`
- `SecretCipher.protect`、`ProtectedWebhookSecret.wrap`
- `WebhookEndpointsRepository.find`
- `IdGenerator.generate`、`WebhookEndpoints.create`（集約が無いワークスペースの最初の登録）、`WebhookEndpoints.register`
- `WebhookEndpointsRepository.insert`（最初の登録）または `WebhookEndpointsRepository.save`
- 操作履歴の追記

### トランザクション境界

- **UnitOfWork の要否**: 必要
- UnitOfWork: 認可の読み取り、`WebhookEndpoints` の読み取りと `insert` または `save`、`AuditLogEntry` の追記
- 使うリポジトリ: `workspaceRepository`、`memberRepository`、`webhookEndpointsRepository`、`auditLogRepository`

## 通知の宛先の URL を変更する（`changeWebhookEndpointUrl`）

登録済みの宛先の URL を変える。

- **シナリオ**: WS-09
- **認可**: アプリ連携の権限。実行者は `member` だけ
- **操作履歴**: `webhook.settings_changed`

### 入出力

- **入力**: 宛先の ID、新しい URL
- **出力**: 変更した宛先

### 使用するドメインの振る舞い・ポート

- `WebhookEndpointsRepository.find`、`WebhookEndpoints.changeUrl`、`WebhookEndpointsRepository.save`
- 操作履歴の追記

### トランザクション境界

- **UnitOfWork の要否**: 必要
- UnitOfWork: 認可の読み取り、`WebhookEndpoints` の読み取りと `save`、`AuditLogEntry` の追記
- 使うリポジトリ: `workspaceRepository`、`memberRepository`、`webhookEndpointsRepository`、`auditLogRepository`

## 通知の宛先の秘密情報を置き換える（`replaceWebhookEndpointSecret`）

登録済みの宛先の秘密情報を、新しい値に置き換える。

- **シナリオ**: WS-09
- **認可**: アプリ連携の権限。実行者は `member` だけ
- **操作履歴**: `webhook.settings_changed`

### 入出力

- **入力**: 宛先の ID、新しい秘密情報
- **出力**: 無し

### 使用するドメインの振る舞い・ポート

- `WebhookSecret.create`（保護の前に呼ぶ）、`SecretCipher.protect`、`ProtectedWebhookSecret.wrap`
- `WebhookEndpointsRepository.find`、`WebhookEndpoints.replaceSecret`、`WebhookEndpointsRepository.save`
- 操作履歴の追記

### トランザクション境界

- **UnitOfWork の要否**: 必要
- UnitOfWork: 認可の読み取り、`WebhookEndpoints` の読み取りと `save`、`AuditLogEntry` の追記
- 使うリポジトリ: `workspaceRepository`、`memberRepository`、`webhookEndpointsRepository`、`auditLogRepository`

## 通知の宛先を削除する（`deleteWebhookEndpoint`）

登録済みの宛先を削除する。

- **シナリオ**: WS-09
- **認可**: アプリ連携の権限。実行者は `member` だけ
- **操作履歴**: `webhook.settings_changed`

### 入出力

- **入力**: 宛先の ID
- **出力**: 無し

### 使用するドメインの振る舞い・ポート

- `WebhookEndpointsRepository.find`、`WebhookEndpoints.remove`、`WebhookEndpointsRepository.save`
- 操作履歴の追記

### トランザクション境界

- **UnitOfWork の要否**: 必要
- UnitOfWork: 認可の読み取り、`WebhookEndpoints` の読み取りと `save`、`AuditLogEntry` の追記。通知と送信は書かない
- 使うリポジトリ: `workspaceRepository`、`memberRepository`、`webhookEndpointsRepository`、`auditLogRepository`

## 通知の宛先を一覧する（`listWebhookEndpoints`）

ワークスペースに登録済みの宛先を示す。

- **シナリオ**: WS-09、PUB-08（宛先が無いことの表示）
- **認可**: アプリ連携の権限。実行者は `member` だけ

### 入出力

- **入力**: 無し（`Actor` と `WorkspaceId` だけ）
- **出力**: 宛先の一覧

### 使用するドメインの振る舞い・ポート

- `WebhookEndpointsRepository.find`、`WebhookEndpoints.list`

### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みが無い

## 通知と宛先の組を一覧する（`listDeliveries`）

通知と宛先の組を、通知の新しい順に、未対処と対処済みで絞り込んで一覧する。

- **シナリオ**: PUB-08
- **認可**: アプリ連携の権限。実行者は `member` だけ

### 入出力

- **入力**: 絞り込み（未対処、対処済み、指定なし）、ページング
- **出力**: 組ごとに、次を持つ
  - 組（`DeliveryRef`）と、受け付けた手動再試行の回数（`retryRound`）。組は、再試行と対処の記録の入力になる。回数は、再試行と対処の記録の入力に、確認した回として添える
  - 通知の内容（通知の種別、確定した日時、変更した対象、スキーマの版、参照元）
  - 宛先
  - 送信結果

### 使用するドメインの振る舞い・ポート

- `Pagination.create`
- `WebhookDeliveryRepository.findPage`
- `ChangeNotificationRepository.findByIds`（そのページの送信が指す通知の ID）
- `ChangeNotification.kinds`
- `WebhookEndpointsRepository.find`、`WebhookEndpoints.find`（送信が指す宛先を引く）

### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みが無い

## 一つの通知の送信結果を確かめる（`getNotificationDeliveries`）

公開内容を変えた一つの操作の通知の、宛先ごとの送信結果を示す。操作を、操作の ID で指す。公開内容を変える操作を行ったメンバーが、その操作の結果として、通知の結果を確かめる。

- **シナリオ**: PUB-01（操作の結果に示す通知の結果）、PUB-08
- **認可**: 実行者は `member` だけ。アプリ連携の権限を持つメンバー、または、その通知の元になった操作を行ったメンバー（`ChangeNotification.wasCausedBy` が真）。対象の属性で可否が決まる読み取りで、判定の時点は [../domains/index.md](../domains/index.md)「認可」に従う

### 入出力

- **入力**: 操作の ID（`OperationId`）。公開内容を変える操作のユースケースの出力（[content.md](content.md)）が持つ値
- **出力**: 通知が未作成（`notCreated`）、または、宛先ごとの番号（`WebhookDelivery.endpointOrder`）と送信結果の列

### 使用するドメインの振る舞い・ポート

- `ChangeNotificationRepository.findByOperationId`
- `ChangeNotification.wasCausedBy`（実行者の `MemberId`）
- `WebhookDeliveryRepository.findByNotification`（その通知の ID）

### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みが無い

## 未対処の送信失敗の件数を求める（`countUnhandledDeliveryFailures`）

表示アプリへの通知に未対処の失敗があることと、その件数を示す。

- **シナリオ**: PUB-08
- **認可**: アプリ連携の権限。実行者は `member` だけ

### 入出力

- **入力**: 無し（`Actor` と `WorkspaceId` だけ）
- **出力**: 未対処の送信失敗の件数

### 使用するドメインの振る舞い・ポート

- `WebhookDeliveryRepository.countUnhandled`

### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みが無い

## 公開内容の変更の通知を作る（`createChangeNotification`）

`content.publication_changed` の消費者。公開内容を変える操作の確定の後に、その操作の公開の変更の記録から `ChangeNotification` と、宛先ごとの `WebhookDelivery` を作り、送信ごとに送信の要求（`webhook.delivery_requested`）を記録する。宛先へは送らない。

- **シナリオ**: APP-11、PUB-08、HOST-09
- **認可**: 実行者は `system` だけ
- **イベント**: `webhook.delivery_requested`（作った送信ごとに一件）

### 入出力

- **入力**: イベントの `workspaceId`（ペイロードと別の項目）、ペイロード（操作の ID）、イベントの発生日時。イベントの `workspaceId` の区画で処理する
- **出力**: 無し

### 使用するドメインの振る舞い・ポート

- `Clock.now`
- 名簿の項目（`PartitionReader.installation()` の `WorkspaceDirectory.findById`）、`WorkspaceRepository.find`、Workspace の `WorkspaceAvailability.of`（[../domains/index.md](../domains/index.md) の「利用できるワークスペース」）
- `ChangeNotificationRepository.findByOperationId`、`WebhookEndpointsRepository.find`、`ChangeNotification.decideCreation`
- `WebhookEndpointsRepository.save`（`decideCreation` が `create` を返した場合に、読んだ集約をそのまま、読んだ `expectedVersion` で渡す）
- Content の `PublicationLedgerRepository.findByOperationId`（`decideCreation` が `create` を返した場合に、操作の ID で公開の変更の記録を読む）
- `IdGenerator.generate`、`ChangeNotification.create`、`WebhookDelivery.beginAll`
- `ChangeNotificationRepository.insert`、`WebhookDeliveryRepository.insertMany`、`collectEvents`

`ChangeNotification.create` に渡す `PublicationInput` は、読んだ記録（Content の `PublicationRecord`）から次のとおり作る。写し方は、この表だけが定める。

| `PublicationInput` | 渡す値 |
| --- | --- |
| `operationId` | イベントのペイロードの値 |
| `sequence`、`actor`、`changeList` | 記録の `sequence`、`actor`、`changeList` |
| `source` | 記録の `operation` が `migrated`（スキーマの反映の確定）なら `{ kind: "schemaApply", schemaVersion }` で、`schemaVersion` は記録の `schemaVersion`。ほかの値なら `{ kind: "content" }` |
| `occurredAt` | イベントの発生日時 |

### トランザクション境界

- **UnitOfWork の要否**: 必要。一つの UnitOfWork
- 名簿の項目は、UnitOfWork の前に読む
- UnitOfWork: `Workspace`、通知（`findByOperationId`）、`WebhookEndpoints`、公開の変更の記録の読み取りと、`WebhookEndpoints` の `save`、`ChangeNotification` の `insert`、その送信の `insertMany`、送信ごとの `webhook.delivery_requested` の記録
- `WebhookEndpoints` の `save` は、宛先を変えず、読んだ `expectedVersion` で行う。読んだ宛先による `decideCreation` の判定を、確定の時点まで保つ（[../domains/common.md](../domains/common.md)「TransactionalRepository」の「読み取りと確定」）
- 書き込みは、`save`、`insert`、`insertMany` の三回に、送信の要求を宛先ごとに一件足した回数。多くて、3 に `DeliveryPolicy.maxEndpoints` を足した回数
- 使うリポジトリ: `workspaceRepository`、`changeNotificationRepository`、`webhookDeliveryRepository`、`webhookEndpointsRepository`、`publicationLedgerRepository`（読み取り）

## 送信を求められた通知を送る（`sendRequestedDelivery`）

`webhook.delivery_requested` の消費者。イベントが指す送信（通知と宛先の組）について、「送信の試行」を一回行う。

- **シナリオ**: APP-11、PUB-08、HOST-09
- **認可**: 実行者は `system` だけ

### 入出力

- **入力**: イベントの `workspaceId`（ペイロードと別の項目）と、ペイロード（通知の ID、宛先の ID）
- **出力**: 無し

### 使用するドメインの振る舞い・ポート

- 共通の「送信の試行」（イベントの通知の ID と宛先の ID の組）

### トランザクション境界

- **UnitOfWork の要否**: 必要。「送信の試行」の結果を記録する UnitOfWork。使うリポジトリと UnitOfWork の分け方は、「送信の試行」のとおり

## 表示アプリへの通知を再試行する（`retryDueDeliveries`）

定期ジョブ「表示アプリへの通知の再試行」。対象のワークスペースの、`WebhookDelivery.isOverdue` が真の送信待ちを送る。

- **シナリオ**: APP-11、PUB-08、HOST-09
- **認可**: 実行者は `system` だけ。起動の周期は [../domains/index.md](../domains/index.md) の「定期ジョブ」が定める

### 入出力

- **入力**: `WorkspaceId`
- **出力**: 無し

### 使用するドメインの振る舞い・ポート

- `Clock.now`
- `WebhookDeliveryRepository.findDue`（一回の起動で、起動の時点の `now` と、前の呼び出しの `next`（`DueCursor`）を渡し、`next` が `null` になるまで続きを読む）
- 共通の「送信の試行」（`findDue` が返した組ごと）

### トランザクション境界

- **UnitOfWork の要否**: 必要。`findDue` は UnitOfWork を開かずに読む。送信ごとの「送信の試行」の結果を記録する UnitOfWork を、それぞれ別に確定する
- 使うリポジトリ: 「送信の試行」のとおり

## 送信失敗の通知を再試行する（`requestDeliveryRetry`）

メンバーが、送信失敗の通知と宛先の組を一件以上選び、それぞれの宛先へ一回だけ送り直すことを、一度に要求する。受け付けるかどうかは、[../domains/webhook.md](../domains/webhook.md) の「DeliveryBatch」が決める。

- **シナリオ**: PUB-08
- **認可**: アプリ連携の権限。実行者は `member` だけ
- **イベント**: `webhook.delivery_requested`（再試行を始めた組ごとに一件）

### 入出力

- **入力**: 通知の ID と宛先の ID の組（`DeliveryRef`）と、確認した回（「通知と宛先の組を一覧する」が返した、その組の `retryRound`）の組の列。`BatchTargets` で作る（共通の「一度に扱う組」）
- **出力**: `BatchRetryDecision` に対応する結果。受け付けた（組ごとの `requested` または `alreadyRequested` と、その組の今の送信結果）、または受け付けなかった（受け付けない組と理由）

### 使用するドメインの振る舞い・ポート

- `BatchTargets.createBy`、`DeliveryBatch.key`
- `WebhookDeliveryRepository.findByRefs`
- `WebhookEndpointsRepository.find`
- `Clock.now`、`DeliveryBatch.requestRetries`
- `WebhookDeliveryRepository.save`（`requested` の送信ごとに一度）、`collectEvents`
- `WebhookEndpointsRepository.save`（`requested` の組がある場合に一度。読んだ集約をそのまま、読んだ `expectedVersion` で渡す）

### トランザクション境界

- **UnitOfWork の要否**: 必要
- UnitOfWork: 認可の読み取り、送信と宛先の読み取り、再試行を受け付けた送信の `save`、`webhook.delivery_requested` の記録、`WebhookEndpoints` の `save`。書き込みは、送信の `save` とイベントが、受け付けた組ごとに一回ずつと、`WebhookEndpoints` の `save` の一回で、多くて `DeliveryBatch.maxTargets` の 2 倍に 1 を足した回数
- `WebhookEndpoints` の `save` は、宛先を変えず、読んだ `expectedVersion` で行う。読んだ宛先による `endpointDeleted` の判定を、確定の時点まで保つ（[../domains/common.md](../domains/common.md)「TransactionalRepository」の「読み取りと確定」）
- 使うリポジトリ: `workspaceRepository`、`memberRepository`、`webhookDeliveryRepository`、`webhookEndpointsRepository`

## 送信失敗を対処済みとして記録する（`markDeliveryFailureHandled`）

再試行しない送信失敗の組を一件以上選び、一度に対処済みにする。受け付けるかどうかは、[../domains/webhook.md](../domains/webhook.md) の「DeliveryBatch」が決める。

- **シナリオ**: PUB-08
- **認可**: アプリ連携の権限。実行者は `member` だけ

### 入出力

- **入力**: 通知の ID と宛先の ID の組（`DeliveryRef`）と、確認した回（「通知と宛先の組を一覧する」が返した、その組の `retryRound`）の組の列。`BatchTargets` で作る（共通の「一度に扱う組」）
- **出力**: `BatchHandlingDecision` に対応する結果。受け付けた（指定したすべての組の現在の状態）、または受け付けなかった（受け付けない組と理由）

### 使用するドメインの振る舞い・ポート

- `BatchTargets.createBy`、`DeliveryBatch.key`
- `WebhookDeliveryRepository.findByRefs`
- `Clock.now`、`DeliveryBatch.markHandled`（実行者の `MemberId`）
- `WebhookDeliveryRepository.save`（`changed` の送信ごとに一度）

### トランザクション境界

- **UnitOfWork の要否**: 必要
- UnitOfWork: 認可の読み取り、送信の読み取り、変わった送信の `save`。書き込みは、`save` が、変わった組ごとに一回で、多くて `DeliveryBatch.maxTargets` 回
- 使うリポジトリ: `workspaceRepository`、`memberRepository`、`webhookDeliveryRepository`
