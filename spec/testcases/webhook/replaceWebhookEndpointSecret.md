# 通知の宛先の秘密情報を置き換える

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-webhook-replaceWebhookEndpointSecret-01 | アプリ連携の権限を持つメンバー。宛先がある | 新しい秘密情報を指定して置き換える | 正常に終わる。`webhook.settings_changed` の操作履歴を、項目 `secret`（`SecretChange`）だけとともに残す。操作履歴は、置き換え前後の秘密情報の値を含まない |
| TC-webhook-replaceWebhookEndpointSecret-02 | アプリ連携の権限を持つメンバー。宛先がある | 秘密情報を入力せずに置き換える | `ValidationError`（`InvalidWebhookSecret`）。宛先は変わらない。操作履歴を追記しない |
| TC-webhook-replaceWebhookEndpointSecret-03 | アプリ連携の権限を持つメンバー。他のメンバーの操作で既に削除された宛先 | その宛先の秘密情報を置き換える | `NotFoundError`。操作履歴を追記しない |
| TC-webhook-replaceWebhookEndpointSecret-04 | アプリ連携の権限を持たないメンバー。宛先がある | 秘密情報を置き換える | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限（`integration:manage`）を示す。宛先は変わらない。操作履歴を追記しない |
