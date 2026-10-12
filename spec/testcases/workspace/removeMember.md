# メンバーの利用を解除する

ユースケースは [../../usecases/workspace.md](../../usecases/workspace.md) が定める。

- W は `active` のワークスペース。実行者 X は、W の `member:manage` を持つメンバー
- M は、アカウント A の、W の、所有者でないメンバー

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-workspace-removeMember-01 | なし | M を解除する | 正常に終わる。M は W のメンバーでなくなる。「メンバーを一覧する」は M を返さない |
| TC-workspace-removeMember-02 | なし | M を解除する | 操作履歴に `member.removed` を一件残す。実行者は X、対象は M で、名称は A の名前 |
| TC-workspace-removeMember-03 | なし | M を解除する | 所属の索引から、A の W の項目が外れる。A（`version` を含む）は変わらない |
| TC-workspace-removeMember-04 | W は、所有者がいない（`vacant`）。A のメールアドレスに宛てた、W の `pending` の初期所有者の招待 L がある | M を解除する | A の参加の囲いの世代が 1 進み、L は変わらず、`pending` のまま。その後に、A の W の項目が外れる。L の値での受諾は、メンバーを作って、A を所有者に指定する |
| TC-workspace-removeMember-05 | 前段（メンバーの削除と操作履歴）が確定した後、後段（所属の索引からの除外）の前に止まった | 「所属するワークスペースを一覧する」と、W と A について「メンバーの認可結果を求める」を行う。続けて、「所属の索引を修復する」を行う | M は W のメンバーでない。所属の索引に A の W の `joined` の項目が残っている。一覧は W を含めず、認可結果は `NotFoundError` で、どちらも項目を変えない。修復が、A の W の項目を外す |
| TC-workspace-removeMember-07 | A の `active` のセッション S がある | M を解除する | A の名前と資格情報、S は変わらない。S の値で「セッションからアカウントを確かめる」が A を返す。A の W への次の操作の認可は `NotFoundError` になる。A の「所属するワークスペースを一覧する」は W を含めない |
| TC-workspace-removeMember-09 | M に紐付くトークン T と、M が発行したアプリに紐付くトークン U がある | M を解除する | T と U は変わらず、失効しない |
| TC-workspace-removeMember-10 | O は W の所有者 | O を解除する | `BusinessRuleError`（`OwnerCannotBeRemoved`）。O は W のメンバーで、所有者のまま。操作履歴を残さない |
| TC-workspace-removeMember-11 | M は解除済み | M を解除する | `NotFoundError`。操作履歴を残さない |
| TC-workspace-removeMember-12 | M の解除と、M を新しい所有者にする「所有者を変更する」が並行し、所有者の変更が先に確定する | 解除を確定する | `ConflictError`（理由 `MEMBER_IS_OWNER`）。M は W のメンバーで、所有者のまま。操作履歴を残さない |
| TC-workspace-removeMember-13 | 実行者 Y は、W の、`member:manage` を持たないメンバー | Y が M を解除する | `ForbiddenError`（理由 `PERMISSION_MISSING`）。不足する権限に `member:manage` を添える。M は W のメンバーのまま。操作履歴を残さない |
| TC-workspace-removeMember-15 | 前段（メンバーの削除と操作履歴）が確定した後、後段の前に、A が新しい招待で W に参加し直し、A の W の項目は `joined` になった | 解除の後段を行う | A の W の項目を外さない。A は W のメンバーのまま、項目は `joined` のまま |
| TC-workspace-removeMember-16 | 前段が確定した後、後段の前に、A が新しい招待の受諾の前段を確定し、A の W の項目は、期限内の `joining` になった | 解除の後段を行う | A の W の項目を外さない |
| TC-workspace-removeMember-17 | 前段が確定した後、後段が名簿の項目を読む前に、W の利用終了が名簿を終了中にした | 解除の後段を行う | 正常に終わる。A の W の `joined` の項目を変えず、招待を書かない。続く、利用終了の最後の段が、A の W の項目を外す |
| TC-workspace-removeMember-18 | A は、W だけに所属する | M を解除する | A は、所属の索引の項目を持たない、どのワークスペースにも所属しないアカウントとして残る。A の「所属するワークスペースを一覧する」は空の列を返す。A はログインできる |
| TC-workspace-removeMember-19 | 前段が確定した後、後段の前に、A が新しい招待 I の受諾の前段を確定し、後段を確定しないまま、参加の期限を過ぎた | 解除の後段を行う | A の参加の囲いの世代が進んだ後に、A の W の項目が外れる。その後の、その受諾の後段は `BusinessRuleError`（`JoiningLapsed`）になる |
| TC-workspace-removeMember-20 | 前段が確定した後、後段の取り下げの確定が、A の新しい招待の受諾の後段との、参加の囲いの競合で確定しない | M を解除する | 正常に終わる。解除は成立している。A の W の項目は変えずに残り、次の「所属の索引を修復する」が、区画の `Member` に合わせて直す |
| TC-workspace-removeMember-21 | 前段が確定した後、後段が名簿で開設済みの項目を読み、取り下げの確定の前に、W の利用終了の開始が確定する | M を解除する | 正常に終わる。`NotFoundError` にしない。解除は成立している。A の W の項目は変えずに残る |
