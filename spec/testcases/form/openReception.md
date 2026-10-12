# フォームの受付を有効にする

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-openReception-01 | 受付停止のフォーム F。有効な通知先が 1 件。送信接続設定が登録済み | `intake:manage` を持つメンバーが、F の受付を有効にする | 受付中を返す。F は受付中になる |
| TC-form-openReception-02 | 受付停止のフォーム F。有効な通知先が 1 件。送信接続設定が登録済み。通知定義に参照の不整合が残る | `intake:manage` を持つメンバーが、F の受付を有効にする | 受付中を返す。F は受付中になる |
| TC-form-openReception-03 | 受付停止のフォーム F。有効な通知先が無い（通知先は無効な 1 件だけ）。送信接続設定が未登録 | `intake:manage` を持つメンバーが、F の受付を有効にする | `BusinessRuleError`（`ReceptionNotReady`）。足りないものの一覧は `enabledRecipient`、`mailConnection` の順。F は受付停止のまま |
| TC-form-openReception-07 | 削除済みフォーム F の `removed` の設定 | `intake:manage` を持つメンバーが、F の受付を有効にする | `BusinessRuleError`（`FormRemoved`） |
| TC-form-openReception-09 | 受付停止のフォーム F。有効な通知先が 1 件。送信接続設定が登録済み | `notification:manage` だけを持つメンバーが、F の受付を有効にする | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `intake:manage`。F は受付停止のまま |
| TC-form-openReception-10 | 受付中のフォーム F（ほかのメンバーが先に受付を有効にした場合を含む） | `intake:manage` を持つメンバーが、F の受付を有効にする | 受付中を返す。保存せず、F の設定の `version` は変わらない |
| TC-form-openReception-11 | 受付停止のフォーム F。有効な通知先は A の 1 件。送信接続設定が登録済み。メンバーが有効化の確認を示した後、ほかのメンバーが A を無効にした | `intake:manage` を持つメンバーが、確認した内容で F の受付を有効にする | `BusinessRuleError`（`ReceptionNotReady`）。足りないものの一覧は `enabledRecipient`。F は受付停止のまま |
