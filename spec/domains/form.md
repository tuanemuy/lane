# Form

フォームの受付、回答の保存、メール通知とその送信結果を保つ。境界、依存方向、集約、ドメインをまたぐ規則は [index.md](index.md)、共通の型とポート（集約の ID、`EmailAddress`、`Actor`、`FailureHandling`、`Pagination`、`BatchTargets`、`TransactionalRepository`、`SingletonRepository`、`WriteScope`、`UnitOfWorkProvider`、`UsageLimit` と `UsageLimiter`、`WithEventDrafts`、`ProtectedSecret` と `SecretCipher`、`StatusItem` と `DomainKind`）は [common.md](common.md) が定める。現行のフォームの定義、項目の値、値の検証は [schema.md](schema.md) の型と関数を使う。回答が持つ受付時のフォームは、Form の型（「ReceivedForm」）で持ち、Schema の `FormDefinition` を持たない。

コードは `packages/core/src/domain/form/` に置く。エンティティの振る舞い、値オブジェクト、ドメインサービスは、入出力を持たない純粋な関数で、同じ引数から同じ結果を返す。Content ドメインの型と振る舞いは使わず、`ContentId` と `ModelKey` だけを使う。

## ユビキタス言語

| 英語名 | 日本語名 | 定義 |
| --- | --- | --- |
| Form settings | フォームの運用設定 | フォーム一つの受付状態、通知先、通知定義、回答の保持方針 |
| Pending form settings | 紐付け待ちの設定 | 初期構成でフォームのキーを指定して宣言し、そのキーのフォームの作成を待つ運用設定 |
| Reception | 受付状態 | 受付中（`open`）または受付停止（`closed`） |
| Recipient | 通知先 | 回答の通知を受け取るメールアドレスと、有効・無効 |
| Notification template | 通知定義 | 通知メールの件名・本文・送信者の表示名・返信先 |
| Template reference | 参照 | 通知定義が、フォームの項目をキーで指すこと |
| Broken reference | 参照の不整合 | 参照するキーの項目がフォームに無い、または項目の値の種類が参照の用途に合わないこと |
| Mail connection | 送信接続設定 | ワークスペースに一つの、メール送信基盤への接続先・認証情報・接続条件。接続条件は、送信のアダプターが解釈する名前と値の組で、送信者のアドレスを含む |
| Mail credentials | 認証情報 | 送信接続設定の、利用者を識別する値と秘密の組。登録の後は、登録済みかどうかだけを示す |
| Submission | 回答 | 受け付けた送信一件の記録。受付時のフォーム、入力内容、問い合わせ対象の送信時の情報、通知定義、通知対象、保持方針を持つ |
| Received form | 受付時のフォーム | 回答が持つ、受付時のフォームの定義の写し。キー、表示名称、項目ごとのキー・表示名称・値の種類・選択肢、メールアドレスの書式を求める項目かどうか |
| Submission notification | 回答の通知 | 回答一件の通知の進行。送信待ち、送信中、送信成功、送信失敗のいずれかの状態と、通知の回、宛先ごとの送信の結果を持つ |
| Inquiry target | 問い合わせ対象 | フォームのコンテンツ参照の項目で、送信者が指定したコンテンツ |
| Reception result | 受付結果 | 受付完了、または理由付きの受付失敗 |
| Notified recipients | 通知対象 | 受付時に有効だった通知先のメールアドレスの組 |
| Attempt | 通知の回 | 一件の回答の通知を送る一回の試み。最初の通知（`initial`）、または n 回目の手動再試行（`retry`） |
| Delivery | 宛先ごとの送信の結果 | 一つの宛先への、最新の送信の結果。受理を確認済み、または失敗 |
| Recipient status | 宛先ごとの送信の状態 | 一つの宛先への送信の状態。送信中、受理を確認済み、失敗のいずれか。通知の状態と、宛先ごとの送信の結果から求める |
| Notification result | 通知の送信結果 | 未送信、送信成功、送信失敗のいずれか。通知の状態から求める。送信を終えていない通知（送信待ちと送信中）は未送信 |
| Retention policy | 回答の保持方針 | 回答を受付日時から保持する日数 |
| Confidential | 秘匿値 | ログ・操作履歴・エラー情報・ドメインイベントへ出さない値を包む値。回答の入力内容、問い合わせ対象の送信時の情報、描画した通知メール、回答の検索の文字列、保護する前の認証情報を包む |

## 秘匿値

ログ、操作履歴、エラー情報、ドメインイベントへ出さない値は、`Confidential<T>` に包んで持つ。

```ts
type Confidential<T> = Readonly<{
  readonly [confidentialBrand]: T;
  toJSON(): "[confidential]";
  toString(): "[confidential]";
}>;

const Confidential: {
  of<T>(value: T): Confidential<T>;
  reveal<T>(wrapped: Confidential<T>): T;
};
```

- 包んだ値は、プロパティの列挙、JSON への変換、文字列への変換、検査用の表示から届かない。どの経路も固定の文字列 `[confidential]` を返す
- 包んだ値を取り出すのは `Confidential.reveal` だけ。取り出した値を渡す先は、その値を保存・送信・保護するポートと、その値を見る権限を確かめた実行者への出力に限る。ログ、操作履歴、エラー情報、ドメインイベントへ渡さない
- ドメインイベントのペイロード、`SubmissionNotification`、`SubmissionIssue`、`FormInputIssue`、`BusinessRuleError`、`DeliveryFailureReason` は、型に入力内容を持つ場所が無い。ログ、操作履歴、エラー情報へ渡すのは、これらと ID に限る
- 回答の入力内容と問い合わせ対象の送信時の情報を出力に含めるのは、回答の一覧と回答の詳細（Form のユースケース `searchSubmissions` と `getSubmission`）だけ。返す実行者は、[../usecases/form.md](../usecases/form.md) の「権限による出し分け」が定める
- 等価性は比較しない

## エンティティ

### FormSettings

フォーム一つの運用設定。対象を `FormId` で指し、ワークスペースの区画の中で `formId` が識別する。初期構成でキーを指定して宣言した運用設定は、対象の作成を待つ間、別の集約 `PendingFormSettings`（紐付け待ちの設定）として持つ。

```ts
type FormSettingsCommon = Readonly<{
  recipients: RecipientList;
  version: number;
  createdAt: Date;
  updatedAt: Date;
}>;

type ActiveFormSettings = FormSettingsCommon &
  Readonly<{
    state: Readonly<{ kind: "active"; formId: FormId }>;
    reception: Reception;
    notification: NotificationTemplate;
    retention: RetentionPolicy;
  }>;

type RemovedFormSettings = FormSettingsCommon &
  Readonly<{
    state: Readonly<{ kind: "removed"; formId: FormId; formKey: FormKey; formLabel: string | null; removedAt: Date }>;
  }>;

type FormSettings = ActiveFormSettings | RemovedFormSettings;
type Reception = "open" | "closed";
```

| フィールド | 制約 |
| --- | --- |
| `state` | `active` は有効、`removed` は削除済み |
| `recipients` | 通知先。規則は「RecipientList」 |
| `reception` | 新規フォームは `closed` |
| `retention` | 新規フォームは `RetentionPolicy.initial()` |
| `formKey` / `formLabel`（`removed`） | 削除の時点のフォームのキーと表示名称。削除済みフォームの表示は、「削除済みフォームの表示」が定める |
| `version` | 楽観ロックの番号。作成時は 0。保存する変更ごとに 1 進む |

#### 削除済みフォームの表示

利用者に見える規則は [../scenario/form-operation.md](../scenario/form-operation.md) の FORM-11 が定める。表示の単位ごとに示すキーと表示名称は、次のとおり。

| 表示の単位 | 示すキーと表示名称 |
| --- | --- |
| フォーム（フォームごとに一行を示す一覧） | 状態「削除済み」と、削除の時点のキーと表示名称（`removed` の `formKey` と `formLabel`） |
| 回答（回答の一覧と詳細） | その回答の受付時のフォームのキーと表示名称（`Submission.form`） |
| 操作履歴の対象（Audit の `FormTarget`）の名称 | フォームのキー。`removed` の設定では、削除の時点のキー（`formKey`） |

削除済みフォームは通知定義を持たないので、参照の不整合の有無は `null` で表す。

#### 現行のフォームとの対応

設定が現行のフォームのものかどうかは、設定より先に読んだ `WorkspaceSchema` の `formIds` で決まる。この規則は、この節だけが定める。

- `formIds` にその `formId` がある `active` の設定が、現行のフォームの設定
- `formIds` にその `formId` が無い `active` の設定（`WorkspaceSchema` を読んだ後に、反映が追加したフォームの設定）は、無い設定として扱う。一件の設定を指す操作では、設定が無い場合と同じ `NotFoundError`、一覧ではその設定を含めない
- `WorkspaceSchema` を読んだ後に、反映が削除したフォームの設定は、後に読む設定が `removed` なので、`removed` の設定として扱う

#### 振る舞い

```ts
const FormSettings: {
  createForNewForm(
    params: Readonly<{ formId: FormId; form: FormDefinition }>,
    now: Date,
  ): ActiveFormSettings;
  bind(pending: PendingFormSettings, formId: FormId, form: FormDefinition, now: Date): ActiveFormSettings;
  remove(settings: ActiveFormSettings, form: FormDefinition, now: Date): RemovedFormSettings;

  open(settings: ActiveFormSettings, facts: Readonly<{ mailConnectionRegistered: boolean }>, now: Date): ReceptionOpening;
  missingForReception(
    recipients: RecipientList,
    facts: Readonly<{ mailConnectionRegistered: boolean }>,
  ): readonly ReceptionRequirement[];
  close(settings: ActiveFormSettings, now: Date): Change<ActiveFormSettings>;

  addRecipient<S extends FormSettings>(settings: S, address: EmailAddress, now: Date): S;
  changeRecipientAddress<S extends FormSettings>(
    settings: S,
    from: EmailAddress,
    to: EmailAddress,
    now: Date,
  ): Change<S>;
  setRecipientEnabled<S extends FormSettings>(
    settings: S,
    address: EmailAddress,
    enabled: boolean,
    now: Date,
  ): Change<S>;
  removeRecipient<S extends FormSettings>(settings: S, address: EmailAddress, now: Date): S;

  changeNotification(
    settings: ActiveFormSettings,
    template: NotificationTemplate,
    form: FormDefinition,
    now: Date,
  ): NotificationChange;
  changeRetention(settings: ActiveFormSettings, retention: RetentionPolicy, now: Date): Change<ActiveFormSettings>;
  retentionToApply(settings: ActiveFormSettings, confirmed: RetentionPolicy): RetentionApplication;

  planSchemaChange(
    input: Readonly<{
      settings: readonly Versioned<FormSettings>[];
      pending: readonly Versioned<PendingFormSettings>[];
      added: readonly Readonly<{ formId: FormId; form: FormDefinition }>[];
      removed: readonly Readonly<{ formId: FormId; form: FormDefinition }>[];
    }>,
    now: Date,
  ): FormSettingsSchemaChange;

  requireActive(settings: FormSettings): ActiveFormSettings;
  enabledRecipients(settings: FormSettings): readonly EmailAddress[];
  isAccepting(settings: FormSettings): boolean;
  referenceIssues(settings: ActiveFormSettings, form: FormDefinition): readonly ReferenceIssue[];
};

type Change<T> =
  | Readonly<{ kind: "changed"; value: T }>
  | Readonly<{ kind: "unchanged"; current: T }>;

type RetentionApplication =
  | Readonly<{ apply: true; policy: RetentionPolicy }>
  | Readonly<{ apply: false; current: RetentionPolicy }>;

type FormSettingsSchemaChange = Readonly<{
  created: readonly ActiveFormSettings[];
  bound: readonly Readonly<{
    pending: Readonly<{ value: PendingFormSettings; expectedVersion: ExpectedVersion<PendingFormSettings> }>;
    settings: ActiveFormSettings;
  }>[];
  removed: readonly Readonly<{ settings: RemovedFormSettings; expectedVersion: ExpectedVersion<FormSettings> }>[];
}>;

type ReceptionRequirement = "enabledRecipient" | "mailConnection";

type ReceptionOpening = Change<ActiveFormSettings> | Readonly<{ kind: "notReady"; missing: readonly ReceptionRequirement[] }>;

type NotificationChange = Change<ActiveFormSettings> | Readonly<{ kind: "broken"; issues: readonly ReferenceIssue[] }>;
```

| 振る舞い | 処理 |
| --- | --- |
| `createForNewForm` | `active`、`closed`、通知先なし、`NotificationTemplate.initial(form)`、`RetentionPolicy.initial()` で作る |
| `bind` | 紐付け待ちの設定の通知先、受付状態、保持方針を引き継いだ `active` を、`version` 0 で作る。その `notification` が `null` なら `NotificationTemplate.initial(form)` にする。通知定義の参照の不整合は紐付けを妨げない。`form.key` が紐付け待ちの設定の `formKey` と異なれば `FormKeyMismatch` |
| `remove` | `removed` にする。通知先を引き継ぎ、`formKey` と `formLabel` に `form` のキーと表示名称を入れる。受付状態、通知定義、保持方針を持たなくなる。受付状態によらず行える |
| `open` | 受付中なら `unchanged`。`missingForReception` が空なら `open` にする。空でなければ変更せず、`notReady` と足りないものの一覧を返す。ユースケースは、`notReady` を一つの `BusinessRuleError`（`ReceptionNotReady`）にし、一覧を添える。通知定義の参照の不整合は妨げない |
| `missingForReception` | 受付を有効にする条件のうち、足りないものを返す。有効な通知先が一件も無ければ `enabledRecipient`、`mailConnectionRegistered` が偽なら `mailConnection`。この順に並べる |
| `close` | 受付停止なら `unchanged`。ほかは `closed` にする |
| `addRecipient` | `RecipientList.add` で、有効な通知先を加える |
| `changeRecipientAddress` | `RecipientList.changeAddress` で、通知先のメールアドレスを置き換える |
| `setRecipientEnabled` | `RecipientList.setEnabled` で、通知先の有効・無効を変える |
| `removeRecipient` | `RecipientList.remove` で、通知先を取り除く |
| `changeNotification` | `NotificationTemplate.referenceIssues(template, ReceivedForm.of(form))` が空でなければ、変更せず、`broken` と不整合を返す。現在と等しい通知定義でも同じ。空で、現在と等しい通知定義なら `unchanged`。ほかは通知定義を置き換える。ユースケースは、`broken` を `BusinessRuleError`（`NotificationReferenceBroken`）にし、不整合の一覧を添える。`form` は現行スキーマのフォームの定義 |
| `changeRetention` | 同じ日数なら `unchanged`。ほかは保持方針を置き換える。保存済みの回答は変えない |
| `retentionToApply` | 既存の回答へ適用する保持方針を決める。`confirmed` は、メンバーが適用範囲と削除予定を確認した保持方針。`settings.retention` が `confirmed` と等しければ `apply` が真で、その保持方針を返す。等しくなければ `apply` が偽で、現在の保持方針を返す。確認していない保持方針を、既存の回答へ適用しない |
| `planSchemaChange` | スキーマの反映の確定に伴う設定と紐付け待ちの設定の書き込みを求める。「スキーマの反映に伴う変更」が定める |
| `requireActive` | `active` の設定をそのまま返す。`removed` の設定は `BusinessRuleError`（`FormRemoved`） |
| `enabledRecipients` | `RecipientList.enabled(settings.recipients)` を返す |
| `isAccepting` | `active` で `open` のときに真 |
| `referenceIssues` | `NotificationTemplate.referenceIssues(settings.notification, ReceivedForm.of(form))` を返す |

受付中の設定が通知先について満たす条件は、`missingForReception` の `enabledRecipient` だけが定める。`enabledRecipient` を返すかどうかは `recipients` だけで決まる。`setRecipientEnabled` と `removeRecipient` は、`active` で `open` の設定について、変更後の `recipients` に `missingForReception` が `enabledRecipient` を返す変更を、`LastEnabledRecipient` で拒否する。`removed` と `closed` の設定には、この制限が無い。

#### 変更の結果

`Change` は、[index.md](index.md) の「変更の無い変更」に当たるかどうかを、振る舞いの結果で表す。`unchanged` は、現在と同じ値への変更で、`current` は渡された集約。`changed` の `value` は、変更を反映して `version` を 1 進めた集約で、`updatedAt` を持つ集約では `updatedAt` を `now` にする。どの変更が `unchanged` に当たるかは、各振る舞いの処理が定める。`MailConnection.update` と `SubmissionNotification.markHandled` も、この型を使う。

#### スキーマの反映に伴う変更

`planSchemaChange` は、スキーマの反映が追加したフォームと削除したフォーム、そのワークスペースの現在の設定と紐付け待ちの設定から、`FormSettingsSchemaChange` を求める。呼ぶ UnitOfWork と、結果を渡すポートのメソッドは、[../usecases/schema.md](../usecases/schema.md) の「反映を確定する（`commitSchemaApply`）」が定める。反映の後に、どの設定がどのフォームに対応するかは、この節の紐付けだけが定める。反映の準備の検証（「NotificationReferenceCheck」）も、この紐付けを使う。

| 入力 | 内容 |
| --- | --- |
| `settings` | `FormSettingsRepository.findAll` が返した、そのワークスペースの設定と `version` |
| `pending` | `PendingFormSettingsRepository.findAll` が返した、そのワークスペースの紐付け待ちの設定と `version` |
| `added` | 反映が追加したフォーム。割り当てた `FormId` と、反映後のフォームの定義 |
| `removed` | 反映が削除したフォーム。その `FormId` と、反映前のフォームの定義 |

紐付け。Form は、紐付け待ちを別の集約 `PendingFormSettings` で持つ。紐付け待ちの設定ごとに `formKey` から `pending` の `KeyBinding` を、`active` の設定ごとに `formId` から `bound` の `KeyBinding` を作り、追加したフォームと削除したフォームのキーと `FormId` から作った `BindingTargetChange` とともに、`KeyBinding.apply`（[common.md](common.md) の「キーで宣言した設定の紐付け」）に渡す。`removed` の設定は紐付けを持たない。反映の後に `active` である設定は、結果が `bound` の紐付け待ちの設定と、結果が `unchanged` の `active` の設定。

