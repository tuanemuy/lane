# 送信接続設定の更新が影響するフォームを一覧する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-listFormsAffectedByMailConnection-01 | 回答の無い受付中のフォーム F1、未送信の通知が 1 件残る受付停止のフォーム F2、送信失敗の通知が 2 件残る受付停止のフォーム F3、送信成功の通知だけを持つ受付停止のフォーム F4 | `notification:manage` を持つメンバーが一覧する | F1、F2、F3 を返し、F4 を含まない。F1 は受付中で件数は 0 と 0。F2 は受付中でなく未送信が 1。F3 は受付中でなく送信失敗が 2 |
| TC-form-listFormsAffectedByMailConnection-02 | 送信失敗の通知が 1 件残る削除済みフォーム F5 | `notification:manage` を持つメンバーが一覧する | F5 を、状態が削除済みで、受付中でなく、送信失敗が 1 のフォームとして返す。キーと表示名称は削除の時点の値 |
| TC-form-listFormsAffectedByMailConnection-03 | 受付停止のフォーム F6 に、通知が 1 回目の再試行の送信待ち（`attempt` は `retry` の `round` 1、状態は `waiting`）の回答が 1 件だけある | `notification:manage` を持つメンバーが一覧する | F6 を返す。未送信が 1、送信失敗が 0 |
| TC-form-listFormsAffectedByMailConnection-04 | 受付中のフォームも、未送信または送信失敗の通知が残るフォームも無い | `notification:manage` を持つメンバーが一覧する | 空の列を返す |
| TC-form-listFormsAffectedByMailConnection-05 | 受付中のフォームがある | `notification:manage` を持たないメンバーが、一覧する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `notification:manage` |
