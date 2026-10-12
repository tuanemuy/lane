# メンバーを一覧する

ユースケースは [../../usecases/workspace.md](../../usecases/workspace.md) が定める。

- W は `active` のワークスペース。断りが無ければ、実行者は W の所有者 O

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-workspace-listMembers-01 | W のメンバーは次の三人。所有者 O（多要素認証は `enabled`）、M2（編集者の権限の組。多要素認証は `disabled`）、M3（権限の組は `submission:read`。多要素認証は `enrolling`）。W は多要素認証を必須にしていない。別のワークスペース V にメンバー N がいる | 一覧する | 三人を返す。O は、権限の組がすべての権限、所有者、多要素認証を設定している。M2 は、役割 `editor`、所有者でない、多要素認証を設定していない。M3 は、役割は無し、所有者でない、多要素認証を設定していない。総件数は 3。W が多要素認証を必須にしていないことと、W に所有者がいることを返す。N を含めない |
| TC-workspace-listMembers-02 | W は多要素認証を必須にしている。W に、編集者の権限の組を持ち、多要素認証が `disabled` のメンバー M2 がいる | 一覧する | W が多要素認証を必須にしていることを返す。M2 を、編集者の権限の組、役割 `editor`、所有者でない、多要素認証を設定していない、で返す |
| TC-workspace-listMembers-03 | W は、所有者がいない（`vacant`）ワークスペース。実行者 X は、W の、`member:manage` を持つメンバー | X が一覧する | W に所有者がいないことを返す。どのメンバーも所有者でない |
| TC-workspace-listMembers-05 | 実行者 Y は、W の、`member:manage` を持たないメンバー | Y が一覧する | `ForbiddenError`（理由 `PERMISSION_MISSING`）。不足する権限に `member:manage` を添える。メンバーを返さない |
