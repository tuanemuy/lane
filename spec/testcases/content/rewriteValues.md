# 値を書き換える

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-rewriteValues-01 | A は公開中で、下書きを持つ。`version` は 2 | 公開権限と編集権限を持つ `content_manage` のトークンで、A と `version` 2 を指定し、書き換え先を公開内容にして、項目 `title` を「X」に書き換える | A の公開内容の `title` は「X」になり、公開内容のほかの項目と、下書きは変わらない。公開内容の更新日時は書き換えの時刻。書き換え前の公開内容は、公開内容だった版として履歴に残る。版にした操作は値の書き換え、実行者はトークンで、トークンを指す値（名称または発行の経路）と、発行者または紐付くメンバーの名前を記録する。入力の操作の ID、A の `version` 3 と書き換え先（公開内容）を返す。公開の変更の記録を一件追記する（入力の操作の ID、`operation` は `rewritten`、実行者、`changeList.changes` は A の `updated`、確定の時点のスキーマの版）。`content.publication_changed` を一つ記録する |
| TC-content-rewriteValues-02 | A は公開中で、下書きを持つ | 公開権限と編集権限を持つ `content_manage` のトークンで、書き換え先を下書きにして、A の `title` を「X」に書き換える | A の下書きの `title` は「X」になり、公開内容は変わらない。書き換え前の下書きは、下書きだった版として履歴に残る。A は公開中で下書きを持つまま。入力の操作の ID を返す。公開の変更の記録を追記せず、`content.publication_changed` を記録しない |
| TC-content-rewriteValues-03 | A は公開中で、下書きを持つ | 公開権限と編集権限を持つ `content_manage` のトークンで、書き換え先を両方にして、A の公開内容の `title` を「X」に、下書きの `title` を「Y」に書き換える | 公開内容の `title` は「X」、下書きの `title` は「Y」になる。書き換え前の公開内容と下書きは、それぞれ履歴の版として残る |
| TC-content-rewriteValues-04 | A と B は公開中 | 公開権限と編集権限を持つ `content_manage` のトークンで、A と B の公開内容を一つの操作で書き換える | A と B の公開内容が変わる。公開の変更の記録は一件で、`changeList.changes` は A と B の変更（どちらも `updated`）を持つ。`content.publication_changed` は一つ |
| TC-content-rewriteValues-05 | C はごみ箱にある | 公開権限と編集権限を持つ `content_manage` のトークンで、C の下書きの `title` を書き換える | C の下書きの `title` が変わる。C はごみ箱のまま |
| TC-content-rewriteValues-06 | A のモデルの項目 `legacy` は非推奨 | 公開権限と編集権限を持つ `content_manage` のトークンで、A の下書きの `legacy` を書き換える | A の下書きの `legacy` が変わる |
| TC-content-rewriteValues-07 | A は公開中。非公開のメディア Md がある。トークンは、`content_manage` の許可と、公開権限、編集権限、メディア公開権限を持つ | トークンで、A の公開内容のメディアの項目を Md に書き換える | A の公開内容が変わり、Md は公開になる。公開したメディアとして Md を返す。Md の使用箇所に、A の公開内容での使用が加わる |
| TC-content-rewriteValues-08 | A は下書きを持つ。非公開のメディア Md がある | 公開権限と編集権限を持つ `content_manage` のトークンで、A の下書きのメディアの項目だけを Md に書き換える | A の下書きが変わる。Md は非公開のまま |
| TC-content-rewriteValues-09 | A は下書きを持つ。A のモデルの項目 `summary` は必須 | 公開権限と編集権限を持つ `content_manage` のトークンで、A の下書きの `summary` を未設定に書き換える | エラーにしない。A の下書きを書き換え、必須項目の欠落を下書きの入力エラーに持つ |
| TC-content-rewriteValues-10 | A の公開内容は、項目 `body` に現行の制約の違反を持つ | 公開権限と編集権限を持つ `content_manage` のトークンで、A の公開内容の `title` だけを、制約を満たす値に書き換える | エラーにしない。A の公開内容の `title` が変わる。`body` の違反は、公開内容の `violations` に残る |
| TC-content-rewriteValues-12 | A と B は公開中。A の公開内容の `title` は「X」 | 公開権限と編集権限を持つ `content_manage` のトークンで、A の公開内容の `title` を「X」に、B の公開内容の `title` を新しい値に、一つの操作で書き換える | B の公開内容が変わり、B の `version` は 1 進む。A は変わらなかった対象として返し、A の `version` は進まず、A の履歴の版は作られない。公開の変更の記録の `changeList.changes` は、B の変更だけを持つ |
| TC-content-rewriteValues-13 | A のモデルに数値の項目 `order` がある | 公開権限と編集権限を持つ `content_manage` のトークンで、A の下書きの `order` に、数値でない文字列を指定して書き換える | `ValidationError` を返す。対象 A と項目ごとの理由を添える。A は変わらない |
| TC-content-rewriteValues-14 | A は公開中。A のモデルの項目 `title` は、文字数の制約を持つ | 公開権限と編集権限を持つ `content_manage` のトークンで、A の公開内容の `title` を、文字数の制約に違反する値に書き換える | `BusinessRuleError`（`RewriteRejected`）を返す。A の入力エラーを添える。A は変わらない |
| TC-content-rewriteValues-16 | A と B は同じモデルで公開中。B の公開内容の一意な項目 `slug` は「x」 | 公開権限と編集権限を持つ `content_manage` のトークンで、A の公開内容の `slug` を「x」に書き換える | `BusinessRuleError`（`RewriteRejected`）を返す。A、`slug`、相手の B を添える。A は変わらない |
| TC-content-rewriteValues-17 | A と B は同じモデルで公開中。A の書き換えの事前の検証の後、確定までに、B の公開内容の一意な項目 `slug` を「x」にする書き換えが確定した | 公開権限と編集権限を持つ `content_manage` のトークンで、A の公開内容の `slug` を「x」に書き換える | `ConflictError`（`UNIQUE_VALUE_TAKEN`）を返す。A は変わらない。B の公開内容の `slug` は「x」のまま |
| TC-content-rewriteValues-18 | U は未公開 | 公開権限と編集権限を持つ `content_manage` のトークンで、書き換え先を公開内容にして、U の `title` を書き換える | `BusinessRuleError`（`RewriteRejected`）を返す。U が公開内容を持たないことを添える。U は公開されない |
| TC-content-rewriteValues-19 | B は公開中で、下書きを持たない | 公開権限と編集権限を持つ `content_manage` のトークンで、書き換え先を下書きにして、B の `title` を書き換える | `BusinessRuleError`（`RewriteRejected`）を返す。B が下書きを持たないことを添える。B に下書きは作られない |
| TC-content-rewriteValues-20 | A と B は公開中。非公開のメディア Md がある。トークンは、`content_manage` の許可と、公開権限、編集権限、メディア公開権限を持つ | トークンで、A の公開内容のメディアの項目を Md に、B の公開内容の `title` を制約に違反する値に、一つの操作で書き換える | `BusinessRuleError`（`RewriteRejected`）を返す。A と B は、どちらも変わらない。Md は非公開のまま。公開の変更の記録を追記せず、`content.publication_changed` を記録しない |
| TC-content-rewriteValues-21 | スクリプトが A と B を読み取った後、ほかの操作で B の内容が変わった | 公開権限と編集権限を持つ `content_manage` のトークンで、A と B を、読み取ったときの `version` を指定して書き換える | `ConflictError`（`STALE_VERSION`）を返す。B を添える。A と B は、どちらも書き換えない |
| TC-content-rewriteValues-22 | 指定するメディアは、ワークスペースに存在しない | 公開権限と編集権限を持つ `content_manage` のトークンで、A の下書きのメディアの項目を、そのメディアに書き換える | `ConflictError`（`MEDIA_NOT_FOUND`）を返す。A は変わらない |
| TC-content-rewriteValues-23 | T は完全削除されている。A のモデルのコンテンツ参照の項目 `related` は必須。A は公開中で、下書きを持つ | 公開権限と編集権限を持つ `content_manage` のトークンで、A の下書きの `related` を T に書き換える | 書き換える。T への参照は参照切れとして保存する |
| TC-content-rewriteValues-24 | T は完全削除されている。A のモデルのコンテンツ参照の項目 `related` は必須。A は公開中 | 公開権限と編集権限を持つ `content_manage` のトークンで、A の公開内容の `related` を T に書き換える | `BusinessRuleError`（`RewriteRejected`）を返す。A、項目、参照先 T、理由（存在しない）を添える。A は変わらない |
| TC-content-rewriteValues-25 | X のコンテンツは無い | 公開権限と編集権限を持つ `content_manage` のトークンで、X の下書きを書き換える | `NotFoundError` を返す。X を示す |
| TC-content-rewriteValues-27 | トークンは `content_manage` の許可と編集権限を持ち、公開権限を持たない。A は公開中で、下書きを持つ | トークンで、A の公開内容を書き換える | `ForbiddenError`（`PERMISSION_MISSING`）を返す。不足する権限（`content:publish`）と、該当する対象 A を添える。A は変わらない |
| TC-content-rewriteValues-28 | トークンは `content_manage` の許可と公開権限を持ち、編集権限を持たない。A は公開中で、下書きを持つ | トークンで、A の下書きを書き換える | `ForbiddenError`（`PERMISSION_MISSING`）を返す。不足する権限（`content:edit`）と、該当する対象 A を添える。A は変わらない |
| TC-content-rewriteValues-29 | トークンは、`content_manage` の許可と公開権限を持ち、メディア公開権限を持たない。A は公開中。非公開のメディア Md がある | トークンで、A の公開内容のメディアの項目を Md に書き換える | `ForbiddenError`（`PERMISSION_MISSING`）を返す。Md を添える。A は変わらない。Md は非公開のまま |
| TC-content-rewriteValues-31 | トークンは `content_manage` の許可を持たない | トークンで、A の下書きを書き換える | `ForbiddenError`（`GRANT_MISSING`）を返す。A は変わらない |
| TC-content-rewriteValues-32 | A は公開中。書き換えの入力は、公開のメディア Md を A の公開内容に新たに使う。事前の検証の後、確定までに、Md が非公開にされた | 公開権限と編集権限を持つ `content_manage` のトークンで、A の公開内容のメディアの項目を Md に書き換える | `ConflictError`（`MEDIA_NOT_PUBLIC`）を返す。A は変わらない |
| TC-content-rewriteValues-33 | A は公開中。書き換えの入力は、A の公開内容に、`ContentBatch.maxPublishedMedia` を超える数の非公開のメディアを新たに使う。トークンは、`content_manage` の許可と、公開権限、編集権限、メディア公開権限を持つ | トークンで、A の公開内容を書き換える | `BusinessRuleError`（`RewriteRejected`）を返す。新たに公開するメディアの数と上限を添える。A は変わらない。どのメディアも非公開のまま |
| TC-content-rewriteValues-34 | A のモデルにリッチテキストの項目 `body` がある。メディア Md は、画像でないファイル | 公開権限と編集権限を持つ `content_manage` のトークンで、A の下書きの `body` を、本文の画像に Md を使う値に書き換える | `ValidationError` を返す。対象 A と、該当する位置と理由（`imageRequired`）を添える。A は変わらない |
| TC-content-rewriteValues-35 | スクリプトが A と B を読み取った後、ほかの操作で A の内容が変わった。B のモデルに数値の項目 `order` がある | 公開権限と編集権限を持つ `content_manage` のトークンで、A と B を、読み取ったときの `version` を指定し、A の `title` と、B の下書きの `order` に数値でない文字列を指定して書き換える | `ConflictError`（`STALE_VERSION`）を返す。A を添える。`rejections` は、A の `STALE_VERSION` と、B の `ValidationError`（項目ごとの理由）を、この順に持つ。A と B は、どちらも書き換えない |
