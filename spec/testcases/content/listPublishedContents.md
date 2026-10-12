# 公開コンテンツの一覧を取得する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-listPublishedContents-01 | モデル M に、公開中の A と B、未公開の U、ごみ箱の C がある。トークンは `public_read` の許可を持ち、範囲に M を含む | トークンで、M の一覧を取得する | A と B の公開内容を返し、U と C を含めない。総件数は 2 |
| TC-content-listPublishedContents-02 | M の公開中の A、B、D の公開内容で、単一選択の項目 `category` が、A と B は `news`、D は `event`。数値の項目 `order` が、A は 1、B は 5 | `public_read` のトークンで、`category` が `news` に等しい、かつ `order` が 3 以上、で絞り込んで取得する | B だけを返す。総件数は 1 |
| TC-content-listPublishedContents-03 | M の A は公開中で、公開内容の `category` は `news`、下書きの `category` は `event` | `public_read` のトークンで、`category` が `news` に等しい、で絞り込んで取得する | A の公開内容を返す |
| TC-content-listPublishedContents-04 | M の公開中の A、B、D の公開内容で、数値の項目 `order` が、A は 3、B は 1、D は 2 | `public_read` のトークンで、`order` の昇順、区切る件数 2、1 ページ目を指定して取得する | B、D の順に返す。総件数は 3 |
| TC-content-listPublishedContents-06 | M の公開中の A と B で、A の作成日時が B より古い | `public_read` のトークンで、並び替えを指定せずに取得する | B、A の順に返す |
| TC-content-listPublishedContents-07 | M の公開中の A の公開内容は、公開中の T をコンテンツ参照の項目 `author` で参照する | `public_read` のトークンで、`author` の展開と深さ 1 を指定して一覧を取得する | A とともに、T の公開内容を返す |
| TC-content-listPublishedContents-08 | M に、条件に合う公開内容が無い | `public_read` のトークンで、条件を指定して取得する | エラーにしない。空の一覧と総件数 0 を返す |
| TC-content-listPublishedContents-09 | M に、公開中のコンテンツが 2 件ある | `public_read` のトークンで、区切る件数 2、2 ページ目を指定して取得する | エラーにしない。空の一覧と総件数 2 を返す |
| TC-content-listPublishedContents-10 | 現行スキーマに、キー `gone` のモデルが無い | `public_read` のトークンで、`gone` の一覧を取得する | `NotFoundError` を返す。キー `gone` を示す |
| TC-content-listPublishedContents-11 | トークンは `public_read` の許可を持たない | トークンで、M の一覧を取得する | `ForbiddenError`（`GRANT_MISSING`）を返す |
| TC-content-listPublishedContents-12 | M の項目 `body` はリッチテキスト | `public_read` のトークンで、`body` で絞り込んで取得する | `ValidationError` を返す。項目と理由を添える |
| TC-content-listPublishedContents-13 | M に数値の項目 `order` がある | `public_read` のトークンで、`order` が「abc」に等しい、で絞り込んで取得する | `ValidationError` を返す。条件の位置、項目 `order`、理由（`valueKindMismatch`）を添える |
| TC-content-listPublishedContents-15 | M に、数値の項目 `order` がある | `public_read` のトークンで、`order` の条件を `ContentDelivery.limits.filters` より 1 個多く指定して取得する | `ValidationError` を返す。理由（`tooManyFilters`）と上限の値 `ContentDelivery.limits.filters` を添える |
