# メンバーの権限を変更する

ユースケースは [../../usecases/workspace.md](../../usecases/workspace.md) が定める。

- W は `active` のワークスペース。実行者 X は、W の `member:manage` を持つメンバー
- M は、W の、所有者でないメンバー。断りが無ければ、権限の組は編集者の組

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-workspace-changeMemberPermissions-01 | なし | M に、管理者の役割を指定して変更する | 変更後の M を返す。権限の組はすべての権限、役割は `admin`。「メンバーを一覧する」が、M を変更後の権限の組で返す |
| TC-workspace-changeMemberPermissions-02 | なし | M に、管理者の役割を指定して変更する | 操作履歴に `member.permissions_changed` を一件残す。実行者は X、対象は M で、名称は M のアカウントの名前。変更前に編集者の権限の組、変更後にすべての権限を残す |
| TC-workspace-changeMemberPermissions-03 | なし | M に、権限の列 `content:edit` を指定して変更する | 変更後の M を返す。権限の組は `content:edit`、役割は無し。M の次の操作の認可は、`content:publish` を必要とする操作を `ForbiddenError` にする |
| TC-workspace-changeMemberPermissions-04 | なし | M に、空の権限の列を指定して変更する | 変更後の M を返す。権限の組は空、役割は `viewer`。M は W のメンバーのまま残る |
| TC-workspace-changeMemberPermissions-06 | O は W の所有者 | O に、編集者の役割を指定して変更する | `BusinessRuleError`（`OwnerPermissionsDerived`）。O が保持する権限の組は変わらない。操作履歴を残さない |
| TC-workspace-changeMemberPermissions-07 | なし | M に、語彙にない権限を含む権限の列を指定して変更する | `ValidationError`（`UnknownPermission`）。M の権限の組は変わらない。操作履歴を残さない |
| TC-workspace-changeMemberPermissions-08 | M は、「メンバーの利用を解除する」で解除済み | M に、管理者の役割を指定して変更する | `NotFoundError`。操作履歴を残さない |
| TC-workspace-changeMemberPermissions-09 | M に紐付くトークン T と、M が手動発行したトークン U がある | M に、空の権限の列を指定して変更する | T と U は変わらない。U は失効せず、許可も変わらない |
| TC-workspace-changeMemberPermissions-11 | 実行者 Y は、W の、`member:manage` を持たないメンバー | Y が、M に管理者の役割を指定して変更する | `ForbiddenError`（理由 `PERMISSION_MISSING`）。不足する権限に `member:manage` を添える。M の権限の組は変わらない。操作履歴を残さない |
