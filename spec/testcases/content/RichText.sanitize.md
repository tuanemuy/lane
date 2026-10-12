# 貼り付けた本文を取り込む（`RichText.sanitize`）

ユースケースを通らない、ドメインの純粋な関数の代表ケース。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-RichText.sanitize-01 | 貼り付けた内容は、段落、見出し、太字、斜体、箇条書き、引用、`https` のリンクだけから成る | `sanitize` に渡す | 同じ構造の `RichText` を返す。`changes` は空 |
| TC-content-RichText.sanitize-02 | 貼り付けた内容は、段落と、表と、スクリプトを含む | `sanitize` に渡す | 表とスクリプトを、中のノードとともに取り除き、段落を残した `RichText` を返す。`changes` は、表とスクリプトのそれぞれに一件ずつ、`removed` と `unsupportedElement` を持つ。表とスクリプトの中のノードの分を持たない |
| TC-content-RichText.sanitize-03 | 貼り付けた内容は、リストの項目の中に見出しを持つ | `sanitize` に渡す | 見出しを段落に変えた `RichText` を返す。`changes` は、`convertedToParagraph` と `disallowedPosition` を持つ |
| TC-content-RichText.sanitize-04 | 貼り付けた内容は、引用の中に画像を持つ | `sanitize` に渡す | 画像を取り除いた `RichText` を返す。`changes` は、`removed` と `disallowedPosition` を持つ |
| TC-content-RichText.sanitize-05 | 貼り付けた内容は、スキームが `javascript` の URL のリンクを付けたテキストを持つ | `sanitize` に渡す | リンクを外し、テキストを残した `RichText` を返す。`changes` は、`linkRemoved` と `unsafeUrl` を持つ |
| TC-content-RichText.sanitize-06 | 貼り付けた内容は、空のテキストと、文字を持つテキストから成る段落 | `sanitize` に渡す | 空のテキストを取り除いた `RichText` を返す。`changes` は空 |
| TC-content-RichText.sanitize-07 | 貼り付けた内容は、スクリプトだけから成る | `sanitize` に渡す | `value` は `null`（未設定）。`changes` は、`removed` と `unsupportedElement` を持つ |
