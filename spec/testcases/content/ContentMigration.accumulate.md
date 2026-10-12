# 移行の報告を集計に畳み込む（`ContentMigration.accumulate`）

ユースケースを通らない、ドメインの純粋な関数の代表ケース。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-ContentMigration.accumulate-01 | 操作 P がモデル `m` を削除する。`m` の `a` と `b` の `removed` の報告（`removedBy` は P）がある | 二つの報告を、`a`、`b` の順に畳み込む | P の `losses` の `contents` は、`total` が 2、`items` が `a`、`b`。`values` は 0 |
| TC-content-ContentMigration.accumulate-02 | 操作 Q が項目 `x` を削除する。`kept` の報告の `losses` に、Q で失う値が 3 件ある。版の報告の `losses` に、Q で失う値が 2 件ある | 二つの報告を畳み込む | Q の `losses` の `values` は 5、`contents` は `total` が 0 |
| TC-content-ContentMigration.accumulate-03 | 操作 P の `removed` の報告が、`SchemaLimits.sampleLimit` より 1 件多くある | すべてをコンテンツID の昇順に畳み込む | P の `contents` の `total` は、`sampleLimit` より 1 多い数。`items` は、先頭の `sampleLimit` 件 |
| TC-content-ContentMigration.accumulate-04 | 操作 P の `removed` の報告と、操作 R の `removed` の報告がある | 二つの報告を畳み込む | P と R の `contents` は、それぞれ自分の報告のコンテンツだけを持つ |
| TC-content-ContentMigration.accumulate-05 | 操作 P がモデル `m` を削除する。`m` の `a` の `removed` の報告（`removedBy` は P）と、残るモデルの `b` の `kept` の報告がある。`b` の報告の `losses` に、P で取り除いた `a` への参照の値が 1 件ある | 二つの報告を、`a`、`b` の順に畳み込む | P の `losses` の `contents` は、`total` が 1、`items` が `a`。`values` は 1 |
