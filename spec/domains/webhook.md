# Webhook

公開内容の変更を表示アプリへ伝え、その送信結果を保つ。境界と依存方向は [index.md](index.md) が定める。Webhook は他のドメインに依存しない。コンテンツとメンバーは、共通の ID（`ContentId`、`MemberId`）で参照する。モデルと項目のキーは、文字列として保持する。ID（`OperationId` を含む）、`Actor`、`FailureHandling`、`Pagination`、`BatchTargets`、`PublishedChangeNotice` と `PublishedChangeList`、`TransactionalRepository`、`SingletonRepository`、`WriteScope`、`UnitOfWorkProvider`、`WithEventDrafts`、`SecretCipher` と `ProtectedSecret`、`StatusItem` と `DomainKind` は [common.md](common.md) の定義を使う。

Webhook のドメインは Content の型を使わず、公開内容の変更の値を共通の `PublishedChangeList` で受ける。

Webhook の集約とポートは、すべてワークスペースの区画にある（[index.md](index.md) の「区画」）。

フォームの通知（Form）と別である。宛先、秘密情報、送信結果、自動の再試行を共有しない。

## ユビキタス言語

| 英語名 | 日本語名 | 定義 |
| --- | --- | --- |
| Webhook endpoints | 通知の宛先の列 | ワークスペースに一つの、表示アプリへの通知の宛先の集まり |
| Webhook endpoint | 通知の宛先 | 表示アプリが通知を受け取る URL と、その秘密情報 |
| Secret | 秘密情報 | CMS と表示アプリが共有する値。通知が CMS から届いたことを、表示アプリが署名で確かめる |
| Change notification | 変更の通知 | 一つの操作で確定した公開内容の変更をまとめた記録。作成の後に変わらない |
| Source | 通知の元 | 通知を作った操作の区分。コンテンツの操作（`content`）、またはスキーマの反映（`schemaApply`） |
| Change | 変更 | 一件のコンテンツの公開内容の変化。共通の `PublishedChangeNotice` |
| Change kind | 変更の種類 | 一件の変更の種類。共通の `PublishedChangeNotice.change` |
| Notification kind | 通知の種別 | 利用者に示す、通知の区分。公開、更新、公開取り消し、スキーマの反映のいずれか。`ChangeNotification.kinds` が返す |
| Sequence | 通し番号 | 公開内容を変える操作の確定の時点で、Content の公開の変更の記録に割り当てられる番号。変更の順序を判別する情報になる |
| Operation ID | 操作の ID | 公開内容を変える操作の一回を指す ID（`OperationId`）。割り当ては [index.md](index.md) の「公開内容の整合性と通し番号」が定める |
| Change list | 変更の一覧 | 一つの操作の、変更の一覧と参照元の一覧。共通の `PublishedChangeList` |
| Unique value change | 一意な値の変更 | 変更した対象の、公開内容の一意な値の、変更前と変更後。共通の `UniqueValueChange` |
| Referrer | 参照元 | 変更した対象を、公開内容から参照しているコンテンツ。コンテンツID とモデルのキーの組（共通の `PublishedReferrer`）で持つ |
| Payload | 通知の内容 | 宛先へ送る情報 |
| Delivery | 送信 | 通知と宛先の組ごとの、宛先への送信。送信結果を持つ。通知と別の集約（`WebhookDelivery`）で、組ごとに `version` を持つ。Webhook の型の名前の `Delivery…`（`DeliveryPolicy`、`DeliveryDispatch`、`DeliveryBatch`、`DeliveryRef`、`DeliveryAttempt`）は、この送信を指す。Content の取得（`ContentDelivery`）、Media の配信（`MediaDelivery`）、Form の宛先ごとの送信の結果（`RecipientDelivery`）と別 |
| Delivery status | 送信結果 | 一つの送信が、送信待ち、送信成功、送信失敗のどれであるか（`WebhookDelivery` の `state.status`） |
| Attempt | 送信の試行 | 一つの送信で、宛先へ一回送ること |
| Delivery request | 送信の要求 | 一つの送信の、時刻を過ぎた試行を行うことを求めるドメインイベント（`webhook.delivery_requested`）。送信を作ったときと、手動再試行を受け付けたときに記録する |
| Automatic retry | 自動の再試行 | 失敗した試行の後に、CMS が間隔を空けて行う試行 |
| Manual retry | 手動再試行 | 送信失敗の送信を、メンバーの操作で一回だけ送り直すこと。失敗しても自動の再試行を行わない |
| Failure handling | 対処の状態 | 送信失敗が未対処か対処済みか。共通の `FailureHandling` |

## エンティティ

### WebhookEndpoints

ワークスペースに一つの集約。区画が識別する。

フィールド。

```ts
type WebhookEndpoint = Readonly<{
  id: WebhookEndpointId;
  url: WebhookUrl;
  secret: ProtectedWebhookSecret;
  createdAt: Date;
  updatedAt: Date;
}>;

type WebhookEndpoints = Readonly<{
  endpoints: readonly WebhookEndpoint[];
  version: number;
  createdAt: Date;
  updatedAt: Date;
}>;
```

| フィールド | 制約 |
| --- | --- |
| `endpoints` | 宛先の列。件数は「不変条件」が定める。登録の順に並ぶ。同じ `id` は一度だけ現れる。同じ URL の宛先を複数持てる |
| `endpoints[].url` | 通知を送る URL |
| `endpoints[].secret` | `SecretCipher` で保護した形。保護していない値は保存しない |
| `version` | 生成時は 0。状態を変える振る舞いごとに 1 増える |

振る舞い。

```ts
type EndpointUpdate =
  | Readonly<{ found: true; endpoints: WebhookEndpoints }>
  | Readonly<{ found: false }>;

const WebhookEndpoints: {
  create(now: Date): WebhookEndpoints;
  register(
    endpoints: WebhookEndpoints,
    params: { id: string; url: string; secret: ProtectedWebhookSecret },
    now: Date,
  ): WebhookEndpoints;
  changeUrl(
    endpoints: WebhookEndpoints | null,
    input: { endpointId: WebhookEndpointId; url: string },
    now: Date,
  ): EndpointUpdate;
  replaceSecret(
    endpoints: WebhookEndpoints | null,
    input: { endpointId: WebhookEndpointId; secret: ProtectedWebhookSecret },
    now: Date,
  ): EndpointUpdate;
  remove(
    endpoints: WebhookEndpoints | null,
    endpointId: WebhookEndpointId,
    now: Date,
  ): EndpointUpdate;
  find(endpoints: WebhookEndpoints | null, endpointId: WebhookEndpointId): WebhookEndpoint | null;
  list(endpoints: WebhookEndpoints | null): readonly WebhookEndpoint[];
};
```

| 振る舞い | 処理 |
| --- | --- |
| `create` | 宛先を持たない集約を返す |
| `register` | 宛先が既に `DeliveryPolicy.maxEndpoints` 件あれば `BusinessRuleError`（`EndpointLimitReached`）で、上限を添える。`WebhookEndpointId` と `WebhookUrl` を生成し、宛先を列の末尾に加える |
| `changeUrl` | その宛先の `WebhookUrl` を置き換えた集約を、`found` が真の結果で返す。同じ URL なら、渡された集約をそのまま持つ |
| `replaceSecret` | その宛先の秘密情報を置き換えた集約を、`found` が真の結果で返す。置き換え前の値は残さない |
| `remove` | その宛先を列から除いた集約を、`found` が真の結果で返す |
| `find` | その ID の宛先を返す。集約が `null`、または宛先が無ければ `null` |
| `list` | 宛先の列を返す。集約が `null` なら空の列 |

- `changeUrl`、`replaceSecret`、`remove` は、集約が `null`、またはその ID の宛先が無ければ、`found` が偽の結果を返し、`WebhookUrl` を作らない

不変条件。

- 一つのワークスペースが持つ宛先は、`DeliveryPolicy.maxEndpoints` 件以下である
- 秘密情報は、登録の後に値を示さない。登録済みであることだけを示す
- 宛先の登録と削除は、作成済みの通知の送信を増やさず、減らさない。どの宛先の送信を作るかは「通知を作る条件」が、送る時点の宛先の URL と秘密情報の使い方と、削除された宛先の送信の扱いは「DeliveryDispatch」が定める

ライフサイクル。

