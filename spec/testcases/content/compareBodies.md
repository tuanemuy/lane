# 内容を比較する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-compareBodies-01 | A の履歴に R1 と R2 がある。項目 `title` の値だけが違う | メンバーが、A の R1 と R2 を指定して比較する | R1 と R2 の値、`title` の差分、A の `version` を返す |
| TC-content-compareBodies-02 | A は公開中で、下書きを持つ。A の履歴に R1 がある | メンバーが、A の R1 と最新の内容を指定して比較する | R1 の値と下書きの値、項目ごとの差分を返す |
| TC-content-compareBodies-04 | A は公開中で、下書きを持つ | メンバーが、A の現在の下書きと現在の公開内容を指定して比較する | 下書きの値と公開内容の値、項目ごとの差分、A の `version` を返す |
| TC-content-compareBodies-05 | A は公開中で、下書きを持たない | メンバーが、A の現在の下書きと現在の公開内容を指定して比較する | エラーにしない。下書きが無いことを返す |
| TC-content-compareBodies-06 | A の履歴の版 R1 は、履歴の版の整理で削除されている | メンバーが、A の R1 と最新の内容を指定して比較する | `NotFoundError` を返す |
| TC-content-compareBodies-07 | R9 は、別のコンテンツ B の履歴の版 | メンバーが、A の R9 と最新の内容を指定して比較する | `NotFoundError` を返す |
| TC-content-compareBodies-08 | A の履歴に R1 と R2 がある。繰り返しの項目 `tags` は、R1 で要素を二つ、R2 で三つ持ち、先頭の二つの要素は同じ。ほかの項目は同じ | メンバーが、A の R1 と R2 を指定して比較する | 差分は、`tags` の 3 番目の位置の `added` だけ |
| TC-content-compareBodies-09 | A の履歴に R1 と R2 がある。ブロックの項目 `sections` の先頭の位置のブロックは、R1 で種類 `text`、R2 で種類 `image`。ほかの項目は同じ | メンバーが、A の R1 と R2 を指定して比較する | 差分は、`sections` の先頭の位置の `changed` だけ |
| TC-content-compareBodies-10 | A の履歴に R1 と R2 がある。項目のまとまり `seo` の中の項目 `title` だけが違い、`seo` の中の項目 `description` は同じ | メンバーが、A の R1 と R2 を指定して比較する | 差分は、`seo` の中の `title` の位置の `changed` だけ |
