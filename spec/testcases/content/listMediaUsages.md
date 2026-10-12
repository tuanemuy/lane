# メディアの使用箇所を一覧する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-listMediaUsages-01 | `content:edit` を持たないメンバー。公開のメディアが、A の公開内容、B の下書き、C の履歴の版、ごみ箱の D の下書きから使われている | 使用箇所を一覧する | 4 件と総件数 4、使用の状況 `in_public_use`、公開範囲 `public` を返す。使用箇所ごとに、コンテンツ、項目の位置、内容の種類、コンテンツの置き場所（通常、ごみ箱）を返す |
| TC-content-listMediaUsages-02 | メンバー。公開のメディアが、A の下書きと、B の公開内容から使われている。一ページの件数は 1 | 1 ページ目を一覧する | 1 件と総件数 2 を返す。使用の状況は、ページによらず `in_public_use` を返す |
| TC-content-listMediaUsages-03 | メンバー。メディアが、A の下書きの項目 `image1` と `image2` から使われている | 使用箇所を一覧する | A の下書きの使用箇所を 1 件返す。項目の位置は `image1` と `image2`。総件数は 1 |
| TC-content-listMediaUsages-04 | メンバー。非公開のメディアが下書きだけから使われている | 使用箇所を一覧する | 下書きの使用箇所を返す。使用の状況 `in_use`、公開範囲 `private` を返す |
| TC-content-listMediaUsages-05 | メンバー。メディアが、A の履歴の版 R から使われている | 使用箇所を一覧する | 内容の種類が履歴の版で、A と R の ID を持つ使用箇所を返す |
| TC-content-listMediaUsages-06 | メンバー。使用箇所の無いメディアがある | 使用箇所を一覧する | 空の一覧と総件数 0、使用の状況 `unused` を返す。エラーにしない |
| TC-content-listMediaUsages-07 | メンバー | 存在しないメディアの ID を指定する | `NotFoundError` を返す |
