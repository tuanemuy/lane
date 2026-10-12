# 紐付けを待つトークンの範囲を一覧する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-token-listPendingTokenScopes-01 | アプリ連携の権限を持つメンバー。初期構成で、範囲にモデルのキー `article` とフォームのキー `contact` を宣言したトークンがある。どちらのキーの対象も現行スキーマに無い | 一覧する | そのトークンの名称と、紐付けを待つモデルのキー `article`、フォームのキー `contact` を返す。出力はトークンの値を含まない |
| TC-token-listPendingTokenScopes-02 | アプリ連携の権限を持つメンバー。上のトークンで、スキーマの反映がモデル `article` を作成し、範囲の要素を紐付けた | 一覧する | そのトークンの、紐付けを待つフォームのキー `contact` だけを返す |
| TC-token-listPendingTokenScopes-03 | アプリ連携の権限を持つメンバー。範囲がワークスペース全体のトークンと、範囲の要素がすべて紐付け済みのトークンだけがある | 一覧する | 空の一覧を返す |
| TC-token-listPendingTokenScopes-04 | アプリ連携の権限を持たないメンバー | 一覧する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限（`integration:manage`）を示す |
