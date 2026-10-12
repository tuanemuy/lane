# 保持方針の既存の回答への適用を確認する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-previewRetentionApplication-01 | F の保持方針は 30 日。F の回答は三件。S1 は保持方針 365 日で 10 日前に受付、S2 は保持方針 365 日で 40 日前に受付、S3 は保持方針 30 日で 5 日前に受付 | `intake:manage` を持つメンバーが、F への適用を確認する | 現在の保持方針（30 日）と、適用範囲 3 件、受付期間（S2 の受付日時から S3 の受付日時まで）、保持期限が変わる 2 件、次の自動整理で削除する 1 件を返す。どの回答の `retention` も変わらない |
| TC-form-previewRetentionApplication-02 | F の回答が無い | `intake:manage` を持つメンバーが、F への適用を確認する | 現在の保持方針と、適用範囲 0 件、受付期間なしを返す |
| TC-form-previewRetentionApplication-03 | F の保持方針は 365 日。F の回答 S1 は保持方針 365 日で 40 日前に受付 | `intake:manage` を持つメンバーが、保存の前の保持方針 30 日で、F への適用を確認する | 保持方針 30 日と、適用範囲 1 件、保持期限が変わる 1 件、次の自動整理で削除する 1 件を返す。F の保持方針は 365 日のまま |
| TC-form-previewRetentionApplication-04 | 削除済みフォーム F の `removed` の設定 | `intake:manage` を持つメンバーが、F への適用を確認する | `BusinessRuleError`（`FormRemoved`） |
| TC-form-previewRetentionApplication-06 | フォーム F の `active` の設定 | `intake:manage` を持たないメンバーが、F への適用を確認する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `intake:manage` |
