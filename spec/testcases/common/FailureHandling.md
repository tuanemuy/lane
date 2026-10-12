# FailureHandling

[../../domains/common.md](../../domains/common.md) の「FailureHandling」の代表ケース。Form の `SubmissionNotification` と Webhook の `WebhookDelivery` が同じ規則で使うので、ユースケースごとのファイルに置かない。回答の削除で未対処でなくなることは [../form/deleteSubmissions.md](../form/deleteSubmissions.md)（`TC-form-deleteSubmissions-03`）が、確認した回の扱いは、手動再試行と対処の記録のユースケースのテストが持つ。

- 後半の行の実行者は、アプリ連携の権限を持つメンバー M。「件数」は、Webhook の `countUnhandledDeliveryFailures` が返す未対処の送信失敗の件数

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-common-FailureHandling-01 | なし | `unhandled()` の値を `isUnhandled` に渡し、`markHandled(unhandled(), M, t)` の値を `isUnhandled` に渡す | 順に、真、偽を返す。`markHandled` の値は、M と t を持つ `handled` |
| TC-common-FailureHandling-02 | M が t1 に記録した `handled` の値 h | `markHandled(h, N, t2)` を呼ぶ | h をそのまま返す |
| TC-common-FailureHandling-03 | 送信失敗（未対処）の送信 D がある。件数は 1 | M が D の再試行を始め（Webhook の `requestDeliveryRetry`）、件数を求める。再試行が送信成功で終わった後に、もう一度求める | どちらも 0 を返す |
| TC-common-FailureHandling-04 | 送信失敗（対処済み）の送信 D がある。件数は 0 | M が D の再試行を始め、再試行が送信失敗で終わった後に、件数を求める | 1 を返す。D は未対処の送信失敗 |
