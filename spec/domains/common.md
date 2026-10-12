# 共通の型とポート

すべてのドメインが使う型とポートを定める。一覧と置き場所は [index.md](index.md) の「共通の型とポート」にある。

## 集約の ID

```ts
declare const idBrand: unique symbol;
type Id<TName extends string> = string & { readonly [idBrand]: TName };

type WorkspaceId = Id<"Workspace">;
type AccountId = Id<"Account">;
type SessionId = Id<"Session">;
type RecoveryRequestId = Id<"RecoveryRequest">;
type MemberId = Id<"Member">;
type InvitationId = Id<"Invitation">;
type AuditLogEntryId = Id<"AuditLogEntry">;
type TokenId = Id<"Token">;
type PreviewExchangeId = Id<"PreviewExchange">;
type DeviceAuthorizationId = Id<"DeviceAuthorization">;
type ModelId = Id<"Model">;
type FormId = Id<"Form">;
type SchemaApplyJobId = Id<"SchemaApplyJob">;
type ContentId = Id<"Content">;
type RevisionId = Id<"Revision">;
type MediaId = Id<"Media">;
type SubmissionId = Id<"Submission">;
type WebhookEndpointId = Id<"WebhookEndpoint">;
type ChangeNotificationId = Id<"ChangeNotification">;
type BackupRecordId = Id<"BackupRecord">;
type OperationId = Id<"Operation">;
```

- 各 ID は `create(value: string)` だけで作る。空の文字列は `BusinessRuleError`
- 形式は `IdGenerator` が定める（UUID の文字列表現）。ドメインは形式を知らない。入力に受ける ID の扱いは「Clock と IdGenerator」が定める
- 設置の区画の集約の ID は、設置の中で一意。ワークスペースの区画の集約の ID は、その区画の中で一意（区画は [index.md](index.md) の「区画」）。リポジトリの引数は「TransactionalRepository」の「区画」が定める
- `OperationId` は、公開内容を変える操作の一回を指す。集約を識別しない。規則は [index.md](index.md) の「公開内容の整合性と通し番号」が定める
- 区画に一つの集約は、`Workspace` と、ワークスペースに一つの集約。ワークスペースに一つの集約は、`AccessPolicy`、`WorkspaceSchema`、`PublicationLedger`、`ContentRetentionPolicy`、`MailConnection`、`PreviewSettings`、`WebhookEndpoints` の七つを指す。区画が識別する。集約は ID のフィールドを持たず、リポジトリのメソッドは ID を引数に取らない。`Workspace` も、自分の `WorkspaceId` をフィールドに持たない。楽観ロックで保存するポートは「SingletonRepository」を継ぐ。`WebhookEndpointId` は、`WebhookEndpoints` の中の宛先を識別する。七つのうち、ワークスペースの作成が作るものは、各ドメインの「初期構成の宣言」が定める
- ワークスペースの区画の集約と記録は、`workspaceId` のフィールドを持たない。どのワークスペースのものかは、読み書きした区画が決める。ワークスペースの ID が要る場所（ユースケースの出力、区画の外のポートへ渡す値、設置の区画の記録）には、ユースケースが、区画を求めた `WorkspaceId` を渡す。設置の区画の集約と記録（名簿の項目、所属の索引の項目、宣言したトークンの索引の項目、スクリプトの認証の依頼の承認の対象）は、`workspaceId` を持つ
- ほかの集約と一対一の集約は、相手の ID で識別する。`SubmissionNotification` は `SubmissionId`、`MediaDerivatives` と `MediaProfile` は `MediaId`
- 区画の外の対象ごとに一つの集約と、キーで識別する集約は、その値で識別する。`JoiningGuard` は `AccountId`、`PendingFormSettings` はフォームのキー（`formKey`）
- 設置に一つの `Installation` は ID を持たない

## Actor

```ts
type Actor =
  | { readonly kind: "member"; readonly memberId: MemberId }
  | { readonly kind: "appToken"; readonly tokenId: TokenId; readonly issuerMemberId: MemberId }
  | { readonly kind: "memberToken"; readonly tokenId: TokenId; readonly memberId: MemberId }
  | { readonly kind: "operator" }
  | { readonly kind: "system" };

type TokenActor = Extract<Actor, { kind: "appToken" | "memberToken" }>;
```

- `member`: 管理画面を操作するメンバー
- `appToken`: アプリに紐付くトークンを示したアプリ。`issuerMemberId` は、トークンを発行したメンバー
- `memberToken`: メンバーに紐付くトークンを示したスクリプトまたは表示アプリ。`memberId` は、紐付くメンバー
- `operator`: 運用コマンドを実行するホスティング運営者
- `system`: 定期ジョブとイベントの消費者
- `TokenActor` は、トークンを示した実行者だけを受け付ける場所（Schema の、スキーマの版の記録とジョブの実行者）が使う

`Actor` を作る入口と、認可の手続きは [index.md](index.md) の「認可」が定める。

## TokenDesignation

トークンを指す表示の元。アプリに紐付くトークンは名称、メンバーに紐付くトークンは発行の経路で指す。表示の文字列は presentation が決める。Token が値を作り、Schema の `ActorNames`、Content の `Performer`、Audit の `AuditActor` と `TokenTarget`が同じ型で受ける。

```ts
type MemberTokenOrigin = "device" | "preview";

type TokenDesignation =
  | Readonly<{ binding: "app"; name: string }>
  | Readonly<{ binding: "member"; origin: MemberTokenOrigin }>;
```

- `MemberTokenOrigin` は、メンバーに紐付くトークンの発行の経路。`device` はスクリプトの認証、`preview` はプレビューの引き換え
- `name` は、アプリに紐付くトークンの名称の値。名称の規則は Token の `TokenName`（[token.md](token.md)）が定める
- 値を作るのは Token の `Token.designation` だけ。受ける側は、受けた時点の値をそのまま保存し、作り直さない
- 型は `packages/core/src/domain/common/tokenDesignation.ts` に置く

## Permission

操作に必要な権限の語彙。すべてのドメインの認可が使う。権限ごとの意味と行える操作、権限の組（`PermissionSet`）、標準の役割との対応は、Workspace が定める（[workspace.md](workspace.md) の「Permission」「PermissionSet」「StandardRole」）。

```ts
type Permission =
  | "content:edit"
  | "content:publish"
  | "content:delete"
  | "media:publish"
  | "schema:change"
  | "submission:read"
  | "submission:delete"
  | "notification:manage"
  | "intake:manage"
  | "member:manage"
  | "integration:manage"
  | "workspace:manage";
```

- 型は `packages/core/src/domain/common/permission.ts` に置く。`Permission` と `PermissionSet` の使用が依存に当たるかどうかは、[index.md](index.md) の「依存方向」が定める

## EmailAddress

メールアドレス。Identity のアカウント、Workspace の招待の宛先、Form の通知先、`SystemMailer` の宛先が使う。

```ts
declare const emailAddressBrand: unique symbol;
type EmailAddress = string & { readonly [emailAddressBrand]: true };

const EmailAddress: {
  isWellFormed(value: string): boolean;
  create(value: string): EmailAddress;
};
```

- `isWellFormed` は、メールアドレスの形の判定。渡された文字列を、そのまま判定する純粋な関数。メールアドレスの形の規則は、この述語だけが持つ
- 形に合う文字列は、次をすべて満たす。空白を含まない。254 文字以下。`@` をちょうど一つ持つ。`@` の前（ローカル部）が 1 文字以上。`@` の後ろ（ドメイン部）が `.` を含み、`.` で始まらず、`.` で終わらない
- `create` は、前後の空白を除き、小文字にした値を `isWellFormed` で判定し、その値を保持する。形に合わない値は `ValidationError`（理由 `InvalidEmailAddress`）。種類は、どのドメインが `create` を呼んでも `ValidationError` で、各ドメインは別の種類に写さない
- 等価性は、保持する値の一致
- 判定は形だけを確かめる。宛先の所有と到達を確かめない
- 型は `packages/core/src/domain/common/emailAddress.ts` に置く

## FailureHandling

送信失敗の対処の状態。`scenario/index.md` の「未対処の失敗」の規則を定める。ドメインの層では、未対処の規則をこの節だけが定める。送信失敗の状態の中にだけ持つ。Form の `SubmissionNotification` と、Webhook の `WebhookDelivery` が使い、規則は同じ。

```ts
type FailureHandling =
  | { readonly status: "unhandled" }
  | { readonly status: "handled"; readonly by: MemberId; readonly at: Date };

const FailureHandling: {
  unhandled(): FailureHandling;
  markHandled(current: FailureHandling, by: MemberId, now: Date): FailureHandling;
  isUnhandled(current: FailureHandling): boolean;
};
```

- 送信失敗になったとき、`unhandled()` で始まる
- `markHandled` は `handled` を返す。`handled` に対しては、そのまま返す
- 遷移は、`unhandled` から `handled` への一方向
- 再試行は、対処の状態を見ない。未対処の送信失敗も、対処済みの送信失敗も、同じ規則で再試行できる
- 再試行を始めた送信は送信待ちになり、`FailureHandling` を持たない。再試行の前の対処の状態を引き継がない。再試行中の送信は、未対処の件数に数えない
- 再試行の結果が再び送信失敗になると、`unhandled()` で始まる
- 送信成功になった送信は、`FailureHandling` を持たない
- 持つ側の集約が、送信の状態の直和型でこれを表す。送信失敗の変種だけが `FailureHandling` のフィールドを持つ
- フォームの通知が回答の削除で未対処でなくなることは、[form.md](form.md) の `SubmissionRepository` の「回答と通知の対応」で表す
- 表示アプリへの通知の自動の再試行の回数と間隔は、Webhook の `DeliveryPolicy`（[webhook.md](webhook.md)）が定める

