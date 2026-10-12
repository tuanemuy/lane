# Token

アプリが示すトークンと、トークンを表示アプリへ渡す経路（プレビュー、スクリプトの認証）を保つ。境界と依存方向は [index.md](index.md) の「依存方向」が定める。ほかの集約は ID（`WorkspaceId`、`MemberId`、`AccountId`、`ModelId`、`FormId`）で参照する。ID、`Permission`、`Actor`、`TokenActor`、`TransactionalRepository`、`SingletonRepository`、`SecretCipher` と `ProtectedSecret`、`KeyBinding` と `BindingTargetChange`、`TokenDesignation` と `MemberTokenOrigin`、`NotFoundTarget` は [common.md](common.md) の定義を使う。

集約の区画は [index.md](index.md) の「集約」、区画とワークスペースの ID の規則は [common.md](common.md) の「集約の ID」に従う。発行元のワークスペースは、`Token` がある区画が決める。

Token はドメインイベントを発行しない。エンティティの振る舞いは、次の状態のエンティティだけを返す。

## ユビキタス言語

| 英語名 | 日本語名 | 定義 |
| --- | --- | --- |
| Token | トークン | アプリが要求に添えて示す値と、その値に許す操作の記録。人のセッションと別 |
| Token value | トークンの値 | アプリが保持し、要求ごとに示す文字列 |
| Value digest | 照合値 | トークンの値から一方向に求めた値。CMS が保存し、示された値の照合に使う |
| Kind | 種類 | 保管先による区分。サーバー用とブラウザ用 |
| Grant | 許可 | トークンに許す操作の単位。公開取得、プレビュー取得、フォーム送信、スキーマ反映、コンテンツ管理 |
| Permission domain | 許可の権限の領域 | 許可が行う操作に対応する、メンバーの権限の集合 |
| Scope | 範囲 | トークンが操作できる対象。ワークスペース全体、またはモデル・フォームの集合 |
| Scope target | 範囲の要素 | 範囲が指す一つのモデルまたはフォーム。紐付け待ち（キー）か、紐付け済み（ID） |
| App-bound token | アプリに紐付くトークン | 手動発行したトークンと、初期構成で宣言したトークン |
| Member-bound token | メンバーに紐付くトークン | スクリプトの認証、またはプレビューの引き換えで発行したトークン。そのメンバーが現在持つ権限の範囲で認可する |
| Issuer | 発行者 | アプリに紐付くトークンを発行したメンバー。初期構成で宣言したトークンでは、初期構成の所有者 |
| Bound member | 紐付くメンバー | メンバーに紐付くトークンが指すメンバー。スクリプトの認証を承認したメンバー、またはプレビューを開いたメンバー |
| Related member | 関係するメンバー | トークンが指すメンバー。アプリに紐付くトークンでは発行者、メンバーに紐付くトークンでは紐付くメンバー（`Token.relatedMemberId`） |
| Issuer permissions | 発行時の権限の組 | アプリに紐付くトークンが保持する、発行時に発行者が持っていた権限の組 |
| Origin restriction | 利用元の限定 | ブラウザ用トークンを使える利用元（Web のオリジン）の指定。本人確認ではない |
| Usage limit | 利用量の制限 | ブラウザ用トークン一つが、期間の中で行える要求の数の上限 |
| Revocation | 失効 | トークンを、次の要求から拒否する状態にすること。取り消せない |
| Token verification | トークンの確認 | 示されたトークンの値、利用元、利用量を確かめて、実行者を求めること。要求の入口が行う |
| Token access | トークンの認可結果 | 実行者のトークンから求めた、その時点の許可・範囲・権限の組。操作ごとに、ユースケースが求める |
| Preview exchange | プレビューの引き換え | プレビューを開く操作で表示アプリへ渡す、一度だけ使える引き換えの値の記録 |
| Exchange code | 引き換えの値 | 表示アプリがサーバー側でトークンに引き換える、一度だけ使える値 |
| Device authorization | スクリプトの認証 | スクリプトの実行環境が、メンバーの承認を経てトークンを得る依頼 |
| Approval URL | 承認用の URL | メンバーがブラウザで開いて承認する URL。設置の公開の URL と承認の値から `DeviceApprovalLink.url` で作る |
| User code | 確認用のコード | コマンドと承認の画面の両方に示し、メンバーが一致を確かめる 8 文字 |
| Device code | 受け取りの値 | コマンドが保持し、承認の結果とトークンを受け取るときに示す値 |
| Preview settings | プレビューの設定 | ワークスペースに一つの、プレビュー先の表示アプリの URL |
| Preview destination | プレビュー先 | プレビューを開く操作でブラウザが移る URL |

## エンティティ

### Token

フィールド。

```ts
type TokenIssuer = Readonly<{ memberId: MemberId; accountId: AccountId }>;
type TokenMember = Readonly<{ memberId: MemberId; accountId: AccountId }>;

type TokenExpiry =
  | Readonly<{ type: "never" }>
  | Readonly<{ type: "at"; at: Date }>;

type ServerProfile = Readonly<{ kind: "server"; grants: ServerGrantSet }>;
type BrowserProfile = Readonly<{
  kind: "browser";
  grants: BrowserGrantSet;
  origins: OriginRestriction;
  usageLimit: TokenUsageLimit | null;
  protectedValue: ProtectedTokenValue;
}>;

type TokenCore = Readonly<{
  id: TokenId;
  valueDigest: TokenValueDigest;
  issuedAt: Date;
  version: number;
}>;

type AppTokenFields = TokenCore &
  Readonly<{
    binding: "app";
    name: TokenName;
    issuer: TokenIssuer;
    issuerPermissions: PermissionSet;
    profile: ServerProfile | BrowserProfile;
    expiry: TokenExpiry;
  }> &
  (
    | Readonly<{ origin: "manual"; scope: WorkspaceScope | BoundSelectionScope }>
    | Readonly<{ origin: "bootstrap"; scope: TokenScope }>
  );

type MemberTokenFields = TokenCore &
  Readonly<{
    binding: "member";
    member: TokenMember;
    scope: WorkspaceScope;
    expiry: Readonly<{ type: "at"; at: Date }>;
  }> &
  (
    | Readonly<{ origin: "device"; profile: ServerProfile }>
    | Readonly<{ origin: "preview"; profile: Readonly<{ kind: "server"; grants: readonly ["preview_read"] }> }>
  );

type Active = Readonly<{ status: "active" }>;
type Revoked = Readonly<{ status: "revoked"; revokedAt: Date }>;

type AppToken = AppTokenFields & (Active | Revoked);
type MemberToken = MemberTokenFields & (Active | Revoked);
type Token = AppToken | MemberToken;
type ActiveToken = (AppTokenFields | MemberTokenFields) & Active;
type RevokedToken = (AppTokenFields | MemberTokenFields) & Revoked;

type TokenStanding = "active" | "expired" | "revoked";
```

| フィールド | 制約 |
| --- | --- |
| `valueDigest` | トークンの値の照合値。照合値の一意性は、[common.md](common.md) の「期限つきの一度きりの値」に従う。生成後に変わらない |
| `issuer` | アプリに紐付くトークンだけが持つ。発行者のメンバーとそのアカウント |
| `member` | メンバーに紐付くトークンだけが持つ。紐付くメンバーとそのアカウント |
| `origin` | `manual` は手動発行、`bootstrap` は初期構成での宣言。`device` と `preview` は [common.md](common.md) の「TokenDesignation」の `MemberTokenOrigin` |
| `name` | アプリに紐付くトークンだけが持つ。メンバーに紐付くトークンは名称を持たず、`origin` で示す |
| `issuerPermissions` | アプリに紐付くトークンだけが持つ。発行時に発行者が持っていた権限の組 |
| `profile` | 種類と、種類ごとに持てる許可。ブラウザ用だけが、利用元の限定、利用量の制限、保護した形の値を持つ |
| `scope` | トークンが操作できる対象（「TokenScope」） |
| `expiry` | 有効期限。期限なし（`never`）または期限（`at`） |
| `version` | 生成時は 0。状態を変える振る舞いごとに 1 増える |

振る舞い。

```ts
type KeyedScopeInput =
  | Readonly<{ type: "workspace" }>
  | Readonly<{ type: "selection"; models: readonly string[]; forms: readonly string[] }>;

type BoundScopeInput =
  | Readonly<{ type: "workspace" }>
  | Readonly<{ type: "selection"; models: readonly ModelId[]; forms: readonly FormId[] }>;

type AppProfileInput =
  | Readonly<{ kind: "server"; grants: readonly string[] }>
  | Readonly<{
      kind: "browser";
      grants: readonly string[];
      origins: readonly string[];
      usageLimit: Readonly<{ max: number; windowMs: number }> | null;
      protectedValue: ProtectedTokenValue;
    }>;

type AppTokenParams = Readonly<{
  id: string;
  name: string;
  issuer: TokenIssuer;
  issuerPermissions: PermissionSet;
  profile: AppProfileInput;
  expiry: TokenExpiry;
  valueDigest: TokenValueDigest;
}> &
  (
    | Readonly<{ origin: "manual"; scope: BoundScopeInput }>
    | Readonly<{ origin: "bootstrap"; scope: KeyedScopeInput }>
  );

type AppTokenInput = Readonly<{
  name: string;
  kind: string;
  grants: readonly string[];
  origins: readonly string[];
  usageLimit: Readonly<{ max: number; windowMs: number }> | null;
  scope: KeyedScopeInput;
  expiry: TokenExpiry | null;
}>;

type MemberTokenParams = Readonly<{
  id: string;
  member: TokenMember;
  valueDigest: TokenValueDigest;
}> &
  (
    | Readonly<{ origin: "device"; grants: ServerGrantSet; expiresAt: Date }>
    | Readonly<{ origin: "preview" }>
  );

type ScopeChange = Readonly<{
  models: BindingTargetChange<ModelId>;
  forms: BindingTargetChange<FormId>;
}>;

type BrowserValue =
  | Readonly<{ outcome: "available"; value: ProtectedTokenValue }>
  | Readonly<{ outcome: "revoked" }>;

type ScopeChangeResult = Readonly<{
  token: AppToken;
  bound: readonly BoundScopeTarget[];
}>;

type RestrictionInput = Readonly<{
  origins: readonly string[];
  usageLimit: Readonly<{ max: number; windowMs: number }> | null;
}>;

type BrowserRestrictions = Readonly<{
  origins: OriginRestriction;
  usageLimit: TokenUsageLimit | null;
}>;

type RestrictionChange =
  | Readonly<{
      outcome: "changed";
      token: AppToken & Active;
      before: BrowserRestrictions;
      after: BrowserRestrictions;
    }>
  | Readonly<{ outcome: "unchanged"; token: AppToken & Active }>
  | Readonly<{ outcome: "revoked"; token: RevokedToken }>;

const Token: {
  assertIssuable(input: AppTokenInput, issuerPermissions: PermissionSet, now: Date): void;
  issueForApp(params: AppTokenParams, now: Date): AppToken & Active;
  issueForMember(params: MemberTokenParams, now: Date): MemberToken & Active;
  revoke(token: Token, now: Date): RevokedToken;
  browserValue(token: Token): BrowserValue;
  label(token: AppToken): string;
  designation(token: Token): TokenDesignation;
  applyScopeChange(token: AppToken, change: ScopeChange, now: Date): ScopeChangeResult;
  changeRestrictions(token: Token, input: RestrictionInput, now: Date): RestrictionChange;
  standing(token: Token, now: Date): TokenStanding;
  lapsedAt(token: Token): Date | null;
  grants(token: Token): readonly TokenGrant[];
  actor(token: Token): TokenActor;
  relatedMemberId(token: Token): MemberId;
  relatedAccountId(token: Token): AccountId;
  shouldRecordUse(lastUsedAt: Date | null, now: Date): boolean;
  isListed(token: Token, now: Date): boolean;
  isAddressable(token: Token, now: Date): boolean;
};
```