`planSchemaChange` は、`added` と `removed` から `BindingTargetChange` を作り、紐付けの結果を次のとおり書き込みにする。紐付けで行うことは、紐付け待ちの設定を削除して `active` の設定を作ること、外れた（`detached`）ときに行うことは、設定を削除済み（`removed`）にすること。

| 対象 | `KeyBinding.apply` の結果 | 結果 |
| --- | --- | --- |
| 紐付け待ちの設定 | `bound`（`added` のフォームの `FormId`） | `bound` に、その設定と `version`、`bind(pending, formId, form, now)` の結果の組を入れる |
| `active` の設定 | `detached` | `removed` に、`remove(settings, form, now)` の結果と、元の設定の `version` の組を入れる |
| どちらも | `unchanged` | 何も入れない |
| `added` のフォームのうち、どの紐付け待ちの設定も `bound` にならなかったもの | — | `created` に、`createForNewForm({ formId, form }, now)` の結果を入れる |

- 三つの列は、`added` と `removed` の順を保つ
- 追加も削除も無い反映では、三つの列がすべて空になる

#### 不変条件

- `reception` が `open` の設定の `recipients` に、`missingForReception` は `enabledRecipient` を返さない
- `removed` は、受付状態、通知定義、保持方針を持たない。変更できるのは通知先だけ
- `active` と `removed` の `formId` は変わらない。キーと `FormId` の対応は、[index.md](index.md) の「モデルとフォームの ID」に従う
- 一つのワークスペースの区画で、同じ `formId` の設定は高々一つ

#### ライフサイクル

```text
紐付け待ち ─ bind（スキーマの反映がそのキーのフォームを作成） → active
スキーマの反映がフォームを作成（紐付け待ちなし） → createForNewForm → active
active ─ remove（スキーマの反映がフォームを削除） → removed
```

- `removed` は、そのフォームの回答が残る間保持する。回答がすべて無くなった `removed`（`FormInventory.disposable` が返す設定）は削除する
- ワークスペースの利用終了で、すべての設定を削除する

### PendingFormSettings

紐付け待ちの設定。初期構成でフォームのキーを指定して宣言した運用設定で、そのキーのフォームが作成されるまで残り、紐付けで `FormSettings` になる。ワークスペースの区画の中で `formKey` が識別する。

```ts
type PendingFormSettings = Readonly<{
  formKey: FormKey;
  reception: Reception;
  recipients: RecipientList;
  notification: NotificationTemplate | null;
  retention: RetentionPolicy;
  version: number;
  createdAt: Date;
}>;

const PendingFormSettings: {
  declare(params: Readonly<{ formKey: FormKey; settings: ValidFormSettings }>, now: Date): PendingFormSettings;
};
```

| フィールド | 制約 |
| --- | --- |
| `reception` | 初期構成の宣言の値 |
| `notification` | `null` は、通知定義を宣言していないことを表す |
| `retention` | 宣言の無い場合は `RetentionPolicy.initial()` |
| `version` | 作成時は 0。作成の後に変わらない |

- `declare` は、初期構成の宣言から紐付け待ちの設定を作る。`settings` は、`FormSettingsInput.validate` が返した値（`ValidFormSettings`）。通知先、受付状態、通知定義をそのまま持ち、`retention` の `null` は `RetentionPolicy.initial()` にする。宣言の規則（通知先の列、受付を有効にする条件）は `FormSettingsInput.validate` だけが確かめる
- 変更の振る舞いを持たない。管理画面は、その内容を示すだけ
- 一つのワークスペースの区画で、同じ `formKey` の紐付け待ちの設定は高々一つ

```text
初期構成の読み込み → declare → 紐付け待ち ─ 紐付け（FormSettings.bind） → （削除。active の FormSettings になる）
```

- 紐付け待ちの設定は、そのキーのフォームが作成されるまで残る。紐付けの規則は、[common.md](common.md) の「キーで宣言した設定の紐付け」に従う
- ワークスペースの利用終了で、すべての紐付け待ちの設定を削除する

### MailConnection

ワークスペースに一つの送信接続設定。区画に一つの集約（[common.md](common.md) の「集約の ID」）。ワークスペースの全フォームが使う。登録した後は更新だけを行い、ワークスペースの利用終了まで削除しない。接続先、認証情報、接続条件を持つ。送信の方式は `MailSender` のアダプターが決め、ドメインは方式を知らない。

```ts
type MailConnection = Readonly<{
  endpoint: MailEndpoint;
  credentials: ProtectedMailCredentials;
  version: number;
  createdAt: Date;
  updatedAt: Date;
}>;

type ProtectedMailCredentials = Readonly<{
  identity: ProtectedMailSecret | null;
  secret: ProtectedMailSecret;
}>;
```

| フィールド | 制約 |
| --- | --- |
| `endpoint` | 接続先と接続条件。必須。送信者のアドレスは、接続条件の一つ（「MailEndpoint と ProtectedMailSecret」） |
| `credentials` | 認証情報。必須。`identity` は利用者を識別する値、`secret` は秘密で、どちらも `SecretCipher` が保護した形だけを持つ。`identity` の `null` は、秘密だけで認証する送信の方式で、この値を持たないことを表す |
| `version` | 楽観ロックの番号。作成時は 0 |

#### 振る舞い

```ts
const MailConnection: {
  register(
    current: MailConnection | null,
    params: Readonly<{ setting: MailSetting; credentials: ProtectedMailCredentials }>,
    now: Date,
  ): MailConnection;
  update(
    connection: MailConnection,
    params: Readonly<{ setting: MailSetting; credentials: ProtectedMailCredentials | null }>,
    now: Date,
  ): Change<MailConnection>;
  testMail(connection: MailConnection, text: Readonly<{ subject: string; body: string }>): Confidential<OutgoingMail>;
  view(connection: MailConnection): MailConnectionView;
};

type MailConnectionView = Readonly<{
  endpoint: MailEndpoint;
  credentialsRegistered: true;
}>;
```

| 振る舞い | 処理 |
| --- | --- |
| `register` | `setting` の接続先と接続条件、`credentials` で送信接続設定を作る。`current` は、そのワークスペースの登録済みの送信接続設定。`null` でなければ `BusinessRuleError`（`MailConnectionAlreadyRegistered`） |
| `update` | 接続先と接続条件を `setting` の値に置き換える。`credentials` が `null` なら登録済みの認証情報を保ち、`null` でなければ置き換える。`credentials` が `null` で、接続先と接続条件が現在と等しければ `unchanged`。`credentials` が `null` でない更新は `changed` |
| `testMail` | 動作確認の試験メールを作る。送信者の表示名と返信先は `null`。件名と本文は `text` の文面で、回答の入力内容を含まない |
| `view` | 認証情報を含まない表示用の値を返す。認証情報は、登録済みであることだけを示す |

- `setting` は、`MailConnectionInput.validate` が作る `MailSetting`。送信の方式で使えることを確かめた値で、利用者の入力の違反は `validate` が返す
- `register` が返す値と、`update` が返した `Change` の値（`value` または `current`）は、保存する前に `MailSender.send` へ渡せる。動作確認は、入力中の設定から作ったこの値と `testMail` で行う。動作確認の結果は、保存の条件にしない

#### 不変条件

- 接続先、接続条件（送信者のアドレスを含む）、認証情報は、常にすべてそろう。接続先と接続条件は、認証情報の `identity` の有無とともに、送信の方式で使えることを確かめた `MailSetting` の値
- 認証情報は、保護した形（`ProtectedMailSecret`）だけを持つ。振る舞いが返す値は、認証情報の値を含まない
- 更新は、保存済みの回答、受付時の通知定義、保持方針を変えない

#### ライフサイクル

```text
register → 登録済み ─ update → 登録済み
```

ワークスペースの利用終了で消去する。

### Submission

回答一件。受付時に確定した内容の写しと、保持方針を持つ。写しは、フォームと通知定義の変更、受付停止、フォームの削除、スキーマの移行、送信接続設定の更新で変わらない。通知の進行は、別の集約の `SubmissionNotification` が持つ。

```ts
type Submission = Readonly<{
  id: SubmissionId;
  formId: FormId;
  receivedAt: Date;
  form: ReceivedForm;
  content: Confidential<SubmissionContent>;
  template: NotificationTemplate;
  notifiedTo: readonly EmailAddress[];
  retention: RetentionPolicy;
}>;
```

| フィールド | 制約 |
| --- | --- |
| `id` | 必須。受付のユースケースが `IdGenerator` で作る |
| `formId` | 受付時のフォーム。削除済みフォームの回答も同じ値を保つ |
| `receivedAt` | 受付日時 |
| `form` | 受付時のフォーム（「ReceivedForm」）。受付時のフォームの定義から写した、項目、選択肢、表示名称、順序を含む |
| `content` | 入力内容と、問い合わせ対象の送信時の情報 |
| `template` | 受付時の通知定義 |
| `notifiedTo` | 受付時の通知対象。受付時に有効だったすべての通知先。一件以上 |
| `retention` | 回答に適用する保持方針。受付時は、受付時のフォームの保持方針 |

回答は楽観ロックの番号（`version`）を持たない。作成の後に行う書き込みは、保持方針の一括の置き換えと削除だけで、どちらも `version` を確かめない（`SubmissionRepository`）。

#### 振る舞い

```ts
const Submission: {
  receive(
    params: Readonly<{
      id: SubmissionId;
      formKey: string;
      target: Readonly<{ formId: FormId; form: FormDefinition; settings: ActiveFormSettings }> | null;
      input: SubmissionInput;
      conditionKeys: readonly string[];
      inquiryFacts: InquiryFacts;
    }>,
    now: Date,
  ): ReceptionDecision;

  applyRetention(submission: Submission, retention: RetentionPolicy): Submission;
  expiresAt(submission: Submission): Date;
  outgoingMail(submission: Submission): Confidential<OutgoingMail>;
};

type AcceptedSubmission = Readonly<{
  submission: Submission;
  notification: SubmissionNotification<Waiting>;
}>;

type ReceptionDecision =
  | (Readonly<{ accepted: true }> & WithEventDrafts<AcceptedSubmission, SubmissionAcceptedEvent>)
  | Readonly<{ accepted: false; rejection: JudgedRejection }>;
```

| 振る舞い | 処理 |
| --- | --- |
| `receive` | 送信を検証し、受け付ける場合は、回答と、その通知（`SubmissionNotification.begin`）を `entity` に、`form.submission_accepted` を `eventDrafts` に持つ結果を返す。「受付」が定める |
| `applyRetention` | 保持方針を置き換える |
| `expiresAt` | `RetentionPolicy.expiresAt(submission.retention, submission.receivedAt)` を返す |
| `outgoingMail` | `NotificationRendering.render` で受付時の通知定義を描画したメールを返す |

#### 受付

フォームの送信の結果は、次の順に決まる。最初に当てはまった行が結果になる。結果の優先は、この表だけが定める。

| 順 | 条件 | 結果 | 決めるもの |
| --- | --- | --- | --- |
| 1 | `TokenAuthorization.resolve` が拒否する | Token が定めるエラー。フォームの有無を示さない | Token の `TokenAuthorization.resolve` |
| 2 | 現行スキーマに、送信先のキーのフォームが無い | 受付失敗（`formNotFound`） | `receive`（`target` が `null`） |
| 3 | `TokenAuthorization.authorize` が `allowed` を返さない | Token が定めるエラー | Token の `TokenAuthorization.authorize` |
| 4 | フォームの送信制限に達している | `RateLimitedError`（`SUBMISSIONS_LIMITED`） | `SubmissionRatePolicy` と `UsageLimiter` |
| 5 | フォームが受付停止 | 受付失敗（`closed`） | `receive` |
| 6 | 入力不備 | 受付失敗（`invalid`） | `receive` |
| 7 | 上のどれでもない | 受付完了 | `receive` |

- フォームの送信制限に数えるのは、順 4 に届いた送信。順 5 と 6 の受付失敗になった送信と、受付完了の確定に失敗した送信も数える。順 1 から 3 で終えた送信は数えない
- 順 2 を順 3 より先に置くので、現行スキーマに無いキーへの送信は、トークンの範囲によらず `formNotFound` になる

`receive` は次の順に確かめ、最初に当てはまった理由で拒否する。

| 順 | 条件 | 拒否の理由 |
| --- | --- | --- |
| 1 | `target` が `null` | `formNotFound`。`formKey` を添える |
| 2 | `FormSettings.isAccepting(target.settings)` が偽 | `closed` |
| 3 | `SubmissionValidation.validate(target.form, input, inquiryFacts)` が入力不備を返す、または `conditionKeys` が空でない | `invalid`。`problems` は、検証が返した問題に、`conditionKeys` のキーごとの `conditionKey` の問題を、渡された順に続けた列 |

受け付けた回答は次の値を持つ。

- `form` は `ReceivedForm.of(target.form)`、`template` は `target.settings.notification`、`retention` は `target.settings.retention`
- `notifiedTo` は `FormSettings.enabledRecipients(target.settings)`。受付中の設定の不変条件から、空にならない。回答の内容や問い合わせ対象に応じた振り分けを行わない
- `content` は、検証が返した `SubmissionContent` を包んだ値
- `receivedAt` は `now`

一件の送信は、一組のフォームの定義と通知定義で受け付ける。フォームの受付は、スキーマの反映の適用中も、現行の定義で続ける。

送信が指定できるものと受付条件は、[../scenario/app-integration.md](../scenario/app-integration.md) の APP-08 が定める。Form は、それを型と `conditionKeys` の拒否で表す。送信の入力（`SubmissionInput`）は項目の値だけを持ち、受付条件を指す場所を型が持たない。`conditionKeys` は、送信が項目の値のほかに持つ指定のキーの列で、空でない送信は、上の表の順 3 で拒否する。名前の語彙を持たず、項目の値のほかの指定は、どれも変更できない受付条件の指定として扱う。受け付けた回答は、`conditionKeys` の指定を何も持たない。送信の本文から `conditionKeys` を作る写し方は、[../usecases/app-api.md](../usecases/app-api.md) の「フォームの受付結果」が定める。

#### 不変条件

- `form`、`content`、`template`、`notifiedTo`、`receivedAt`、`formId` は、作成の後に変わらない。変わるのは `retention` だけ
- `notifiedTo` は一件以上
- 通知の成功・失敗・再試行は、回答を変えない。保持期限は変わらない
- 回答は、同じ `SubmissionId` の `SubmissionNotification` を一つ持つ

#### ライフサイクル

```text
receive → 保存済み ─ applyRetention → 保存済み
```

回答は、回答の削除権限を持つメンバーの削除、保持期限を過ぎた後の自動整理、ワークスペースの利用終了で削除する。回答とともに消えるものは、`SubmissionRepository` の「回答と通知の対応」が定める。

#### ドメインイベント

```ts
type SubmissionAcceptedEvent = DomainEventBase<
  "form.submission_accepted",
  Readonly<{ submissionId: SubmissionId }>
>;
```

`aggregateId` は `submissionId`。ペイロードは宛先を持たない。消費者は、回答と通知を読み、最初の通知の回（`initial`）を送る。

### SubmissionNotification

回答一件の通知の進行と、宛先ごとの送信の結果。`SubmissionId` で識別する。作成と削除は、`SubmissionRepository` の「回答と通知の対応」に従う。入力内容と通知定義を持たない。

```ts
type SubmissionNotification<S extends NotificationState = NotificationState> = Readonly<{
  submissionId: SubmissionId;
  formId: FormId;
  receivedAt: Date;
  notifiedTo: readonly EmailAddress[];
  attempt: NotificationAttempt;
  state: S;
  deliveries: readonly RecipientDelivery[];
  version: number;
}>;

type NotificationAttempt =
  | Readonly<{ kind: "initial" }>
  | Readonly<{ kind: "retry"; round: number }>;

type Waiting = Readonly<{ kind: "waiting"; queuedAt: Date }>;
type Sending = Readonly<{ kind: "sending"; startedAt: Date; recipients: readonly EmailAddress[] }>;
type Succeeded = Readonly<{ kind: "succeeded"; settledAt: Date }>;
type Failed = Readonly<{
  kind: "failed";
  settledAt: Date;
  cause: "delivery" | "noRecipients";
  handling: FailureHandling;
}>;
type NotificationState = Waiting | Sending | Succeeded | Failed;

type RecipientDelivery = Readonly<{
  address: EmailAddress;
  outcome:
    | Readonly<{ kind: "accepted"; at: Date }>
    | Readonly<{ kind: "failed"; at: Date; reason: DeliveryFailureReason }>;
}>;

type RecipientStatus = Readonly<{
  address: EmailAddress;
  status:
    | Readonly<{ kind: "sending" }>
    | Readonly<{ kind: "accepted"; at: Date }>
    | Readonly<{ kind: "failed"; at: Date; reason: DeliveryFailureReason }>;
}>;

type NotificationResult = "unsent" | "succeeded" | "failed";
```

| フィールド | 制約 |
| --- | --- |
| `submissionId` | 必須。集約の ID。回答の ID と同じ |
| `formId` / `receivedAt` / `notifiedTo` | 回答の、受付時のフォーム、受付日時、通知対象の写し。作成の後に変わらない。受付時のフォームのキーと表示名称は、回答（`Submission.form`）が持つ |
| `attempt` | 通知の回。`initial` は最初の通知。`retry` は手動再試行で、`round` は何回目の再試行かを表す 1 以上の整数。作成時は `initial` |
| `state` | その回の状態。下の表 |
| `deliveries` | 宛先ごとの、最新の送信の結果（受理を確認済み、または失敗）。一つのメールアドレスにつき一件。一度も結果を記録していない宛先は持たない。送信中の宛先は、`Sending` の `recipients` が持つ。送信中を含む宛先ごとの送信の状態は、`recipientStatuses` が求める |
| `version` | 楽観ロックの番号。作成時は 0。保存する変更ごとに 1 進む |