確認した回。手動再試行と対処の記録は、メンバーが対象を選んだときに確認した送信の回（確認した回）を添える。回は、手動再試行を受け付けるごとに進む値で、持つ側の集約が定める。今の回が確認した回と違う送信は、確認の後に再試行が始まっている。始まった再試行が進行中でも、終わっていても同じ。その送信は、手動再試行では、先に始まった一件の再試行として扱い、二通目を送らず、受け付けない対象に数えない。対処の記録では、送信失敗でない送信として受け付けない（`scenario/index.md` の「未対処の失敗」）。

## Pagination と PaginationResult

```ts
declare const paginationBrand: unique symbol;
type Pagination = Readonly<{
  page: number;
  limit: number;
}> & { readonly [paginationBrand]: true };

type PaginationResult<T> = Readonly<{
  items: readonly T[];
  count: number;
}>;

const Pagination: {
  readonly maxLimit: 100;
  create(input: Readonly<{ page: number; limit: number }>, maxLimit: number): Pagination;
};
```

- `page` は 1 始まりのページの番号。`limit` は一ページの件数
- `items` は、そのページの要素。`count` は、条件に該当する総件数で、そのページの件数ではない
- `Pagination` は `create` だけが作るブランド型。`create` は、`page` が 1 以上の整数で、`limit` が 1 以上 `maxLimit` 以下の整数であることを確かめる。範囲外の指定は `ValidationError`（理由 `InvalidPagination`）で、項目ごとの理由を `page` と `limit` に持つ
- `Pagination.maxLimit` は、一覧の 1 ページの件数の上限。値は [../scenario/app-integration.md](../scenario/app-integration.md) の APP-12 の「一覧の 1 ページの件数」が定め、技術設計の層では、この定数だけが持つ
- 一覧のユースケースは、利用者が指定したページと件数を `create` に通してから、ポートを呼ぶ。`create` の `maxLimit` には `Pagination.maxLimit` を渡す。一覧のユースケースに共通の規則で、各ユースケースは書き直さない
- `Pagination` を受けるポートのメソッドは、`page` と `limit` の範囲を確かめず、`limit` の上限を持たない
- 型は `packages/core/src/domain/common/pagination.ts` に置く

## BatchTargets

一度の操作で扱う対象の列。重複を除いた、1 件以上、上限以下の列だけを表す。利用者が一度の操作に指定した対象の列を受けるユースケースが使う。使う操作と、操作ごとの上限は、各ドメインが定める。

```ts
declare const batchTargetsBrand: unique symbol;
type BatchTargets<T> = readonly T[] & { readonly [batchTargetsBrand]: true };

const BatchTargets: {
  create<T extends string>(items: readonly T[], max: number): BatchTargets<T>;
  createBy<T>(items: readonly T[], max: number, key: (item: T) => string): BatchTargets<T>;
};
```

- `create` は、同じ値の要素を一つにする。`createBy` は、`key` が同じ文字列を返す要素を一つにする。どちらも、最初に現れた要素を残し、残した要素の順を保つ
- 重複を除いた件数が 0、または `max` を超える列は `ValidationError`（理由 `InvalidTargetCount`）。エラーは、上限の値（`max`）を添える。上限を超える列を切り詰めない
- `max` は、使う側のドメインが定数で持つ。`create` と `createBy` は、`max` が 1 以上の整数であることを前提にする
- ID の列は `create`、ID に `version` などの値を添えた対象の列と、ID の組の列は `createBy` を使う。`key` は、対象を識別する部分（ID、または ID の組）から作る
- 対象の列を受けるユースケースは、利用者が指定した列をこれに通してから、ポートを呼ぶ。重複と件数の範囲の扱いは、この節だけが定める。各ドメインは書き直さない
- 対象を指定しなくてよい入力（主な対象に加える、任意の対象）は、使う側が「指定なし」と「`BatchTargets`」の直和で受ける。指定なしは `BatchTargets` で表さず、`create` と `createBy` に空の列を渡さない
- 型は `packages/core/src/domain/common/batchTargets.ts` に置く

## PublishedChangeNotice

公開内容の変更の一件を、表示アプリへの通知に載せる形で表す値。Content が作って公開の変更の記録に保存し、Webhook が同じ型で受けて `ChangeNotification` に持つ（[index.md](index.md) の「公開内容の整合性と通し番号」）。

```ts
type PublicationChangeKind = "published" | "updated" | "unpublished";

type UniqueValueChange = Readonly<{
  field: string;
  before: string | null;
  after: string | null;
}>;

type PublishedChangeNotice = Readonly<{
  contentId: ContentId;
  model: string;
  change: PublicationChangeKind;
  uniqueValues: readonly UniqueValueChange[];
}>;

type PublishedReferrer = Readonly<{
  contentId: ContentId;
  model: string;
}>;

type PublishedChangeList = Readonly<{
  changes: readonly PublishedChangeNotice[];
  referrers: readonly PublishedReferrer[];
}>;
```

- `change` は、未公開のコンテンツの公開（`published`）、公開内容の置き換え（`updated`）、公開内容がなくなること（`unpublished`）のいずれか
- `model` は、そのコンテンツのモデルのキー。`PublishedChangeNotice` では変更したコンテンツの、`PublishedReferrer` では参照元のコンテンツのモデルを指す。`field` は、一意な項目のキー。どちらも文字列で持ち、Schema の型を使わない
- `uniqueValues` は、変更の前後で異なる一意な値。`null` は、未設定または公開内容なし
- `PublishedChangeList` は、一つの操作の変更の全体。`changes` は 1 件以上で、同じコンテンツは一度だけ現れる。`referrers` は、変更したコンテンツを公開内容から参照するコンテンツ。コンテンツID とモデルのキーの組で持ち、`changes` のコンテンツを含まず、同じコンテンツは一度だけ現れる
- 公開内容の値、下書き、非公開の内容を持たない
- 値の作り方（`PublicationChange` からの変換、参照元の求め方）は Content が、通知の内容への写し方は Webhook が定める
- 型は `packages/core/src/domain/common/publishedChange.ts` に置く

## TransactionalRepository

```ts
declare const versionBrand: unique symbol;
type ExpectedVersion<T> = number & { readonly [versionBrand]: T };
type Versioned<T> = { readonly entity: T; readonly expectedVersion: ExpectedVersion<T> };

interface TransactionalRepository<TEntity, TId> {
  insert(entity: TEntity): Promise<void>;
  findById(id: TId): Promise<Versioned<TEntity> | null>;
  save(entity: TEntity, expectedVersion: ExpectedVersion<TEntity>): Promise<void>;
  delete(id: TId, expectedVersion: ExpectedVersion<TEntity>): Promise<void>;
}
```

- 一意性・整合性の担保: `insert` は、同じ区画に同じ ID の集約が既にある場合に `ConflictError`（理由 `ALREADY_EXISTS`）を返す。追記だけの集約の追記と、複数件の `insert` も同じ。ポートが担保する。同じ ID の集約がある `insert` が、ポートが加える一意性・参照整合性の違反にも当たる場合も、理由は `ALREADY_EXISTS`。別の区画の同じ ID を、ポートは確かめない。各集約のポートは、これに加える一意性・参照整合性を自分のファイルで定める。ポートが担保する一意性・参照整合性の範囲は、そのリポジトリの区画の中に限る。区画をまたぐ参照先の存在は、呼び出し側が手順の順序で担保する（[index.md](index.md) の「区画」）
- 並行性: 集約は、楽観ロックの番号（`version`）を持つ。この文書と [index.md](index.md) は、この番号を `version` と書き、「版」と書かない。`findById` は、読んだ集約の `version` を `expectedVersion` として返す。`save` と `delete` は、渡された `expectedVersion` が保存済みの `version` と違う場合に `ConflictError`（理由 `STALE_VERSION`）を返す。`version` を進めるのは、集約の振る舞い。進める振る舞いは、各集約が定める。`save` は、渡された集約の `version` をそのまま保存し、自分では進めない。`version` を進めない集約を `save` した後の `findById` は、同じ `expectedVersion` を返す
- 可視性: UnitOfWork の外の読み取りは、確定済みの書き込みを反映する。UnitOfWork の中で、その UnitOfWork が書いた集約を読み直した結果は約束しない
- 読み取りと確定: UnitOfWork の中の読み取りは、読んだ時点で確定済みの書き込みを反映する。読んだ内容が、その UnitOfWork の確定の時点まで変わらないことを、ポートは約束しない。読んだ後に、並行した書き込み（別の集約の `insert`、`save`、`delete` を含む）が確定していても、その UnitOfWork は確定し得る。並行した書き込みとの両立を禁じるのは、書き込みに渡した `expectedVersion`、`WriteScope` の条件、各ポートが契約に明記した確定の時点の拒否（一意性、参照整合性、公開内容の整合性）、復旧の置き換えとの排他（[installation.md](installation.md) の「取得・置き換え・更新の排他」）の四つだけ。読んだ内容に依る判定を確定の時点まで保つユースケースは、判定の元の集約を、読んだ `expectedVersion` で `save` するか、条件を `WriteScope` で表す
- 区画: リポジトリは、一つの区画に結び付く。設置の区画のリポジトリは設置に一つ、ワークスペースの区画のリポジトリはワークスペースごとに一つある。ワークスペースの区画のリポジトリは、結び付いたワークスペースの集約と記録だけを読み書きする。ワークスペースの区画に結び付くリポジトリのメソッドは、ワークスペースの ID を引数に取らない。区画は、`UnitOfWorkProvider.run` に渡す `WriteScope` と、`PartitionReader.workspace(workspaceId)` が決める。ワークスペースの区画の集約と記録は `workspaceId` を持たない（「集約の ID」）。どの書き込み（`insert`、`save`、`append`、複数件を受けるメソッド）も、渡した値を、結び付いた区画に書く。別の区画の値を渡す呼び出しは、型に無い
- メソッドの名前は、区画を名前に含めない。区画の全件を対象にするメソッドは、対象を名前に持たない形にする（`findPage(filter, pagination)`、`count()`、`scan(after, limit)`、`deleteAll(limit)`）。ほかの条件で引くメソッドは、その条件だけを名前に持つ（`findByAccount(accountId)`）
- 設置の区画のポートと、区画の外・区画に結び付かないポートは、対象のワークスペースを引数で受ける（`WorkspaceDirectory`、`MembershipIndex`、`DeclaredTokenIndex`、`MediaStorage` など）
- メソッドの分類: 各ポートのメソッドは、読み取り、書き込み、UnitOfWork の外の記録のいずれか。`findById` と `find` は読み取り、`insert`、`save`、`delete` は書き込み。各ポートが加えるメソッドの分類は、そのポートの契約が定める。UnitOfWork の外の記録は、楽観ロックを使わず、UnitOfWork の確定と全部成功か全部反映されないかを共にしない書き込み（トークンの最終利用日時の記録）
- メソッドの分類ごとに、どこで得たリポジトリが持つかは、「UnitOfWorkProvider」の `WriteSide` と `ReadSide` の表が定める。型で分けるので、UnitOfWork の外の書き込みと、UnitOfWork の中の UnitOfWork の外の記録は書けない
- 読み取りだけのユースケースは UnitOfWork を開かない
- `version` と ID を持つ集約のポートは、これを継いで、読み取りのクエリと書き込みを加える。呼び手が無いメソッドがある集約のポートは、そのメソッドを除いた形を継ぐ（一件ずつ削除する呼び手が無いポートは `Omit<TransactionalRepository<TEntity, TId>, "delete">`、一件ずつ作る呼び手が無いポートは `Omit<TransactionalRepository<TEntity, TId>, "insert">`）
- 区画に一つの集約を楽観ロックで保存するポートは、「SingletonRepository」を継ぐ
- どちらも継がないポート（追記だけの集約、`version` を持たない集約、集約に数えない記録、設置に一つの `Installation`）は、持つメソッドを自分のインターフェースにすべて宣言し、契約を自分のファイルで定める。すべてのリポジトリのメソッドに適用する下の規則は、継がないポートにも適用する

