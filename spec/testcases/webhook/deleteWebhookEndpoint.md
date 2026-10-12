# 通知の宛先を削除する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-webhook-deleteWebhookEndpoint-01 | アプリ連携の権限を持つメンバー。宛先がある | その宛先を削除する | 正常に終わる。宛先は一覧に現れない。`webhook.settings_changed` の操作履歴を、項目 `url`（変更前は削除した宛先の URL、変更後は `null`）とともに残す |
| TC-webhook-deleteWebhookEndpoint-02 | アプリ連携の権限を持つメンバー。宛先と、その宛先の送信を持つ作成済みの通知がある | その宛先を削除する | 通知は削除されず、送信結果も変わらない |
| TC-webhook-deleteWebhookEndpoint-03 | アプリ連携の権限を持つメンバー。他のメンバーの操作で既に削除された宛先 | その宛先を削除する | `NotFoundError`。操作履歴を追記しない |
| TC-webhook-deleteWebhookEndpoint-05 | アプリ連携の権限を持たないメンバー。宛先がある | その宛先を削除する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限（`integration:manage`）を示す。宛先は残る。操作履歴を追記しない |