- 生成: そのワークスペースで最初の宛先を登録するとき、または初期構成で宛先を宣言したとき。`create` に続けて `register` を適用した結果を `insert` する。宛先を登録したことのないワークスペースは、集約を持たない
- 変更: 宛先の登録、URL の変更、秘密情報の置き換え、宛先の削除。すべての宛先を削除した集約は、空の列で残る
- 削除: ワークスペースの利用終了の消去だけが、集約を削除する

### ChangeNotification

一つの操作で確定した公開内容の変更をまとめた記録。宛先ごとの送信結果は、別の集約の `WebhookDelivery` が持つ。

フィールド。

```ts
type NotificationSource =
  | Readonly<{ kind: "content" }>
  | Readonly<{ kind: "schemaApply"; schemaVersion: number }>;

type ChangeNotification = Readonly<{
  id: ChangeNotificationId;
  sequence: number;
  operationId: OperationId;
  source: NotificationSource;
  actor: Actor;
  changeList: PublishedChangeList;
  occurredAt: Date;
  createdAt: Date;
}>;
```

| フィールド | 制約 |
| --- | --- |
| `id` | 通知の ID。通知の内容の `notificationId` になり、通知を識別する情報（APP-11）に当たる。同じ通知のすべての宛先とすべての試行で同じ |
| `sequence` | 元になった操作の通し番号。公開の変更の記録の通し番号を引き継ぐ。1 以上の整数。変更の順序を判別する情報（APP-11）に当たる |
| `operationId` | 元になった操作の ID。公開の変更の記録の操作の ID を引き継ぐ。同じワークスペースの中で一意 |
| `actor` | 元になった操作の実行者 |
| `source` | 通知の元。`schemaApply` は、スキーマの反映の確定（移行の適用と直接反映を含む）による変更で、反映後のスキーマの版（1 以上の整数）を持つ。`content` は、それ以外の公開内容を変える操作による変更で、スキーマの版を持たない |
| `changeList` | 一つの操作の、変更の一覧と参照元の一覧。公開の変更の記録の値を引き継ぐ。型と不変条件は [common.md](common.md) の「PublishedChangeNotice」が定める |
| `occurredAt` | 元になった操作を確定した日時。イベントの発生日時。通知の一覧に示す値 |
| `createdAt` | 通知を作った日時 |

通知は、楽観ロックの `version` を持たない。

振る舞い。

```ts
type PublicationInput = Readonly<{
  sequence: number;
  operationId: OperationId;
  source: NotificationSource;
  actor: Actor;
  changeList: PublishedChangeList;
  occurredAt: Date;
}>;

type CreationDecision =
  | Readonly<{ action: "skip" }>
  | Readonly<{ action: "create"; endpointIds: readonly WebhookEndpointId[] }>;

type NotificationKind = "publish" | "update" | "unpublish" | "schemaApply";

const ChangeNotification: {
  decideCreation(
    input: Readonly<{ existing: ChangeNotification | null; endpoints: WebhookEndpoints | null }>,
  ): CreationDecision;
  create(params: { id: string; publication: PublicationInput }, now: Date): ChangeNotification;
  payload(notification: ChangeNotification): NotificationPayload;
  kinds(notification: ChangeNotification): readonly NotificationKind[];
  wasCausedBy(notification: ChangeNotification, memberId: MemberId): boolean;
};
```

`PublicationInput` は、一つの操作の公開の変更の記録から、ユースケースが作って渡す値。Webhook のドメインが受け取るのは、この値だけで、Content の記録の型（`PublicationRecord`）ではない。記録からの写し方は、[../usecases/webhook.md](../usecases/webhook.md) の `createChangeNotification` だけが定める。

- 渡される値は、Content の不変条件（[common.md](common.md) の「PublishedChangeNotice」と、Content の公開の変更の記録）を満たす。Webhook は、その値を信頼する

| 振る舞い | 処理 |
| --- | --- |
| `decideCreation` | 通知を作るか、作らないかを決める。「通知を作る条件」が定める |
| `create` | `publication` の値を引き継いだ通知を返す。`createdAt` は `now`。ドメインイベントを返さない |
| `payload` | 「NotificationPayload」の値を返す |
| `kinds` | 利用者に示す、通知の種別を返す。`source` が `schemaApply` なら `schemaApply` だけを返す。`content` なら、`changeList.changes` に現れる変更の種類（`published` は `publish`、`updated` は `update`、`unpublished` は `unpublish`）を、重複なく返す。並びは、`changes` に現れた順によらず、`publish`、`update`、`unpublish` の順 |
| `wasCausedBy` | 通知の `actor` が `member` で、その `memberId` が引数の `memberId` と同じなら真。ほかの種類の実行者の操作による通知には偽 |

通知を作る条件（`decideCreation`）。上の行から順に当てはめる。

| 条件 | 結果 |
| --- | --- |
| `existing` が `null` でない（その操作の ID の通知が既にある） | `skip`。作らない。同じイベントの再配送に当たる |
| `WebhookEndpoints.list(endpoints)` が空（宛先が一つも無い） | `skip`。作らない。送信も送信失敗も生じない |
| 上のどれでもない | `create`。`endpointIds` は、その時点の宛先の ID を、登録の順に並べた列 |

- `existing` は、その操作の ID の通知（`ChangeNotificationRepository.findByOperationId` の結果）。再配送は、操作の ID で見分ける
- 通知を作るかどうかと、通知の宛先は、その操作の ID の通知が無い消費ごとに、その消費の時点の宛先で決まる。公開内容の変更の確定の後、消費の前に登録された宛先を含み、その間に削除された宛先を含まない。同じイベントの再配送も一つの消費なので、宛先が無くて `skip` で終えた操作は、後の消費の時点に宛先があれば通知になる。消費の時点の宛先を通知の確定まで保つ方法は、[../usecases/webhook.md](../usecases/webhook.md) の `createChangeNotification` の「トランザクション境界」が定める
- `create` の `endpointIds` は、`WebhookDelivery.beginAll` に渡す列

不変条件。

- 通知は、公開内容の変更が確定した操作についてだけ存在する
- 一つの操作で確定した複数のコンテンツの変更は、変更した対象と参照元をすべて持つ一つの通知になる。ワークスペースと操作の ID の組に、通知は最大一つである
- 通知は、作成の後に変わらない
- 通知は、作成の時点の宛先ごとに、送信（`WebhookDelivery`）を一つ持つ。一件以上。通知は、そのすべての送信とともに作られ、一方だけを作ることはない。利用終了の消去の途中を除き、一方だけが残ることはない

ライフサイクル。

- 生成: `content.publication_changed` の消費者が、宛先のあるワークスペースで作る
- 変更: 無い
- 削除: ワークスペースの利用終了の消去だけが、そのワークスペースの通知をすべて削除する。通知は、対象のコンテンツの完全削除と宛先の削除の後も残る

### WebhookDelivery

通知と宛先の組ごとの送信。送信結果を持つ。通知の ID と宛先の ID の組（`DeliveryRef`）で識別する。

フィールド。

```ts
type DeliveryRef = Readonly<{ notificationId: ChangeNotificationId; endpointId: WebhookEndpointId }>;

type PendingAttempt =
  | Readonly<{ kind: "first" }>
  | Readonly<{ kind: "automatic"; number: number; previous: Readonly<{ at: Date; failure: Undelivered }> }>
  | Readonly<{ kind: "manual"; previous: Readonly<{ at: Date; failure: AttemptFailure }> }>;

type DeliveryAttempt = PendingAttempt & Readonly<{ round: number }>;

declare const pendingBrand: unique symbol;
type Pending = Readonly<{ status: "pending"; attempt: PendingAttempt; nextAttemptAt: Date; overdueAt: Date }> & {
  readonly [pendingBrand]: true;
};
type Succeeded = Readonly<{ status: "succeeded"; succeededAt: Date }>;
type Failed = Readonly<{
  status: "failed";
  failedAt: Date;
  failure: AttemptFailure;
  handling: FailureHandling;
}>;
type DeliveryState = Pending | Succeeded | Failed;

type WebhookDelivery<S extends DeliveryState = DeliveryState> = Readonly<{
  notificationId: ChangeNotificationId;
  endpointId: WebhookEndpointId;
  endpointOrder: number;
  sequence: number;
  state: S;
  retryRound: number;
  version: number;
  createdAt: Date;
}>;
```

