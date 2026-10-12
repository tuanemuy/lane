# Identity

人が本人であることを確かめ、セッションを保つ。境界と依存方向は [index.md](index.md) の「依存方向」が、集約とポートの区画は「集約とポートの置き場所」が定める。Identity の型とポートは、ワークスペースへの所属を持たない（「Account」の不変条件）。

人の認証は、メールアドレスとパスワードで行う。多要素認証は、認証アプリが表示する時刻基準の確認コード（TOTP、RFC 6238）で行う。Identity はドメインイベントを発行しない。エンティティの振る舞いは、次の状態のエンティティだけを返す。

## ユビキタス言語

| 英語名 | 日本語名 | 定義 |
| --- | --- | --- |
| Account | アカウント | 設置に一つの、人の登録。メールアドレスで識別する。資格情報と多要素認証の設定を持つ |
| Credential | 資格情報 | メールアドレスとパスワードの組 |
| Password hash | パスワードの照合値 | パスワードから一方向に求めた値。パスワードそのものは保存しない |
| Multi-factor authentication (MFA) | 多要素認証 | パスワードに加えて確認コードを求める設定。アカウントに設定し、所属するすべてのワークスペースへのログインに適用する |
| Confirmation code | 確認コード | 認証アプリが表示する 6 桁の数字 |
| Enrollment | 認証アプリへの登録 | 確認コードの元になる秘密を認証アプリに登録し、確認コードで確かめるまでの手続き |
| Session | セッション | 人がログインしてからログアウトまたは期限切れまでの、管理画面の利用の単位。アプリのトークンと別 |
| Session token | セッションの値 | ブラウザが保持し、要求ごとに示す値。CMS は照合値だけを保存する |
| Login attempt limit | ログインの試行の制限 | 同じメールアドレスへのログインの失敗が重なったときに、一定の時間、試行を受け付けないこと |
| Recovery request | 回復依頼 | 一度だけ使える回復の値の発行の記録 |
| Recovery token | 回復の値 | 回復用の URL に含める一度きりの値。CMS は照合値だけを保存する |
| Recovery URL | 回復用の URL | 設置の公開の URL と回復の値から作る URL。本人が開いて、新しいパスワードを設定する |
| Email recovery | メールによる回復 | 回復用の URL を、アカウントのメールアドレスへ設置全体のメール送信で届ける回復依頼 |
| Operator recovery | 管理者復旧 | ホスティング運営者が運用コマンドで発行し、回復用の URL を CMS の外で本人へ渡す回復依頼 |

## エンティティ

### Account

フィールド。

```ts
type MfaDisabled = Readonly<{ status: "disabled" }>;
type MfaEnrolling = Readonly<{
  status: "enrolling";
  secret: ProtectedTotpSecret;
}>;
type MfaEnabled = Readonly<{
  status: "enabled";
  secret: ProtectedTotpSecret;
  enabledAt: Date;
}>;
type MfaSetting = MfaDisabled | MfaEnrolling | MfaEnabled;

type AccountBase = Readonly<{
  id: AccountId;
  email: EmailAddress;
  name: AccountName;
  passwordHash: PasswordHash;
  version: number;
  createdAt: Date;
  updatedAt: Date;
}>;

type Account = AccountBase & Readonly<{ mfa: MfaSetting }>;
type MfaEnrollingAccount = AccountBase & Readonly<{ mfa: MfaEnrolling }>;
type MfaEnabledAccount = AccountBase & Readonly<{ mfa: MfaEnabled }>;
type MfaNotEnabledAccount = AccountBase & Readonly<{ mfa: MfaDisabled | MfaEnrolling }>;
```

| フィールド | 制約 |
| --- | --- |
| `id` | 必須。生成後に変わらない |
| `email` | 必須。設置の中で一意。生成後に変わらない |
| `name` | 必須 |
| `passwordHash` | 必須 |
| `mfa` | 必須。生成時は `disabled` |
| `version` | 生成時は 0。状態を変える振る舞いごとに 1 増える。現在と同じ値への変更は [index.md](index.md) の「変更の無い変更」 |

振る舞い。

```ts
const Account: {
  create(
    params: { id: string; email: string; name: string; passwordHash: PasswordHash },
    now: Date,
  ): Account;
  startMfaEnrollment(
    account: MfaNotEnabledAccount,
    secret: ProtectedTotpSecret,
    now: Date,
  ): MfaEnrollingAccount;
  enableMfa(account: MfaEnrollingAccount, now: Date): MfaEnabledAccount;
  disableMfa(account: MfaEnabledAccount, now: Date): Account;
  resetPassword(account: Account, passwordHash: PasswordHash, now: Date): Account;
  resetPasswordAndDisableMfa(
    account: MfaEnabledAccount,
    passwordHash: PasswordHash,
    now: Date,
  ): Account;
  isMfaEnabled(account: Account): account is MfaEnabledAccount;
  totpSecret(account: Account): ProtectedTotpSecret | null;
};
```

| 振る舞い | 処理 |
| --- | --- |
| `create` | `AccountId`、`EmailAddress`、`AccountName` を生成し、多要素認証が `disabled` のアカウントを返す |
| `startMfaEnrollment` | `mfa` を `enrolling` にし、渡された秘密を持たせる。`enrolling` のアカウントに呼ぶと、秘密を置き換える |
| `enableMfa` | `mfa` を `enabled` にする。秘密は `enrolling` のものを引き継ぐ |
| `disableMfa` | `mfa` を `disabled` にする |
| `resetPassword` | `passwordHash` を置き換える。`mfa` を変えない |
| `resetPasswordAndDisableMfa` | 多要素認証を設定済みのアカウントだけを受け取る。`passwordHash` を置き換え、`mfa` を `disabled` にする |
| `isMfaEnabled` | `mfa.status` が `enabled` なら真 |
| `totpSecret` | `mfa` が `enrolling` または `enabled` なら、その秘密。`disabled` なら `null` |

パスワードと確認コードの照合を前提とする振る舞い（`startMfaEnrollment`、`enableMfa`、`disableMfa`、`resetPassword`、`resetPasswordAndDisableMfa`）は、ドメインサービスが、照合の結果を確かめた後に呼ぶ。

不変条件。