| 振る舞い | 処理 |
| --- | --- |
| `assertIssuable` | 「発行時の入力の規則」を、値の生成と保護の前に確かめる。何も保存せず、同じ入力から同じ結果になる |
| `issueForApp` | `assertIssuable` と同じ規則を、渡された値に確かめ、`TokenId` と各値を生成する。`manual` の範囲は `TokenScope.ofBound`、`bootstrap` の範囲は `TokenScope.ofDeclared` で作る。`active` のトークンを返す |
| `issueForMember` | `device` は、渡された許可の組と期限を持つトークンを返す。`preview` は、プレビュー取得の許可だけを持ち、期限が `now` の `TokenLifetimePolicy.previewTokenMs` 後のトークンを返す |
| `revoke` | `revoked` にし、`revokedAt` を `now` にする。期限を過ぎたトークンにも行える。`revoked` のトークンは、渡されたトークンをそのまま返す。エラーにしない |
| `browserValue` | `revoked` のトークンは、種類によらず `revoked` を返す。`active` のブラウザ用トークンは、`profile.protectedValue` を `available` で返す。期限を過ぎたトークンでも返す。`active` のサーバー用トークンは `BusinessRuleError`（`TokenValueNotStored`） |
| `label` | アプリに紐付くトークンの `name` を返す |
| `designation` | [common.md](common.md) の「TokenDesignation」の値を返す |
| `changeRestrictions` | ブラウザ用トークンの利用元の限定と利用量の制限を置き換える。`revoked` のトークンは、種類によらず、変えずに `revoked` を返す。`active` のサーバー用トークン（メンバーに紐付くトークンを含む）は `BusinessRuleError`（`RestrictionNotChangeable`）。それ以外は、`OriginRestriction.create` と `TokenUsageLimit` で入力を作り（違反は、見つけたすべてを項目ごとの理由にした一つの `ValidationError`）、現在と同じ値なら `unchanged`、違えば、置き換えて `version` を増やしたトークンと、変更前後の値を `changed` で返す。期限を過ぎたトークンにも行える。トークンの値、照合値、保護した値は変わらない |
| `applyScopeChange` | 「範囲の更新」の規則で範囲を置き換える。範囲が変われば `version` を増やす。変わらなければ、渡されたトークンをそのまま返し、`bound` は空 |
| `standing` | `revoked` なら `revoked`。期限（`at`）を持ち、`now` で期限切れなら `expired`。それ以外は `active`。期限なし（`never`）のトークンは、期限切れにならない |
| `lapsedAt` | トークンの保持期間の起点を返す（[common.md](common.md) の「期限つきの一度きりの値」）。`active` で期限なしは `null`。`active` で期限を持てば、その期限。`revoked` は、`revokedAt` と期限（持つ場合）の早い方 |
| `grants` | `profile.grants` を返す |
| `actor` | アプリに紐付くトークンは、共通の `Actor` の `appToken`（`tokenId` と、`issuerMemberId` に `issuer.memberId`）を返す。メンバーに紐付くトークンは、`memberToken`（`tokenId` と、`memberId` に `member.memberId`）を返す。`TokenActor` は [common.md](common.md) が定める |
| `relatedMemberId` | 関係するメンバーを返す。アプリに紐付くトークンは `issuer.memberId`、メンバーに紐付くトークンは `member.memberId`。`TokenListFilter` の `relatedMember` と、関係するメンバーの存在の判定が使う |
| `relatedAccountId` | 関係するメンバーのアカウントを返す。アプリに紐付くトークンは `issuer.accountId`、メンバーに紐付くトークンは `member.accountId`。操作履歴の実行者の名称（関係するメンバーのアカウントの名前）を読むのに使う |
| `shouldRecordUse` | 最終利用日時（「TokenRepository」の `recordUse`）が `null`、または `now` が `lastUsedAt` の 60 秒後以降なら真 |
| `isListed` | 「一覧に載るトークン」の条件を満たせば真 |
| `isAddressable` | ID で指す操作の対象にできるトークンなら真。`OneTimeValueRetention.isForgotten(Token.lapsedAt(token), now)` が偽のトークン。状態によらない |

期限切れと期限内の境界は、[common.md](common.md) の「期限つきの一度きりの値」が定める。`Token`、`PreviewExchange`、`DeviceAuthorization` の振る舞いの「期限切れ」「期限内」は、この境界を指す。

[index.md](index.md) の「変更の無い変更」に当たるのは、振る舞いが渡されたエンティティをそのまま返すときと、`changeRestrictions` が `unchanged` を返すとき。

不変条件。

- ブラウザ用トークンが持つ許可は、公開取得とフォーム送信に限る
- メンバーに紐付くトークンは、サーバー用で、範囲がワークスペース全体で、期限を持つ
- 集合の範囲で絞らない許可（「TokenGrant」の表）を持つトークンの範囲は、ワークスペース全体である
- プレビューの引き換えで発行したトークンが持つ許可は、プレビュー取得だけである（型が表す）
- アプリに紐付くトークンの許可は、発行時に発行者が持っていた権限の組の範囲にある
- 種類、許可、有効期限、発行者、紐付くメンバー、発行時の権限の組は、発行の後に変わらない。範囲が変わるのは `applyScopeChange` だけ、ブラウザ用トークンの利用元の限定と利用量の制限が変わるのは `changeRestrictions` だけである。範囲の紐付け待ちの要素は破棄しない。不要なトークンは失効させる
- 手動発行したトークンの範囲は、紐付け待ちの要素を持たない（型が表す）
- 手動発行したトークンの範囲の要素は、発行の時点の現行スキーマにあるモデル・フォームである。現行スキーマに無い要素は、トークンの削除まで範囲に残り、どの対象にも当たらず、範囲を示すときに含めない
- `revoked` のトークンは `active` に戻らない
- どのトークンも、照合値を保存する。サーバー用トークンの値は保存しない。ブラウザ用トークンの値は、第三者に知られる前提の値で、アプリ連携の権限を持つメンバーに後からも示すので、`SecretCipher` で保護した形（`ProtectedTokenValue`）を照合値に加えて保存する

ライフサイクル。

- 生成: 手動発行、初期構成の読み込み、スクリプトの認証の受け取り、プレビューの引き換え
- 遷移: `active` →（`revoke`）→ `revoked`。`active` のまま、`applyScopeChange` が範囲を、`changeRestrictions` がブラウザ用トークンの利用元の限定と利用量の制限を変える。期限切れは状態として保存せず、`standing` が現在時刻から求める
- 終端: 使えなくなったトークンの保持と削除は、[common.md](common.md) の「期限つきの一度きりの値」に従う。定期ジョブ「期限切れの記録の掃除」が `TokenRepository.deleteLapsed` で削除する
- 削除: ワークスペースの利用終了で、そのワークスペースのトークンをすべて削除する（「利用終了の消去」）

#### 発行時の入力の規則

手動発行と初期構成の宣言に共通の規則。`assertIssuable` と `issueForApp` が確かめる。

- 名称は `TokenName`、種類は `TokenKind`、許可は種類ごとの許可の組（`ServerGrantSet` / `BrowserGrantSet`）、利用元の限定は `OriginRestriction`、利用量の制限は `TokenUsageLimit` で作る。違反の理由は、それぞれの値オブジェクトが定める
- 有効期限は、期限なし（`never`）か期限（`at`）のどちらかを必ず指定する。どちらも指定していない入力（`null`）は `ExpiryNotSpecified`。`at` が `now` 以前なら `ExpiryInPast`
- 集合の範囲（`selection`）は、`TokenScope.ofDeclared` の規則で確かめる。手動発行の入力と初期構成の宣言は、集合の範囲をモデル・フォームのキーで指定する（`KeyedScopeInput`）
- 上の違反は、項目ごとの理由を持つ一つの `ValidationError`。違反が無い入力について、次の組の規則を、この順に確かめ、最初に当たった規則の `BusinessRuleError` を返す（[index.md](index.md) の「共通の型とポート」のエラーの種類の、入力の定義の間の整合の違反）
  1. サーバー用の入力は、利用元の限定と利用量の制限を持たない。`origins` が空で `usageLimit` が `null` でなければ `RestrictionNotAllowedForKind`
  2. 集合の範囲で絞らない許可（「TokenGrant」の表）と、集合の範囲を組にした入力は `ScopeNotAllowedForGrant`
  3. `GrantPolicy.refusals` が空でなければ `GrantExceedsIssuerPermissions`
- 初期構成の宣言のキーは、ID にせず、紐付け待ちの要素になる

#### 自分に紐付くトークンの失効

スクリプトの認証の解除。

- スクリプトの認証で発行したトークン（`origin` が `device`）は、そのトークンを示した要求が、`revoke` で失効させられる
- プレビューの引き換えで発行したトークン（`origin` が `preview`）を示した要求は、`ForbiddenError`（`SELF_REVOCATION_NOT_ALLOWED`）
- 失効は、許可と、紐付くメンバーの状態（解除済み、多要素認証を必須にしたワークスペースで未設定）によらない。`TokenAuthorization.resolve` を通さない
- 受け付ける実行者の種類は、[../usecases/token.md](../usecases/token.md) の `revokeOwnToken` が定める

#### 一覧に載るトークン

- `revoked` のトークンは載らない
- メンバーに紐付くトークンは、`standing` が `active` の間だけ載る
- アプリに紐付くトークンは、期限を過ぎても、`now` が期限の `TokenReviewPolicy.expiredListedMs` 後より前で、`Token.isAddressable` が真の間、期限切れとして載る。定期ジョブが削除したかどうかによらない

#### 範囲の更新

`applyScopeChange` の規則。要素の紐付けと外れは [common.md](common.md) の「キーで宣言した設定の紐付け」に従う。`ScopeChange` に渡す値は、[../usecases/schema.md](../usecases/schema.md) の「ほかのドメインとの対応」に従う。Token が定めるのは、結果ごとに範囲へ行うことだけ。

1. 範囲が `workspace` のトークンは変えない
2. 要素ごとに、モデルの要素は `models`、フォームの要素は `forms` を渡して `KeyBinding.apply` を呼ぶ。`bound` は、その ID の紐付け済みの要素に置き換え、置き換えた要素を `bound` で返す。`detached` は、範囲から除く。`unchanged` は残す

- すべての要素を除いたトークンは、範囲が空の集合になる。範囲が空のトークンは、どのモデル・フォームも操作できない。失効はしない

### PreviewExchange

フィールド。

```ts
type PreviewExchangeBase = Readonly<{
  id: PreviewExchangeId;
  member: TokenMember;
  codeDigest: ExchangeCodeDigest;
  version: number;
  createdAt: Date;
  expiresAt: Date;
}>;

type PendingPreviewExchange = PreviewExchangeBase & Readonly<{ status: "pending" }>;
type RedeemedPreviewExchange = PreviewExchangeBase &
  Readonly<{ status: "redeemed"; redeemedAt: Date; tokenId: TokenId }>;
type PreviewExchange = PendingPreviewExchange | RedeemedPreviewExchange;

type PreviewExchangeStanding = "usable" | "expired" | "redeemed";
```

| フィールド | 制約 |
| --- | --- |
| `member` | プレビューを開いたメンバーとそのアカウント。発行するトークンが紐付くメンバーになる。生成後に変わらない |
| `codeDigest` | 引き換えの値の照合値。生成後に変わらない |
| `expiresAt` | `createdAt` の `TokenLifetimePolicy.exchangeCodeMs` 後。生成後に変わらない |
| `version` | 生成時は 0。状態を変える振る舞い（`redeem`）ごとに 1 増える |

引き換えの値は、[common.md](common.md) の「期限つきの一度きりの値」に従う。

振る舞い。

```ts
const PreviewExchange: {
  create(
    params: {
      id: string;
      member: TokenMember;
      codeDigest: ExchangeCodeDigest;
    },
    now: Date,
  ): PendingPreviewExchange;
  redeem(
    exchange: PendingPreviewExchange,
    tokenId: TokenId,
    now: Date,
  ): RedeemedPreviewExchange;
  standing(exchange: PreviewExchange, now: Date): PreviewExchangeStanding;
  lapsedAt(exchange: PreviewExchange): Date;
};
```

| 振る舞い | 処理 |
| --- | --- |
| `create` | 共通の `PreviewExchangeId` を生成し、`pending` の引き換えを返す |
| `redeem` | 期限切れなら `BusinessRuleError`（`PreviewExchangeExpired`）。それ以外は `redeemed` にし、発行したトークンを記録する。期限は `PreviewExchangeProcess.redeem` が先に確かめる |
| `standing` | `pending` で期限内なら `usable`、期限切れなら `expired`。`redeemed` は `redeemed` |
| `lapsedAt` | [common.md](common.md) の「期限つきの一度きりの値」の既定の値を返す |

不変条件。

- 引き換えに使えるのは、`standing` が `usable` の引き換えに限る。一度使うと `redeemed` になり、再び使えない
- 一つの引き換えから発行するトークンは、最大一つである
- 引き換えの値は保存しない。保存するのは照合値である

ライフサイクル。

- 生成: メンバーが編集画面でプレビューを開いたとき。開くたびに一つ作る
- 遷移: `pending` →（`redeem`）→ `redeemed`
- 削除: 定期ジョブ「期限切れの記録の掃除」が、`PreviewExchangeRepository.deleteLapsed` で削除する。ワークスペースの利用終了で、そのワークスペースの引き換えをすべて削除する

### DeviceAuthorization

フィールド。

```ts
type DeviceAuthorizationBase = Readonly<{
  id: DeviceAuthorizationId;
  grants: ServerGrantSet;
  approvalDigest: ApprovalCodeDigest;
  deviceCodeDigest: DeviceCodeDigest;
  userCode: UserCode;
  version: number;
  createdAt: Date;
  expiresAt: Date;
}>;

type DeviceApproval = Readonly<{
  workspaceId: WorkspaceId;
  member: TokenMember;
  approvedAt: Date;
  tokenExpiresAt: Date;
}>;

type PendingDeviceAuthorization = DeviceAuthorizationBase & Readonly<{ status: "pending" }>;
type ApprovedDeviceAuthorization = DeviceAuthorizationBase &
  Readonly<{ status: "approved"; approval: DeviceApproval }>;
type DeviceDenial =
  | Readonly<{ reason: "denied_by_account" }>
  | Readonly<{ reason: "insufficient_permissions"; refusals: readonly GrantRefusal[] }>;

type DeviceRefusal =
  | Readonly<{ reason: "insufficient_permissions"; refusals: readonly GrantRefusal[] }>
  | Readonly<{ reason: "member_unavailable" }>;

type DeniedDeviceAuthorization = DeviceAuthorizationBase &
  Readonly<{ status: "denied"; deniedAt: Date; denial: DeviceDenial }>;
type RedeemedDeviceAuthorization = DeviceAuthorizationBase &
  Readonly<{ status: "redeemed"; approval: DeviceApproval; redeemedAt: Date; tokenId: TokenId }>;
type DeviceAuthorization =
  | PendingDeviceAuthorization
  | ApprovedDeviceAuthorization
  | DeniedDeviceAuthorization
  | RedeemedDeviceAuthorization;

type DeviceAuthorizationStanding = "pending" | "approved" | "denied" | "redeemed" | "expired";
```

