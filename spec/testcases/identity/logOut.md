# ログアウトする

ユースケースは [../../usecases/identity.md](../../usecases/identity.md) が定める。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-identity-logOut-01 | アカウント A の `active` のセッション S がある | S の値でログアウトする | 正常に終わる。S を削除する。S の値で「セッションからアカウントを確かめる」は `UnauthenticatedError`（理由 `SESSION_INVALID`）になる |
| TC-identity-logOut-03 | A の `active` のセッション S と T がある | S の値でログアウトする | T は残る。T の値で「セッションからアカウントを確かめる」が A を返す |
| TC-identity-logOut-04 | 入力する値に該当するセッションが無い | その値でログアウトする | 正常に終わる。何も書かない |
| TC-identity-logOut-05 | A の `active` のセッション S がある。A のメンバーに紐付くトークンと、A のメンバーが手動発行したトークンがある | S の値でログアウトする | どちらのトークンも変わらず、失効しない |
| TC-identity-logOut-06 | A の `pending_mfa` のセッション S と、期限切れのセッション E がある | S の値と E の値で、それぞれログアウトする | どちらも正常に終わる。S と E を削除する |
