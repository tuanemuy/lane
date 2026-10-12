# 送信接続設定の動作を確認する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-testMailConnection-01 | 送信接続設定が未登録。`MailSender` は受理を返す | `notification:manage` を持つメンバーが、認証情報を含む設定と、試験メールの宛先を入力して動作を確認する | `accepted` を返す。`MailSender` は、入力した設定（送信者のアドレスの接続条件を含む）と、その宛先一つを一度受け取る。試験メールは、送信者の表示名と返信先を持たず、回答の入力内容を含まない。送信接続設定は未登録のまま |
| TC-form-testMailConnection-02 | 送信接続設定が登録済み。`MailSender` は受理を返す | `notification:manage` を持つメンバーが、認証情報を入力せず（`credentials` が `null`）、接続先を変えた設定で動作を確認する | `accepted` を返す。`MailSender` が受け取る設定は、入力した接続先と、登録済みの認証情報を持つ。登録済みの送信接続設定は変わらない |
| TC-form-testMailConnection-03 | 送信接続設定が登録済み。`MailSender` は `failed`（`authentication`）を返す | `notification:manage` を持つメンバーが動作を確認する | エラーにしない。`failed` と理由 `authentication` を返す。登録済みの送信接続設定は変わらない |
| TC-form-testMailConnection-07 | 送信接続設定が登録済み | `notification:manage` を持たないメンバーが、動作を確認する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `notification:manage`。メールを送らない |
| TC-form-testMailConnection-09 | 送信接続設定が登録済み。`MailSender` が試験メールを受理した後、結果を返す前に、処理が中断する | `notification:manage` を持つメンバーが、接続先を変えた設定で動作を確認する | 登録済みの送信接続設定は変わらない。受理された試験メールは取り消さない |