- メールアドレスは設置の中で一意である
- 確認コードの元になる秘密は、`enrolling` と `enabled` のときだけ持つ
- ログインで確認コードを求めるのは、`enabled` のアカウントに限る
- 多要素認証の解除は、所属するワークスペースの設定に依らず、いつでも行える。多要素認証を必須にしたワークスペースでの、解除したアカウントのメンバーの認可は、Workspace の `MemberAuthorization.resolve` が定める
- アカウントは、ワークスペースへの所属と、最後に利用したワークスペースを持たない。アカウントの `version` は、資格情報と多要素認証の変更だけで進む

ライフサイクル。

- 生成: 招待の受諾でアカウントを持たない人が参加するとき、または初期構成の読み込みで所有者を宣言したとき
- 多要素認証: `disabled` →（`startMfaEnrollment`）→ `enrolling` →（`enableMfa`）→ `enabled` →（`disableMfa` / `resetPasswordAndDisableMfa`）→ `disabled`。`enrolling` は期限を持たず、`enrolling` から `disabled` へ戻る遷移は無い
- 存続: アカウントは、生成の後、所属の有無によらず残る。どのワークスペースにも所属しなくなったアカウントも、ログインでき、招待を受けて参加できる

### Session

フィールド。

```ts
type SessionBase = Readonly<{
  id: SessionId;
  accountId: AccountId;
  tokenDigest: SessionTokenDigest;
  version: number;
  createdAt: Date;
  expiresAt: Date;
}>;

type PendingMfaSession = SessionBase & Readonly<{ status: "pending_mfa" }>;
type ActiveSession = SessionBase & Readonly<{ status: "active"; authenticatedAt: Date }>;
type Session = PendingMfaSession | ActiveSession;

type SessionStanding = "active" | "pending_mfa" | "expired";
```

| フィールド | 制約 |
| --- | --- |
| `tokenDigest` | 必須。セッションの値の照合値 |
| `expiresAt` | `pending_mfa` は `createdAt` の `SessionPolicy.pendingMfaMs` 後。`active` は `authenticatedAt` の `SessionPolicy.activeMs` 後 |
| `version` | 生成時は 0。`completeMfa` で 1 増える |

セッションの値は、[common.md](common.md) の「期限つきの一度きりの値」に従う。

振る舞い。

```ts
const Session: {
  start(
    params: { id: string; accountId: AccountId; tokenDigest: SessionTokenDigest },
    now: Date,
  ): ActiveSession;
  startPendingMfa(
    params: { id: string; accountId: AccountId; tokenDigest: SessionTokenDigest },
    now: Date,
  ): PendingMfaSession;
  completeMfa(session: PendingMfaSession, now: Date): ActiveSession;
  standing(session: Session, now: Date): SessionStanding;
  lapsedAt(session: Session): Date;
};
```

| 振る舞い | 処理 |
| --- | --- |
| `start` | `ActiveSession` を返す。`authenticatedAt` は `now`。本人であることを確かめた後にだけ呼ぶ。呼ぶ場面は、下の「ライフサイクル」の生成 |
| `startPendingMfa` | `PendingMfaSession` を返す。パスワードの照合の後、確認コードの入力を待つセッション |
| `completeMfa` | `active` にし、`authenticatedAt` を `now`、`expiresAt` を `SessionPolicy.activeMs` 後にする。期限を判定しない。呼ぶのは `Authentication.completeMfa` だけ |
| `standing` | 期限切れ（[common.md](common.md) の「期限つきの一度きりの値」の期限の境界）なら `expired`。それ以外は `status` を返す |
| `lapsedAt` | 保持期間の起点。値は [common.md](common.md) の「期限つきの一度きりの値」に従う |

不変条件。

- 管理画面の操作を認めるのは、`standing` が `active` のセッションに限る。`pending_mfa` のセッションは、確認コードの入力だけに使える
- セッションは一つのアカウントに属し、アカウントを変えない。アカウントが無いセッションは保存されない（`SessionRepository.insert`）
- `active` のセッションの有効期限は、利用で延びない。`expiresAt` を変えるのは `completeMfa` だけ。期限を過ぎたセッションでの操作は、再ログインを必要とする

ライフサイクル。

- 生成: パスワードの照合に成功したとき（`Authentication.loginWithPassword`）。招待の受諾でアカウントを作った直後と、アクセス回復の完了の直後（`SessionIssuance.start`。示した一度きりの値と、設定したパスワードが本人であることを示す）
- 遷移: `pending_mfa` →（`completeMfa`）→ `active`
- 終了: ログアウトでユースケースがリポジトリの `delete` を呼ぶ。セッションを終わらせる操作は、ログアウトだけ。アクセス回復と、多要素認証の設定・解除は、そのアカウントの既存のセッションを変えない。定期ジョブ「期限切れの記録の掃除」が、[common.md](common.md) の「期限つきの一度きりの値」の対象を削除する

### RecoveryRequest

フィールド。

```ts
type RecoveryChannel = "email" | "operator";

type RecoveryRequestBase = Readonly<{
  id: RecoveryRequestId;
  accountId: AccountId;
  channel: RecoveryChannel;
  tokenDigest: RecoveryTokenDigest;
  version: number;
  createdAt: Date;
  expiresAt: Date;
}>;

type PendingRecoveryRequest = RecoveryRequestBase & Readonly<{ status: "pending" }>;
type UsedRecoveryRequest = RecoveryRequestBase & Readonly<{ status: "used"; usedAt: Date }>;
type RevokedRecoveryRequest = RecoveryRequestBase &
  Readonly<{ status: "revoked"; revokedAt: Date }>;
type RecoveryRequest =
  | PendingRecoveryRequest
  | UsedRecoveryRequest
  | RevokedRecoveryRequest;

type RecoveryStanding = "usable" | "expired" | "used" | "revoked";
```

| フィールド | 制約 |
| --- | --- |
| `channel` | `email` はメールによる回復、`operator` は管理者復旧 |
| `tokenDigest` | 必須。回復の値の照合値 |
| `expiresAt` | `email` は `createdAt` の `RecoveryPolicy.emailMs` 後。`operator` は `createdAt` の `RecoveryPolicy.operatorMs` 後 |
| `version` | 生成時は 0。`redeem`、`revoke` で 1 増える |

回復の値は、[common.md](common.md) の「期限つきの一度きりの値」に従う。

振る舞い。