| フィールド | 制約 |
| --- | --- |
| `grants` | コマンドが要求した許可。一つ以上。生成後に変わらない |
| `approvalDigest` | 承認用の URL に含める承認の値の照合値 |
| `deviceCodeDigest` | 受け取りの値の照合値 |
| `userCode` | 確認用のコード。コマンドと承認の画面に示す。秘密ではなく、検索に使わない |
| `expiresAt` | `createdAt` の `TokenLifetimePolicy.deviceAuthorizationMs` 後。承認と受け取りの両方の期限。生成後に変わらない |
| `approval.workspaceId` | 承認の場面で、メンバーが自分の所属するワークスペースから選んだ対象。依頼の時点では決まっていない。依頼は設置の区画にあり、ワークスペースを ID で参照する。ワークスペースの存在を、ポートは確かめない |
| `approval.member` | 承認したメンバーとそのアカウント。発行するトークンが紐付くメンバーになる |
| `denial` | 依頼が保存する拒否の理由。`denied_by_account` は承認用の URL を開いたアカウントが承認しなかった、`insufficient_permissions` は、承認の時点でメンバーの権限が要求された許可に足りない（不足を `refusals` に持つ） |
| `tokenId` | `redeemed` のときだけ持つ。受け取りが発行するトークンの ID で、依頼の `id` と同じ値。発行が確定しなかった依頼の `tokenId` は、存在しないトークンを指す |
| `approval.tokenExpiresAt` | `approvedAt` の `TokenLifetimePolicy.deviceTokenMs` 後。承認の画面に示し、発行するトークンの期限にする |
| `version` | 生成時は 0。状態を変える振る舞い（`approve`、`deny`、`redeem`）ごとに 1 増える |

`DeviceRefusal` は、承認の後に成り立たなくなった依頼の受け取りに返す理由。依頼に保存しない。条件は `DeviceAuthorizationProcess.refusal` が定める。

承認の値と受け取りの値は、[common.md](common.md) の「期限つきの一度きりの値」に従う。

振る舞い。

```ts
const DeviceAuthorization: {
  start(
    params: {
      id: string;
      grants: readonly string[];
      approvalDigest: ApprovalCodeDigest;
      deviceCodeDigest: DeviceCodeDigest;
      userCode: UserCode;
    },
    now: Date,
  ): PendingDeviceAuthorization;
  approve(
    authorization: DeviceAuthorization,
    input: {
      workspaceId: WorkspaceId;
      memberId: MemberId;
      accountId: AccountId;
      permissions: PermissionSet;
    },
    now: Date,
  ): ApprovedDeviceAuthorization | DeniedDeviceAuthorization;
  deny(authorization: DeviceAuthorization, now: Date): DeniedDeviceAuthorization;
  tokenExpiresAt(authorization: DeviceAuthorization, now: Date): Date;
  redeem(
    authorization: ApprovedDeviceAuthorization,
    tokenId: TokenId,
    now: Date,
  ): RedeemedDeviceAuthorization;
  standing(authorization: DeviceAuthorization, now: Date): DeviceAuthorizationStanding;
  lapsedAt(authorization: DeviceAuthorization): Date;
};
```

| 振る舞い | 処理 |
| --- | --- |
| `start` | 共通の `DeviceAuthorizationId` と `ServerGrantSet` を生成し、`pending` の依頼を返す。ワークスペースを指定しない |
| `approve` | `pending` でなければ `BusinessRuleError`（`DeviceAuthorizationNotPending`）。期限切れなら `DeviceAuthorizationExpired`。`GrantPolicy.refusals(grants, permissions)` が空でなければ、エラーにせず、`denial` が `insufficient_permissions`（その結果を `refusals` に持つ）の `denied` を返す。空なら `approved` にし、`approval` に、渡されたワークスペース、メンバーとアカウント（`member`）、`approvedAt`（`now`）、`tokenExpiresAt` を記録する |
| `deny` | 承認用の URL を開いたアカウントが、承認しないことを選んだときに呼ぶ。ワークスペースとメンバーを受け取らない。`pending` でなければ `DeviceAuthorizationNotPending`。`denial` が `denied_by_account` の `denied` にする。期限を過ぎた `pending` にも行える |
| `tokenExpiresAt` | 発行するトークンの有効期限を返す。`approved` と `redeemed` は `approval.tokenExpiresAt`。それ以外は、`now` に承認した場合の期限（`now` の `TokenLifetimePolicy.deviceTokenMs` 後）。承認の前に、承認の画面へ示す |
| `redeem` | 期限切れなら `BusinessRuleError`（`DeviceAuthorizationExpired`）。それ以外は `redeemed` にし、発行するトークンの ID を記録する。期限は `DeviceAuthorizationProcess.redeem` が先に確かめる |
| `standing` | `pending` と `approved` は、期限切れなら `expired`。それ以外は `status` を返す |
| `lapsedAt` | [common.md](common.md) の「期限つきの一度きりの値」の既定の値を返す |

不変条件。

- 承認できるのは、`standing` が `pending` の依頼に限る。トークンを受け取れるのは、`standing` が `approved` の依頼に限る
- `approved` の依頼の `approval.member` は、承認の時点で、要求された許可に足りる権限を持っていた（`approve`）
- `denied` の依頼は、ほかの状態に変わらない
- 一つの承認から発行するトークンは、最大一つである。`redeemed` にできるのは承認ごとに一度だけで、`redeemed` にした受け取りだけが、トークンを発行する。発行するトークンの ID は依頼の `id` なので、復旧の置き換えで `approved` の写しに戻った依頼の受け取りも、同じ承認のワークスペースへの、先に確定したトークンと同じ ID の `insert` になり、二つ目のトークンを作らない。復旧の置き換えで `pending` の写しに戻った依頼への承認は、置き換えの前の承認と別の承認である
- 承認の値、受け取りの値、トークンの値は保存しない。トークンの値は、受け取りの値を示したコマンドにだけ返し、ブラウザに示さない

ライフサイクル。

- 生成: スクリプトの実行環境で認証のコマンドを実行したとき。コマンドは、必要な許可だけを指定する
- 遷移: `pending` →（`approve`）→ `approved` →（`redeem`）→ `redeemed`。`pending` →（`approve`。承認の時点の権限の不足）→ `denied`。`pending` →（`deny`）→ `denied`
- 終端: `denied` と `redeemed`。期限切れは状態として保存せず、`standing` が現在時刻から求める。期限切れの `approved` は、どの遷移も持たない。期限切れの `pending` が持つ遷移は、`deny` による `denied` だけ
- 削除: 定期ジョブ「期限切れの記録の掃除」が、設置の区画で、`DeviceAuthorizationRepository.deleteLapsed` で削除する。ワークスペースの利用終了での扱いは「利用終了の消去」が定める

### PreviewSettings

フィールド。

```ts
type PreviewSettingsBase = Readonly<{
  version: number;
  createdAt: Date;
  updatedAt: Date;
}>;

type UnsetPreviewSettings = PreviewSettingsBase & Readonly<{ status: "unset" }>;
type ConfiguredPreviewSettings = PreviewSettingsBase &
  Readonly<{ status: "configured"; baseUrl: PreviewBaseUrl }>;
type PreviewSettings = UnsetPreviewSettings | ConfiguredPreviewSettings;
```

| フィールド | 制約 |
| --- | --- |
| `baseUrl` | プレビュー先の表示アプリの URL。`configured` のときだけ持つ |
| `version` | 生成時は 0。状態を変える振る舞いごとに 1 増える |

振る舞い。

```ts
const PreviewSettings: {
  create(now: Date): UnsetPreviewSettings;
  setUrl(settings: PreviewSettings, baseUrl: string, now: Date): ConfiguredPreviewSettings;
  clearUrl(settings: PreviewSettings, now: Date): UnsetPreviewSettings;
  baseUrl(settings: PreviewSettings): PreviewBaseUrl | null;
};
```

| 振る舞い | 処理 |
| --- | --- |
| `create` | URL が未設定（`unset`）の設定を返す |
| `setUrl` | `PreviewBaseUrl` を生成し（違反は `ValidationError`）、`configured` にする。同じ URL の `configured` なら、渡された設定をそのまま返す |
| `clearUrl` | `unset` にする。`unset` の設定は、渡された設定をそのまま返す |
| `baseUrl` | `configured` なら `baseUrl`、`unset` なら `null` を返す |

`PreviewSettings` は、URL の保持だけを持つ。プレビュー先と、開けない理由は「PreviewDestination」が決める。

不変条件。

- 区画に一つの集約。区画にあることの規則は [index.md](index.md) の「ワークスペースの作成と利用終了」に従う
- URL は、`PreviewBaseUrl` の形を満たす

ライフサイクル。

- 生成: 「初期構成の宣言」の生成
- 遷移: `unset` →（`setUrl`）→ `configured` →（`clearUrl`）→ `unset`。`configured` →（`setUrl`）→ `configured`
- 削除: 「利用終了の消去」

## 値オブジェクト

ブランド付きの型は、`create` だけが生成する。利用者の入力と初期構成の宣言から作る値（`TokenName`、`TokenKind`、許可の組、範囲、`WebOrigin`、`OriginRestriction`、`TokenUsageLimit`、`TokenValue`、`PreviewBaseUrl`）の違反は、項目と理由を持つ `ValidationError` を投げる。それ以外の値（照合値、保護した値、`UserCode`）の違反は `BusinessRuleError` を投げる。理由のコードは「エラーコード」が列挙する。

| 名前 | 型 | バリデーションルール | 等価性 |
| --- | --- | --- | --- |
| `TokenName` | `string`（ブランド） | 前後の空白を除いて 1 文字以上。違反は `InvalidTokenName` | 値の一致 |
| `TokenKind` | `"server" \| "browser"` | `TokenKind.create(value: string)` は、どちらでもない値を `UnknownTokenKind` で拒否する | 値の一致 |
| `TokenGrant` | 文字列リテラルの直和型 | 下の定義 | 値の一致 |
| `ServerGrantSet`、`BrowserGrantSet` | `readonly TokenGrant[]`（ブランド） | 下の定義 | 要素の一致 |
| `GrantPolicy` | 関数 | 下の定義 | — |
| `TokenScope` | 直和型 | 下の定義 | 種類と要素の一致 |
| `TokenValue` | `string`（ブランド） | 下の定義。保存せず、ログにも出さない | 比較しない |
| `TokenValueDigest` | `string`（ブランド） | 空でない。トークンの値の照合値 | 値の一致 |
| `ProtectedTokenValue` | `ProtectedSecret`（ブランド） | `SecretCipher.protect` が返したブラウザ用トークンの値。`ProtectedTokenValue.wrap(secret: ProtectedSecret)` で包む | 値の一致 |
| `WebOrigin` | `string`（ブランド） | `http` または `https` のスキーム、ホスト、省略できるポートからなる。経路、クエリ、フラグメント、認証情報を持たない。スキームとホストを小文字にして保持する。違反は `InvalidOrigin` | 保持する値の一致 |
| `OriginRestriction` | 直和型 | 下の定義 | 種類と、並べた `origins` の列の一致 |
| `TokenUsageLimit` | `Readonly<{ max: number; windowMs: number }>` | `max` と `windowMs`（ミリ秒）は整数で、`max` の範囲は APP-12 の表の「ブラウザ用トークンの利用量の制限の回数の範囲」、`windowMs` の範囲は「ブラウザ用トークンの利用量の制限の期間の範囲」の行が定める。違反は `InvalidUsageLimit`。`UsageLimiter` の `UsageLimit` としてそのまま渡せる | 値の一致 |
| `TokenUsagePolicy` | 関数 | `key(tokenId: TokenId, limit: TokenUsageLimit): string` は、`token:`、ID、`:`、`limit.max`、`:`、`limit.windowMs` を続けた、`UsageLimiter` のキーを返す。制限を変えたトークンは、新しいキーで利用を数え直す。同じキーに違う `UsageLimit` を渡さない（[common.md](common.md) の「UsageLimiter」） | — |
| `TokenLifetimePolicy` | 定数 | 下の定義 | — |
| `TokenReviewPolicy` | 定数と関数 | 下の定義 | — |
| `TokenListFilter` | オブジェクトと関数 | 下の定義 | — |
| `ExchangeCodeDigest` | `string`（ブランド） | 空でない。引き換えの値の照合値 | 値の一致 |
| `ApprovalCodeDigest` | `string`（ブランド） | 空でない。承認の値の照合値 | 値の一致 |
| `DeviceCodeDigest` | `string`（ブランド） | 空でない。受け取りの値の照合値 | 値の一致 |
| `UserCode` | `string`（ブランド） | 下の定義 | 値の一致 |
| `DeviceApprovalLink` | 定数と関数 | 下の定義 | — |
| `PreviewBaseUrl` | `string`（ブランド） | 下の定義 | 値の一致 |
| `PreviewDestination` | 直和型と関数 | 下の定義 | — |
| `TokenAccess` | 直和型 | `TokenAuthorization.resolve` が生成する | — |

