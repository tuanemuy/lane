# 送信接続設定を登録する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-registerMailConnection-01 | 送信接続設定が未登録 | `notification:manage` を持つメンバーが、接続先、送信者のアドレスを含む接続条件、認証情報（`identity` と秘密）を入力して登録する | 入力した接続先と接続条件と、`credentialsRegistered` が真の `MailConnectionView` を返す。認証情報の値を含まない。以後の「送信接続設定を取得する」は同じ内容を返す。操作履歴に `mail_connection.changed` が一件残り、認証情報の値を含まない |
| TC-form-registerMailConnection-02 | 送信接続設定が未登録 | `notification:manage` を持つメンバーが、認証情報を入力せずに登録する | `ValidationError`。`FormInputIssue` が、認証情報の項目（`credentials`）と理由（`MailSecretRequired`）を持つ。送信接続設定は未登録のまま。操作履歴は残らない |
| TC-form-registerMailConnection-03 | 送信接続設定が未登録。`MailSender.settingRule` の規則は、送信者のアドレスの接続条件が無い設定に、その名前の `required` の問題を返す | `notification:manage` を持つメンバーが、送信者のアドレスを含まない接続条件で登録する | `ValidationError`。`FormInputIssue` は、その接続条件の位置（`conditions.名前`）と、理由 `MailSettingRejected`、規則の理由 `required` を持つ。送信接続設定は未登録のまま。操作履歴は残らない |
| TC-form-registerMailConnection-04 | 送信接続設定が登録済み | `notification:manage` を持つメンバーが登録する | `BusinessRuleError`（`MailConnectionAlreadyRegistered`）。登録済みの送信接続設定は変わらない。操作履歴は残らない |
| TC-form-registerMailConnection-05 | 送信接続設定が未登録 | `notification:manage` を持たないメンバーが、登録する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `notification:manage`。送信接続設定は未登録のまま。操作履歴は残らない |