```ts
const RecoveryRequest: {
  issue(
    params: {
      id: string;
      accountId: AccountId;
      channel: RecoveryChannel;
      tokenDigest: RecoveryTokenDigest;
    },
    now: Date,
  ): PendingRecoveryRequest;
  redeem(request: PendingRecoveryRequest, now: Date): UsedRecoveryRequest;
  revoke(request: PendingRecoveryRequest, now: Date): RevokedRecoveryRequest;
  standing(request: RecoveryRequest, now: Date): RecoveryStanding;
  lapsedAt(request: RecoveryRequest): Date;
};
```

| 振る舞い | 処理 |
| --- | --- |
| `issue` | `pending` の回復依頼を返す。有効期限は `channel` から決まる |
| `redeem` | `used` にする。期限を判定しない。`AccessRecovery.complete` が、`AccessRecovery.admit` が `standing` で `usable` に絞った回復依頼に呼ぶ |
| `revoke` | `revoked` にする。期限を過ぎた `pending` にも行える |
| `standing` | `pending` で期限内なら `usable`、期限切れなら `expired`（期限の境界は [common.md](common.md) の「期限つきの一度きりの値」）。それ以外は `status` を返す |
| `lapsedAt` | 保持期間の起点。値は [common.md](common.md) の「期限つきの一度きりの値」に従う |

不変条件。

- 回復に使えるのは、`standing` が `usable` の回復依頼に限る。一度使うと `used` になり、再び使えない
- 一つのアカウントが持つ `pending` の回復依頼は、経路（`channel`）ごとに最大一つである。新しい回復依頼の発行は、同じ経路の前の `pending` を `revoked` にする
- メールによる回復と管理者復旧は、互いを妨げず、互いを失効させない。ログインの前の人が始めるメールによる回復は、ホスティング運営者が発行した管理者復旧の回復依頼を変えない。一方の回復の完了は、他方の `pending` を変えず、他方は期限で終わる
- 回復依頼の発行は、アカウントの資格情報を変えない
- 回復依頼は一つのアカウントに属する。アカウントが無い回復依頼は保存されない（`RecoveryRequestRepository.insert`）

ライフサイクル。

- 生成: メンバーがアクセス回復を依頼したとき（`email`。設置全体のメール送信を使えない設置では発行しない）。ホスティング運営者が管理者復旧を実行したとき（`operator`）
- 遷移: `pending` →（`redeem`）→ `used`。`pending` →（`revoke`）→ `revoked`
- 削除: 定期ジョブ「期限切れの記録の掃除」が、[common.md](common.md) の「期限つきの一度きりの値」の対象を削除する

## 値オブジェクト

ブランド付きの型は、`create` だけが生成する。`TotpCode` だけは `parse` が生成する。利用者の入力から作る値オブジェクト（`AccountName`、`PlainPassword`）の違反は、`ValidationError`（項目ごとの理由。理由は表のコード）になる。複数の項目を受け取るユースケースは、項目ごとの違反を集めて一つの `ValidationError` で返す。利用者の入力に由来しない値（照合値、保護した秘密）の違反は `BusinessRuleError`（`InvalidStoredValue`）になる。集約の ID、`Versioned`、`TransactionalRepository`、`EmailAddress` は [common.md](common.md) の定義を使う。`ProtectedSecret` は [common.md](common.md) の「ProtectedSecret」、`UsageLimit` と `UsageStanding` は「UsageLimit と UsageStanding」が定めるドメインの型を使う。

| 名前 | 型 | バリデーションルール | 等価性 |
| --- | --- | --- | --- |
| `AccountName` | `string`（ブランド） | 前後の空白を除いた値を保持する。除いた値が 1 文字以上 100 文字以下。違反は `InvalidAccountName` | 値の一致 |
| `PlainPassword` | `string`（ブランド） | 12 文字以上 128 文字以下。文字の種類は問わない。違反は `PasswordPolicyViolation`。保存しない | 比較しない |
| `PasswordHash` | `string`（ブランド） | 空でない。`PasswordHasher` が生成した値。照合に必要な情報（方式、強度、ソルト）を含む | 値の一致 |
| `TotpCode` | `string`（ブランド） | 数字 6 桁。`TotpCode.parse(value: string): TotpCode \| null` が生成する。数字 6 桁でない入力には `null` を返し、エラーを投げない。数字 6 桁でない入力は、確認コードの誤りと同じ扱いになる | 値の一致 |
| `ProtectedTotpSecret` | `ProtectedSecret`（ブランド） | `ProtectedSecret` を包む。`TotpAuthenticator` が `SecretCipher` で保護した、確認コードの元になる秘密。保護していない秘密は保存しない | 値の一致 |
| `SessionTokenDigest` | `string`（ブランド） | 空でない。セッションの値の照合値 | 値の一致 |
| `RecoveryTokenDigest` | `string`（ブランド） | 空でない。回復の値の照合値 | 値の一致 |
| `TotpEnrollment` | `Readonly<{ secret: ProtectedTotpSecret; setupKey: string; otpauthUri: string }>` | `TotpAuthenticator.enroll` が生成する。`setupKey` と `otpauthUri` は認証アプリへの登録に示す情報で、保存しない | 比較しない |
| `LoginAttemptPolicy` | 定数と関数 | 下の定義 | — |
| `SessionPolicy` | 定数 | セッションの有効期間。`pendingMfaMs` は確認コードの入力を待つ間、`activeMs` はログインの後。値は [../scenario/app-integration.md](../scenario/app-integration.md) の APP-12 の表（「確認コードの入力を待つ間のセッションの有効期間」「ログインの後のセッションの有効期間」）が定め、技術設計の層では、この定数だけが持つ | — |
| `RecoveryPolicy` | 定数 | 回復依頼の有効期間。`emailMs` はメールによる回復、`operatorMs` は管理者復旧。値は [../scenario/app-integration.md](../scenario/app-integration.md) の APP-12 の表（「回復用の URL の有効期間」の二行）が定め、技術設計の層では、この定数だけが持つ | — |
| `RecoveryLink` | 定数と関数 | 下の定義 | — |

`PlainPassword`、確認コード、セッションの値、回復の値は、公開データ、通常のログ、エラー情報、操作履歴に含めない。

### LoginAttemptPolicy

```ts
const LoginAttemptPolicy: {
  limit: UsageLimit;
  key(email: EmailAddress): string;
  refusal(standing: UsageStanding): Date | null;
};
```

