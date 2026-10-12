# 期限つきの一度きりの値

[../../domains/common.md](../../domains/common.md) の「期限つきの一度きりの値」の、保持期間と掃除の代表ケース。どの集約と掃除のユースケースにも同じ規則で当たるので、ユースケースごとのファイルに置かない。照合値の一意性と `deleteLapsed` の対象は、各集約のポート適合テストが持つ。`Token` の `lapsedAt` に依る結果は [../token/revealBrowserTokenValue.md](../token/revealBrowserTokenValue.md)（`TC-token-revealBrowserTokenValue-07`）と [../token/authenticateToken.md](../token/authenticateToken.md)（`TC-token-authenticateToken-29`）が持つ。

- 「保持期間を過ぎた」は、操作の時刻で `OneTimeValueRetention.isForgotten` が真であることを指す

## 保持期間

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-common-OneTimeValueRetention-01 | なし | `now` と、`lapsedAt` に `cutoff(now)`、`cutoff(now)` の 1 ミリ秒後、`null` を、それぞれ `isForgotten` に渡す | 順に、真、偽、偽を返す |
| TC-common-OneTimeValueRetention-02 | ワークスペース W に、引き換え X、Y、Z がある。X は期限切れで保持期間を過ぎ、定期ジョブはまだ削除していない。Y は期限切れで保持期間の中。Z は引き換え済みで保持期間の中 | X、Y、Z の値で、それぞれ引き換える（Token の `redeemPreviewExchange`） | X は、存在しない値と同じ結果を返す。Y は期限切れの理由を、Z は使用済みの理由を返す |

## 掃除

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-common-OneTimeValueRetention-03 | ワークスペース W に、保持期間を過ぎた招待が `BoundedDeletion.maxLimit` 件より多くある。前の起動が、一回目の `deleteLapsed` の確定の後に中断した | 定期ジョブ「期限切れの記録の掃除」を起動する（Workspace の `purgeExpiredInvitations`） | 前の起動が削除した招待は無いまま。残りの、保持期間を過ぎた招待をすべて削除する |
| TC-common-OneTimeValueRetention-04 | 利用終了の途中のワークスペース C の区画が `closing`。C に、保持期間を過ぎた招待がある | C を対象に、定期ジョブ「期限切れの記録の掃除」を起動する（Workspace の `purgeExpiredInvitations`） | エラーにせず、その招待を削除する |