| フィールド | 制約 |
| --- | --- |
| `notificationId`、`endpointId` | 集約の ID。通知と、通知を作る時点に登録されていた宛先の組。生成後に変わらない |
| `endpointOrder` | 通知を作る時点の宛先の列の中の、その宛先の位置。登録の順で、1 始まり。同じ通知の送信の間で重ならない。操作を行ったメンバーに、宛先を区別して示す番号になる（PUB-01）。生成後に変わらない |
| `sequence` | 通知の通し番号の写し。組の一覧の並び順に使う。生成後に変わらない |
| `Pending.attempt` | 次に行う試行。`first` は、最初の試行で、直前の試行を持たない。`automatic` は、自動の再試行で、`number` は何回目の試行かを表す 2 以上 `DeliveryPolicy.maxAutomaticAttempts` 以下の整数。`manual` は、一回だけの手動再試行で、何回目の手動再試行かは送信の `retryRound` が持つ。`automatic` と `manual` は、直前の試行（`previous`。試行の日時と失敗）を必ず持つ。`automatic` の直前の失敗は、送って届かなかった失敗（`Undelivered`）だけ。`manual` の直前の失敗は、送信失敗になったときの失敗（`AttemptFailure`） |
| `Pending.nextAttemptAt` | 次の試行を行える時刻 |
| `Pending.overdueAt` | 定期ジョブ「表示アプリへの通知の再試行」が、この送信を送る時刻（`isOverdue`）。`attempt` と `nextAttemptAt` から `WebhookDelivery.pending` が求める |
| `Failed.handling` | 対処の状態。規則は [common.md](common.md) の「FailureHandling」が定める |
| `retryRound` | 受け付けた手動再試行の回数。生成時は 0。`requestRetry` が再試行を受け付けるごとに 1 増える。「確認した回」の判定と、試行の同一性に使う |
| `version` | 生成時は 0。状態を変える振る舞いごとに 1 増える |
| `createdAt` | 送信を作った日時。通知の `createdAt` と同じ |

送信結果。

| `status` | 利用者に示す状態 | 意味 |
| --- | --- | --- |
| `pending` | 送信待ち | 次の試行を待つ。最初の試行の前、自動の再試行の間、手動再試行の結果を待つ間 |
| `succeeded` | 送信成功 | 宛先が通知を受け取った。表示アプリ上の表示反映は示さない |
| `failed` | 送信失敗 | 宛先が通知を受け取らないまま、試行を終えた。送らなかった失敗（`notSent`）を含む。`failed` になる条件は「試行の結果の記録」が定める |

振る舞い。

```ts
type DeliveryOutcome =
  | Readonly<{ status: "delivered" }>
  | Readonly<{ status: "failed"; failure: AttemptFailure }>;

type RetryRequest =
  | (Readonly<{ requested: true }> & WithEventDrafts<WebhookDelivery<Pending>, WebhookEvent>)
  | Readonly<{ requested: false; reason: "alreadyRequested" | "notFailed" }>;

type HandlingRecord =
  | Readonly<{ handled: true; delivery: WebhookDelivery<Failed> }>
  | Readonly<{ handled: false; reason: "notFailed" }>;

const WebhookDelivery: {
  beginAll(
    notification: ChangeNotification,
    endpointIds: readonly WebhookEndpointId[],
    now: Date,
  ): WithEventDrafts<readonly WebhookDelivery<Pending>[], WebhookEvent>;
  pending(attempt: PendingAttempt, nextAttemptAt: Date): Pending;
  ref(delivery: WebhookDelivery): DeliveryRef;
  isDue(delivery: WebhookDelivery, now: Date): boolean;
  isOverdue(delivery: WebhookDelivery, now: Date): boolean;
  dueAttempt(delivery: WebhookDelivery, now: Date): DeliveryAttempt | null;
  record(
    delivery: WebhookDelivery,
    attempt: DeliveryAttempt,
    outcome: DeliveryOutcome,
    now: Date,
  ): WebhookDelivery;
  requestRetry(delivery: WebhookDelivery, confirmedRound: number, now: Date): RetryRequest;
  markHandled(delivery: WebhookDelivery, confirmedRound: number, by: MemberId, now: Date): HandlingRecord;
};
```

| 振る舞い | 処理 |
| --- | --- |
| `beginAll` | `endpointIds` の宛先ごとに、`pending`（`attempt` は `first`、`nextAttemptAt` は `now`）の送信を一つと、その送信の `webhook.delivery_requested` を一件返す。送信もイベントも、`endpointIds` の順に並ぶ。`endpointOrder` は、`endpointIds` の中の位置（1 始まり）。`sequence` は通知の通し番号、`retryRound` は 0、`createdAt` は `now`。`endpointIds` は、`decideCreation` が `create` で返した列 |
| `pending` | 送信待ちの状態を作る。`Pending` を作るのは、この振る舞いだけ。`overdueAt` は、`attempt` が `automatic` なら `nextAttemptAt`、`first` または `manual` なら `nextAttemptAt` に `DeliveryPolicy.requestGraceMs` を足した時刻。送信を `pending` にする振る舞い（`beginAll`、`record`、`requestRetry`）は、これで状態を作る |
| `ref` | 送信の ID（通知の ID と宛先の ID の組）を返す |
| `isDue` | `pending` で、`nextAttemptAt` が `now` 以前（同じ日時を含む）なら真 |
| `isOverdue` | 定期ジョブ「表示アプリへの通知の再試行」が送る送信かどうかを返す。`pending` で、`overdueAt` が `now` 以前（同じ日時を含む）なら真 |
| `dueAttempt` | `isDue` が真なら、送信の `attempt`（これから行う試行）に、`round` として送信の `retryRound` を添えて返す。偽なら `null` |
| `record` | 一回の試行（`attempt`）の結果を記録する。「試行の結果の記録」が定める |
| `requestRetry` | 「手動再試行」の表で、再試行を受け付けるかどうかを決める。受け付けた送信を `pending` にし、その送信の `webhook.delivery_requested` を一件返す。受け付けない送信は変えず、理由（`alreadyRequested`、`notFailed`）を返す |
| `markHandled` | 確認の後に再試行が始まった送信（「確認した回」）と、`failed` でない送信は、変えず、`notFailed` を返す。ほかの送信は、`handling` を `FailureHandling.markHandled(handling, by, now)` の結果にする |

- `markHandled` の、既に対処済みの送信への適用は、変更の無い変更（[index.md](index.md) の「変更の無い変更」）

試行の結果の記録（`record`）。上の行から順に当てはめる。

| 条件 | 結果 |
| --- | --- |
| 送信が `pending` でない | 変えず、渡された送信をそのまま返す。重複した試行の結果は、確定した送信成功と送信失敗を変えない |
| 送信の `attempt` が、渡された `attempt` と同じ試行でない | 変えず、渡された送信をそのまま返す。その試行の結果は、既に記録されている。重複した試行の失敗は、試行の番号を二つ進めない |
| `outcome` が `delivered` | `succeeded` にし、`succeededAt` を `now` にする |
| `outcome` の `failure` が `undelivered` で、`attempt` が `first`、または `number` が `DeliveryPolicy.maxAutomaticAttempts` より小さい `automatic` | `pending` のまま、`attempt` を次の `automatic` にする。`number` は、失敗した試行の番号（`first` は 1）に 1 を足した値。`previous` は、日時が `now`、失敗が `failure`。`nextAttemptAt` は、`now` に `DeliveryPolicy.retryDelaysMs[n - 1]`（`n` は失敗した試行の番号）を足した時刻 |
| 上のどれでもない（`failure` が `notSent`、最後の自動の試行の失敗、`manual` の試行の失敗） | `failed` にする。`failedAt` を `now`、`failure` を `failure`、`handling` を `FailureHandling.unhandled()` にする |

- `attempt` は、その結果を得た試行。`DeliveryDispatch.plan` が、送る前の送信から返した値
- 送信の `attempt` と渡された `attempt` は、渡された `round` が送信の `retryRound` と等しく、種類（`kind`）が同じで、`automatic` では `number` も同じとき、同じ試行である。手動再試行を受け付けた送信では `retryRound` が進むので、受け付けの前の回の試行の結果は同じ試行に当たらない
- 自動の再試行を行うかどうかは、失敗の変種（`AttemptFailure.kind`）と試行の種類だけで決まる。失敗の理由（`reason`）で分岐しない