トークンの値、引き換えの値、承認の値、受け取りの値は、公開データ、通常のログ、操作履歴に含めない。

### TokenGrant

```ts
type TokenGrant =
  | "public_read"
  | "preview_read"
  | "form_submit"
  | "schema_apply"
  | "content_manage";

type ServerGrant = TokenGrant;
type BrowserGrant = "public_read" | "form_submit";

const TokenGrant: {
  permissionDomain(grant: TokenGrant): PermissionSet;
};

const ServerGrantSet: { create(values: readonly string[]): ServerGrantSet };
const BrowserGrantSet: { create(values: readonly string[]): BrowserGrantSet };
```

| 値 | 許可 | 行える操作 | 権限の領域 | 集合の範囲 |
| --- | --- | --- | --- | --- |
| `public_read` | 公開取得 | 公開内容の取得 | 空 | 絞る |
| `preview_read` | プレビュー取得 | 最新の内容の取得 | 空 | 絞る |
| `form_submit` | フォーム送信 | フォームの送信 | 空 | 絞る |
| `schema_apply` | スキーマ反映 | スキーマの反映と直接反映、適用状況の確認、適用の事前確認、適用済みとしての記録、移行の適用 | `schema:change` | 絞らない。操作はワークスペースのスキーマ全体を対象にする |
| `content_manage` | コンテンツ管理 | コンテンツとメディアの操作 | `content:edit`、`content:publish`、`content:delete`、`media:publish` | 絞る |

- 「集合の範囲」の列は、その許可の操作を、モデル・フォームの集合の範囲で絞るかを定める。絞らない許可の操作は、ワークスペース全体（`AccessTarget` の `workspace`）を対象にする
- `create` は、語彙にない値を `UnknownGrant`、種類が持てない値を `GrantNotAllowedForKind`、空の入力を `EmptyGrants` で拒否する。重複を除き、語彙の定義順に並べる
- トークンは、この表の許可以外の操作を行えない。回答の閲覧・削除、運用設定とメンバーの変更は、どの許可にも含まれない

### GrantPolicy

```ts
type GrantRefusal = Readonly<{ grant: TokenGrant; requiresAnyOf: PermissionSet }>;

const GrantPolicy: {
  refusals(grants: readonly TokenGrant[], permissions: PermissionSet): readonly GrantRefusal[];
  effectivePermissions(grants: readonly TokenGrant[], base: PermissionSet): PermissionSet;
};
```

トークンに与えられる許可は、発行の時点で関係するメンバーが持つ権限の範囲に限る。この規則を、発行と承認と、要求の認可が使う。

- `refusals`: 許可を与えられるのは、その許可の権限の領域が空であるか、領域と `permissions` の `PermissionSet.intersect` が空でない場合である。与えられない許可を、その領域とともに返す。空なら、すべての許可を与えられる
- `effectivePermissions`: `grants` の権限の領域の和と、`base` の `PermissionSet.intersect` を返す。トークンで行う操作は、この組の範囲に限る

### TokenScope

```ts
type Bound<TId> = Extract<KeyBinding<TId>, { status: "bound" }>;
type Pending<TId> = Extract<KeyBinding<TId>, { status: "pending" }>;

type ScopeTarget =
  | Readonly<{ kind: "model"; binding: KeyBinding<ModelId> }>
  | Readonly<{ kind: "form"; binding: KeyBinding<FormId> }>;
type BoundScopeTarget =
  | Readonly<{ kind: "model"; binding: Bound<ModelId> }>
  | Readonly<{ kind: "form"; binding: Bound<FormId> }>;
type PendingScopeTarget =
  | Readonly<{ kind: "model"; binding: Pending<ModelId> }>
  | Readonly<{ kind: "form"; binding: Pending<FormId> }>;

type WorkspaceScope = Readonly<{ type: "workspace" }>;
type SelectionScope = Readonly<{ type: "selection"; targets: readonly ScopeTarget[] }>;
type BoundSelectionScope = Readonly<{ type: "selection"; targets: readonly BoundScopeTarget[] }>;
type TokenScope = WorkspaceScope | SelectionScope;

type AccessTarget =
  | Readonly<{ kind: "workspace" }>
  | Readonly<{ kind: "model"; id: ModelId }>
  | Readonly<{ kind: "form"; id: FormId }>;

const TokenScope: {
  workspace(): WorkspaceScope;
  ofBound(input: { models: readonly ModelId[]; forms: readonly FormId[] }): BoundSelectionScope;
  ofDeclared(input: { models: readonly string[]; forms: readonly string[] }): SelectionScope;
  resolveKeys(
    input: { models: readonly string[]; forms: readonly string[] },
    existing: {
      modelIds: Readonly<Record<string, ModelId>>;
      formIds: Readonly<Record<string, FormId>>;
    },
  ): { models: readonly ModelId[]; forms: readonly FormId[] };
  covers(scope: TokenScope, target: AccessTarget): boolean;
  pendingTargets(scope: TokenScope): readonly PendingScopeTarget[];
  isEmpty(scope: TokenScope): boolean;
};
```

- 要素の紐付けの状態は、[common.md](common.md) の「キーで宣言した設定の紐付け」の `KeyBinding`
- `ofBound` は紐付け済みの要素、`ofDeclared` は `ModelKey` / `FormKey` を生成して、そのキーの紐付け待ちの要素を作る。どちらも、要素が一つも無い入力を `EmptyScope` で拒否し、重複を除く
- `resolveKeys` は、`input` のキーを、`existing`（`WorkspaceSchema` の `modelIds` / `formIds`。現行スキーマのキーから ID への対応）で ID にして返す。`existing` に無いキーがあれば、`BusinessRuleError`（`ScopeTargetNotFound`）を投げる。現行スキーマの状態に依る規則なので、入力の違反と分ける。エラーは、該当する対象（種類とキー）の一覧を持つ
- `covers` の規則
  - `workspace` の範囲は、すべての対象を含む
  - `selection` の範囲は、対象と同じ種類で同じ ID の紐付け済みの要素を持つ場合に、その対象を含む。紐付け待ちの要素は、どの対象も含まない
  - 対象が `workspace`（集合の範囲で絞らない許可の操作）は、`workspace` の範囲だけが含む
- モデルにもフォームにも属さない対象（メディア）への操作は、範囲で絞らない。許可と権限で判定する
- 複数のモデルにまたがる読み取りは、許可と権限を `TokenAuthorization.authorize` で判定し、ユースケースが、`covers` が真になるモデルの対象に結果を限る。当たる読み取りは、[../usecases/content.md](../usecases/content.md) の「実行者と認可」が定める
- `workspace` の範囲は、発行の後に追加したモデル・フォームも含む
- `isEmpty` は、`selection` で要素が無ければ真

### TokenValue

```ts
const TokenValue: {
  prefixes: { server: "lane_sk_"; browser: "lane_pk_" };
  compose(kind: TokenKind, secret: string): TokenValue;
  create(kind: TokenKind, raw: string): TokenValue;
  secretPart(raw: string): string | null;
};
```

- トークンの値は、種類の接頭辞に、予測できない値を続けた文字列である。接頭辞は種類を示し、秘密ではない
- `compose` は、`SecretTokenIssuer.issueForWorkspace` が、発行元のワークスペースの ID を含めて生成した値に、種類の接頭辞を付ける
- `secretPart` は、示された値から種類の接頭辞を除いた部分を返す。どちらの接頭辞でも始まらない値には `null` を返す。発行元のワークスペースの ID は、この部分を `SecretTokenIssuer.workspaceOf` に渡して求める。`null` の値は、発行元が分からない値として扱う
- `create` は、初期構成で宣言した値を検査する。種類の接頭辞で始まり、接頭辞の後が英数字・`-`・`_` の 32 文字以上でなければ `InvalidTokenValue`
- 値から求めた発行元のワークスペースと、その区画のトークンから、確かめるトークンを決める規則は、「TokenAuthentication」の `identify` が定める
- 照合値は、接頭辞を含む値の全体から `SecretTokenIssuer.digest` で求める。示されたトークンは、同じ計算の照合値で引く
- サーバー用トークンの値は、発行の結果にだけ示す。初期構成で宣言したトークンの値は、宣言した人が保管し、CMS は示さない

### OriginRestriction

```ts
type OriginRestriction =
  | Readonly<{ type: "any" }>
  | Readonly<{ type: "listed"; origins: readonly WebOrigin[] }>;

const OriginRestriction: {
  create(origins: readonly string[]): OriginRestriction;
  permits(restriction: OriginRestriction, origin: string | null): boolean;
};
```

- `create` は、空の入力から `any`、1 件以上の入力から `listed` を作る。`origins` は、重複を除き、保持する値の文字列のコードポイントの昇順に並べる。重複を除いた件数が APP-12 の表の「ブラウザ用トークンの利用元の件数の上限」を超える入力と、`WebOrigin` にならない値は `InvalidOrigin`
- `permits` は、`any` なら真。`listed` なら、要求が示した利用元から生成した `WebOrigin` が `origins` にあるときだけ真。利用元を示さない要求と、`WebOrigin` にならない値は偽
- 利用元は要求が自己申告する値で、本人確認にならない

### TokenLifetimePolicy

```ts
const TokenLifetimePolicy: {
  readonly deviceTokenMs: number;
  readonly previewTokenMs: number;
  readonly exchangeCodeMs: number;
  readonly deviceAuthorizationMs: number;
};
```

値は、[../scenario/app-integration.md](../scenario/app-integration.md) の APP-12 の表の、用途の欄と同じ名前の行が定める。

| 定数 | 用途 |
| --- | --- |
| `deviceTokenMs` | スクリプトの認証で発行するトークンの有効期間 |
| `previewTokenMs` | プレビューの引き換えで発行するトークンの有効期間 |
| `exchangeCodeMs` | 引き換えの値の有効期間 |
| `deviceAuthorizationMs` | スクリプトの認証の依頼の有効期間 |

### TokenReviewPolicy

見直しの条件。トークンの一覧が、トークンごとの区別と絞り込みに使う。

```ts
const TokenReviewPolicy: {
  readonly expiringSoonMs: number;
  readonly unusedMs: number;
  readonly expiredListedMs: number;
  isExpiringSoon(token: Token, now: Date): boolean;
  isLongUnused(token: Token, lastUsedAt: Date | null, now: Date): boolean;
  isRelatedMemberRemoved(relatedMemberPresent: boolean): boolean;
};
```

`expiringSoonMs`（有効期限が近いとする期間）、`unusedMs`（長く使われていないとする期間）、`expiredListedMs`（期限切れのトークンを一覧に残す期間）の値は、[../scenario/app-integration.md](../scenario/app-integration.md) の APP-12 の表の「有効期限が近いとする期間」「長く使われていないとする期間」「期限切れのトークンを一覧に残す期間」の行が定める。

| 関数 | 真になる条件 |
| --- | --- |
| `isExpiringSoon` | 有効期限が近いトークン。`standing` が `active` で、期限（`at`）を持ち、期限が `now` の `expiringSoonMs` 後以前 |
| `isLongUnused` | 長く使われていないトークン。`standing` が `active` で、`lastUsedAt`（一度も使われていなければ発行日時）が `now` の `unusedMs` 前以前 |
| `isRelatedMemberRemoved` | 関係するメンバーが解除されたトークン。`relatedMemberPresent` が偽。`relatedMemberPresent` の意味は「TokenListFilter」が定める |

- `isExpiringSoon` は、トークンが持つ値と `now` だけから、`isLongUnused` は、それに最終利用日時を加えて決まる
- トークンごとの「関係するメンバーが解除された」「長く使われていない」の区別と、絞り込みは、同じ値（`ListedToken` の `relatedMemberPresent` と `lastUsedAt`）に依る

### TokenListFilter

トークンの一覧の絞り込み。条件は、トークンが持つ値、`now`、`lastUsedAt`、`relatedMemberPresent` から決まる。

```ts
type TokenOrigin = "manual" | "bootstrap" | MemberTokenOrigin;

type TokenListFilter = Readonly<{
  origins?: readonly TokenOrigin[];
  relatedMember?: MemberId;
  expiringSoon?: true;
  longUnused?: true;
  relatedMemberRemoved?: true;
  pendingScopeOnly?: true;
}>;

type ListedToken = Readonly<{ token: Token; lastUsedAt: Date | null; relatedMemberPresent: boolean }>;

const TokenListFilter: {
  matches(
    token: Token,
    filter: TokenListFilter,
    context: Readonly<{ now: Date; lastUsedAt: Date | null; relatedMemberPresent: boolean }>,
  ): boolean;
};
```

