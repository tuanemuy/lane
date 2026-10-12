# 公開内容の変更の通知を作る

断りが無い限り、次を前提にする。

- 「公開が確定した」は、公開内容を変える操作が確定し、その操作の公開の変更の記録（`PublicationRecord`）が保存され、`content.publication_changed`（ワークスペースの ID と操作の ID）が記録されたことを指す
- 操作は、その `content.publication_changed` の消費。ワークスペースは、名簿で開設済みで、`active`
- 「送信の要求」は、`webhook.delivery_requested`（ペイロードは、通知の ID と宛先の ID）を指す

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-webhook-createChangeNotification-01 | 宛先 A、B をこの順に登録済み。そのワークスペースで公開が確定した | その操作の `content.publication_changed` を受け取る | 通知を一つと、A と B の送信を一つずつ作る。通知は、記録の通し番号、操作の ID、実行者、変更の一覧と参照元の一覧を持ち、通知の元は `content`、確定した日時はイベントの発生日時。二つの送信は送信待ち（最初の試行）で、次の試行の時刻は作成の日時。A の送信の番号（`endpointOrder`）は 1、B は 2。送信の要求を、送信ごとに一件、合わせて二件記録する。A にも B にも送らない。操作履歴に残さない |
| TC-webhook-createChangeNotification-02 | 宛先 A が登録済み。一つの操作で 150 件のコンテンツの公開内容を変えた | その操作の `content.publication_changed` を受け取る | 150 件の変更と、その参照元をすべて持つ通知を一つと、A の送信を一つ作る。送信の要求を一件記録する |
| TC-webhook-createChangeNotification-03 | 宛先 A が登録済み。公開中のコンテンツの一意な値を変えて公開した。そのコンテンツを、公開中の別のコンテンツが参照している | その操作の `content.publication_changed` を受け取る | 通知は、変更した対象のコンテンツID とモデルのキー、変更の種類 `updated`、一意な値の変更前と変更後、参照元のコンテンツID とモデルのキーを持つ |
| TC-webhook-createChangeNotification-04 | 宛先 A が登録済み。スキーマの反映が公開内容を変えた（記録の `operation` は `migrated`） | その操作の `content.publication_changed` を受け取る | 通知の元が `schemaApply` で、記録のスキーマの版を持つ通知を一つ作る |
| TC-webhook-createChangeNotification-06 | 宛先が一つも無いワークスペースで公開が確定した | `content.publication_changed` を受け取る | 通知も送信も作らない。送信の要求を記録しない。送信失敗も生じない |
| TC-webhook-createChangeNotification-09 | 通知を作成済み。その後、宛先 C が登録された | 同じ `content.publication_changed` をもう一度受け取る | 通知も送信も作らない。送信の要求を記録しない。そのワークスペースの、その操作の ID の通知は一つのまま変わらず、C の送信を持たない。作成済みの送信は変わらない |
| TC-webhook-createChangeNotification-10 | 宛先 A と B が登録済み | 同じ `content.publication_changed` を並行して二つの消費者が受け取る | その操作の通知は一つだけ作られ、送信の要求は、A と B に一件ずつだけ記録される。作成が確定しなかった消費者は、`ConflictError`（`ALREADY_EXISTS`）を投げて終える。そのイベントの再配送は、何も作らずに終える |
| TC-webhook-createChangeNotification-12 | 宛先 A と B が登録済み。公開の確定の後、イベントの消費の前に、宛先 B が削除され、宛先 C が登録された | その操作の `content.publication_changed` を受け取る | A と C の送信を作り、送信の要求を二件記録する。B の送信を作らず、B の送信失敗も生じない |
| TC-webhook-createChangeNotification-14 | 宛先 A と B が登録済み。消費者が宛先を読んだ後、作成の確定の前に、宛先 B の削除が確定した | その操作の `content.publication_changed` を受け取る | `ConflictError`（`STALE_VERSION`）を投げて終える。通知、送信、送信の要求は残らない。そのイベントの再配送は、A の送信だけを持つ通知を作る |
| TC-webhook-createChangeNotification-16 | 宛先が一つも無い間に、その操作の `content.publication_changed` を一度受け取り、通知を作らずに終えた。その後、宛先 A が登録された | 同じ `content.publication_changed` をもう一度受け取る（再配送） | A の送信を持つ通知を一つ作る。送信の要求を一件記録する |