確認した回。規則は [common.md](common.md) の「FailureHandling」が定める。Webhook の送信の回は `retryRound` で、`confirmedRound` は、メンバーが再試行または対処の記録に選んだときに確認した、その送信の `retryRound`。`requestRetry` と `markHandled` は、この判定を最初に当てはめる。

手動再試行（`requestRetry`）。上の行から順に当てはめる。

| 条件 | 結果 |
| --- | --- |
| 確認の後に再試行が始まっている | `requested` が偽で、理由は `alreadyRequested`。送信を変えず、イベントを返さない |
| 送信が `failed` でない | `requested` が偽で、理由は `notFailed`。送信を変えず、イベントを返さない |
| 上のどれでもない | 送信を `pending` にし、`retryRound` を 1 増やして、その送信の `webhook.delivery_requested` を一件返す。`attempt` は `manual`（`previous` は、日時が `failedAt`、失敗が `failure`）、`nextAttemptAt` は `now` |

不変条件。

- `succeeded` の送信は、ほかの状態に変わらない
- `retryRound` は減らない
- 直前の試行を持たない送信待ちは、最初の試行を待つ送信だけである。自動の再試行と手動再試行を待つ送信は、直前の試行の日時と失敗を持つ
- 一つの送信への並行した書き込みの扱いは、`WebhookDeliveryRepository.save` の契約が定める

ライフサイクル。

- 生成: 通知の作成とともに、通知を作る時点の宛先ごとに作る
- 送信の遷移: `pending` →（`record`）→ `succeeded`、`pending`（次の試行）、または `failed`。`failed` →（`requestRetry`）→ `pending`
- 対処の遷移: `failed` の中で、未対処 →（`markHandled`）→ 対処済み
- 削除: ワークスペースの利用終了の消去だけが、そのワークスペースの送信をすべて削除する

#### ドメインイベント

```ts
type DeliveryRequestedEvent = DomainEventBase<
  "webhook.delivery_requested",
  Readonly<{
    notificationId: ChangeNotificationId;
    endpointId: WebhookEndpointId;
  }>
>;
type WebhookEvent = DeliveryRequestedEvent;
```

送信の要求。一つの送信について、時刻を過ぎた試行を行うことを求める。Webhook が発行するイベントは、これだけである。

- 発行するのは、送信を作る `beginAll` と、手動再試行を受け付けた `requestRetry`。どちらも、送信ごとに一件
- `aggregateId` は、発行元の集約である送信（`WebhookDelivery`）の ID。通知の ID と宛先の ID の組を、`DeliveryBatch.key` で一つの文字列にした値
- イベントは、通知の内容と、試行の種類を持たない

## 値オブジェクト

ブランド付きの型は、`create` だけが生成する。利用者の入力と初期構成の宣言から作る値（`WebhookUrl`、`WebhookSecret`）の違反は、項目と理由を持つ `ValidationError` を投げる。理由のコードは「エラーコード」が列挙する。

| 名前 | 型 | バリデーションルール | 等価性 |
| --- | --- | --- | --- |
| `WebhookUrl` | `string`（ブランド） | 絶対 URL。スキームは `http` または `https`。認証情報とフラグメントを持たない。2,048 文字以下。違反は `InvalidWebhookUrl` | 値の一致 |
| `WebhookSecret` | `string`（ブランド） | 1 文字以上 256 文字以下。空白だけの値は不可。違反は `InvalidWebhookSecret` | 比較しない |
| `ProtectedWebhookSecret` | `ProtectedSecret`（ブランド） | `SecretCipher.protect` が返した秘密情報。`ProtectedWebhookSecret.wrap(secret: ProtectedSecret)` で包む | 値の一致 |
| `WebhookEndpointInput` | `Readonly<{ url: string; secret: string }>` | 下の定義 | — |
| `AttemptFailure` | 直和型 | 下の定義 | — |
| `DeliveryPolicy` | 定数 | 下の定義 | — |
| `NotificationPayload` | オブジェクト | `ChangeNotification.payload` が生成する | — |

秘密情報は、公開データ、通常のログ、操作履歴、通知の内容に含めない。

`WebhookUrl` と `WebhookSecret` の長さの上限は、技術設計の層の値で、利用者に値を示さない。上限を超える入力は、ほかの規則の違反と同じ理由（`InvalidWebhookUrl`、`InvalidWebhookSecret`）で示す。

### WebhookEndpointInput

```ts
type WebhookEndpointInput = Readonly<{ url: string; secret: string }>;

type EndpointInputIssue = Readonly<{
  field: "url" | "secret";
  code: "InvalidWebhookUrl" | "InvalidWebhookSecret";
}>;

type EndpointListIssue =
  | Readonly<{ index: number; field: "url" | "secret"; code: EndpointInputIssue["code"] }>
  | Readonly<{ index: null; field: null; code: "TooManyEndpoints" }>;

const WebhookEndpointInput: {
  validate(input: WebhookEndpointInput): readonly EndpointInputIssue[];
  validateAll(inputs: readonly WebhookEndpointInput[]): readonly EndpointListIssue[];
};
```

- 管理画面での登録と、初期構成の宣言が使う入力
- `validate` は、`WebhookUrl` と `WebhookSecret` の規則に合わない項目を、問題の一覧で返す。空の一覧は、問題が無いことを表す。例外を投げない。秘密情報の保護の前に呼べ、何も保存しない
- `validateAll` は、各入力の問題に、列の中の位置（0 始まり）を添えて返す。件数が `DeliveryPolicy.maxEndpoints` を超えれば、`TooManyEndpoints` を加える

### AttemptFailure

```ts
type Undelivered =
  | Readonly<{ kind: "undelivered"; reason: "rejected"; statusCode: number }>
  | Readonly<{ kind: "undelivered"; reason: "timeout" }>
  | Readonly<{ kind: "undelivered"; reason: "unreachable" }>;

type NotSent =
  | Readonly<{ kind: "notSent"; reason: "secret_unavailable" }>
  | Readonly<{ kind: "notSent"; reason: "endpoint_deleted" }>;

type AttemptFailure = Undelivered | NotSent;
```

| `kind` | 意味 | 自動の再試行 |
| --- | --- | --- |
| `undelivered` | 送って、宛先が通知を受け取らなかった | 自動の試行が残れば行う |
| `notSent` | 何も送らなかった | 行わない。送信失敗にする |

| `reason` | `kind` | 意味 |
| --- | --- | --- |
| `rejected` | `undelivered` | 宛先が、受け取りを示さない応答を返した。応答の状態を `statusCode` に持つ |
| `timeout` | `undelivered` | 宛先が、`DeliveryPolicy.timeoutMs` の間に応答を返さなかった |
| `unreachable` | `undelivered` | 宛先に接続できなかった |
| `secret_unavailable` | `notSent` | 宛先の秘密情報の保護を解けず、署名できなかった |
| `endpoint_deleted` | `notSent` | 送る時点で、宛先が削除されていた |

失敗の理由は、秘密情報、通知の内容、宛先の応答の本文を含まない。

### DeliveryPolicy

```ts
const DeliveryPolicy: {
  maxEndpoints: 10;
  maxAutomaticAttempts: 5;
  retryDelaysMs: readonly [60_000, 300_000, 1_800_000, 7_200_000];
  timeoutMs: 10_000;
  requestGraceMs: 300_000;
};
```

| 定数 | 意味 |
| --- | --- |
| `maxEndpoints` | 一つのワークスペースが登録できる宛先の数 |
| `maxAutomaticAttempts` | 自動で行う試行の回数。最初の試行を含む |
| `retryDelaysMs` | 失敗した試行から次の自動の試行までの間隔。1 回目の失敗の後から順に使う。要素の数は、`maxAutomaticAttempts` から 1 を引いた数 |
| `timeoutMs` | 一回の試行で、宛先の応答の状態を待つ時間 |
| `requestGraceMs` | 最初の試行と手動再試行を待つ送信について、送信の要求（`webhook.delivery_requested`）が届かなかったと見なすまでの時間 |

