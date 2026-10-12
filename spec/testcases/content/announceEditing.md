# 編集中であることを知らせる

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-announceEditing-01 | A に、メンバー甲の編集中の記録は無い | 編集権限を持つ甲が、時点 t に、A を編集中であることを知らせる | 甲の記録ができる。名前は甲のアカウントの名前、編集を始めた日時は t、期限は t から `EditingPresence.ttlMs` 後。ほかのメンバーが A の編集中のメンバーを確認すると、甲が現れる |
| TC-content-announceEditing-02 | A に、甲の有効な記録があり、編集を始めた日時は t0 | 甲が、時点 t に、A を編集中であることを知らせる | 甲の記録の、編集を始めた日時は t0 のまま、期限は t から `EditingPresence.ttlMs` 後になる |
| TC-content-announceEditing-03 | A に、甲の失効した記録がある | 甲が、時点 t に、A を編集中であることを知らせる | 甲の記録の、編集を始めた日時は t、期限は t から `EditingPresence.ttlMs` 後になる |
| TC-content-announceEditing-04 | A に、甲の有効な記録がある | 編集権限を持つメンバー乙が、A の下書きを、読み取ったときの `version` で保存する | 乙の保存は成功する |
| TC-content-announceEditing-05 | メンバーは編集権限を持たない | A を編集中であることを知らせる | `ForbiddenError`（`PERMISSION_MISSING`）を返す。不足する権限（`content:edit`）を添える。記録は作られない |
| TC-content-announceEditing-06 | 指定したコンテンツID のコンテンツが無い | 編集権限を持つメンバーが、そのコンテンツを編集中であることを知らせる | `NotFoundError` を返す。記録は作られない |
| TC-content-announceEditing-07 | C はごみ箱にある | 編集権限を持つメンバーが、C を編集中であることを知らせる | `BusinessRuleError`（`ContentInTrash`）を返す。記録は作られない |
