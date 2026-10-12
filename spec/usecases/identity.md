# Identity のユースケース

人の認証、セッション、多要素認証、アクセス回復を実現する。型・振る舞い・ポートは [../domains/identity.md](../domains/identity.md) が定める。規則は domains が、振る舞いの結果は `spec/testcases/identity/` のユースケーステストが定め、ユースケースは、振る舞いとポートを組み合わせる。見出しの識別子は、`spec/testcases/identity/` のファイル名と同じ。

## 共通

- 実行者の種類: `Actor` を入力に取るユースケース（`issueOperatorRecoveryToken`、`purgeExpiredSessionsAndRecoveryRequests`）は、受け付ける実行者の種類を「認可」に書く。判定は [../domains/index.md](../domains/index.md) の「認可」。ほかのユースケースは `Actor` を取らない
- セッションのアカウント: ログインの後のユースケースは、「セッションからアカウントを確かめる」が返したアカウントを入力に取る
- 照合: 照合の結果（`passwordMatches`、`codeMatches`）は、ユースケースがポートを呼んで求め、ドメインサービスに渡す。求め方と、結果の扱いは [../domains/identity.md](../domains/identity.md) の「ドメインサービス」
- ログインの試行の制限: セッションを始めるドメインサービスを呼ぶユースケースは、照合の前に、アカウントのメールアドレス（ログインでは入力したメールアドレス）の `LoginAttemptPolicy.key` と `LoginAttemptPolicy.limit` で `UsageLimiter.check` を呼び、結果を `standing` として渡す。`Authentication` の結果が `rejected` なら、`UsageLimiter.record` で失敗を一つ記録する。`now` には、ユースケースの初めに求めた現在時刻を渡す
- 秘密の値: 扱いは [../domains/identity.md](../domains/identity.md) の「値オブジェクト」。値で引く対象の `NotFoundError` は [../domains/index.md](../domains/index.md) の「共通の型とポート」のエラーの表
- 一度きりの値: セッションの値と回復の値で集約を引くユースケースは、[../domains/common.md](../domains/common.md) の「期限つきの一度きりの値」に従う。`lapsedAt` は `Session.lapsedAt`、`RecoveryRequest.lapsedAt`
- 回復用の URL: メールの本文に入れる回復用の URL は、[../domains/index.md](../domains/index.md) の「URL の組み立て」に従い、`RecoveryLink.url` で作る
- UnitOfWork: Identity のユースケースが開く UnitOfWork の `WriteScope` は、どれも `{ kind: "installation" }`（[../domains/common.md](../domains/common.md) の「WriteScope」）。各ユースケースの「UnitOfWork」は、この区画の UnitOfWork を指す。ワークスペースの区画と、所属の索引（Workspace）を読み書きしない

## パスワードでログインする（`logInWithPassword`）

メールアドレスとパスワードを照合し、セッションを始める。

- **シナリオ**: AUTH-02
- **認可**: 不要（ログインの前）
- **入力**: メールアドレス、パスワード
- **出力**: セッションの値と有効期限。結果の種類は `Authentication.loginWithPassword` の `authenticated` / `mfa_required`
- **使用するドメインの振る舞い・ポート**: `EmailAddress.create`、`Clock.now`、`LoginAttemptPolicy.key`・`limit`、`UsageLimiter.check`・`record`、`AccountRepository.findByEmail`、`PasswordHasher.verify`、`SecretTokenIssuer.issue`、`SessionTokenDigest.create`、`IdGenerator.generate`、`Authentication.loginWithPassword`、`SessionRepository.insert`
- **トランザクション境界**: 一つの UnitOfWork（`sessionRepository.insert`）。`UsageLimiter.check` は、UnitOfWork の前に呼ぶ

## 確認コードを入力してログインを完了する（`completeMfaLogin`）

確認コードの入力を待つセッションを、確認コードの照合で `active` にする。

- **シナリオ**: AUTH-02
- **認可**: セッションの値
- **入力**: セッションの値、確認コード
- **出力**: `active` になったセッションの有効期限。結果の種類は `Authentication.completeMfa` の `authenticated`
- **使用するドメインの振る舞い・ポート**: `SecretTokenIssuer.digest`、`SessionTokenDigest.create`、`SessionRepository.findByTokenDigest`、`Clock.now`、`Session.lapsedAt`、`OneTimeValueRetention.isForgotten`、`Session.standing`、`AccountRepository.findById`、`Account.isMfaEnabled`、`LoginAttemptPolicy.key`・`limit`、`UsageLimiter.check`・`record`、`TotpCode.parse`、`Account.totpSecret`、`TotpAuthenticator.verify`、`Authentication.completeMfa`、`SessionRepository.save`
- **トランザクション境界**: 一つの UnitOfWork（`sessionRepository.save`）。`UsageLimiter.check` は、UnitOfWork の前に呼ぶ

