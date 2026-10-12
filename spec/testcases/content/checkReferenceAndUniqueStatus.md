# 参照と一意な値の状態を確認する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-checkReferenceAndUniqueStatus-01 | A の下書きが、必須の参照で公開中の T1 を、任意の参照で未公開の T2 を、本文のリンクでごみ箱の T3 を指す | メンバーが、A の状態を確認する | T1 への参照は解決できる。T2 への参照は、任意の参照で、解決できない（理由 `unpublished`）。T3 への参照は、本文のリンクで、解決できない（理由 `trashed`）。T1、T2、T3 のそれぞれの状態を返す |
| TC-content-checkReferenceAndUniqueStatus-02 | A の下書きが、任意の参照で T を、必須の参照で U を指す。T と U は完全削除されている | メンバーが、A の状態を確認する | T への参照と U への参照は、どちらも解決できない（理由 `missing`） |
| TC-content-checkReferenceAndUniqueStatus-03 | A は公開中で、公開内容は T1 を、下書きは T2 を参照する | メンバーが、A の状態を確認する | 最新の内容の結果は、下書きの参照である T2 について返し、T1 を含まない。公開内容の結果は、T1 について返し、T2 を含まない |
| TC-content-checkReferenceAndUniqueStatus-04 | A は公開中で、下書きを持つ。公開内容は、任意の参照で T を指す。T は公開を取り消されている。下書きは T を参照しない | メンバーが、A の状態を確認する | 公開内容の結果は、T への参照を、任意の参照で、解決できない（理由 `unpublished`）として返す。最新の内容の結果は、T を含まない |
| TC-content-checkReferenceAndUniqueStatus-05 | A は公開中で、下書きを持たない | メンバーが、A の状態を確認する | 最新の内容の結果は、公開内容の参照について返す。公開内容の結果は無い（`null`） |
| TC-content-checkReferenceAndUniqueStatus-06 | A は未公開で、下書きを持つ | メンバーが、A の状態を確認する | 公開内容の結果は無い（`null`） |
| TC-content-checkReferenceAndUniqueStatus-07 | A の下書きの一意な項目 `slug` の値が「x」。同じモデルの B の公開内容の `slug` も「x」 | メンバーが、A の状態を確認する | `slug` について、同じ値を公開内容に持つコンテンツとして B を返す |
| TC-content-checkReferenceAndUniqueStatus-08 | A の下書きの `slug` の値が「x」。同じモデルの未公開の C の下書きの `slug` も「x」。「x」を公開内容に持つコンテンツは無い | メンバーが、A の状態を確認する | `slug` について、重複するコンテンツを返さない |
| TC-content-checkReferenceAndUniqueStatus-09 | 指定したコンテンツID のコンテンツが無い | メンバーが、そのコンテンツの状態を確認する | `NotFoundError` を返す |
| TC-content-checkReferenceAndUniqueStatus-11 | トークンは `content_manage` の許可を持たない | トークンで、A の状態を確認する | `ForbiddenError`（`GRANT_MISSING`）を返す |
