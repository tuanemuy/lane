# コンテンツを複製する入力を求める

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-prepareDuplicateInput-01 | A は公開中で、下書きを持つ。A のモデルは、一意な項目 `slug` を持つ | 編集権限を持つメンバーが、A を複製元にして入力を求める | A の下書きの値を返す。`slug` は未設定で、未設定にした項目のキーとして `slug` を返す。コンテンツは作られない。A は変わらない |
| TC-content-prepareDuplicateInput-02 | A は公開中で、下書きを持たない | 編集権限を持つメンバーが、A を複製元にして入力を求める | A の公開内容の値を返す |
| TC-content-prepareDuplicateInput-03 | A の最新の内容が、非推奨の項目の値、非推奨の選択肢の選択、非推奨の種類のブロックを持つ | 編集権限を持つメンバーが、A を複製元にして入力を求める | 非推奨の項目の値、非推奨の選択肢の選択、非推奨の種類のブロックを取り除いた値を返す |
| TC-content-prepareDuplicateInput-04 | A はごみ箱にある | 編集権限を持つメンバーが、A を複製元にして入力を求める | `BusinessRuleError`（`ContentInTrash`）を返す |
| TC-content-prepareDuplicateInput-05 | A は完全削除されている | 編集権限を持つメンバーが、A を複製元にして入力を求める | `NotFoundError` を返す |
| TC-content-prepareDuplicateInput-06 | メンバーは編集権限を持たない | A を複製元にして入力を求める | `ForbiddenError`（`PERMISSION_MISSING`）を返す。不足する権限（`content:edit`）を添える |