`matches` は、`Token.isListed(token, context.now)` が真で、指定した条件をすべて満たせば真。条件を指定しない絞り込みは、一覧に載るすべてのトークンに真。

| 条件 | 満たすトークン |
| --- | --- |
| `origins` | 発行の経路（`origin`）が、いずれかに一致する。メンバーの利用の解除の確認（Workspace）が、`relatedMember` と組にして使う |
| `relatedMember` | `Token.relatedMemberId(token)` が一致する。メンバーの利用の解除の確認（Workspace）が、そのメンバーが発行したトークンを読むのに使う |
| `expiringSoon` | `TokenReviewPolicy.isExpiringSoon(token, now)` が真 |
| `longUnused` | `TokenReviewPolicy.isLongUnused(token, context.lastUsedAt, now)` が真 |
| `relatedMemberRemoved` | `TokenReviewPolicy.isRelatedMemberRemoved(context.relatedMemberPresent)` が真 |
| `pendingScopeOnly` | `TokenScope.pendingTargets(scope)` が空でない |

- `expiringSoon`、`longUnused`、`relatedMemberRemoved`、`pendingScopeOnly` は、`true` を指定したときだけ絞る。`false` は型が受け付けない
- `relatedMemberPresent` は、トークンと同じ区画に、`Token.relatedMemberId` の `Member` があること。`TokenRepository.findPage` だけが、区画の `Member` から求める。メンバーの存在は Workspace が保ち、トークンは、関係するメンバーの解除を自分の状態として持たない

### UserCode

```ts
const UserCode: {
  derive(secret: string): UserCode;
  create(raw: string): UserCode;
  format(code: UserCode): string;
};
```

- 確認用のコードは、`BCDFGHJKLMNPQRSTVWXZ` の 20 文字からなる 8 文字。違反は `InvalidUserCode`
- `derive` は、`SecretTokenIssuer.issue` が生成した値の先頭 8 文字それぞれの文字コードを 20 で割った余りで、文字を選ぶ
- `format` は、4 文字ごとに `-` で区切った表示用の文字列を返す
- 確認用のコードは、メンバーが承認の画面とコマンドの一致を目で確かめるための値である。依頼は承認の値で引く

### DeviceApprovalLink

承認用の URL の形。URL の組み立ての分担は [index.md](index.md) の「URL の組み立て」が定める。

```ts
const DeviceApprovalLink: {
  readonly path: "/device-authorizations";
  url(publicUrl: string, approvalCode: string): string;
};
```

- `url` は、`path` と承認の値（`approvalCode`）から、[common.md](common.md) の「期限つきの一度きりの値」の「URL の組み立て」の規則で URL を作る
- 承認の画面は、この経路で承認の値を受ける

### PreviewBaseUrl

- 絶対 URL。スキームは `http` または `https`
- 認証情報、クエリ、フラグメントを持たない
- 末尾の `/` を除いて保持する
- 違反は `InvalidPreviewBaseUrl`

### PreviewDestination

プレビューを開く操作で、開けるかどうかと、開けない理由を決め、開ける場合にブラウザが移る URL を作る。集約を受け取らない。入力は、URL の設定、経路の解決の結果、最新の内容についての Content の結果の事実だけ。

```ts
type PreviewDestination =
  | Readonly<{ resolved: true; baseUrl: PreviewBaseUrl; path: string }>
  | Readonly<{ resolved: false; reason: "path_not_defined" }>
  | Readonly<{ resolved: false; reason: "base_url_not_set" }>
  | Readonly<{ resolved: false; reason: "path_values_missing"; fields: readonly FieldKey[] }>
  | Readonly<{ resolved: false; reason: "draft_has_type_errors" }>;

const PreviewDestination: {
  resolve(input: {
    baseUrl: PreviewBaseUrl | null;
    path: PreviewPathResolution;
    draftHasTypeErrors: boolean;
  }): PreviewDestination;
  url(destination: Extract<PreviewDestination, { resolved: true }>, exchangeCode: string): string;
};
```

`resolve` は、上の行から順に当てはめ、最初に当たった行の結果を返す。開けない理由が重なる場合の優先順は、この順。

| 順 | 条件 | 結果 |
| --- | --- | --- |
| 1 | `path` が `not_defined`（経路を定義していないモデル） | `path_not_defined` |
| 2 | `baseUrl` が `null` | `base_url_not_set` |
| 3 | `path` が `values_missing`（経路に使う値が未設定） | `path_values_missing`。`fields` は、`path.fields`（未設定の項目のキー） |
| 4 | `draftHasTypeErrors` が真 | `draft_has_type_errors` |
| 5 | 上のどれでもない（`path` は `resolved`） | `resolved: true`。`baseUrl` と、`path.path` を持つ |

- `path` は、Schema の `PreviewPathResolution`（[schema.md](schema.md) の「PreviewPath」）。Token は、この型を定義し直さない
- `baseUrl` は `PreviewSettings.baseUrl` の結果
- `draftHasTypeErrors` は、最新の内容が、型を満たさない入力エラーのある下書きであること。最新の内容が公開内容なら偽。判定の規則（プレビュー取得が返さない下書き）は Content が定める
- `resolved` が偽の結果は、開けない理由として出力で示す。`PreviewExchange` を作らない
- ごみ箱のコンテンツと存在しないコンテンツは、`resolve` の対象でない

`url` は、`destination.baseUrl`、`destination.path`、`?preview_code=`、`exchangeCode`（引き換えの値）をこの順に連結した URL を返す。`exchangeCode` は URL のクエリの値としてパーセントエンコードする。`destination.path` はクエリとフラグメントを持たない（[schema.md](schema.md) の「PreviewPath」）。

- プレビュー先は、運用設定の URL と、スキーマで定義した経路だけから決まる。要求が指定した URL へは移らない。プレビュー先に含めるのは引き換えの値で、トークンの値を含めない

### TokenAccess

```ts
type TokenVerification =
  | Readonly<{
      status: "verified";
      actor: TokenActor;
      workspaceId: WorkspaceId;
      usageLimit: TokenUsageLimit | null;
    }>
  | Readonly<{ status: "rejected"; reason: "revoked" | "expired" | "origin_not_allowed" }>;

type TokenRejection =
  | "workspace_not_available"
  | "revoked"
  | "expired"
  | "member_removed"
  | "mfa_setup_required";

type GrantedTokenAccess = Readonly<{
  status: "granted";
  actor: TokenActor;
  kind: TokenKind;
  grants: readonly TokenGrant[];
  scope: TokenScope;
  permissions: PermissionSet;
  designation: TokenDesignation;
  relatedAccountId: AccountId;
}>;

type TokenAccess =
  | GrantedTokenAccess
  | Readonly<{ status: "rejected"; reason: TokenRejection }>;
```

`TokenVerification` は、要求の入口が呼ぶ「示されたトークンを確かめる」（`authenticateToken`）の結果で、`TokenAuthentication.verify` が生成する。`TokenAccess` は、各ユースケースが操作ごとに求める認可結果で、`TokenAuthorization.resolve` が生成する。利用できないワークスペースへの入口ごとの結果は [index.md](index.md) の「利用できるワークスペース」の表が、拒否の変種ごとのエラーは「エラーコード」の表が定める。

| 結果 | 意味 |
| --- | --- |
| `TokenAuthentication.identify` の `unknown` | 示された値に該当するトークンが無い。当たる入力は「TokenAuthentication」の `identify` が定める |
| `TokenAuthentication.identify` の `workspace_not_available` | 値から分かった発行元のワークスペースが利用できない。当たる入力は `identify` が定める |
| `verified` | 示された値のトークンが、失効も期限切れもしておらず、利用元が許されている |
| `TokenVerification` の `rejected`: `revoked` | 失効した |
| `TokenVerification` の `rejected`: `expired` | 期限を過ぎた |
| `TokenVerification` の `rejected`: `origin_not_allowed` | 利用元を限定したブラウザ用トークンを、許可していない利用元から使った |
| `granted` | トークンを使える。`grants`、`scope`、`permissions` の範囲で操作を行える。操作の可否は `TokenAuthorization.authorize` が判定する。`designation` は `Token.designation`、`relatedAccountId` は `Token.relatedAccountId` |
| `TokenAccess` の `rejected`: `workspace_not_available` | 発行元のワークスペースが利用できない |
| `TokenAccess` の `rejected`: `revoked` | 確かめた後に失効した |
| `TokenAccess` の `rejected`: `expired` | 確かめた後に期限を過ぎた |
| `TokenAccess` の `rejected`: `member_removed` | 利用できるワークスペースに、紐付くメンバーがいない（メンバーが解除された） |
| `TokenAccess` の `rejected`: `mfa_setup_required` | 紐付くメンバーが、多要素認証を必須にしたワークスペースで、多要素認証を設定していない |

## ドメインサービス

どのドメインサービスも、保存を伴わない純粋な関数で、ポートを呼ばない。ワークスペースが利用できるかどうかの受け取り方は、[index.md](index.md) の「利用できるワークスペース」に従う。読むものと UnitOfWork の分け方は [../usecases/token.md](../usecases/token.md) が、途中で止まったときに残る状態は、ユースケーステスト（`spec/testcases/token/`）が定める。

### TokenAuthentication

責務: 示された値から、確かめるトークンを決め、そのトークンが、その時点で使える値であることと、要求の利用元が許されていることを確かめて、実行者を求める。許可、範囲、権限は判定しない。

```ts
type IssuerLookup =
  | Readonly<{ found: false }>
  | Readonly<{ found: true; workspaceId: WorkspaceId; available: false }>
  | Readonly<{ found: true; workspaceId: WorkspaceId; available: true; token: Token | null }>;

type TokenIdentification =
  | Readonly<{ outcome: "identified"; token: Token; workspaceId: WorkspaceId }>
  | Readonly<{ outcome: "unknown" }>
  | Readonly<{ outcome: "workspace_not_available" }>;

const TokenAuthentication: {
  identify(input: { embedded: IssuerLookup; declared: IssuerLookup }): TokenIdentification;
  verify(
    input: { token: Token; workspaceId: WorkspaceId; origin: string | null },
    now: Date,
  ): TokenVerification;
};
```

`identify` は、発行元のワークスペースを求めた結果から、確かめるトークンを決める。入力は、ユースケースが読んだ事実だけ。

- `embedded` は、値が含むワークスペースの ID（`SecretTokenIssuer.workspaceOf`）から求めた発行元。`declared` は、照合値を宣言したトークンの索引（`DeclaredTokenIndex`）で引いて求めた発行元
- `found: false` は、その経路ではワークスペースの ID を求められないこと。`embedded` では `SecretTokenIssuer.workspaceOf` が `null` を返した値、`declared` では索引に照合値の項目が無い値。値を示していない要求と、`TokenValue.secretPart` が `null` を返した値は、どちらの経路も `found: false`
- `found: true` は、ワークスペースの ID を求められたこと。`available` は、そのワークスペースが利用できるかどうか。`token` は、利用できるワークスペースの区画で、照合値に該当したトークン。無ければ `null`

上の行から順に当てはめ、最初に当たった行の結果を返す。

| 順 | 条件 | 結果 |
| --- | --- | --- |
| 1 | `embedded` が `available` で、`token` がある | `identified`。`embedded` のトークンとワークスペースの ID |
| 2 | `declared` が `available` で、`token` がある | `identified`。`declared` のトークンとワークスペースの ID |
| 3 | `declared` が `found` で、`available` でない | `workspace_not_available` |
| 4 | `declared` が `available` で、`token` が無い | `unknown` |
| 5 | `embedded` が `found` で、`available` でない | `workspace_not_available` |
| 6 | 上のどれでもない | `unknown` |

- 利用終了の完了の後は、宣言したトークンの索引の項目が無いので、宣言したトークンの値は `found: false` で、行 6 の `unknown` に当たる。値がワークスペースの ID を含むトークンは行 5 に当たる

`verify` の処理。

1. `Token.standing` が `revoked` なら `revoked`、`expired` なら `expired` で拒否する
2. ブラウザ用トークンで、`OriginRestriction.permits(profile.origins, origin)` が偽なら `origin_not_allowed` で拒否する
3. それ以外は `verified`。`actor` は `Token.actor`、`workspaceId` は `input.workspaceId`、`usageLimit` は、ブラウザ用トークンの `profile.usageLimit`、サーバー用は `null`

- `token` と `workspaceId` は、`identify` の `identified` の結果
- 利用元は要求ごとに変わるので、`verify` だけが確かめる。`TokenAuthorization` は利用元を受け取らない

依存するポート: 無し。

### TokenAuthorization

責務: 実行者のトークンから、その時点の許可・範囲・権限の組を求め、操作の可否を判定する。規則は、アプリに紐付くトークンとメンバーに紐付くトークンのそれぞれに定める。