すべてのリポジトリのメソッドに、次の規則を適用する。各ポートは、これと違う場合だけ自分のファイルで定める。

- 存在しない対象: `save` と `delete` は、対象が存在しない場合に `ConflictError`（理由 `STALE_VERSION`）を返す。`findById` は `null` を返す
- 参照先の存在: 各ポートが契約に明記しない参照先（ワークスペース、アカウント、発行者など）の存在を、ポートは担保しない。呼び出し側が確かめる。同じ区画の参照先の存在を契約に明記したポートは、確定の時点で担保し、参照先が無い書き込みを `ConflictError` で拒否する。理由は、そのポートが定める。参照先の削除と並行し得る書き込みは、この形で担保する
- 競合と違反が現れる時点: `ConflictError` は、メソッドの呼び出しか、UnitOfWork の確定のどちらかで現れる。どちらの場合も、その UnitOfWork の書き込みは一つも反映されない。ワークスペースの条件（「WriteScope」）を満たさない場合も同じ。呼び出し側は、`run` が `ConflictError` を投げることだけを前提にする
- やり直さない: アダプターとユースケースは、失敗した書き込みと、`ConflictError` で確定しなかった UnitOfWork を、読み直してやり直さない。エラーをそのまま返し、要求した側が送り直す。イベントの消費者と定期ジョブの、失敗の後の扱いは、[index.md](index.md) の「イベントの消費者と定期ジョブ」が定める。そのまま返さずに別の結果へ写す競合は、[index.md](index.md) の「競合の扱い」が定める。写す場合も、確定をやり直さない
- 件数の指定: `limit` を数で受けるメソッドの `limit` は、1 以上、各メソッドが定める上限以下の整数。一度に渡せる件数に上限があるメソッドは、上限を超える入力を切り詰めない。範囲外の指定と、整数でない指定は `SystemError`（理由 `INVALID_QUERY`）を返す。ページの指定は `Pagination` で受け、ポートは範囲を確かめない（「Pagination と PaginationResult」）
- 対象の集合の指定: 対象を ID の集合、または ID の組の列で受けるメソッドは、重複した ID と、同じ組を、一つとして扱う。件数の上限は、重複を除いた数に適用する。存在しない ID は結果に含めない。ID に値を添えた対象の列（`{ id, expectedVersion }` や、保存する集約と `expectedVersion` の組の列）を受けるメソッドは、同じ ID を二度含む列に `SystemError`（理由 `INVALID_QUERY`）を返し、何も書かない
- 区画が無いワークスペース: 区画が無いワークスペース（作成の前、存在しない ID）のリポジトリの読み取りは、一覧に空の結果と件数 0 を、`findById` と `find` に `null` を返す。利用終了が完了したワークスペースの区画は、`closed` の `Workspace` だけを持つ（[workspace.md](workspace.md) の「利用終了の消去」）。その区画では、`WorkspaceRepository.find` が `closed` の `Workspace` を返し、ほかのリポジトリの読み取りは、区画が無いワークスペースと同じ結果を返す
- 絞り込みの集合: 値の集合を取る絞り込みに空の集合を渡すと、どれにも一致しない（0 件）。絞り込まない場合は、その条件を指定しない
- 順序: ID の順は、文字列のコードポイントの昇順。日時が同じものの順は、ID の昇順
- 件数を区切る削除（`deleteAll`、`deleteLapsed`、`SubmissionRepository` と `EditingPresenceStore` の `deleteExpired`、`WorkspaceContext.deleteEvents` など）: 最大 `limit` 件を削除し、削除した件数を返す。`limit` の上限は `BoundedDeletion.maxLimit`。返した件数が `limit` より少なければ、対象は残っていない。各ポートは、対象の条件だけを書く。ユースケースは件数の入力を持たず、`limit` に `BoundedDeletion.maxLimit` を渡し、返した件数が `limit` より少なくなるまで繰り返す。UnitOfWork に属するポートは、一回の呼び出しごとに一つの UnitOfWork で確定する。UnitOfWork に属さないポート（「UnitOfWorkProvider」）は、UnitOfWork を開かずに呼び、一回の呼び出しの完了で反映する。件数を区切る削除のポートは、ワークスペースの状態を確かめない。ほかの文書は、上限をこの名前で、進め方をこの箇条で参照する
- 日時で区切る削除: 対象は、各ポートが定める日時が、渡された日時以前（同じ日時を含む）のもの。一度きりの値の削除（`deleteLapsed`）は「期限つきの一度きりの値」が定める
- 区画の消去（`deleteAll(limit)`）: ワークスペースの区画のリポジトリが持ち、結び付いた区画の、そのリポジトリの集約と記録を削除する。ポートは、ワークスペースの状態を確かめない。区画が `closing` のワークスペースに限ることは、呼び出し側（利用終了のユースケース）が確かめる
- 保存先の障害と、デコードできない保存値は `SystemError` を返す。これはアダプターごとの検証の対象で、ポート適合テストの対象にしない
- 並行性のポート適合テストは、二つの UnitOfWork を並行に実行し、契約が両立を禁じる場合に一方だけが確定することを確かめる。特定の割り込みの順序を再現する検証は、アダプターごとの検証の対象にする
- UnitOfWork の中で、その UnitOfWork が書いた集約を読み直した結果は、ポート適合テストの対象にしない

```ts
const BoundedDeletion: { readonly maxLimit: 1000 };
```

外部の相手を持つポート（`SystemMailer`、`MailSender`、`WebhookSender`、`MediaStorage`、`ImageProcessor`、`BackupStore`、`RecoveryJournal`、`InstallationEnvironment`）のポート適合テストは、応答と状態を指定できる試験用の相手に対して実行する。試験用の相手で再現できない約束（容量の不足、保管先を直接壊した場合の検出、ログへ出さないこと）は、アダプターごとの検証の対象で、ポート適合テストの対象にしない。

## SingletonRepository

区画に一つの集約を、楽観ロックで保存するポートの基底。`Workspace` と、ワークスペースに一つの集約のうち楽観ロックで保存するもののポートが継ぐ。

```ts
interface SingletonRepository<TEntity> {
  insert(entity: TEntity): Promise<void>;
  find(): Promise<Versioned<TEntity> | null>;
  save(entity: TEntity, expectedVersion: ExpectedVersion<TEntity>): Promise<void>;
}
```

- `insert` は、結び付いた区画にその集約が既にある場合に `ConflictError`（理由 `ALREADY_EXISTS`）を返す。区画に一つであることを、ポートが担保する
- `find` は、結び付いた区画の集約を返す。無ければ `null`
- `save` は、`find` が返した `expectedVersion` が保存済みの `version` と違う場合に `ConflictError`（理由 `STALE_VERSION`）を返す
- 削除は、各ポートが自分で定める。一件の削除は `delete(expectedVersion)`、利用終了の消去は `deleteAll(limit)`
- 可視性、メソッドの分類、すべてのリポジトリのメソッドに適用する規則は、「TransactionalRepository」と同じ

## WriteScope

一つの UnitOfWork が書く区画と、ワークスペースの区画で確定の時点に満たすべき条件。区画の定義と、集約とポートの置き場所は [index.md](index.md) の「区画」が定める。ユースケースは、`UnitOfWorkProvider.run` に `WriteScope` を一つ渡す。

```ts
type WriteScope =
  | { readonly kind: "installation" }
  | { readonly kind: "workspace"; readonly workspaceId: WorkspaceId; readonly condition: WorkspaceCondition };

type WorkspaceCondition =
  | { readonly kind: "none" }
  | { readonly kind: "active" }
  | { readonly kind: "stableSchema"; readonly schemaVersion: number }
  | { readonly kind: "schemaVersion"; readonly schemaVersion: number };
```