- 制限の単位はメールアドレス。失敗は、パスワードの照合の失敗と、ログインの確認コードの照合の失敗を指す。同じメールアドレスへの失敗を、アカウントの有無によらず数える
- 制限の対象は、ログインの試行だけ。アクセス回復の確認コードの照合（`AccessRecovery.admit`）と、ログイン済みの本人確認（`MfaEnrollment`）の照合の失敗は、数えず、制限しない。アクセス回復は、確認コードを使わずに多要素認証の解除を選べる
- `limit` の値（`max` と `windowMs`）は [../scenario/app-integration.md](../scenario/app-integration.md) の APP-12 の表（「ログインの試行の制限」）が定め、技術設計の層では、この定数だけが持つ
- 制限の期間と結果は [common.md](common.md) の「UsageLimiter」に従う
- ログインの成功と、アクセス回復の完了は、失敗の数を消さず、制限を解かない
- `refusal` は、`standing` が `limited` なら `standing.retryAt` を、`available` なら `null` を返す。ログインの試行の制限で止めるかどうかを、この関数だけが決める。セッションを始めるドメインサービス（`Authentication`、`SessionIssuance`）は、どの経路でも、`refusal` が日時を返したアカウントにセッションを始めない

### RecoveryLink

回復用の URL の形。URL の組み立ての分担は [index.md](index.md) の「URL の組み立て」が定める。

```ts
const RecoveryLink: {
  readonly path: "/recovery";
  url(publicUrl: string, token: string): string;
};
```

- `url` は、`path` と回復の値（`SecretTokenIssuer.issue` の `value`）から、[common.md](common.md) の「期限つきの一度きりの値」の URL の組み立てで URL を作る。`publicUrl` は、設置の公開の URL
- 経路の形は、この定数だけが持つ
- 保存を伴わない純粋な関数

## ドメインサービス

ドメインサービスは、ポートを呼ばない、保存を伴わない純粋な関数。パスワードの照合と照合値の計算（`PasswordHasher`）、秘密の生成と確認コードの照合（`TotpAuthenticator`）は、ユースケースがポートを呼び、ドメインサービスは、その結果（一致の真偽、`PasswordHash`、`TotpEnrollment`）を受け取る。照合の結果をどう扱うか（どの結果より先に何を確かめるか、どのエラーにするか）は、ドメインサービスが決める。

ユースケースが渡す照合の結果。

| 入力 | 求め方 |
| --- | --- |
| `passwordMatches` | `PasswordHasher.verify(password, hash)` の結果。`hash` は、アカウントの `passwordHash`。アカウントが無い場合は `null` を渡す（結果は偽） |
| `codeMatches` | `TotpCode.parse(code)` が `null`、または `Account.totpSecret(account)` が `null` なら、ポートを呼ばずに偽。それ以外は、`TotpAuthenticator.verify(secret, code, now)` の結果 |

ログインとアクセス回復（ログインの前）の照合の失敗は、例外ではなく戻り値の `outcome` で返す。ユースケースが、下の理由の `UnauthenticatedError` に対応させる。応答は、アカウントが無い場合とパスワードが誤っている場合を区別しない。ログイン済みの本人確認（`MfaEnrollment`）の照合の失敗は、`BusinessRuleError` を投げる。

`UnauthenticatedError` の理由のコード。

```ts
type UnauthenticatedReason = "CREDENTIALS_REJECTED" | "CODE_REJECTED" | "SESSION_INVALID";
```

| 理由 | 意味 | 対応する結果 |
| --- | --- | --- |
| `CREDENTIALS_REJECTED` | ログインの資格情報の誤り。アカウントが無い場合を含む | `Authentication.loginWithPassword` の `rejected` |
| `CODE_REJECTED` | 確認コードの誤り | `Authentication.completeMfa` の `rejected`、`AccessRecovery.admit` の `code_rejected` |
| `SESSION_INVALID` | その操作に使えるセッションが無い | 照合値に該当するセッションが無い（ログアウトと削除で失効したセッション、保持期間を過ぎたセッションを含む）、`Session.standing` が `expired`。管理画面の操作では、`Session.standing` が `active` でないセッションを、ログインを終えたセッションが無いものとして扱う。確認コードの入力では、`Session.standing` が `pending_mfa` でないセッションと、アカウントが多要素認証を設定済み（`enabled`）でないセッション（アクセス回復で解除した場合）を、確認コード待ちのセッションが無いものとして扱う。セッションを必要とする操作にセッションの値が示されない場合（宛先のアカウントがある招待の受諾を含む）も、この理由。ログアウトはこの理由を返さない。照合値で引けたセッションを `standing` によらず削除し、引けない値でも正常に終える |

`CREDENTIALS_REJECTED` と `CODE_REJECTED` は、入力し直しを求める。`SESSION_INVALID` は、再ログインを求める。

### Authentication

責務: 資格情報と確認コードの照合の結果から、セッションを開始する。

```ts
type PasswordLoginResult =
  | Readonly<{ outcome: "limited"; retryAt: Date }>
  | Readonly<{ outcome: "rejected" }>
  | Readonly<{ outcome: "mfa_required"; session: PendingMfaSession }>
  | Readonly<{ outcome: "authenticated"; session: ActiveSession }>;

type MfaLoginResult =
  | Readonly<{ outcome: "limited"; retryAt: Date }>
  | Readonly<{ outcome: "rejected" }>
  | Readonly<{ outcome: "authenticated"; session: ActiveSession }>;

const Authentication: {
  loginWithPassword(
    input: {
      account: Account | null;
      passwordMatches: boolean;
      standing: UsageStanding;
      session: { id: string; tokenDigest: SessionTokenDigest };
    },
    now: Date,
  ): PasswordLoginResult;
  completeMfa(
    input: {
      account: MfaEnabledAccount;
      session: PendingMfaSession;
      codeMatches: boolean;
      standing: UsageStanding;
    },
    now: Date,
  ): MfaLoginResult;
};
```

| メソッド | 処理 |
| --- | --- |
| `loginWithPassword` | `LoginAttemptPolicy.refusal(standing)` が日時を返せば、その日時の `limited`。照合の結果より先に確かめる。`account` が `null`、または `passwordMatches` が偽なら `rejected`。それ以外は、多要素認証が `enabled` なら `Session.startPendingMfa` を呼んで `mfa_required`、`enabled` でなければ `Session.start` を呼んで `authenticated` を返す |
| `completeMfa` | 確認コード待ちのセッションと、多要素認証を設定済みのアカウントだけを受け取る。`session.accountId` が `account.id` と違えば `BusinessRuleError`（`SessionAccountMismatch`）。`LoginAttemptPolicy.refusal(standing)` が日時を返せば、その日時の `limited`。`codeMatches` が偽なら `rejected`。真なら `Session.completeMfa` を呼び、`authenticated` を返す |


