# Audit

行った操作の履歴を保つ。境界と依存方向は [index.md](index.md) が定める。Audit は他のドメインに依存しない。ほかのドメインも Audit に依存しない。操作履歴は、ワークスペースの区画にある（[index.md](index.md) の「区画」）。

Audit はドメインイベントを発行しない。

## ユビキタス言語

| 英語名 | 日本語名 | 定義 |
| --- | --- | --- |
| Audit log | 操作履歴 | ワークスペースごとの、設定とスキーマを変えた操作の記録。コンテンツの版の履歴と別 |
| Audit log entry | 操作履歴の一件 | 一つの操作の記録。日時、実行者、操作の種類、対象、変更の内容を持つ |
| Operation | 操作 | 操作履歴に残す操作の種類と、その対象の組 |
| Actor | 実行者 | 操作を行った主体。メンバー、アプリに紐付くトークンとその発行者、メンバーに紐付くトークンとそのメンバー、ホスティング運営者 |
| Target | 対象 | 操作が変えたもの。記録の時点の名称を持つ |
| Change | 変更の内容 | 操作が変えた項目と、変更前後の値 |
| Secret change | 秘密情報の変更 | 秘密情報を持つ項目を変えた事実。値を持たない |

## エンティティ

### AuditLogEntry

フィールド。

```ts
type AuditLogEntry = Readonly<{
  id: AuditLogEntryId;
  occurredAt: Date;
  actor: AuditActor;
  operation: AuditOperation;
}>;
```

| フィールド | 制約 |
| --- | --- |
| `id` | 必須 |
| `occurredAt` | 必須。操作の時刻。操作のユースケースが `Clock.now` で初めに求めた現在時刻で、`record` に `now` として渡す。UnitOfWork が確定した時刻ではない |
| `actor` | 必須 |
| `operation` | 必須。操作の種類、対象、変更の内容（`changes`。持つ種類だけ） |

状態の判別子と `version` を持たない。ワークスペースの ID のフィールドを持たない。操作を行ったワークスペースは、記録がある区画が決める。

振る舞い。

```ts
const AuditLogEntry: {
  record(
    params: {
      id: string;
      actor: AuditActor;
      operation: AuditOperation;
    },
    now: Date,
  ): AuditLogEntry;
  matches(entry: AuditLogEntry, filter: AuditLogFilter): boolean;
};
```

| 振る舞い | 処理 |
| --- | --- |
| `record` | `AuditLogEntryId` を生成し、`occurredAt` を `now` にした記録を返す。`changes` の項目のうち、`ValueChange` の変更前後の値が等しい項目を除く |
| `matches` | 記録が絞り込みの条件をすべて満たせば真。条件の意味は `AuditLogFilter` が定める |

不変条件。

- 記録は追記だけを行う。追記した記録は変更も、個別の削除もしない
- 記録は、値を所有するドメインが秘密とする値を含まない
- 記録は、フォーム回答の入力内容を含まない。`AuditOperation` は回答を対象にする種類を持たない
- 記録は、記録の時点の実行者と対象の名称を持つ。実行者や対象が後で無くなっても、記録は残る

ライフサイクル。

- 生成: 操作履歴に残す操作を確定するとき（「トランザクション境界」）
- 削除: ワークスペースの利用終了で、そのワークスペースの記録をすべて削除する（「利用終了の消去」）

## 値オブジェクト

利用者の入力から作る値オブジェクト（`AuditLogFilter`）の違反は、`ValidationError`（項目ごとの理由）になる。利用者の入力に由来しない値（`AuditLabel`）の違反は `BusinessRuleError` になる。集約の ID、`Actor`、`TokenDesignation`、`NotFoundTarget`、`Pagination`、`PaginationResult` は [common.md](common.md) の定義を使う。

| 名前 | 型 | バリデーションルール | 等価性 |
| --- | --- | --- | --- |
| `AuditActor` | 直和型 | 下の定義 | — |
| `AuditOperation` | 直和型 | 下の定義。操作の種類ごとに、取れる対象の種類と、変更の内容の項目が決まる | — |
| `AuditOperationType` | 文字列リテラルの直和型 | `AuditOperation["type"]` | 値の一致 |
| `AuditTarget` | 直和型 | 下の定義 | — |
| `AuditLabel` | `string`（ブランド） | 前後の空白を除いて 1 文字以上。違反は `InvalidAuditLabel` | 値の一致 |
| `ValueChange<T>` / `SecretChange` / `AuditChange` | 型 | 下の定義 | — |
| `AuditLogFilter` | オブジェクト | 下の定義 | — |

