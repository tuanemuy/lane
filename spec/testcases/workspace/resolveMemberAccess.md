# メンバーの認可結果を求める

ユースケースは [../../usecases/workspace.md](../../usecases/workspace.md) が定める。

- W は `active` のワークスペース。断りが無ければ、多要素認証を必須にしていない
- A は、セッションのアカウント。M は、A の W のメンバー。断りが無ければ、所有者でない
- 断りが無ければ、所属の索引は、A の W の `joined` の項目を持ち、W は名簿で開設済み。「A の項目」は、所属の索引の、A の項目を指す

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-workspace-resolveMemberAccess-01 | M の権限の組は `content:edit` | W と A について求める | `granted` を返す。メンバーの ID は M、`grant` は `member` で、権限の組は `content:edit` |
| TC-workspace-resolveMemberAccess-03 | M は W の所有者。M が保持する権限の組は編集者の組 | W と A について求める | `granted` を返す。`grant` は `owner`。`MemberGrant.permissions` はすべての権限 |
| TC-workspace-resolveMemberAccess-04 | W は、所有者がいない（`vacant`）ワークスペース。M の権限の組は、すべての権限 | W と A について求める | `granted` を返す。`grant` は `member` で、権限の組はすべての権限。求めた結果から作った `Actor` での、所有者だけの操作（「多要素認証を必須にする」）は `ForbiddenError`（理由 `OWNER_ONLY`） |
| TC-workspace-resolveMemberAccess-05 | A は W のメンバーでなく、A の W の項目も無い | W と A について求める | `NotFoundError`。対象の種類は `workspace`、キーは W の ID。W の情報を示さない |
| TC-workspace-resolveMemberAccess-07 | 利用終了の途中のワークスペース C の区画に、`active` の `Workspace` と、すべての権限を持つ A の `Member` がある（利用終了が、名簿の状態の変更の後、開始の前） | C と A について求める | `NotFoundError`。C の情報を示さない。区画の `Workspace` を `closing` にする確定を待たない |
| TC-workspace-resolveMemberAccess-09 | W は多要素認証を必須にしている。A の多要素認証は `disabled` | W と A について求める | `mfa_setup_required` を返す。メンバーの ID は M。エラーにしない |
| TC-workspace-resolveMemberAccess-14 | A の W の項目が `joining`（参加が後段の後、確かめ直しの前に止まった） | W と A について求める | `granted` を返す。A の W の項目は `joining` のまま変わらない |
| TC-workspace-resolveMemberAccess-15 | A は W のメンバーでない。A の W の `joined` の項目がある（解除の後段の前に止まった） | W と A について求める | `NotFoundError`。A の W の項目は変わらない |