```ts
type TokenAuthorizationInput =
  | Readonly<{ workspaceAvailable: false }>
  | Readonly<{ workspaceAvailable: true; token: AppToken }>
  | Readonly<{ workspaceAvailable: true; token: MemberToken; member: MemberAccess }>;

type AccessRequirement = Readonly<{
  grant: TokenGrant;
  target: AccessTarget | null;
  permissions: readonly Permission[];
}>;

type AccessDecision =
  | Readonly<{ allowed: true }>
  | Readonly<{ allowed: false; reason: "grant_missing"; grant: TokenGrant }>
  | Readonly<{ allowed: false; reason: "out_of_scope"; target: AccessTarget }>
  | Readonly<{ allowed: false; reason: "permission_missing"; permissions: readonly Permission[] }>;

const TokenAuthorization: {
  resolve(input: TokenAuthorizationInput, now: Date): TokenAccess;
  authorize(access: GrantedTokenAccess, requirement: AccessRequirement): AccessDecision;
};
```

`resolve` の処理。

1. `workspaceAvailable` が偽なら、[index.md](index.md) の「利用できるワークスペース」の表の結果を返す
2. `Token.standing` が `revoked` なら `revoked`、`expired` なら `expired` で拒否する
3. 元になる権限の組を、トークンの種類ごとの規則で決める
   - アプリに紐付くトークン: `issuerPermissions`。発行者の `Member` を使わない
   - メンバーに紐付くトークン: `member` が `granted` であることを必要とする。`not_available` は `member_removed`、`mfa_setup_required` は `mfa_setup_required` で拒否する。`granted` なら `MemberGrant.permissions(member.grant)`
4. `permissions` を `GrantPolicy.effectivePermissions(grants, 元になる権限の組)`、`actor` を `Token.actor`、`designation` を `Token.designation`、`relatedAccountId` を `Token.relatedAccountId` にした `granted` を返す

- `token` は、実行者（`appToken` / `memberToken`）の `tokenId` のトークン
- `member` は、紐付くメンバー（`token.member.memberId`）についての、Workspace の `MemberAuthorization.resolve` の結果。`MemberAccess` は Workspace の型（[workspace.md](workspace.md)）で、Token は定義し直さない
- 認可の時点と、失効・解除・権限の変更が現れる時点は、[index.md](index.md) の「認可」に従う

`authorize` の処理。

1. `access.grants` が `requirement.grant` を持たなければ `grant_missing`
2. `GrantPolicy.refusals([requirement.grant], access.permissions)` が空でなければ、その `requiresAnyOf` を `permission_missing` で返す
3. `requirement.target` が `null` でなく、`TokenScope.covers(access.scope, target)` が偽なら `out_of_scope`
4. `PermissionSet.missing(access.permissions, requirement.permissions)` が空でなければ、不足する権限を `permission_missing` で返す
5. それ以外は `allowed`

- 各ユースケースは、受け付ける実行者の種類を定める（[index.md](index.md) の「認可」）。`appToken` と `memberToken` を受け付けないユースケースは、トークンの実行者の要求に `resolve` を呼ばない
- ユースケースは、操作が必要とする許可、対象、権限を `AccessRequirement` で指定する。必要な権限は、管理画面の同じ操作が必要とする `Permission` と同じである。`allowed` でない結果は、足りない許可、範囲の外の対象、不足する権限を持つ
- `target` は、「TokenGrant」の「集合の範囲」の列と「TokenScope」の規則で決める。集合の範囲で絞らない許可の操作は `workspace` を渡す。モデルにもフォームにも属さない対象（メディア）への操作と、複数のモデルにまたがる読み取りは `null` を渡す
- 対象（モデル、フォーム、コンテンツ）が存在しないことの判定は、ユースケースが `authorize` より前に行う。ユースケースは、要求のキーを現行スキーマの `ModelId` / `FormId` に解決し、存在する対象だけを `AccessRequirement.target` に渡す。存在しない対象への応答は、各ドメインのユースケースが定める。紐付け待ちの要素のキーを指定した要求は、その対象がまだ無いので、存在しない対象への要求になる

依存するポート: 無し。

### PreviewExchangeProcess

責務: 引き換えの値の引き換えの結果を決め、引き換えられる場合に、プレビューを開いたメンバーに紐付くトークンを作る。

```ts
type PreviewRefusal = "expired" | "redeemed" | "member_unavailable";

type PreviewRedeemInput =
  | Readonly<{ workspaceAvailable: false }>
  | Readonly<{
      workspaceAvailable: true;
      exchange: PreviewExchange;
      member: MemberAccess;
      token: { id: string; valueDigest: TokenValueDigest };
    }>;

type PreviewRedeemResult =
  | Readonly<{
      outcome: "redeemable";
      exchange: RedeemedPreviewExchange;
      token: MemberToken & Active;
    }>
  | Readonly<{ outcome: "refused"; refusal: PreviewRefusal }>;

const PreviewExchangeProcess: {
  redeem(input: PreviewRedeemInput, now: Date): PreviewRedeemResult;
};
```

`redeem` の処理。どの場合もエラーにしない。

1. `workspaceAvailable` が偽なら、[index.md](index.md) の「利用できるワークスペース」の表の結果を返す
2. `PreviewExchange.standing` が `expired` なら `refusal` が `expired`、`redeemed` なら `refusal` が `redeemed` の `refused` を返す
3. `member` が `granted` でなければ、`refusal` が `member_unavailable` の `refused` を返す
4. `Token.issueForMember` を `origin: "preview"`、`exchange.member` で呼ぶ
5. `PreviewExchange.redeem` を、手順 4 のトークンの ID で呼び、`outcome` が `redeemable` の結果で両方を返す

確定の時点でワークスペースが `active` でなかった引き換え（[index.md](index.md) の「競合の扱い」の、値の引き換えの `WORKSPACE_NOT_ACTIVE`）の結果は、`workspaceAvailable` が偽の入力に `redeem` を適用した結果。

`refused` の変種ごとのエラーは「エラーコード」の表が定める。

- `refused` では、引き換えは変わらず、トークンは発行されない
- `exchange` は、引き換えの値の照合値に該当する引き換え。`member` は、`exchange.member.memberId` のメンバーについての、Workspace の `MemberAuthorization.resolve` の結果
- 引き換えは、引き換えの値だけで行う。表示アプリは、ほかのトークンを示さない

依存するポート: 無し。

### DeviceAuthorizationProcess

責務: スクリプトの認証の受け取りの結果を決め、受け取れる場合に、承認したメンバーに紐付くトークンを作る。

```ts
type DeviceMemberStanding =
  | Readonly<{ workspaceAvailable: false }>
  | Readonly<{ workspaceAvailable: true; member: MemberAccess }>;

type DeviceRedeemResult =
  | Readonly<{ outcome: "expired" }>
  | Readonly<{ outcome: "refused"; refusal: DeviceRefusal }>
  | Readonly<{
      outcome: "redeemable";
      authorization: RedeemedDeviceAuthorization;
      token: MemberToken & Active;
    }>;

type DeviceAuthorizationResult =
  | Readonly<{ status: "pending" }>
  | Readonly<{ status: "redeemed" }>
  | Readonly<{ status: "expired" }>
  | Readonly<{ status: "denied"; denial: DeviceDenial }>;

type DeviceApprovalEligibility =
  | Readonly<{ selectable: true }>
  | Readonly<{ selectable: false; reason: "mfa_setup_required" }>
  | Readonly<{
      selectable: false;
      reason: "insufficient_permissions";
      refusals: readonly GrantRefusal[];
    }>;

const DeviceAuthorizationProcess: {
  eligibility(
    grants: ServerGrantSet,
    member: Exclude<MemberAccess, { status: "not_available" }>,
  ): DeviceApprovalEligibility;
  refusal(grants: ServerGrantSet, standing: DeviceMemberStanding): DeviceRefusal | null;
  redeem(
    input: {
      authorization: ApprovedDeviceAuthorization;
      standing: DeviceMemberStanding;
      token: { id: string; valueDigest: TokenValueDigest };
    },
    now: Date,
  ): DeviceRedeemResult;
  result(
    authorization: Exclude<DeviceAuthorization, ApprovedDeviceAuthorization>,
    now: Date,
  ): DeviceAuthorizationResult;
};
```

`eligibility` は、承認の画面を開いたアカウントが所属するワークスペースの一つを、依頼の対象に選べるかを判定する。上の行から順に当てはめ、最初に当たった行の結果を返す。

| 順 | 条件 | 結果 |
| --- | --- | --- |
| 1 | `member` が `mfa_setup_required` | 選べない（`mfa_setup_required`） |
| 2 | `GrantPolicy.refusals(grants, MemberGrant.permissions(member.grant))` が空でない | 選べない（`insufficient_permissions`）。その結果を `refusals` に持つ |
| 3 | 上のどれでもない | 選べる |

- `member` は、そのアカウントの、そのワークスペースでの、Workspace の `MemberAuthorization.resolve` の結果。`not_available` のワークスペースは、承認の画面に示さないので、`eligibility` に渡さない
- 選べないワークスペースは、依頼を変えない。依頼は `pending` のまま残り、ほかのワークスペースを選べる

`refusal` は、承認したメンバーが、依頼の許可のトークンを、いま受け取れるかを判定する。上の行から順に当てはめ、最初に当たった行の結果を返す。

| 順 | 条件 | 結果 |
| --- | --- | --- |
| 1 | `workspaceAvailable` が偽 | [index.md](index.md) の「利用できるワークスペース」の表の結果 |
| 2 | `member` が `granted` でない | `member_unavailable` |
| 3 | `GrantPolicy.refusals(grants, MemberGrant.permissions(member.grant))` が空でない | `insufficient_permissions`。その結果を `refusals` に持つ |
| 4 | 上のどれでもない | `null`（受け取れる） |

- `standing` の `member` は、`approval.member.memberId` のメンバーについての、`approval.workspaceId` のワークスペースでの、Workspace の `MemberAuthorization.resolve` の結果

`redeem` の処理。どの場合もエラーにしない。

1. `DeviceAuthorization.standing` が `expired`（承認の後に期限を過ぎた依頼）なら、`outcome` が `expired` の結果を返す
2. `refusal(authorization.grants, standing)` が `null` でなければ、その `refusal` の `refused` を返す
3. `Token.issueForMember` を、`id` を依頼の `id` にし、`origin: "device"`、`approval.member`、`authorization.grants`、`approval.tokenExpiresAt` で呼ぶ
4. `DeviceAuthorization.redeem` を、手順 3 のトークンの ID で呼び、`outcome` が `redeemable` の結果で両方を返す

`result` は、`approved` でない依頼の結果を求める。

| `DeviceAuthorization.standing` | 結果 |
| --- | --- |
| `pending` | `pending`（承認待ち） |
| `redeemed` | `redeemed`（受け取り済み）。トークンの値を含まない。この依頼からは、もうトークンを得られない |
| `expired` | `expired`（期限切れ） |
| `denied` | `denied`（拒否）と `denial`。権限が足りない拒否は、不足する権限を持つ |

`approved` の依頼には `redeem` を、ほかの依頼には `result` を適用する。

- `refused` と `expired` は、依頼を変えない。次の受け取りも同じ判定を行う。`refused` の原因が解消した依頼は、期限内の次の受け取りで `redeemable` になる
- `redeemable` は、依頼を `redeemed` にして、トークンを発行する結果

依頼を `redeemed` にした後、トークンの発行を確定する前に、`refusal` で確かめ直す。確かめ直しが `null` でない、確定の時点でワークスペースが `active` でなかった（[index.md](index.md) の「競合の扱い」の、値の引き換えの `WORKSPACE_NOT_ACTIVE`）、または途中で止まった受け取りは、トークンを発行しない。その結果は、`redeemed` の依頼に `result` を適用した結果。

依存するポート: 無し。

### エラーコード

Token が返すエラーの理由を、すべて列挙する。共通のコード（`PERMISSION_MISSING`、`ACTOR_NOT_ALLOWED`、`MFA_SETUP_REQUIRED`、`STALE_VERSION`、`ALREADY_EXISTS`、`SECRET_DIGEST_TAKEN`、`SCHEMA_CHANGED`、`WORKSPACE_NOT_ACTIVE`、`InvalidPagination`、`INVALID_QUERY`、`SECRET_UNAVAILABLE`）は [common.md](common.md) が定める。

```ts
type TokenErrorCode =
  // ValidationError（項目ごとの理由）
  | "InvalidTokenName"
  | "InvalidTokenValue"
  | "UnknownTokenKind"
  | "UnknownGrant"
  | "GrantNotAllowedForKind"
  | "EmptyGrants"
  | "EmptyScope"
  | "InvalidScopeKey"
  | "InvalidOrigin"
  | "InvalidUsageLimit"
  | "ExpiryNotSpecified"
  | "ExpiryInPast"
  | "InvalidPreviewBaseUrl"
  // BusinessRuleError
  | "RestrictionNotAllowedForKind"
  | "ScopeNotAllowedForGrant"
  | "GrantExceedsIssuerPermissions"
  | "RestrictionNotChangeable"
  | "ScopeTargetNotFound"
  | "TokenValueNotStored"
  | "TokenMemberUnavailable"
  | "PreviewExchangeExpired"
  | "PreviewExchangeRedeemed"
  | "DeviceAuthorizationExpired"
  | "DeviceAuthorizationNotPending"
  | "InvalidUserCode"
  // UnauthenticatedError
  | "TOKEN_UNKNOWN"
  | "TOKEN_REVOKED"
  | "TOKEN_EXPIRED"
  | "TOKEN_MEMBER_REMOVED"
  // ForbiddenError
  | "ORIGIN_NOT_ALLOWED"
  | "GRANT_MISSING"
  | "OUT_OF_SCOPE"
  | "SELF_REVOCATION_NOT_ALLOWED"
  // RateLimitedError
  | "TOKEN_USAGE_LIMITED"
  // DeclarationIssue の理由
  | "TokenValueDuplicated";
```

