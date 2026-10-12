# 初期所有者の招待を発行し直す（`reissueInitialOwnerInvitation`）

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-installation-reissueInitialOwnerInvitation-01 | `operator`。初期所有者が未参加のワークスペース（所有者の指定が `vacant`）に、期限内、または期限を過ぎた `pending` の初期所有者の招待がある | 前の招待と別のメールアドレスを指定して発行し直す | 招待の宛先、有効期限、メールを送信できたことを返す。前の招待は失効し、前の招待では参加できない。新しい宛先の `pending` の初期所有者の招待ができ、その宛先に招待のメールを送る。操作履歴に `invitation.created`（実行者は `operator`）を 1 件残す |
| TC-installation-reissueInitialOwnerInvitation-03 | `operator`。作成が、名簿を開設済みにした後、作成を終える前に止まったワークスペースがある（名簿で開設済み、所有者の指定が `vacant`。初期所有者の招待は無い、または発行の確定の後に届けていない `pending` の招待がある） | そのワークスペースを指定して発行し直す | 新しい招待ができ、宛先に届ける。届けていない前の招待は失効する。操作履歴に `invitation.created`（実行者は `operator`）を残す。届けた後に、名簿の項目の `creation.finished` が真になり、作成が終わる。以後の同じ ID での作成は、作成済みを返す |
| TC-installation-reissueInitialOwnerInvitation-05 | `operator`。設置全体のメール送信を宣言していない設置。初期所有者が未参加のワークスペースがある | 発行し直す | メールを送らず、一度だけ使える招待の値を結果に返す。URL を返さない。前の招待は失効する |
| TC-installation-reissueInitialOwnerInvitation-06 | `operator`。設置全体のメール送信が失敗する設置。初期所有者が未参加のワークスペースがある | 発行し直す | 送信の失敗と理由を結果に示す。発行し直した招待は残り、前の招待は失効したまま |
| TC-installation-reissueInitialOwnerInvitation-07 | `operator`。復旧の後の利用の再開で、所有者を失ったワークスペースがある（所有者の指定が `vacant` の `active`）。メンバーが 2 人残っている | そのワークスペースを指定して、残っていないメールアドレスへ発行し直す | 新しい招待ができ、宛先に届ける。その招待で参加した人が、メンバーになり、所有者になる。残るメンバーの利用と、ワークスペースの設定は変わらない |
| TC-installation-reissueInitialOwnerInvitation-08 | 同上 | そのワークスペースに残っているメンバーのメールアドレスへ発行し直す | 新しい招待ができ、宛先に届ける。そのメンバーが受諾すると、メンバーを作り直さずに、所有者に指定される |
| TC-installation-reissueInitialOwnerInvitation-09 | `operator`。初期所有者が未参加のワークスペース。宛先に、そのワークスペースの `pending` のメンバーの招待がある | そのメールアドレスへ発行し直す | 発行し直せる。前の初期所有者の招待は失効する。メンバーの招待は失効せず、`pending` のまま残る |
| TC-installation-reissueInitialOwnerInvitation-10 | `operator`。作成が、区画の生成の後、名簿を開設済みにする前に止まり、名簿で準備中のまま残るワークスペースがある | 存在しないワークスペース、または準備中のワークスペースを指定して発行し直す | どちらも `NotFoundError`。招待を作らない |
| TC-installation-reissueInitialOwnerInvitation-12 | `operator`。所有者がいる `active` のワークスペースがある | 発行し直す | `BusinessRuleError`（`InitialOwnerAlreadyJoined`）。招待を作らず、操作履歴を残さない |
| TC-installation-reissueInitialOwnerInvitation-13 | `operator`。利用終了の途中のワークスペースがある | 発行し直す | `ConflictError`（理由 `WORKSPACE_CLOSING`）。招待を作らない |
