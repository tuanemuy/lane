# 一件のコンテンツを移行で変換する（`ContentMigration.migrateContent`）

ユースケースを通らない、ドメインの純粋な関数の代表ケース。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-ContentMigration.migrateContent-01 | 反映の計画が、モデル `m` を削除する。コンテンツ D は `m` に属する。反映後も残るモデルのコンテンツ A は未公開で、下書きの本文が、テキスト「詳細」に付けた D へのリンクを持つ | A を `migrateContent` に渡す | `kept` の報告と、変換した A を返す。A の下書きの本文は D へのリンクを持たず、テキスト「詳細」は残る。報告の `blockers` は空。報告の `losses` は、`m` を削除する操作の値を 1 件持つ |
