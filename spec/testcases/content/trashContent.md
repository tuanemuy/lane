# コンテンツをごみ箱へ移動する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-trashContent-01 | A は未公開で、通常の管理対象にあり、履歴の版を持つ。`version` は 1 | 削除権限を持つメンバーが、A と確認した `version` 1 を指定して移動する | A はごみ箱にあり、未公開。移動した日時は移動の時刻、移動した実行者はそのメンバー。下書きと履歴の版は変わらない。`version` 2 を返す。新しい履歴の版は作られない。入力の操作の ID を返す。公開の変更の記録を追記せず、`content.publication_changed` を記録しない。A は、通常の一覧、参照先の候補、プレビュー取得に現れない |
| TC-content-trashContent-02 | B は公開中で、下書きを持たない。`version` は 3。公開内容は、公開のメディア Md を使う。メンバーは、削除権限と公開権限を持つ | メンバーが、B をごみ箱へ移動する | B はごみ箱にあり、未公開。`version` は 5（公開取り消しとごみ箱への移動）。最後の変更はごみ箱への移動。公開内容だった値を下書きに持つ。入力の操作の ID と、公開中だった対象として B を返す。公開の変更の記録を一件追記する（入力の操作の ID、`operation` は `trashed`、実行者、`changeList.changes` は B の `unpublished`、確定の時点のスキーマの版）。`content.publication_changed` を一つ記録する。Md は公開のまま。Md の使用箇所は、B の下書きでの使用になる |
| TC-content-trashContent-03 | B は公開中で、下書きを持つ。メンバーは、削除権限と公開権限を持つ | メンバーが、B をごみ箱へ移動する | B はごみ箱にあり、下書きは変わらない。なくした公開内容は、公開内容だった版として履歴に残る |
| TC-content-trashContent-04 | A は未公開、B は公開中。メンバーは、削除権限と公開権限を持つ | メンバーが、A と B を一つの操作で移動する | A と B は、どちらもごみ箱にある。公開の変更の記録は一件で、`changeList.changes` は B の変更（`unpublished`）だけを持つ。`content.publication_changed` は一つ |
| TC-content-trashContent-05 | A は未公開。D が、下書きの必須の参照で A を参照する | 削除権限を持つメンバーが、A をごみ箱へ移動する | A はごみ箱にある。D の下書きは変わらない |
| TC-content-trashContent-06 | B は公開中。公開中の R1 が、公開内容の任意の参照で B を参照する。メンバーは、削除権限と公開権限を持つ | メンバーが、B をごみ箱へ移動する | B はごみ箱にある。R1 の公開内容は変わらない |
| TC-content-trashContent-07 | A のモデルは非推奨。A は未公開 | 削除権限を持つメンバーが、A をごみ箱へ移動する | A はごみ箱にある |
| TC-content-trashContent-09 | B は公開中。公開中の R2 が、公開内容の必須の参照で B を参照する。A は未公開。メンバーは、削除権限と公開権限を持つ | メンバーが、A と B を一つの操作で移動する | `BusinessRuleError`（`WithdrawRejected`）を返す。対象 B、参照元 R2、項目を添える。A と B は、どちらも通常の管理対象のまま。B は公開中のまま |
| TC-content-trashContent-10 | B の移動の事前の確認の後、確定までに、B を必須の参照で参照する R2 の公開が確定した。メンバーは、削除権限と公開権限を持つ | メンバーが、公開中の B をごみ箱へ移動する | `ConflictError`（`PUBLISHED_REFERENCE_REQUIRED`）を返す。B は公開中のまま、通常の管理対象にある |
| TC-content-trashContent-11 | C はごみ箱にある | `content_manage` の許可と必要な権限を持つトークンで、C と現在の `version` を指定して移動する | `BusinessRuleError`（`ContentInTrash`）を返す。C の移動した日時と実行者は変わらない |
| TC-content-trashContent-12 | メンバーが A を確認した後、ほかのメンバーが A をごみ箱へ移動し、`version` が進んだ | メンバーが、A と確認した `version` を指定して移動する | `ConflictError`（`STALE_VERSION`）を返す。A の移動した日時と実行者は変わらない |
| TC-content-trashContent-13 | A は未公開、B は公開中。メンバーは、削除権限を持ち、公開権限を持たない | メンバーが、A と B を、現在の `version` で一つの操作で移動する | `ForbiddenError`（`PERMISSION_MISSING`）を返す。不足する権限（`content:publish`）と、該当する対象 B を添える。A と B は、どちらも通常の管理対象のまま |
| TC-content-trashContent-14 | メンバーが未公開の A を確認した後、A が公開され、A の `version` が進んだ。メンバーは、削除権限を持ち、公開権限を持たない | メンバーが、A と確認したときの `version` を指定して移動する | `ConflictError`（`STALE_VERSION`）を返す。`ForbiddenError` にしない。A は公開中のまま、通常の管理対象にある |
| TC-content-trashContent-15 | メンバーが A を `version` 1 で確認した後、ほかの操作で A の下書きが変わり、`version` は 2 | メンバーが、A と `version` 1 を指定して移動する | `ConflictError`（`STALE_VERSION`）を返す。A は通常の管理対象のまま |
| TC-content-trashContent-16 | A と B を確認した後、B が完全削除されている | 削除権限を持つメンバーが、A と B を一つの操作で移動する | `NotFoundError` を返す。B を示す。A は通常の管理対象のまま |
| TC-content-trashContent-18 | メンバーは、編集権限を持ち、削除権限を持たない | A をごみ箱へ移動する | `ForbiddenError`（`PERMISSION_MISSING`）を返す。不足する権限（`content:delete`）を添える。A は通常の管理対象のまま |
| TC-content-trashContent-19 | トークンは `content_manage` の許可と削除権限を持ち、範囲に A のモデルを含む。A は未公開 | トークンで、A と読み取ったときの `version` を指定して移動する | A はごみ箱にある。移動した実行者として、トークンと、トークンを指す値（名称または発行の経路）と、発行者または紐付くメンバーの名前を記録する |
| TC-content-trashContent-21 | B は公開中。公開中の R2 が、公開内容の必須の参照で B を参照する。メンバーは、削除権限を持ち、公開権限を持たない | メンバーが、B を、現在の `version` でごみ箱へ移動する | `ForbiddenError`（`PERMISSION_MISSING`）を返す。`WithdrawRejected` にしない。B は公開中のまま |