依存するポート: 無し。

### SessionIssuance

責務: ログインのほかの経路で、本人であることを確かめた直後に、セッションを始めるかどうかを決める。

```ts
type SessionIssuanceResult =
  | Readonly<{ outcome: "started"; session: ActiveSession }>
  | Readonly<{ outcome: "login_required"; retryAt: Date }>;

const SessionIssuance: {
  start(
    input: {
      accountId: AccountId;
      session: { id: string; tokenDigest: SessionTokenDigest };
      standing: UsageStanding;
    },
    now: Date,
  ): SessionIssuanceResult;
};
```

- `LoginAttemptPolicy.refusal(standing)` が日時を返せば、セッションを作らずに、その日時を `retryAt` にした `login_required` を返す。その人は、`retryAt` の後にログインする
- `null` なら、`Session.start` を呼んで `started` を返す
- 経路によらず同じ規則を適用する

依存するポート: 無し。

### MfaEnrollment

責務: ログイン済みの本人であることを照合の結果で確かめたうえで、多要素認証を設定し、解除する。

```ts
type MfaEnrollmentStart = Readonly<{
  account: MfaEnrollingAccount;
  enrollment: TotpEnrollment;
}>;

const MfaEnrollment: {
  start(
    input: { account: Account; passwordMatches: boolean; enrollment: TotpEnrollment },
    now: Date,
  ): MfaEnrollmentStart;
  confirm(input: { account: Account; codeMatches: boolean }, now: Date): MfaEnabledAccount;
  disable(
    input: { account: Account; passwordMatches: boolean; codeMatches: boolean },
    now: Date,
  ): Account;
};
```

| メソッド | 処理 |
| --- | --- |
| `start` | 多要素認証が `enabled` のアカウントは `BusinessRuleError`（`MfaAlreadyEnabled`）。照合の結果より先に確かめる。`passwordMatches` が偽なら `BusinessRuleError`（`PasswordVerificationFailed`）。真なら、`enrollment.secret` を渡して `Account.startMfaEnrollment` を呼ぶ。認証アプリへ登録する情報を `enrollment` で返す。`enrollment` は、ユースケースが `TotpAuthenticator.enroll` で作ったもの。エラーで終わるときは、どこにも残らない |
| `confirm` | 多要素認証が `enrolling` でないアカウントは `BusinessRuleError`（`MfaEnrollmentNotStarted`）。照合の結果より先に確かめる。`codeMatches`（`enrolling` の秘密との照合の結果）が偽なら `BusinessRuleError`（`CodeVerificationFailed`）。真なら `Account.enableMfa` を呼ぶ |
| `disable` | 多要素認証が `enabled` でないアカウントは、照合の結果を見ずに、渡されたアカウントをそのまま返す（変更の無い変更）。`enabled` のアカウントは、`passwordMatches`、`codeMatches` の順に確かめる。偽なら `BusinessRuleError`（`PasswordVerificationFailed` / `CodeVerificationFailed`）。両方が真なら `Account.disableMfa` を呼ぶ |

- ログイン済みの本人確認の失敗は、再ログインを求める `UnauthenticatedError` にしない。セッションは変わらず、利用者は入力し直す
- `BusinessRuleError` では、アカウントは変わらない

依存するポート: 無し。

### AccessRecovery

責務: 回復依頼を発行し、回復依頼と引き換えに資格情報を設定し直す。

```ts
type RecoveryMfaChoice =
  | Readonly<{ action: "verify"; codeMatches: boolean }>
  | Readonly<{ action: "remove" }>;

type RecoveryIssueResult = Readonly<{
  request: PendingRecoveryRequest;
  revoked: RevokedRecoveryRequest | null;
}>;

type RecoveryReset =
  | Readonly<{ kind: "password"; account: Account }>
  | Readonly<{ kind: "password_and_mfa"; account: MfaEnabledAccount }>;

type RecoveryAdmission =
  | Readonly<{ outcome: "code_rejected" }>
  | Readonly<{
      outcome: "admitted";
      request: PendingRecoveryRequest;
      password: PlainPassword;
      reset: RecoveryReset;
    }>;

type RecoveryCompletion = Readonly<{ account: Account; request: UsedRecoveryRequest }>;

const AccessRecovery: {
  issue(
    input: {
      account: Account;
      channel: RecoveryChannel;
      pending: PendingRecoveryRequest | null;
      id: string;
      tokenDigest: RecoveryTokenDigest;
    },
    now: Date,
  ): RecoveryIssueResult;
  admit(
    input: {
      request: RecoveryRequest;
      account: Account;
      newPassword: string;
      mfa: RecoveryMfaChoice | null;
    },
    now: Date,
  ): RecoveryAdmission;
  complete(
    input: {
      admission: Extract<RecoveryAdmission, { outcome: "admitted" }>;
      passwordHash: PasswordHash;
    },
    now: Date,
  ): RecoveryCompletion;
};
```

| メソッド | 処理 |
| --- | --- |
| `issue` | `pending`（そのアカウントの、`channel` が同じ `pending` の回復依頼）があれば `RecoveryRequest.revoke` を呼ぶ。別の経路の回復依頼は、受け取らず、変えない。`RecoveryRequest.issue` で新しい回復依頼を作り、両方を返す |
| `admit` | 回復を受け付けられるかを、下の表で決める。照合値の計算の前に呼ぶ |
| `complete` | `admit` が `admitted` を返した回復を確定する。`admission.reset` が `password` なら `Account.resetPassword` を、`password_and_mfa` なら `Account.resetPasswordAndDisableMfa` を、`reset.account` と `passwordHash` で呼ぶ。`admission.request` に `RecoveryRequest.redeem` を呼び、両方を返す。多要素認証を解除したかどうかは、`reset` の `kind` が表す。`passwordHash` は、ユースケースが `admission.password` を `PasswordHasher.hash` に渡して求めたもの |

`admit` の判定。上の行から順に確かめ、初めに当たった行の結果を返す。

