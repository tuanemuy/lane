# 一意性の重複を集計に加える（`ContentMigration.withDuplicates`）

ユースケースを通らない、ドメインの純粋な関数の代表ケース。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-ContentMigration.withDuplicates-01 | `blockers` が空の集計がある。`findDuplicates` の結果は、`contentCount` が `SchemaLimits.sampleLimit` より 1 多く、`items` が、上限の件数の `contents` を持つ一組 | 集計と結果を `withDuplicates` に渡す | `blockers` の `total` は、`sampleLimit` より 1 多い数。`items` は、組の `contents` の各コンテンツの `uniqueViolation` で、`duplicates` は組のほかのコンテンツ |
