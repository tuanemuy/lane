# 通知の署名の確認（verifyNotification）

`packages/client` の `verifyNotification`。契約は [app-api.md](../../usecases/app-api.md) の「クライアント」と、[webhook.md](../../domains/webhook.md) の「WebhookSender」の送信の規則が定める。CMS へ接続しない。断りが無い限り、K は秘密情報、B は通知の内容（`NotificationPayload`）の JSON の本文のバイト列、T は `X-Lane-Timestamp` の値、S は、K、T、B から「WebhookSender」の送信の規則で計算した署名。「確かめる」は、K、B、T、S を渡して呼ぶことを指す。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-client-verifyNotification-01 | なし | 確かめる | B の通知の内容（`NotificationPayload`）を返す |
| TC-client-verifyNotification-02 | S は、K と違う秘密情報で計算した値 | 確かめる | 確かめられないことを返す。通知の内容を返さない |
| TC-client-verifyNotification-05 | T は、現在の 30 日前の時刻 | 確かめる | 通知の内容を返す。時刻の古さで拒否しない |
| TC-client-verifyNotification-06 | `WebhookSender` が、秘密情報 K の宛先へ送った要求がある | その要求の本文のバイト列と、`X-Lane-Timestamp`、`X-Lane-Signature` ヘッダーの値を、K とともに渡す | 送った `payload` と等しい通知の内容を返す |
