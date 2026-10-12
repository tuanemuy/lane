# 公開を取り消す

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-unpublishContent-01 | A は公開中で、下書きを持たない。`version` は 2。A を公開内容から参照するコンテンツは無い | 公開権限を持つメンバーが、A と確認した `version` 2 を指定して公開を取り消す | A は未公開になる。公開内容だった値は下書きになり、公開内容の更新日時を引き継ぐ。履歴の版は作られない。入力の操作の ID と `version` 3 を返す。公開の変更の記録を一件追記する（入力の操作の ID、`operation` は `unpublished`、実行者、`changeList.changes` は A の `unpublished`、`changeList.referrers` は空、確定の時点のスキーマの版）。`content.publication_changed` を一つ記録する。公開取得は A を返さない |
| TC-content-unpublishContent-02 | B は公開中で、下書きを持つ | 公開権限を持つメンバーが、B の公開を取り消す | B は未公開になり、下書きは変わらない。なくした公開内容は、公開内容だった版として履歴に残る。版にした操作は公開取り消し |
| TC-content-unpublishContent-03 | A の公開内容は、公開のメディア Md を使う | 公開権限を持つメンバーが、A の公開を取り消す | Md は公開のまま |
| TC-content-unpublishContent-04 | 公開中の R1 が、公開内容の任意の参照で A を参照する。D が、下書きの必須の参照で A を参照する | 公開権限を持つメンバーが、A の公開を取り消す | A は未公開になる。R1 の公開内容と、D の下書きは変わらない。公開の変更の記録の `changeList.referrers` は R1 を持ち、D を持たない |
| TC-content-unpublishContent-05 | A と B は公開中で、公開内容が互いを必須の参照で参照する | 公開権限を持つメンバーが、A と B の公開を一つの操作で取り消す | A と B は、どちらも未公開になる。公開の変更の記録は一件で、`changeList.changes` は A と B の変更（どちらも `unpublished`）を持ち、`changeList.referrers` は A と B を含まない。`content.publication_changed` は一つ |
| TC-content-unpublishContent-06 | A のモデルは非推奨。A は公開中 | 公開権限を持つメンバーが、A の公開を取り消す | A は未公開になる |
| TC-content-unpublishContent-08 | 公開中の R2 が、公開内容の必須の参照で A を参照する | 公開権限を持つメンバーが、A だけの公開を取り消す | `BusinessRuleError`（`WithdrawRejected`）を返す。対象 A、参照元 R2、項目を添える。A は公開中のまま。公開の変更の記録を追記せず、`content.publication_changed` を記録しない |
| TC-content-unpublishContent-10 | A と B は公開中。公開中の R2 が、必須の参照で A を参照する。B はどこからも参照されていない | 公開権限を持つメンバーが、A と B の公開を一つの操作で取り消す | `BusinessRuleError`（`WithdrawRejected`）を返す。A と B は、どちらも公開中のまま |
| TC-content-unpublishContent-11 | A の公開取り消しの事前の確認の後、確定までに、A を必須の参照で参照する R2 の公開が確定した | 公開権限を持つメンバーが、A の公開を取り消す | `ConflictError`（`PUBLISHED_REFERENCE_REQUIRED`）を返す。A は公開中のまま。公開の変更の記録を追記せず、`content.publication_changed` を記録しない |
| TC-content-unpublishContent-12 | メンバーが A を確認した後、A を必須の参照で参照する R2 が公開された。A の `version` は変わっていない | メンバーが、A と確認した `version` を指定して公開を取り消す | `BusinessRuleError`（`WithdrawRejected`）を返す。対象 A、参照元 R2 を添える。A は公開中のまま |
| TC-content-unpublishContent-13 | メンバーが A を `version` 2 で確認した後、ほかのメンバーが A の公開を取り消し、`version` は 3 | メンバーが、A と `version` 2 を指定して公開を取り消す | `ConflictError`（`STALE_VERSION`）を返す。`ContentNotPublished` にしない。A は変わらない |
| TC-content-unpublishContent-14 | U は未公開 | `content_manage` の許可と必要な権限を持つトークンで、U と現在の `version` を指定して公開を取り消す | `BusinessRuleError`（`ContentNotPublished`）を返す。U は変わらない |
| TC-content-unpublishContent-15 | C はごみ箱にある | `content_manage` の許可と必要な権限を持つトークンで、C と現在の `version` を指定して公開を取り消す | `BusinessRuleError`（`ContentInTrash`）を返す。C は変わらない |
| TC-content-unpublishContent-16 | メンバーが A を `version` 2 で確認した後、ほかの操作で A の下書きが変わり、`version` は 3 | メンバーが、A と `version` 2 を指定して公開を取り消す | `ConflictError`（`STALE_VERSION`）を返す。A は公開中のまま |
| TC-content-unpublishContent-17 | A と B を確認した後、B が完全削除されている | 公開権限を持つメンバーが、A と B の公開を一つの操作で取り消す | `NotFoundError` を返す。B を示す。A は公開中のまま |
| TC-content-unpublishContent-19 | メンバーは、編集権限を持ち、公開権限を持たない | A の公開を取り消す | `ForbiddenError`（`PERMISSION_MISSING`）を返す。不足する権限（`content:publish`）を添える。A は公開中のまま |
| TC-content-unpublishContent-20 | トークンは `content_manage` の許可と公開権限を持ち、範囲に A のモデルを含む | トークンで、A と読み取ったときの `version` を指定して公開を取り消す | A は未公開になる |
