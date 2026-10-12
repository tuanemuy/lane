# 期限切れのセッションと回復依頼を掃除する

ユースケースは [../../usecases/identity.md](../../usecases/identity.md) が定める。

- 断りが無ければ、実行者は `system`。「保持期間を過ぎた」は、[../../domains/common.md](../../domains/common.md) の「期限つきの一度きりの値」の保持期間を、ユースケースの初めの現在時刻で過ぎたことを指す

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-identity-purgeExpiredSessionsAndRecoveryRequests-01 | 保持期間を過ぎた `active` のセッションと `pending_mfa` のセッションが一つずつある。保持期間を過ぎた `pending`、`used`、`revoked` の回復依頼が一つずつある。期限内のセッション S と回復依頼 R がある | 掃除する | 削除したセッションの件数 2 と、回復依頼の件数 3 を返す。S と R は残る。S の値で「セッションからアカウントを確かめる」がアカウントを返す |
| TC-identity-purgeExpiredSessionsAndRecoveryRequests-02 | 有効期限を過ぎ、保持期間の内のセッションと、有効期限を過ぎ、保持期間の内の `pending` の回復依頼 R がある | 掃除する | どちらの件数も 0 を返す。R の値で「回復の値の状態を確かめる」が `expired` を返す |
| TC-identity-purgeExpiredSessionsAndRecoveryRequests-04 | 保持期間を過ぎたセッションがある。ワークスペース W に、保持期間を過ぎた招待がある | 掃除する | セッションを削除する。W の招待は残る。設置の区画の記録だけを削除する |
