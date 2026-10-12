# フォームの運用設定を取得する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-getFormSettings-01 | 現行のフォーム F の `active` の設定。有効な通知先、通知定義、保持方針を持つ。送信接続設定が登録済み | `notification:manage` と `intake:manage` を持つメンバーが、F を指定して取得する | `FormId`、キー、表示名称、状態、通知先（メールアドレスと有効・無効）、通知定義（件名と本文は記法の文字列）、不整合のある参照（空）、受付状態、保持方針、有効な通知先の件数、受付を有効にするのに足りないもの（空）、不整合が無いことを返す |
| TC-form-getFormSettings-02 | 現行のフォーム F の `active` の設定 | `notification:manage` だけを持つメンバーが取得する | キー、表示名称、通知先、通知定義、不整合のある参照と理由を返す。受付状態と保持方針を含まない |
| TC-form-getFormSettings-03 | 現行のフォーム F の `active` の設定。有効な通知先が 1 件。送信接続設定が登録済み | `intake:manage` だけを持つメンバーが取得する | 受付状態、保持方針、有効な通知先の件数 1、受付を有効にするのに足りないもの（空）、参照の不整合の有無を返す。通知先のメールアドレスと送信接続設定の内容を含まない |
| TC-form-getFormSettings-04 | 現行のフォーム F は、テキストの項目 `name`、メールアドレスの形式を求めるテキストの項目 `email`、問い合わせ対象の項目 `product` を持つ | `notification:manage` を持つメンバーが取得する | 現行のフォームの項目ごとに、キー、表示名称、参照に使える用途を返す。`name` は `field` と `ifAnswered`、`email` は `field`、`ifAnswered`、`replyTo`、`product` は `inquiry` と `ifAnswered` |
| TC-form-getFormSettings-05 | 受付停止のフォーム F の `active` の設定。有効な通知先が無い。送信接続設定が未登録 | `intake:manage` だけを持つメンバーが取得する | 受付を有効にするのに足りないものは、`enabledRecipient`、`mailConnection` の順 |
| TC-form-getFormSettings-06 | F の通知定義の本文が、現行のフォームに無い項目 `company` を参照する | `notification:manage` を持つメンバーが取得する | 不整合のある参照に、`company` と理由 `fieldRemoved` を持つ |
| TC-form-getFormSettings-07 | 削除済みフォーム F の `removed` の設定。通知先を持つ | `notification:manage` を持つメンバーが取得する | 状態が削除済みであることと、通知先を返す。通知定義、受付状態、保持方針を含まない |
| TC-form-getFormSettings-08 | その `FormId` の設定が無い | `notification:manage` を持つメンバーが取得する | `NotFoundError` |
| TC-form-getFormSettings-10 | 現行のフォーム F の `active` の設定 | `notification:manage` も `intake:manage` も持たないメンバーが取得する | `ForbiddenError`（`PERMISSION_MISSING`） |