| `state.kind` | 状態 | 意味 |
| --- | --- | --- |
| `waiting` | 送信待ち | その回を始めるイベントを記録済みで、送信を始めていない |
| `sending` | 送信中 | 送信を始めた記録。その回に送る宛先（`recipients`。一件以上）を持つ |
| `succeeded` | 送信成功 | その回の宛先のすべてで、メール送信基盤の受理を確認した |
| `failed` | 送信失敗 | その回が送信成功にならなかった。対処の状態（[common.md](common.md) の `FailureHandling`）を持つ |

状態は回によらず同じ四つで、最初の通知と再試行の別は `attempt` だけが持つ。対処の状態は、`failed` の中にだけ持つ。未対処の規則は [common.md](common.md) の「FailureHandling」が定める。

二つの `NotificationAttempt` は、`kind` が同じで、`retry` では `round` も同じ場合に等しい。

#### 振る舞い

```ts
const SubmissionNotification: {
  begin(submission: Submission, now: Date): SubmissionNotification<Waiting>;

  start(
    notification: SubmissionNotification,
    attempt: NotificationAttempt,
    settings: RecipientSource,
    now: Date,
  ): NotificationStep;
  complete(
    notification: SubmissionNotification,
    attempt: NotificationAttempt,
    results: readonly RecipientResult[],
    now: Date,
  ): NotificationCompletion;
  abandon(
    notification: SubmissionNotification<Waiting | Sending>,
    settings: RecipientSource,
    now: Date,
  ): SubmissionNotification<Failed>;

  retryPlan(notification: SubmissionNotification, settings: RecipientSource): RetryPlan;
  requestRetry(
    notification: SubmissionNotification,
    confirmed: NotificationAttempt,
    settings: RecipientSource,
    now: Date,
  ): RetryDecision;
  requestRetries(candidates: readonly RetryCandidate[], now: Date): BulkRetryDecision;

  markHandled(
    notification: SubmissionNotification,
    confirmed: NotificationAttempt,
    by: MemberId,
    now: Date,
  ): HandlingDecision;
  markHandledAll(candidates: readonly HandlingCandidate[], by: MemberId, now: Date): BulkHandlingDecision;

  targets(notification: SubmissionNotification, settings: RecipientSource): readonly EmailAddress[];
  result(notification: SubmissionNotification): NotificationResult;
  recipientStatuses(notification: SubmissionNotification): readonly RecipientStatus[];

  stallMarginMs: 300_000;
  stallAfterMs: number;
  stalled(notification: SubmissionNotification, now: Date): "waiting" | "sending" | null;
};

type RecipientSource = FormSettings;

type NotificationStep =
  | Readonly<{ action: "send"; notification: SubmissionNotification<Sending>; recipients: readonly EmailAddress[] }>
  | Readonly<{ action: "record"; notification: SubmissionNotification<Failed> }>
  | Readonly<{ action: "ignore" }>;

type NotificationCompletion =
  | Readonly<{ recorded: true; notification: SubmissionNotification<Succeeded | Failed> }>
  | Readonly<{ recorded: false }>;

type RetryPlan = Readonly<{
  targets: readonly EmailAddress[];
  removed: readonly EmailAddress[];
  added: readonly EmailAddress[];
}>;

type RetryRefusal = "alreadyRequested" | "notFailed" | "noRecipients";

type RetryDecision =
  | (Readonly<{ requested: true }> & WithEventDrafts<SubmissionNotification<Waiting>, NotificationRetryRequestedEvent>)
  | Readonly<{ requested: false; reason: RetryRefusal }>;

type RetryCandidate =
  | Readonly<{ kind: "found"; notification: SubmissionNotification; confirmed: NotificationAttempt; settings: RecipientSource }>
  | Readonly<{ kind: "notFound"; submissionId: SubmissionId }>;

type RetryOutcome =
  | Readonly<{ submissionId: SubmissionId; outcome: "requested"; notification: SubmissionNotification<Waiting> }>
  | Readonly<{ submissionId: SubmissionId; outcome: "alreadyRequested"; notification: SubmissionNotification }>;

type BulkRetryDecision =
  | (Readonly<{ requested: true }> & WithEventDrafts<readonly RetryOutcome[], NotificationRetryRequestedEvent>)
  | Readonly<{
      requested: false;
      refusals: readonly Readonly<{ submissionId: SubmissionId; reason: Exclude<RetryRefusal, "alreadyRequested"> | "notFound" }>[];
    }>;

type HandlingCandidate =
  | Readonly<{ kind: "found"; notification: SubmissionNotification; confirmed: NotificationAttempt }>
  | Readonly<{ kind: "notFound"; submissionId: SubmissionId }>;

type HandlingDecision =
  | Change<SubmissionNotification<Failed>>
  | Readonly<{ kind: "refused"; reason: "notFailed" }>;

type BulkHandlingDecision =
  | Readonly<{ handled: true; notifications: readonly SubmissionNotification<Failed>[] }>
  | Readonly<{ handled: false; refusals: readonly Readonly<{ submissionId: SubmissionId; reason: "notFailed" | "notFound" }>[] }>;
```

| 振る舞い | 処理 |
| --- | --- |
| `begin` | 回答から、`attempt` が `initial`、`state` が `waiting`（`queuedAt` は `now`）、`deliveries` が空の通知を作る。`formId`、`receivedAt`、`notifiedTo` は回答の値 |
| `start` | その回の送信を始める。送る・送信失敗を記録する・何もしない、のどれかを決める。「通知の送信」が定める |
| `complete` | 始めた回の `MailSender` の結果を宛先ごとの結果（`deliveries`）に記録し、`succeeded` または `failed` にする。その回の `sending` でない通知には、結果を記録しない。「通知の送信」が定める |
| `abandon` | 成功を確認できない回を、`failed` にする |
| `retryPlan` | 手動再試行の宛先と、受付時の通知対象から外れた宛先・加わった宛先を返す |
| `requestRetry` | メンバーが宛先を確認した回（`confirmed`）の通知の手動再試行を決める。受け付ける場合は、次の回の `waiting` にした通知を `entity` に、`form.notification_retry_requested` を `eventDrafts` に持つ結果を返す。「手動再試行」が定める |
| `requestRetries` | 複数の通知の手動再試行を、全部受け付けるか、どれも受け付けないかで決める。「手動再試行」が定める |
| `markHandled` | メンバーが対処の記録に選んだときに確認した回（`confirmed`）の、一件の通知の対処の記録を決める。「対処の記録」が定める |
| `markHandledAll` | 複数の通知の対処の記録を、全部受け付けるか、どれも受け付けないかで決める。「対処の記録」が定める |
| `targets` | その回の宛先を返す。「通知の送信」が定める |
| `result` | 通知の送信結果を返す。「通知の送信結果」が定める |
| `recipientStatuses` | 宛先ごとの送信の状態を返す。「通知の送信結果」が定める |
| `stalled` | 進まないまま止まった通知かどうかを返す。「止まった通知」が定める |

状態を変える振る舞いは `version` を 1 進める。

#### 通知の送信

その回の宛先は `targets` が決める。`settings` は、その時点の、回答のフォームの運用設定（`RecipientSource`。`active` または `removed`）。回答が残るフォームは、`active` か `removed` の設定を必ず持つ。これは、`FormSettings` の「ライフサイクル」と、「利用終了の消去」の順が保つ。有効な通知先（下の表の `enabled`）は、`FormSettings.enabledRecipients(settings)` で、振る舞いが求める。

| `attempt.kind` | 宛先 |
| --- | --- |
| `initial`（最初の通知） | `notifiedTo` のうち、`enabled` に含まれるメールアドレス |
| `retry`（手動再試行） | `enabled` のうち、`deliveries` の結果が `accepted` でないメールアドレス |

- 受付後に通知先から外した宛先（無効にした、削除した、修正前のメールアドレス）は `enabled` に無いので、どの回の宛先にもならない
- 受付後に追加した通知先と、修正後のメールアドレスは、手動再試行の回の宛先になる
- 結果が `accepted` の宛先は、どの回の宛先にもならない

`start` は、始める回（引数の `attempt`）と通知の状態から、次のとおり決める。

| 通知の状態 | 結果 |
| --- | --- |
| 引数の `attempt` が `notification.attempt` と等しくない、または `state` が `succeeded` か `failed` | `ignore` |
| `state` が `waiting` で、`targets` が一件以上 | `send`。`state` を `sending` にし、`recipients` に `targets` の宛先を入れる。`deliveries` は変えない |
| `state` が `waiting` で、`targets` が空 | `record`。`state` を `failed`（`noRecipients`、`FailureHandling.unhandled()`）にする。`deliveries` は変えない |
| `state` が `sending` で、`stalled(notification, now)` が `sending` を返す | `record`。`abandon` を適用する |
| `state` が `sending` で、`stalled(notification, now)` が `sending` を返さない | `ignore`。送信中の回を、進めている処理に任せる |

UnitOfWork の分け方とメールの送信は、[../usecases/form.md](../usecases/form.md) の共通の「通知の送信」が、途中で止まったときに残る状態は、[../testcases/form/sendSubmissionNotification.md](../testcases/form/sendSubmissionNotification.md) が定める。ほかの文書は、この節の判定を、結果の名前（`send` / `record` / `ignore`、`recorded`）で指す。

`complete` は、始めた回（引数の `attempt`）と通知の状態から、次のとおり決める。

| 通知の状態 | 結果 |
| --- | --- |
| 引数の `attempt` が `notification.attempt` と等しくない、または `state` が `sending` でない | `recorded` が偽。通知を変えない。その回は、別の処理が先に終えている |
| 引数の `attempt` が `notification.attempt` と等しく、`state` が `sending` | `recorded` が真。下のとおり結果を記録した通知を返す |

結果の記録は、`state.recipients` の宛先ごとに、`results` の結果を `deliveries` に入れる。受理は `accepted`、失敗は `failed` と理由にする。`results` に無い宛先は `failed`（`unconfirmed`）にする。`recipients` に無い宛先の結果は使わない。その宛先の前の回の結果は、置き換える。`recipients` の宛先がすべて `accepted` なら `state` を `succeeded` に、一つでも `failed` なら `failed`（`delivery`、`FailureHandling.unhandled()`）にする。

判定から、次が成り立つ。

- 同じ回を並行して始めた処理は、楽観ロックで一方だけが `send` の結果の通知を確定する
- 結果が `accepted` の宛先は変わらない。イベントの再配送、CMS の停止・再起動で、成功を確認済みの宛先へ送らない（[../scenario/form-operation.md](../scenario/form-operation.md) の「共通の規則」）
- 送信失敗の通知を、CMS は自動で再試行しない。次に送るのは `requestRetry` を受け付けた回だけ

`abandon` は、`state` が `waiting` なら `targets` の宛先を、`sending` なら `state.recipients` の宛先を、`deliveries` で `failed`（`unconfirmed`）にする。`state` は `failed`（`FailureHandling.unhandled()`）にする。`cause` は、`failed` にした宛先が一件以上なら `delivery`、無ければ `noRecipients`。

#### 止まった通知

`stallAfterMs` は、`RecipientList.max` と `MailSendPolicy.recipientTimeoutMs` の積に、`stallMarginMs` を加えた値。一つの回の送信は、`MailSender.send` の契約から、宛先の数と `MailSendPolicy.recipientTimeoutMs` の積までに終わるので、送信の途中の回に `stalled` が `sending` を返すことは無い。通知先の上限と一通の上限のどちらを変えても、この関係は保たれる。

`stalled` は、`state` が `waiting` で `queuedAt` から `stallAfterMs` 以上が過ぎた通知に `waiting` を、`sending` で `startedAt` から `stallAfterMs` 以上が過ぎた通知に `sending` を返す。そのほかは `null`。

止まった通知を進めるのは、定期ジョブ「止まった通知の検出」（[index.md](index.md) の「定期ジョブ」）。進め方は、[../usecases/form.md](../usecases/form.md) の「止まった通知を検出する」が定める。

- `waiting` のまま止まった通知は、その回を始めるイベントが届いていない通知。イベントが配送されない、または隔離された場合と、設置の停止・再起動の後に当たる。停止中に送らなかった通知を起動の後に送る規則（[../scenario/form-operation.md](../scenario/form-operation.md) の「共通の規則」）は、この検出が満たす

#### 通知の送信結果

`result` は、`state` だけから求める。

| `state.kind` | 結果 |
| --- | --- |
| `waiting`、`sending` | `unsent`（未送信） |
| `succeeded` | `succeeded`（送信成功） |
| `failed` | `failed`（送信失敗） |

- 未送信は、その回の送信を終えていないことを表す。未送信の内訳は二軸で、`state.kind`（`waiting` は送信待ち、`sending` は送信中）と、`attempt.kind`（`initial` は最初の通知、`retry` は再試行）が示す。未送信での絞り込みは、内訳によらず、未送信の通知をすべて含む
- 送信成功は、その回に送信対象としたすべての宛先について、メール送信基盤の受理を確認したことを表す。受信箱への到着を表さない
- 送信失敗は、通知が届いていないことを表さない。一部の宛先だけの成功、成功を確認できない場合、有効な宛先が残らない場合（`noRecipients`）は送信失敗
- 再試行を始めた通知は `waiting` なので、未送信になる。送信結果での絞り込みと、フォームごとの件数（`summarizeByForm`）は、この結果で行う

`recipientStatuses` は、宛先ごとの送信の状態を、宛先ごとに一件返す。宛先ごとの送信の状態は、この振る舞いだけが求める。

| 宛先 | 状態 |
| --- | --- |
| `state` が `sending` の通知の、`state.recipients` の宛先 | `sending`（送信中）。その宛先の前の回の結果は示さない |
| 上に当たらない、`deliveries` に結果を持つ宛先 | `deliveries` の結果。`accepted`（受理を確認済み）、または `failed`（失敗）と理由 |

送信中でなく、結果も持たない宛先は含まない。並びは、メールアドレスの文字列の昇順。

#### 手動再試行

`retryPlan` は、`targets` に、`retry` の回の規則を適用した宛先を入れる。`enabled` は `FormSettings.enabledRecipients(settings)`。`removed` は `notifiedTo` にあって `enabled` に無いメールアドレス、`added` は `enabled` にあって `notifiedTo` に無いメールアドレス。

確認した回の規則は、[common.md](common.md) の「FailureHandling」が定める。Form の通知の回は `attempt` で、二つの回は `NotificationAttempt` の等しさで比べる。

`requestRetry` は、メンバーが宛先を確認した回（`confirmed`）と通知の状態から、次の順に確かめる。

| 順 | 条件 | 結果 |
| --- | --- | --- |
| 1 | `notification.attempt` が `confirmed` と等しくない | `alreadyRequested`。通知を変えず、イベントを発行しない |
| 2 | `state` が `failed` でない | `notFailed` |
| 3 | `retryPlan(...).targets` が空 | `noRecipients` |
| 4 | 上のどれでもない | `attempt` を次の `retry` の回にし、`state` を `waiting`（`queuedAt` は `now`）にして、`form.notification_retry_requested` を発行する |

次の `retry` の回は、`attempt` が `initial` なら `round` 1、`retry` なら `round` に 1 を加えた回。

`requestRetries` は、`found` の組ごとに `requestRetry` を適用し、次のとおり決める。`notFound` の組は、その `SubmissionId` の通知が無いことを表す。

| 条件 | 結果 |
| --- | --- |
| `notFound` の組、または `notFailed` か `noRecipients` の通知が一件でもある | `requested` が偽。該当する組と理由（`notFound` の組は `notFound`）を、`candidates` の順に `refusals` に入れる。どの通知も変えず、イベントを発行しない |
| 上に当たらない | `requested` が真。再試行を受け付けた通知は `requested` と次の回の通知、`alreadyRequested` の通知は `alreadyRequested` と渡された通知を、`candidates` の順に `entity` に入れる。`eventDrafts` は、受け付けた通知ごとに一件 |

- `alreadyRequested` の結果が持つ通知は、先に始まった再試行の今の状態と送信結果を示す
- 再試行は回答を複製しない。回答を変えない

#### 対処の記録

`markHandled` は、一件の通知を、次の順に決める。

| 条件 | 結果 |
| --- | --- |
| `notification.attempt` が `confirmed` と等しくない | `refused`。理由は `notFailed`。通知を変えない |
| `state` が `failed` でない | `refused`。理由は `notFailed`。通知を変えない |
| `state` が `failed` で、対処済み | `unchanged` |
| `state` が `failed` で、未対処 | `changed`。`handling` を `FailureHandling.markHandled(handling, by, now)` にした通知 |

`markHandledAll` は、`found` の組ごとに `markHandled` を適用し、結果を集める。`notFound` の組は、その `SubmissionId` の通知が無いことを表す。

| 条件 | 結果 |
| --- | --- |
| `notFound` の組、または `markHandled` が `notFailed` を返した通知が一件でもある | `handled` が偽。該当する組と理由（`notFound` の組は `notFound`、ほかは `notFailed`）を、`candidates` の順に `refusals` に入れる。どの通知も変えない |
| 上に当たらない | `handled` が真。`markHandled` が `changed` を返した通知だけを、渡された順に `notifications` に入れる |

#### 不変条件

- `deliveries` は、一つのメールアドレスにつき一件。結果が `accepted` の宛先は、以後変わらない
- 送信中の宛先は、`sending` の `state` だけが持つ。`sending` でない通知は、送信中の宛先を持たない
- `sending` の `recipients` は、一件以上で、互いに異なり、結果が `accepted` の宛先を含まない
- `attempt` は、`initial`、`retry` の `round` 1、2、… の順にだけ進む。進むのは、`requestRetry` が再試行を受け付けたときだけ

#### ライフサイクル

```text
begin → waiting ─ start → sending ─ complete → succeeded
                └ 宛先なし → failed             └ failed
                           sending ─ abandon → failed
         waiting ─ abandon → failed
failed ─ requestRetry → waiting（次の回） ─ start → sending → …
failed（未対処） ─ markHandled / markHandledAll → failed（対処済み）
```

回答の削除で消える（`SubmissionRepository` の「回答と通知の対応」）。通知だけを削除する操作は持たない。

#### ドメインイベント

```ts
type NotificationRetryRequestedEvent = DomainEventBase<
  "form.notification_retry_requested",
  Readonly<{ submissionId: SubmissionId; round: number }>
>;
```