## ログアウトする（`logOut`）

セッションを終了する。

- **シナリオ**: AUTH-02
- **認可**: セッションの値
- **入力**: セッションの値
- **出力**: なし
- **使用するドメインの振る舞い・ポート**: `SecretTokenIssuer.digest`、`SessionTokenDigest.create`、`SessionRepository.findByTokenDigest`、`Clock.now`、`Session.lapsedAt`、`OneTimeValueRetention.isForgotten`、`SessionRepository.delete`
- **トランザクション境界**: 一つの UnitOfWork（`sessionRepository.delete`）

## セッションからアカウントを確かめる（`resolveSessionAccount`）

要求が示したセッションの値から、操作を認めるアカウントを求める。管理画面の要求ごとに行う。

- **シナリオ**: AUTH-02
- **認可**: セッションの値
- **入力**: セッションの値
- **出力**: セッションと、そのアカウント。入口は、アカウントの ID を、セッションで確かめたアカウントとして後続のユースケースへ渡す
- **使用するドメインの振る舞い・ポート**: `SecretTokenIssuer.digest`、`SessionTokenDigest.create`、`SessionRepository.findByTokenDigest`、`Clock.now`、`Session.lapsedAt`、`OneTimeValueRetention.isForgotten`、`Session.standing`、`AccountRepository.findById`
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

## 多要素認証の設定を始める（`startMfaEnrollment`）

パスワードで本人であることを確かめ、認証アプリに登録する情報を作る。

- **シナリオ**: AUTH-03
- **認可**: セッションのアカウント
- **入力**: セッションのアカウント、パスワード
- **出力**: 認証アプリに登録する情報（`TotpEnrollment` の `setupKey` と `otpauthUri`）
- **使用するドメインの振る舞い・ポート**: `Clock.now`、`AccountRepository.findById`、`PasswordHasher.verify`、`TotpAuthenticator.enroll`、`MfaEnrollment.start`、`AccountRepository.save`
- **トランザクション境界**: 一つの UnitOfWork（`accountRepository` の読み取りと `save`）

## 多要素認証の設定を確定する（`confirmMfaEnrollment`）

認証アプリが表示する確認コードを照合し、多要素認証を有効にする。

- **シナリオ**: AUTH-03
- **認可**: セッションのアカウント
- **入力**: セッションのアカウント、確認コード
- **出力**: 多要素認証を有効にした日時
- **使用するドメインの振る舞い・ポート**: `Clock.now`、`AccountRepository.findById`、`TotpCode.parse`、`Account.totpSecret`、`TotpAuthenticator.verify`、`MfaEnrollment.confirm`、`AccountRepository.save`
- **トランザクション境界**: 一つの UnitOfWork（`accountRepository` の読み取りと `save`）

## 多要素認証を解除する（`disableMfa`）

パスワードと確認コードで本人であることを確かめ、多要素認証を解除する。解除の前に示す、多要素認証を必須にしたワークスペース（AUTH-03）は、Workspace の「所属するワークスペースを一覧する（`listMyWorkspaces`）」が返す。

- **シナリオ**: AUTH-03
- **認可**: セッションのアカウント
- **入力**: セッションのアカウント、パスワード、確認コード
- **出力**: なし
- **使用するドメインの振る舞い・ポート**: `Clock.now`、`AccountRepository.findById`、`PasswordHasher.verify`、`TotpCode.parse`、`Account.totpSecret`、`TotpAuthenticator.verify`、`MfaEnrollment.disable`、`AccountRepository.save`。Workspace のポートを読まない
- **トランザクション境界**: 一つの UnitOfWork（`accountRepository` の読み取りと `save`）

## アクセス回復を依頼する（`requestAccessRecovery`）

入力したメールアドレスのアカウントに、回復用の URL をメールで届ける。

- **シナリオ**: AUTH-04
- **認可**: 不要（ログインの前）
- **入力**: メールアドレス
- **出力**: 結果の種類 `requested` / `mail_not_configured`
- **使用するドメインの振る舞い・ポート**: `EmailAddress.create`、`Clock.now`、`SystemMailer.isConfigured`、`AccountRepository.findByEmail`、`RecoveryRequestRepository.findPendingByAccount`（`channel` は `email`）、`SecretTokenIssuer.issue`、`RecoveryTokenDigest.create`、`IdGenerator.generate`、`AccessRecovery.issue`（`channel` は `email`）、`RecoveryRequestRepository.save`・`insert`、設置の公開の URL（`InstallationEnvironment`）、`RecoveryLink.url`、`SystemMailer.send`
- **トランザクション境界**: 一つの UnitOfWork（同じ経路の前の `pending` を `revoked` にする `save` と、新しい `pending` の `insert`）。`SystemMailer.send` は、UnitOfWork の確定の後に呼ぶ

## 管理者復旧の回復の値を発行する（`issueOperatorRecoveryToken`）

