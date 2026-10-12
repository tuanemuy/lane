# 期限切れのスクリプトの認証の依頼を削除する

- 断りが無ければ、実行者は `system`
- 「保持期間を過ぎた」と「保持期間の中」は、[../../domains/common.md](../../domains/common.md) の「期限つきの一度きりの値」の保持期間を指す

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-token-deleteExpiredDeviceAuthorizations-01 | 保持期間を過ぎた `pending`、`approved`、`denied`、`redeemed` の依頼と、期限を過ぎ、保持期間の中の `pending` の依頼がある | 実行する | 保持期間を過ぎた四つの依頼を削除する。期限を過ぎ、保持期間の中の依頼は残る。削除した依頼の件数 4 を返す |