### AuditActor

```ts
type AuditActor =
  | Readonly<{ kind: "member"; memberId: MemberId; name: AuditLabel }>
  | Readonly<{
      kind: "appToken";
      tokenId: TokenId;
      token: Extract<TokenDesignation, { binding: "app" }>;
      issuerMemberId: MemberId;
      issuerName: AuditLabel;
    }>
  | Readonly<{
      kind: "memberToken";
      tokenId: TokenId;
      token: Extract<TokenDesignation, { binding: "member" }>;
      memberId: MemberId;
      memberName: AuditLabel;
    }>
  | Readonly<{ kind: "operator" }>;

type AuditActorSource =
  | Readonly<{ actor: Extract<Actor, { kind: "member" }>; name: string }>
  | Readonly<{
      actor: Extract<Actor, { kind: "appToken" }>;
      token: Extract<TokenDesignation, { binding: "app" }>;
      issuerName: string;
    }>
  | Readonly<{
      actor: Extract<Actor, { kind: "memberToken" }>;
      token: Extract<TokenDesignation, { binding: "member" }>;
      memberName: string;
    }>
  | Readonly<{ actor: Extract<Actor, { kind: "operator" }> }>;

const AuditActor: {
  from(source: AuditActorSource): AuditActor;
};
```

- 共通の `Actor` のうち、操作履歴に残す操作を行う種類（`member`、`appToken`、`memberToken`、`operator`）に、記録の時点の名称と、トークンを指す値を加えた値。`system` を持たない。定期ジョブとイベントの消費者が確定する操作（スキーマの反映の確定）の記録の実行者は、その操作を要求した実行者
- `AuditActorSource` は、`Actor` の種類ごとに、その種類が必要とする値だけを持つ。値の欠けた入力は型が受け付けない
- `from` は、アカウントの名前から `AuditLabel` を生成して `AuditActor` を返す
- `member` の `name` はメンバーのアカウントの名前
- `appToken` と `memberToken` の `token` は、[common.md](common.md) の「TokenDesignation」の値
- `appToken` の `issuerName` は、`Actor` の `issuerMemberId` が指す発行者のアカウントの名前。`memberToken` の `memberName` は、`Actor` の `memberId` が指すメンバーのアカウントの名前

### AuditOperation と AuditTarget

```ts
type WorkspaceTarget = Readonly<{ kind: "workspace" }>;
type MemberTarget = Readonly<{ kind: "member"; memberId: MemberId; label: AuditLabel }>;
type InvitationTarget = Readonly<{
  kind: "invitation";
  invitationId: InvitationId;
  label: AuditLabel;
}>;
type TokenTarget = Readonly<{ kind: "token"; tokenId: TokenId; token: TokenDesignation }>;
type AppTokenTarget = TokenTarget & Readonly<{ token: Extract<TokenDesignation, { binding: "app" }> }>;
type SchemaTarget = Readonly<{ kind: "schema"; label: AuditLabel }>;
type FormTarget = Readonly<{ kind: "form"; formId: FormId; label: AuditLabel }>;
type WebhookEndpointTarget = Readonly<{
  kind: "webhook_endpoint";
  endpointId: WebhookEndpointId;
  label: AuditLabel;
}>;

type AuditTarget =
  | WorkspaceTarget
  | MemberTarget
  | InvitationTarget
  | TokenTarget
  | SchemaTarget
  | FormTarget
  | WebhookEndpointTarget;

type ValueChange<T> = Readonly<{ before: T; after: T }>;
type SecretChange = Readonly<{ changed: true }>;
type AuditValue = string | number | null | readonly string[];
type AuditChange = Readonly<{ field: string; change: ValueChange<AuditValue> | SecretChange }>;

type AuditOperation =
  | Readonly<{ type: "invitation.created"; target: InvitationTarget }>
  | Readonly<{ type: "member.invited"; target: InvitationTarget }>
  | Readonly<{ type: "member.joined"; target: MemberTarget }>
  | Readonly<{ type: "member.invitation_revoked"; target: InvitationTarget }>
  | Readonly<{ type: "member.permissions_changed"; target: MemberTarget; changes: readonly AuditChange[] }>
  | Readonly<{ type: "member.removed"; target: MemberTarget }>
  | Readonly<{ type: "workspace.owner_changed"; target: MemberTarget; changes: readonly AuditChange[] }>
  | Readonly<{ type: "token.issued"; target: TokenTarget }>
  | Readonly<{ type: "token.revoked"; target: TokenTarget }>
  | Readonly<{ type: "token.restrictions_changed"; target: AppTokenTarget; changes: readonly AuditChange[] }>
  | Readonly<{ type: "schema.reflected"; target: SchemaTarget }>
  | Readonly<{ type: "schema.migrations_applied"; target: SchemaTarget }>
  | Readonly<{ type: "schema.pushed"; target: SchemaTarget }>
  | Readonly<{ type: "schema.migrations_marked_applied"; target: SchemaTarget }>
  | Readonly<{ type: "form.notification_definition_changed"; target: FormTarget; changes: readonly AuditChange[] }>
  | Readonly<{ type: "form.recipients_changed"; target: FormTarget; changes: readonly AuditChange[] }>
  | Readonly<{ type: "form.retention_policy_changed"; target: FormTarget; changes: readonly AuditChange[] }>
  | Readonly<{ type: "mail_connection.changed"; target: WorkspaceTarget; changes: readonly AuditChange[] }>
  | Readonly<{ type: "webhook.settings_changed"; target: WebhookEndpointTarget; changes: readonly AuditChange[] }>
  | Readonly<{ type: "preview.url_changed"; target: WorkspaceTarget; changes: readonly AuditChange[] }>
  | Readonly<{ type: "content.retention_policy_changed"; target: WorkspaceTarget; changes: readonly AuditChange[] }>
  | Readonly<{ type: "bootstrap.loaded"; target: WorkspaceTarget }>
  | Readonly<{ type: "bootstrap.setting_bound"; target: FormTarget | AppTokenTarget }>;
```

