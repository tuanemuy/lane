# 通知の宛先の URL を変更する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-webhook-changeWebhookEndpointUrl-01 | アプリ連携の権限を持つメンバー。URL が `https://old.example.com/hook` の宛先がある | URL を `https://new.example.com/hook` に変更する | 変更した宛先を返す。宛先の URL は新しい URL になる。`webhook.settings_changed` の操作履歴を、項目 `url`（変更前は `https://old.example.com/hook`、変更後は `https://new.example.com/hook`）だけとともに残す |
| TC-webhook-changeWebhookEndpointUrl-02 | アプリ連携の権限を持つメンバー。宛先がある | 現在と同じ URL に変更する | 成功として現在の宛先を返す。宛先は変わらず、`version` は進まない。操作履歴を追記しない |
| TC-webhook-changeWebhookEndpointUrl-03 | アプリ連携の権限を持つメンバー。宛先がある | 形式が正しくない URL に変更する | `ValidationError`（`InvalidWebhookUrl`）。宛先は変わらない。操作履歴を追記しない |
| TC-webhook-changeWebhookEndpointUrl-04 | アプリ連携の権限を持つメンバー。他のメンバーの操作で既に削除された宛先 | その宛先の URL を変更する | `NotFoundError`。操作履歴を追記しない |
| TC-webhook-changeWebhookEndpointUrl-06 | アプリ連携の権限を持たないメンバー。宛先がある | URL を変更する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限（`integration:manage`）を示す。宛先は変わらない。操作履歴を追記しない |