`maxEndpoints`、`maxAutomaticAttempts`、`retryDelaysMs`、`timeoutMs` の値は、APP-12（`spec/scenario/app-integration.md`）の表示アプリへの通知の行を指す。`requestGraceMs` は技術設計の層の値で、利用者に示さない。技術設計の層では、この定数だけが値を持ち、ほかの文書（ユースケース、動的フロー、テストケース）は、値を書かずに、この定数を名前で参照する。自動の再試行が終わる条件は「試行の結果の記録」が定める。

### NotificationPayload

```ts
type NotificationPayload = Readonly<{
  notificationId: ChangeNotificationId;
  sequence: number;
  schemaVersion: number | null;
  changes: readonly PublishedChangeNotice[];
  referrers: readonly PublishedReferrer[];
}>;
```

| 項目 | 内容 | 表示アプリの使い方 |
| --- | --- | --- |
| `notificationId` | 通知の ID。通知を識別する情報 | 同じ通知の重複を見分ける。同じ通知は、何度届いても同じ値を持つ |
| `sequence` | 通し番号。変更の順序を判別する情報 | 変更の順序を、対象ごとに判別する |
| `schemaVersion` | 通知の `source` が `schemaApply` なら、その `schemaVersion`（反映後のスキーマの版）。`content` なら `null` | スキーマの反映による変更を見分ける |
| `changes[].change` | 変更の種類。公開、更新、公開取り消し | 対象のページを更新する、または取り下げる |
| `changes[].contentId`、`model` | 変更した対象のコンテンツID と、モデルのキー | キャッシュを破棄し、公開取得で取り直す対象を決める |
| `changes[].uniqueValues` | 一意な値が変わった対象の、変更前と変更後 | 旧 URL からの転送を設定する |
| `referrers[].contentId`、`model` | 変更した対象を公開内容から参照するコンテンツの、コンテンツID とモデルのキー | 参照元のキャッシュを破棄し、公開取得で取り直す対象を決める |

- `changes` は通知の `changeList.changes`、`referrers` は `changeList.referrers`。値を変えずに写す。操作の ID、実行者、ワークスペースの ID、確定した日時を持たない
- 通知の内容は、同じ通知のすべての宛先とすべての試行で同じである。表示アプリは、重複して届いた通知を `notificationId` で、到着順が入れ替わった通知を `sequence` で見分ける
- 通し番号の割り当ては、[content.md](content.md) の `PublicationLedgerRepository` の「通し番号」に従う。ワークスペースの全体の順で、対象ごとの番号ではない。通知を作る時点に宛先が無い操作は通知にならないので、宛先に届く通知の番号は連続するとは限らない
- 順序の判別は、対象（`changes[].contentId` と `referrers[].contentId`）ごとに行う。ある対象の処理済みの番号より小さい番号の通知は、その対象については古い変更で、ほかの対象については古い変更と限らない
- 表示アプリは、通知を受けて、通知が示すモデルのキーとコンテンツID で、公開取得で内容を取り直す。ページは、取り直した内容から組み立てる

#### 通し番号と復旧

バックアップからの復旧の後の通し番号の規則は、技術設計の層では、ここだけが定める。

- バックアップからの復旧（HOST-08）の後は、Content の公開の変更の記録の台帳が、復旧点の通し番号の続きから割り当てる。復旧点より後の番号は、復旧の後の別の操作に割り当たり得て、表示アプリが処理済みとした番号より小さくなり得る
- CMS は、復旧を表示アプリへ通知しない。通知の内容は、復旧の前後で同じ項目だけを持つ
- 開発者が、利用の再開の後に、表示アプリのキャッシュと生成済みのページを公開取得で全体を取り直し、表示アプリが処理済みとして記録した通し番号を捨てる（HOST-08 手順 9、APP-11）。捨てた後の通知は、どの対象についても、新しい変更として処理される

## ドメインサービス

### DeliveryDispatch

責務: 一つの送信と、その通知、その時点の宛先から、送るか、送らずに失敗を記録するか、何もしないかを決める。

```ts
type OutboundNotification = Readonly<{
  url: WebhookUrl;
  secret: ProtectedWebhookSecret;
  payload: NotificationPayload;
}>;

type DispatchStep =
  | Readonly<{ action: "send"; attempt: DeliveryAttempt; message: OutboundNotification }>
  | Readonly<{
      action: "fail";
      attempt: DeliveryAttempt;
      failure: Readonly<{ kind: "notSent"; reason: "endpoint_deleted" }>;
    }>
  | Readonly<{ action: "ignore" }>;

const DeliveryDispatch: {
  plan(
    input: {
      delivery: WebhookDelivery;
      notification: ChangeNotification;
      endpoints: WebhookEndpoints | null;
    },
    now: Date,
  ): DispatchStep;
};
```

`plan` は、上の行から順に当てはめる。

| 条件 | 結果 |
| --- | --- |
| `WebhookDelivery.dueAttempt(delivery, now)` が `null`（送信待ちでない、または次の試行の時刻の前） | `ignore` |
| `WebhookEndpoints.find(endpoints, delivery.endpointId)` が `null`（宛先が削除されている） | `fail`。何も送らず、`endpoint_deleted` の失敗を記録する |
| 上のどれでもない | `send`。`message` は、その宛先の現在の URL と秘密情報、`ChangeNotification.payload(notification)` |

- `send` と `fail` の `attempt` は、`dueAttempt` が返した試行。その試行の結果を `WebhookDelivery.record` に渡すときに添える
- `endpoints` は、送信と同じ区画の、送る時点の `WebhookEndpoints`。`notification` は、`delivery.notificationId` の通知
- `plan` を呼ぶ主体、呼ぶ対象の送信、UnitOfWork の分け方は、[../usecases/webhook.md](../usecases/webhook.md) の「送信の試行」だけが定める。利用できないワークスペースの扱いは、[index.md](index.md) の「利用できるワークスペース」が定める

依存するポート: 無し。`plan` の結果を `WebhookSender` へ渡すのは、ユースケースである。

### DeliveryBatch

責務: 複数の送信への、手動再試行と対処の記録を、全部受け付けるか、どれも受け付けないかで決める。

```ts
type ConfirmedDelivery = Readonly<{ ref: DeliveryRef; delivery: WebhookDelivery | null; confirmedRound: number }>;

type RetryOutcome =
  | Readonly<{ outcome: "requested"; delivery: WebhookDelivery<Pending> }>
  | Readonly<{ outcome: "alreadyRequested"; delivery: WebhookDelivery }>;

type BatchRetryDecision =
  | (Readonly<{ accepted: true }> & WithEventDrafts<readonly RetryOutcome[], WebhookEvent>)
  | Readonly<{
      accepted: false;
      refusals: readonly Readonly<{ ref: DeliveryRef; reason: "notFound" | "notFailed" | "endpointDeleted" }>[];
    }>;

type BatchHandlingDecision =
  | Readonly<{
      accepted: true;
      deliveries: readonly WebhookDelivery<Failed>[];
      changed: readonly WebhookDelivery<Failed>[];
    }>
  | Readonly<{ accepted: false; refusals: readonly Readonly<{ ref: DeliveryRef; reason: "notFound" | "notFailed" }>[] }>;

const DeliveryBatch: {
  maxTargets: number;
  key(ref: DeliveryRef): string;
  requestRetries(
    input: { candidates: readonly ConfirmedDelivery[]; endpoints: WebhookEndpoints | null },
    now: Date,
  ): BatchRetryDecision;
  markHandled(
    input: { candidates: readonly ConfirmedDelivery[]; by: MemberId },
    now: Date,
  ): BatchHandlingDecision;
};
```

| メソッド | 処理 |
| --- | --- |
| `key` | 組を識別する文字列を返す。通知の ID と宛先の ID が同じ組にだけ、同じ文字列を返す |
| `requestRetries` | 「複数の組の手動再試行」が定める |
| `markHandled` | `candidates` の組ごとに、`delivery` が `null` なら `notFound` とし、ほかは `WebhookDelivery.markHandled(delivery, confirmedRound, by, now)` を適用して、`handled` が偽の送信を `notFailed` とする。該当する組が一件でもあれば、`accepted` を偽にして `refusals` に入れ、どの送信も変えない。無ければ、渡されたすべての送信の、適用の後の状態（どれも対処済み）を、渡された順で `deliveries` に入れる。そのうち、適用で変わった送信だけを、渡された順で `changed` に入れる。既に対処済みだった送信は、変更の無い変更で、`deliveries` に入り、`changed` に入らない。ユースケースは、`changed` の送信だけを保存する |

