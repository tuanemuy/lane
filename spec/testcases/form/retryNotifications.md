# 通知を手動で再試行する

断りが無い限り、「再試行できる回答」は、通知が最初の通知（`attempt` は `initial`）の送信失敗（未対処）で、宛先 A が `accepted`、宛先 B が `failed`、フォームの有効な通知先が A と B の回答を指す。操作は、対象ごとに、確認した回を添える。断りが無い限り、確認した回は、「手動再試行の宛先を確認する」が返した、通知の今の回。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-retryNotifications-01 | 再試行できる回答 S1。未対処の送信失敗の件数は 1 | `notification:manage` を持つメンバーが、S1 を再試行する | S1 の再試行を始めた結果を返す。S1 の通知は 1 回目の再試行の送信待ち（`attempt` は `retry` の `round` 1、状態は `waiting`）になり、対処の状態を持たない。未対処の送信失敗の件数は 0 になる。`form.notification_retry_requested` を一件記録し、ペイロードは、S1 の `submissionId` と `round` 1（始める `retry` の回）だけを持つ。回答 S1 は変わらず、回答の件数は増えない。このユースケースはメールを送らない |
| TC-form-retryNotifications-03 | S1 の通知は送信失敗で、対処済み。有効な通知先に、`accepted` でない宛先がある | `notification:manage` を持つメンバーが、S1 を再試行する | S1 の再試行を始める。通知は `waiting` になり、対処済みの記録を持たない |
| TC-form-retryNotifications-04 | 再試行できる回答 S1 と S2 | `notification:manage` を持つメンバーが、S1 と S2 を再試行する | 通知ごとに、再試行を始めた結果を返す。`form.notification_retry_requested` を、S1 と S2 に一件ずつ記録する |
| TC-form-retryNotifications-06 | 削除済みフォームの回答 S1 の通知は送信失敗。`removed` の設定の有効な通知先に、`accepted` でない宛先がある | `notification:manage` を持つメンバーが、S1 を再試行する | S1 の再試行を始める。イベントを一件記録する |
| TC-form-retryNotifications-07 | 再試行できる回答 S1 と S2。メンバーが宛先を確認した後に、ほかのメンバーが S2 を再試行し、S2 の通知は 1 回目の再試行の送信待ち（`attempt` は `retry` の `round` 1、状態は `waiting`） | `notification:manage` を持つメンバーが、S1 と S2 を、確認した回 `initial` を添えて再試行する | エラーにしない。S1 は再試行を始めた、S2 は再試行が先に始まっていた（`alreadyRequested`）、という結果を返す。S2 の結果は、今の送信結果（未送信）を持つ。イベントは S1 の一件だけ。S2 の通知は `retry` の `round` 1 の `waiting` のまま変わらない |
| TC-form-retryNotifications-08 | 再試行できる回答 S1 と S2。メンバーが S1 と S2 を、確認した回 `initial` を添えて再試行し、その 1 回目の再試行が、S1 は送信失敗、S2 は送信成功で終わった | 同じメンバーが、同じ操作（確認した回 `initial`）をもう一度行う | エラーにしない。S1 と S2 は `alreadyRequested` で、今の送信結果（S1 は送信失敗、S2 は送信成功）を返す。どちらの通知も変わらない。イベントを記録せず、二通目を送らない |
| TC-form-retryNotifications-10 | S1 の通知は、1 回目の再試行の送信失敗（`attempt` は `retry` の `round` 1）。B は `failed` のまま | `notification:manage` を持つメンバーが、宛先を確認し直し、S1 を、確認した回 `retry` の `round` 1 を添えて再試行する | S1 の再試行を始める。通知の `attempt` は `retry` の `round` 2 になり、イベントの `round` は 2 |
| TC-form-retryNotifications-11 | 再試行できる回答 S1。S2 の通知は、最初の通知の送信成功 | `notification:manage` を持つメンバーが、S1 と S2 を再試行する | エラーにしない。再試行を始めず、S2 と理由 `notFailed` を返す。S1 と S2 の通知は変わらず、イベントを記録しない |
| TC-form-retryNotifications-12 | 再試行できる回答 S1。S2 の通知は送信失敗で、S2 のフォームの設定（`active`、または削除済みフォームの `removed`）に有効な通知先が無い | `notification:manage` を持つメンバーが、S1 と S2 を再試行する | エラーにしない。再試行を始めず、S2 と理由 `noRecipients` を返す。S1 と S2 の通知は変わらず、イベントを記録しない |
| TC-form-retryNotifications-14 | 再試行できる回答 S1。S2 の通知は、最初の通知の送信成功。S9 は、削除または自動整理で既に無い | `notification:manage` を持つメンバーが、S1、S9、S2 の順に選んで再試行する。続けて、S9 と S2 を対象から外し、S1 だけを再試行する | 一度目はエラーにしない。再試行を始めず、S9 と理由 `notFound`、S2 と理由 `notFailed` を、この順に返す。S1 と S2 の通知は変わらず、イベントを記録しない。二度目は、S1 の再試行を始める |
| TC-form-retryNotifications-15 | 再試行できる回答 S1 | 二人のメンバーが並行して、S1 を再試行する | 一方が `ConflictError`（`STALE_VERSION`）。S1 の通知の `attempt` は `retry` の `round` 1 で、イベントは一件だけ。競合した側が同じ操作を送り直すと、S1 は `alreadyRequested` になる |
| TC-form-retryNotifications-17 | 再試行できる回答 S1 | `submission:read` と `submission:delete` だけを持つメンバーが、S1 を再試行する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `notification:manage`。S1 の通知は変わらず、イベントを記録しない |
