# 多要素認証を解除する

ユースケースは [../../usecases/identity.md](../../usecases/identity.md) が定める。

- A は、セッションのアカウント。多要素認証は `enabled`

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-identity-disableMfa-01 | A が所属するワークスペースは、どれも多要素認証を必須にしていない | 正しいパスワードと正しい確認コードを入力する | 正常に終わる。A の多要素認証は `disabled` になる。A の次の「パスワードでログインする」は、状態 `authenticated` を返す |
| TC-identity-disableMfa-02 | A が所属するワークスペースは、どれも多要素認証を必須にしていない | 誤ったパスワードと正しい確認コードを入力する | `BusinessRuleError`（`PasswordVerificationFailed`）。A は `enabled` のまま変わらない。セッションは変わらない |
| TC-identity-disableMfa-03 | A が所属するワークスペースは、どれも多要素認証を必須にしていない | 正しいパスワードと誤った確認コードを入力する | `BusinessRuleError`（`CodeVerificationFailed`）。A は `enabled` のまま変わらない。セッションは変わらない |
| TC-identity-disableMfa-04 | A は、多要素認証を必須にした `active` のワークスペース W と、必須にしていない V のメンバー | 正しいパスワードと正しい確認コードを入力する | 正常に終わる。A の多要素認証は `disabled` になる。A の W と V のメンバーと権限の組、W と V の設定は変わらない |
| TC-identity-disableMfa-05 | 多要素認証が `disabled` のアカウント B | B が、パスワードと確認コードを入力する | 正常に終わる。B を書かず、B は `disabled` のまま、`version` は変わらない |
| TC-identity-disableMfa-06 | A のメールアドレスへのログインの失敗が、`LoginAttemptPolicy.limit.max` より 1 回少ない回数ある | 誤ったパスワードと正しい確認コードを入力する。続けて、A のメールアドレスと正しいパスワードで「パスワードでログインする」を行う | 解除は `BusinessRuleError`（`PasswordVerificationFailed`）。続くログインは、状態 `mfa_required` を返す |