複数の組の手動再試行（`requestRetries`）。`candidates` の組ごとに、送信があれば `WebhookDelivery.requestRetry(delivery, confirmedRound, now)` を適用し、組ごとの結果を次の表で決めてから、全体の結果を決める。

| 組ごとの条件（上の行から順） | 組ごとの結果 |
| --- | --- |
| `delivery` が `null` | 受け付けない理由 `notFound`。`requestRetry` を適用しない |
| `requestRetry` の理由が `alreadyRequested` | `alreadyRequested`。受け付けない理由に数えない |
| `requestRetry` の理由が `notFailed` | 受け付けない理由 `notFailed` |
| `WebhookEndpoints.find` が、その送信の宛先を返さない | 受け付けない理由 `endpointDeleted`。`requestRetry` の結果を使わない |
| 上のどれでもない | `requested` |

| 全体の条件 | 全体の結果 |
| --- | --- |
| 受け付けない理由を持つ組が一件でもある | `accepted` が偽。該当する組と理由を `refusals` に入れる。どの送信も変えず、イベントを返さない |
| 上に当たらない | `accepted` が真。`requested` の組は `requested` と変えた送信、`alreadyRequested` の組は `alreadyRequested` と渡された送信を、`candidates` の順に `entity` に入れる。`eventDrafts` は、`requested` の組ごとに一件 |

- `alreadyRequested` の組の結果が持つ送信は、先に始まった再試行の今の状態を示す。ユースケースは、`requested` の送信だけを保存する
- `maxTargets` は、一度の操作で扱う組の数の上限。値は [../scenario/app-integration.md](../scenario/app-integration.md) の APP-12 の「一つの操作が対象にできる回答、通知、通知と宛先の組の件数」。組の列の受け方は、[../usecases/webhook.md](../usecases/webhook.md) の「共通」の「一度に扱う組」が定める
- `delivery` が `null` の組は、送信を読めない組（通知が無い、または、その宛先の送信が通知に無い組）
- `endpoints` は、そのワークスペースの、要求の時点の `WebhookEndpoints`

依存するポート: 無し。

### エラーコード

Webhook に固有のエラーの理由を、すべて列挙する。共通のコードは [common.md](common.md) の「エラー」が定める。

```ts
type WebhookErrorCode =
  // ValidationError（項目ごとの理由）
  | "InvalidWebhookUrl"
  | "InvalidWebhookSecret"
  | "TooManyEndpoints"
  // BusinessRuleError
  | "EndpointLimitReached";
```

| コード | 種類 | 意味 |
| --- | --- | --- |
| `InvalidWebhookUrl`、`InvalidWebhookSecret` | `ValidationError` | 宛先の URL、秘密情報が規則に合わない |
| `TooManyEndpoints` | `ValidationError` | 宣言した宛先が `DeliveryPolicy.maxEndpoints` 件を超える |
| `EndpointLimitReached` | `BusinessRuleError` | 宛先が既に `DeliveryPolicy.maxEndpoints` 件ある。上限を添える |

- `content.publication_changed` が届いた操作の、公開の変更の記録が無い場合の扱いは、[index.md](index.md) の「イベントの消費者と定期ジョブ」に従う

## ポート

リポジトリは、[common.md](common.md) の「TransactionalRepository」の節の、すべてのリポジトリに共通する規則に従う。下の表は、共通の規則に加える契約だけを書く。ページングは共通の `Pagination` と `PaginationResult` を使う。`findDue` の `limit` の上限は 100。

秘密情報の保護は、ユースケースが共通の `SecretCipher` で行う。

継ぐ基底と、メソッドの分類（[common.md](common.md) の「TransactionalRepository」）は次のとおり。基底のメソッドの分類は、common.md が定める。UnitOfWork の外の記録に当たるメソッドは無い。

| ポート | 継ぐ基底 | 加える読み取り | 加える書き込み |
| --- | --- | --- | --- |
| `WebhookEndpointsRepository` | `SingletonRepository<WebhookEndpoints>` | なし | `deleteAll` |
| `ChangeNotificationRepository` | 継がない。通知は楽観ロックの `version` を持たない | `findById`、`findByIds`、`findByOperationId` | `insert`、`deleteAll` |
| `WebhookDeliveryRepository` | `Omit<TransactionalRepository<WebhookDelivery, DeliveryRef>, "insert" \| "delete">` | `findByRefs`、`findByNotification`、`findPage`、`countUnhandled`、`countPending`、`findDue` | `insertMany`、`deleteAll` |

### WebhookEndpointsRepository

目的: ワークスペースの通知の宛先の列を保存する。区画に一つの集約なので、`SingletonRepository`（[common.md](common.md)）を継ぐ。集約の削除は、利用終了の消去の `deleteAll` だけが行う。

```ts
interface WebhookEndpointsRepository extends SingletonRepository<WebhookEndpoints> {
  deleteAll(limit: number): Promise<number>;
}
```

| メソッド | 契約 |
| --- | --- |
| `find` | 宛先を登録したことのないワークスペースでは `null` |
| `save` | 宛先の件数を、ポートは確かめない |
| `deleteAll` | 対象は、結び付いた区画の集約（0 または 1 件）。楽観ロックを使わない。その宛先を指す送信は変えない |

### ChangeNotificationRepository

目的: 変更の通知を保存し、ID と操作の ID で返す。通知は楽観ロックの `version` を持たないので、`TransactionalRepository` を継がず、持つメソッドをすべてここに宣言する。

```ts
interface ChangeNotificationRepository {
  insert(notification: ChangeNotification): Promise<void>;
  findById(id: ChangeNotificationId): Promise<ChangeNotification | null>;
  findByIds(ids: readonly ChangeNotificationId[]): Promise<readonly ChangeNotification[]>;
  findByOperationId(operationId: OperationId): Promise<ChangeNotification | null>;
  deleteAll(limit: number): Promise<number>;
}
```

| メソッド | 契約 |
| --- | --- |
| `insert` | 通知が持つ変更と参照元を、件数によらずすべて保存し、切り詰めない。同じ区画の中で、通知の ID と、操作の ID が、それぞれ一意であることを、ポートが担保する。どちらの重複も `ConflictError`（理由 `ALREADY_EXISTS`）。同じイベントを並行して消費した `insert` は、一方だけが確定する。通し番号の一意性は、ポートが確かめない。通知の送信の存在は担保しない |
| `findById` | `id` で一件を返す。無ければ `null`。`version` を返さない。対象のコンテンツと宛先が削除されていても返す |
| `findByIds` | 渡した ID の通知を、通し番号の降順、同じ通し番号の中は通知の ID の昇順で返す。渡せる ID は `Pagination.maxLimit`（[common.md](common.md)）件以下 |
| `findByOperationId` | 結び付いた区画の、その操作の ID の通知を返す。無ければ `null` |
| `deleteAll` | 対象は、結び付いた区画のすべての通知。その通知の送信は削除しない |

- 並行性: 楽観ロックを使わない
- 可視性: 確定した書き込みは、以後のすべての読み取りに即座に反映する

### WebhookDeliveryRepository

目的: 通知と宛先の組ごとの送信を保存し、組の一覧、未対処と送信待ちの件数、定期ジョブが送る送信を返す。送信は、一つの通知の全部を `insertMany` で作り、利用終了の消去の `deleteAll` で削除する。`TransactionalRepository` から `insert` と `delete` を除いて継ぐ。

```ts
type DueCursor = Readonly<{
  overdueAt: Date;
  notificationId: ChangeNotificationId;
  endpointId: WebhookEndpointId;
}>;

type DeliveryListFilter = Readonly<{
  handling?: "unhandled" | "handled";
}>;

interface WebhookDeliveryRepository
  extends Omit<TransactionalRepository<WebhookDelivery, DeliveryRef>, "insert" | "delete"> {
  insertMany(deliveries: readonly WebhookDelivery[]): Promise<void>;
  findByRefs(refs: readonly DeliveryRef[]): Promise<readonly Versioned<WebhookDelivery>[]>;
  findByNotification(notificationId: ChangeNotificationId): Promise<readonly WebhookDelivery[]>;
  findPage(filter: DeliveryListFilter, pagination: Pagination): Promise<PaginationResult<WebhookDelivery>>;
  countUnhandled(): Promise<number>;
  countPending(): Promise<number>;
  findDue(
    now: Date,
    page: Readonly<{ after: DueCursor | null; limit: number }>,
  ): Promise<Readonly<{ refs: readonly DeliveryRef[]; next: DueCursor | null }>>;
  deleteAll(limit: number): Promise<number>;
}
```

