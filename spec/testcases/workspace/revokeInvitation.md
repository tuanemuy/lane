# 招待を取り消す

ユースケースは [../../usecases/workspace.md](../../usecases/workspace.md) が定める。

- W は `active` のワークスペース。実行者 X は、W の `member:manage` を持つメンバー
- I は、「メンバーを招待する」が W に作った、宛先 e の招待。断りが無ければ、`pending` で期限内

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-workspace-revokeInvitation-01 | なし | I を取り消す | 正常に終わる。I は `revoked` になる。I の値での「招待の内容と状態を確かめる」は `revoked` を返し、「招待を受諾する」は `BusinessRuleError`（`InvitationInvalid`）になる。「未使用の招待を一覧する」は I を返さない |
| TC-workspace-revokeInvitation-02 | なし | I を取り消す | 操作履歴に `member.invitation_revoked` を一件残す。実行者は X、対象は I で、名称は e |
| TC-workspace-revokeInvitation-03 | I は `pending` で、有効期限を過ぎている | I を取り消す | 正常に終わる。I は `revoked` になる。操作履歴に `member.invitation_revoked` を一件残す |
| TC-workspace-revokeInvitation-05 | I は `accepted` | I を取り消す | `BusinessRuleError`（`InvitationAccepted`）。対象が参加済みであることを示す。I と、I で参加したメンバーは変わらない。操作履歴を残さない |
| TC-workspace-revokeInvitation-07 | 指定する ID の招待が無い | その ID の招待を取り消す | `NotFoundError`。操作履歴を残さない |
| TC-workspace-revokeInvitation-08 | K は、W の初期所有者の招待 | K を取り消す | `NotFoundError`。K は変わらない。操作履歴を残さない |
| TC-workspace-revokeInvitation-10 | 実行者 Y は、W の、`member:manage` を持たないメンバー | Y が I を取り消す | `ForbiddenError`（理由 `PERMISSION_MISSING`）。不足する権限に `member:manage` を添える。I は変わらない。操作履歴を残さない |
