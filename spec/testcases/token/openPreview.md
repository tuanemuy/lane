# プレビューを開く

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-token-openPreview-01 | 経路を `/articles/` と一意な項目 `slug` の値で定義したモデルの、保存済みのコンテンツ。`slug` は `hello`。プレビュー先の表示アプリの URL は `https://app.example.com`。ワークスペースのメンバー | そのコンテンツのプレビューを開く | `https://app.example.com/articles/hello?preview_code=` に引き換えの値を続けた URL を返す。URL はトークンの値を含まない。引き換えの値は、ワークスペースの ID を含む。`pending` のプレビューの引き換えを一つ作る。引き換えは開いたメンバーを持ち、期限は作成の `TokenLifetimePolicy.exchangeCodeMs` 後。トークンは発行されていない。コンテンツ、公開内容、メディアの公開状態は変わらない |
| TC-token-openPreview-02 | 一行目と同じコンテンツと設定。どの権限も持たない役割のメンバー | そのコンテンツのプレビューを開く | プレビュー先の URL を返す |
| TC-token-openPreview-03 | 一行目と同じコンテンツと設定 | 同じコンテンツのプレビューを二度開く | 引き換えの値が互いに異なる URL を返す。プレビューの引き換えを二つ作る |
| TC-token-openPreview-04 | プレビュー先を定義していないモデルのコンテンツ | プレビューを開く | 開けない理由 `path_not_defined` を返す。エラーにしない。プレビューの引き換えを作らない |
| TC-token-openPreview-05 | プレビュー先の表示アプリの URL が未設定 | プレビューを開く | 開けない理由 `base_url_not_set` を返す。プレビューの引き換えを作らない |
| TC-token-openPreview-06 | 経路に使う一意な値が未設定のコンテンツ | プレビューを開く | 開けない理由 `path_values_missing` と、未設定の項目を返す。プレビューの引き換えを作らない |
| TC-token-openPreview-07 | 最新の内容が、型を満たさない入力エラー（必須項目の欠落）のある下書きであるコンテンツ | プレビューを開く | 開けない理由 `draft_has_type_errors` と、Content が示すエラー箇所を返す。プレビューの引き換えを作らない |
| TC-token-openPreview-08 | 最新の内容が、文字数の違反だけのある下書きであるコンテンツ。経路に使う値と URL は設定済み | プレビューを開く | プレビュー先の URL を返す |
| TC-token-openPreview-09 | ワークスペースのメンバー。ごみ箱のコンテンツ C | 存在しないコンテンツID と、C のコンテンツID を、それぞれ指定してプレビューを開く | どちらも `NotFoundError`。プレビューの引き換えを作らない |
