# 型の導出

`packages/client` が、`defineSchema` の結果の型から導く型。規則は [schema.md](../../domains/schema.md) の「型の対応」、公開する型は [app-api.md](../../usecases/app-api.md) の「型の導出」が定める。型を導くために CMS へ接続しない。期待結果は、型の検査（代入できる、できない）で確かめる。断りが無い限り、スキーマ S は、次の項目を持つモデル article と、フォーム contact を持つ。

- article: 必須のテキスト title、任意のテキスト lead、任意の複数選択 tags（選択肢 a、b）、任意のメディア cover、モデル author への任意の参照 writer
- contact: 必須のテキスト name、任意のテキスト note

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-client-types-01 | スキーマ S | `Content<S, "article">` を導く | title は文字列で、`null` を含まない。lead は文字列または `null`。tags は、`"a"` と `"b"` の直和の配列で、`null` を含まない。項目のほかに、コンテンツID、作成日時、更新日時を持つ |
| TC-client-types-02 | スキーマ S | `Content<S, "article">` を、展開の指定なしで導く | writer は、要約（モデルのキー `"author"`、コンテンツID、一意な値の組）、または `null` |
| TC-client-types-03 | スキーマ S | `Content<S, "article", E>` を、writer を展開する項目に含み、深さが 1 の指定 `E` で導く | writer は、展開した内容（author の一件の取得結果）、または `null` |
| TC-client-types-04 | スキーマ S | 型に、スキーマに無いモデルのキー `"page"` を指定する | 型の検査が受け付けない |
| TC-client-types-05 | スキーマ S | `ContentInput<S, "article">` を導く | title も lead も省略できる。title に数を指定する値は、型の検査が受け付けない。cover は、メディアの ID と、使用箇所ごとの説明。writer は、コンテンツID |
| TC-client-types-06 | スキーマ S | `ContentIssues<S, "article">` を導く | 位置は、article の項目のキー（`"title"`、`"lead"`、`"tags"`、`"cover"`、`"writer"`）から始まる。article に無いキー `"body"` から始まる位置は、型の検査が受け付けない。理由は、`TypeErrorKind` と `ConstraintKind` |
| TC-client-types-07 | スキーマ S | `Submission<S, "contact">` を導く | name は省略できない。note は省略できる |
| TC-client-types-08 | スキーマ S | `SubmissionIssues<S, "contact">` を導く | キーは、`"name"` と `"note"` の直和 |
| TC-client-types-09 | スキーマ S | `Reception<S, "contact">` を導く | 受付完了と、理由付きの受付失敗の直和 |
| TC-client-types-10 | スキーマ S は、派生形式 thumb（`output.format` は `webp`）を定義する | `Content<S, "article">` の cover を導く | 派生形式のキー `"thumb"` の結果を持つ。結果は、配信先、画像形式（`"jpeg"`・`"webp"`・`"avif"`・`"png"` の直和）、幅、高さ、または `null` |
| TC-client-types-11 | S の lead と、tags の選択肢 b に、`deprecated: true` を宣言したスキーマ | 型を導く | lead のプロパティの宣言と、選択肢 b の宣言は、非推奨の印（`@deprecated`）を持つ。lead と b は、型から除かれない |
| TC-client-types-13 | S の title に、文字数の制約と表示名称を宣言したスキーマ | `Content` と `ContentInput` を導く | 型は、S から導いた型と同じ |
| TC-client-types-14 | スキーマ S を、CMS に反映していない | 型を導き、識別子を求める | 型も識別子も、CMS に接続せずに求まる |
