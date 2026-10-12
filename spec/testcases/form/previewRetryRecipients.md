# 手動再試行の宛先を確認する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-previewRetryRecipients-01 | 回答 S の通知は送信失敗。`notifiedTo` は A と B。A は `accepted`、B は `failed`。フォームの有効な通知先は A と B | `notification:manage` を持つメンバーが、S の再試行の宛先を確認する | S の `RetryPlan` は、宛先が B、外れた宛先と加わった宛先が空。再試行を受け付けられる。確認した回は `initial`。通知の回は進まず、イベントを記録しない |
| TC-form-previewRetryRecipients-02 | S の通知は送信失敗。`notifiedTo` は A と B。A は `accepted`、B は `failed`。受付の後に、B を無効にし、通知先 C を追加した | `notification:manage` を持つメンバーが、S の再試行の宛先を確認する | 宛先は C、外れた宛先は B、加わった宛先は C。再試行を受け付けられる |
| TC-form-previewRetryRecipients-03 | 削除済みフォームの回答 S の通知は送信失敗。`removed` の設定の有効な通知先は、受付時の通知対象と同じ | `notification:manage` を持つメンバーが、S の再試行の宛先を確認する | 宛先は、`removed` の設定の有効な通知先のうち、`accepted` でない宛先。再試行を受け付けられる |
| TC-form-previewRetryRecipients-04 | S の通知は送信失敗。フォームの有効な通知先は A だけで、A は `accepted` | `notification:manage` を持つメンバーが、S の再試行の宛先を確認する | 宛先は空。再試行を受け付けられない。理由は `noRecipients` |
| TC-form-previewRetryRecipients-05 | S1 の通知は 1 回目の再試行の送信待ち（`attempt` は `retry` の `round` 1、状態は `waiting`）。S2 の通知は送信成功 | `notification:manage` を持つメンバーが、S1 と S2 の再試行の宛先を確認する | どちらも再試行を受け付けられない。理由は `notFailed`。S1 の確認した回は `retry` の `round` 1 |
| TC-form-previewRetryRecipients-07 | S1 の通知は再試行を受け付けられる。S2 の通知は送信失敗で、有効な通知先が残らない | `notification:manage` を持つメンバーが、S1 と S2 の再試行の宛先を確認する | 通知ごとに結果を返す。S1 は受け付けられ、S2 は受け付けられず理由は `noRecipients` |
| TC-form-previewRetryRecipients-09 | 回答 S1 がある。S9 は既に無い | `notification:manage` を持つメンバーが、S1 と S9 の再試行の宛先を確認する | `NotFoundError`。S9 の ID を添える |
| TC-form-previewRetryRecipients-10 | 通知が送信失敗の回答 S | `submission:read` と `submission:delete` だけを持つメンバーが、S の再試行の宛先を確認する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `notification:manage` |
