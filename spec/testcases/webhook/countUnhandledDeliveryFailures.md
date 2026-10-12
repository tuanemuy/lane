# 未対処の送信失敗の件数を求める

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-webhook-countUnhandledDeliveryFailures-01 | アプリ連携の権限を持つメンバー。送信失敗（未対処）の組が二つ、送信失敗（対処済み）、送信待ち、送信成功の組が一つずつある | 件数を求める | 2 を返す |
| TC-webhook-countUnhandledDeliveryFailures-04 | アプリ連携の権限を持たないメンバー。送信失敗（未対処）の組がある | 件数を求める | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限（`integration:manage`）を示す。件数を示さない |
| TC-webhook-countUnhandledDeliveryFailures-07 | アプリ連携の権限を持つメンバー。一つの通知の、宛先 A の送信と宛先 B の送信が、どちらも送信失敗（未対処） | 件数を求める | 2 を返す。通知と宛先の組を数える。「通知と宛先の組を一覧する」を未対処で絞り込んだ結果の総件数と一致する |
