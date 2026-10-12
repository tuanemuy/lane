# フォームの送信を受け付ける

断りが無い限り、次を前提にする。

- F は現行スキーマにある受付中のフォーム。有効な通知先 A と B、無効な通知先 C、通知定義 T、保持方針 R を持つ
- トークン K は、`form_submit` の許可を持ち、範囲に F を含む。失効も期限切れもしていない
- ワークスペースは利用できる
- 送信は、入口が K から作った `Actor`（`appToken`）で、F のキーと、F の現行の定義を満たす入力内容を送る。項目の値のほかの指定を持たない（`conditionKeys` は空）。トークンの値、利用元、トークンの利用量の制限は、入口の `authenticateToken` が確かめる（`spec/testcases/token/`）

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-acceptSubmission-01 | 共通の前提 | アプリが F へ送信する | `accepted` を返す。回答が一件残る。回答の `form` は F の現行の定義から写した受付時のフォーム（キー、表示名称、項目、選択肢、順序）、`template` は T、`retention` は R、`notifiedTo` は A と B。その回答の通知が一件残り、`attempt` は `initial`、状態は `waiting`、宛先ごとの結果（`deliveries`）は空。`form.submission_accepted` を一件記録し、ペイロードは `submissionId` だけを持つ。消費者が受けるイベントの `workspaceId` は、送信先のワークスペース。`accepted` を返すまでにメールを送らない |
| TC-form-acceptSubmission-02 | F の問い合わせ対象の項目が、モデル `product` を許可する。`product` の公開中のコンテンツ P がある | アプリが、問い合わせ対象に P のコンテンツID を指定して F へ送信する | `accepted` を返す。回答は、その項目の問い合わせ対象の送信時の情報として、P のモデルのキー、コンテンツID、表示名称、一意な値を持つ |
| TC-form-acceptSubmission-03 | 共通の前提 | アプリが、同じ入力内容で F へ二度送信する | 二度とも `accepted` を返す。別の `SubmissionId` の回答と通知が二組残り、`form.submission_accepted` を二件記録する |
| TC-form-acceptSubmission-04 | スキーマの反映の適用中 | アプリが F へ送信する | `accepted` を返す。回答の `form` は、現行の定義から写した値 |
| TC-form-acceptSubmission-05 | 共通の前提 | アプリが、F の現行の識別子と一致する識別子を添えて F へ送信し、続けて、識別子を添えずに F へ送信する | 一つ目は `accepted` と、添えた識別子が一致することを返す。二つ目は `accepted` と、空の識別子の一致を返す |
| TC-form-acceptSubmission-06 | 共通の前提 | アプリが、F の現行の識別子と一致しない識別子を添えて、現行の定義を満たす入力内容を F へ送信する | `accepted` と、添えた識別子が一致しないことを返す |
| TC-form-acceptSubmission-08 | 現行スキーマにキー `contact-old` のフォームが無い。スキーマの反映で削除した `contact-old` のフォームの `removed` の設定が残る | アプリが、キー `contact-old` へ送信する | `rejected`（`formNotFound`）を返し、キー `contact-old` を添える。回答も通知もイベントも残らない |
| TC-form-acceptSubmission-10 | トークン K2 は `form_submit` の許可を持ち、範囲に F だけを含む。現行スキーマにキー `survey` のフォームが無い | アプリが、K2 でキー `survey` へ送信する | `rejected`（`formNotFound`）を返す。`ForbiddenError` にしない |
| TC-form-acceptSubmission-11 | K は、入口が `Actor` を作った後に失効した。現行スキーマにキー `survey` のフォームが無い | アプリが、K でキー `survey` へ送信する | `UnauthenticatedError`（`TOKEN_REVOKED`）。`formNotFound` にしない |
| TC-form-acceptSubmission-12 | F は受付停止 | アプリが F へ送信する | `rejected`（`closed`）を返す。回答も通知もイベントも残らない |
| TC-form-acceptSubmission-13 | F に必須の項目 `name` がある | アプリが、`name` を入力せずに F へ送信する | `rejected`（`invalid`）を返す。問題は、`name` の理由（`missingRequired`）の一件。回答も通知もイベントも残らない |
| TC-form-acceptSubmission-14 | F に、メールアドレスの形式を求めるテキストの項目 `email` がある | アプリが、`email` に、形式に合わない文字列 v を入力して F へ送信する | `rejected`（`invalid`）を返す。`email` の理由（形式の違反）を添える。受付失敗は、項目のキーと理由だけを持ち、v を持たない。受付失敗を JSON と文字列へ変換した結果に、v は現れない |
| TC-form-acceptSubmission-15 | F に、真であることを求める（`mustBeTrue`）真偽値の必須の項目 `agree` がある | アプリが、`agree` に偽を入力して F へ送信し、続けて、`agree` を入力せずに F へ送信する | どちらも `rejected`（`invalid`）を返す。偽の送信の `agree` の理由は `mustBeTrue`、未入力の送信の `agree` の理由は `missingRequired`。回答も通知もイベントも残らない |
| TC-form-acceptSubmission-16 | F に、項目 `email` と、`email` との一致を求める（`mustMatch`）同じ型の項目 `emailConfirm` がある | アプリが、二つの項目に違う値を入力して F へ送信する | `rejected`（`invalid`）を返す。`emailConfirm` の理由は `mustMatch` で、相手の項目 `email` を添える。回答も通知もイベントも残らない |
| TC-form-acceptSubmission-17 | F に、選択肢の複数選択の項目 `topics` がある | アプリが、`topics` に数値の列を入力して F へ送信する | `rejected`（`invalid`）を返す。`topics` の理由は、値の種類が合わないこと（`typeError`）。エラーの形にしない。回答も通知もイベントも残らない |
| TC-form-acceptSubmission-18 | F にキー `extra` の項目が無い | アプリが、`extra` の値を含めて F へ送信する | `rejected`（`invalid`）を返す。問題は、定義されていない項目のキー `extra`（`undefinedKey`）の一件。回答も通知もイベントも残らない |
| TC-form-acceptSubmission-19 | 共通の前提 | アプリが、現行の定義を満たす項目の値に加えて、通知先の指定（`conditionKeys` は `recipients`）を含めて F へ送信する | `rejected`（`invalid`）を返す。問題は、受付条件のキー `recipients`（`conditionKey`）の一件。回答も通知もイベントも残らない。A と B のほかへ通知しない |
| TC-form-acceptSubmission-20 | F は受付停止 | アプリが、受付状態の指定（`conditionKeys` は `reception`）を含めて F へ送信する | `rejected`（`closed`）を返す。送信は、受付状態を変えない |
| TC-form-acceptSubmission-22 | F の問い合わせ対象の項目が、モデル `product` を許可する。`product` の未公開のコンテンツ P がある | アプリが、問い合わせ対象に P のコンテンツID を指定して F へ送信する | `rejected`（`invalid`）を返す。その項目の理由は `inquiryTarget`（`unavailable`）。回答も通知もイベントも残らない |
| TC-form-acceptSubmission-23 | F の問い合わせ対象の項目が、モデル `product` を許可する。モデル `article` の公開中のコンテンツ Q がある | アプリが、問い合わせ対象に Q のコンテンツID を指定して F へ送信する | `rejected`（`invalid`）を返す。その項目の理由は `inquiryTarget`（`modelNotAllowed`）。回答も通知もイベントも残らない |
| TC-form-acceptSubmission-24 | F が、フォームの送信制限に達している | アプリが、必須の項目を入力せずに F へ送信する | `RateLimitedError`（`SUBMISSIONS_LIMITED`）。フォームの送信制限の `retryAt` を添える。入力不備を返さない。受付結果にしない。回答も通知もイベントも残らない |
| TC-form-acceptSubmission-25 | 現行スキーマを読んだ後、受付の確定までに、スキーマの反映が確定する（F を削除する反映を含む） | アプリが F へ送信する | `ConflictError`（`SCHEMA_CHANGED`）。受付結果にしない。回答も通知もイベントも残らない。送信をやり直さない。フォームの送信制限の利用の記録は残る |
| TC-form-acceptSubmission-27 | 回答の保存が `SystemError` で失敗する | アプリが F へ送信する | その `SystemError` を返す。受付結果にしない。エラーは、入力内容を持たない。回答も通知もイベントも残らない。通知しない |
| TC-form-acceptSubmission-28 | 送信の受け付けを始めた後、受付の UnitOfWork がトークンを読む前に、K の失効が確定した | アプリが、K で F へ送信する | `UnauthenticatedError`（`TOKEN_REVOKED`）。回答も通知もイベントも残らない |
| TC-form-acceptSubmission-29 | トークン K3 は `form_submit` の許可を持たない | アプリが、K3 で F へ送信する | `ForbiddenError`（`GRANT_MISSING`）。回答も通知もイベントも残らない |
| TC-form-acceptSubmission-30 | トークン K3 は `form_submit` の許可を持たない。K3 で、F へ `SubmissionRatePolicy.limit.max` 回送信し、すべて `ForbiddenError` で拒否された | 続けて、アプリが K で F へ送信する | `accepted` を返す（K3 の送信は、F の送信制限に数えられていない） |
| TC-form-acceptSubmission-33 | メンバーに紐付くトークン K5 は `form_submit` の許可を持つ | スクリプトが、K5 の `Actor`（`memberToken`）で F へ送信する | `accepted` を返す |
