# 多要素認証の設定を確定する

ユースケースは [../../usecases/identity.md](../../usecases/identity.md) が定める。

- A は、セッションのアカウント。多要素認証は `enrolling`。E は、「多要素認証の設定を始める」が返した登録の情報

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-identity-confirmMfaEnrollment-01 | A の `active` のセッション S がある | E から求めた、現在の時間枠の確認コードを入力する | 多要素認証を有効にした日時（現在時刻）を返す。A の多要素認証は `enabled` になる。S は変わらず、S の値で「セッションからアカウントを確かめる」が A を返す |
| TC-identity-confirmMfaEnrollment-03 | なし | 誤った確認コードを入力する | `BusinessRuleError`（`CodeVerificationFailed`）。A は `enrolling` のまま変わらない。セッションは変わらない。続けて正しい確認コードを入力すると、確定できる |
| TC-identity-confirmMfaEnrollment-04 | A の多要素認証が `disabled`（設定を始めていない） | 確認コードを入力する | `BusinessRuleError`（`MfaEnrollmentNotStarted`）。A は `disabled` のまま変わらない |