| 条件 | 結果 |
| --- | --- |
| `RecoveryRequest.standing` が `usable` でない | `BusinessRuleError`（`expired` は `RecoveryRequestExpired`、`used` は `RecoveryRequestUsed`、`revoked` は `RecoveryRequestRevoked`） |
| `request.accountId` が `account.id` と違う | `BusinessRuleError`（`RecoveryAccountMismatch`） |
| `PlainPassword.create(newPassword)` が違反 | `ValidationError`（`PasswordPolicyViolation`） |
| 多要素認証が `enabled` でないアカウント | `admitted`。`mfa` を見ない。`reset` は `password` |
| `enabled` のアカウントで、`mfa` が `null` | `BusinessRuleError`（`RecoveryMfaChoiceRequired`） |
| `enabled` のアカウントで、`mfa` が `verify`、`codeMatches` が偽 | `code_rejected` |
| `enabled` のアカウントで、`mfa` が `verify`、`codeMatches` が真 | `admitted`。`reset` は `password` |
| `enabled` のアカウントで、`mfa` が `remove` | `admitted`。`reset` は `password_and_mfa` |

- `RecoveryMfaChoice` の `codeMatches` は、ユースケースが、利用者の入力した確認コードから、上の表の求め方で求めて渡す
- 多要素認証の解除は、`enabled` のアカウントの回復だけが行う。設定の途中（`enrolling`）のアカウントの回復は、`mfa` を変えない
- `BusinessRuleError`、`ValidationError`、`code_rejected` のどれでも、アカウントと回復依頼は変わらない

依存するポート: 無し。

### エラーコード

Identity が返す理由のコードを、すべて列挙する。共通のコード（`STALE_VERSION`、`ALREADY_EXISTS`、`SECRET_DIGEST_TAKEN`、`ACTOR_NOT_ALLOWED`、`INVALID_QUERY`、`SECRET_UNAVAILABLE` など）は [common.md](common.md) にある。

```ts
type IdentityErrorCode =
  // ValidationError
  | "InvalidAccountName"
  | "PasswordPolicyViolation"
  // BusinessRuleError
  | "InvalidStoredValue"
  | "MfaAlreadyEnabled"
  | "MfaEnrollmentNotStarted"
  | "PasswordVerificationFailed"
  | "CodeVerificationFailed"
  | "SessionAccountMismatch"
  | "RecoveryRequestExpired"
  | "RecoveryRequestUsed"
  | "RecoveryRequestRevoked"
  | "RecoveryAccountMismatch"
  | "RecoveryMfaChoiceRequired"
  // UnauthenticatedError
  | UnauthenticatedReason
  // RateLimitedError
  | "LOGIN_ATTEMPTS_LIMITED"
  // ConflictError
  | "EMAIL_TAKEN"
  | "PENDING_RECOVERY_EXISTS"
  | "ACCOUNT_MISSING"
  // DeclarationIssue の理由
  | "OwnerDeclarationConflict";
```

| 種類 | 理由 | 意味 |
| --- | --- | --- |
| `ValidationError` | `InvalidAccountName` / `PasswordPolicyViolation` | 利用者の入力から値オブジェクトを作れない。メールアドレスの違反は、共通の `InvalidEmailAddress` |
| `DeclarationIssue.code` | `OwnerDeclarationConflict` | 初期構成で、同じメールアドレスの所有者の名前またはパスワードが一致しない。エラーとして投げない |
| `BusinessRuleError` | `InvalidStoredValue` | 利用者の入力に由来しない値（照合値、保護した秘密）が空 |
| `BusinessRuleError` | `MfaAlreadyEnabled` / `MfaEnrollmentNotStarted` | 多要素認証の設定の状態に合わない操作 |
| `BusinessRuleError` | `PasswordVerificationFailed` / `CodeVerificationFailed` | ログイン済みの本人確認の失敗 |
| `BusinessRuleError` | `SessionAccountMismatch` | セッションとアカウントが合わない操作 |
| `BusinessRuleError` | `RecoveryRequestExpired` / `RecoveryRequestUsed` / `RecoveryRequestRevoked` / `RecoveryAccountMismatch` / `RecoveryMfaChoiceRequired` | 回復依頼の状態に合わない操作 |
| `UnauthenticatedError` | `CREDENTIALS_REJECTED` / `CODE_REJECTED` / `SESSION_INVALID` | 「ドメインサービス」の表のとおり。セッションの分だけを定め、トークンの分は Token が定める |
| `RateLimitedError` | `LOGIN_ATTEMPTS_LIMITED` | ログインの試行の制限中。再び試せる時刻を持つ。`Authentication` の二つのメソッドの `limited` |
| `ConflictError` | `EMAIL_TAKEN` | 同じメールアドレスのアカウントがある |
| `ConflictError` | `PENDING_RECOVERY_EXISTS` | 同じアカウントの、同じ経路（`channel`）の `pending` の回復依頼がある |
| `ConflictError` | `ACCOUNT_MISSING` | セッションまたは回復依頼のアカウントが、確定の時点で存在しない |

## ポート

`SessionRepository` は `TransactionalRepository` を継ぐ。`AccountRepository` と `RecoveryRequestRepository` は、一件ずつ削除する呼び手が無いので、`Omit<TransactionalRepository<TEntity, TId>, "delete">` を継ぐ。どのリポジトリも、[common.md](common.md) の「TransactionalRepository」の、すべてのリポジトリのメソッドに適用する規則に従う。照合値で引く集約のリポジトリは、加えて「期限つきの一度きりの値」に従う。下の表は、これに加える契約だけを書く。

どのリポジトリのメソッドも、ワークスペースの ID を引数に取らず、ワークスペースへの所属を読み書きしない。

共通のポート `SecretTokenIssuer`、`UsageLimiter`、`SystemMailer` は、[common.md](common.md) が定める。

### AccountRepository

目的: アカウントを保存し、ID とメールアドレスで引く。

```ts
interface AccountRepository
  extends Omit<TransactionalRepository<Account, AccountId>, "delete"> {
  insertMany(accounts: readonly Account[]): Promise<void>;
  findByEmail(email: EmailAddress): Promise<Account | null>;
  findByIds(ids: readonly AccountId[]): Promise<readonly Account[]>;
}
```

メソッドの分類: `findByEmail` と `findByIds` は読み取り。`insertMany` は書き込み。

