# コンテンツの保持方針を確認する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-getRetentionPolicy-01 | ワークスペースは、保持方針を変更していない | ワークスペース設定の権限を持つメンバーが、保持方針を確認する | 履歴を保持する日数とごみ箱のコンテンツを保持する日数として、`ContentRetentionPolicy.standard` の日数を返す。指定できる範囲として、`RetentionDays.min` 日以上 `RetentionDays.max` 日以下を返す |
| TC-content-getRetentionPolicy-02 | 保持方針を、履歴 90 日、ごみ箱 7 日に変更している | ワークスペース設定の権限を持つメンバーが、保持方針を確認する | 履歴を保持する日数 90、ごみ箱のコンテンツを保持する日数 7 を返す |
| TC-content-getRetentionPolicy-03 | メンバーは、ワークスペース設定の権限を持たない | 保持方針を確認する | `ForbiddenError`（`PERMISSION_MISSING`）を返す。不足する権限（`workspace:manage`）を添える |
