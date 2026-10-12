# 回答の対象のフォームを一覧する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-listSubmissionForms-01 | 回答の無い現行のフォーム F1、回答のある現行のフォーム F2、回答が残る削除済みフォーム F3、回答が残らない削除済みフォーム F4 | `submission:read` を持つメンバーが一覧する | F1、F2、F3 を、`FormId`、キー、表示名称、削除済みかどうかとともに返す。F3 は削除済みで、キーと表示名称は削除の時点の値（回答の受付時の値ではない）。F4 を含まない |
| TC-form-listSubmissionForms-02 | 現行のフォーム F1 と、紐付け待ちの設定 P | `submission:read` を持つメンバーが一覧する | F1 を返す。P を含まない |
| TC-form-listSubmissionForms-03 | 現行のフォーム F1 | `notification:manage` だけを持つメンバーが一覧する | F1 を返す |
| TC-form-listSubmissionForms-04 | 現行のフォーム F1 | `intake:manage` だけを持つメンバーが一覧する | `ForbiddenError`（`PERMISSION_MISSING`） |