操作の種類。

| `type` | 残す操作 | 対象 |
| --- | --- | --- |
| `invitation.created` | 初期所有者の招待の発行と発行し直し | 招待。名称は宛先のメールアドレス |
| `member.invited` | メンバーの招待の作成。同じ宛先の前の招待を置き換えた招待も、この一件だけを記録する | 招待。名称は宛先のメールアドレス |
| `member.joined` | メンバーを作った招待の受諾 | 参加したメンバー。名称はアカウントの名前 |
| `member.invitation_revoked` | 招待の取り消し | 招待。名称は宛先のメールアドレス |
| `member.permissions_changed` | メンバーの権限の変更 | メンバー。名称はアカウントの名前 |
| `member.removed` | メンバーの利用の解除 | メンバー。名称はアカウントの名前 |
| `workspace.owner_changed` | 所有者の変更。初期所有者の招待の受諾による所有者の指定を含む | 新しい所有者のメンバー。名称はアカウントの名前 |
| `token.issued` | トークンの発行。発行の経路によらない | トークン |
| `token.revoked` | トークンの失効。発行の経路によらない | トークン |
| `token.restrictions_changed` | ブラウザ用トークンの利用元の限定と利用量の制限の変更 | トークン |
| `schema.reflected` | スキーマの版の追記（下の箇条） | スキーマ。名称は反映後の版 |
| `schema.migrations_applied` | スキーマの版の追記（下の箇条） | スキーマ。名称は反映後の版 |
| `schema.pushed` | スキーマの版の追記（下の箇条） | スキーマ。名称は反映後の版 |
| `schema.migrations_marked_applied` | スキーマの版の追記（下の箇条） | スキーマ。名称は記録後の版 |
| `form.notification_definition_changed` | 通知定義の変更 | フォーム。名称はフォームのキー |
| `form.recipients_changed` | 通知先の変更（追加、メールアドレスの修正、有効・無効の切り替え、削除） | フォーム。名称はフォームのキー |
| `form.retention_policy_changed` | 回答の保持方針の変更 | フォーム。名称はフォームのキー |
| `mail_connection.changed` | 送信接続設定の変更（登録と更新） | ワークスペース |
| `webhook.settings_changed` | 表示アプリへの通知設定の変更（宛先の登録・変更・削除、秘密情報の置き換え） | 通知の宛先。名称は宛先の URL |
| `preview.url_changed` | プレビュー先の表示アプリの URL の変更（設定と取り除き） | ワークスペース |
| `content.retention_policy_changed` | コンテンツの保持方針の変更 | ワークスペース |
| `bootstrap.loaded` | 初期構成の読み込み | ワークスペース |
| `bootstrap.setting_bound` | 初期構成の運用設定の紐付け | 紐付けたフォーム（名称はフォームのキー）、または範囲を紐付けたトークン |

