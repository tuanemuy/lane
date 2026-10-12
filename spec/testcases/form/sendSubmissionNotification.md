# 回答の通知を送る

断りが無い限り、次を前提にする。

- S はフォーム F の回答。N は S の通知で、`notifiedTo` は A と B、`attempt` は `initial`、状態は `waiting`
- F の有効な通知先は A と B。送信接続設定は登録済み。ワークスペースは利用できる
- 操作は、S の `form.submission_accepted` の消費

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-sendSubmissionNotification-01 | `MailSender` は A と B の受理を返す | イベントを消費する | `MailSender` を一度呼び、宛先は A と B。`MailSender` を呼ぶ時点で、N は `sending` で、`recipients` は A と B。N は送信成功（`succeeded`）になり、A と B の結果は `accepted`。S は変わらない |
| TC-form-sendSubmissionNotification-02 | S の受付時のフォームは、表示名称を持つテキストの項目 `name`、表示名称を持つ選択肢の単一選択 `plan` と複数選択 `topics`、未入力のテキストの項目 `note`、問い合わせ対象の項目 `product` を、この順に持つ。S の受付時の通知定義の本文は、全項目の一覧（`allFields`）、`note` の未入力の出し分け（`ifAnswered`）、`product` の送信時の情報（表示名称、モデルのキー、コンテンツID、一意な値）、受付日時を持つ | イベントを消費する | 送るメールの本文は、受付時の項目を、受付時の表示名称と順序で一行ずつ持つ。`plan` は選択肢の表示名称、`topics` は選んだ選択肢の表示名称を定義の順に `, ` で列挙した文字列、`note` は値が空の行になる。`note` の出し分けの部分を出さない。`product` の送信時の情報と、S の受付日時を持つ |
| TC-form-sendSubmissionNotification-03 | S の受付時の通知定義の本文は、受付時のフォームに無い項目 `company` を参照する（通知定義に不整合が残るまま受け付けた回答）。`MailSender` は受理を返す | イベントを消費する | 通知を送る。送るメールの本文は、値の代わりに `[不整合な参照: company]` を持つ。N は送信成功になる |
| TC-form-sendSubmissionNotification-04 | S の受付時の通知定義の返信先は、受付時のフォームに無い項目を指す。`MailSender` は受理を返す | イベントを消費する | 通知を送る。送るメールは返信先を持たない（`replyTo` は `null`）。N は送信成功になる |
| TC-form-sendSubmissionNotification-05 | 受付の後に、F の通知定義と、F の項目の表示名称を変えた | イベントを消費する | 送るメールは、S が持つ受付時の通知定義と受付時のフォームで描画した件名と本文を持つ |
| TC-form-sendSubmissionNotification-06 | 受付の後に、送信接続設定の接続先と、送信者のアドレスの接続条件を更新した | イベントを消費する | `MailSender` は更新後の送信接続設定を受け取る |
| TC-form-sendSubmissionNotification-07 | 試験用の `MailSender` は、`send` の呼び出しの中で（処理が送信接続設定を読み終えた後に）、「送信接続設定を更新する」を確定させてから、A と B の受理を返す | イベントを消費する | `MailSender` を呼ぶのは一度だけで、受け取る送信接続設定は更新前の値。更新後の設定で送り直さない。N は送信成功になる |
| TC-form-sendSubmissionNotification-08 | `MailSender` は A の受理と、B の `failed`（`recipientRejected`）を返す | イベントを消費する | エラーにしない。N は送信失敗（`failed`、`delivery`）になり、対処の状態は未対処。A は `accepted`、B は `failed` と理由 `recipientRejected`。`MailSender` を呼ぶのは一度だけで、再試行しない。S は変わらない |
| TC-form-sendSubmissionNotification-09 | 受付の後に、B を無効にし、通知先 C を追加した | イベントを消費する | `MailSender` の宛先は A だけ。B と C へ送らない |
| TC-form-sendSubmissionNotification-10 | 受付の後に、F の有効な通知先が無くなった | イベントを消費する | `MailSender` を呼ばない。N は送信失敗（`failed`、`noRecipients`）になり、対処の状態は未対処 |
| TC-form-sendSubmissionNotification-11 | 受付の後に、F を受付停止にした | イベントを消費する | `MailSender` の宛先は A と B。通知を取り消さない |
| TC-form-sendSubmissionNotification-12 | 受付の後に、スキーマの反映で F を削除した。`removed` の設定の有効な通知先は A と B | イベントを消費する | `MailSender` の宛先は A と B。通知を取り消さない |
| TC-form-sendSubmissionNotification-13 | N は 1 回目の再試行の送信待ち（`attempt` は `retry` の `round` 1、状態は `waiting`）。A は `accepted`、B は `failed`。受付の後に通知先 C を追加した。`MailSender` は受理を返す | N の `form.notification_retry_requested`（`round` 1）を消費する | `MailSender` の宛先は B と C。A へ送らない。N は送信成功（`succeeded`）になる |
| TC-form-sendSubmissionNotification-14 | N は 1 回目の再試行の送信待ち（`attempt` は `retry` の `round` 1、状態は `waiting`）。A の結果は `accepted`、B の結果は `failed`。再試行の前は対処済みだった。`MailSender` は B の `failed`（`connection`）を返す | N の `form.notification_retry_requested`（`round` 1）を消費する | N は送信失敗（`failed`、`delivery`）になり、対処の状態は未対処で始まる |
| TC-form-sendSubmissionNotification-15 | N は最初の通知が送信成功（`succeeded`） | 同じイベントを再び消費する | `MailSender` を呼ばない。N は変わらない |
| TC-form-sendSubmissionNotification-16 | N は最初の通知の送信中（`attempt` は `initial`、状態は `sending`）で、送信を始めてから `SubmissionNotification.stallAfterMs` が過ぎていない（前の処理が送信の途中） | 同じイベントを再び消費する | `MailSender` を呼ばない。何も書かず、N は `sending` のまま変わらない。前の処理が、その後に結果を保存できる |
| TC-form-sendSubmissionNotification-17 | N は最初の通知の送信中（`attempt` は `initial`、状態は `sending`）のまま、送信を始めてから `SubmissionNotification.stallAfterMs` 以上が過ぎた（前の処理が、送信の後、結果の保存の前に中断した） | 同じイベントを再び消費する | `MailSender` を呼ばない。送信中だった宛先（`recipients`）の結果は `failed`（`unconfirmed`）になり、N は送信失敗（`failed`、`delivery`）、対処の状態は未対処 |
| TC-form-sendSubmissionNotification-18 | N は 1 回目の再試行の送信待ち（`attempt` は `retry` の `round` 1、状態は `waiting`） | `form.submission_accepted` を消費する | `MailSender` を呼ばない。N は変わらない |
| TC-form-sendSubmissionNotification-19 | 同じイベントを二つの処理が並行して消費する。`MailSender` は A と B の受理を返す | イベントを消費する | 一方だけが `sending` の通知を確定する。`sending` の保存が競合した処理は、エラーを投げて終え、何も書かず、`MailSender` を呼ばない。`MailSender` を呼ぶのは一度だけ。N は送信成功になる。競合した処理のイベントが再び届いても、何も送らない |
| TC-form-sendSubmissionNotification-20 | `sending` の通知の確定の後、結果を保存する UnitOfWork が通知を読み直した後、確定までに、別の処理が N を保存する | イベントを消費する | エラーを投げて終える。メールは送っている。この処理の結果は記録されず、N は別の処理が保存した状態のまま |
| TC-form-sendSubmissionNotification-24 | `sending` の通知の確定の後、結果の保存の確定までに、ワークスペースの利用終了が始まる（確定が `WORKSPACE_NOT_ACTIVE` で拒否される） | イベントを消費する | エラーにせずに終える。メールは送っている。結果を記録せず、N は `sending` のまま残る |
| TC-form-sendSubmissionNotification-25 | `sending` の通知の確定の後、結果の保存の前に、S を「回答を削除する」で削除した | イベントを消費する | 送信は取り消さない。結果を保存せずに終える。S と N は無いまま |
| TC-form-sendSubmissionNotification-26 | 前の処理が、`sending` の通知の確定の前に中断した（N は `waiting` のまま。メールは送っていない）。`MailSender` は A と B の受理を返す | 同じイベントを再び消費する | `MailSender` を一度呼び、宛先は A と B。N は送信成功になる |
| TC-form-sendSubmissionNotification-27 | `MailSender` は A と B の受理を返す。`sending` の通知の確定と送信の後、結果を保存する UnitOfWork が通知を読むまでに `SubmissionNotification.stallAfterMs` 以上が過ぎ、定期ジョブ「止まった通知の検出」が N を送信失敗にした | イベントを消費する | エラーにせずに終える。結果を記録せず、N は検出が記録した送信失敗（A と B の結果は `failed`（`unconfirmed`））のまま |