| メソッド | 契約 |
| --- | --- |
| `insert` | メールアドレスの一意性をポートが担保する。同じメールアドレスのアカウントがあれば `ConflictError`（理由 `EMAIL_TAKEN`）。並行した `insert` でも一方だけが成功する |
| `findById` | 無ければ `null` |
| `save` | 楽観ロックを必要とする |
| `insertMany` | 複数のアカウントを、一回の呼び出しで保存する。一意性の契約は `insert` と同じで、渡した列の中の重複も違反になる。一件でも違反があれば、どのアカウントも保存しない。渡せる件数の上限は 1,000 件。空の列では何もしない。初期構成の読み込みが使う |
| `findByEmail` | `EmailAddress` の一致で引く。無ければ `null` |
| `findByIds` | 渡した ID のうち、存在するアカウントを、ID の昇順で返す。渡せる ID の上限は 100 件。空の集合には空の列を返す |

### SessionRepository

目的: セッションを保存し、セッションの値の照合値で引く。

```ts
interface SessionRepository extends TransactionalRepository<Session, SessionId> {
  findByTokenDigest(digest: SessionTokenDigest): Promise<Versioned<Session> | null>;
  deleteLapsed(before: Date, limit: number): Promise<number>;
}
```

メソッドの分類: `findByTokenDigest` は読み取り。`deleteLapsed` は書き込み。

| メソッド | 契約 |
| --- | --- |
| `insert` | セッションのアカウントが、UnitOfWork の確定の時点で存在することを、ポートが担保する。存在しなければ `ConflictError`（理由 `ACCOUNT_MISSING`）。同じ UnitOfWork で `insert` したアカウントのセッションは保存できる |
| `save` | 楽観ロックを必要とする。同じ `pending_mfa` のセッションへの並行した `completeMfa` は、一方だけが成功する |
| `delete` | 楽観ロックを必要とする。削除したセッションは、次の `findByTokenDigest` から返らない |
| `findByTokenDigest` | 照合値の一致で引く。期限を過ぎたセッションも返す。期限の判定は `Session.standing` が行う。無ければ `null` |
| `deleteLapsed` | 一度きりの値の削除（[common.md](common.md) の「期限つきの一度きりの値」）。`lapsedAt` は `Session.lapsedAt`。定期ジョブ「期限切れの記録の掃除」が使う |

### RecoveryRequestRepository

目的: 回復依頼を保存し、回復の値の照合値とアカウントで引く。

```ts
interface RecoveryRequestRepository
  extends Omit<TransactionalRepository<RecoveryRequest, RecoveryRequestId>, "delete"> {
  findByTokenDigest(
    digest: RecoveryTokenDigest,
  ): Promise<Versioned<RecoveryRequest> | null>;
  findPendingByAccount(
    accountId: AccountId,
    channel: RecoveryChannel,
  ): Promise<Versioned<PendingRecoveryRequest> | null>;
  deleteLapsed(before: Date, limit: number): Promise<number>;
}
```

メソッドの分類: `findByTokenDigest` と `findPendingByAccount` は読み取り。`deleteLapsed` は書き込み。

| メソッド | 契約 |
| --- | --- |
| `insert` | 一つのアカウントの `pending` が `channel` ごとに最大一つであることを、ポートが担保する。確定の時点で、同じアカウントの、同じ `channel` の `pending` が二つになる書き込みは `ConflictError`（理由 `PENDING_RECOVERY_EXISTS`）。同じアカウントの、`channel` が違う `pending` は、どちらの順の `insert` も成功する。期限を過ぎた `pending` も `pending` として数える。同じ UnitOfWork で、同じ `channel` の前の `pending` を `revoked` に `save` し、新しい `pending` を `insert` する書き込みは成功する。回復依頼のアカウントが、UnitOfWork の確定の時点で存在することを、ポートが担保する。存在しなければ `ConflictError`（理由 `ACCOUNT_MISSING`） |
| `save` | 楽観ロックを必要とする。同じ回復依頼への並行した `redeem` は、一方だけが成功する |
| `findByTokenDigest` | 照合値の一致で引く。`used`、`revoked`、期限を過ぎた回復依頼も返す。無ければ `null` |
| `findPendingByAccount` | そのアカウントの、その `channel` の `pending` の回復依頼を返す。期限を過ぎたものも返す。別の `channel` の回復依頼は返さない。無ければ `null` |
| `deleteLapsed` | 一度きりの値の削除（[common.md](common.md) の「期限つきの一度きりの値」）。`lapsedAt` は `RecoveryRequest.lapsedAt`。定期ジョブ「期限切れの記録の掃除」が使う |

### PasswordHasher

目的: パスワードを、保存できる照合値にし、照合する。置き場所は `packages/core/src/domain/identity/ports/`。

```ts
interface PasswordHasher {
  hash(password: PlainPassword): Promise<PasswordHash>;
  verify(password: string, hash: PasswordHash | null): Promise<boolean>;
}
```

| メソッド | 契約 |
| --- | --- |
| `hash` | パスワードごとに異なるソルトを使い、総当たりに計算資源を要する一方向の方式で照合値を求める。同じパスワードから、呼び出しごとに異なる照合値を返す。照合値からパスワードを求められない |
| `verify` | `hash` が求めた照合値とパスワードが対応すれば真。対応しなければ偽。`hash` が生成していない形の `PasswordHash` には偽を返す。`hash` が `null` のときは、照合値がある場合と同じ計算を行って偽を返す。入力の長さと内容によらず、例外を投げずに真偽を返す |

エラーケース: 計算の失敗は `SystemError`。

ポート適合テストの対象は、`hash` と `verify` の対応、同じパスワードから異なる照合値を返すこと、偽を返す入力。総当たりに計算資源を要すること、照合値からパスワードを求められないこと、`null` で同じ計算を行うこと、ログへ出さないことは、メソッドで観測できないので、アダプターごとの検証の対象にする。

### TotpAuthenticator

目的: 確認コードの元になる秘密を生成し、確認コードを照合する。置き場所は `packages/core/src/domain/identity/ports/`。

```ts
interface TotpAuthenticator {
  enroll(account: { email: EmailAddress }): Promise<TotpEnrollment>;
  verify(secret: ProtectedTotpSecret, code: TotpCode, at: Date): Promise<boolean>;
}
```

