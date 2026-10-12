# 回答を取得する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-getSubmission-01 | 回答 S。単一選択の項目と、問い合わせ対象の項目に値を持つ。問い合わせ対象のコンテンツ P は、ごみ箱に無い | `submission:read` を持つメンバーが、S を取得する | 受付日時、`FormId`、受付時のフォーム（項目、選択肢、表示名称、順序）、入力内容、問い合わせ対象の送信時の情報（モデルのキー、コンテンツID、表示名称、一意な値）を返す。P の現在の置き場所（`active`）を、送信時の情報と分けて持つ |
| TC-form-getSubmission-02 | S の受付の後に、スキーマの反映で、フォームの項目と選択肢の表示名称を変え、項目を一つ削除した | `submission:read` を持つメンバーが、S を取得する | 受付時の項目、選択肢、表示名称、順序と、受付時の入力内容を返す |
| TC-form-getSubmission-03 | S の受付の後に、問い合わせ対象のコンテンツ P の公開を取り消し、表示名称を変えた | `submission:read` を持つメンバーが、S を取得する | 問い合わせ対象の送信時の情報は、受付時の値のまま。P の現在の置き場所は `active` |
| TC-form-getSubmission-04 | S の受付の後に、問い合わせ対象のコンテンツ P を完全削除した | `submission:read` を持つメンバーが、S を取得する | 問い合わせ対象の送信時の情報は、受付時の値のまま。P の現在の置き場所は `null` |
| TC-form-getSubmission-05 | 削除済みフォームの回答 S。S の受付の後、フォームの削除の前に、スキーマの反映でフォームの表示名称を変えた | `submission:read` を持つメンバーが、S を取得する | 受付時のフォームのキーと表示名称、受付時のフォーム、入力内容を返す。削除の時点の表示名称を返さない |
| TC-form-getSubmission-06 | 回答 S | `submission:read` だけを持つメンバーが、S を取得する | S の内容を返す。通知の状態を含まない |
| TC-form-getSubmission-07 | 回答 S の通知は送信失敗（未対処）。宛先 A は `accepted`、宛先 B は `failed`（`recipientRejected`） | `notification:manage` だけを持つメンバーが、S を取得する | 受付日時、受付時のフォームのキーと表示名称、通知の状態、通知の回、受付時の通知対象、宛先ごとの送信の状態と失敗の理由、送信結果（`failed`）、対処の状態（未対処）を返す。受付時のフォーム、入力内容、問い合わせ対象の情報を含まない |
| TC-form-getSubmission-08 | 回答 S の通知は送信失敗 | `submission:read` と `notification:manage` を持つメンバーが、S を取得する | 回答の内容と、通知の状態の両方を返す |
| TC-form-getSubmission-09 | 回答 S の通知は 1 回目の再試行の送信待ち（`attempt` は `retry` の `round` 1、状態は `waiting`） | `notification:manage` を持つメンバーが、S を取得する | 送信結果は未送信で、通知の回は再試行（`retry`）。対処の状態を持たない |
| TC-form-getSubmission-10 | 回答 S | `submission:read` も `notification:manage` も持たないメンバーが、S を取得する | `ForbiddenError`（`PERMISSION_MISSING`）。入力内容を返さない |
| TC-form-getSubmission-11 | S は、削除または自動整理で既に無い | `submission:read` を持つメンバーが、S を取得する | `NotFoundError` |
| TC-form-getSubmission-14 | S の通知 N は 1 回目の再試行の送信中（`sending`）。A の結果は `accepted`、送信中の宛先（`recipients`）は B | `notification:manage` を持つメンバーが、S を取得する | 宛先ごとの送信の状態は、A が受理を確認済み、B が送信中 |
| TC-form-getSubmission-15 | S の受付の後に、問い合わせ対象のコンテンツ P をごみ箱へ移した | `submission:read` を持つメンバーが、S を取得する | 問い合わせ対象の送信時の情報は、受付時の値のまま。P の現在の置き場所は `trashed` |
