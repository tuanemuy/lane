# 送信接続設定を更新する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-updateMailConnection-01 | 送信接続設定が登録済み | `notification:manage` を持つメンバーが、接続先、送信者のアドレスを含む接続条件、新しい認証情報（`identity` と秘密）を入力して更新する | 入力した接続先と接続条件の `MailConnectionView` を返す。認証情報の値を含まない。保存した送信接続設定は、入力した値と新しい認証情報を持つ。操作履歴に `mail_connection.changed` が一件残り、認証情報の値を含まない |
| TC-form-updateMailConnection-02 | 送信接続設定が登録済み | `notification:manage` を持つメンバーが、認証情報を入力せず（`credentials` が `null`）、接続先を変えて更新する | 接続先が置き換わる。保存した送信接続設定の認証情報は、更新の前と同じ |
| TC-form-updateMailConnection-03 | 送信接続設定が登録済み | `notification:manage` を持つメンバーが、認証情報を入力せず、登録済みと同じ接続先と接続条件で更新する | 現在の `MailConnectionView` を返す。保存せず、`version` は変わらない。操作履歴は残らない |
| TC-form-updateMailConnection-04 | 送信接続設定が登録済み。受付中のフォーム、受付済みの回答、未送信の通知がある | `notification:manage` を持つメンバーが更新する | 回答の `template` と `retention`、通知の状態、`FormSettings` は変わらない |
| TC-form-updateMailConnection-05 | 送信接続設定が登録済み。受付中のフォーム F がある | `notification:manage` を持つメンバーが、接続先を空にして更新する | `ValidationError`。`FormInputIssue` が、接続先の項目と理由（`InvalidMailDestination`）を持つ。送信接続設定は変わらない。F は受付中のまま。操作履歴は残らない |
| TC-form-updateMailConnection-06 | 送信接続設定が登録済み。受付中のフォームは無い。`MailSender.settingRule` の規則は、送信者のアドレスの接続条件が無い設定に、その名前の `required` の問題を返す | `notification:manage` を持つメンバーが、接続条件から送信者のアドレスを除いて更新する | `ValidationError`。`FormInputIssue` は、その接続条件の位置（`conditions.名前`）と、理由 `MailSettingRejected`、規則の理由 `required` を持つ。送信接続設定は変わらない。操作履歴は残らない |
| TC-form-updateMailConnection-07 | 送信接続設定が未登録 | `notification:manage` を持つメンバーが更新する | `NotFoundError`。送信接続設定は未登録のまま |
| TC-form-updateMailConnection-08 | 送信接続設定が登録済み | `notification:manage` を持たないメンバーが、更新する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `notification:manage`。送信接続設定は変わらない。操作履歴は残らない |
| TC-form-updateMailConnection-10 | 送信接続設定が登録済み。受付中のフォームは無い | `notification:manage` を持つメンバーが、接続条件を一つも持たない入力で更新する | `ValidationError`。`FormInputIssue` は、`conditions` と理由 `InvalidMailCondition` を持つ。送信接続設定は変わらない。操作履歴は残らない |
