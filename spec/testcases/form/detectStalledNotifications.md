# 止まった通知を検出する

断りが無い限り、通知は、定期ジョブが対象にするワークスペース W の通知。「止まった」は、`SubmissionNotification.stalled` が `null` でないことを指す。「定期ジョブが検出を実行する」は、W を対象に実行することを指す。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-detectStalledNotifications-01 | 回答 S の通知 N は最初の通知の送信待ち（`attempt` は `initial`、状態は `waiting`）で、止まっている（イベントが届いていない）。`notifiedTo` の A と B は、フォームの有効な通知先。`MailSender` は受理を返す | 定期ジョブが検出を実行する | 送信を行った通知 1 件、送信失敗にした通知 0 件を返す。`MailSender` を一度呼び、宛先は A と B。N は送信成功になる。送信失敗にしない |
| TC-form-detectStalledNotifications-02 | N は 1 回目の再試行の送信待ち（`attempt` は `retry` の `round` 1、状態は `waiting`）で、止まっている。フォームの有効な通知先は A と B。A の結果は `accepted`、B の結果は `failed`。`MailSender` は受理を返す | 定期ジョブが検出を実行する | `MailSender` の宛先は B。A へ送らない。N は送信成功になる |
| TC-form-detectStalledNotifications-04 | N は最初の通知の送信待ち（`attempt` は `initial`、状態は `waiting`）で、止まっている。受付の後に、フォームの有効な通知先が無くなった | 定期ジョブが検出を実行する | 送信を行った通知 0 件、送信失敗にした通知 1 件を返す。`MailSender` を呼ばない。N は送信失敗（`noRecipients`、未対処）になる |
| TC-form-detectStalledNotifications-05 | N は 最初の通知の送信中（`attempt` は `initial`、状態は `sending`）で、止まっている。送信中の宛先（`recipients`）は A と B | 定期ジョブが検出を実行する | 送信を行った通知 0 件、送信失敗にした通知 1 件を返す。A と B の結果は `failed`（`unconfirmed`）になり、N は送信失敗（`delivery`、未対処）。メールを送らない。回答 S は変わらない |
| TC-form-detectStalledNotifications-06 | N は 1 回目の再試行の送信中（`attempt` は `retry` の `round` 1、状態は `sending`）で、止まっている。A の結果は `accepted`、送信中の宛先（`recipients`）は B | 定期ジョブが検出を実行する | A は `accepted` のまま。B の結果は `failed`（`unconfirmed`）になり、N は送信失敗（未対処） |
| TC-form-detectStalledNotifications-07 | N は送信中（`sending`）で、送信を始めてから `SubmissionNotification.stallAfterMs` が過ぎていない | 定期ジョブが検出を実行する | どちらの件数も 0 を返す。N は変わらない |
| TC-form-detectStalledNotifications-09 | W は利用終了の途中。W に、止まった送信待ちの通知 N1 と、止まった送信中の通知 N2 がある | 定期ジョブが、W を対象に検出を実行する | メールを送らず、何も書かない。N1 は `waiting`、N2 は `sending` のまま。どちらの件数も 0 を返す |
| TC-form-detectStalledNotifications-10 | 止まった送信中の通知 N1 と N2。定期ジョブが N1 を読んだ後、確定までに、並行した消費者が N1 の結果を保存する | 定期ジョブが検出を実行する | N1 の保存は競合し、N1 は消費者が保存した結果のまま残る。定期ジョブは、N1 を同じ起動の中でやり直さず、N2 へ進む。N2 は送信失敗（未対処）になる |
| TC-form-detectStalledNotifications-11 | 止まった送信待ちの通知 N1 と N2。定期ジョブが `findStalled` で N1 と N2 を求めた後、N1 の通知を読む前に、N1 の回答を「回答を削除する」で削除した。`MailSender` は受理を返す | 定期ジョブが検出を実行する | エラーにしない。送信を行った通知 1 件（N2）、送信失敗にした通知 0 件を返す。N1 はどちらの件数にも数えない |