ホスティング運営者が、指定したアカウントの回復の値を発行し、運用コマンドの結果として受け取る。運用コマンドの入口が、返した値から回復用の URL を作って示す。

- **シナリオ**: HOST-11
- **認可**: 実行者は `operator` だけ。設置の状態でこのコマンドを行えることは、運用コマンドの入口が、ユースケースを呼ぶ前に確かめる（[../domains/index.md](../domains/index.md) の「設置の状態」）
- **入力**: `Actor`、メールアドレス
- **出力**: 回復の値と有効期限
- **使用するドメインの振る舞い・ポート**: `EmailAddress.create`、`Clock.now`、`AccountRepository.findByEmail`、`RecoveryRequestRepository.findPendingByAccount`（`channel` は `operator`）、`SecretTokenIssuer.issue`、`RecoveryTokenDigest.create`、`IdGenerator.generate`、`AccessRecovery.issue`（`channel` は `operator`）、`RecoveryRequestRepository.save`・`insert`
- **トランザクション境界**: 一つの UnitOfWork（同じ経路の前の `pending` を `revoked` にする `save` と、新しい `pending` の `insert`）

## 回復の値の状態を確かめる（`checkRecoveryToken`）

回復の値が使えるかと、回復に確認コードが要るかを求める。

- **シナリオ**: AUTH-04、HOST-11
- **認可**: 回復の値
- **入力**: 回復の値
- **出力**: `RecoveryStanding`（`usable` / `expired` / `used` / `revoked`）と、`usable` の場合に、アカウントが多要素認証を設定しているか
- **使用するドメインの振る舞い・ポート**: `SecretTokenIssuer.digest`、`RecoveryTokenDigest.create`、`RecoveryRequestRepository.findByTokenDigest`、`Clock.now`、`RecoveryRequest.lapsedAt`、`OneTimeValueRetention.isForgotten`、`RecoveryRequest.standing`、`AccountRepository.findById`、`Account.isMfaEnabled`
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

## 新しいパスワードを設定してアクセスを回復する（`completeAccessRecovery`）

回復依頼と引き換えに、パスワードを置き換え、セッションを始める。多要素認証を設定したアカウントは、確認コードを照合するか、多要素認証を解除する。

- **シナリオ**: AUTH-04、HOST-11
- **認可**: 回復の値
- **入力**: 回復の値、新しいパスワード、多要素認証の選択（`RecoveryMfaChoice`。多要素認証を設定していないアカウントでは無し）
- **出力**: 多要素認証を解除したかどうか（`RecoveryReset` の `kind`）と、`SessionIssuance.start` の結果の種類 `started`（セッションの値と有効期限）/ `login_required`（`retryAt`）
- **使用するドメインの振る舞い・ポート**: `SecretTokenIssuer.digest`、`RecoveryTokenDigest.create`、`RecoveryRequestRepository.findByTokenDigest`、`Clock.now`、`RecoveryRequest.lapsedAt`、`OneTimeValueRetention.isForgotten`、`AccountRepository.findById`、`TotpCode.parse`、`Account.totpSecret`、`TotpAuthenticator.verify`（確認コードを入力した場合）、`AccessRecovery.admit`、`PasswordHasher.hash`（`admitted` の場合）、`LoginAttemptPolicy.key`・`limit`、`UsageLimiter.check`、`AccessRecovery.complete`、`SecretTokenIssuer.issue`、`SessionTokenDigest.create`、`IdGenerator.generate`、`SessionIssuance.start`、`AccountRepository.save`、`RecoveryRequestRepository.save`、`SessionRepository.insert`（`started` の場合）
- **トランザクション境界**: 一つの UnitOfWork（`recoveryRequestRepository` と `accountRepository` の読み取り、両方の `save`、`started` の場合は `sessionRepository.insert`）。`PasswordHasher.hash` と `UsageLimiter.check` は、UnitOfWork の前に呼ぶ

## 期限切れのセッションと回復依頼を掃除する（`purgeExpiredSessionsAndRecoveryRequests`）

定期ジョブ「期限切れの記録の掃除」のうち、設置の区画のセッションと回復依頼の分。ワークスペースごとに分けない。

- **シナリオ**: なし（[../domains/index.md](../domains/index.md) の「定期ジョブ」）
- **認可**: 実行者は `system` だけ
- **入力**: `Actor` だけ
- **出力**: 削除したセッションと回復依頼の件数
- **使用するドメインの振る舞い・ポート**: `Clock.now`、`OneTimeValueRetention.cutoff`、`SessionRepository.deleteLapsed`、`RecoveryRequestRepository.deleteLapsed`
- **トランザクション境界**: [../domains/common.md](../domains/common.md) の「期限つきの一度きりの値」の「掃除」。対象のポートは `SessionRepository.deleteLapsed` と `RecoveryRequestRepository.deleteLapsed`