拒否の変種（`TokenAuthentication.identify`、`TokenVerification`、`TokenAccess`、`AccessDecision`、`PreviewRefusal`）から返すエラーは、下の表の「返す場合」の列だけが定める。ユースケースは、結果の変種をこの表のエラーにする。`DeviceRefusal` と `DeviceAuthorizationProcess.result` の結果はエラーにせず、ユースケースの出力で返す。

| 種類 | 理由 | 返す場合 |
| --- | --- | --- |
| `UnauthenticatedError` | `TOKEN_UNKNOWN` | `TokenAuthentication.identify` の `unknown` |
| `NotFoundError` | 無し（対象の種類は `workspace`） | `TokenAuthentication.identify` の `workspace_not_available`。`TokenAccess` の `rejected`: `workspace_not_available` |
| `UnauthenticatedError` | `TOKEN_REVOKED` | `TokenVerification` の `rejected`: `revoked`。`TokenAccess` の `rejected`: `revoked` |
| `UnauthenticatedError` | `TOKEN_EXPIRED` | `TokenVerification` の `rejected`: `expired`。`TokenAccess` の `rejected`: `expired` |
| `ForbiddenError` | `ORIGIN_NOT_ALLOWED` | `TokenVerification` の `rejected`: `origin_not_allowed` |
| `UnauthenticatedError` | `TOKEN_MEMBER_REMOVED` | `TokenAccess` の `rejected`: `member_removed` |
| `ForbiddenError` | `MFA_SETUP_REQUIRED` | `TokenAccess` の `rejected`: `mfa_setup_required` |
| `ForbiddenError` | `GRANT_MISSING` | `AccessDecision` の `grant_missing`。必要な許可を添える |
| `ForbiddenError` | `OUT_OF_SCOPE` | `AccessDecision` の `out_of_scope`。対象を添える |
| `ForbiddenError` | `PERMISSION_MISSING` | `AccessDecision` の `permission_missing`。不足する権限を添える |
| `BusinessRuleError` | `PreviewExchangeExpired` | `PreviewRefusal` の `expired` |
| `BusinessRuleError` | `PreviewExchangeRedeemed` | `PreviewRefusal` の `redeemed` |
| `BusinessRuleError` | `TokenMemberUnavailable` | `PreviewRefusal` の `member_unavailable` |
| `RateLimitedError` | `TOKEN_USAGE_LIMITED` | 示されたトークンを確かめる要求で、`TokenUsagePolicy.key` のキーの `UsageLimiter.check` が `limited`。再び試せる時刻（`retryAt`）を持つ |
| `BusinessRuleError` | `RestrictionNotChangeable` | `Token.changeRestrictions` が投げる |
| `ForbiddenError` | `SELF_REVOCATION_NOT_ALLOWED` | 「自分に紐付くトークンの失効」が返す |
| `DeclarationIssue.code` | `TokenValueDuplicated` | 初期構成のトークンの値が、宣言の中で重なる。エラーとして投げない |

- `ModelKey` / `FormKey` にならない範囲のキーは `InvalidScopeKey`
- Token が返す `NotFoundError` は、理由のコードを持たない。対象の種類は [common.md](common.md) の `NotFoundTarget` の値（`token`、`preview_exchange`、`device_authorization`、`workspace`）

## ポート

ポートの置き場所は [index.md](index.md) の「集約とポートの置き場所」に従う。

どのリポジトリも、[common.md](common.md) の「TransactionalRepository」「SingletonRepository」の共通の規則に従う。照合値の一意性（`SECRET_DIGEST_TAKEN`）と `deleteLapsed` は、[common.md](common.md) の「期限つきの一度きりの値」に従う。下の表は、各ポートが加えるメソッドの分類と、共通の規則に加える契約だけを書く。ページングは共通の `Pagination` と `PaginationResult` を使う。

値の生成と照合値の計算は `SecretTokenIssuer`、ブラウザ用トークンの値の保護は `SecretCipher`、利用量の制限は `UsageLimiter` を、ユースケースが使う。定義は [common.md](common.md) にある。

### TokenRepository

目的: 区画のトークンを保存し、照合値で引き、一覧を返す。

```ts
interface TokenRepository extends Omit<TransactionalRepository<Token, TokenId>, "delete"> {
  // 読み取り
  findByValueDigest(
    digest: TokenValueDigest,
  ): Promise<(Versioned<Token> & Readonly<{ lastUsedAt: Date | null }>) | null>;
  findPage(
    filter: TokenListFilter,
    now: Date,
    pagination: Pagination,
  ): Promise<PaginationResult<ListedToken>>;
  findSelectionScoped(): Promise<readonly Versioned<AppToken>[]>;
  // 書き込み
  insertMany(tokens: readonly Token[]): Promise<void>;
  saveMany(items: readonly Versioned<Token>[]): Promise<void>;
  deleteLapsed(before: Date, limit: number): Promise<number>;
  deleteAll(limit: number): Promise<number>;
  // UnitOfWork の外の記録
  recordUse(id: TokenId, usedAt: Date): Promise<void>;
}
```

| メソッド | 分類 | 契約 |
| --- | --- | --- |
| `insert` | 書き込み | 照合値の一意性は、共通の規則に従う |
| `findById` | 読み取り | 状態によらず返す |
| `save` | 書き込み | 同じトークンへの並行した失効と範囲の更新は、一方だけが成功する |
| `findByValueDigest` | 読み取り | 区画のトークンを、照合値の一致で引き、最終利用日時（`recordUse`）を添えて返す。状態と期限によらず返す。無ければ `null`。呼び出し側は、トークンの値から求めたワークスペースの区画のリポジトリで呼ぶ |
| `findPage` | 読み取り | 区画のトークンのうち、`TokenListFilter.matches(token, filter, { now, lastUsedAt, relatedMemberPresent })` が真になるトークンを、発行日時の降順、同じ日時は ID の降順で返す。`count` は、真になるトークンの総件数。`lastUsedAt` は最終利用日時（`recordUse`）、`relatedMemberPresent` は「TokenListFilter」の値で、判定に使った値を、返すトークンごとに `ListedToken` に添える |
| `findSelectionScoped` | 読み取り | 区画の、範囲が `selection` で `revoked` でないトークンをすべて返す。範囲が空のトークンと期限を過ぎたトークンを含む。順序は保証しない |
| `insertMany` | 書き込み | 複数のトークンを、一回の呼び出しで保存する。一意性の契約は `insert` と同じで、渡した列の中の ID と照合値の重複も違反になる。一件でも違反があれば、どのトークンも保存しない。件数の上限を置かない。空の列では何もしない。初期構成で宣言したトークンの作成が使う |
| `saveMany` | 書き込み | 複数のトークンを、一回の呼び出しで保存する。`items` の各要素は、保存するトークン（`entity`）と、そのトークンを読んだときの `expectedVersion`。契約は、要素ごとの `save` と同じ。一件でも `expectedVersion` が保存済みの `version` と違えば、全体が `ConflictError`（理由 `STALE_VERSION`）で、どのトークンも保存しない。存在しないトークンを含む場合も同じ。件数の上限を置かない。区画の、範囲を集合で持つトークンの総数を、一回で保存できる。空の列では何もしない。スキーマの反映の確定が、範囲が変わったトークンの保存に使う |
| `deleteLapsed` | 書き込み | [common.md](common.md) の「期限つきの一度きりの値」の削除。保持期間の起点は `Token.lapsedAt` |
| `deleteAll` | 書き込み | 区画の消去。対象は、区画のすべてのトークン。状態によらない。楽観ロックを使わない |
| `recordUse` | UnitOfWork の外の記録 | トークンの最終利用日時を記録する。記録は、記録済みの日時と `usedAt` の新しい方にする。記録の無いトークンの最終利用日時は `null`。最終利用日時は `Token` の外の値で、変えるのは `recordUse` だけ。トークンの `version` を変えない。並行した呼び出しの結果は、渡された日時のうち最も新しいものになる。存在しないトークンには何もしない。トークンを削除すると、その最終利用日時も無くなる |

### PreviewExchangeRepository

目的: 区画のプレビューの引き換えを保存し、引き換えの値の照合値で引く。

```ts
interface PreviewExchangeRepository
  extends Omit<TransactionalRepository<PreviewExchange, PreviewExchangeId>, "delete"> {
  // 読み取り
  findByCodeDigest(digest: ExchangeCodeDigest): Promise<Versioned<PreviewExchange> | null>;
  // 書き込み
  deleteLapsed(before: Date, limit: number): Promise<number>;
  deleteAll(limit: number): Promise<number>;
}
```

| メソッド | 分類 | 契約 |
| --- | --- | --- |
| `insert` | 書き込み | 照合値の一意性は、共通の規則に従う |
| `findById` | 読み取り | 共通の規則に従う |
| `save` | 書き込み | 同じ引き換えへの並行した `redeem` は、一方だけが成功する。成功しなかった側の UnitOfWork は、トークンの `insert` も反映されない |
| `findByCodeDigest` | 読み取り | 区画の引き換えを、照合値の一致で引く。`redeemed` と期限を過ぎた引き換えも返す。無ければ `null`。呼び出し側は、引き換えの値から `SecretTokenIssuer.workspaceOf` で求めたワークスペースの区画のリポジトリで呼ぶ |
| `deleteLapsed` | 書き込み | [common.md](common.md) の「期限つきの一度きりの値」の削除。保持期間の起点は `PreviewExchange.lapsedAt`。発行済みのトークンは削除しない |
| `deleteAll` | 書き込み | 区画の消去。対象は、区画のすべての引き換え。楽観ロックを使わない |

### DeviceAuthorizationRepository

目的: スクリプトの認証の依頼を保存し、承認の値と受け取りの値の照合値で引く。

```ts
interface DeviceAuthorizationRepository
  extends Omit<TransactionalRepository<DeviceAuthorization, DeviceAuthorizationId>, "delete"> {
  // 読み取り
  findByApprovalDigest(
    digest: ApprovalCodeDigest,
  ): Promise<Versioned<DeviceAuthorization> | null>;
  findByDeviceCodeDigest(
    digest: DeviceCodeDigest,
  ): Promise<Versioned<DeviceAuthorization> | null>;
  // 書き込み
  deleteLapsed(before: Date, limit: number): Promise<number>;
}
```

| メソッド | 分類 | 契約 |
| --- | --- | --- |
| `insert` | 書き込み | 照合値の種類は、承認の値と受け取りの値の二つ |
| `findById` | 読み取り | 共通の規則に従う |
| `save` | 書き込み | 同じ依頼への並行した承認と、承認しない操作、並行した `redeem` は、一方だけが成功する。成功しなかった側は `ConflictError`（理由 `STALE_VERSION`）。`approval.workspaceId` のワークスペースと、`tokenId` のトークンの存在を、ポートは確かめない。別の区画にあり、呼び出し側が手順の順序で扱う |
| `findByApprovalDigest` | 読み取り | 照合値の一致で引く。状態と期限によらず返す。無ければ `null` |
| `findByDeviceCodeDigest` | 読み取り | 照合値の一致で引く。状態と期限によらず返す。無ければ `null` |
| `deleteLapsed` | 書き込み | [common.md](common.md) の「期限つきの一度きりの値」の削除。保持期間の起点は `DeviceAuthorization.lapsedAt` |

### PreviewSettingsRepository

目的: ワークスペースのプレビューの設定を保存する。区画に一つの集約のポートで、[common.md](common.md) の「SingletonRepository」を継ぐ。

```ts
interface PreviewSettingsRepository extends SingletonRepository<PreviewSettings> {
  // 書き込み
  deleteAll(limit: number): Promise<number>;
}
```

| メソッド | 分類 | 契約 |
| --- | --- | --- |
| `insert` | 書き込み | 共通の規則に従う。区画に `Workspace` を作る UnitOfWork で呼ぶ |
| `find` | 読み取り | 共通の規則に従う |
| `save` | 書き込み | 共通の規則に従う |
| `deleteAll` | 書き込み | 区画の消去。対象は、区画の設定（0 または 1 件）。楽観ロックを使わない |

### DeclaredTokenIndex

目的: 初期構成で宣言したトークンの照合値から、発行元のワークスペースを引く。宣言した値はワークスペースの ID を含まないので、値を受けた入口が、この索引で区画を求める。`InstallationContext` と `PartitionReader.installation()` が `declaredTokenIndex` として公開する。集約を保存しないので、`TransactionalRepository` を継がず、楽観ロックを使わない。置き場所は `packages/core/src/domain/token/ports/`。