- `installation` は、設置の区画を書く。条件を持たない
- `workspace` は、`workspaceId` のワークスペースの区画を書く。`condition` は、確定の時点で、その区画の `Workspace` と `WorkspaceSchema` が満たすべき条件

| `condition` | 確定の時点で確かめること | 満たさない場合 |
| --- | --- | --- |
| `none` | なし | — |
| `active` | `Workspace` が `active` | `ConflictError`（理由 `WORKSPACE_NOT_ACTIVE`）。`Workspace` が存在しない場合も同じ |
| `stableSchema` | `Workspace` が `active`。`WorkspaceSchema` のスキーマの版が `schemaVersion` と同じで、適用中でない | `Workspace` が `active` でない、または存在しない場合は `ConflictError`（理由 `WORKSPACE_NOT_ACTIVE`）。スキーマの版が違う、適用中、または `WorkspaceSchema` が存在しない場合は `ConflictError`（理由 `SCHEMA_CHANGED`） |
| `schemaVersion` | `Workspace` が `active`。`WorkspaceSchema` のスキーマの版が `schemaVersion` と同じ。適用中かどうかを見ない | `Workspace` が `active` でない、または存在しない場合は `ConflictError`（理由 `WORKSPACE_NOT_ACTIVE`）。スキーマの版が違う、または `WorkspaceSchema` が存在しない場合は `ConflictError`（理由 `SCHEMA_CHANGED`） |

- 条件を満たさない UnitOfWork は、どの書き込みもイベントも確定しない。`run` が `ConflictError` を投げる
- `Workspace` とスキーマの両方を満たさない場合の理由は `WORKSPACE_NOT_ACTIVE`
- 条件を満たさない UnitOfWork の理由は、同じ UnitOfWork の書き込みが、楽観ロックの競合と、ポートが契約に明記した拒否（一意性、参照整合性、公開内容の整合性）のどれに当たる場合も、条件の理由（上の表）にする。`fn` の中の書き込みが `ConflictError` を投げた場合も、`run` は条件を確かめ、満たさなければ、条件の理由の `ConflictError` を投げる
- 条件は、`fn` が書き込みを行わない場合と、`fn` が `Workspace` と `WorkspaceSchema` を読まない場合にも確かめる
- 条件は、集約を書き換えない。`Workspace` と `WorkspaceSchema` の `version` を見ず、進めない
- 条件が見るのは、同じ区画の `Workspace` と `WorkspaceSchema` だけ。設置の区画の記録（ワークスペースの名簿）と、ほかのワークスペースの区画を見ない。名簿で開設済みでないワークスペースを受け付けないことは、入口の認可が担う（[index.md](index.md) の「利用できるワークスペース」）
- 同じ条件の UnitOfWork どうしは競合しない。条件を変える書き込みの確定とだけ両立しない。`active` は、`Workspace` を `active` でなくする確定（当たる遷移は [workspace.md](workspace.md) の「利用中の条件」）。`stableSchema` は、それに加えて、`WorkspaceSchema` を適用中にする確定と、スキーマの版を進める確定。`schemaVersion` は、`active` と両立しない確定に加えて、スキーマの版を進める確定。適用中にする確定と、版を進める確定に当たる操作は、[schema.md](schema.md) の「WriteScope とスキーマ」が定める。並行した二つは、条件の UnitOfWork が先に確定するか、条件の UnitOfWork が `ConflictError` になるかのどちらかになる
- `stableSchema` と `schemaVersion` に渡すスキーマの版は、ユースケースが書く値を検証した前提のスキーマの版。どの版を前提にするかは、各ユースケースのトランザクション境界が定める
- `WriteScope` は、その UnitOfWork の書き込みの性質から決まる。各ユースケースが、トランザクション境界に、UnitOfWork ごとに一つ書く。ユースケースごとの割り当ては、各ユースケースのトランザクション境界と、ユースケースが呼ぶ手順を公開するドメインの節（各ドメインの「利用再開前の処理」「利用終了の消去」と、この文書の「掃除」）が持つ

| `WriteScope` | 書き込みの性質 |
| --- | --- |
| `installation` | 設置の区画の集約と記録を書く |
| `workspace` の `stableSchema` | 読んだスキーマの構造に従う値を書き、スキーマの反映の適用中は確定させない |
| `workspace` の `schemaVersion` | 読んだスキーマの定義に従う値を書き、スキーマの反映の適用中も確定させる |
| `workspace` の `active` | スキーマに従う値を書かず、`Workspace` が `active` の区画にだけ確定させる |
| `workspace` の `none` | `Workspace` の状態によらず確定させる。`Workspace` が無い区画と、`closing`、`closed` の区画にも確定する |

## UnitOfWorkProvider

```ts
type InstallationRepositories = {
  // Identity
  accountRepository: AccountRepository;
  sessionRepository: SessionRepository;
  recoveryRequestRepository: RecoveryRequestRepository;
  // Workspace
  workspaceDirectory: WorkspaceDirectory;
  membershipIndex: MembershipIndex;
  // Token
  deviceAuthorizationRepository: DeviceAuthorizationRepository;
  declaredTokenIndex: DeclaredTokenIndex;
  // Installation
  installationRepository: InstallationRepository;
};

type WorkspaceRepositories = {
  // Workspace
  workspaceRepository: WorkspaceRepository;
  memberRepository: MemberRepository;
  invitationRepository: InvitationRepository;
  accessPolicyRepository: AccessPolicyRepository;
  joiningGuardRepository: JoiningGuardRepository;
  // Audit
  auditLogRepository: AuditLogRepository;
  // Token
  tokenRepository: TokenRepository;
  previewExchangeRepository: PreviewExchangeRepository;
  previewSettingsRepository: PreviewSettingsRepository;
  // Schema
  workspaceSchemaRepository: WorkspaceSchemaRepository;
  schemaVersionRecordRepository: SchemaVersionRecordRepository;
  schemaApplyJobRepository: SchemaApplyJobRepository;
  // Media
  mediaRepository: MediaRepository;
  mediaUsageRepository: MediaUsageRepository;
  mediaDerivativesRepository: MediaDerivativesRepository;
  mediaProfileRepository: MediaProfileRepository;
  storageReservationRepository: StorageReservationRepository;
  objectRemovalRepository: ObjectRemovalRepository;
  // Content
  contentRepository: ContentRepository;
  revisionRepository: RevisionRepository;
  publicationLedgerRepository: PublicationLedgerRepository;
  contentRetentionPolicyRepository: ContentRetentionPolicyRepository;
  contentStagingArea: ContentStagingArea;
  // Form
  formSettingsRepository: FormSettingsRepository;
  pendingFormSettingsRepository: PendingFormSettingsRepository;
  mailConnectionRepository: MailConnectionRepository;
  submissionRepository: SubmissionRepository;
  submissionNotificationRepository: SubmissionNotificationRepository;
  // Webhook
  webhookEndpointsRepository: WebhookEndpointsRepository;
  changeNotificationRepository: ChangeNotificationRepository;
  webhookDeliveryRepository: WebhookDeliveryRepository;
};

type InstallationContext = WriteSide<InstallationRepositories>;
type WorkspaceContext = WriteSide<WorkspaceRepositories> & {
  collectEvents(events: readonly EventDraft[]): void;
  deleteEvents(limit: number): Promise<number>;
};

type ContextOf<S extends WriteScope> = S extends { kind: "installation" }
  ? InstallationContext
  : WorkspaceContext;

interface UnitOfWorkProvider {
  run<S extends WriteScope, T>(scope: S, fn: (context: ContextOf<S>) => Promise<T>): Promise<T>;
}

interface PartitionReader {
  installation(): InstallationReader;
  workspace(workspaceId: WorkspaceId): WorkspaceReader;
}

type InstallationReader = ReadSide<InstallationRepositories>;
type WorkspaceReader = ReadSide<WorkspaceRepositories> & {
  contentReferenceQuery: ContentReferenceQuery;
  quarantinedEventQuery: QuarantinedEventQuery;
};
```

`WriteSide<T>` と `ReadSide<T>` は、`T` の各リポジトリを、メソッドの分類（「TransactionalRepository」）で絞った型にする。

| 型 | 各リポジトリが持つメソッド | 得る場所 |
| --- | --- | --- |
| `WriteSide<T>` | 読み取りと書き込み | `UnitOfWorkProvider.run` の `fn` が受ける文脈 |
| `ReadSide<T>` | 読み取りと、UnitOfWork の外の記録 | `PartitionReader` |

文脈。

- `fn` が受ける文脈の型は、`scope` の区画で決まる。`installation` は `InstallationContext`、`workspace` は `WorkspaceContext`。文脈は、その区画のリポジトリだけを公開する。区画の外のリポジトリへの書き込みは、型が受け付けない
- 文脈のリポジトリは、各ドメインのファイルが定めるポートを、上の表の `WriteSide` で絞ったもの。`WorkspaceContext` のリポジトリは、`scope.workspaceId` のワークスペースの区画に結び付く
- `workspaceDirectory` は、ワークスペースの名簿のポート、`membershipIndex` は、所属の索引のポート（どちらも [workspace.md](workspace.md)）。`declaredTokenIndex` は、宣言したトークンの索引のポート（[token.md](token.md) の「DeclaredTokenIndex」）。三つとも、集約に数えない記録を保存する
- ドメインイベントを記録できるのは `WorkspaceContext` だけ。設置の区画の UnitOfWork は、イベントを記録しない
- `deleteEvents` は、結び付いた区画が記録したドメインイベントを、配送の状態（未配送、再配送を待つもの、隔離されたもの）によらず、最大 `limit` 件削除し、削除した件数を返す。件数を区切る削除（「TransactionalRepository」）で、書き込みの一回に数える。削除は、UnitOfWork の確定とともに反映される。削除したイベントは、配送の対象と隔離から外れ、`QuarantinedEventQuery.count` に数えない。削除の前に配送を始めたイベントは、消費者に届き得る。届いたイベントの扱いは、[index.md](index.md) の「イベントの消費者と定期ジョブ」に従う。同じ UnitOfWork の `collectEvents` に渡したイベントは、対象にしない。ワークスペースの状態を、ポートは確かめない。呼ぶのは、ワークスペースの利用終了の区画の消去だけ（[index.md](index.md) の「区画をまたぐ手順」）
- 一つの UnitOfWork は、一つの区画だけを書く。二つの区画を書く操作の進め方は、[index.md](index.md) の「区画をまたぐ手順」が定める

