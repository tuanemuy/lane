# 通知先を削除する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-removeRecipient-01 | 受付中のフォーム F。有効な通知先 A と B がある | `notification:manage` を持つメンバーが、A を削除する | B だけの通知先を返す。操作履歴に `form.recipients_changed` が一件残る |
| TC-form-removeRecipient-02 | 受付中のフォーム F。有効な通知先 A と、無効な通知先 B がある | `notification:manage` を持つメンバーが、A を削除する | `BusinessRuleError`（`LastEnabledRecipient`）。通知先は変わらず、F は受付中のまま。操作履歴は残らない |
| TC-form-removeRecipient-03 | 受付停止のフォーム F の `active` の設定と、削除済みフォーム G の `removed` の設定。どちらも、通知先は有効な A だけ | `notification:manage` を持つメンバーが、F と G のそれぞれで、A を削除する | どちらも、空の通知先を返す |
| TC-form-removeRecipient-05 | フォーム F の `active` の設定。他のメンバーが通知先 A を先に削除した | `notification:manage` を持つメンバーが、A を削除する | `NotFoundError`。対象の種類 `recipient` と、キー A を持つ。通知先は変わらない |
| TC-form-removeRecipient-07 | フォーム F。通知先 A と B がある | `notification:manage` を持たないメンバーが、A を削除する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `notification:manage`。通知先は変わらない。操作履歴は残らない |
