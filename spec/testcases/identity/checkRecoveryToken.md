# 回復の値の状態を確かめる

ユースケースは [../../usecases/identity.md](../../usecases/identity.md) が定める。

- R は、アカウント A の回復依頼

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-identity-checkRecoveryToken-01 | R は `pending` で期限内。A の多要素認証は `disabled` | R の値で確かめる | `usable` と、A が多要素認証を設定していないことを返す。R は `pending` のまま変わらない |
| TC-identity-checkRecoveryToken-02 | R は `pending` で期限内。A の多要素認証は `enabled` | R の値で確かめる | `usable` と、A が多要素認証を設定していることを返す |
| TC-identity-checkRecoveryToken-04 | R は「アクセス回復を依頼する」が作った `pending` で、作成から `RecoveryPolicy.emailMs` が過ぎている | R の値で確かめる | `expired` を返す。エラーにしない |
| TC-identity-checkRecoveryToken-05 | R は `used` | R の値で確かめる | `used` を返す。エラーにしない |
| TC-identity-checkRecoveryToken-06 | R は `revoked` | R の値で確かめる | `revoked` を返す。エラーにしない |
| TC-identity-checkRecoveryToken-10 | 入力する値に該当する回復依頼が無い | その値で確かめる | `NotFoundError`。エラー情報は、対象の種類だけを持ち、入力した回復の値を含まない |