UnitOfWork の外の読み取り。

- `PartitionReader` は、UnitOfWork の外のリポジトリを区画ごとに返す。`installation()` は設置の区画、`workspace(workspaceId)` はそのワークスペースの区画のリポジトリを返す
- `WorkspaceReader` は、リポジトリに加えて、読み取りだけの `ContentReferenceQuery`（[content.md](content.md)）と `QuarantinedEventQuery` を公開する。どちらも、そのワークスペースの区画に結び付く
- `PartitionReader` の呼び出しは、区画への接続を行わない。存在しないワークスペースの ID にも `WorkspaceReader` を返し、その読み取りは「TransactionalRepository」の「区画が無いワークスペース」の規則に従う
- `fn` の中からも呼べる。ネストに当たらない。ユースケースは、UnitOfWork が書く区画と別の区画の読み取り（ワークスペースの区画の UnitOfWork の中で読むアカウントなど）に使う
- 二つの区画の読み取りは、同じ時点の状態を約束しない。それぞれが、読んだ時点で確定済みの書き込みを反映する
- 区画の外のポートと、区画に結び付かないポート（[index.md](index.md) の「集約とポートの置き場所」）は、UnitOfWork に属さない。文脈と `PartitionReader` は公開せず、ユースケースが直接受け取る。`fn` の中で呼んでも、UnitOfWork と確定を共にしない。呼び出しの結果は、UnitOfWork が確定しなくても残る

確定。

- `fn` が正常に終わり、`scope` の条件を満たすと、`fn` の中の全書き込みと `collectEvents` に渡したイベントを確定する。全部成功するか、全部反映されないかのどちらかになる
- `fn` が例外を投げると、何も反映せず、`run` はその例外をそのまま投げる。条件を満たさない UnitOfWork の理由は「WriteScope」に従う。確定の時点の競合・一意性・参照整合性の違反と、条件を満たさないことは、`ConflictError` として `run` が投げる。復旧の置き換えと重なった UnitOfWork の結果は、[installation.md](installation.md) の「取得・置き換え・更新の排他」が定める。`run` は、確定しなかった UnitOfWork をやり直さない（「TransactionalRepository」の「やり直さない」）
- ネストしない。`fn` の中で `run` を呼ぶと、区画が同じでも違っても `SystemError`（理由 `NESTED_UNIT_OF_WORK`）を投げる。外側の `fn` がこれを捕まえて正常に終わっても、外側の `run` は同じ `SystemError` を投げ、何も反映しない
- 一つの UnitOfWork は、同じ集約を一度だけ書く。件数を区切る削除の呼び方は「TransactionalRepository」の「件数を区切る削除」に従う。ユースケースがこれを守る
- UnitOfWork の中の読み取りの可視性と、確定の時点で確かめるものは、「TransactionalRepository」に従う。ユースケースは、読み取りを書き込みの前に済ませ、書き込む値を、書き込みの前に読んだエンティティから求める
- `collectEvents` に渡したイベントは、確定とともに配送の対象になる。確定しなかった UnitOfWork のイベントは配送されない
- イベントの配送のポート適合テストは、確定した UnitOfWork のイベントが試験用の消費者に届くことと、確定しなかった UnitOfWork のイベントが届かないことで確かめる
- アダプターは、一つの UnitOfWork を複数の確定に分けない

書き込みの量。

- 量は、一つの UnitOfWork の中の、書き込みの回数で数える。文脈のリポジトリの書き込みのメソッドの呼び出し一回と、`collectEvents` に渡したイベント一件を、それぞれ一回に数える。読み取りは数えない
- 一回の呼び出しが書く集約と記録の数は数えない。複数件を受けるメソッド（`insertMany`、`saveMany`、`deleteMany`、`promote` など）と、件数を区切る削除は、一回の呼び出しで一回。一回の呼び出しが受ける件数の上限は、各メソッドの契約が定める
- アダプターが確定を約束する回数は、ワークスペースの区画の UnitOfWork で `UnitOfWorkCapacity.workspaceWrites` 回、設置の区画の UnitOfWork で `UnitOfWorkCapacity.installationWrites` 回。ほかの文書は、回数をこの名前で参照する。この回数までの UnitOfWork を確定できないアダプターは、契約を満たさない。原子的に確定する手段が一回の書き込みに限られる実行基盤のアダプターは、この回数が一回の書き込みに収まる保存の形を取る
- この回数を超える UnitOfWork の確定を、ポートは約束しない。ユースケースは、一つの UnitOfWork の書き込みを、この回数までに収める
- 対象の件数に比例して書く箇所のうち、この回数に収まらない箇所は、複数件を受ける一括のメソッドにする。一括操作の上限（`BatchTargets` の `max`）の対象を扱う UnitOfWork が、この回数に収まる。収まる箇所は、件数ぶんの呼び出しでよい
- 量が対象の件数に比例し、上限を持たない書き込み（スキーマの反映の変換、保持方針による自動整理、ワークスペースの区画のデータの消去）は、件数を区切って複数の UnitOfWork に分けるか、Content の準備領域（[content.md](content.md)）を使う
- ほかの書き込みと同じ確定で行う必要があり、件数で区切れない書き込みは、契約が件数の上限を持たないと定めた一括のメソッドで行う。そのメソッドは、件数によらず一回の呼び出しで確定し、書き込みの一回に数える。当たるメソッドは、各ポートの契約が定める

```ts
const UnitOfWorkCapacity: {
  readonly workspaceWrites: 300;
  readonly installationWrites: 50;
};
```

## ProtectedSecret

`SecretCipher` で保護した秘密情報。秘密情報を持つ集約と値オブジェクトが使うドメインの型。

```ts
declare const protectedBrand: unique symbol;
type ProtectedSecret = string & { readonly [protectedBrand]: true };
```

- 各ドメインは、`ProtectedSecret` を自分のブランド型（`ProtectedMailSecret` など）に包んで持つ。保護していない秘密情報を、集約・イベント・操作履歴・ログに持たせない
- 型は `packages/core/src/domain/common/protectedSecret.ts` に置く

## SecretCipher

`ProtectedSecret` は、「ProtectedSecret」の型を使う。

```ts
interface SecretCipher {
  protect(plain: string): Promise<ProtectedSecret>;
  reveal(secret: ProtectedSecret): Promise<string>;
  currentKeyId(): Promise<string>;
}
```

- 用途は、後で元の値が要る秘密情報。どの値を保護して保存するかは、その値を持つ集約のファイルが定める。期限つきの一度きりの値の保存の規則は「期限つきの一度きりの値」が定める
- `protect` は、同じ値から毎回違う `ProtectedSecret` を返してよい。`reveal(protect(x))` は `x` を返す
- `reveal` は、保護に使った鍵を使えない場合と、`protect` が返した値でない場合に `SystemError`（理由 `SECRET_UNAVAILABLE`）を返す
- `currentKeyId` は、保護に使う鍵を識別する値を返す。バックアップからの復旧で、秘密情報を戻せるかの照合に使う。保護に使う鍵を使えない構成（鍵が無い、読めない）では、`protect`、`reveal`、`currentKeyId` のどれも `SystemError`（理由 `SECRET_UNAVAILABLE`）を返す

## Clock と IdGenerator

```ts
interface Clock {
  now(): Date;
}

declare const generatedIdBrand: unique symbol;
type GeneratedId = string & { readonly [generatedIdBrand]: true };

interface IdGenerator {
  generate(): GeneratedId;
  parse(value: string): GeneratedId;
}
```

- `GeneratedId` は、`IdGenerator` の形式に合う文字列。`generate` と `parse` だけが作る
- `generate` は、呼ぶたびに違う値を返す。`now` は、逆行しないことを約束しない
- 入力に ID を受けるユースケースは、その ID に `GeneratedId` を要求し、ID を作らない
- やり直しで同じ ID を送り直す作成の ID（コンテンツID、ジョブの ID、ワークスペースの ID）は、呼び出し側が指定する。入口が、境界で `parse` に通す。形式に合わない ID は、入口が `ValidationError` として拒否し、ユースケースに届かない。ユースケースは、この ID の形式のエラーを持たない。コンテンツID とワークスペースの ID の指定が無い要求では、入口が `generate` で作って渡す。ジョブの ID は省けない
- 利用者が要求に書いた既存の対象の ID（一括操作の対象の列の要素を含む）は、入口が、境界で空でないことを確かめ、空の文字列を `ValidationError`（理由 `InvalidId`）として拒否する。各 ID の `create` が返す `BusinessRuleError` は、利用者の入力に由来しない ID だけに当たる
- 操作の ID は、呼び出し側が指定しない。作る主体は [index.md](index.md) の「公開内容の整合性と通し番号」が定める
- `parse` は、形式に合う値を `GeneratedId` としてそのまま返し、形式に合わない値に `ValidationError`（理由 `InvalidId`）を返す。形式は UUID の小文字の標準の文字列表現（8-4-4-4-12 桁の 16 進）で、バージョンによらず受け付ける（`packages/client` が名前から求める ID を含む）。大文字を含む表記は形式に合わない。呼び出し側が指定する ID と、アダプターが保存先から読んだ ID に使う

