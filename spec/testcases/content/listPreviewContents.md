# プレビュー取得で一覧を取得する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-listPreviewContents-01 | モデル M に、未公開の U、公開中で下書きを持つ A、公開中で下書きを持たない B、ごみ箱の C がある。トークンは `preview_read` の許可を持つサーバー用トークンで、範囲に M を含む | トークンで、M の一覧を取得する | U の下書き、A の下書き、B の公開内容を返し、C を含めない。総件数は 3。コンテンツごとに、返した内容が下書きか公開内容かを返す |
| TC-content-listPreviewContents-02 | M の A は公開中で、公開内容の `category` は `news`、下書きの `category` は `event` | `preview_read` のトークンで、`category` が `event` に等しい、で絞り込んで取得する | A の下書きを返す |
| TC-content-listPreviewContents-03 | M に、入力エラーの無い下書きを持つ A と、下書きが必須項目の欠落を持つ E がある | `preview_read` のトークンで、M の一覧を取得する | エラーにしない。A を返し、E を結果と総件数に含めない。総件数は 1。含めなかった対象として、件数 1 と E のコンテンツID を返す |
| TC-content-listPreviewContents-04 | M の A の `category` は `news`。M の E は、下書きが必須項目の欠落を持ち、`category` は `event` | `preview_read` のトークンで、`category` が `news` に等しい、で絞り込んで取得する | A を返す。含めなかった対象として E を返す |
| TC-content-listPreviewContents-05 | M の A の最新の内容は、ごみ箱の T を必須の参照で参照する。M の B は、参照を持たない | `preview_read` のトークンで、M の一覧を取得する | `BusinessRuleError`（`DeliveryRejected`）を返す。参照元 A と参照先 T、原因（参照先がごみ箱にある）を添える。B も返さない |
| TC-content-listPreviewContents-06 | トークンは `preview_read` の許可を持たない | トークンで、M の一覧をプレビュー取得で取得する | `ForbiddenError`（`GRANT_MISSING`）を返す |