- `changes` は、操作が変えた項目の列。`field` は項目の名前で、一つの記録の中で重ならない。項目の名前と意味、値を `AuditValue` で表す形は、その操作を行うユースケースが定める
- `ValueChange` の `null` は未設定を指す。作成では `before`、削除では `after` を `null` にする。`SecretChange` は、秘密情報を持つ項目を変えた事実で、値を持たない。秘密情報を持つ項目は、その項目を所有するドメインのユースケースが `SecretChange` で渡し、`ValueChange` に秘密情報を渡さない
- `ValueChange` の前後の値の等しさは、文字列・数・`null` では値の一致、文字列の列では要素と順の一致
- 記録は、一つの操作の、操作の種類と対象の組ごとに一件。一つの操作が同じ種類で複数の対象を変える場合は、対象ごとに一件を記録する
- `schema.` で始まる四つの種類は、追記した `SchemaVersionRecord` の `change.kind` に対応する。対応は [../usecases/schema.md](../usecases/schema.md) の「操作履歴の種類」が定める。`schema.applied` は Schema のドメインイベントの名前で、操作の種類に使わない
- トークンの対象の `token` は、[common.md](common.md) の「TokenDesignation」の値
- 対象が現在もあるかは、[../usecases/audit.md](../usecases/audit.md) の `getAuditLogEntry` が、対象の種類ごとに、対象を所有するドメインの読み取りで求める。種類ごとの条件は、ユースケーステスト（[../testcases/audit/getAuditLogEntry.md](../testcases/audit/getAuditLogEntry.md)）が定める
- 現在と同じ値への変更は、[index.md](index.md) の「変更の無い変更」に従う

### AuditLogFilter

```ts
type AuditActorFilter =
  | Readonly<{ kind: "member"; memberId: MemberId }>
  | Readonly<{ kind: "token"; tokenId: TokenId }>
  | Readonly<{ kind: "operator" }>;

type AuditLogFilter = Readonly<{
  from: Date | null;
  to: Date | null;
  operationTypes: readonly AuditOperationType[] | null;
  actor: AuditActorFilter | null;
}>;

const AuditLogFilter: {
  create(params: {
    from?: Date | null;
    to?: Date | null;
    operationTypes?: readonly string[] | null;
    actor?: AuditActorFilter | null;
  }): AuditLogFilter;
};
```

| 条件 | 意味 |
| --- | --- |
| `from` | `occurredAt` が `from` 以降。`null` は下限なし |
| `to` | `occurredAt` が `to` より前。`null` は上限なし |
| `operationTypes` | `operation.type` がいずれかに一致。`null` は種類で絞らない。重複した値は一つとして扱う |
| `actor` | `member` は、`actor.kind` が `member` で `memberId` が一致。`token` は、`actor.kind` が `appToken` または `memberToken` で `tokenId` が一致。`operator` は、`actor.kind` が `operator`。`null` は実行者で絞らない |

- 複数の条件は、すべてを満たす記録に絞る
- `create` は、`from` が `to` より後なら `ValidationError`（`InvalidPeriod`）、`operationTypes` に `AuditOperationType` でない値があれば `ValidationError`（`UnknownOperationType`）
- 絞り込む実行者は、一覧が返した記録の `AuditActor` から選ぶ。`member` の `memberId` から `member` の条件、`appToken` と `memberToken` の `tokenId` から `token` の条件、`operator` から `operator` の条件を作る

### エラーコード

Audit が返す理由のコードを、すべて列挙する。共通のコード（`ALREADY_EXISTS`、`WORKSPACE_NOT_ACTIVE`、`PERMISSION_MISSING`、`ACTOR_NOT_ALLOWED`、`InvalidPagination`、`INVALID_QUERY`）は [common.md](common.md) にある。

```ts
type AuditErrorCode =
  // ValidationError
  | "InvalidPeriod"
  | "UnknownOperationType"
  // BusinessRuleError
  | "InvalidAuditLabel";
```

理由の条件は、`AuditLabel` と `AuditLogFilter.create` が定める。

## ドメインサービス

無い。規則は `AuditLogEntry` と値オブジェクトが持つ。

## ポート

### AuditLogRepository

