# 履歴の版を移行で変換する（`ContentMigration.migrateRevision`）

ユースケースを通らない、ドメインの純粋な関数の代表ケース。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-ContentMigration.migrateRevision-01 | 反映の計画が、モデル `m` を削除する。コンテンツ D は `m` に属する。反映後も残るモデルのコンテンツ A の履歴の版 R1 の本文が、テキスト「詳細」に付けた D へのリンクを持つ | R1 を `migrateRevision` に渡す | 変換した R1 を返す。R1 の本文は D へのリンクを持たず、テキスト「詳細」は残る。R1 の `references` は D を含まない。`losses` は、`m` を削除する操作の値を 1 件持つ |
