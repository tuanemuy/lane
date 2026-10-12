# 多要素認証を設定していないメンバーを確かめる

ユースケースは [../../usecases/workspace.md](../../usecases/workspace.md) が定める。

- W は `active` のワークスペース。O は W の所有者。断りが無ければ、実行者は O

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-workspace-listMembersWithoutMfa-01 | W のメンバーは次の四人。O（多要素認証は `enabled`）、M2（`disabled`）、M3（`enrolling`）、M4（`enabled`） | 確かめる | M2 と M3 を返す。O と M4 を含めない。W とメンバーは変わらない |
| TC-workspace-listMembersWithoutMfa-02 | W のメンバーは、全員が多要素認証を `enabled` にしている | 確かめる | 空の列を返す |
| TC-workspace-listMembersWithoutMfa-03 | 実行者 X は、W の、すべての権限を持つ、所有者でないメンバー | X が確かめる | `ForbiddenError`（理由 `OWNER_ONLY`）。メンバーを返さない |
