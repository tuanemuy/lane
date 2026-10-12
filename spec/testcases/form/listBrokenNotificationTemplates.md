# 参照の不整合が残る通知定義を一覧する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-listBrokenNotificationTemplates-01 | 現行のフォーム F1 の通知定義は、現行のフォームに無い項目を参照する。現行のフォーム F2 の通知定義に不整合は無い。削除済みフォーム F3 と、紐付け待ちの設定 P がある | `notification:manage` を持つメンバーが一覧する | F1 だけを、`FormId`、キー、表示名称とともに返す。F2、F3、P を含まない |
| TC-form-listBrokenNotificationTemplates-02 | F1 の通知定義の不整合を修正して保存した。ほかに不整合が残るフォームは無い | `notification:manage` を持つメンバーが一覧する | 空の列を返す |
| TC-form-listBrokenNotificationTemplates-03 | F1 の通知定義に不整合が残る | `intake:manage` だけを持つメンバーが、一覧する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `notification:manage` |
