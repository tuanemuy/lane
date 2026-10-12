# 期限切れの招待を掃除する

ユースケースは [../../usecases/workspace.md](../../usecases/workspace.md) が定める。

- 断りが無ければ、実行者は `system` で、対象のワークスペースは W。「保持期間を過ぎた」は、[../../domains/common.md](../../domains/common.md) の「期限つきの一度きりの値」の保持期間を、ユースケースの初めの現在時刻で過ぎたことを指す

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-workspace-purgeExpiredInvitations-01 | ワークスペース W に、保持期間を過ぎた `pending` の招待と `accepted` の招待 I がある。W に、保持期間を過ぎた `revoked` の招待がある。W に、期限内の `pending` の招待 J がある。別のワークスペース V に、保持期間を過ぎた招待 L がある | 掃除する | 削除した件数 3 を返す。J は残る。V の L は残る。I で参加したメンバーは、W のメンバーのまま変わらない。操作履歴を残さない |
| TC-workspace-purgeExpiredInvitations-02 | W に、有効期限を過ぎ、保持期間の内の `pending` の招待 I がある | 掃除する | 削除した件数 0 を返す。I の値で「招待の内容と状態を確かめる」が `expired` を返す |
| TC-workspace-purgeExpiredInvitations-05 | P は、名簿で終了中で、区画を作る前に止まった作成のワークスペース（区画が無い） | P を対象に掃除する | 削除した件数 0 を返す。エラーにしない |
