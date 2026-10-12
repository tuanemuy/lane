# フォームの受付を停止する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-closeReception-01 | 受付中のフォーム F | `intake:manage` を持つメンバーが、F の受付を停止する | 受付停止を返す。F は受付停止になる |
| TC-form-closeReception-02 | 受付中のフォーム F。F の回答 S の通知は最初の通知の送信待ち（`attempt` は `initial`、状態は `waiting`） | `intake:manage` を持つメンバーが、F の受付を停止する | S は残り、S の通知は `waiting` のまま変わらない |
| TC-form-closeReception-04 | 削除済みフォーム F の `removed` の設定 | `intake:manage` を持つメンバーが、F の受付を停止する | `BusinessRuleError`（`FormRemoved`） |
| TC-form-closeReception-06 | 受付中のフォーム F | `intake:manage` を持たないメンバーが、F の受付を停止する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `intake:manage`。F は受付中のまま |
| TC-form-closeReception-07 | 受付停止のフォーム F（ほかのメンバーが先に受付を停止した場合を含む） | `intake:manage` を持つメンバーが、F の受付を停止する | 受付停止を返す。保存せず、F の設定の `version` は変わらない |