| メソッド | 契約 |
| --- | --- |
| `enroll` | 呼び出しごとに新しい秘密を生成する。秘密は 160 ビット以上の予測できない値とする。`secret` は保護した形で返す。`setupKey` は、秘密の Base32（RFC 4648、大文字、パディングなし）の表記。`otpauthUri` は `otpauth://totp/` で始まり、ラベルにアカウントのメールアドレス、`secret` パラメータに `setupKey` と同じ値を含む。返した `secret` と、`setupKey` から RFC 6238 で求めた確認コードは、`verify` で一致する。状態を保存しない |
| `verify` | RFC 6238 の TOTP（SHA-1、30 秒、6 桁）で照合する。`at` を含む時間枠と、その前後一つずつの時間枠の確認コードを一致とする。それ以外は偽。状態を保存しない |

エラーケース: 保護した形を解けない秘密は `SystemError`（理由 `SECRET_UNAVAILABLE`）。

ポート適合テストの対象は、`enroll` が返す値の形と、`verify` の一致・不一致。秘密が予測できないこと、鍵が違う構成での `SECRET_UNAVAILABLE`、ログへ出さないことは、アダプターごとの検証の対象にする。

## トランザクション境界

共通の `UnitOfWorkProvider.run(scope, fn)` を使う（[common.md](common.md) の「WriteScope」「UnitOfWorkProvider」）。`InstallationContext` が `accountRepository`、`sessionRepository`、`recoveryRequestRepository` を公開する。UnitOfWork ごとの `WriteScope`、確定の単位、書き込みの回数は、各ユースケース（[../usecases/identity.md](../usecases/identity.md)）と、Installation に公開する手順の節（「初期構成の宣言」）が定める。この節は、集約の間の整合性だけを定める。

- `Account` を書くのは、Identity のユースケース、参加でアカウントを作る Workspace のユースケース（`insert`）、初期構成を読み込む Installation のユースケース（`insertMany`）

## 初期構成の宣言

初期構成の読み込みで、Identity が受ける宣言と、その検証・生成の手順。宣言の形の全体と、`path` に渡す位置は [installation.md](installation.md) が、各ドメインを呼ぶ順は [index.md](index.md) の「手順を呼ぶ順」の表が定める。`DeclarationIssue` と検証の性質は [common.md](common.md) の「DeclarationIssue」が定める。生成は、Installation のユースケースが、`Installation` の `insert` と同じ、設置の区画の UnitOfWork（`{ kind: "installation" }`）で確定する（[index.md](index.md) の「区画をまたぐ手順」の初期構成の読み込み）。

```ts
type OwnerDeclaration = Readonly<{ email: string; name: string; password: string }>;

const OwnerDeclaration: {
  validate(
    entries: readonly Readonly<{ path: string; declaration: OwnerDeclaration }>[],
  ): readonly DeclarationIssue[];
};
```

宣言は、ワークスペースごとの所有者一人の、メールアドレス、名前、初回のログインに使うパスワード。`entries` は、宣言のすべてのワークスペースの所有者。

検証。

| 規則 | `path` | `code` |
| --- | --- | --- |
| `EmailAddress.create`、`AccountName.create`、`PlainPassword.create` | 渡された `path` に `.email`、`.name`、`.password` を続ける | 投げた `ValidationError` の理由 |
| 同じメールアドレス（`EmailAddress` の等価性）を複数の所有者に宣言する場合、名前とパスワードが一致する | 一致しない二件目以降の `path` | `OwnerDeclarationConflict` |

- `OwnerDeclarationConflict` は、この検証だけが返す理由。エラーとして投げない

生成。設置の区画の最初の段（[index.md](index.md) の「区画をまたぐ手順」の初期構成の読み込み）で行う。

- UnitOfWork の前に、メールアドレスごとに、`PasswordHasher.hash` で照合値を求め、`IdGenerator.generate` の ID で `Account.create` を呼ぶ。同じメールアドレスの宣言は、一つのアカウントにする
- UnitOfWork の中で、作ったアカウントの全部を、`AccountRepository.insertMany` の一回の呼び出しで保存する
- 前提: 先に終えているべきドメインは無い
- 生成は、宣言ごとの `AccountId` を返す。Workspace の生成が、所属の索引の項目（同じ設置の区画の UnitOfWork）と、ワークスペースの区画の所有者のメンバーの作成に使う（[workspace.md](workspace.md) の「初期構成の宣言」）
- 所有者のアカウントは、ワークスペースの区画の生成より先に確定する
- 宣言の照合値から除く値: 所有者のパスワード（`OwnerDeclaration.password`）
- 宣言したパスワードは保存しない。`PlainPassword` の扱い（「値オブジェクト」）に加え、起動の結果にも含めない

## 利用再開前の処理

バックアップから復旧した設置で、Identity が利用の再開の前に済ませる処理は無い。残件数を持たない。

- アカウントは、ワークスペースへの所属を持たない。所属の索引と `Member` の食い違いと、アカウントが無いメンバーは、Workspace の「利用再開前の処理」が扱う（[workspace.md](workspace.md)）
- セッションと回復依頼は直さない
- 確認に示す項目と、残件数に数える項目: 無い。`kind` の識別子を持たない

## 利用終了の消去

ワークスペースの利用終了で、Identity が消去するものは無い。アカウント、セッション、回復依頼は、設置の区画にあり、利用終了の後も残る（「Account」のライフサイクル）。終了前の確認に公開する項目を持たない。

## 状態確認の項目

状態確認に示す Identity の項目は無い。`kind` の識別子と、必要な対処の判定を持たない。

## ユースケース（概要）

詳細は [../usecases/identity.md](../usecases/identity.md) が定める。

- パスワードでログインする
- 確認コードを入力してログインを完了する
- ログアウトする
- セッションからアカウントを確かめる
- 多要素認証の設定を始める
- 多要素認証の設定を確定する
- 多要素認証を解除する
- アクセス回復を依頼する
- 管理者復旧の回復の値を発行する
- 回復の値の状態を確かめる
- 新しいパスワードを設定してアクセスを回復する
- 期限切れのセッションと回復依頼を掃除する

Identity のユースケースは、上の一覧がすべて。パスワードは、アクセス回復で置き換える。アカウントの名前は、作成の時に決まる。

アカウントの作成は、招待の受諾（Workspace）と初期構成の読み込み（Installation）のユースケースが `Account.create` を使って行う。初期構成の読み込みの作り方は「初期構成の宣言」が定める。