目的: 操作履歴の記録を追記し、絞り込んで読む。置き場所は [index.md](index.md) の「集約とポートの置き場所」に従う。追記だけを行う集約のポートなので、`TransactionalRepository` を継がず、持つメソッドを下にすべて宣言する。[common.md](common.md) の「TransactionalRepository」の、可視性と、すべてのリポジトリのメソッドに適用する規則に従う。

```ts
interface AuditLogRepository {
  // 読み取り
  findById(id: AuditLogEntryId): Promise<AuditLogEntry | null>;
  findPage(filter: AuditLogFilter, pagination: Pagination): Promise<PaginationResult<AuditLogEntry>>;
  // 書き込み
  append(entry: AuditLogEntry): Promise<void>;
  appendMany(entries: readonly AuditLogEntry[]): Promise<void>;
  deleteAll(limit: number): Promise<number>;
}
```

| メソッド | 分類 | 契約 |
| --- | --- | --- |
| `findById` | 読み取り | 区画の記録を返す。無ければ `null` |
| `findPage` | 読み取り | 区画の記録のうち、`AuditLogEntry.matches` が真になる記録を、`occurredAt` の新しい順、同じ時刻は ID の昇順で返す。`count` は条件に該当する総件数。最後のページを越えた `page` には、空の列と総件数を返す |
| `append` | 書き込み | 記録を一件追記する。ID の重複は共通の規則（`ALREADY_EXISTS`）。楽観ロックを使わない。ID の違う並行した追記は、どちらも確定する |
| `appendMany` | 書き込み | 複数の記録を、一回の呼び出しで追記する。契約は、記録ごとの `append` と同じで、渡した列の中の ID の重複も違反になる。一件でも違反があれば、どの記録も追記しない。件数の上限を置かない。空の列では何もしない |
| `deleteAll` | 書き込み | 区画の消去。対象は、区画のすべての記録。楽観ロックを使わない |

- 追記と並行した `findPage` は、ページをまたぐと記録の重複または欠落が起こり得る

## トランザクション境界

[common.md](common.md) の「WriteScope」「UnitOfWorkProvider」に従う。`WorkspaceContext` が `auditLogRepository` を公開する。`InstallationContext` は公開しない。

- 操作履歴に残す操作の追記は、[index.md](index.md) の「同じ UnitOfWork で確定するもの」に従う。操作のユースケースが、操作の書き込みを行う、ワークスペースの区画の UnitOfWork の中で `AuditLogEntry.record` と `append` を呼ぶ。二つの区画を書く操作は、ワークスペースの区画の段で追記する。設置の区画だけを書く操作は、操作履歴を残さない
- 追記を含む UnitOfWork の `WriteScope` は、操作のユースケースが定める。追記は条件を変えない
- 対象の件数に比例する記録は、`appendMany` の一回の呼び出しで追記する
- `record` に渡す `now` は、「AuditLogEntry」の `occurredAt` の定義に従う
- 記録に含める名称（実行者、対象）は、書き込みの前に読んだエンティティから求める
- 操作履歴の読み取りは、書き込みを伴わない

## 利用終了の消去

ワークスペースの利用終了で、Installation のユースケースが呼ぶ、Audit の消去。呼ぶ順は [index.md](index.md) の「手順を呼ぶ順」が持つ。

| 呼ぶメソッド | UnitOfWork |
| --- | --- |
| `AuditLogRepository.deleteAll(BoundedDeletion.maxLimit)` | 内。ワークスペースの区画（`workspace` の `none`） |

- 手順の前提（先に消去を終えているべきドメイン）: Token、Form、Webhook、Content、Schema

終了前の確認に公開する項目。型は [common.md](common.md) の「各ドメインが公開する項目」の `DomainKind`。

| 区分 | `kind` | 内容 |
| --- | --- | --- |
| 利用終了で失われるデータの種類 | `audit_log` | 操作履歴 |

- 消去する資格情報と、CMS の通常データの外に残る情報は、Audit には無い
- ワークスペースごとに違う件数を、Audit は公開しない

## 状態確認の項目

無い。Audit は、設置の状態確認に示す項目（`kind`、件数、必要な対処）を持たない。

## ユースケース（概要）

詳細は [../usecases/audit.md](../usecases/audit.md) が定める。

- 操作履歴を一覧する
- 操作履歴の一件を確かめる

記録の追記は、操作履歴に残す操作を行う各ドメインのユースケースが行う（「トランザクション境界」）。記録の削除は、ワークスペースの利用終了（Installation）のユースケースが「利用終了の消去」を呼んで行う。
