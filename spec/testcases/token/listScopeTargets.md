# トークンの範囲に指定できる対象を一覧する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-token-listScopeTargets-01 | アプリ連携の権限だけを持つメンバー（スキーマ変更権限を持たない）。現行スキーマに、キー `article`・表示名称「記事」のモデル、キー `news`・表示名称「お知らせ」のモデル、キー `contact`・表示名称「お問い合わせ」のフォームが、この順に定義されている | 一覧する | モデルの列として `article`（「記事」）、`news`（「お知らせ」）をこの順に、フォームの列として `contact`（「お問い合わせ」）を返す |
| TC-token-listScopeTargets-02 | アプリ連携の権限を持つメンバー。現行スキーマに、表示名称を定義していない、キー `author` のモデルがある | 一覧する | そのモデルを、キー `author` と、表示名称 `author` で返す |
| TC-token-listScopeTargets-04 | アプリ連携の権限を持つメンバー。現行スキーマに、モデルもフォームも無い | 一覧する | モデルの列もフォームの列も空で返す |
| TC-token-listScopeTargets-05 | アプリ連携の権限を持たないメンバー（スキーマ変更権限を持つ） | 一覧する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限（`integration:manage`）を示す |
| TC-token-listScopeTargets-06 | アプリ連携の権限だけを持つメンバー（スキーマ変更権限も、コンテンツの権限も持たない） | 一覧する | 与えられない許可の列は、スキーマ反映（領域 `schema:change`）とコンテンツ管理（領域 `content:edit`、`content:publish`、`content:delete`、`media:publish`）を持つ。公開取得、プレビュー取得、フォーム送信は含まない |
