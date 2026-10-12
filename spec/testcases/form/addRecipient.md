# 通知先を追加する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-addRecipient-01 | フォーム F の `active` の設定。通知先 A がある | `notification:manage` を持つメンバー M が、F にメールアドレス B を追加する | 変更後の通知先 A、B をこの順に返す。B は有効。操作履歴に `form.recipients_changed` が一件残る。実行者は M、対象は F で、名称は F のキー |
| TC-form-addRecipient-02 | 削除済みフォーム F の `removed` の設定。F の回答が残る | `notification:manage` を持つメンバーが、F にメールアドレス B を追加する | B を有効な通知先として加えた通知先を返す。操作履歴の対象の名称は、削除の時点のキー |
| TC-form-addRecipient-03 | フォーム F の `active` の設定。F の受付済みの回答 S がある | `notification:manage` を持つメンバーが、F にメールアドレス B を追加する | S の `notifiedTo` と、S の通知は変わらない |
| TC-form-addRecipient-04 | フォーム F の `active` の設定。メンバーが設定を読んだ後、確定までに、スキーマの反映が始まる（適用中になる）、または F を変えないスキーマの反映が確定する | `notification:manage` を持つメンバーが、F にメールアドレス B を追加する | どちらの場合も、B を加えた通知先を返す。`UnavailableError`（`SCHEMA_APPLYING`）にも `ConflictError`（`SCHEMA_CHANGED`）にもしない |
| TC-form-addRecipient-07 | フォーム F の `active` の設定。通知先 A がある | `notification:manage` を持つメンバーが、F に A を追加する | `BusinessRuleError`（`DuplicateRecipient`）。通知先は変わらない。操作履歴は残らない |
| TC-form-addRecipient-09 | フォーム F の `active` の設定 | `intake:manage` だけを持つメンバーが、通知先を追加する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `notification:manage`。通知先は変わらない。操作履歴は残らない |
| TC-form-addRecipient-11 | フォーム F の `active` の設定。通知先が `RecipientList.max` 件ある。有効な通知先と無効な通知先を含む | `notification:manage` を持つメンバーが、F にメールアドレスを追加する | `BusinessRuleError`（`RecipientLimitReached`）。上限を添える。通知先は変わらない。操作履歴は残らない |