| メソッド | 契約 |
| --- | --- |
| `insertMany` | 一つの通知の送信を、一回の呼び出しで保存する。同じ組（通知の ID と宛先の ID）の送信が既にある場合と、渡した列の中に同じ組がある場合は `ConflictError`（理由 `ALREADY_EXISTS`）で、どの送信も保存しない。ポートが担保する。空の列は、何も書かない。書き込みの量は、件数によらず一回。通知と宛先の存在は担保しない |
| `findById` | 組で一件を返す。対象のコンテンツと宛先が削除されていても返す |
| `save` | 楽観ロックで保存する。同じ送信への並行した結果の記録、手動再試行、対処の記録は、一方だけが確定する。同じ通知の別の宛先の送信の `save` とは競合しない |
| `findByRefs` | 渡した組の送信を、`version` とともに、通知の ID、宛先の ID の昇順で返す。渡せる組は `DeliveryBatch.maxTargets` 件以下 |
| `findByNotification` | 渡した通知のすべての送信を、宛先の ID の昇順で返す。送信が無ければ空の列。`version` を返さない |
| `findPage` | 下のクエリ要件に従う |
| `countUnhandled` | 結び付いた区画の、`failed` で未対処の送信の数を返す。`findPage` を `handling: "unhandled"` で呼んだ結果の `count` と一致する |
| `countPending` | 結び付いた区画の、`pending` の送信の数を返す。ワークスペースの利用終了の前に、送信を終えていない通知を示すために使う |
| `findDue` | 定期ジョブ「表示アプリへの通知の再試行」（[index.md](index.md) の「定期ジョブ」）が、対象のワークスペースごとに使う。呼び出しの時点で `WebhookDelivery.isOverdue(delivery, now)` が真の送信の組を、`overdueAt` の昇順、同じ時刻は通知の ID、宛先の ID の昇順で、`after` より後ろから `limit` 件まで返す。`next` は続きがあれば最後に返した送信の位置、無ければ `null`。結び付いた区画の送信だけを対象にし、ワークスペースの状態を確かめない |
| `deleteAll` | 対象は、結び付いた区画のすべての送信。送信結果によらない。楽観ロックを使わない |

- 可視性: 確定した書き込みは、以後のすべての読み取りに即座に反映する

`findDue` の続きを読む位置。

- 続きを読む位置は、一つのワークスペースについての一回の起動の中で、同じ送信を二度返さないためのものである
- 一回の起動は、起動の時点の `now` で対象だった送信を、`overdueAt` の早い順に一度ずつ返す
- 起動の途中で `overdueAt` が変わった送信（その起動が送って失敗した送信、手動再試行を要求された送信）は、`overdueAt` が起動の時点の `now` より後になるので、その起動では返さない。後の起動が扱う
- 起動の途中で送信成功または送信失敗になった送信は、`pending` でないので返さない

`findPage` のクエリ要件。

- 一覧の単位は、通知と宛先の組である。一つの通知は、宛先の数だけの組になる
- `handling` を指定しなければ、結び付いた区画のすべての組を返す
- `handling: "unhandled"` は、`failed` で未対処の組だけを返す。`handling: "handled"` は、`failed` で対処済みの組だけを返す
- 並び順は、通し番号（`sequence`）の降順、同じ通し番号の中は通知の ID の昇順、同じ通知の中は宛先の ID の昇順。通し番号の一意性をポートが確かめないので、同じ通し番号の二つの通知の組も、この順で一つに決まる
- `count` は、条件に該当する組の総数

### WebhookSender

目的: 一つの宛先へ、一つの通知を一回送る。置き場所は `packages/core/src/domain/webhook/ports/`。

```ts
type SendResult =
  | Readonly<{ status: "delivered" }>
  | Readonly<{ status: "failed"; failure: Undelivered | Readonly<{ kind: "notSent"; reason: "secret_unavailable" }> }>;

interface WebhookSender {
  send(message: OutboundNotification, at: Date): Promise<SendResult>;
}
```

| メソッド | 契約 |
| --- | --- |
| `send` | `message.url` へ、`message.payload` を本文にした要求を一回送る。結果は `SendResult` で返し、送信の失敗と、秘密情報の保護を解けないことで例外を投げない |

送信の規則。表示アプリが受け取る要求の形は、このポートの契約である。どのアダプターも同じ要求を送る。

| 要求の部分 | 内容 |
| --- | --- |
| メソッド | `POST`。宛先は `message.url` |
| 本文 | `NotificationPayload` の JSON（UTF-8）。項目の名前は `NotificationPayload`、`PublishedChangeNotice`、`UniqueValueChange`、`PublishedReferrer` の型の名前と同じ。列の要素は、`payload` の順のまま並べる。`null` の項目と空の列も省かない |
| `Content-Type` ヘッダー | `application/json` |
| `X-Lane-Timestamp` ヘッダー | `at` の UNIX 時刻（秒）の 10 進表記。秒未満を切り捨てる |
| `X-Lane-Signature` ヘッダー | 署名。16 進の小文字 |

- 署名は、`X-Lane-Timestamp` の値、`.`、本文のバイト列をこの順に連結した値の、HMAC-SHA256 である。鍵は、保護を解いた秘密情報の UTF-8 のバイト列
- 署名した本文と、送る本文は、同じバイト列である。表示アプリは、受け取った本文のバイト列から署名を計算し直す
- 表示アプリは、同じ秘密情報で署名を確かめて、通知が CMS から届いたことを確かめる。`X-Lane-Timestamp` は、その要求を送った時刻で、試行ごとに変わる。CMS は、時刻の古さで通知を拒否することを表示アプリに求めない
- 秘密情報の保護を解くのは、アダプターである。保護を解いた値を、戻り値、要求、ログ、エラー情報に出さない

送信失敗の判定。

| 条件 | 結果 |
| --- | --- |
| `DeliveryPolicy.timeoutMs` の間に受け取った最終の応答の状態が 200 番台 | `delivered` |
| 最終の応答の状態が 200 番台でない。転送を示す応答（300 番台）を含む。転送先へは送らない | `failed`（`undelivered`、`rejected`、応答の状態） |
| `DeliveryPolicy.timeoutMs` の間に、最終の応答の状態を受け取らなかった | `failed`（`undelivered`、`timeout`） |
| 接続できなかった。名前を解決できない、接続を拒否された、暗号化の通信を確立できない | `failed`（`undelivered`、`unreachable`） |
| 秘密情報の保護を解けない（`SecretCipher.reveal` の `SECRET_UNAVAILABLE` に当たる） | `failed`（`notSent`、`secret_unavailable`）。要求を送らない |

- 待ち時間は、要求を送り始めてから、最終の応答の状態を受け取るまでを数える。応答の本文の受信を待たない
- 情報の応答（100 番台）は、判定に使わない。その後の最終の応答の状態で判定する
- 一意性・整合性: 同じ通知の重複を抑えない。呼び出すたびに一回送る。自動の再試行は、ポートではなく `WebhookDelivery` の規則が決める
- 並行性: 状態を持たない。並行した `send` は互いに影響しない
- 可視性: UnitOfWork に属さない（`fn` の中で呼んでも、UnitOfWork と確定を共にしない）
- 宛先の応答の本文を読まず、保存しない

## トランザクション境界

共通の `UnitOfWorkProvider.run(scope, fn)` を使う。Webhook の書き込みは、`{ kind: "workspace", workspaceId, condition }` の `WriteScope` で行い、`fn` が受ける `WorkspaceContext` の `webhookEndpointsRepository`、`changeNotificationRepository`、`webhookDeliveryRepository` を使う。リポジトリの読み取り、UnitOfWork の規則、`WriteScope` は [common.md](common.md) に従う。

