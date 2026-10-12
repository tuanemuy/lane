# 未使用の招待を一覧する

ユースケースは [../../usecases/workspace.md](../../usecases/workspace.md) が定める。

- W は `active` のワークスペース。実行者 X は、W の `member:manage` を持つメンバー

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-workspace-listPendingInvitations-01 | W に、「メンバーを招待する」が作った次の招待がある。期限内の `pending` の I1（編集者の権限の組）、有効期限を過ぎた `pending` の I2（権限の組は `submission:read`）、`accepted` の I3、`revoked` の I4 | 一覧する | I1 と I2 を返す。I1 は期限内、役割は `editor`。I2 は期限切れ、役割は無し。どちらも、招待の値を含まない。総件数は 2。I3 と I4 を含めない |
| TC-workspace-listPendingInvitations-02 | W は、所有者がいない（`vacant`）ワークスペース。W に、期限内の `pending` の I1 と、ホスティング運営者が発行し直した、期限内の `pending` の初期所有者の招待 K がある。別のワークスペース V に、`pending` の招待 J がある | 一覧する | I1 だけを返す。総件数は 1。K と J を含めない |
| TC-workspace-listPendingInvitations-03 | W に、宛先 e の、期限内の `pending` の I1 がある。e のアカウントは、初期所有者の招待の受諾で W の所有者になった | 一覧する | I1 を期限内で返す。I1 の値での「招待の内容と状態を確かめる」が、そのアカウントに `invalid` を返すことは、一覧に現れない |
| TC-workspace-listPendingInvitations-05 | 実行者 Y は、W の、`member:manage` を持たないメンバー | Y が一覧する | `ForbiddenError`（理由 `PERMISSION_MISSING`）。不足する権限に `member:manage` を添える。招待を返さない |