## ドメインイベント

イベントの型とドラフトの作り方は `docs/backend_implementation_example.md` に従う（`DomainEventBase`、`EventDraft`、`WithEventDrafts`）。集約の振る舞いは `WithEventDrafts` を返し、ユースケースが `collectEvents` に渡す。判定の直和を返す振る舞いは、確定する変種が `WithEventDrafts` を持つ。確定しない変種（拒否、変更なし）は、ドラフトを持たない。

- 配送は一回以上で、順序を保証しない。消費者は、同じイベントが再び届いても、前の消費が確定した書き込みを重ねず、変えない。前の消費が何も書かずに終えたイベントを、再び届いた時点の状態で処理し直すかどうかは、消費者のドメインまたはユースケースが定める
- 確定したイベントは、消費者が処理を終えるまで配送の対象に残る。配送を繰り返しても処理できず隔離されたイベントは、ホスティング運営者が運用手順で配送し直す。隔離されたイベントの件数は `QuarantinedEventQuery` が返す。バックアップと復旧でのイベントの扱いは [installation.md](installation.md) の「BackupStore」が、利用終了でのイベントの消去は「UnitOfWorkProvider」の `deleteEvents` が定める
- イベントは、秘密情報、回答の入力内容、非公開のコンテンツの内容を持たない。対象の ID を持ち、消費者が集約と記録を読む
- イベントは、ワークスペースの区画の UnitOfWork だけが記録する。イベントは、記録した区画に属する
- ドラフト（`EventDraft`）は、イベントの ID と `workspaceId` を持たない。ペイロードも `workspaceId` を持たない。集約の振る舞いは、ワークスペースの ID を知らずにドラフトを作る
- `collectEvents` が、UnitOfWork の `scope.workspaceId` を、記録するイベントの `workspaceId` に付ける。消費者が受けるイベント（`DomainEventBase`）は、ペイロードとは別の項目として `workspaceId` を持つ。消費者は、そのワークスペースの区画を読み書きする
- `aggregateId` は、イベントを発行した集約の、区画の中の ID。区画に一つの集約（「集約の ID」）が発行するイベントのドラフトは `aggregateId` を持たず、`collectEvents` が `workspaceId` を入れる
- ペイロードは大きさの上限を持つ。JSON にした UTF-8 のバイト列で 65,536 バイト以下（`EventPayloadLimit.maxBytes`）。上限を適用するのは `collectEvents` だけ。集約の振る舞いとユースケースは、ペイロードの大きさを確かめない。上限を超えるペイロードのドラフトを渡した `collectEvents` は `SystemError`（理由 `EVENT_PAYLOAD_TOO_LARGE`）を投げる。`fn` がこれを捕まえて正常に終わっても、`run` は同じ `SystemError` を投げ、何も反映しない
- イベントは、対象の件数に比例する内容をペイロードに持たない。その内容は、同じ UnitOfWork が区画の記録に保存し、ペイロードは記録を引く ID を持つ。各イベントのペイロードは、発生元のドメインの「ドメインイベント」が定める

```ts
const EventPayloadLimit: { readonly maxBytes: 65536 };
```

## QuarantinedEventQuery

配送を繰り返しても消費者が処理を終えず、配送の対象から外して隔離したイベントの件数を返す。イベントは記録した区画に属するので、ワークスペースの区画に結び付く。`PartitionReader.workspace(workspaceId)` が `quarantinedEventQuery` として公開し、対象の区画は `workspaceId` が決める。ユースケースが使うポートで、`packages/core/src/application/` に置く。Installation の状態確認が使う。

```ts
interface QuarantinedEventQuery {
  count(): Promise<number>;
}
```

- `count` は、結び付いた区画が記録したイベントのうち、隔離されたままのものの件数を返す。イベントの種類で分けない。ほかのワークスペースの区画のイベントを数えない
- 区画が無いワークスペースには 0 を返す
- 設置の全体の件数の求め方は、[index.md](index.md) の「区画をまたぐ手順」の状態確認が定める
- 配送の対象に残っているイベント（未配送、再配送を待つもの）と、消費者が処理を終えたイベントを数えない
- 配送し直して消費者が処理を終えたイベントは、数えなくなる。`WorkspaceContext.deleteEvents` が削除したイベントの扱いは、「UnitOfWorkProvider」が定める
- 隔離までの配送の回数は、実行基盤の設定が定める。ポートは回数を約束しない。ワーカーの入口が設置の状態で処理を始めなかった配送（[index.md](index.md) の「設置の状態」）も、回数に数える
- 読み取りだけを行う。確定済みの隔離を反映する
- ポート適合テストは、イベントごとの処理の成否を指定できる試験用の消費者に対して実行する。処理を失敗させ続けたイベントの隔離の成立を待って、件数を確かめる。隔離の前の件数と、配送し直した後の件数は、アダプターごとの検証の対象にする

## 期限つきの一度きりの値

推測できない値を相手に渡し、値そのものを保存せずに照合値（`SecretTokenIssuer` が計算する）で引く集約に共通の規則。`Session`、`RecoveryRequest`、`Invitation`、`Token`、`PreviewExchange`、`DeviceAuthorization` が従う。各集約は、この規則を書き直さず、表の「集約が定めるもの」だけを自分のファイルで定める。

| 規則 | 内容 | 集約が定めるもの |
| --- | --- | --- |
| 照合値の一意性 | 照合値は、同じ区画の、同じ種類の集約の、同じ種類の照合値の中で一意。違う種類の照合値どうしは重なってよい。ポートが担保する。同じ照合値の集約がある `insert` は `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）。照合値は、値の全体から求める | 照合値を二つ以上持つ集約では、照合値の種類 |
| 値と区画 | ワークスペースの区画の集約（`Invitation`、`Token`、`PreviewExchange`）の値は `SecretTokenIssuer.issueForWorkspace` が、設置の区画の集約（`Session`、`RecoveryRequest`、`DeviceAuthorization`）の値は `SecretTokenIssuer.issue` が発行する。初期構成で宣言したトークンの値は、ホスティング運営者が決めた値で、ワークスペースの ID を含まない。値から区画を求める手順と、該当する集約が無い値の扱いは、[index.md](index.md) の「ワークスペースを知らない要求」が定める | なし |
| URL の組み立て | 値を含む CMS の画面の URL は、設置の公開の URL の末尾の `/` を除き、経路、`/`、値を順につないだ文字列。値は英数字と `-` `_` だけからなるので、符号化しない。組み立ての分担は [index.md](index.md) の「URL の組み立て」が定める | 値を含む CMS の画面の URL を持つ集約では、その経路と、URL を作る関数 |
| 期限の境界 | `now` が `expiresAt` 以降（同じ日時を含む）なら期限切れ。`now` が `expiresAt` より前なら期限内 | 期限を持たない場合の扱い |
| 保持期間の起点 | 各集約は、下の「保持期間」を数え始める時刻を返す振る舞い `lapsedAt(entity): Date \| null` を公開する。`null` は、起点が決まっていないこと。`lapsedAt` が `now` 以前の集約は、使えない（期限切れ、使用済み、取り消し済みのいずれか）。`null` と、`now` より後の `lapsedAt` は、使えるかどうかを決めない | 既定（下の箇条）と違う `lapsedAt` の値 |
| 削除 | 各ポートは `deleteLapsed(before: Date, limit: number): Promise<number>` を持つ。対象は、`lapsedAt` が `null` でなく、`before` 以前（同じ日時を含む）の集約。状態によらない。`lapsedAt` が `null` の集約と、`before` より後の集約を消さない。`lapsedAt` の昇順、同じ時刻は ID の昇順に削除する。楽観ロックを使わない | なし |

- 既定の `lapsedAt` は `expiresAt` で、使用済み・取り消し済みになった時刻を使わない。既定と違う値を返す集約は、自分のファイルで値を定める
- `before` は、呼び出し側のユースケースが `OneTimeValueRetention.cutoff(now)` で求めて渡す。ポートは、保持期間を足し引きしない。求め方と呼び方は、下の「掃除」が定める
- 保持期間: 使えない集約は、`lapsedAt` の `OneTimeValueRetention.ms` 後まで、示された値に、期限切れ・使用済み・取り消し済みの理由を返す。保持期間を過ぎた集約（`OneTimeValueRetention.isForgotten(lapsedAt, now)` が真）は、掃除が削除したかどうかによらず、該当する集約が無い値と同じに扱う。値を受けて集約を引くユースケースは、引いた集約をこの述語に通し、真なら、集約が無い場合の結果を返す。結果は、掃除の起動の時期に依らない。この規則は、この節だけが定める
- 保存するのは照合値だけ。値そのものを保存する集約は、`SecretCipher` で保護した形で保存し、その集約のファイルが理由を定める
- 保持期間は固定の値。定数 `OneTimeValueRetention` が持つ。ほかの文書は、この名前で参照する

```ts
const OneTimeValueRetention: {
  readonly ms: 2_592_000_000;
  cutoff(now: Date): Date;
  isForgotten(lapsedAt: Date | null, now: Date): boolean;
};
```

- `ms` は、`spec/scenario/app-integration.md` の APP-12 の「理由を示す期間」の値。理由を示す期間は `lapsedAt` から数える。既定の `lapsedAt` の集約では、有効期限（`expiresAt`）から数える。`cutoff` は、`now` の `ms` 前の日時を返す
- `isForgotten` は、`lapsedAt` が `null` でなく、`cutoff(now)` 以前（同じ日時を含む）なら真を返す。`deleteLapsed(cutoff(now), limit)` の対象の条件と同じ

### 掃除

定期ジョブ「期限切れの記録の掃除」のユースケース（[index.md](index.md) の「定期ジョブ」）に共通の進め方は、ここだけが定める。各ユースケースは、対象のポートを書き、進め方を書き直さない。

- `before` は、ユースケースが、初めに一度読んだ `Clock.now` を `OneTimeValueRetention.cutoff` に渡して求める。一回の起動の中で変えない
- `deleteLapsed(before, limit)` は、対象のポートごとに、「TransactionalRepository」の「件数を区切る削除」の進め方で呼ぶ
- `WriteScope` は、設置の区画の集約では `installation`、ワークスペースの区画の集約では `workspace` の `none`。集約の状態と、ワークスペースの状態によらず削除する
- 途中で失敗または中断した場合、確定済みの削除は残り、残りは次の起動が削除する
- 出力は、削除した件数

## SecretTokenIssuer

目的: 一度きりの値や推測できない値を生成し、保存する照合値を求める。ユースケースが使うポートで、置き場所は `packages/core/src/application/ports/secretTokenIssuer.ts`。使う側のドメインは、照合値を自分のブランド型に包んで持つ。

```ts
type IssuedSecret = Readonly<{ value: string; digest: string }>;

