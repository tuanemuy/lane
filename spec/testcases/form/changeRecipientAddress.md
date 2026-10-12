# 通知先のメールアドレスを修正する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-changeRecipientAddress-01 | フォーム F の `active` の設定。通知先は、無効な A、有効な B の順 | `notification:manage` を持つメンバーが、A を C に修正する | 変更後の通知先 C、B をこの順に返す。C は無効、B は有効。操作履歴に `form.recipients_changed` が一件残る |
| TC-form-changeRecipientAddress-02 | 削除済みフォーム F の `removed` の設定。通知先 A がある | `notification:manage` を持つメンバーが、A を C に修正する | A を C に置き換えた通知先を返す |
| TC-form-changeRecipientAddress-03 | フォーム F の `active` の設定。通知先 A がある | `notification:manage` を持つメンバーが、A を A に修正する | 現在の通知先を返す。`DuplicateRecipient` にしない。保存せず、`version` は変わらない。操作履歴は残らない |
| TC-form-changeRecipientAddress-05 | フォーム F の `active` の設定。通知先 A と B がある | `notification:manage` を持つメンバーが、A を B に修正する | `BusinessRuleError`（`DuplicateRecipient`）。通知先は変わらない |
| TC-form-changeRecipientAddress-06 | フォーム F の `active` の設定。他のメンバーが通知先 A を先に削除した | `notification:manage` を持つメンバーが、A を C に修正する | `NotFoundError`。対象の種類 `recipient` と、キー A を持つ。通知先は変わらない |
| TC-form-changeRecipientAddress-08 | フォーム F の `active` の設定。通知先 A がある | `notification:manage` を持たないメンバーが、A を C に修正する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `notification:manage`。通知先は変わらない。操作履歴は残らない |
