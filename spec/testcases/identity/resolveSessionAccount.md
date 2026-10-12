# セッションからアカウントを確かめる

ユースケースは [../../usecases/identity.md](../../usecases/identity.md) が定める。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-identity-resolveSessionAccount-01 | アカウント A の `active` で期限内のセッション S がある | S の値で確かめる | S と A を返す |
| TC-identity-resolveSessionAccount-02 | A の `active` で期限内のセッション S がある | S の値で二回確かめる | 二回とも同じ有効期限を返す。有効期限は延びない |
| TC-identity-resolveSessionAccount-03 | A の `active` のセッション S の認証から `SessionPolicy.activeMs` が過ぎている。S は削除されていない | S の値で確かめる | `UnauthenticatedError`（理由 `SESSION_INVALID`） |
| TC-identity-resolveSessionAccount-04 | A の `pending_mfa` のセッション S がある | S の値で確かめる | `UnauthenticatedError`（理由 `SESSION_INVALID`） |
| TC-identity-resolveSessionAccount-05 | 入力する値に該当するセッションが無い | その値で確かめる | `UnauthenticatedError`（理由 `SESSION_INVALID`） |
| TC-identity-resolveSessionAccount-06 | なし | セッションの値を示さずに確かめる | `UnauthenticatedError`（理由 `SESSION_INVALID`） |