interface SecretTokenIssuer {
  issue(): Promise<IssuedSecret>;
  issueForWorkspace(workspaceId: WorkspaceId): Promise<IssuedSecret>;
  workspaceOf(value: string): WorkspaceId | null;
  digest(value: string): Promise<string>;
}
```

| メソッド | 契約 |
| --- | --- |
| `issue` | 256 ビット以上の予測できない値を返す。`value` は英数字と `-` `_` だけからなる 43 文字以上の文字列。呼び出しごとに違う `value` を返す。`digest` は `digest(value)` と同じ値。ワークスペースの ID を含まない |
| `issueForWorkspace` | `issue` と同じ強さの予測できない部分に、`workspaceId` を含めた値を返す。`value` は英数字と `-` `_` だけからなる文字列。同じ `workspaceId` でも、呼び出しごとに違う `value` を返す。`digest` は `digest(value)` と同じ値。`workspaceOf(value)` は、渡した `workspaceId` を返す |
| `workspaceOf` | `issueForWorkspace` が返した値から、含まれるワークスペースの ID を取り出す。その形でない値（`issue` が返した値、空の文字列、任意の文字列）には `null` を返す。例外を投げない。値が実在の集約を指すことと、ワークスペースが存在することを確かめない。状態を読まない |
| `digest` | 同じ値から常に同じ照合値を、違う値から違う照合値を返す。照合値は空でない。照合値から値を求められない。空の入力と任意の長さの入力にも照合値を返す |

- ワークスペースの ID は秘密でない。値の予測できない部分が、値を知らない人による推測を防ぐ。ワークスペースの ID を含む値は、ワークスペースが違えば違う値になるので、別の区画の値と一致しない
- 状態を持たない。UnitOfWork との関係は「UnitOfWorkProvider」に従う
- エラーケース: 生成の失敗は `SystemError`
- ポート適合テストの対象は、`value` の形と長さ、呼び出しごとに違うこと、`issueForWorkspace` と `workspaceOf` の対応、`workspaceOf` が `null` を返す入力、`digest` の決定性と区別。値が予測できないことと、照合値から値を求められないことは、アダプターごとの検証の対象にする

## UsageLimit と UsageStanding

利用量の制限の上限と、キーの制限の状態。上限を定め、制限の状態から結果を決めるドメインの振る舞いが使うドメインの型。

```ts
type UsageLimit = Readonly<{ max: number; windowMs: number }>;

type UsageStanding =
  | Readonly<{ status: "available" }>
  | Readonly<{ status: "limited"; retryAt: Date }>;
```

- `UsageLimit` は、`max` が 1 以上の整数、`windowMs` が 1 以上の整数（ミリ秒）
- `UsageStanding` の値と `retryAt` の意味は、「UsageLimiter」の `check` が定める
- 型は `packages/core/src/domain/common/usageLimit.ts` に置く

## UsageLimiter

目的: キーごとの利用を数え、期間の中の利用が上限に達したキーを制限する。ユースケースが使うポートで、置き場所は `packages/core/src/application/ports/usageLimiter.ts`。何を一つの利用として数えるかと、キーと上限は、使う側のドメインが定める。`UsageLimit` と `UsageStanding` は、「UsageLimit と UsageStanding」の型を使う。

```ts
interface UsageLimiter {
  check(key: string, limit: UsageLimit, now: Date): Promise<UsageStanding>;
  record(key: string, limit: UsageLimit, now: Date): Promise<void>;
}
```

- キーは、制限の単位を指す空でない文字列。用途ごとに、使う側が重ならない接頭辞を付ける。キーが違えば、利用を別々に数える
- 呼び出し側の条件: 同じキーには、`check` と `record` で常に同じ `UsageLimit` を渡す。条件を外れた呼び出しの結果は約束しない
- 有効な期間は、キーごとに高々一つ。期間は、そのキーに有効な期間が無いときの `record` の `now` から始まり、`windowMs` の後に終わる。期間は延びない
- 呼び出しの `now` が、有効な期間の終わりの時刻より前なら、その期間は有効。終わりの時刻以降（同じ時刻を含む）なら、その期間は終わっていて、有効な期間は無い。終わった期間の利用は数えない
- 同じキーへの `now` は、前の呼び出しより戻ってよい。戻った `now`（有効な期間の始まりより前の時刻を含む）は、有効な期間の中の利用として数える。並行した要求が、それぞれの初めに求めた時刻を渡す場合が当たる

| メソッド | 契約 |
| --- | --- |
| `check` | `now` に有効な期間があり、その期間に記録した利用が `max` 以上なら `limited` を返し、`retryAt` は期間の終わりの時刻。それ以外と、記録の無いキーは `available`。`now` が期間の終わりの時刻ちょうどなら `available`。状態を変えない |
| `record` | そのキーの利用を一つ記録する。`now` に有効な期間があれば、その期間の利用として数える。無ければ、`now` から新しい期間を始める。制限中のキーにも記録でき、`retryAt` は変わらない |

- 一意性・整合性: 利用の数はポートが保つ。並行した `record` は、どれも失わずに数える。呼び出し側は事前の確認を必要としない
- 並行性: 楽観ロックを使わない。`check` と `record` の組は原子的でない。並行した要求は、`check` が `available` を返した後に、上限を超えて記録され得る。超えた分も同じ期間に数える
- 可視性: `record` の完了後の `check` は、その記録を反映する。UnitOfWork との関係は「UnitOfWorkProvider」に従う
- 記録は期間の終わりまで保てば足りる。終わった期間の記録は数えないので、アダプターが捨てる。捨てる時期は、アダプターが決める。記録は、バックアップと復旧の対象の外にある
- エラーケース: 範囲を外れた `UsageLimit` と空のキーは `SystemError`（理由 `INVALID_QUERY`）。保存先の失敗は `SystemError`
- 記録を保つ期間と、バックアップの対象にしないことは、アダプターごとの検証の対象で、ポート適合テストの対象にしない

## SystemMailer

目的: 設置全体のメール送信で、一通のメールを送る。ワークスペースの送信接続設定（Form の `MailSender`）と別。ユースケースが使うポートで、置き場所は `packages/core/src/application/ports/systemMailer.ts`。宛先、件名、本文は、使う側のユースケースが決める。

```ts
type SystemMail = Readonly<{ to: EmailAddress; subject: string; body: string }>;

type SystemMailResult =
  | Readonly<{ status: "accepted" }>
  | Readonly<{ status: "not_configured" }>
  | Readonly<{ status: "failed"; reason: string }>;

interface SystemMailer {
  isConfigured(): Promise<boolean>;
  send(mail: SystemMail): Promise<SystemMailResult>;
}
```

| メソッド | 契約 |
| --- | --- |
| `isConfigured` | 設置全体のメール送信を宣言している設置で真、宣言していない設置（外部メールを使えない設置）で偽を返す。何も送らず、メール送信基盤へ接続しない。設置の起動から変わらない。`send` が `not_configured` を返す条件と一致する |
| `send` | 宛先一つへ、件名と本文（プレーンテキスト）のメールを一通送る。送信者は設置の設定から決まる。結果は下の表のいずれかで返し、送信の失敗で例外を投げない |

| 結果 | 意味 |
| --- | --- |
| `accepted` | メール送信基盤が受理した。受信箱への到着は示さない |
| `not_configured` | 設置全体のメール送信を宣言していない設置（外部メールを使えない設置）。入力によらずこの結果を返し、空の件名と空の本文でも `SystemError` にしない。何も送らない。この結果を受けたときの出力は、呼び出し側のユースケースが定める（[index.md](index.md) の「URL の組み立て」） |
| `failed` | メール送信基盤が受理しなかった、または受理を確認できなかった。`reason` は利用者に示せる理由で、認証情報と本文を含まない |

- 一意性・整合性: 同じメールの重複を抑えない。呼び出すたびに一通を送る。再送は呼び出し側が利用者の操作で行い、ポートは自動で再試行しない
- 並行性: 状態を持たない。並行した `send` は互いに影響しない
- 可視性: UnitOfWork との関係は「UnitOfWorkProvider」に従う。呼び出し側は、原因の書き込みが確定した後に呼ぶ
- 本文と件名を、通常のログとエラー情報に出さない。本文は一度きりの値を含む
- 送る前に経路を決める呼び出し側は `isConfigured` を使う
- エラーケース: 設定している設置で、件名または本文が空のメールは `SystemError`（理由 `INVALID_QUERY`）。片方だけが空の場合も当たる
- ポート適合テストは、応答を指定できる試験用の相手に対して実行する（「TransactionalRepository」）。メール送信基盤へ接続しないこと（`isConfigured`）と、ログへ出さないことは、アダプターごとの検証の対象にする

## DeclarationIssue

初期構成の宣言の問題の一件。各ドメインのファイルの「初期構成の宣言」が公開する検証が、問題の列（`readonly DeclarationIssue[]`）を返す。Installation のユースケースが、各ドメインの検証を順に呼び、列をつなぐ。

```ts
type DeclarationIssue = Readonly<{
  path: string;
  code: string;
}>;
```

- `path` は、宣言の中の問題の位置。プロパティの名前と 0 始まりの位置を `.` でつないだ文字列
- `code` は、その位置の値に、ドメインの生成規則が投げるエラー（`ValidationError` / `BusinessRuleError`）の理由のコード。問題の列を返す関数で検証する値では、その列の理由のコード
- 問題が無い宣言の検証は、空の列を返す
- 検証は、保存を伴わない純粋な関数で、秘密情報の保護の前に呼べる。一つの宣言の問題を、最初の一件で打ち切らずにすべて返す

## 各ドメインが公開する項目

各ドメインが、自分のファイルの「状態確認の項目」「利用終了の消去」「利用再開前の処理」に公開する項目の一件の型。各ドメインは項目をこの型で公開し、Installation のユースケースが受けて束ねる（[index.md](index.md) の「依存方向」）。

```ts
type DomainKind = Readonly<{ domain: string; kind: string }>;
type DomainCount = DomainKind & Readonly<{ count: number }>;
type StatusItem = DomainCount & Readonly<{ actionRequired: boolean }>;

