# プレビュー取得で単件を取得する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-getPreviewContent-01 | モデル M の A は公開中で、下書きを持つ。トークンは `preview_read` の許可を持つサーバー用トークンで、範囲に M を含む | トークンで、M のキーと A のコンテンツID を指定して取得する | A の下書きの値を返す。返した内容が下書きであることと、下書きの更新日時を返す |
| TC-content-getPreviewContent-02 | A は公開中で、下書きを持たない | `preview_read` のトークンで、A を取得する | A の公開内容の値を返す。返した内容が公開内容であることを返す |
| TC-content-getPreviewContent-03 | U は未公開 | `preview_read` のトークンで、U を取得する | U の下書きの値を返す |
| TC-content-getPreviewContent-04 | A の下書きは、文字数の制約の違反だけを持つ | `preview_read` のトークンで、A を取得する | A の下書きの値を返し、制約違反があることを返す |
| TC-content-getPreviewContent-05 | A は公開中で、下書きは必須項目の欠落を持つ | `preview_read` のトークンで、A を取得する | `BusinessRuleError`（`DeliveryRejected`）を返す。入力エラーの位置と理由を添える。公開内容を返さない |
| TC-content-getPreviewContent-06 | M の A の下書きの一意な項目 `slug` は「x」。A の公開内容の `slug` は「old」 | `preview_read` のトークンで、M のキー、`slug`、値「x」を指定して取得する | A の下書きの値を返す |
| TC-content-getPreviewContent-07 | M の A と B の最新の内容の `slug` が、どちらも「x」 | `preview_read` のトークンで、M のキー、`slug`、値「x」を指定して取得する | `BusinessRuleError`（`DeliveryRejected`）を返す。A と B のコンテンツID を添える。どちらの内容も返さない |
| TC-content-getPreviewContent-08 | A の最新の内容は、T をコンテンツ参照の項目 `author` で参照する。T は公開中で、下書きを持つ | `preview_read` のトークンで、A を、`author` の展開と深さ 1 を指定して取得する | T の下書きの値を展開して返す |
| TC-content-getPreviewContent-09 | A の最新の内容は、未公開の T を必須の参照で参照する。T の下書きは、型を満たさない入力エラーを持たない | `preview_read` のトークンで、A を取得する | T への参照を返す |
| TC-content-getPreviewContent-10 | A の最新の内容は、T を任意の参照の項目 `author` で参照する。T の下書きは必須項目の欠落を持つ | `preview_read` のトークンで、A を、展開を指定せずに取得する | `BusinessRuleError`（`DeliveryRejected`）を返す。参照元 A と参照先 T、原因（参照先の最新の内容が、型を満たさない入力エラーを持つ）を添える。T への参照を未設定として返さない |
| TC-content-getPreviewContent-11 | A の最新の内容は、ごみ箱の T を必須の参照で参照する | `preview_read` のトークンで、A を取得する | `BusinessRuleError`（`DeliveryRejected`）を返す。参照元 A と参照先 T、原因（参照先がごみ箱にある）を添える |
| TC-content-getPreviewContent-12 | A の最新の内容は、ごみ箱の T を任意の参照で参照する | `preview_read` のトークンで、A を取得する | エラーにしない。T への参照を未設定として返す |
| TC-content-getPreviewContent-13 | A の下書きは、非公開のメディア Md を使う | `preview_read` のトークンで、A を取得する | Md を、期限付きの配信先で返す。Md は非公開のまま |
| TC-content-getPreviewContent-14 | C はごみ箱にある | `preview_read` のトークンで、C を取得する | `NotFoundError` を返す |
| TC-content-getPreviewContent-16 | トークンは `public_read` の許可を持ち、`preview_read` の許可を持たない | トークンで、A をプレビュー取得で取得する | `ForbiddenError`（`GRANT_MISSING`）を返す |
| TC-content-getPreviewContent-20 | A の下書きの `title` は「x」。スキーマの反映が適用中で、その準備領域に、`title` を「y」にした A の変換結果がある | `preview_read` のサーバー用トークンで、A を取得する | 反映前の構造で、`title` が「x」の A の下書きを返す |
