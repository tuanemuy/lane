# フォームの運用設定を一覧する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-listFormSettings-01 | 現行のフォーム F1、回答が残る削除済みフォーム F2、回答が残らない削除済みフォーム F3 | `notification:manage` と `intake:manage` を持つメンバーが一覧する | F1 と F2 を返し、F3 を含まない。フォームごとに `FormId`、キー、表示名称、状態を持つ。F2 は状態が削除済みで、参照の不整合の有無は `null`、キーと表示名称は削除の時点の値 |
| TC-form-listFormSettings-02 | 受付中の現行のフォーム F1 | `notification:manage` だけを持つメンバーが一覧する | F1 に、参照の不整合の有無を加えて返す。受付状態を含まない |
| TC-form-listFormSettings-03 | 受付中の現行のフォーム F1 | `intake:manage` だけを持つメンバーが一覧する | F1 に、受付状態（受付中）と、参照の不整合の有無を加えて返す |
| TC-form-listFormSettings-04 | 現行のフォーム F1 の通知定義が、現行のフォームに無い項目を参照する | `notification:manage` を持つメンバーが一覧する | F1 の参照の不整合の有無は真 |
| TC-form-listFormSettings-05 | キー `contact` の紐付け待ちの設定 P。通知先、通知定義、受付状態、回答の保持方針の宣言を持つ | `notification:manage` だけを持つメンバーが一覧する | P を、キー `contact` と、宣言した通知先と通知定義（件名と本文は記法の文字列）とともに返す。受付状態と回答の保持方針を含まない |
| TC-form-listFormSettings-06 | キー `contact` の紐付け待ちの設定 P。通知先、通知定義、受付状態、回答の保持方針の宣言を持つ | `intake:manage` だけを持つメンバーが一覧する | P を、キー `contact` と、宣言した受付状態と回答の保持方針とともに返す。通知先と通知定義を含まない |
| TC-form-listFormSettings-07 | 現行のフォームがある | `notification:manage` も `intake:manage` も持たないメンバーが一覧する | `ForbiddenError`（`PERMISSION_MISSING`） |