`aggregateId` は `submissionId`。`round` は、始める `retry` の回。ペイロードは宛先を持たない。

## 値オブジェクト

制約を持つ値オブジェクト（`RetentionPolicy`、`NotificationTemplate`、`MailEndpoint`、`MailSetting`、`RecipientList`）は、ブランド型で表す。値を作る経路は、ドメインの関数（「MailConnectionInput と FormSettingsInput」の `validate`、各値オブジェクトの `initial` と `empty`、`RecipientList` の `check` と変更の関数）と、リポジトリのアダプターが保存値を戻すデコードの二つで、ほかの経路の値を型が受け付けない。保存する値は、どれもドメインの関数が作った値なので、デコードは保存値をそのまま戻し、制約と `MailSettingRule` を確かめ直さない。利用者の入力の制約は `validate` が確かめ、違反を項目ごとの理由で返す。理由のコードは「エラーコード」が定める。等価性は、断りが無い限り、すべてのフィールドの値の一致で決める。

### NotificationRecipient

```ts
type NotificationRecipient = Readonly<{ address: EmailAddress; enabled: boolean }>;
```

- `address` は共通の `EmailAddress`。形式だけを確かめ、宛先への到達を確かめない
- 通知先は、メールアドレスで識別する

### RecipientList

通知先の列と、その規則（メールアドレスの重複、件数の上限）。`FormSettings` と `PendingFormSettings` が持ち、通知先の変更と `FormSettingsInput.validate` が、同じこの規則を使う。

```ts
type RecipientList = readonly NotificationRecipient[] & { readonly [recipientListBrand]: true };

type RecipientListIssue =
  | Readonly<{ reason: "DuplicateRecipient"; index: number }>
  | Readonly<{ reason: "RecipientLimitReached" }>;

const RecipientList: {
  max: number;
  empty(): RecipientList;
  check(
    recipients: readonly NotificationRecipient[],
  ): Readonly<{ valid: true; list: RecipientList }> | Readonly<{ valid: false; issues: readonly RecipientListIssue[] }>;
  add(list: RecipientList, address: EmailAddress): RecipientList;
  changeAddress(list: RecipientList, from: EmailAddress, to: EmailAddress): Change<RecipientList>;
  setEnabled(list: RecipientList, address: EmailAddress, enabled: boolean): Change<RecipientList>;
  remove(list: RecipientList, address: EmailAddress): RecipientList;
  enabled(list: RecipientList): readonly EmailAddress[];
};
```

- 規則: メールアドレスは互いに異なる。件数は、有効と無効を合わせて `max` 件以下。並びは追加した順
- `max` の値は、[../scenario/app-integration.md](../scenario/app-integration.md) の APP-12 の「一つのフォームの通知先の件数」。`max` は、一つの回の送信に掛かる時間の上限を決める。`SubmissionNotification.stallAfterMs` は、この値から導く（「止まった通知」）
- `check` は、規則の違反をすべて返す。重複は、二件目以降の位置（`index`）ごとに `DuplicateRecipient`。`max` を超える列は `RecipientLimitReached`
- `add` は、有効な通知先を末尾に加える。同じメールアドレスがあれば `BusinessRuleError`（`DuplicateRecipient`）、既に `max` 件あれば `BusinessRuleError`（`RecipientLimitReached`）
- `changeAddress` は、`from` の通知先のメールアドレスを `to` にする。有効・無効と位置は保つ。次の順に確かめる。`from` が無ければ `NotFoundError`（対象の種類 `recipient`、キーは `from`。`to` と等しくても同じ）。`from` と `to` が等しければ `unchanged`（`DuplicateRecipient` にしない）。`to` が既にあれば `DuplicateRecipient`
- `setEnabled` は、通知先の有効・無効を変える。同じ値なら `unchanged`。通知先が無ければ `NotFoundError`（対象の種類 `recipient`、キーは `address`）
- `remove` は、通知先を取り除く。通知先が無ければ `NotFoundError`（対象の種類 `recipient`、キーは `address`）
- `enabled` は、有効な通知先のメールアドレスを、並びの順に返す

### RetentionPolicy

```ts
type RetentionPolicy = Readonly<{ days: number }> & { readonly [retentionPolicyBrand]: true };

const RetentionPolicy: {
  minDays: number;
  maxDays: number;
  initialDays: number;
  initial(): RetentionPolicy;
  expiresAt(policy: RetentionPolicy, receivedAt: Date): Date;
  isExpired(policy: RetentionPolicy, receivedAt: Date, now: Date): boolean;
};
```

- `days` は `minDays` 以上 `maxDays` 以下の整数。`RetentionInput.validate` が確かめる（`InvalidRetentionPolicy`）
- `initial` は `initialDays` 日
- `minDays`、`maxDays`、`initialDays` の値は、[../scenario/app-integration.md](../scenario/app-integration.md) の APP-12 の「回答の保持期間」の範囲と標準値
- `expiresAt` は、`receivedAt` に `days` × 86,400,000 ミリ秒を加えた時点
- `isExpired` は、`now` が `expiresAt` 以降のときに真
- コンテンツの履歴とごみ箱の保持方針（Content の `ContentRetentionPolicy`）と別に持つ

### TemplateText

通知定義の件名と本文。文章と参照の並びで持つ。

```ts
type InlineNode =
  | Readonly<{ type: "text"; text: string }>
  | Readonly<{ type: "field"; field: FieldKey }>
  | Readonly<{ type: "inquiry"; field: FieldKey; attribute: InquiryAttribute }>
  | Readonly<{ type: "receivedAt" }>;

type InquiryAttribute = "label" | "model" | "contentId" | "uniqueValue";

type IfAnsweredNode = Readonly<{ type: "ifAnswered"; field: FieldKey; children: readonly InlineNode[] }>;
type AllFieldsNode = Readonly<{ type: "allFields" }>;

type SubjectText = readonly (InlineNode | IfAnsweredNode)[];
type BodyText = readonly (InlineNode | IfAnsweredNode | AllFieldsNode)[];

const TemplateText: {
  parseSubject(source: string): TemplateParse<SubjectText>;
  parseBody(source: string): TemplateParse<BodyText>;
  format(text: SubjectText | BodyText): string;
};

type TemplateParse<T> =
  | Readonly<{ parsed: true; text: T }>
  | Readonly<{ parsed: false; errors: readonly TemplateSyntaxError[] }>;

type TemplateSyntaxError = Readonly<{
  position: number;
  reason: "unknownTag" | "invalidKey" | "unclosedTag" | "unclosedBlock" | "unexpectedBlockEnd" | "nestedBlock" | "notAllowedHere" | "empty";
}>;
```

| ノード | 記法 | 描画 |
| --- | --- | --- |
| `text` | そのままの文章 | その文章 |
| `field` | `{{field:キー}}` | 回答の項目の値 |
| `inquiry` | `{{inquiry:キー.label}}`、`.model`、`.contentId`、`.uniqueValue` | 問い合わせ対象の送信時の情報。表示名称、モデルのキー、コンテンツID、一意な値 |
| `receivedAt` | `{{receivedAt}}` | 受付日時 |
| `allFields` | `{{allFields}}` | 回答の全項目の一覧 |
| `ifAnswered` | `{{#answered:キー}}` から `{{/answered}}` まで | 項目が未入力でない場合に、囲んだ部分を出す |

