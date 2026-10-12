# 通知の宛先を一覧する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-webhook-listWebhookEndpoints-01 | アプリ連携の権限を持つメンバー。宛先が二つ登録済み | 一覧する | 二つの宛先を返す。宛先ごとに、ID、URL、秘密情報が登録済みであることを示す。出力は秘密情報の値を含まない |
| TC-webhook-listWebhookEndpoints-02 | アプリ連携の権限を持つメンバー。宛先は未登録 | 一覧する | 空の一覧を返す |
| TC-webhook-listWebhookEndpoints-04 | アプリ連携の権限を持たないメンバー | 一覧する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限（`integration:manage`）を示す |