type ResumeItemDefinition = DomainKind &
  Readonly<{ unit: "installation" | "workspace" }> &
  (
    | Readonly<{ countsAsRemaining: true; shownInReview: boolean }>
    | Readonly<{ countsAsRemaining: false; shownInReview: true }>
  );
type ResumeItem = ResumeItemDefinition & Readonly<{ count: number }>;
```

- `domain` は公開したドメインの名前、`kind` はそのドメインが公開した識別子、`count` は件数（0 以上の整数）。`kind` の表記は [index.md](index.md) の「手順を呼ぶ順」に従う
- `DomainKind` は、種類の名前だけの項目（利用終了で失われるデータの種類、CMS の通常データの外に残る情報、消去する資格情報）を受ける。件数を示す項目は `DomainCount` が受ける。一つのドメインが、同じ `kind` を、種類の名前と件数の項目の両方に公開してよい。その `kind` は、`DomainKind` の列と `DomainCount` の列の両方に、それぞれの形で入る
- `StatusItem` は、状態確認の項目の一件。`actionRequired` は、そのドメインが公開した判定の値。対処の内容と、対処する人は、公開したドメインの「状態確認の項目」が定める
- `ResumeItemDefinition` は、利用再開前の処理の項目の定義。`unit` は件数の単位（`installation` は設置の全体で一つの件数、`workspace` はワークスペースごとの件数）、`countsAsRemaining` は残件数に数えるかどうか、`shownInReview` は利用再開前の確認（[installation.md](installation.md) の「ResumeReview」）の列に示すかどうか。項目は、残件数と確認の少なくとも一方に入る。各ドメインは、項目の定義の列（定数）と、件数を求める関数を公開する。`ResumeItem` は、定義に件数を添えた一件
- 受ける側は、`domain` と `kind` の値を列挙せず、件数と判定を求めない。ユースケースが、各ドメインの節が公開する定数と関数を呼んで作り、渡す
- 型は `packages/core/src/domain/common/domainItem.ts` に置く

## キーで宣言した設定の紐付け

初期構成でモデル・フォームのキーを指定して宣言した設定（Form のフォームの運用設定、Token のトークンの範囲の要素）を、スキーマの反映の確定が対象の ID に紐付け、対象の削除で外す規則。この節は、キーと ID の対応の規則だけを定める。紐付け待ちをどう持つか（集約の中の状態として `KeyBinding` を持つか、別の集約として持つか）と、紐付けと外れたときに行うことは、使う側のドメインが定める（Form は [form.md](form.md) の「スキーマの反映に伴う変更」、Token は [token.md](token.md) の「範囲の更新」）。別の集約として持つ使う側は、紐付け待ちのキーから `pending` の `KeyBinding` を作って `KeyBinding.apply` に渡す。

```ts
type KeyBinding<TId> =
  | Readonly<{ status: "pending"; key: string }>
  | Readonly<{ status: "bound"; id: TId }>;

type BindingTargetChange<TId> = Readonly<{
  created: readonly Readonly<{ key: string; id: TId }>[];
  removed: readonly Readonly<{ key: string; id: TId }>[];
}>;

type BindingOutcome<TId> =
  | Readonly<{ kind: "unchanged" }>
  | Readonly<{ kind: "bound"; id: TId }>
  | Readonly<{ kind: "detached" }>;

const KeyBinding: {
  apply<TId>(binding: KeyBinding<TId>, change: BindingTargetChange<TId>): BindingOutcome<TId>;
};
```

- `created` と `removed` は、一回の反映の確定が作成・削除したモデル（または、フォーム）のキーと ID。作る主体と、Schema のどの値から作るかは、使う側（Form は [form.md](form.md) の「スキーマの反映に伴う変更」、Token は [token.md](token.md) の「範囲の更新」）と [../usecases/schema.md](../usecases/schema.md) の「ほかのドメインとの対応」が定める
- `pending` で、`key` が `created` のキーと一致する紐付けは `bound`（その ID）。`bound` で、`id` が `removed` の ID と一致する紐付けは `detached`。それ以外は `unchanged`
- 紐付けは、キーの一致だけで決める。`bound` の紐付けは ID で対象を指すので、キーの変更で変わらない（[index.md](index.md) の「モデルとフォームの ID」）
- `detached` になった設定は、`pending` に戻らない。削除したキーと同じキーの対象を後から追加しても、外れた設定を引き継がない
- `KeyBinding.apply` は、保存を伴わない純粋な関数。型は `packages/core/src/domain/common/keyBinding.ts` に置く

## エラー

種類の一覧と意味は [index.md](index.md) の「共通の型とポート」にある。理由のコードは、各ドメインの `ErrorCode` と、入口だけが返す理由コードの表（[../usecases/app-api.md](../usecases/app-api.md) の「入口の理由コード」）がすべて列挙する。

- `NotFoundError` が持つ対象の種類は `NotFoundTarget`。値は、存在しない対象の名前を、英小文字の語を `_` でつないだ単数形で書く。対象の種類を足すドメインは、この型に値を足す。型は `packages/core/src/domain/common/notFoundTarget.ts` に置く

```ts
type NotFoundTarget =
  | "workspace" | "account" | "member" | "invitation" | "recovery_request"
  | "token" | "preview_exchange" | "device_authorization"
  | "model" | "form" | "field" | "schema_apply_job"
  | "content" | "revision" | "media"
  | "form_settings" | "mail_connection" | "submission" | "recipient"
  | "webhook_endpoint" | "change_notification"
  | "audit_log_entry" | "backup";
```

共通のコードは次のとおり。

| 種類 | 理由 | 意味 |
| --- | --- | --- |
| `ForbiddenError` | `PERMISSION_MISSING` | メンバーまたはトークンが保持する権限が足りない |
| `ForbiddenError` | `OWNER_ONLY` | 所有者だけが行える操作 |
| `ForbiddenError` | `ACTOR_NOT_ALLOWED` | ユースケースが受け付けない種類の実行者 |
| `ConflictError` | `STALE_VERSION` | 対象の集約が、読んだ後に変わった、または存在しない |
| `ConflictError` | `ALREADY_EXISTS` | 同じ区画に、同じ ID の集約が既にある |
| `ConflictError` | `SCHEMA_CHANGED` | 確定の時点で、スキーマの版が読んだ版と違う。条件が `stableSchema` の UnitOfWork では、適用中の場合も含む |
| `ConflictError` | `SECRET_DIGEST_TAKEN` | 同じ照合値の、同じ種類の一度きりの値が既にある |
| `ValidationError` | `InvalidEmailAddress` | メールアドレスの形に合わない |
| `ValidationError` | `InvalidPagination` | ページまたは件数の指定が範囲外 |
| `ValidationError` | `InvalidTargetCount` | 一度の操作で扱う対象が 0 件、または上限を超える |
| `ValidationError` | `InvalidId` | 呼び出し側が指定する ID が、`IdGenerator` の形式に合わない。利用者が要求に書いた既存の対象の ID が、空の文字列 |
| `ConflictError` | `WORKSPACE_NOT_ACTIVE` | 確定の時点で、区画の `Workspace` が `active` でない、または存在しない。ユースケースが返す結果は [index.md](index.md) の「競合の扱い」 |
| `ConflictError` | `RESTORE_IN_PROGRESS` | 復旧の置き換えと重なった呼び出しと UnitOfWork の拒否。当たる呼び出しと UnitOfWork は [installation.md](installation.md) の「取得・置き換え・更新の排他」が定める |
| `ForbiddenError` | `MFA_SETUP_REQUIRED` | 多要素認証を必須にしたワークスペースで、多要素認証を設定していない |
| `UnavailableError` | `SCHEMA_APPLYING` | スキーマの反映の適用中 |
| `UnavailableError` | `INSTALLATION_NOT_OPERATING` | 設置が利用者の要求（`request`）を受け付けない。[installation.md](installation.md) の `InstallationState.blockingIssues` が問題を返すか、`InstallationState.admit` が受け付けない |
| `SystemError` | `INVALID_QUERY` | ポートの契約が呼び出し側に課す引数の条件を外れた呼び出し |
| `SystemError` | `NESTED_UNIT_OF_WORK` | UnitOfWork のネスト |
| `SystemError` | `EVENT_PAYLOAD_TOO_LARGE` | ドメインイベントのペイロードが大きさの上限を超える |
| `SystemError` | `SECRET_UNAVAILABLE` | 保護した秘密情報を元に戻せない。保護に使う鍵を使えない |

公開内容の整合性の違反の理由と、担保するポートは、[index.md](index.md) の「公開内容の整合性と通し番号」にある。
