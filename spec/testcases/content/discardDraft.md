# 下書きを破棄する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-discardDraft-01 | A は公開中で、下書きを持つ。`version` は 3 | 編集権限を持つメンバーが、A と `version` 3 を指定して破棄する | A は下書きを持たない。最新の内容は公開内容になる。`version` 4 を返す。公開内容と、公開内容の更新日時は変わらない。破棄した下書きは、下書きだった版として履歴に残る。版にした操作は下書きの破棄、実行者は破棄したメンバー。`content.publication_changed` を記録しない |
| TC-content-discardDraft-02 | A のモデルは非推奨。A は公開中で、下書きを持つ | 編集権限を持つメンバーが、A の下書きを破棄する | 破棄する |
| TC-content-discardDraft-03 | メンバーが A の差分を `version` 3 で確認した後、ほかの操作で A の下書きが変わり、`version` は 4 | メンバーが、A と `version` 3 を指定して破棄する | `ConflictError`（`STALE_VERSION`）を返す。A の下書きは残る |
| TC-content-discardDraft-04 | メンバーが A の差分を `version` 3 で確認した後、ほかのメンバーが A を公開した。A は下書きを持たず、`version` は 4 | メンバーが、A と `version` 3 を指定して破棄する | `ConflictError`（`STALE_VERSION`）を返す。`ContentHasNoDraft` にしない。A は変わらない |
| TC-content-discardDraft-05 | A は公開中で、下書きを持たない | 編集権限を持つメンバーが、A と現在の `version` を指定して破棄する | `BusinessRuleError`（`ContentHasNoDraft`）を返す。A は変わらない |
| TC-content-discardDraft-06 | A は未公開 | 編集権限を持つメンバーが、A と現在の `version` を指定して破棄する | `BusinessRuleError`（`ContentNotPublished`）を返す。A の下書きは残る |
| TC-content-discardDraft-07 | A はごみ箱にある | 編集権限を持つメンバーが、A と現在の `version` を指定して破棄する | `BusinessRuleError`（`ContentInTrash`）を返す。A は変わらない |
| TC-content-discardDraft-08 | A は、メンバーが確認した後に完全削除されている | 編集権限を持つメンバーが、A の下書きを破棄する | `NotFoundError` を返す |
| TC-content-discardDraft-10 | メンバーは編集権限を持たない | A の下書きを破棄する | `ForbiddenError`（`PERMISSION_MISSING`）を返す。不足する権限（`content:edit`）を添える。A の下書きは残る |
| TC-content-discardDraft-11 | トークンは `content_manage` の許可と編集権限を持ち、範囲に A のモデルを含む。A は公開中で、下書きを持つ | トークンで、A と読み取ったときの `version` を指定して破棄する | 破棄する。版にした実行者として、トークンと、トークンを指す値（名称または発行の経路）と、発行者または紐付くメンバーの名前を記録する |
