# WebhookSender

契約は [webhook.md](../../domains/webhook.md) の「ポート」の `WebhookSender` と、[common.md](../../domains/common.md) の、外部の相手を持つポートの適合テストの前提が定める。

## 前提

- `message`（`OutboundNotification`）は、`ChangeNotification.create` で作った通知の `ChangeNotification.payload(notification)` と、`WebhookUrl`、`SecretCipher.protect` で保護して `ProtectedWebhookSecret.wrap` で包んだ秘密情報から組み立てる
- 「受信側」は、受け取った要求（メソッド、ヘッダー、本文のバイト列）を記録し、ケースが指定する応答を返す、試験用の宛先を指す。`message.url` は受信側の URL にする
- `at` は、断りが無ければ、秒未満を持たない固定の日時にする
- 待ち時間の上限は `DeliveryPolicy.timeoutMs`

## 要求の形

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookSender-01 | 受信側が状態 200 を返す | `send(message, at)` を呼ぶ | 受信側は、`message.url` への `POST` の要求を 1 回だけ受け取る |
| TC-ports-webhookSender-02 | 受信側が状態 200 を返す | `send(message, at)` を呼ぶ | 要求の `Content-Type` ヘッダーは `application/json` |
| TC-ports-webhookSender-03 | 受信側が状態 200 を返す。通知は、変更が 2 件（一意な値の変更を持つ）、参照元が 2 件 | `send(message, at)` を呼ぶ | 本文は UTF-8 の JSON で、`notificationId`、`sequence`、`schemaVersion`、`changes`、`referrers` だけを持つ。`changes` の各変更は、`contentId`、`model`、`change`、`uniqueValues`（`field`、`before`、`after`）を、`referrers` の各参照元は、`contentId`、`model` を、`message.payload` と同じ値で持つ。`changes`、`uniqueValues`、`referrers` の要素は、`message.payload` と同じ順に並ぶ |
| TC-ports-webhookSender-04 | 受信側が状態 200 を返す。通知は、変更が 150 件、参照元が 600 件 | `send(message, at)` を呼ぶ | 本文は、150 件の `changes` と 600 件の `referrers` をすべて持つ。署名は、受け取った本文のバイト列から計算した値と一致する |
| TC-ports-webhookSender-05 | 受信側が状態 200 を返す。通知の `source` は `content`。一意な値の変更の `before` が `null` | `send(message, at)` を呼ぶ | 本文は `schemaVersion` の項目を持ち、値は `null`。その変更は `before` の項目を持ち、値は `null` |
| TC-ports-webhookSender-08 | 受信側が状態 200 を返す。通知は、参照元が 0 件、変更の `uniqueValues` が 0 件 | `send(message, at)` を呼ぶ | 本文は `referrers` と `uniqueValues` の項目を持ち、値は空の配列 |
| TC-ports-webhookSender-09 | 受信側が状態 200 を返す。一意な値の変更の `before` と `after` が、ASCII でない文字を含む | `send(message, at)` を呼ぶ | 受け取った本文を UTF-8 として読んだ JSON の値は、`before` と `after` と同じ文字列。署名は、受け取った本文のバイト列から計算した値と一致する |
| TC-ports-webhookSender-10 | 受信側が状態 200 を返す。一意な値の変更の `after` が、`"`、`\`、改行を含む | `send(message, at)` を呼ぶ | 受け取った本文を JSON として読んだ値は、`after` と同じ文字列。署名は、受け取った本文のバイト列から計算した値と一致する |
| TC-ports-webhookSender-12 | 受信側が状態 200 を返す | `send(message, at)` を呼ぶ | 要求のヘッダーと本文は、保護を解いた秘密情報を含まない |

## 時刻と署名

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookSender-13 | 受信側が状態 200 を返す | `send(message, at)` を呼ぶ | `X-Lane-Timestamp` ヘッダーは、`at` の UNIX 時刻（秒）の 10 進表記 |
| TC-ports-webhookSender-14 | 受信側が状態 200 を返す。`at` は、秒未満が 999 ミリ秒の日時 | `send(message, at)` を呼ぶ | `X-Lane-Timestamp` ヘッダーは、`at` の秒未満を切り捨てた UNIX 時刻（秒） |
| TC-ports-webhookSender-15 | 受信側が状態 200 を返す。`at` は、`send` を呼ぶ時点より 1 日前 | `send(message, at)` を呼ぶ | `X-Lane-Timestamp` ヘッダーは `at` の値。署名は、その値での計算と一致する |
| TC-ports-webhookSender-16 | 受信側が状態 200 を返す。秘密情報は ASCII の文字列 | `send(message, at)` を呼ぶ | `X-Lane-Signature` ヘッダーは、受け取った本文のバイト列、`X-Lane-Timestamp` の値、秘密情報から、[webhook.md](../../domains/webhook.md) の「WebhookSender」の署名の規則で求めた値と一致する |
| TC-ports-webhookSender-17 | 受信側が状態 200 を返す | `send(message, at)` を呼ぶ | `X-Lane-Signature` ヘッダーは、16 進の小文字 64 文字 |
| TC-ports-webhookSender-18 | 受信側が状態 200 を返す。秘密情報は ASCII でない文字を含む | `send(message, at)` を呼ぶ | 署名は、秘密情報の UTF-8 のバイト列を鍵にした計算と一致する |
| TC-ports-webhookSender-20 | 受信側が状態 200 を返す | 同じ `message` を、秒が違う二つの `at` で `send` する | 二つの要求の `X-Lane-Timestamp` はそれぞれの `at` の値で、署名は互いに違い、それぞれの時刻での計算と一致する |
| TC-ports-webhookSender-21 | 受信側が状態 200 を返す | `payload` と `at` が同じで、秘密情報だけが違う二つの `message` を `send` する | 二つの要求の署名は互いに違い、それぞれの秘密情報での計算と一致する |

## 結果の判定

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookSender-22 | 受信側が状態 200 を返す場合、204 を返す場合、299 を返す場合 | それぞれ `send(message, at)` を呼ぶ | どれも `{ status: "delivered" }` を返す |
| TC-ports-webhookSender-25 | 受信側が状態 302 と、別の受信側への転送先を返す | `send(message, at)` を呼ぶ | `failed` を返す。`failure` は `undelivered`（`rejected`）と `statusCode` 302。転送先の受信側は要求を受け取らない |
| TC-ports-webhookSender-27 | 受信側が状態 400 を返す場合、404 を返す場合、500 を返す場合 | それぞれ `send(message, at)` を呼ぶ | どれも `failed` を返す。`failure` は `undelivered`（`rejected`）と、受信側が返した状態の `statusCode` |
| TC-ports-webhookSender-30 | 受信側が、情報の応答（状態 103）に続けて状態 200 を返す | `send(message, at)` を呼ぶ | `{ status: "delivered" }` を返す |
| TC-ports-webhookSender-32 | 受信側が、要求を受け取ったまま `DeliveryPolicy.timeoutMs` を超えて応答を返さない場合と、情報の応答（状態 103）を返した後、`DeliveryPolicy.timeoutMs` を超えて最終の応答を返さない場合 | それぞれ `send(message, at)` を呼ぶ | どちらも、例外を投げず、`failed` を返す。`failure` は `undelivered`（`timeout`） |
| TC-ports-webhookSender-34 | 受信側が、`DeliveryPolicy.timeoutMs` 以内に状態 200 を返す場合と状態 500 を返す場合。どちらも、応答の本文を `DeliveryPolicy.timeoutMs` を超えて送り終えない | それぞれ `send(message, at)` を呼ぶ | 本文の完了を待たずに、200 では `{ status: "delivered" }` を、500 では `failed`（`undelivered`、`rejected`、`statusCode` 500）を返す |
| TC-ports-webhookSender-36 | `message.url` のホスト名が解決できない場合、ポートが接続を拒否する場合、`https` で受信側の証明書を検証できない場合 | それぞれ `send(message, at)` を呼ぶ | どれも、例外を投げず、`failed` を返す。`failure` は `undelivered`（`unreachable`）。証明書を検証できない受信側は、要求の本文を受け取らない |
| TC-ports-webhookSender-39 | `message.url` が `http` の受信側と、`https` で検証できる証明書の受信側が、状態 200 を返す | それぞれ `send(message, at)` を呼ぶ | どちらも `{ status: "delivered" }` を返す |

## 送る回数

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookSender-41 | 受信側が状態 500 を返す場合と、`DeliveryPolicy.timeoutMs` を超えて応答を返さない場合 | それぞれ `send(message, at)` を一度呼ぶ | どちらも、受信側は要求を 1 回だけ受け取る |
| TC-ports-webhookSender-43 | 受信側が状態 200 を返す | 同じ `message` と `at` で `send` を二度呼ぶ | 受信側は要求を 2 回受け取る。二つの要求の `X-Lane-Timestamp` は同じ値で、署名は、それぞれ受け取った本文のバイト列から計算した値と一致する。どちらの呼び出しも `delivered` を返す |

## 並行性

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookSender-45 | 受信側 A が状態 200、受信側 B が状態 500 を返す | A 宛てと B 宛ての `send` を並行して呼ぶ | A 宛ては `delivered`、B 宛ては `rejected` と 500 の `failed` を返す。それぞれの受信側は、自分宛ての本文と、自分の秘密情報での署名を 1 回ずつ受け取る |
| TC-ports-webhookSender-46 | 受信側 A が `DeliveryPolicy.timeoutMs` を超えて応答を返さず、受信側 B が状態 200 を返す | A 宛てと B 宛ての `send` を並行して呼ぶ | B 宛ては、A 宛ての結果を待たずに `delivered` を返す。A 宛ては `timeout` の `failed` を返す |

## UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookSender-49 | 受信側が状態 200 を返す | `UnitOfWorkProvider.run` の `fn` の中で `send(message, at)` を呼び、その後に `fn` が例外を投げる | `run` はその例外を投げる。受信側は要求を 1 回受け取っている。送信は取り消されず、UnitOfWork と確定を共にしない |

## 秘密情報

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-webhookSender-50 | `message.secret` が、`SecretCipher.protect` が返した値でない。受信側が状態 200 を返す | `send(message, at)` を呼ぶ | 例外を投げず、`failed` を返す。`failure` は `notSent`（`secret_unavailable`）。受信側は要求を受け取らない |
| TC-ports-webhookSender-51 | 受信側が状態 200 を返す場合と、状態 500 を返す場合 | それぞれ `send(message, at)` を呼ぶ | 戻り値は、200 では `status` だけを、500 では `failure` の `kind`、`reason`、`statusCode` だけを持つ。どちらも、保護を解いた秘密情報を含まない |
