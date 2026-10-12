# 通知定義を入力例で描画する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-previewNotificationTemplate-01 | 現行のフォーム F。項目 `name` が入力例を持つ | `notification:manage` を持つメンバーが、件名と本文で `name` を参照する通知定義を描画する | 件名と本文の参照の位置に `name` の入力例を持つ `RenderedNotification` を返す。不整合のある参照は空。F の通知定義は変わらない。メールを送らない |
| TC-form-previewNotificationTemplate-02 | 現行のフォーム F。項目 `note` が入力例を持たない | `notification:manage` を持つメンバーが、本文で `note` を参照する通知定義を描画する | `note` を未入力として描画し、参照の位置は空文字列になる |
| TC-form-previewNotificationTemplate-03 | 現行のフォーム F。メールアドレス形式の項目 `email` が、メールアドレスの入力例を持つ | `notification:manage` を持つメンバーが、返信先に `email` を指定する通知定義を描画する | 返信先は `email` の入力例のメールアドレス |
| TC-form-previewNotificationTemplate-04 | 現行のフォーム F に項目 `company` が無い | `notification:manage` を持つメンバーが、本文で `company` を参照する通知定義を描画する | 本文は、値の代わりに `[不整合な参照: company]` を持つ。不整合のある参照に `company` と理由 `fieldRemoved` を持つ |
| TC-form-previewNotificationTemplate-06 | 削除済みフォーム F の `removed` の設定 | `notification:manage` を持つメンバーが、F の通知定義を描画する | `BusinessRuleError`（`FormRemoved`） |
| TC-form-previewNotificationTemplate-08 | 現行のフォーム F | `notification:manage` を持たないメンバーが、F の通知定義を描画する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `notification:manage` |
