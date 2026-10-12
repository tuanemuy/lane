# パスワードでログインする

ユースケースは [../../usecases/identity.md](../../usecases/identity.md) が定める。

- A は、登録済みのアカウント。断りが無ければ、A のメールアドレスへのログインの失敗の記録は無い

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-identity-logInWithPassword-01 | A の多要素認証が `disabled` | A のメールアドレスと正しいパスワードでログインする | 状態 `authenticated`、セッションの値、現在時刻の `SessionPolicy.activeMs` 後の有効期限を返す。返した値で「セッションからアカウントを確かめる」が A を返す。操作履歴を残さない |
| TC-identity-logInWithPassword-02 | A の多要素認証が `enabled` | A のメールアドレスと正しいパスワードでログインする | 状態 `mfa_required`、セッションの値、現在時刻の `SessionPolicy.pendingMfaMs` 後の有効期限を返す。返した値で「セッションからアカウントを確かめる」は `UnauthenticatedError`（理由 `SESSION_INVALID`）になる |
| TC-identity-logInWithPassword-03 | A の多要素認証が `enrolling` | A のメールアドレスと正しいパスワードでログインする | 状態 `authenticated` を返す。確認コードを求めない |
| TC-identity-logInWithPassword-04 | A がある | A のメールアドレスと誤ったパスワードでログインする | `UnauthenticatedError`（理由 `CREDENTIALS_REJECTED`）。セッションを作らない。A のメールアドレスへの失敗を一つ記録する。エラー情報は、入力したパスワードを含まない |
| TC-identity-logInWithPassword-05 | 入力するメールアドレスのアカウントが無い | 登録のないメールアドレスでログインする | `UnauthenticatedError`（理由 `CREDENTIALS_REJECTED`）。パスワードの誤りと同じ応答を返す。そのメールアドレスへの失敗を一つ記録する |
| TC-identity-logInWithPassword-07 | A のメールアドレスへの失敗が、最初の失敗から `LoginAttemptPolicy.limit.windowMs` の間に `LoginAttemptPolicy.limit.max` 回に達している。最初の失敗から `LoginAttemptPolicy.limit.windowMs` が過ぎていない | A のメールアドレスと正しいパスワードでログインする | `RateLimitedError`（理由 `LOGIN_ATTEMPTS_LIMITED`）。`retryAt` は最初の失敗の `LoginAttemptPolicy.limit.windowMs` 後。セッションを作らない |
| TC-identity-logInWithPassword-09 | A のメールアドレスへの失敗が `LoginAttemptPolicy.limit.max` 回に達し、最初の失敗から `LoginAttemptPolicy.limit.windowMs` が過ぎている | A のメールアドレスと正しいパスワードでログインする | 状態 `authenticated` とセッションの値を返す |
| TC-identity-logInWithPassword-10 | A のメールアドレスへの失敗が、`LoginAttemptPolicy.limit.max` より 1 回少ない回数あり、その後にログインに成功し、さらに 1 回失敗している。すべて最初の失敗から `LoginAttemptPolicy.limit.windowMs` 以内 | A のメールアドレスと正しいパスワードでログインする | `RateLimitedError`（理由 `LOGIN_ATTEMPTS_LIMITED`） |
| TC-identity-logInWithPassword-11 | A は、どのワークスペースにも所属しない。A の多要素認証が `disabled` | A のメールアドレスと正しいパスワードでログインする | 状態 `authenticated` とセッションの値を返す |
