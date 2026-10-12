# 通知定義を変更する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-changeNotificationTemplate-01 | 現行のフォーム F の `active` の設定 | `notification:manage` を持つメンバー M が、F の現行の項目だけを参照する件名、本文、送信者の表示名、返信先を入力して変更する | 変更後の通知定義を返す。F の通知定義が入力の内容に置き換わる。操作履歴に `form.notification_definition_changed` が一件残る。実行者は M、対象は F |
| TC-form-changeNotificationTemplate-02 | F の受付済みの回答 S がある | `notification:manage` を持つメンバーが、F の通知定義を変更する | S の `template` は変わらない |
| TC-form-changeNotificationTemplate-04 | 現行のフォーム F の `active` の設定。F に項目 `company` が無い（編集を始めた後に、スキーマの反映で無くなった場合を含む） | `notification:manage` を持つメンバーが、本文で `company` を参照する通知定義に変更する | `BusinessRuleError`（`NotificationReferenceBroken`）。`company` と理由 `fieldRemoved` の `ReferenceIssue` を添える。F の通知定義は変わらない。操作履歴は残らない |
| TC-form-changeNotificationTemplate-05 | 現行のフォーム F の `active` の設定。F の項目 `name` は、メールアドレスの形式でない `text` の項目 | `notification:manage` を持つメンバーが、返信先に `name` を指定する通知定義に変更する | `BusinessRuleError`（`NotificationReferenceBroken`）。`replyTo` の `name` と理由 `kindUnsuitable` の `ReferenceIssue` を添える。F の通知定義は変わらない |
| TC-form-changeNotificationTemplate-07 | F の通知定義の本文が `company` を参照する。スキーマの反映で F の項目 `company` が無くなった | `notification:manage` を持つメンバーが、現在の通知定義と等しい内容に変更する | `BusinessRuleError`（`NotificationReferenceBroken`）。`company` と理由 `fieldRemoved` の `ReferenceIssue` を添える。F の通知定義は変わらない。操作履歴は残らない |
| TC-form-changeNotificationTemplate-08 | 削除済みフォーム F の `removed` の設定（編集を始めた後に、スキーマの反映で F が削除された場合を含む） | `notification:manage` を持つメンバーが、F の通知定義を変更する | `BusinessRuleError`（`FormRemoved`）。操作履歴は残らない |
| TC-form-changeNotificationTemplate-09 | 現行のフォーム F の `active` の設定 | `notification:manage` を持つメンバーが、件名を空にして変更し、続けて、本文に記法の誤りを含めて変更する | どちらも `ValidationError`。`FormInputIssue` は、項目の位置と理由 `TemplateSyntax` と、記法の誤りを持つ。F の通知定義は変わらない。操作履歴は残らない |
| TC-form-changeNotificationTemplate-12 | 現行のフォーム F の `active` の設定。参照の検証の後、確定までに、スキーマの反映が確定する（F を削除する反映を含む） | `notification:manage` を持つメンバーが、F の通知定義を変更する | `ConflictError`（`SCHEMA_CHANGED`）。F の通知定義は変わらない。操作履歴は残らない |
| TC-form-changeNotificationTemplate-14 | 現行のフォーム F の `active` の設定 | `intake:manage` だけを持つメンバーが、F の通知定義を変更する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `notification:manage`。F の通知定義は変わらない。操作履歴は残らない |
| TC-form-changeNotificationTemplate-15 | 現行のフォーム F の `active` の設定。F の通知定義に参照の不整合は無い | `notification:manage` を持つメンバーが、現在の通知定義と等しい内容に変更する | 現在の通知定義を返す。保存せず、F の設定の `version` は変わらない。操作履歴は残らない |
