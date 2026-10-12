# 回答を一覧・検索する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-searchSubmissions-01 | フォーム F の回答 S1、S2、S3 を、この順に受け付けた。現行スキーマの F の一覧表示用の項目は `name` | `submission:read` だけを持つメンバーが、条件なしで一覧する | S3、S2、S1 の順に返し、総件数は 3。回答ごとに `SubmissionId`、受付日時、`FormId`（F）、受付時のフォームのキーと表示名称、`name` の値を持つ。通知の送信結果と対処の状態を含まない |
| TC-form-searchSubmissions-02 | フォーム F の回答 S1 の通知は送信成功、S2 の通知は送信失敗（未対処） | `submission:read` と `notification:manage` を持つメンバーが一覧する | 回答ごとに、一覧表示用の項目の値と、通知の送信結果を返す。S1 は `succeeded`、S2 は `failed` で未対処 |
| TC-form-searchSubmissions-03 | フォーム F の回答 S1 の通知は送信失敗（未対処）。S1 は一覧表示用の項目 `name` に値を持つ | `notification:manage` だけを持つメンバーが一覧する | S1 を、受付日時、受付時のフォームのキーと表示名称、送信結果（`failed`）、対処の状態（未対処）とともに返す。`name` の値と入力内容を含まない |
| TC-form-searchSubmissions-04 | フォーム F の回答 S1 | `submission:delete` だけを持つメンバーが一覧する | S1 を、一覧表示用の項目の値とともに返す |
| TC-form-searchSubmissions-05 | フォーム F の回答 S1 と、フォーム G の回答 S2 | `submission:read` を持つメンバーが、F の `FormId` で絞り込む | S1 だけを返し、総件数は 1 |
| TC-form-searchSubmissions-06 | 削除済みフォーム F の回答 S1 と、現行のフォーム G の回答 S2 | `submission:read` を持つメンバーが、F の `FormId` で絞り込む | S1 だけを返す。S1 は受付時のフォームのキーと表示名称を持ち、一覧表示用の項目の値は空 |
| TC-form-searchSubmissions-07 | 昨日受け付けた回答 S1 と、10 日前に受け付けた回答 S2 | `submission:read` を持つメンバーが、S1 の受付日時だけを含む受付期間で絞り込む | S1 だけを返す |
| TC-form-searchSubmissions-08 | 回答 S1、S2、S3 を、この順に受け付けた。S1 の通知は最初の通知の送信待ち、S2 の通知は 1 回目の再試行の送信中（`attempt` は `retry` の `round` 1、状態は `sending`）、S3 の通知は送信失敗 | `notification:manage` を持つメンバーが、送信結果を未送信にして絞り込み、続けて送信失敗にして絞り込む | 未送信は S2、S1 の順に返す。S1 の内訳は、送信待ちで最初の通知。S2 の内訳は、送信中で再試行。送信失敗は S3 だけを返し、S2 を含まない |
| TC-form-searchSubmissions-10 | 通知が送信失敗で未対処の回答 S1、送信失敗で対処済みの回答 S2、送信成功の回答 S3 | `notification:manage` を持つメンバーが、送信失敗の未対処で絞り込み、続けて送信失敗の対処済みで絞り込む | 未対処は S1 だけ、対処済みは S2 だけを返す。S3 はどちらにも現れない |
| TC-form-searchSubmissions-11 | 現行スキーマの F の一覧表示用の項目は `name`。S1 の `name` の値は検索の文字列を含み、S2 の `name` の値は含まない | `submission:read` を持つメンバーが、その文字列で検索する | S1 だけを返す |
| TC-form-searchSubmissions-12 | 現行スキーマの F の一覧表示用の項目は `fullName`。S1 は、受付時のフォームに `fullName` を持たない | `submission:read` を持つメンバーが、条件なしで一覧し、続けて S1 の入力内容に含まれる文字列で検索する | 一覧は S1 を返し、一覧表示用の項目の値は空。検索は S1 を返さない |
| TC-form-searchSubmissions-13 | フォーム F の回答 S1 | `submission:read` を持つメンバーが、開始が終了より後の受付期間で絞り込む | `ValidationError`（`InvalidReceivedRange`） |
| TC-form-searchSubmissions-14 | 通知が送信失敗の回答 S1 | `submission:read` だけを持つメンバーが、送信結果を送信失敗にして絞り込む | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `notification:manage` |
| TC-form-searchSubmissions-15 | フォーム F の回答 S1 | `notification:manage` だけを持つメンバーが、文字列で検索する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `submission:read` |
| TC-form-searchSubmissions-16 | 条件に該当する回答が無い | `submission:read` を持つメンバーが一覧する | エラーにしない。空の列と総件数 0 を返す |
| TC-form-searchSubmissions-18 | フォーム F の回答 S1 | `submission:read` も `notification:manage` も持たないメンバーが、一覧する | `ForbiddenError`（`PERMISSION_MISSING`） |
| TC-form-searchSubmissions-20 | 現行スキーマの F の一覧表示用の項目は `age`。S1 の受付時のフォームの `age` は数値の項目 | `submission:read` を持つメンバーが、条件なしで一覧し、続けて S1 の `age` の値の文字列で検索する | 一覧は S1 を返し、`age` の値は、受付時の値の文字列。検索は S1 を返さない |
