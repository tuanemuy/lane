# 新しいパスワードを設定してアクセスを回復する

ユースケースは [../../usecases/identity.md](../../usecases/identity.md) が定める。

- R は、アカウント A の回復依頼。断りが無ければ、`pending` で期限内
- 「新しいパスワード」は、条件を満たすパスワードを指す

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-identity-completeAccessRecovery-01 | A の多要素認証は `disabled`。A はワークスペース W のメンバー | R の値と新しいパスワードを入力する。多要素認証の選択は無し | 多要素認証を解除していないことと、セッションの値、現在時刻の `SessionPolicy.activeMs` 後の有効期限を返す。返した値で「セッションからアカウントを確かめる」が A を返す。R は `used` になる。新しいパスワードでの「パスワードでログインする」は成功し、前のパスワードでは `UnauthenticatedError`（理由 `CREDENTIALS_REJECTED`）になる。A の W のメンバーと権限の組、所属の索引の A の項目は変わらない |
| TC-identity-completeAccessRecovery-03 | A の多要素認証は `enabled` | R の値、新しいパスワード、正しい確認コードを入力する | 多要素認証を解除していないことを返す。A は `enabled` のまま。R は `used` になる。新しいパスワードでの「パスワードでログインする」は、状態 `mfa_required` を返す |
| TC-identity-completeAccessRecovery-04 | A の多要素認証は `enabled` | R の値と新しいパスワードを入力し、多要素認証の解除を選ぶ | 多要素認証を解除したことを返す。A の多要素認証は `disabled` になる。R は `used` になる。新しいパスワードでの「パスワードでログインする」は、状態 `authenticated` を返す |
| TC-identity-completeAccessRecovery-05 | A の多要素認証は `enabled` | R の値と新しいパスワードを入力する。多要素認証の選択は無し | `BusinessRuleError`（`RecoveryMfaChoiceRequired`）。A のパスワードと多要素認証は変わらない。R は `pending` のまま |
| TC-identity-completeAccessRecovery-06 | A の多要素認証は `enabled` | R の値、新しいパスワード、誤った確認コードを入力する | `UnauthenticatedError`（理由 `CODE_REJECTED`）。A のパスワードは変わらない。R は `pending` のまま残り、正しい確認コードで入力し直すと回復できる。エラー情報は、入力したパスワード、確認コード、回復の値を含まない |
| TC-identity-completeAccessRecovery-07 | A の多要素認証は `disabled` | R の値と、条件を満たさないパスワードを入力する | `ValidationError`（`PasswordPolicyViolation`）。A のパスワードは変わらない。R は `pending` のまま残り、条件を満たすパスワードで送信し直すと回復できる |
| TC-identity-completeAccessRecovery-08 | R は `pending` で、有効期限を過ぎている | R の値と新しいパスワードを入力する | `BusinessRuleError`（`RecoveryRequestExpired`）。A のパスワードは変わらない |
| TC-identity-completeAccessRecovery-09 | R は `used` | R の値と新しいパスワードを入力する | `BusinessRuleError`（`RecoveryRequestUsed`）。A のパスワードは変わらない |
| TC-identity-completeAccessRecovery-10 | R は `revoked` | R の値と新しいパスワードを入力する | `BusinessRuleError`（`RecoveryRequestRevoked`）。A のパスワードは変わらない |
| TC-identity-completeAccessRecovery-11 | 入力する値に該当する回復依頼が無い | その値と新しいパスワードを入力する | `NotFoundError`。エラー情報は、入力した回復の値とパスワードを含まない |
| TC-identity-completeAccessRecovery-14 | A の多要素認証は `disabled`。A の `active` のセッション S がある | R の値と新しいパスワードを入力する | パスワードを置き換える。S は変わらず、S の値で「セッションからアカウントを確かめる」が A を返す |
| TC-identity-completeAccessRecovery-15 | R は `channel` が `operator`。A に、`channel` が `email` の期限内の `pending` の回復依頼 Q がある | R の値と新しいパスワードを入力する | R は `used` になる。Q は `pending` のまま変わらない |
| TC-identity-completeAccessRecovery-16 | A の多要素認証は `disabled`。A のメールアドレスへのログインの失敗が `LoginAttemptPolicy.limit.max` 回に達し、最初の失敗から `LoginAttemptPolicy.limit.windowMs` が過ぎていない | R の値と新しいパスワードを入力する。続けて、新しいパスワードで「パスワードでログインする」を行う | 回復は完了し、R は `used` になる。ログインが要ることと `retryAt` を返し、セッションの値を返さない。続くログインは `RateLimitedError`（理由 `LOGIN_ATTEMPTS_LIMITED`）。`retryAt` の後のログインは、状態 `authenticated` を返す |
| TC-identity-completeAccessRecovery-17 | A の多要素認証は `enrolling` | R の値と新しいパスワードを入力し、多要素認証の解除を選ぶ | 多要素認証を解除していないことを返す。パスワードを置き換え、R は `used` になる。A の多要素認証は `enrolling` のまま、秘密は変わらない |
| TC-identity-completeAccessRecovery-18 | A の多要素認証は `enabled`。A のメールアドレスへのログインの失敗が、`LoginAttemptPolicy.limit.max` より 1 回少ない回数ある | R の値、新しいパスワード、誤った確認コードを入力する。続けて、A のメールアドレスと正しいパスワードで「パスワードでログインする」を行う | 回復は `UnauthenticatedError`（理由 `CODE_REJECTED`）。続くログインは、状態 `mfa_required` を返す |
