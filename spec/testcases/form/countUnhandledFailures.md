# 未対処の送信失敗の件数を取得する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-countUnhandledFailures-01 | 送信失敗で未対処の通知が 2 件、再試行を始めて送信を終えていない通知（`attempt` は `retry`、状態は `waiting`）が 1 件、送信失敗で対処済みの通知が 1 件、送信成功の通知が 1 件、未送信の通知が 1 件ある | `notification:manage` を持つメンバーが、件数を取得する | 2 を返す |
| TC-form-countUnhandledFailures-02 | W に、送信失敗で未対処の通知が無い | W の `notification:manage` を持つメンバーが、件数を取得する | 0 を返す |
| TC-form-countUnhandledFailures-03 | W に、送信失敗で未対処の通知が 1 件ある | `notification:manage` を持たないメンバーが、件数を取得する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `notification:manage` |
