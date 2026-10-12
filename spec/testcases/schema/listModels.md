# モデルを一覧する（listModels）

断りが無い限り、実行者は、権限を一つも持たないメンバー。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-schema-listModels-01 | 現行スキーマは、順に、複数件モデル article（表示名称「記事」）、表示名称を持たない単一モデル settings、非推奨の複数件モデル legacy を持つ | 一覧する | 対象の `WorkspaceId` と、三つのモデルを、article、settings、legacy の順に返す。article は、表示名称「記事」と `collection`、settings は、表示名称 `settings` と `single`、legacy は、非推奨であることを持つ |
| TC-schema-listModels-02 | スキーマを一度も反映していない | 一覧する | エラーにしない。空の一覧を返す |
