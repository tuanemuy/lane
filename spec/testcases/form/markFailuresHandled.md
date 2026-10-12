# 通知の送信失敗を対処済みとして記録する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-markFailuresHandled-01 | 回答 S の通知は送信失敗で、未対処。未対処の送信失敗の件数は 1 | `notification:manage` を持つメンバー M が、S を対処済みにする | 対処済みになった通知として S を返す。S の通知は送信失敗のまま、対処済み（対処者は M、時刻は現在時刻）になる。未対処の送信失敗の件数は 0 になる |
| TC-form-markFailuresHandled-02 | 回答 S1、S2 の通知は送信失敗（`delivery`）、S3 の通知は送信失敗（`noRecipients`）で、どれも未対処。未対処の送信失敗の件数は 3 | `notification:manage` を持つメンバーが、S1、S2、S3 を一度に対処済みにする | 三件とも対処済みになる。未対処の送信失敗の件数は 0 になる |
| TC-form-markFailuresHandled-04 | S1 の通知は送信失敗で、メンバー M1 による対処済み。S2 の通知は送信失敗で、未対処 | `notification:manage` を持つメンバー M2 が、S1 と S2 を対処済みにする | 対処済みになった通知として S2 だけを返す。S1 は M1 による対処済みのままで、保存せず、`version` は変わらない |
| TC-form-markFailuresHandled-05 | S1 の通知は送信失敗で、未対処。S2 の通知は最初の通知の送信待ち（`attempt` は `initial`、状態は `waiting`）、S3 の通知は送信成功、S4 の通知は 1 回目の再試行の送信待ち（`attempt` は `retry` の `round` 1、状態は `waiting`） | `notification:manage` を持つメンバーが、S1 から S4 を対処済みにする | エラーにしない。記録せず、S2、S3、S4 と、それぞれの理由 `notFailed` を返す。どの通知も変わらない |
| TC-form-markFailuresHandled-08 | S1 の通知は送信失敗で、未対処。S2 の通知は送信成功。S9 は、削除または自動整理で既に無い | `notification:manage` を持つメンバーが、S1、S9、S2 の順に選んで対処済みにする | エラーにしない。記録せず、S9 と理由 `notFound`、S2 と理由 `notFailed` を、この順に返す。S1 の通知は未対処のまま |
| TC-form-markFailuresHandled-09 | S1 と S2 の通知は送信失敗（`attempt` は `initial`）で、未対処。メンバーが二件を選んだ後、ほかのメンバーの手動再試行で S2 の 1 回目の再試行が始まり、再び送信失敗になった | `notification:manage` を持つメンバーが、選んだときの回（どちらも `initial`）を添えて、S1 と S2 を対処済みにする | エラーにしない。記録せず、S2 と理由 `notFailed` を返す。S1 は未対処のまま。S2 は、1 回目の再試行の送信失敗として未対処のまま |
| TC-form-markFailuresHandled-12 | S の通知は送信失敗で、未対処 | `submission:read` だけを持つメンバーが、S を対処済みにする | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `notification:manage`。対処の状態は未対処のまま |
