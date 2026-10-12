# 所属の索引を修復する

ユースケースは [../../usecases/workspace.md](../../usecases/workspace.md) が定める。

- 断りが無ければ、実行者は `system`。W は、名簿で開設済みの `active` のワークスペース。A はアカウント
- 「A の項目」は、所属の索引の、A の W の項目を指す。「期限」は、参加の期限（[../../domains/workspace.md](../../domains/workspace.md) の「Membership」）を指す

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-workspace-repairMembershipIndex-01 | A は W のメンバーで、A の項目は `joined` | 修復する | A の項目は変わらない。`joined` にした項目と、外した項目の件数は 0 |
| TC-workspace-repairMembershipIndex-02 | A は W のメンバーで、A の項目は `joining`（参加が後段の後、確かめ直しの前に止まった） | 修復する | A の項目は `joined` になる。世代は変わらない。A の `version` は変わらない。操作履歴を残さない |
| TC-workspace-repairMembershipIndex-03 | A は W のメンバーでない。A の項目は `joined`（解除の後段の前に止まった）。A のメールアドレスに宛てた、W の `pending` の招待 I がある | 修復する | 期限を待たずに、A の参加の囲いの世代が 1 進み、その後に、A の項目が外れる。I は変わらず、`pending` のまま。A は残る |
| TC-workspace-repairMembershipIndex-04 | A は W のメンバーでない。A の項目は、期限の前の `joining`（参加の途中） | 修復する | A の項目は変わらない |
| TC-workspace-repairMembershipIndex-05 | A は W のメンバーでない。A の項目は、期限を過ぎた `joining`。A のメールアドレスに宛てた、W の `pending` の招待 I がある | 修復する | A の参加の囲いの世代が 1 進み、I は変わらない。その後に、A の項目が外れる。その参加の受諾の後段は `BusinessRuleError`（`JoiningLapsed`）になる |
| TC-workspace-repairMembershipIndex-06 | 同上。W に、A の参加の囲いが無い | 修復する | 世代が 1 の A の囲いを置き、その後に、A の項目を外す |
| TC-workspace-repairMembershipIndex-07 | A は W のメンバーでない。A の項目は、期限を過ぎた `joining`。修復が項目を読んだ後、取り下げの確定の前に、受諾の後段が A の `Member` を作った | 修復の取り下げを行う | 取り下げない。A の項目は `joined` になる。項目を外さない |
| TC-workspace-repairMembershipIndex-08 | A は W のメンバーでない。A の項目は、期限を過ぎた `joining`。前の起動が、取り下げの確定の後、項目の除外の前に止まった | 修復する | 同じ手順を行う。A の参加の囲いの世代がもう一度進み、A の項目が外れる |
| TC-workspace-repairMembershipIndex-09 | 名簿に無いワークスペース X の、A の項目がある（利用終了の最後の段の後に、参加の確かめ直しが置いた） | 修復する | A の X の項目を外す |
| TC-workspace-repairMembershipIndex-10 | 名簿で準備中のワークスペース P の、A の `joining` の項目がある（初期構成の読み込みが途中で止まった） | 修復する | A の P の項目は変わらない |
| TC-workspace-repairMembershipIndex-11 | 利用終了の途中のワークスペース C の、A の `joined` の項目と、アカウント B の `joining` の項目がある。C の区画の `Member` は消去済み | 修復する | どちらの項目も変わらない |
| TC-workspace-repairMembershipIndex-12 | A は W のメンバーでなく、A の項目は `joined`。アカウント B は、W のメンバーでなく、B の項目は、期限を過ぎた `joining`。B の項目の取り下げの確定が、参加の後段との競合で確定しない | 修復する | エラーにしない。A の項目が外れる。B の項目は変わらずに残り、確定しなかった項目の件数に 1 を返す。次の起動が、B の項目を直す |
| TC-workspace-repairMembershipIndex-13 | A は W のメンバーでなく、A の項目は `joined`。修復が W を名簿で開設済みと読んだ後、A の項目の取り下げの確定の前に、W の利用終了の開始が確定する | 修復する | エラーにしない。A の項目は変わらずに残り、確定しなかった項目の件数に 1 を返す。残りの項目を続ける |
| TC-workspace-repairMembershipIndex-14 | 一つ目のページに、`joining` で `Member` がある A の項目と、名簿に無いワークスペースの B の項目がある。次のページに、`joining` で `Member` がある C の項目がある。一つ目のページの `confirm` と `remove` の UnitOfWork が `ConflictError` で確定しない | 修復する | エラーにしない。A と B の項目は変わらずに残り、確定しなかった項目の件数に 2 を返す。C の項目は `joined` になる |
