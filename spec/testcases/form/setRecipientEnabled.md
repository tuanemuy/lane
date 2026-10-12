# 通知先を有効・無効にする

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-setRecipientEnabled-01 | 受付中のフォーム F。有効な通知先 A と B がある | `notification:manage` を持つメンバーが、A を無効にする | A が無効、B が有効の通知先を返す。操作履歴に `form.recipients_changed` が一件残る |
| TC-form-setRecipientEnabled-02 | フォーム F。無効な通知先 A がある | `notification:manage` を持つメンバーが、A を有効にする | A が有効の通知先を返す |
| TC-form-setRecipientEnabled-04 | 受付中のフォーム F。有効な通知先は A だけ | `notification:manage` を持つメンバーが、A を無効にする | `BusinessRuleError`（`LastEnabledRecipient`）。通知先は変わらず、F は受付中のまま。操作履歴は残らない |
| TC-form-setRecipientEnabled-05 | 受付停止のフォーム F の `active` の設定と、削除済みフォーム G の `removed` の設定。どちらも、有効な通知先は A だけ | `notification:manage` を持つメンバーが、F と G のそれぞれで、A を無効にする | どちらも、A が無効の通知先を返す |
| TC-form-setRecipientEnabled-07 | フォーム F の `active` の設定。通知先 A が無い | `notification:manage` を持つメンバーが、A を無効にする | `NotFoundError`。対象の種類 `recipient` と、キー A を持つ。通知先は変わらない |
| TC-form-setRecipientEnabled-09 | フォーム F。有効な通知先 A と B がある | `notification:manage` を持たないメンバーが、A を無効にする | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `notification:manage`。通知先は変わらない。操作履歴は残らない |
