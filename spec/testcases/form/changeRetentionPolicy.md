# 回答の保持方針を変更する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-changeRetentionPolicy-01 | フォーム F の `active` の設定。保持方針は 365 日 | `intake:manage` を持つメンバー M が、F の保持する日数を 30 にする | 変更後の保持方針（30 日）を返す。F の保持方針は 30 日になる。操作履歴に `form.retention_policy_changed` が一件残る。実行者は M、対象は F |
| TC-form-changeRetentionPolicy-02 | F の保持方針は 365 日。保持方針が 365 日の回答 S がある | `intake:manage` を持つメンバーが、F の保持する日数を 30 にする | S の `retention` は 365 日のまま |
| TC-form-changeRetentionPolicy-04 | フォーム F の `active` の設定。保持方針は 365 日 | `intake:manage` を持つメンバーが、指定できる範囲の外の保持する日数を入力する | `ValidationError`（`InvalidRetentionPolicy`）。F の保持方針は 365 日のまま。操作履歴は残らない |
| TC-form-changeRetentionPolicy-05 | 削除済みフォーム F の `removed` の設定 | `intake:manage` を持つメンバーが、F の保持方針を変更する | `BusinessRuleError`（`FormRemoved`）。操作履歴は残らない |
| TC-form-changeRetentionPolicy-07 | フォーム F の `active` の設定。保持方針は 365 日 | `notification:manage` だけを持つメンバーが、F の保持方針を変更する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `intake:manage`。F の保持方針は変わらない。操作履歴は残らない |
| TC-form-changeRetentionPolicy-08 | フォーム F の `active` の設定。保持方針は 365 日 | `intake:manage` を持つメンバーが、F の保持する日数を 365 にする | 現在の保持方針（365 日）を返す。保存せず、F の設定の `version` は変わらない。操作履歴は残らない |
