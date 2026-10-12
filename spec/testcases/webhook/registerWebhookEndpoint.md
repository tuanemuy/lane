# 通知の宛先を登録する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-webhook-registerWebhookEndpoint-01 | アプリ連携の権限を持つメンバー。宛先は未登録 | 宛先の URL と秘密情報を指定して登録する | 登録した宛先の ID、URL と、秘密情報が登録済みであることを返す。出力は秘密情報の値を含まない。`webhook.settings_changed` の操作履歴を、対象の名称を宛先の URL とし、項目 `url`（変更前は `null`、変更後は登録した URL）と `secret`（`SecretChange`）とともに残す。操作履歴は秘密情報の値を含まない |
| TC-webhook-registerWebhookEndpoint-02 | アプリ連携の権限を持つメンバー。宛先が一つ登録済み | 二つ目の宛先を登録する | 宛先を登録する。ワークスペースは二つの宛先を持つ |
| TC-webhook-registerWebhookEndpoint-03 | アプリ連携の権限を持つメンバー。宛先 A だけに送信を持つ、作成済みの通知がある | 宛先 B を登録する | 作成済みの通知の送信は増えない。通知は宛先 A の送信だけを持つ |
| TC-webhook-registerWebhookEndpoint-04 | アプリ連携の権限を持つメンバー | 形式が正しくない URL を指定し、秘密情報を入力せずに登録する | `ValidationError`。項目ごとの理由として、URL に `InvalidWebhookUrl`、秘密情報に `InvalidWebhookSecret` を示す。宛先も操作履歴も残らない |
| TC-webhook-registerWebhookEndpoint-05 | アプリ連携の権限を持つメンバー。宛先が上限（`DeliveryPolicy.maxEndpoints` 件）まで登録済み | もう一件の宛先を登録する | `BusinessRuleError`（`EndpointLimitReached`）。上限を添える。宛先の件数は変わらない。操作履歴を追記しない |
| TC-webhook-registerWebhookEndpoint-06 | アプリ連携の権限を持たないメンバー（編集権限とメンバー管理の権限を持つ） | 宛先を登録する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限（`integration:manage`）を示す。宛先も操作履歴も残らない |