- キーは `FieldKey` の形式。形式に合わなければ `invalidKey`
- 文章の中の `\{{` は `{{` を、`\\` は `\` を表す。ほかの `\` はそのままの文字とする
- `{{` と `}}` の内側の前後の空白は無視する
- `ifAnswered` は入れ子にできない（`nestedBlock`）。`allFields` は、件名と `ifAnswered` の中に置けない（`notAllowedHere`）
- 件名の文章は改行を持てない（`notAllowedHere`）
- ノードが無い、または空白だけの文章のノードだけからなる件名・本文は `empty`
- 隣り合う文章のノードは一つにまとめ、空の文章のノードを持たない
- `format` は記法の文字列を返す。文章の `\` を `\\` に、`{{` を `\{{` にする。`format` の結果を解析すると、元の並びと等しい値になる
- 件名と本文を作るノードは、この直和型の六つ

### NotificationTemplate

```ts
type NotificationTemplate = Readonly<{
  subject: SubjectText;
  body: BodyText;
  senderName: string | null;
  replyTo: FieldKey | null;
}> & { readonly [notificationTemplateBrand]: true };

const NotificationTemplate: {
  initial(form: FormDefinition): NotificationTemplate;
  referenceIssues(template: NotificationTemplate, form: ReceivedForm): readonly ReferenceIssue[];
  usagesOf(field: ReceivedField): readonly ReferenceUsage[];
};

type ReferenceIssue = Readonly<{
  location: "subject" | "body" | "replyTo";
  usage: ReferenceUsage;
  field: FieldKey;
  reason: ReferenceIssueReason;
}>;

type ReferenceIssueReason = "fieldRemoved" | "kindUnsuitable";

type ReferenceUsage = "field" | "inquiry" | "ifAnswered" | "replyTo";
```

- `senderName` は送信者の表示名。前後の空白を除いて 1 文字以上 100 文字以下で、改行を持たない。`null` は表示名を付けないことを表す。`NotificationTemplateInput.validate` が確かめる（`InvalidSenderName`）
- `replyTo` は、返信先に使う回答の項目のキー。`null` は返信先を指定しないことを表す
- `initial` は、新規フォームの通知定義を返す。件名はフォームの表示名称（無ければキー）の文章、本文は `allFields`、空行、`receivedAt` の並び、`senderName` と `replyTo` は `null`

`usagesOf` は、項目が合う用途を、下の表の順に返す。通知定義の編集で、参照に選べる項目と用途を示すために使う。`referenceIssues` は、通知定義のすべての参照を `form` に照らし、不整合を、件名、本文、返信先の順に返す。項目の値の種類が用途に合うかは、`usagesOf` で決める。空は不整合が無いことを表す。通知定義の編集時の検証、スキーマの反映で生じる不整合の検出、通知の描画は、この一つの規則を使う。現行のフォームの定義に照らす場合は、`ReceivedForm.of` で写した値を渡す。

| 用途 | 用途に合う項目 |
| --- | --- |
| `field` | `contentReference` を除く、すべての値の種類の項目 |
| `inquiry` | `contentReference` の項目 |
| `ifAnswered` | すべての値の種類の項目 |
| `replyTo` | `text` で、`email` が真の項目 |

- 参照の不整合の理由の直和（`ReferenceIssueReason`）は、この節だけが定める。理由は次の二つ
- 参照するキーの項目が `form` に無ければ、理由は `fieldRemoved`
- 項目の値の種類が用途に合わなければ、理由は `kindUnsuitable`
- 項目のキーを変えた反映は削除と追加なので、変更前のキーへの参照は `fieldRemoved` になる
- `allFields` と `receivedAt` は項目を参照しないので、不整合にならない

### ReceivedForm

回答が持つ受付時のフォーム。回答の表示、通知の描画、一覧表示用の項目の値が使う分だけを、受付時のフォームの定義から写して持つ。

```ts
type ReceivedForm = Readonly<{
  key: FormKey;
  label: string | null;
  fields: readonly ReceivedField[];
}>;

type ReceivedField = Readonly<{ key: FieldKey; label: string | null }> &
  (
    | Readonly<{ kind: "text"; email: boolean }>
    | Readonly<{ kind: "number" | "boolean" | "date" | "datetime" | "contentReference" }>
    | Readonly<{ kind: "select" | "multiSelect"; options: readonly Readonly<{ key: OptionKey; label: string | null }>[] }>
  );

const ReceivedForm: {
  of(form: FormDefinition): ReceivedForm;
};
```

- `of` は、フォームの定義のキー、表示名称と、項目ごとのキー、表示名称、値の種類、選択肢のキーと表示名称を、定義の順に写す。`email` は、テキストの項目の `constraints.format` が `email` のときに真
- `FormDefinition` から `ReceivedForm` を作るのは `of` だけ。Schema のフォームの定義の型が変わっても、変わるのは `of` の写し方だけで、保存した回答の `form` の型は変わらない
- 等価性は、すべてのフィールドの値の一致で決める

### SubmissionContent

```ts
type FormLeaf = Readonly<{ richText: never; media: never }>;
type FormFieldValue = Extract<
  FieldValue<FormLeaf>,
  { kind: "text" | "number" | "boolean" | "date" | "datetime" | "select" | "multiSelect" | "reference" }
>;
type FormValues = Readonly<Record<string, FormFieldValue>>;

type InquiryTargetSnapshot = Readonly<{
  model: ModelKey;
  contentId: ContentId;
  label: string | null;
  uniqueValues: readonly Readonly<{ field: FieldKey; value: string }>[];
}>;

type SubmissionContent = Readonly<{
  values: FormValues;
  inquiryTargets: Readonly<Record<string, InquiryTargetSnapshot>>;
}>;
```

- `FormValues` は、項目のキーから値への対応。未入力の項目はキーを持たない。値の形は Schema の `FieldValue` に従い、`contentReference` の項目の値は `reference` で表す
- `inquiryTargets` は、問い合わせ対象の項目のキーから、送信時の情報への対応。`values` に値を持つ `contentReference` の項目のキーと過不足なく一致する
- `InquiryTargetSnapshot` は、受付時の事実（`InquiryTargetFact` の `published`）の `model`、`label`、`uniqueValues` と、そのコンテンツID
- 通知と回答の表示は、この送信時の情報を使う。問い合わせ対象の更新、公開取り消し、削除で変わらない。回答は、問い合わせ対象のコンテンツの削除を妨げない

### SubmissionInput と InquiryFacts

```ts
type SubmissionInput = Readonly<Record<string, unknown>>;

type InquiryTargetFact =
  | Readonly<{
      status: "published";
      model: ModelKey;
      label: string | null;
      uniqueValues: readonly Readonly<{ field: FieldKey; value: string }>[];
    }>
  | Readonly<{ status: "unavailable" }>;

type InquiryFacts = Readonly<Record<string, InquiryTargetFact>>;
```

- `SubmissionInput` は、送信者が指定した、項目のキーから値への対応。値が項目の値の種類に合うかは、`SubmissionValidation.validate` が確かめ、合わない値は入力不備（`invalid`）になる
- `InquiryFacts` は、コンテンツID から、そのコンテンツの受付時の事実への対応。`published` は受付時に公開中のコンテンツの事実、ほかは `unavailable`。ユースケースが Content から得て渡す（[../usecases/form.md](../usecases/form.md) の「フォームの送信を受け付ける」）。対応に無いコンテンツID も `unavailable` として扱う

### ReceptionResult

```ts
type ReceptionResult<R extends ReceptionRejection = ReceptionRejection> =
  | Readonly<{ outcome: "accepted" }>
  | Readonly<{ outcome: "rejected"; rejection: R }>;

type ReceptionRejection =
  | Readonly<{ reason: "formNotFound"; formKey: string }>
  | Readonly<{ reason: "closed" }>
  | Readonly<{ reason: "invalid"; problems: SubmissionProblems }>
  | Readonly<{ reason: "rateLimited"; retryAt: Date }>
  | Readonly<{ reason: "busy" }>
  | Readonly<{ reason: "storageFailed" }>;

type JudgedRejection = Extract<ReceptionRejection, { reason: "formNotFound" | "closed" | "invalid" }>;

type SubmissionProblem =
  | Readonly<{ kind: "field"; issue: SubmissionIssue }>
  | Readonly<{ kind: "undefinedKey"; key: string }>
  | Readonly<{ kind: "conditionKey"; key: string }>;

type SubmissionProblems<P extends SubmissionProblem = SubmissionProblem> = readonly [P, ...P[]];

type ReceptionResponse = Readonly<{
  result: ReceptionResult<JudgedRejection>;
  fingerprints: readonly FingerprintMatch[];
}>;
```

`ReceptionResult` は、フォームの送信の受付結果で、受付失敗の理由は `ReceptionRejection` の六つ。受付失敗の理由の定義は、この節だけが持つ。`JudgedRejection` の三つは `Submission.receive` が判定する。残りの三つは、フォームの送信を受ける入口が、「受付失敗への写し方」に従って、エラーから作る。

フォームの送信は、前提とするスキーマの識別子（Schema の `ClaimedFingerprints`）を添えられる。`ReceptionResponse` の `fingerprints` は、Schema の `WorkspaceSchema.matchFingerprints` の結果。

| 受付結果 | 意味 | 決めるもの |
| --- | --- | --- |
| `accepted` | 入力検証を終え、回答の保存、その通知の作成、通知を始めるイベントの記録が確定した | `Submission.receive` |
| `formNotFound` | 現行スキーマに、そのキーのフォームが無い | `Submission.receive`。トークンの認可との順は、「受付」の結果の優先の表が定める |
| `closed` | フォームが受付停止 | `Submission.receive` |
| `invalid` | 入力不備。問題（`SubmissionProblem`）を一件以上持つ。問題は、項目ごとの理由（`field`）、定義されていない項目のキー（`undefinedKey`）、変更できない受付条件として送られた指定のキー（`conditionKey`）のいずれか | `Submission.receive` |
| `rateLimited` | フォームの送信制限（`SubmissionRatePolicy`）か、トークンの利用量の制限に達した。`retryAt` は、再び試せる時刻 | 「受付失敗への写し方」 |
| `busy` | 確定の時点の競合で、受付が確定しなかった。回答も通知も残らない | 「受付失敗への写し方」 |
| `storageFailed` | 保存に失敗し、受付が確定しなかった。回答も通知も残らない | 「受付失敗への写し方」 |

- 受付結果は、通知の送信結果を待たずに返す。通知の送信失敗は受付失敗にしない
- 受付失敗は、通知先と送信接続設定の内容を持たない
- 同じ内容の送信を重複として統合しない。送信ごとに新しい回答を作る

#### 受付失敗への写し方

フォームの送信を受ける入口は、示されたトークンを確かめる段と受付のユースケースが投げたエラーのうち、次の表に当たるものを受付失敗に写す。写し方は、入口によらず、この表だけが定める。表に当たらないエラーは写さず、入口がエラーとして返す。

| エラー | 受付失敗の理由 |
| --- | --- |
| `RateLimitedError` | `rateLimited`。`retryAt` は、エラーの再び試せる時刻 |
| `ConflictError` のうち、[index.md](index.md) の「競合の扱い」で、要求の入口の「ほかの理由」に当たるもの | `busy` |
| 受付のユースケースが投げた `SystemError` | `storageFailed` |

### SubmissionRatePolicy

```ts
const SubmissionRatePolicy: {
  limit: UsageLimit;
  key(workspaceId: WorkspaceId, formId: FormId): string;
};
```

- 制限の単位はフォーム（`key`）、上限は `limit`。期間と制限の結果は、[common.md](common.md) の「UsageLimiter」に従う。`limit` の値は、[../scenario/app-integration.md](../scenario/app-integration.md) の APP-12 の「フォームの送信制限」
- `key` は、`form-submission:` に `WorkspaceId`、`:`、`FormId` を続けた文字列。`FormId` の一意性は区画の中に限り、`UsageLimiter` は区画に結び付かないので、ワークスペースの ID を含める
- 数える送信と、ほかの結果との優先は、「受付」の結果の優先の表が定める
- トークンごとの利用量の制限は、この制限と別に、Token ドメインが定める

### SubmissionBatch

```ts
const SubmissionBatch: {
  maxTargets: number;
};
```

- 一度の操作で扱う回答・通知の数の上限。値は [../scenario/app-integration.md](../scenario/app-integration.md) の APP-12 の「一つの操作が対象にできる回答、通知、通知と宛先の組の件数」

### FormSweep

```ts
const FormSweep: {
  maxLimit: 1000;
};
```

- `FormSweep.maxLimit` は、件数を区切る読み取りと置き換えの `limit` の上限。件数を区切る削除の `limit` は、[common.md](common.md) の `BoundedDeletion.maxLimit`
- `SubmissionNotificationRepository.findByIds` が一度に受ける ID の数の上限は、`SubmissionBatch.maxTargets` と、[common.md](common.md) の `Pagination.maxLimit` の大きいほう。一度の操作の対象の通知も、一覧の一ページの回答の通知も、一回の呼び出しで読める。二つの値のどちらを変えても、この関係は保たれる

### DeliveryFailureReason

```ts
type DeliveryFailureReason = "connection" | "authentication" | "recipientRejected" | "rejected" | "unconfirmed";
```

| 値 | 意味 |
| --- | --- |
| `connection` | メール送信基盤へ接続できない、または、メールを渡していないことを確かめられた失敗。メールは渡っていない |
| `authentication` | メール送信基盤が認証を拒否した |
| `recipientRejected` | メール送信基盤が宛先を拒否した |
| `rejected` | メール送信基盤が、ほかの理由でメールを受理しなかった |
| `unconfirmed` | 送信を試みたが、受理を確認できない。届いている場合がある |

文章を持たない。メール送信基盤の応答の文面と、メールの内容を持つ場所が無い。

### OutgoingMail と RecipientResult

```ts
type OutgoingMail = Readonly<{
  senderName: string | null;
  replyTo: EmailAddress | null;
  subject: string;
  body: string;
}>;

type RecipientResult = Readonly<{
  address: EmailAddress;
  outcome: Readonly<{ status: "accepted" }> | Readonly<{ status: "failed"; reason: DeliveryFailureReason }>;
}>;
```

`senderName` は送信者の表示名。`subject` は改行を持たない文字列で、空文字列であり得る（描画した参照の値が空の件名）。`body` はプレーンテキスト。送信者のアドレスは、`MailSender` のアダプターが、送信接続設定の接続条件から得る（「MailSender」）。

### MailSendPolicy

```ts
const MailSendPolicy: {
  recipientTimeoutMs: 30_000;
};
```

- `recipientTimeoutMs` は、一通のメールの送信を待つ時間の上限。`MailSender.send` の契約と、`SubmissionNotification.stallAfterMs` が使う

### MailEndpoint と ProtectedMailSecret

```ts
type MailEndpoint = Readonly<{
  destination: string;
  conditions: Readonly<Record<string, string>>;
}> & { readonly [mailEndpointBrand]: true };

type MailSetting = Readonly<{
  endpoint: MailEndpoint;
}> & { readonly [mailSettingBrand]: true };

type ProtectedMailSecret = ProtectedSecret & { readonly [protectedMailSecretBrand]: true };

const ProtectedMailSecret: {
  create(secret: ProtectedSecret): ProtectedMailSecret;
};
```

- `destination` は接続先。前後の空白を除いて 1 文字以上 2,048 文字以下で、空白と制御文字を含まない
- `conditions` は接続条件。名前と値の組で、名前と値の意味は `MailSender` のアダプターが解釈する。名前は 1 文字以上 64 文字以下、値は 1 文字以上 1,024 文字以下、組は 1 個以上 32 個以下。値は空文字列でない
- 送信者のアドレスは、接続条件の一つ。その名前は `MailSender` のアダプターが決め、有無と形式は、アダプターの規則（`MailSettingRule`）が確かめる
- `MailSetting` は、接続先と接続条件のうち、認証情報の `identity` の有無とともに、アダプターの規則が問題を返さなかったもの。ドメインの関数のうち、作るのは `MailConnectionInput.validate` だけ（作る経路は「値オブジェクト」の冒頭）
- ドメインは、接続先と接続条件の中身を解釈しない
- 等価性は、`destination` の一致と、`conditions` の名前と値の組の一致（順序によらない）
- `ProtectedMailSecret` は、[common.md](common.md) の `ProtectedSecret` を包むブランド型。認証情報の `identity` と `secret` のそれぞれを持つ。`SecretCipher.protect` が返した値と、リポジトリのアダプターが戻す保存値からだけ作る。保護していない値は、型が受け付けない

### MailConnectionInput と FormSettingsInput

管理画面での設定と、初期構成の宣言が使う、入力の型と検証。`validate` はポートに依存しない純粋な関数で、秘密情報を保護する前に呼べる。利用者の入力の違反は、ここが項目ごとの理由として返し、ユースケースが `ValidationError` にする。

```ts
type MailConnectionInput = Readonly<{
  destination: string;
  conditions: Readonly<Record<string, string>>;
  credentials: MailCredentialsInput | null;
}>;

type MailCredentialsInput = Readonly<{ identity: string | null; secret: string }>;

type MailConnectionMode =
  | Readonly<{ kind: "register" }>
  | Readonly<{ kind: "update"; current: MailConnection }>;

type ValidMailConnection<M extends MailConnectionMode> = Readonly<{
  setting: MailSetting;
  credentials: M extends Readonly<{ kind: "register" }>
    ? Confidential<MailCredentialsInput>
    : Confidential<MailCredentialsInput> | null;
}>;

type NotificationTemplateInput = Readonly<{
  subject: string;
  body: string;
  senderName: string | null;
  replyTo: string | null;
}>;

type FormSettingsInput = Readonly<{
  reception: Reception;
  recipients: readonly Readonly<{ address: string; enabled: boolean }>[];
  notification: NotificationTemplateInput | null;
  retentionDays: number | null;
}>;

type ValidFormSettings = Readonly<{
  reception: Reception;
  recipients: RecipientList;
  notification: NotificationTemplate | null;
  retention: RetentionPolicy | null;
}> & { readonly [validFormSettingsBrand]: true };

type FormInputIssue<R extends string = FormInputReason> = Readonly<{ path: string }> &
  (
    | Readonly<{ reason: Extract<R, "TemplateSyntax">; syntax: readonly TemplateSyntaxError[] }>
    | Readonly<{ reason: Extract<R, "MailSettingRejected">; setting: MailSettingIssue["reason"] }>
    | Readonly<{ reason: Exclude<R, "TemplateSyntax" | "MailSettingRejected"> }>
  );

type InputValidation<T, R extends string = FormInputReason> =
  | Readonly<{ valid: true; value: T }>
  | Readonly<{ valid: false; issues: readonly FormInputIssue<R>[] }>;

const MailConnectionInput: {
  validate<M extends MailConnectionMode>(
    input: MailConnectionInput,
    mode: M,
    rule: MailSettingRule,
  ): InputValidation<ValidMailConnection<M>>;
};

const NotificationTemplateInput: {
  validate(input: NotificationTemplateInput): InputValidation<NotificationTemplate>;
};

const RecipientAddressInput: {
  validate(address: string): InputValidation<EmailAddress>;
};

const RetentionInput: {
  validate(days: number): InputValidation<RetentionPolicy>;
};

const FormSettingsInput: {
  validate(
    input: FormSettingsInput,
    facts: Readonly<{ mailConnectionRegistered: boolean }>,
  ): InputValidation<ValidFormSettings, FormInputReason | FormSettingsReason>;
};
```

- どの `validate` も、見つけた問題をすべて返す。`path` は入力の中の位置（`destination`、`conditions.名前`、`credentials.identity`、`recipients.0.address`、`notification.subject` など）。`syntax` は、`reason` が `TemplateSyntax` の問題だけが持つ記法の誤りで、一件以上。`setting` は、`reason` が `MailSettingRejected` の問題だけが持つ、アダプターの規則の理由。`FormInputIssue` は入力された値を持たない
- `MailConnectionInput.validate` は、`MailEndpoint` の制約（`InvalidMailDestination`、`InvalidMailCondition`）と、認証情報の制約を確かめる。空の接続先は `InvalidMailDestination`（`path` は `destination`）、組が一つも無い接続条件は `InvalidMailCondition`（`path` は `conditions`）、値が空文字列の接続条件は `InvalidMailCondition`（`path` は `conditions.名前`）で、登録と更新のどちらでも拒否する
- `credentials` は保護していない認証情報で、`null` は入力しなかったことを表す。登録（`register`）では `null` を `MailSecretRequired`（`path` は `credentials`）とする。更新（`update`）では `null` を受け付け、登録済みの認証情報を保つことを表す。`secret` の空文字列は `MailSecretRequired`（`path` は `credentials.secret`）。`identity` は、前後の空白を除いて 1 文字以上 320 文字以下で、違反は `InvalidMailIdentity`（`path` は `credentials.identity`）。`identity` の `null` は、利用者を識別する値を持たないことを表し、違反にしない。結果の `credentials` は `Confidential` に包む。登録の結果の `credentials` は、`null` にならない
- `MailConnectionInput.validate` は、上の制約の違反が無い入力について、接続先、接続条件（送信者のアドレスを含む）、認証情報の `identity` の有無を `rule.check` に渡し、送信の方式で使えるかを確かめる。`identity` の有無は、`credentials` を入力した場合はその `identity`、入力しなかった更新では `current` の認証情報の `identity` で決める。返った問題は、それぞれ `MailSettingRejected` の問題にし、`path` は、`target` が `destination` なら `destination`、`condition` なら `conditions.名前`、`identity` なら `credentials.identity`。問題が無ければ `MailSetting` を作る。`rule` は `MailSender.settingRule()` の結果（「MailSender」）
- `NotificationTemplateInput.validate` は、`TemplateText.parseSubject` と `TemplateText.parseBody` の規則（`TemplateSyntax`）、`senderName` の制約（`InvalidSenderName`）、`replyTo` が `FieldKey` の形式であること（`InvalidFieldKey`）を確かめる。フォームの項目への参照の検証は含まない。参照は、`FormSettings.changeNotification` が現行のフォームの定義で確かめる
- `RecipientAddressInput.validate` は `EmailAddress` の生成規則（`InvalidEmailAddress`）を、`RetentionInput.validate` は保持方針の範囲（`InvalidRetentionPolicy`）を確かめる。通知先の追加・修正と、保持方針の変更が使う
- `FormSettingsInput.validate` は、通知先ごとの `RecipientAddressInput.validate` の規則、通知先の列の規則（`RecipientList.check`。`DuplicateRecipient` の `path` は `recipients.位置`、`RecipientLimitReached` の `path` は `recipients`）、`NotificationTemplateInput.validate` の規則、`RetentionInput.validate` の規則（`retentionDays` の `null` は宣言なし）を確かめる。`reception` が `open` の入力は、`FormSettings.missingForReception` が返す足りないものごとに、`ReceptionNotReady` の問題を返す。`path` は、`enabledRecipient` では `recipients`、`mailConnection` では `reception`。初期構成では、同じワークスペースの `mailConnection` の宣言の有無を `mailConnectionRegistered` に渡す。設定の全体を入力するのは初期構成の宣言だけで、この `validate` を呼ぶのは「初期構成の宣言」の検証だけ
- `ValidFormSettings` を作るのは、`FormSettingsInput.validate` だけ。`PendingFormSettings.declare` は、この値だけを受け取る。初期構成の通知定義は、フォームの作成の前なので、項目への参照を検証しない。紐付けの後の不整合は `FormSettings.referenceIssues` が示す

### エラーコード

Form ドメインの拒否の理由のコードは、次ですべて。共通の理由のコード（`STALE_VERSION`、`ALREADY_EXISTS`、`SCHEMA_CHANGED`、`WORKSPACE_NOT_ACTIVE`、`PERMISSION_MISSING`、`ACTOR_NOT_ALLOWED`、`InvalidPagination`、`InvalidTargetCount`、`INVALID_QUERY`、`SECRET_UNAVAILABLE`）は [common.md](common.md) が定める。フォームの送信の受付失敗のうち、ドメインの判定（`formNotFound`、`closed`、`invalid`）は、エラーではなく `JudgedRejection` が表す。

```ts
type FormErrorCode =
  | FormRuleCode
  | FormRateLimitCode
  | FormInputReason
  | FormRequestReason
  | FormDeclarationReason;

type FormRuleCode =
  | "DuplicateRecipient"
  | "LastEnabledRecipient"
  | "RecipientLimitReached"
  | "ReceptionNotReady"
  | "NotificationReferenceBroken"
  | "FormRemoved"
  | "FormKeyMismatch"
  | "MailConnectionAlreadyRegistered";

type FormRateLimitCode = "SUBMISSIONS_LIMITED";

type FormInputReason =
  | "InvalidEmailAddress"
  | "InvalidFieldKey"
  | "TemplateSyntax"
  | "InvalidSenderName"
  | "InvalidRetentionPolicy"
  | "InvalidMailDestination"
  | "InvalidMailCondition"
  | "InvalidMailIdentity"
  | "MailSecretRequired"
  | "MailSettingRejected";

type FormSettingsReason = Extract<FormRuleCode, "DuplicateRecipient" | "RecipientLimitReached" | "ReceptionNotReady">;

type FormRequestReason = "InvalidReceivedRange";

type FormDeclarationReason = "FormKeyDuplicated";
```

| コード | 種類 | 意味 |
| --- | --- | --- |
| `DuplicateRecipient` | `BusinessRuleError` | 同じメールアドレスの通知先が既にある |
| `LastEnabledRecipient` | `BusinessRuleError` | 通知先の変更の後、受付中のフォームの通知先に `missingForReception` が `enabledRecipient` を返す |
| `RecipientLimitReached` | `BusinessRuleError` | 通知先が `RecipientList.max` 件を超える |
| `ReceptionNotReady` | `BusinessRuleError` | 受付を有効にする条件がそろわない。足りないもの（`ReceptionRequirement`）の一覧を添える |
| `NotificationReferenceBroken` | `BusinessRuleError` | 通知定義が、不整合のある参照を持つ。`ReferenceIssue` の一覧を添える |
| `FormRemoved` | `BusinessRuleError` | 削除済みフォームに、受付状態・通知定義・保持方針の操作を求めた |
| `FormKeyMismatch` | `BusinessRuleError` | 紐付けるフォームのキーが、紐付け待ちの設定のキーと違う |
| `MailConnectionAlreadyRegistered` | `BusinessRuleError` | 送信接続設定が登録済み |
| `SUBMISSIONS_LIMITED` | `RateLimitedError` | フォームの送信制限（`SubmissionRatePolicy`）に達している。再び試せる時刻（`retryAt`）を持つ |
| `FormInputReason` の各コード | `ValidationError` | 利用者の入力の項目ごとの理由。`FormInputIssue` が位置とともに持つ。`validate` だけが返す。`MailSettingRejected` は、接続先、接続条件、または認証情報の `identity` の有無を送信の方式で使えないことで、アダプターの規則の理由（`setting`）を持つ |
| `InvalidReceivedRange` | `ValidationError` | 受付期間の開始が終了より後 |
| `FormKeyDuplicated` | `DeclarationIssue.code` | 初期構成で、同じワークスペースのフォームのキーが重なる。エラーとして投げない |

- エラーの種類の使い分けは、[index.md](index.md) の「共通の型とポート」のエラーの種類の表に従う。エラーの種類は、コードごとに一つ
- 初期構成の宣言の検証（`FormDeclaration.validate`）は、エラーを投げず、問題を `DeclarationIssue` で返す。`DeclarationIssue.code` に入るコードは、`FormInputReason`、`FormSettingsReason`（`FormSettingsInput.validate` が、宣言した設定の全体に対して返す三つ）、`FormKeyDuplicated`、フォームのキーの形式の違反の理由（Schema の `INVALID_KEY`）

## ドメインサービス

どのサービスもポートに依存しない。

### SubmissionValidation

送信を、受付時点の現行のフォームの定義で検証する。Schema の `ValueValidation` に、フォーム固有の検証を加える。

```ts
const SubmissionValidation: {
  inquiryTargets(form: FormDefinition, input: SubmissionInput): readonly ContentId[];
  validate(form: FormDefinition, input: SubmissionInput, inquiryFacts: InquiryFacts): SubmissionValidationResult;
};

type SubmissionValidationResult =
  | Readonly<{ valid: true; content: SubmissionContent }>
  | Readonly<{
      valid: false;
      problems: SubmissionProblems<Extract<SubmissionProblem, { kind: "field" | "undefinedKey" }>>;
    }>;

type SubmissionIssue = Readonly<{
  field: FieldKey;
  issue:
    | ValueIssue["issue"]
    | Readonly<{ type: "mustBeTrue" }>
    | Readonly<{ type: "mustMatch"; other: FieldKey }>
    | Readonly<{ type: "inquiryTarget"; error: "unavailable" | "modelNotAllowed" }>;
}>;
```

`inquiryTargets` は、`contentReference` の項目に指定されたコンテンツID を返す。ユースケースは、この ID の事実を Content から得て `validate` に渡す。

`validate` は次の順に行い、見つけた問題をすべて返す。`SubmissionIssue` と `SubmissionProblem` は入力された値を持たない。

1. 入力の値を、項目の値の種類に従って `FormValues` に読む。読めない値を持つ項目は `typeError`（`kindMismatch`）とし、以後の検証から外す
2. `ValueValidation.validateLevel(ValueValidation.formLevel(form.fields), values)` を呼ぶ。その `issues` を `field` の問題に、`undefinedKeys` を `undefinedKey` の問題にする
3. フォーム固有の検証を行う。手順 1 と 2 で `typeError` を持った項目は対象から外す
4. 問題が一件も無ければ `valid` とし、`SubmissionContent` を返す。問題があれば、`field` の問題、`undefinedKey` の問題の順に並べて返す

手順 1 の読み方は次のとおり。

| 項目の値の種類 | 読める値 | 値 |
| --- | --- | --- |
| `text` | 文字列 | その文字列。空文字列は未入力 |
| `number` | 有限の数 | その数 |
| `boolean` | 真偽値 | その真偽値 |
| `date` | `CalendarDate` の形式の文字列 | その暦日 |
| `datetime` | `ValueConversion` がテキストから日時へ変換できる文字列 | その時点 |
| `select` | 文字列 | その選択肢のキー。キーの形式でない文字列は `typeError`（`optionMismatch`） |
| `multiSelect` | 文字列の並び | 重複を除いた選択肢のキーの組。空の並びは未入力。キーの形式でない文字列を含めば `typeError`（`optionMismatch`） |
| `contentReference` | 空でない文字列 | そのコンテンツID |

定義されていないキーの値は読まず、手順 2 がキーを返す。

手順 3 の検証は次のとおり。

| 検証 | 対象 | 条件 | 問題 |
| --- | --- | --- | --- |
| 真であること | `mustBeTrue` が真の `boolean` の項目 | 値が偽 | `mustBeTrue` |
| 二項目の一致 | `mustMatch` を持つ項目 | 自分の値と `mustMatch` の項目の値が等しくない。片方だけが未入力の場合を含む | `mustMatch`。`mustMatch` を持つ側の項目に付ける |
| 問い合わせ対象の公開状態 | 値を持つ `contentReference` の項目 | そのコンテンツID の事実が `unavailable` | `inquiryTarget`（`unavailable`） |
| 問い合わせ対象の許可範囲 | 値を持つ `contentReference` の項目 | 事実が `published` で、`model` が項目の `models` に無い | `inquiryTarget`（`modelNotAllowed`） |

- 未入力の項目の `mustBeTrue` は問題にならず、未入力は必須（`missingRequired`）が確かめる
- 二項目の一致は、値の種類ごとの値の一致で比べる。複数選択は、選んだ選択肢の組の一致で比べる。両方が未入力なら一致とする。`mustMatch` の項目が `typeError` を持つ場合は確かめない
- 欠落した値を `defaultValue` で補わない。入力された値を取り除いて受け付けない
- `content.inquiryTargets` は、`published` の事実から、項目のキーごとに作る
- 問い合わせ対象を、公開権限と通知先の指定に使わない

### NotificationRendering

通知定義を、回答の内容で描画する。

```ts
const NotificationRendering: {
  render(template: NotificationTemplate, source: RenderSource): RenderedNotification;
  sampleSource(form: FormDefinition, receivedAt: Date): RenderSource;
};

type RenderSource = Readonly<{ form: ReceivedForm; content: SubmissionContent; receivedAt: Date }>;

type RenderedNotification = Readonly<{
  subject: string;
  body: string;
  senderName: string | null;
  replyTo: EmailAddress | null;
  issues: readonly ReferenceIssue[];
}>;
```

`render` は、`source.form` に照らして参照を解決する。回答の通知と手動再試行では、`source` は回答が持つ受付時のフォーム、内容、受付日時で、`template` は受付時の通知定義。現行のフォームの定義と現行の通知定義を使わない。

ノードごとの描画は次のとおり。

| ノード | 描画 |
| --- | --- |
| `text` | その文章 |
| `field` | 項目の値の文字列。未入力は空文字列 |
| `inquiry` | 問い合わせ対象の送信時の情報のうち、`attribute` の値。`label` は表示名称（`null` は空文字列）、`model` はモデルのキー、`contentId` はコンテンツID、`uniqueValue` は一意な値を `, ` で連結した文字列。未入力は空文字列 |
| `receivedAt` | 受付日時を `Instant` の形式で表した文字列 |
| `allFields` | `source.form.fields` の順に、項目ごとに一行。行は、項目の表示名称（無ければキー）、`: `、項目の値の文字列。未入力の項目は値を空にして並べる |
| `ifAnswered` | 項目が未入力でなければ `children` を描画する。未入力なら何も出さない |

項目の値の文字列は次のとおり。

| 値の種類 | 文字列 |
| --- | --- |
| `text` | その文字列 |
| `number`、`boolean`、`date`、`datetime` | `ValueConversion` がテキストへ変換した文字列 |
| `select` | `form` の選択肢の表示名称。無ければ選択肢のキー |
| `multiSelect` | 選んだ選択肢の表示名称を、`form` の選択肢の順に `, ` で連結した文字列。表示名称が無い選択肢はキー |
| `contentReference` | 問い合わせ対象の表示名称。無ければコンテンツID |

不整合のある参照は次のとおり描画する。不整合は `NotificationTemplate.referenceIssues(template, source.form)` が決める。

- `field`、`inquiry` の不整合は、値の代わりに `[不整合な参照: キー]` を出す
- `ifAnswered` の不整合は、`children` を出さず、`[不整合な参照: キー]` を出す
- `replyTo` が不整合の場合、返信先は `null`。返信先を指定せずに送る
- `issues` は、描画で見つけた不整合

そのほかの規則は次のとおり。

- 件名は、描画した値に含まれる改行を空白一つに置き換える
- `replyTo` は、項目の値が未入力の場合と、値が `EmailAddress` として生成できない場合も `null`
- `senderName` は `template.senderName`

`sampleSource` は、入力例による表示のための `RenderSource` を作る。引数の `form` は現行のフォームの定義で、結果の `form` は `ReceivedForm.of(form)`。

- 項目の値は、`FormField.example` を `text` の値として、`ValueConversion.convert` でその項目の値の種類へ変換した値。複数選択は、単一選択への変換を経て、その選択肢だけを選んだ値にする
- 入力例を持たない項目、変換できない入力例を持つ項目、`contentReference` の項目は未入力とする
- 入力例による表示は `render(template, sampleSource(form, now))` で求める。結果は保存も送信もしない

### NotificationReferenceCheck

スキーマの反映で不整合となる通知定義の参照を求める。

```ts
const NotificationReferenceCheck: {
  afterApply(
    input: Readonly<{
      next: Schema;
      forms: FormChanges;
      formIds: Readonly<Record<string, FormId>>;
      settings: readonly Versioned<FormSettings>[];
      pending: readonly Versioned<PendingFormSettings>[];
    }>,
  ): readonly BrokenNotificationReference<ReferenceIssueReason>[];
};
```

- `next` は反映後のスキーマ。`forms` は、Schema が反映の計画で求めた `FormChanges`（追加・削除・変更したフォーム）。`formIds` は、反映前の `WorkspaceSchema` の、フォームのキーから `FormId` への対応。`settings` と `pending` は、そのワークスペースの設定と紐付け待ちの設定。渡す値は、[../usecases/schema.md](../usecases/schema.md) の「ほかのドメインとの対応」に従う。追加・削除したフォームは `forms` だけから決め、検証の対象は、この関数が選ぶ

検証の対象は、「スキーマの反映に伴う変更」の紐付けで、反映の後に `active` である設定。準備の時点では、追加するフォームの `FormId` が決まっていないので、`BindingTargetChange` は、`created` を `forms.added` のキーと、ID の代わりの `null` で、`removed` を `forms.removed` のキーと、`formIds` のその `FormId` で作る（`KeyBinding.apply` の `TId` は `FormId | null`）。

| 設定 | 照らすフォームのキー |
| --- | --- |
| 結果が `unchanged` の `active` の設定 | `formIds` でその `formId` に対応するキー。対応するキーが無い設定は、対象にしない |
| 結果が `bound` の紐付け待ちの設定のうち、通知定義を持つ（`notification` が `null` でない）もの | `formKey` |

- 通知定義を持たない紐付け待ちの設定は、紐付けで `NotificationTemplate.initial` になり、不整合を持たないので、対象にしない
- 対象ごとに、`NotificationTemplate.referenceIssues` を、反映後のフォームの定義を `ReceivedForm.of` で写した値で呼び、結果を Schema の `BrokenNotificationReference`（フォームのキー、項目のキー、理由を型引数で受ける運び手）に、理由を `ReferenceIssueReason` にして入れる。同じ組は一つにまとめる
- 結果は、`next.forms` の順、同じフォームでは不整合が現れる順に並べる
- この検出は、反映を失敗させない。直接反映の要求での確認の扱いは、[schema.md](schema.md) の「反映の要求の検証」が定める。通知定義を補正も削除もしない

参照の不整合は、保存した状態として持たず、`FormSettings.referenceIssues` が求める。

### FormInventory

フォームの運用設定と、フォームごとの通知の件数から、フォームを単位にする一覧の対象を決める。

```ts
type FormNotificationSummary = Readonly<{ formId: FormId; unsent: number; succeeded: number; failed: number }>;

type MailConnectionImpact = Readonly<{
  settings: FormSettings;
  unsent: number;
  failed: number;
}>;

const FormInventory: {
  remaining<S extends FormSettings | Versioned<FormSettings>>(
    settings: readonly S[],
    summaries: readonly FormNotificationSummary[],
  ): readonly S[];
  disposable(
    settings: readonly Versioned<FormSettings>[],
    summaries: readonly FormNotificationSummary[],
  ): readonly Versioned<RemovedFormSettings>[];
  affectedByMailConnection(
    settings: readonly Versioned<FormSettings>[],
    summaries: readonly FormNotificationSummary[],
  ): readonly MailConnectionImpact[];
};
```

`summaries` は、`SubmissionNotificationRepository.summarizeByForm` の結果。`unsent`、`succeeded`、`failed` は、`SubmissionNotification.result` がその値の通知の件数で、通知の送信結果だけを数える。`summaries` に `formId` があるフォームは、回答を持つフォーム。回答と通知は作成と削除でそろう（`SubmissionRepository` の「回答と通知の対応」）ので、通知を持つフォームと回答を持つフォームは一致する。「回答が残る」の判定は、この一致だけに依り、回答一件あたりの通知の数に依らない。

| メソッド | 返すもの |
| --- | --- |
| `remaining` | `active` のすべての設定と、`summaries` にその `formId` がある（回答が残る）`removed` の設定。渡された順を保つ。フォームの運用設定の一覧と、回答の対象のフォームの一覧が使う |
| `disposable` | `summaries` にその `formId` が無い（回答が残らない）`removed` の設定と `version`。定期ジョブ「回答の自動整理」が削除する対象 |
| `affectedByMailConnection` | `active` と `removed` の設定（`MailConnectionImpact.settings` は、渡した `entity`）のうち、`FormSettings.isAccepting` が真、または `summaries` のそのフォームの `unsent` か `failed` が 1 以上のもの。渡された順を保つ。受付中かどうかは、`FormSettings.isAccepting(settings)` が示す。件数は `summaries` の値で、`summaries` に無いフォームは 0。送信接続設定の更新が影響するフォームを表す |

- `remaining` と `disposable` は、`removed` の設定を過不足なく二つに分ける
- 送信成功の通知だけを持つ受付停止のフォームと、回答の無い受付停止のフォームは、送信接続設定の更新の影響を受けない

### SubmissionQuery

回答の一覧・検索の条件を定める。リポジトリの絞り込みは、この述語と同じ結果を返す。

```ts
type NotificationCondition =
  | Readonly<{ result: "unsent" }>
  | Readonly<{ result: "succeeded" }>
  | Readonly<{ result: "failed"; handling: FailureHandling["status"] | null }>;

type SubmissionFilter = Readonly<{
  formId: FormId | null;
  receivedFrom: Date | null;
  receivedTo: Date | null;
  notification: NotificationCondition | null;
  search: Readonly<{ text: Confidential<string>; listFields: Readonly<Record<string, readonly FieldKey[]>> }> | null;
}>;

const SubmissionFilter: {
  create(params: Readonly<{
    formId: FormId | null;
    receivedFrom: Date | null;
    receivedTo: Date | null;
    notification: NotificationCondition | null;
    search: SubmissionFilter["search"];
  }>): SubmissionFilter;
};

type ListValue = Readonly<{ field: FieldKey; value: string | null }>;

const SubmissionQuery: {
  matches(submission: Submission, notification: SubmissionNotification, filter: SubmissionFilter): boolean;
  listValues(submission: Submission, listFields: readonly FieldKey[]): Confidential<readonly ListValue[]>;
  searchTexts(submission: Submission, listFields: readonly FieldKey[]): Confidential<readonly string[]>;
  normalize(text: string): string;
};
```

`SubmissionFilter.create` は、`receivedFrom` が `receivedTo` より後の指定を `ValidationError`（`InvalidReceivedRange`）で拒否する。

`null` の条件は絞り込まない。指定した条件は、すべてを満たすものに絞り込む。`matches` の `notification` は、その回答の通知。

| 条件 | 該当するもの |
| --- | --- |
| `formId` | `formId` が一致する。削除済みフォームの `FormId` も指定できる |
| `receivedFrom` / `receivedTo` | `receivedAt` が `receivedFrom` 以降で、`receivedTo` より前 |
| `notification` | `SubmissionNotification.result` が `result` と一致する。`failed` の `handling` が `null` でなければ、加えて、通知の `handling.status` が一致する。対処の状態での絞り込みは、送信失敗の絞り込みだけが持つ |
| `search` | 下の規則で、一覧表示用のテキストの項目の値が `text` を含む |

`search` の規則は次のとおり。

- `listFields` は、`FormId` から、現行スキーマのそのフォームの一覧表示用の項目のキーへの対応。ユースケースが現行スキーマから作る。削除済みフォームは対応を持たない
- 一覧表示用の項目の値は、`listValues` だけが求める。`listFields` は、現行スキーマの、その回答のフォームの一覧表示用の項目のキー。結果は、`listFields` の順に、キーごとに一件。受付時のフォーム（`Submission.form`）に同じキーの項目があり、回答がその値を持てば、その値の文字列。文字列は、値の種類によらず、「NotificationRendering」の「項目の値の文字列」で求める。ほか（受付時のフォームに同じキーの項目が無い、未入力）は `null`。回答の一覧の出力に示す一覧表示用の項目の値は、この結果
- 検索が照合する値は、`searchTexts` だけが求める。結果は、`listValues` の結果のうち、受付時のフォームでその項目の値の種類が `text` で、値が `null` でないものの値。テキストでない項目の値は、一覧に示し、検索に使わない
- キーは一致だけで照合する。項目のキーの変更の経緯を追わない
- 回答は、`searchTexts(submission, listFields のその回答の formId のキー)` の値のどれかを `normalize` した文字列が、`text` を `normalize` した文字列を含む場合に、検索に該当する。現行の一覧表示用のテキストの項目のキーを受付時に持たない回答と、削除済みフォームの回答（`listFields` に対応が無い）は、検索に該当しない
- `normalize` は、Unicode の NFKC で正規化し、小文字にする
- `text` を `normalize` した結果が空の `search` は、絞り込まない

## ポート

ポートは `packages/core/src/domain/form/ports/` に置く。どのポートも、ドメイン型を受け渡す。

リポジトリには、[common.md](common.md) の「TransactionalRepository」の節の、すべてのリポジトリに共通する規則を適用する。下の表は、それに加える契約だけを書く。

Form のリポジトリは、どれもワークスペースの区画にある（[index.md](index.md) の「区画」）。

継ぐ基底と、メソッドの分類（[common.md](common.md) の「TransactionalRepository」）は次のとおり。基底のメソッドの分類は、common.md が定める。

| ポート | 継ぐ基底 | 加える読み取り | 加える書き込み |
| --- | --- | --- | --- |
| `FormSettingsRepository` | `Omit<TransactionalRepository<FormSettings, FormId>, "insert">` | `findAll` | `applySchemaChange`、`deleteAll` |
| `PendingFormSettingsRepository` | 継がない。紐付け待ちの設定は作成の後に変わらない | `findAll` | `insertMany`、`deleteMany`、`deleteAll` |
| `MailConnectionRepository` | `SingletonRepository<MailConnection>` | なし | `deleteAll` |
| `SubmissionRepository` | 継がない。回答は `version` を持たない | `findById`、`findPage`、`previewRetention`、`countExpired` | `insert`、`deleteMany`、`replaceRetentionByForm`、`deleteExpired`、`deleteAll` |
| `SubmissionNotificationRepository` | `Omit<TransactionalRepository<SubmissionNotification, SubmissionId>, "delete">` | `findByIds`、`countUnhandledFailures`、`summarizeByForm`、`findUnsettled`、`findStalled` | なし |

UnitOfWork の外の記録に当たるメソッドは無い。

### FormSettingsRepository

`FormSettings` を保存し、取り出す。設定を作る書き込みは、スキーマの反映の確定の `applySchemaChange` だけで、一件ずつ作る呼び手が無いので、`insert` を除いた `TransactionalRepository` を継ぐ。

```ts
interface FormSettingsRepository extends Omit<TransactionalRepository<FormSettings, FormId>, "insert"> {
  applySchemaChange(change: FormSettingsSchemaChange): Promise<void>;
  findAll(): Promise<readonly Versioned<FormSettings>[]>;
  deleteAll(limit: number): Promise<number>;
}
```

| メソッド | 契約 |
| --- | --- |
| `findById` | その `formId` の設定を返す |
| `save` | 楽観ロックで保存する。`formId` が変わらない変更（`active` の変更、`active` から `removed`、`removed` の通知先の変更）に使う |
| `delete` | その `formId` の設定を、楽観ロックで削除する。定期ジョブ「回答の自動整理」が、回答が残らない `removed` の設定の削除に使う。回答の有無は確かめない |
| `applySchemaChange` | スキーマの反映の確定に伴う設定の書き込みを、一回の呼び出しで行う。引数は `FormSettings.planSchemaChange` の結果。`created` の設定と、`bound` の組の `settings` を保存する。`bound` の組の `pending` は書かない（`PendingFormSettingsRepository.deleteMany` が削除する）。`removed` の設定を、`expectedVersion` で保存する（`save` と同じ契約）。全部を確定するか、何も確定しないかのどちらかになる。`removed` の保存の `version` の競合（対象が既に無い場合を含む）が一件でもあれば `ConflictError`（理由 `STALE_VERSION`）、一意性の違反が一件でもあれば `ConflictError`（理由 `ALREADY_EXISTS`）で、その UnitOfWork の書き込みは一つも反映されない。同じ設定を指す要素を二つ渡さないことは、呼び出し側の条件。三つの列がすべて空なら、何も書かない。件数の上限を設けない。件数は、スキーマのフォームの数までになる。書き込みの量は、件数によらず一回（[common.md](common.md) の「UnitOfWorkProvider」） |
| `findAll` | 結び付いた区画の、すべての設定を、`expectedVersion` とともに返す。`createdAt` の昇順で並べる。件数を区切らない |
| `deleteAll` | 結び付いた区画の、すべての設定を対象にする |

- 一意性: 同じ `formId` の設定が高々一つであることを、ポートが担保する。違反の結果は、`applySchemaChange` の契約のとおり。並行した書き込みでも、一方だけが確定する
- フォームと設定の対応（現行スキーマのフォームごとに `active` が一つ、削除したフォームは `removed`）は、ポートではなく、設定を作る呼び出し側（スキーマの反映の確定）が保つ
- 同じ設定への並行した変更は、楽観ロックで一方だけが確定する。受付中の設定の通知先の不変条件（`FormSettings` の「不変条件」）は、この楽観ロックと、`setRecipientEnabled`・`removeRecipient` の拒否が保つ

### PendingFormSettingsRepository

`PendingFormSettings` を保存し、取り出し、削除する。紐付け待ちの設定は作成の後に変わらず、初期構成の読み込みが作る。紐付け待ちの設定を消す書き込みは、このポートの `deleteMany` と `deleteAll` だけ。

```ts
interface PendingFormSettingsRepository {
  insertMany(pending: readonly PendingFormSettings[]): Promise<void>;
  findAll(): Promise<readonly Versioned<PendingFormSettings>[]>;
  deleteMany(
    targets: readonly Readonly<{ value: PendingFormSettings; expectedVersion: ExpectedVersion<PendingFormSettings> }>[],
  ): Promise<void>;
  deleteAll(limit: number): Promise<number>;
}
```

| メソッド | 契約 |
| --- | --- |
| `insertMany` | 複数の紐付け待ちの設定を、一回の呼び出しで保存する。初期構成の読み込みが使う。一意性の違反（渡した列の中の同じ `formKey` を含む）が一件でもあれば `ConflictError`（理由 `ALREADY_EXISTS`）で、どの設定も保存しない。空の列は、何も書かない。件数の上限を設けない。書き込みの量は、件数によらず一回 |
| `findAll` | 結び付いた区画の、すべての紐付け待ちの設定を、`expectedVersion` とともに、`formKey` の昇順で返す。件数を区切らない |
| `deleteMany` | 渡した紐付け待ちの設定を、それぞれの `expectedVersion` で、一回の呼び出しで削除する。スキーマの反映の確定が、紐付けた設定（`FormSettingsSchemaChange` の `bound` の `pending`）に使う。`version` の競合（対象が既に無い場合を含む）が一件でもあれば `ConflictError`（理由 `STALE_VERSION`）で、どの設定も削除しない。同じ `formKey` を二度含む列の結果は、[common.md](common.md) の「TransactionalRepository」の対象の集合の指定に従う。空の列は、何も書かない。件数の上限を設けない。書き込みの量は、件数によらず一回 |
| `deleteAll` | 結び付いた区画の、すべての紐付け待ちの設定を対象にする |

- 一意性: 同じ `formKey` の紐付け待ちの設定が高々一つであることを、ポートが担保する。並行した `insertMany` でも、一方だけが確定する
- 紐付け待ちの設定のキーと、同じ区画の `FormSettings` の `removed` の `formKey` は、重なってよい

### MailConnectionRepository

`MailConnection` を保存し、取り出す。区画に一つの集約なので、`SingletonRepository`（[common.md](common.md)）を継ぐ。登録した設定は、利用終了の消去（`deleteAll`）まで残る。

```ts
interface MailConnectionRepository extends SingletonRepository<MailConnection> {
  deleteAll(limit: number): Promise<number>;
}
```

| メソッド | 契約 |
| --- | --- |
| `insert` | `SingletonRepository` の契約のとおり。区画に一つであることを、ポートが担保する |
| `find` | 結び付いた区画の送信接続設定を返す。未登録なら `null`。保存した `ProtectedMailSecret` と同じ値を返す |
| `save` | 楽観ロックで保存する |
| `deleteAll` | 結び付いた区画の送信接続設定を対象にする。返す件数は 0 または 1 |

保存する認証情報は `ProtectedMailSecret` に限る。ポートは保護も復元も行わない。

### SubmissionRepository

回答を保存し、取り出し、一覧する。回答は `version` を持たないので、`TransactionalRepository` を継がず、持つメソッドをすべてここに宣言する。回答は、受付の `insert` で作り、まとめて削除するメソッド（`deleteMany`、`deleteExpired`、`deleteAll`）だけが消す。

```ts
interface SubmissionRepository {
  insert(submission: Submission): Promise<void>;
  findById(id: SubmissionId): Promise<Submission | null>;
  deleteMany(ids: readonly SubmissionId[]): Promise<void>;
  findPage(filter: SubmissionFilter, pagination: Pagination): Promise<PaginationResult<Submission>>;

  previewRetention(formId: FormId, policy: RetentionPolicy, now: Date): Promise<RetentionPreview>;
  replaceRetentionByForm(formId: FormId, policy: RetentionPolicy, limit: number): Promise<number>;

  countExpired(now: Date): Promise<number>;
  deleteExpired(now: Date, limit: number): Promise<number>;
  deleteAll(limit: number): Promise<number>;
}

type RetentionPreview = Readonly<{
  total: number;
  oldestReceivedAt: Date | null;
  newestReceivedAt: Date | null;
  changing: number;
  expiring: number;
}>;
```

| メソッド | 契約 |
| --- | --- |
| `insert` | 回答を保存する。同じ `id` の回答が既にあれば `ConflictError`（理由 `ALREADY_EXISTS`）。ポートが担保する。対になる通知は、下の「回答と通知の対応」のとおり |
| `findById` | `id` で一件を返す。無ければ `null`。`expectedVersion` を返さない |
| `deleteMany` | 渡した ID のうち、結び付いた区画にある回答を、一度の削除として削除する。区画に無い ID（既に削除された回答、ほかの区画の回答）は、削除済みとして扱い、何も書かない。区画にある回答は、全部を削除するか、どれも削除しないかのどちらかで、ポートが担保する。回答を削除するほかの書き込み（`deleteMany`、`deleteExpired`、`deleteAll`）と並行しても `ConflictError` にならず、確定の後、渡した ID の回答は一件も残らない。通知の進行中の回答も削除する。渡せる ID は 1 件以上 `SubmissionBatch.maxTargets` 件以下 |
| `findPage` | 結び付いた区画の回答のうち、`SubmissionQuery.matches` が真になる回答を、`receivedAt` の降順で返す。削除済みフォームの回答を含む |
| `previewRetention` | そのフォームのすべての回答について、件数、最も古い・新しい `receivedAt`、保持方針が `policy` と異なる件数（`changing`）、`RetentionPolicy.isExpired(policy, receivedAt, now)` が真になる件数（`expiring`）を返す。回答が無ければ件数は 0、日時は `null` |
| `replaceRetentionByForm` | そのフォームの回答のうち、保持方針が `policy` と異なる回答に、`Submission.applyRetention` を適用した結果を保存する。`receivedAt` の昇順に最大 `limit` 件を置き換え、置き換えた件数を返す。`limit` の上限は `FormSweep.maxLimit`。返した件数が `limit` より少なければ、対象は残っていない |
| `countExpired` | 結び付いた区画の、`RetentionPolicy.isExpired(retention, receivedAt, now)` が真の回答の件数を返す |
| `deleteExpired` | 結び付いた区画の、`RetentionPolicy.isExpired(retention, receivedAt, now)` が真の回答を対象にする。`Submission.expiresAt` の早い順に削除する。通知の送信結果と進行によらず削除する |
| `deleteAll` | 結び付いた区画の回答を対象にする |

- 回答と通知の対応: 回答を削除するどのメソッドも、同じ `SubmissionId` の通知を、同じ UnitOfWork の確定で削除する。この担保はポートが行う。作成の時点で回答と通知がそろうことは、呼び出し側が、回答の `insert` と、その通知の `SubmissionNotificationRepository.insert` を同じ UnitOfWork で行って保つ。一方だけを `insert` した結果を、ポートは約束しない（[common.md](common.md) の「TransactionalRepository」の、参照先の存在）
- 並行性: 楽観ロックを使わない。回答の書き込みは、`ConflictError`（`STALE_VERSION`）にならない。回答を書くのは、保持方針の置き換えと削除だけ。並行した置き換えどうしは、後に確定した保持方針が残る。置き換えと削除が並行した回答は、削除される。通知の進行（`SubmissionNotificationRepository.save`）は回答を書かないので、回答の書き込みと競合しない。削除が先に確定した通知の `save` は `ConflictError`（理由 `STALE_VERSION`）になり、削除した回答と通知は戻らない
- 可視性: 削除した回答は、確定の後、どの読み取りにも現れない。一覧と並行した書き込みは、ページをまたぐと重複または欠落が起こり得る
- 回答の入力内容（`content`）と、検索のためにアダプターが持つ値は、回答の削除と同時に削除する。削除した回答の入力内容を、通知のために残さない
- 検索の文字列（`SubmissionFilter.search.text`）と `content` をログとエラー情報に出さないことと、容量の上限で保存できない `insert` が `SystemError` になることは、アダプターごとの検証の対象で、ポート適合テストの対象にしない

### SubmissionNotificationRepository

通知の進行を保存し、取り出し、集計する。一件ずつ削除する呼び手が無いので、`delete` を除いた `TransactionalRepository` を継ぐ。通知の作成と削除が回答とそろうことは、`SubmissionRepository` の「回答と通知の対応」が定める。通知の状態による回答の一覧は、`SubmissionRepository.findPage` が担う。

```ts
interface SubmissionNotificationRepository
  extends Omit<TransactionalRepository<SubmissionNotification, SubmissionId>, "delete"> {
  findByIds(ids: readonly SubmissionId[]): Promise<readonly Versioned<SubmissionNotification>[]>;
  countUnhandledFailures(): Promise<number>;
  summarizeByForm(): Promise<readonly FormNotificationSummary[]>;

  findUnsettled(limit: number): Promise<readonly SubmissionId[]>;
  findStalled(now: Date, limit: number): Promise<readonly SubmissionId[]>;
}
```

| メソッド | 契約 |
| --- | --- |
| `insert` | 回答一件に通知が一つであることを、ポートが担保する。同じ `submissionId` が既にあれば `ConflictError`（理由 `ALREADY_EXISTS`） |
| `findById` | `submissionId` で一件を返す |
| `save` | 楽観ロックで保存する。同じ通知の並行した進行は、一方だけが確定する |
| `findByIds` | 渡した ID の通知を、`expectedVersion` とともに、`submissionId` の昇順で返す。区画に無い ID の通知は返さない。渡せる ID の数の上限は、「FormSweep」が定める |
| `countUnhandledFailures` | 結び付いた区画の、`state` が `failed` で `FailureHandling.isUnhandled(handling)` が真の通知の件数を返す。対処の状態で数えるメソッドは、これだけ。未対処の送信失敗の件数は、どれもこの値から求める |
| `summarizeByForm` | 結び付いた区画の通知を持つフォームごとに、`FormNotificationSummary`（「FormInventory」）を返す。回答を持たないフォームは含まない。`formId` の昇順で並べる。通知の送信結果で数えるメソッドは、これだけ。通知の送信結果ごとの件数は、どれもこの結果から求める |
| `findUnsettled` | 結び付いた区画の、`SubmissionNotification.result` が `unsent` の通知の ID を、`receivedAt` の昇順に最大 `limit` 件返す。`limit` の上限は `FormSweep.maxLimit` |
| `findStalled` | 結び付いた区画の、`SubmissionNotification.stalled(notification, now)` が `null` でない通知の ID を、`receivedAt` の昇順に最大 `limit` 件返す。`limit` の上限は `FormSweep.maxLimit` |

### MailSender

送信接続設定を使って、メールを送る。回答の通知と、送信接続設定の動作確認が使う。設置全体のメール送信（`SystemMailer`）と別。送信の方式と、接続条件の語彙は、アダプターが決める。契約は、宛先ごとの結果だけを定め、宛先をまたぐ接続の使い方を定めない。

```ts
interface MailSender {
  settingRule(): MailSettingRule;
  send(
    connection: MailConnection,
    mail: Confidential<OutgoingMail>,
    recipients: readonly EmailAddress[],
  ): Promise<readonly RecipientResult[]>;
}

type MailSettingCandidate = Readonly<{
  destination: string;
  conditions: Readonly<Record<string, string>>;
  hasIdentity: boolean;
}>;

type MailSettingRule = Readonly<{
  check(candidate: MailSettingCandidate): readonly MailSettingIssue[];
}>;

type MailSettingIssue = Readonly<{
  target:
    | Readonly<{ type: "destination" }>
    | Readonly<{ type: "condition"; name: string }>
    | Readonly<{ type: "identity" }>;
  reason: "required" | "unknown" | "invalid";
}>;
```

| メソッド | 契約 |
| --- | --- |
| `settingRule` | このアダプターの送信の方式の規則を返す。規則の `check` は、接続先、接続条件（送信者のアドレスを含む）、認証情報の `identity` の有無（`hasIdentity`）がこの方式で使えるかを確かめ、見つけた問題をすべて返す。空は問題が無いことを表す。`settingRule` と `check` は、メール送信基盤へ接続せず、状態を持たず、同じ引数から同じ結果を返す。`check` は純粋な関数で、`MailConnectionInput.validate` が値オブジェクトの生成の中で呼ぶ |
| `send` | 渡した `connection` で、宛先ごとに一通のメールを送る。メールの宛先は、その一つの宛先だけとする。送信者のアドレスは、`connection.endpoint` の、送信者のアドレスの接続条件の値。送信者の表示名、返信先、件名、本文は `mail` の値。件名が空文字列のメールも、件名を空にして送る。`recipients` に同じ宛先が重なる場合は、一通だけ送る。結果は、重複を除いた宛先ごとに一件を、`recipients` に最初に現れた順に返す |

規則の `check` の問題は次のとおり。問題は `MailConnectionInput.validate` が `MailSettingRejected` の問題にし、問題のある設定から `MailSetting` は作られない。

| `reason` | 意味 |
| --- | --- |
| `required` | アダプターが必要とする名前の接続条件、または認証情報の `identity` が無い。`condition` の `name` は必要な名前。名前の有無だけで決まる。`MailEndpoint` の値は空文字列にならないので、値が空の接続条件は、規則に届かない |
| `unknown` | アダプターが解釈しない名前の接続条件がある。または、`identity` を使わない方式に、`identity` を持つ認証情報を渡した |
| `invalid` | 接続先、または接続条件の値が、アダプターの受け付ける形でない。メールアドレスの形式でない送信者のアドレスを含む |

- 一つのアダプターは、`identity` を必要とするか、使わないかのどちらか。必要とするアダプターは、`hasIdentity` が偽の設定に `required` を、使わないアダプターは、`hasIdentity` が真の設定に `unknown` を返す。アダプターは、必ずこの二つの一方に当たる
- 接続条件の名前ごとに、アダプターは、必要とするか、任意で解釈するか、解釈しないかのどれか。必要とする名前が無い設定に `required` を、解釈しない名前を持つ設定に `unknown` を返す
- どのアダプターも、送信者のアドレスを表す接続条件を一つ必要とする。名前はアダプターが決める。その接続条件が無い設定に `required` を、値がメールアドレス（共通の `EmailAddress`）の形式でない設定に `invalid` を返す

`send` の結果は次のとおり。

| 結果 | 意味 |
| --- | --- |
| `accepted` | メール送信基盤が、その宛先へのメールを受理したことを確認した。受信箱への到着は示さない |
| `failed` | 受理しなかった、または受理を確認できなかった。理由は `DeliveryFailureReason` |

宛先ごとの `failed` の理由は、その宛先への送信で起きたことから決まる。

| その宛先への送信で起きたこと | 理由 |
| --- | --- |
| メール送信基盤へ接続できない。または、その宛先のメールを渡していないことを、アダプターが確かめられる失敗 | `connection` |
| メール送信基盤が認証を拒否した | `authentication` |
| メール送信基盤が、その宛先を拒否した | `recipientRejected` |
| メール送信基盤が、ほかの理由でそのメールを受理しなかった | `rejected` |
| そのメールを渡したかどうかを確かめられないまま応答が途切れた、または `MailSendPolicy.recipientTimeoutMs` までに応答が返らない | `unconfirmed` |

- 送信の失敗で例外を投げない。どの失敗も、宛先ごとの `failed` で返す
- 接続の後に切れた失敗で、メールを渡していないことを確かめられるかは、送信の方式による。確かめられない失敗は `unconfirmed` にし、`connection` にしない
- 一通あたりの待ち時間の上限は `MailSendPolicy.recipientTimeoutMs`。一回の呼び出しは、重複を除いた宛先の数とこの値の積までに、すべての宛先の結果を返す。宛先ごとに別の接続で送るアダプターも、一本の接続で順に送るアダプターも、この上限を守る
- 一つの宛先の失敗が、ほかの宛先の結果を決めることを、契約は定めない。宛先ごとに別の接続で送るアダプターと、一本の接続で順に送るアダプターのどちらも、宛先ごとに上の表で結果を返す
- 保護した認証情報（`connection.credentials` の `identity` と `secret`）は、アダプターが `SecretCipher.reveal` で元の値に戻して使う。`reveal` が `SystemError`（理由 `SECRET_UNAVAILABLE`）になる場合は、何も送らず、すべての宛先を `failed`（`authentication`）で返す
- 一意性・整合性: 同じメールの重複を抑えない。呼び出すたびに送る。同じ宛先へ二度送らないことは、`SubmissionNotification` の宛先ごとの結果で呼び出し側が保つ
- 自動で再試行しない。一つの宛先への送信は一度だけ試みる
- 並行性: 状態を持たない。一回の呼び出しは、渡した `connection` だけを使い、送信の途中で設定を読み直さない
- 可視性: UnitOfWork に属さない
- エラーケース: 空の `recipients` は `SystemError`（理由 `INVALID_QUERY`）
- 件名、本文、返信先、認証情報の値をログとエラー情報に出さないことは、アダプターごとの検証の対象で、ポート適合テストの対象にしない
- ポート適合テストは、応答と状態を指定できる試験用の相手に対して実行し、送信の方式に依らないケースで確かめる（[common.md](common.md)）

## トランザクション境界

`UnitOfWorkProvider.run(scope, fn)` と `WriteScope` の契約は [common.md](common.md) が定める。Form の集約とポートは、すべてワークスペースの区画にある。この節は、定義元だけを示す。

| 定めること | 定義元 |
| --- | --- |
| UnitOfWork ごとの条件（`WorkspaceCondition`）、一つの UnitOfWork で確定する書き込みの組、書き込みの回数、UnitOfWork の前後に行うこと | [../usecases/form.md](../usecases/form.md) の各ユースケースの「トランザクション境界」と、共通の「書き込みの UnitOfWork」「通知の送信」 |
| 途中で止まったときに残る状態と、その後の扱い | `spec/testcases/form/` の各ユースケーステスト |
| Installation のユースケースが呼ぶ手順（初期構成の読み込みの生成、利用再開前の処理、利用終了の消去）の UnitOfWork | この文書の「初期構成の宣言」「利用再開前の処理」「利用終了の消去」 |
| スキーマの反映の確定に伴う `FormSettings` と紐付け待ちの設定の書き込み | 「スキーマの反映に伴う変更」と、`FormSettingsRepository.applySchemaChange`、`PendingFormSettingsRepository.deleteMany` の契約。確定の UnitOfWork は [../usecases/schema.md](../usecases/schema.md) |
| 回答と通知が、作成と削除でそろうこと | `SubmissionRepository` と `SubmissionNotificationRepository` の契約 |
| `MailSender` と `UsageLimiter` が UnitOfWork に属さないこと | [common.md](common.md) の「UnitOfWorkProvider」 |
| 競合の扱い、イベントの消費者と定期ジョブの失敗の扱い、通知を送れないワークスペースの扱い | [index.md](index.md) の「競合の扱い」「イベントの消費者と定期ジョブ」「利用できるワークスペース」 |

## 初期構成の宣言

初期構成の読み込みで、Form が受ける宣言と、その検証・生成の手順。宣言の形の全体と、`path` に渡す位置は、[installation.md](installation.md) の「初期構成の読み込み」が、各ドメインを呼ぶ順は [index.md](index.md) の「手順を呼ぶ順」が定める。`DeclarationIssue` は [common.md](common.md) が定める。生成は、Installation のユースケースが、宣言したワークスペースごとに、その区画の一つの UnitOfWork（`{ kind: "workspace", workspaceId, condition: { kind: "none" } }`）で、ほかのドメインの生成とともに確定する（[index.md](index.md) の「区画をまたぐ手順」の初期構成の読み込み）。

```ts
type FormSettingsDeclaration = Readonly<{ form: string; settings: FormSettingsInput }>;

type FormDeclaration = Readonly<{
  mailConnection: Readonly<{
    path: string;
    declaration: MailConnectionInput;
  }> | null;
  forms: readonly Readonly<{ path: string; declaration: FormSettingsDeclaration }>[];
}>;

const FormDeclaration: {
  validate(workspaces: readonly FormDeclaration[], rule: MailSettingRule): readonly DeclarationIssue[];
};
```

- `workspaces` は、宣言のワークスペースごとの、送信接続設定と、フォームのキーを指定した運用設定（受付状態、通知先、通知定義、回答の保持方針）
- `rule` は、`MailConnectionInput.validate` の `rule` と同じ（「MailConnectionInput と FormSettingsInput」）。検証はポートを呼ばない
- 宣言の照合値から除く値: 送信接続設定の認証情報（`mailConnection.declaration.credentials`）。ほかの値（接続先、接続条件、フォームの運用設定）は、照合値に含める
- 前提: 先に生成を終えているべきドメインは無い。フォームの運用設定は、スキーマのフォームを待つ紐付け待ちの設定（`PendingFormSettings`）で作り、ほかのドメインの集約を参照しない

検証。

| 規則 | `path` | `code` |
| --- | --- | --- |
| `MailConnectionInput.validate(input, { kind: "register" }, rule)` | `mailConnection.path` に、返った `FormInputIssue.path` を続ける | 返った `FormInputIssue.reason` |
| フォームのキーが、Schema の `FormKey` の形式に合う | フォームの `path` に `.form` を続ける | 投げた `ValidationError` の理由 |
| フォームのキーが、同じワークスペースの宣言の中で重ならない | 重なる二件目以降の `path` に `.form` を続ける | `FormKeyDuplicated` |
| `FormSettingsInput.validate(settings, { mailConnectionRegistered })`。`mailConnectionRegistered` は、同じワークスペースの `mailConnection` が `null` でないこと | フォームの `path` に `.settings.` と、返った `FormInputIssue.path` を続ける | 返った `FormInputIssue.reason` |

- `FormKeyDuplicated` は、この検証だけが返す理由。エラーとして投げない
- フォームのキーが指す対象の存在と、通知定義が参照する項目の存在は確かめない。読み込みの時点で、ワークスペースのスキーマは空である

生成。ワークスペースごとに、その区画の UnitOfWork で、次の二つを保存する。

- 送信接続設定: `MailConnection.register`（`current` は `null`）の結果を `MailConnectionRepository.insert` で保存する。宣言が無いワークスペースは作らない。`register` に渡す認証情報は、`MailConnectionInput.validate` の結果の `credentials` の `identity`（`null` でなければ）と `secret` を、UnitOfWork の前に、それぞれ `SecretCipher.protect` で保護し、`ProtectedMailSecret.create` で包んだ値
- フォームの運用設定: フォームごとに、フォームのキーと、検証が返した `ValidFormSettings` の値を渡して `PendingFormSettings.declare` を呼び、結果のすべてを、`PendingFormSettingsRepository.insertMany` の一回の呼び出しで保存する。フォームの宣言が無いワークスペースは呼ばない
- 二つの保存は、互いの結果を読まない
- 生成の書き込みは、宣言したフォームの数によらず、`MailConnectionRepository.insert` の一回と `insertMany` の一回

## 利用再開前の処理

バックアップから復旧した設置で、利用の再開の前に Form が済ませる処理。Installation のユースケースが、対象のワークスペースごとに呼ぶ。呼ぶ順は [index.md](index.md) の「手順を呼ぶ順」が定める。どの処理も、そのワークスペースの区画で行う。

- 前提: Workspace（名簿をワークスペースの区画に合わせて直す処理）を、先に終えている。対象のワークスペースを、直した名簿から求めるため
- 対象のワークスペース: 直した名簿に項目があるすべてのワークスペース。名簿の状態（準備中、開設済み、終了中）で絞らない

Form の処理は二つ。

| 処理 | 行うこと | UnitOfWork |
| --- | --- | --- |
| 保持期間を過ぎた回答の削除 | `SubmissionRepository.deleteExpired(now, limit)` を、[common.md](common.md) の「TransactionalRepository」の「件数を区切る削除」の進め方で呼ぶ | 条件は `none` |
| 送信を終えていない通知を送信失敗にする | `SubmissionNotificationRepository.findUnsettled(FormSweep.maxLimit)` を、UnitOfWork を開く前に一度読む。返った通知ごとに、`SubmissionNotificationRepository.findById` で通知を、`FormSettingsRepository.findById` でその `formId` の設定を読み、`SubmissionNotification.abandon(notification, settings, now)` の結果を `save` で保存する。一度の実行が扱うのは、読んだ列の通知まで | 通知一件ごとに別の UnitOfWork。条件は `none` |

- 回答の削除を、通知を送信失敗にする処理より先に行う。削除した回答の通知を、送信失敗として残さないため。復旧は、回答の保持期間を延ばさない
- 途中で止まった処理は、回答の削除からやり直す

利用再開前の確認に、Form が公開する内容。Installation のユースケースが、ワークスペースごとに、`PartitionReader.workspace(workspaceId)` のリポジトリで読んだ値を渡して求める。

```ts
type FormResumeItemKind =
  | "submissions_expired"
  | "notifications_succeeded"
  | "notifications_failed"
  | "notifications_undetermined";

const FormResumeItems: {
  definitions: readonly (ResumeItemDefinition & Readonly<{ kind: FormResumeItemKind }>)[];
  counts(summaries: readonly FormNotificationSummary[], expired: number): readonly ResumeItem[];
};
```

項目は、[common.md](common.md) の「各ドメインが公開する項目」の `ResumeItemDefinition`（`domain` は `form`）の定数の列で公開し、`counts` は同じ節の `ResumeItem` を返す。`counts` の `summaries` は `SubmissionNotificationRepository.summarizeByForm()`、`expired` は `SubmissionRepository.countExpired(now)` の結果。四つの項目を、表の順に返す。件数が 0 の項目も返す。下の `unsent`、`succeeded`、`failed` は、`summaries` のその値を、すべてのフォームについて足した値。

| `kind` | 項目 | 件数 | `unit` | `countsAsRemaining` | `shownInReview` |
| --- | --- | --- | --- | --- | --- |
| `submissions_expired` | 保持期間を過ぎた保存回答。利用再開前の処理が削除する | `expired` | `workspace` | 真 | 真 |
| `notifications_succeeded` | 通知の送信成功 | `succeeded` | `workspace` | 偽 | 真 |
| `notifications_failed` | 通知の送信失敗。送信済みか判断できない通知を含む | `failed` と `unsent` の和 | `workspace` | 偽 | 真 |
| `notifications_undetermined` | 送信済みか判断できない通知（送信を終えていない通知）。利用再開前の処理が、送信成功を確認できない送信失敗にする。`notifications_failed` の内数 | `unsent` | `workspace` | 真 | 真 |

- 残件数: `countsAsRemaining` が真の項目ごとの件数。どれも 0 なら、Form の処理は残っていない。処理が対象を済ませるごとに減る
- 残件数と確認の列への振り分けは、[installation.md](installation.md) の「ResumeReview」が定める
- フォームの送信の受付は、一つの UnitOfWork で確定し、進行中の状態を持たないので、復旧点で進行中だった受付は無い

## 状態確認の項目

設置の状態確認（Installation の `checkInstallationStatus`）と、利用終了の前の確認に、Form が示す件数の項目。[common.md](common.md)「各ドメインが公開する項目」の `StatusItem`（`domain` は `form`）で公開する。Installation のユースケースが、ワークスペースごとに、`PartitionReader.workspace(workspaceId)` のリポジトリで読んだ値を渡して求める。

```ts
type FormStatusKind = "notifications_unsent" | "notifications_unhandled";

type FormStatusItem = StatusItem & Readonly<{ domain: "form"; kind: FormStatusKind }>;

const FormStatus: {
  of(summaries: readonly FormNotificationSummary[], unhandledFailures: number): readonly FormStatusItem[];
  actionRequired(kind: FormStatusKind, count: number): boolean;
};
```

| `kind` | 項目 | 件数 | 必要な対処（`actionRequired`） |
| --- | --- | --- | --- |
| `notifications_unsent` | 未送信のフォームの通知 | `summaries` のフォームごとの `unsent` を、すべてのフォームについて足した値。送信を終えていない通知（`waiting` と `sending`）の件数 | 常に偽。件数を示すだけ |
| `notifications_unhandled` | 未対処の送信失敗のフォームの通知 | `unhandledFailures` | 件数が 1 以上なら真。通知設定の権限を持つメンバーが、管理画面で、再試行するか、対処済みとして記録する |

- `summaries` は `SubmissionNotificationRepository.summarizeByForm()`、`unhandledFailures` は `SubmissionNotificationRepository.countUnhandledFailures()` の結果
- `of` は、二つの項目を、表の順に返す。件数が 0 の項目も返す。`actionRequired` は、`FormStatus.actionRequired(kind, count)` の結果
- 二つの件数は足し合わせない
- 件数だけを示す。回答の内容、通知先、送信接続設定の内容を示さない

## 利用終了の消去

ワークスペースの利用終了で、Form の集約を、そのワークスペースの区画で消去する。Installation のユースケースが呼ぶ順は、[index.md](index.md) の「手順を呼ぶ順」が定める。

- 前提: 先に消去を終えているべきドメインは無い。Form の集約を参照する、ほかのドメインの集約は無い。Form の集約は、すべてこの区画にあり、設置の区画に消去するものを持たない

消去するのは、次の四つのメソッドの対象。表の順に呼ぶ。どれも区画の消去で、`{ kind: "workspace", workspaceId, condition: { kind: "none" } }` の UnitOfWork の内で、[common.md](common.md) の「TransactionalRepository」の「件数を区切る削除」の進め方で呼ぶ。一つのメソッドの対象が残らなくなってから、次のメソッドを呼ぶ。

| 呼ぶメソッド | 消去するもの |
| --- | --- |
| `SubmissionRepository.deleteAll` | 回答と、その通知 |
| `FormSettingsRepository.deleteAll` | フォームの運用設定（`active` と `removed`） |
| `PendingFormSettingsRepository.deleteAll` | 紐付け待ちの設定 |
| `MailConnectionRepository.deleteAll` | 送信接続設定 |

- 回答を消し切ってから、運用設定を消す。消去の途中で止まった区画（その時点のバックアップから復旧した区画を含む）でも、回答が残るフォームは設定を持つ（「通知の送信」）
- 送信制限の記録（`UsageLimiter`）は、区画に結び付かないので、消去の対象にしない（[common.md](common.md) の「UsageLimiter」）

利用終了の前の確認に、Form が公開する内容。種類の名前だけの項目は、[common.md](common.md)「各ドメインが公開する項目」の `DomainKind`（`domain` は `form`）の列で公開する。

```ts
type FormClosureKind<K extends string> = DomainKind & Readonly<{ domain: "form"; kind: K }>;

const FormClosure: {
  pending(summaries: readonly FormNotificationSummary[], unhandledFailures: number): readonly FormStatusItem[];
  lostData: readonly FormClosureKind<"submission" | "form_settings" | "mail_connection">[];
  remainsOutside: readonly FormClosureKind<"sent_notification_mail">[];
  erasedCredentials: readonly FormClosureKind<"mail_connection_secret">[];
};
```

| 公開するもの | 値 | 内容 |
| --- | --- | --- |
| 未完了の通知（`pending`） | `FormStatus.of(summaries, unhandledFailures)` と同じ二つの項目（`notifications_unsent`、`notifications_unhandled`）を返す | 未送信の通知は、利用終了で送られずに消える。未対処の送信失敗は、再試行できなくなる |
| 利用終了で失われるデータ（`lostData`） | `submission` | 回答（入力内容、問い合わせ対象の送信時の情報）と、通知の送信結果 |
| | `form_settings` | フォームの運用設定（受付状態、通知先、通知定義、回答の保持方針）。削除済みの設定と、紐付け待ちの設定を含む |
| | `mail_connection` | 送信接続設定 |
| CMS の通常データの外に残る情報（`remainsOutside`） | `sent_notification_mail` | 送信済みの通知メール。メール送信基盤と、通知先の受信箱に残り、CMS は消去しない |
| 消去する資格情報（`erasedCredentials`） | `mail_connection_secret` | 送信接続設定の認証情報 |

## ユースケース（概要）

名前は `spec/usecases/form.md` と一致する。各ユースケースが受け付ける実行者の種類と、必要な権限は、`spec/usecases/form.md` が定める。

- 送信接続設定を取得する
- 送信接続設定を登録する
- 送信接続設定を更新する
- 送信接続設定の動作を確認する
- 送信接続設定の更新が影響するフォームを一覧する
- フォームの運用設定を一覧する
- フォームの運用設定を取得する
- 通知先を追加する
- 通知先のメールアドレスを修正する
- 通知先を有効・無効にする
- 通知先を削除する
- 通知定義を変更する
- 通知定義を入力例で描画する
- 参照の不整合が残る通知定義を一覧する
- フォームの受付を有効にする
- フォームの受付を停止する
- 回答の保持方針を変更する
- 保持方針の既存の回答への適用を確認する
- 保持方針を既存の回答へ適用する
- フォームの送信を受け付ける
- 回答の通知を送る（`form.submission_accepted` と `form.notification_retry_requested` の消費）
- 回答の対象のフォームを一覧する
- 回答を一覧・検索する
- 回答を取得する
- 回答を削除する
- 未対処の送信失敗の件数を取得する
- 通知の送信失敗を対処済みとして記録する（一件以上を一度に）
- 手動再試行の宛先を確認する
- 通知を手動で再試行する
- 保持期限を過ぎた回答を自動整理する（定期ジョブ「回答の自動整理」）
- 回答が残らない削除済みフォームの運用設定を削除する（定期ジョブ「回答の自動整理」）
- 止まった通知を検出する（定期ジョブ「止まった通知の検出」）

スキーマの反映の準備と確定は、Schema のユースケース（[../usecases/schema.md](../usecases/schema.md)）が行う。初期構成の読み込みによる `MailConnection` と `PendingFormSettings` の作成、復旧後の利用の再開での、保持期限を過ぎた回答の削除と送信済みか判断できない通知の `abandon`、ワークスペースの利用終了の消去、状態確認は、Installation のユースケースが、この文書の振る舞い・手順・項目を使って行う。

詳細は `spec/usecases/` が定める。
