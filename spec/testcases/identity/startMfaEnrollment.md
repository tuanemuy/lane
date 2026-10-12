# 多要素認証の設定を始める

ユースケースは [../../usecases/identity.md](../../usecases/identity.md) が定める。

- A は、セッションのアカウント

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-identity-startMfaEnrollment-01 | A の多要素認証が `disabled` | A の正しいパスワードを入力する | 認証アプリに登録する情報（`setupKey` と `otpauthUri`）を返す。保護した秘密は返さない。A の多要素認証は `enrolling` になる |
| TC-identity-startMfaEnrollment-02 | A の多要素認証が `enrolling`。前の実行が返した登録の情報を P とする | A の正しいパスワードを入力する | P と違う登録の情報を返す。A は `enrolling` のまま、秘密を置き換える。P から求めた確認コードでの「多要素認証の設定を確定する」は `BusinessRuleError`（`CodeVerificationFailed`）になる |
| TC-identity-startMfaEnrollment-03 | A の多要素認証が `disabled` | 誤ったパスワードを入力する | `BusinessRuleError`（`PasswordVerificationFailed`）。登録の情報を返さない。A は `disabled` のまま変わらない。セッションは変わらず、同じセッションで操作を続けられる |
| TC-identity-startMfaEnrollment-04 | A の多要素認証が `enabled` | A の正しいパスワードを入力する | `BusinessRuleError`（`MfaAlreadyEnabled`）。登録の情報を返さない。A は `enabled` のまま、秘密は変わらない |
