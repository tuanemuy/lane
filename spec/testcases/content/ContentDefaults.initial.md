# 初期値を求める（`ContentDefaults.initial`）

ユースケースを通らない、ドメインの純粋な関数の代表ケース。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-ContentDefaults.initial-01 | モデルに、初期値「x」を持つテキストの項目 `title` と、初期値を持たない数値の項目 `order` がある | モデル直下の項目を渡して `initial` を呼ぶ | `title` は「x」。`order` は未設定 |
| TC-content-ContentDefaults.initial-02 | モデルに、作成時点を初期値に指定した日付の項目 `publishedOn` と日時の項目 `startsAt` がある | 暦日 d と時点 t を `at` に渡して `initial` を呼ぶ | `publishedOn` は d、`startsAt` は t |
| TC-content-ContentDefaults.initial-03 | モデルに、固定の日付を初期値に持つ日付の項目 `opensOn` がある | 暦日 d を `at` に渡して `initial` を呼ぶ | `opensOn` は、定義した固定の日付。d にならない |
| TC-content-ContentDefaults.initial-04 | モデルに、初期値を持つ非推奨の項目 `legacy` がある | モデル直下の項目を渡して `initial` を呼ぶ | `legacy` は未設定 |
| TC-content-ContentDefaults.initial-05 | モデルの単一選択の項目 `tone` の初期値は、非推奨の選択肢 `old` | モデル直下の項目を渡して `initial` を呼ぶ | `tone` は未設定 |
| TC-content-ContentDefaults.initial-06 | モデルの複数選択の項目 `tags` の初期値は、非推奨の選択肢 `old` と、非推奨でない選択肢 `new` | モデル直下の項目を渡して `initial` を呼ぶ | `tags` は `new` だけを持つ |
| TC-content-ContentDefaults.initial-07 | モデルに、項目のまとまり `seo` があり、その中のテキストの項目 `description` は初期値「y」を持つ。繰り返しの項目 `sections` がある | モデル直下の項目を渡して `initial` を呼ぶ | `seo` の `description` は「y」。`sections` は要素を持たない |
| TC-content-ContentDefaults.initial-08 | 繰り返しの項目 `sections` の要素の項目 `heading` は、初期値「z」を持つ | `sections` の要素の項目を渡して `initial` を呼ぶ（要素の追加） | `heading` は「z」 |
| TC-content-ContentDefaults.initial-09 | `initial` の結果を、変えずに値にする | その値で `ContentValidation.checkEdit` を、`before` を `null` にして呼ぶ | 非推奨の違反（`deprecated`）を返さない |