- `WriteScope` の条件、UnitOfWork の分け方、一つの UnitOfWork の書き込みの回数は、[../usecases/webhook.md](../usecases/webhook.md) の「共通」、各ユースケースの「トランザクション境界」、「送信の試行」だけが定める。途中で止まったときに残る状態は、ユースケースのテスト（`spec/testcases/webhook/`）が定める。Installation が呼ぶ手順（「初期構成の宣言」「利用終了の消去」）の UnitOfWork は、それぞれの節が定める
- 競合の扱いは [index.md](index.md) の「競合の扱い」、イベントの消費者と定期ジョブが処理を行えない対象に当たったときの扱いは同じ文書の「イベントの消費者と定期ジョブ」、通知を作れない・送れないワークスペースの扱いは同じ文書の「利用できるワークスペース」が定める

## 初期構成の宣言

初期構成の読み込みで、Webhook が受ける宣言と、その検証・生成の手順。宣言の形の全体と、`path` に渡す位置は、[installation.md](installation.md) の「初期構成の読み込み」が、各ドメインを呼ぶ順は [index.md](index.md) の「手順を呼ぶ順」が定める。`DeclarationIssue` は [common.md](common.md) が定める。生成は、Installation のユースケースが、宣言したワークスペースごとに、その区画の一つの UnitOfWork（`{ kind: "workspace", workspaceId, condition: { kind: "none" } }`）で、ほかのドメインの生成とともに確定する（[index.md](index.md) の「区画をまたぐ手順」の初期構成の読み込み）。

```ts
const WebhookEndpointsDeclaration: {
  validate(
    entries: readonly Readonly<{ path: string; declaration: readonly WebhookEndpointInput[] }>[],
  ): readonly DeclarationIssue[];
};
```

宣言は、ワークスペースの表示アプリへの通知の宛先の列（`WebhookEndpointInput`）。宛先を宣言しないワークスペースは、`entries` に含めない。

- 宣言の照合値から除く値: 宛先ごとの秘密情報（`WebhookEndpointInput.secret`）。宛先の URL は、照合値に含める
- 前提: 先に生成を終えているべきドメインは無い。宛先の集約は、ほかのドメインの集約を参照しない

検証。

| 規則 | `path` | `code` |
| --- | --- | --- |
| `WebhookEndpointInput.validateAll` | `index` が `null` でない問題は、渡された `path` に、`index` と `field` を `.` でつないで続ける。`index` が `null` の問題は、渡された `path` | 返った `EndpointListIssue.code` |

生成。宛先を宣言したワークスペースごとに、その区画の UnitOfWork で、`WebhookEndpoints.create` に続けて、宛先ごとに、`IdGenerator.generate` の ID で `WebhookEndpoints.register` を適用し、結果を `WebhookEndpointsRepository.insert` で保存する。

- `register` に渡す秘密情報は、UnitOfWork の前に、宛先ごとに `SecretCipher.protect` で保護し、`ProtectedWebhookSecret.wrap` で包んだ値
- 書き込みは、宛先の数によらず `insert` の一回
- 宛先を宣言しないワークスペースは、集約を作らない

## 状態確認の項目

設置の状態確認（Installation の `checkInstallationStatus`）と、利用終了の前の確認に、Webhook が示す件数の項目。[common.md](common.md)「各ドメインが公開する項目」の `StatusItem`（`domain` は `webhook`）で公開する。Installation のユースケースが、ワークスペースごとに、`PartitionReader.workspace(workspaceId)` のリポジトリで読んだ値を渡して求める。

```ts
type WebhookStatusItem = StatusItem &
  Readonly<{ domain: "webhook" }> &
  (Readonly<{ kind: "deliveries_unhandled" }> | Readonly<{ kind: "deliveries_pending"; actionRequired: false }>);

const WebhookStatus: {
  of(counts: Readonly<{ pending: number; unhandled: number }>): readonly WebhookStatusItem[];
};
```

| `kind` | 項目 | 件数 | 必要な対処（`actionRequired`） |
| --- | --- | --- | --- |
| `deliveries_unhandled` | 未対処の送信失敗の表示アプリへの通知 | `WebhookDeliveryRepository.countUnhandled()` の結果。通知と宛先の組の数 | 件数が 1 以上なら真。アプリ連携の権限を持つメンバーが、管理画面で、再試行するか、対処済みとして記録する |
| `deliveries_pending` | 送信待ちの表示アプリへの通知 | `WebhookDeliveryRepository.countPending()` の結果。通知と宛先の組の数 | 常に偽 |

- `of` は、`deliveries_pending`、`deliveries_unhandled` の二つの項目を、この順に返す。件数が 0 の項目も返す。状態確認と利用終了の前の確認は、同じ項目を示す（HOST-06 と HOST-09 の「未完了の通知」）
- 件数だけを示す。宛先の URL、秘密情報、通知の内容を示さない
- 届いていない `content.publication_changed`（隔離されたイベント）は、この項目に数えない。隔離されたイベントの件数（[common.md](common.md) の「QuarantinedEventQuery」）が示す

## 利用終了の消去

ワークスペースの利用終了で、Webhook の集約を、そのワークスペースの区画で消去する。Installation のユースケースが呼ぶ順は、[index.md](index.md) の「手順を呼ぶ順」が定める。

- 前提: 先に消去を終えているべきドメインは無い。Webhook の集約を参照する、ほかのドメインの集約は無い。Webhook の集約は、すべてこの区画にあり、設置の区画に消去するものを持たない

消去するのは、次の三つのメソッドの対象。どれも `{ kind: "workspace", workspaceId, condition: { kind: "none" } }` の UnitOfWork の内で呼ぶ。呼び方は、[common.md](common.md) の「TransactionalRepository」の件数を区切る削除に従う。

| 呼ぶメソッド | 消去するもの |
| --- | --- |
| `WebhookDeliveryRepository.deleteAll(BoundedDeletion.maxLimit)` | 通知と宛先の組ごとの送信 |
| `ChangeNotificationRepository.deleteAll(BoundedDeletion.maxLimit)` | 変更の通知 |
| `WebhookEndpointsRepository.deleteAll(BoundedDeletion.maxLimit)` | 通知の宛先の列（URL と秘密情報） |

- 三つのメソッドは、互いの対象を読まない。呼ぶ順は、結果を変えない
- 利用終了を始めたワークスペースの送信待ちは送らず、消去が削除する

利用終了の前の確認に、Webhook が公開する内容。種類の名前だけの項目は、[common.md](common.md)「各ドメインが公開する項目」の `DomainKind`（`domain` は `webhook`）の列で公開する。

```ts
type WebhookClosureKind<K extends string> = DomainKind & Readonly<{ domain: "webhook"; kind: K }>;

const WebhookClosure: {
  pending(counts: Readonly<{ pending: number; unhandled: number }>): readonly WebhookStatusItem[];
  lostData: readonly WebhookClosureKind<"change_notification" | "webhook_endpoints">[];
  remainsOutside: readonly WebhookClosureKind<"delivered_notification">[];
  erasedCredentials: readonly WebhookClosureKind<"webhook_endpoint_secret">[];
};
```

| 公開するもの | 値 | 内容 |
| --- | --- | --- |
| 未完了の通知（`pending`） | `WebhookStatus.of` と同じ項目（「状態確認の項目」） | 送信待ちは、利用終了で送られずに消える。未対処の送信失敗は、再試行できなくなる |
| 利用終了で失われるデータ（`lostData`） | `change_notification` | 変更の通知と、宛先ごとの送信結果 |
| | `webhook_endpoints` | 通知の宛先 |
| CMS の通常データの外に残る情報（`remainsOutside`） | `delivered_notification` | 表示アプリが受け取った通知と、それに基づいて表示アプリが保持した内容。CMS は回収しない |
| 消去する資格情報（`erasedCredentials`） | `webhook_endpoint_secret` | 宛先ごとの秘密情報 |

## ユースケース（概要）

- 通知の宛先を登録する
- 通知の宛先の URL を変更する
- 通知の宛先の秘密情報を置き換える
- 通知の宛先を削除する
- 通知の宛先を一覧する
- 通知と宛先の組を一覧する
- 一つの通知の送信結果を確かめる
- 未対処の送信失敗の件数を求める
- 公開内容の変更の通知を作る
- 送信を求められた通知を送る
- 表示アプリへの通知を再試行する
- 送信失敗の通知を再試行する
- 送信失敗を対処済みとして記録する

初期構成で宣言した宛先の作成、ワークスペースの利用終了の消去、状態確認は、Installation のユースケースが、この文書の振る舞い・手順・項目を使って行う。