```ts
type DeclaredTokenEntry = Readonly<{
  valueDigest: TokenValueDigest;
  workspaceId: WorkspaceId;
}>;

interface DeclaredTokenIndex {
  // 読み取り
  findByDigest(digest: TokenValueDigest): Promise<WorkspaceId | null>;
  // 書き込み
  register(entries: readonly DeclaredTokenEntry[]): Promise<void>;
  removeByWorkspace(workspaceId: WorkspaceId): Promise<number>;
}
```

| メソッド | 分類 | 契約 |
| --- | --- | --- |
| `findByDigest` | 読み取り | 照合値の一致で引き、ワークスペースの ID を返す。無ければ `null`。その区画にトークンがあることを確かめない |
| `register` | 書き込み | 渡した項目を、一回の呼び出しで登録する。一件でも違反があれば、どの項目も登録しない。件数の上限を置かない。空の列では何もしない。一意性の契約は下のとおり |
| `removeByWorkspace` | 書き込み | そのワークスペースの項目を、一回の呼び出しですべて消し、消した件数を返す。該当が無ければ 0。消した照合値は、次の `register` に使える |

`register` の一意性。照合値が設置の全体で一意であること（一つの照合値が指すワークスペースは一つ）を、ポートが担保する。

| 場合 | 結果 |
| --- | --- |
| 索引に、同じ照合値でワークスペースの違う項目がある。または、渡した列の中に、同じ照合値でワークスペースの違う項目がある | `ConflictError`（理由 `SECRET_DIGEST_TAKEN`） |
| 索引に、同じ照合値で同じワークスペースの項目がある。または、渡した列の中に、同じ照合値で同じワークスペースの項目が重なる | 違反でない。項目を変えずに通過し、項目は一つのまま |
| 同じ照合値でワークスペースの違う項目の、並行した `register` | 一方だけが確定し、他方は `ConflictError`（理由 `SECRET_DIGEST_TAKEN`） |
| 同じ照合値で同じワークスペースの項目の、並行した `register` | 両立する。どちらも確定し、項目は一つ |

- 書き込みは、設置の区画の UnitOfWork の中で呼び、その UnitOfWork のほかの書き込みと、全部成功か全部反映されないかを共にする。読み取りは、`PartitionReader.installation()` からも呼べる
- 一意性・整合性: ワークスペースと、その区画のトークンの存在を、ポートは確かめない。別の区画にあり、呼び出し側が「初期構成の宣言」「利用終了の消去」の順序で担保する
- 可視性: 確定した書き込みは、以後のすべての読み取りに反映する
- 索引は、照合値とワークスペースの ID だけを持つ。トークンの値、許可、状態を持たない。失効・期限切れ・削除したトークンの項目は、利用終了まで残り、その値は、区画の読み取りの結果（失効、期限切れ、該当なし）で扱う

## トランザクション境界

[common.md](common.md) の「WriteScope」「UnitOfWorkProvider」に従う。`WorkspaceContext` が `tokenRepository`、`previewExchangeRepository`、`previewSettingsRepository` を、`InstallationContext` が `deviceAuthorizationRepository` と `declaredTokenIndex` を公開する。

- プレビューの引き換えは、`PreviewExchange` と `Token` を一つの UnitOfWork で確定する。一度だけ使えることは、引き換えの `save` の楽観ロックと、同じ UnitOfWork のトークンの `insert` で成り立つ
- スクリプトの認証の受け取りの段の順は、[index.md](index.md) の「区画をまたぐ手順」が定める。一度だけ使えることは、依頼の `save` の楽観ロックで成り立つ
- UnitOfWork ごとの `WriteScope` と、UnitOfWork の中で読むものは、各ユースケースのトランザクション境界だけが定める（[../usecases/token.md](../usecases/token.md)）。Installation のユースケースが呼ぶ生成と消去の `WriteScope` は、「初期構成の宣言」「利用終了の消去」が定める

## 初期構成の宣言

初期構成の読み込みとワークスペースの作成で、Token が受ける宣言と、その検証・生成の手順。宣言の形の全体と、`path` に渡す位置は、[installation.md](installation.md) の「初期構成の読み込み」が定める。各ドメインを呼ぶ順は、[index.md](index.md) の「手順を呼ぶ順」の表が定める。`DeclarationIssue` と検証の規則は [common.md](common.md) の「DeclarationIssue」が定める。生成は、Installation のユースケースが、設置の区画の段（索引の登録）と、ワークスペースごとの区画の段（`Workspace` を作る UnitOfWork。`workspace` の `none`）に分けて確定する（[index.md](index.md) の「区画をまたぐ手順」）。

```ts
type TokenDeclaration = AppTokenInput & Readonly<{ value: string }>;

const TokenDeclaration: {
  validate(
    entries: readonly Readonly<{ path: string; declaration: TokenDeclaration }>[],
    now: Date,
  ): readonly DeclarationIssue[];
};

const PreviewUrlDeclaration: {
  validate(entries: readonly Readonly<{ path: string; declaration: string }>[]): readonly DeclarationIssue[];
};
```

- `TokenDeclaration` は、`AppTokenInput`（`name`、`kind`、`grants`、`origins`、`usageLimit`、`scope`、`expiry`）に、宣言した値を加えたもの。`scope` は、ワークスペース全体（`workspace`）、またはモデル・フォームのキーの集合（`selection`）。`expiry` の `never` は期限なしの明示、`null` は、有効期限も期限なしの明示も無い宣言。`TokenDeclaration.validate` の `entries` は、宣言のすべてのワークスペースのトークン
- `PreviewUrlDeclaration` の宣言は、プレビュー先の表示アプリの URL。宣言しないワークスペースは、`entries` に含めない

検証。

| 関数 | 規則 | `path` | `code` |
| --- | --- | --- | --- |
| `TokenDeclaration.validate` | `Token.assertIssuable(input, PermissionSet.all(), now)`。`input` は、宣言から `value` を除いた値。発行者は初期構成の所有者で、すべての権限を持つ | 渡された `path` に、違反した項目の名前を続ける | 投げた `ValidationError` の理由、または `BusinessRuleError` のコード |
| `TokenDeclaration.validate` | `TokenKind.create(kind)` の種類で `TokenValue.create(kind, value)` | 渡された `path` に `.value` を続ける | 投げた `ValidationError` の理由 |
| `TokenDeclaration.validate` | 値が、`entries` の全体で重ならない | 重なる二件目以降の `path` に `.value` を続ける | `TokenValueDuplicated` |
| `PreviewUrlDeclaration.validate` | `PreviewBaseUrl.create` | 渡された `path` | 投げた `ValidationError` の理由 |

- `TokenValueDuplicated` は、この検証だけが返す理由。エラーとして投げない
- モデル・フォームのキーが指す対象の存在は確かめない。読み込みの時点で、ワークスペースのスキーマは空である
- `now` は、有効期限が過去でないことの判定に使う

生成。Installation のユースケースが、[index.md](index.md) の「手順を呼ぶ順」と「区画をまたぐ手順」の段で呼ぶ。Token が定めるのは、行うことと、その前提。

| 行うこと | UnitOfWork | 前提 |
| --- | --- | --- |
| トークンごとに、`SecretTokenIssuer.digest` で宣言した値の照合値を求める。ブラウザ用のトークンは、`SecretCipher.protect` の結果を `ProtectedTokenValue.wrap` で包む | 外 | 宣言の検証を終えている |
| 宣言のすべてのワークスペースのトークンの、照合値とワークスペースの ID の組を、`DeclaredTokenIndex.register` の一回の呼び出しで登録する | 内。設置の区画。初期構成の読み込みの最初の段と同じ UnitOfWork | 照合値を求めている。ワークスペースの ID は、Installation のユースケースが決めて渡す |
| ワークスペースごとに、`PreviewSettings.create(now)` の結果を `PreviewSettingsRepository.insert` で保存する。URL を宣言したワークスペースは、`create` に続けて `PreviewSettings.setUrl` を適用した結果を保存する。運用コマンドによるワークスペースの作成は、`create` の結果を保存する | 内。ワークスペースの区画。`Workspace` を作る UnitOfWork | 無い |
| ワークスペースごとに、トークンごとに、`IdGenerator.generate` の ID で `Token.issueForApp`（`origin` は `bootstrap`、`issuer` は Workspace の生成が返した所有者のメンバー、`issuerPermissions` は `PermissionSet.all()`）を呼び、そのワークスペースのトークンの全部を `TokenRepository.insertMany` の一回の呼び出しで保存する。`selection` の範囲の要素は「紐付け待ち（キー）」になる | 内。ワークスペースの区画。`PreviewSettings` の保存と同じ UnitOfWork | 索引の登録が確定している。Workspace の生成が、同じ UnitOfWork で所有者のメンバーを作っている |

- 宣言した値は保存しない。保存するのは照合値と、ブラウザ用トークンの保護した値
- 手順の前提（先に終えているべきドメイン）: Workspace。トークンの `issuer` に、Workspace の生成が作る所有者のメンバーを使う
- 索引の登録を、トークンの作成より先に確定する。値を示した要求が、区画より先に索引を引けるようにするため
- 途中で止まったとき: 索引に項目があって、区画にトークンが無い状態が残る。次の起動が、準備中のワークスペースから続け、区画の生成を確定する。索引の登録のやり直しは、同じ組の項目を変えずに通過する
- 宣言の中の値の重複は、検証が `TokenValueDuplicated` で返す。索引の一意性は、並行した登録にも成り立つ

宣言の照合値（[installation.md](installation.md)）。Token の宣言に、照合値から除く値は無い。トークンの値（`value`）とプレビュー先の表示アプリの URL を含め、Token の宣言のすべての項目を、宣言の照合値に含める。トークンの値を変えた宣言は、別の宣言になる。

## 利用終了の消去

ワークスペースの利用終了で、Installation のユースケースが呼ぶ、Token の消去。呼ぶ順は [index.md](index.md) の「手順を呼ぶ順」が持つ。

| 呼ぶメソッド | UnitOfWork | 前提 |
| --- | --- | --- |
| `PreviewExchangeRepository.deleteAll(BoundedDeletion.maxLimit)` | 内。ワークスペースの区画（`workspace` の `none`） | 無い |
| `TokenRepository.deleteAll(BoundedDeletion.maxLimit)` | 同じ形 | 無い |
| `PreviewSettingsRepository.deleteAll(BoundedDeletion.maxLimit)` | 同じ形 | 無い |
| `DeclaredTokenIndex.removeByWorkspace(workspaceId)` | 内。設置の区画（`installation`）。「利用終了の完了」の最後の段と同じ UnitOfWork。一回の呼び出しで、やり直しは 0 を返して通過する | 区画のトークンの消去を終えている |

- 手順の前提（先に消去を終えているべきドメイン）: 無い
- 区画のトークンの消去の後、索引の除外までの間は、索引に項目があって、区画にトークンが無い
- スクリプトの認証の依頼は、設置の区画にあり、消去しない。そのワークスペースを指す依頼のうち、終端でない依頼（`approved`）は、期限で終わる。終端の依頼（`redeemed`）は、変わらない。どちらも、定期ジョブ「期限切れの記録の掃除」が削除する

終了前の確認に公開する項目。型は [common.md](common.md) の「各ドメインが公開する項目」の `DomainKind`。

| 区分 | `kind` | 内容 |
| --- | --- | --- |
| 利用終了で失われるデータの種類 | `token` | トークン（アプリに紐付くトークンと、メンバーに紐付くトークン） |
| 利用終了で失われるデータの種類 | `preview_exchange` | プレビューの引き換え |
| 利用終了で失われるデータの種類 | `preview_settings` | プレビュー先の表示アプリの URL |
| 消去する資格情報 | `token_value` | トークンの照合値と、ブラウザ用トークンの保護した値。消去の後、その値を示した要求は拒否される。アプリに渡した値は、CMS から回収しない |

- CMS の通常データの外に残る情報は、Token には無い
- ワークスペースごとに違う件数を、Token は公開しない

## 状態確認の項目

無い。Token は、設置の状態確認に示す項目（`kind`、件数、必要な対処）を持たない。

## ユースケース（概要）

- トークンを手動発行する
- トークンの範囲に指定できる対象を一覧する
- トークンを一覧する
- 紐付けを待つトークンの範囲を一覧する
- ブラウザ用トークンの値を確かめる
- ブラウザ用トークンの利用元の限定と利用量の制限を変える
- トークンを失効させる
- 自分に紐付くトークンを失効させる
- 示されたトークンを確かめる
- トークンの要求を操作ごとに認可する
- スクリプトの認証を始める
- スクリプトの認証の内容を確かめる
- スクリプトの認証を承認する
- スクリプトの認証を承認しない
- スクリプトの認証の結果を受け取る
- プレビューを開く
- 引き換えの値をトークンに引き換える
- プレビュー先の表示アプリの URL を確かめる
- プレビュー先の表示アプリの URL を設定する
- プレビュー先の表示アプリの URL を取り除く
- 期限切れのプレビューの引き換え・トークンを削除する
- 期限切れのスクリプトの認証の依頼を削除する

`PreviewSettings` の作成と、初期構成で宣言したトークンとプレビュー先の表示アプリの URL の作成は、Installation のユースケースが、このドメインの振る舞いとポートを使って行う。範囲の紐付けと除外は、Schema のスキーマの反映の確定が行う。
