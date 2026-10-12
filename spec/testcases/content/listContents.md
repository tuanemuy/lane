# コンテンツを一覧・検索する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-listContents-01 | モデル M に、未公開の A、公開中で下書きを持つ B、ごみ箱の C がある | メンバーが、M のキーを指定して一覧する | A と B を返し、C を含めない。総件数は 2。コンテンツごとに、コンテンツID、`version`、公開状態、下書きの有無、作成日時、最新の内容の更新日時を返す。B の一覧表示用の項目の値は下書きのもの |
| TC-content-listContents-02 | M の一覧表示用のテキスト項目の値が、A は「春の特集」、B は「夏の案内」 | メンバーが、検索語「特集」で一覧する | A だけを返す。総件数は 1 |
| TC-content-listContents-03 | M に A と B がある | メンバーが、B のコンテンツID を検索語にして一覧する | B だけを返す |
| TC-content-listContents-04 | M に、下書きが入力エラーを持つ A と、入力エラーを持たない B がある | メンバーが、「下書きの入力エラーあり」で絞り込む | A だけを返す |
| TC-content-listContents-05 | M に、公開内容が現行の制約を満たさない A と、満たす B がある | メンバーが、「現行の制約を満たさない公開内容あり」で絞り込む | A だけを返す |
| TC-content-listContents-06 | M に、未公開の A、公開中で下書きを持つ B、公開中で下書きを持たない C がある | メンバーが、公開状態「公開中」と下書き「あり」で絞り込む | B だけを返す |
| TC-content-listContents-07 | M の A、B、C の最新の内容で、単一選択の項目 `category` が、A と B は `news`、C は `event`。数値の項目 `order` が、A は 1、B は 5 | メンバーが、`category` が `news` に等しい、かつ `order` が 3 以上、で絞り込む | B だけを返す。総件数は 1 |
| TC-content-listContents-08 | M の A、B のテキストの項目 `title` が、A は「b」、B は「a」 | メンバーが、`title` の昇順で一覧する | B、A の順に返す |
| TC-content-listContents-09 | M は非推奨のモデルで、コンテンツ A がある | メンバーが、M を一覧する | A を返し、M が非推奨であることを返す |
| TC-content-listContents-10 | M に日付の項目 `publishedOn` がある | メンバーが、`publishedOn` に日付でない値を指定して絞り込む | `ValidationError`を返す。該当する条件と理由を添える |
| TC-content-listContents-11 | M に、条件に合うコンテンツが無い | メンバーが、条件を指定して一覧する | エラーにしない。空の一覧と総件数 0 を返す |
| TC-content-listContents-12 | 現行スキーマに、キー `gone` のモデルが無い | メンバーが、`gone` を指定して一覧する | `NotFoundError` を返す。キー `gone` を示す |
| TC-content-listContents-13 | M に、未公開の A、型を満たさない入力エラーを持つ下書きの B、公開中の C、ごみ箱の D がある。トークンは `content_manage` の許可を持ち、範囲に M を含む | トークンで、M を一覧する | A、B、C を返し、D を含めない |
| TC-content-listContents-15 | トークンは `content_manage` の許可を持たない | トークンで、M を一覧する | `ForbiddenError`（`GRANT_MISSING`）を返す |
