# 一つの通知の送信結果を確かめる

「番号」は、通知を作る時点の宛先の、登録の順の番号（`WebhookDelivery.endpointOrder`）を指す。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-webhook-getNotificationDeliveries-01 | アプリ連携の権限を持たないメンバー。宛先 A、B をこの順に登録済み。そのメンバーが管理画面で行った公開で作られた通知で、A の送信は送信成功、B の送信は送信待ち | 公開の出力の操作の ID を指定する | 番号 1 を送信成功、番号 2 を送信待ちとして、番号の順に返す。宛先の URL、秘密情報、通知の内容を含まない |
| TC-webhook-getNotificationDeliveries-02 | アプリ連携の権限を持つメンバー。他のメンバーの操作で作られた、宛先 A と B を持つ通知 | その通知の操作の ID を指定する | 番号ごとの送信結果を返す。宛先の URL、秘密情報、通知の内容を含まない |
| TC-webhook-getNotificationDeliveries-04 | アプリ連携の権限を持つメンバー M1 と、持たないメンバー M2。他のメンバーの公開 P1 が確定し、`content.publication_changed` はまだ消費されていない。宛先が無い間に確定した公開 P2 の通知は無い。X は、どの操作の ID でもない | M1 と M2 が、それぞれ P1、P2、X の操作の ID を指定する | どの指定にも、`notCreated` を返す。エラーにしない。三つの結果は区別できない |
| TC-webhook-getNotificationDeliveries-06 | アプリ連携の権限を持たないメンバー。他のメンバーの操作で作られた通知 | その通知の操作の ID を指定する | `ForbiddenError`（`PERMISSION_MISSING`） |
| TC-webhook-getNotificationDeliveries-07 | アプリ連携の権限を持たないメンバー。トークンによる操作で作られた通知 | その通知の操作の ID を指定する | `ForbiddenError`（`PERMISSION_MISSING`） |
| TC-webhook-getNotificationDeliveries-10 | 操作を行ったメンバー。宛先 A、B をこの順に登録して作られた通知。その後、宛先 A が削除され、A の送信は `endpoint_deleted` の送信失敗になった | その通知の操作の ID を指定する | 番号 1 を送信失敗、番号 2 を B の送信結果として返す。B の番号は 2 のまま変わらない |
