# 確認コードを入力してログインを完了する

ユースケースは [../../usecases/identity.md](../../usecases/identity.md) が定める。

- A は、多要素認証が `enabled` のアカウント。S は、A の「パスワードでログインする」が始めた `pending_mfa` のセッション
- 断りが無ければ、S は作成から `SessionPolicy.pendingMfaMs` 以内で、A のメールアドレスへのログインの失敗の記録は無い

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-identity-completeMfaLogin-01 | S がある | S の値と、正しい確認コードを入力する | 現在時刻の `SessionPolicy.activeMs` 後の有効期限を返す。S は `active` になり、セッションの値は変わらない。同じ値で「セッションからアカウントを確かめる」が A を返す |
| TC-identity-completeMfaLogin-02 | S がある | S の値と、誤った確認コードを入力する | `UnauthenticatedError`（理由 `CODE_REJECTED`）。S は `pending_mfa` のまま残る。A のメールアドレスへの失敗を一つ記録する。続けて正しい確認コードを入力すると、ログインを完了できる。エラー情報は、入力した確認コードとセッションの値を含まない |
| TC-identity-completeMfaLogin-03 | S の作成から `SessionPolicy.pendingMfaMs` が過ぎている | S の値と、正しい確認コードを入力する | `UnauthenticatedError`（理由 `SESSION_INVALID`）。S は `active` にならない |
| TC-identity-completeMfaLogin-05 | S がある。S の作成の後に、A が「新しいパスワードを設定してアクセスを回復する」で多要素認証を解除した | S の値と、確認コードを入力する | `UnauthenticatedError`（理由 `SESSION_INVALID`）。S は `active` にならない。A のメールアドレスへの失敗を記録しない。A のメールアドレスと新しいパスワードでの「パスワードでログインする」は、状態 `authenticated` を返す |
| TC-identity-completeMfaLogin-06 | A の `active` のセッション T がある | T の値と、正しい確認コードを入力する | `UnauthenticatedError`（理由 `SESSION_INVALID`）。T は変わらず、T の値で「セッションからアカウントを確かめる」が A を返す。A のメールアドレスへの失敗を記録しない |
| TC-identity-completeMfaLogin-07 | S がある。A のメールアドレスへの失敗が、最初の失敗から `LoginAttemptPolicy.limit.windowMs` の間に `LoginAttemptPolicy.limit.max` 回に達している | S の値と、正しい確認コードを入力する | `RateLimitedError`（理由 `LOGIN_ATTEMPTS_LIMITED`）。`retryAt` は最初の失敗の `LoginAttemptPolicy.limit.windowMs` 後。S は `pending_mfa` のまま残る |
| TC-identity-completeMfaLogin-08 | S がある。A のメールアドレスへの失敗が、`LoginAttemptPolicy.limit.max` より 1 回少ない回数ある | S の値と、誤った確認コードを入力する。続けて、A のメールアドレスと正しいパスワードで「パスワードでログインする」を行う | 確認コードの入力は `UnauthenticatedError`（理由 `CODE_REJECTED`）。続くログインは `RateLimitedError`（理由 `LOGIN_ATTEMPTS_LIMITED`） |
