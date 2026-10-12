# 管理者復旧の回復の値を発行する

ユースケースは [../../usecases/identity.md](../../usecases/identity.md) が定める。

- 実行者は `operator`。A は、登録済みのアカウント

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-identity-issueOperatorRecoveryToken-01 | A に `pending` の回復依頼が無い | A のメールアドレスを指定して発行する | 回復の値と、現在時刻の `RecoveryPolicy.operatorMs` 後の有効期限を返す。URL を返さない。返した値で「回復の値の状態を確かめる」が `usable` を返す。A の、`channel` が `operator` の `pending` の回復依頼を一つ作る。メールを送らない。A のパスワードと多要素認証の設定は変わらず、それまでのパスワードでログインできる |
| TC-identity-issueOperatorRecoveryToken-02 | A に、`channel` が `operator` の `pending` の回復依頼 R がある | A のメールアドレスを指定して発行する | 新しい回復の値と有効期限を返す。R は `revoked` になる。R の値での「新しいパスワードを設定してアクセスを回復する」は `BusinessRuleError`（`RecoveryRequestRevoked`）になる |
| TC-identity-issueOperatorRecoveryToken-03 | A は、どのワークスペースでも権限の組が空のメンバー | A のメールアドレスを指定して発行する | 回復の値と有効期限を返す。対象は役割と権限によらない |
| TC-identity-issueOperatorRecoveryToken-04 | 指定するメールアドレスのアカウントが無い | 登録のないメールアドレスを指定して発行する | `NotFoundError`。回復依頼を作らず、回復の値を返さない |
| TC-identity-issueOperatorRecoveryToken-06 | A に、`channel` が `email` の期限内の `pending` の回復依頼 R がある | A のメールアドレスを指定して発行する | 回復の値と有効期限を返す。R は `pending` のまま変わらず、R の値で「回復の値の状態を確かめる」が `usable` を返す |
| TC-identity-issueOperatorRecoveryToken-07 | A に `pending` の回復依頼が無い。A のメールアドレスでの二つの発行が並行し、一方が先に確定する | もう一方を確定する | `ConflictError`（理由 `PENDING_RECOVERY_EXISTS`）。回復の値を返さない。A の、`channel` が `operator` の `pending` の回復依頼は、先に確定した発行の一つだけで、その値で「回復の値の状態を確かめる」が `usable` を返す |
