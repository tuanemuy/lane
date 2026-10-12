# 所有者を変更する

ユースケースは [../../usecases/workspace.md](../../usecases/workspace.md) が定める。

- W は `active` のワークスペースで、断りが無ければ、多要素認証を必須にしていない。O は W の所有者。断りが無ければ、実行者は O
- M は、W の、編集者の権限の組を保持するメンバー

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-workspace-transferOwnership-01 | なし | M を新しい所有者に指定して変更する | 正常に終わる。W の所有者は M になる。M の次の操作の認可は、すべての権限を持ち、所有者になる。M が保持する権限の組は変わらず、M を保存しない |
| TC-workspace-transferOwnership-02 | なし | M を新しい所有者に指定して変更する | O は W のメンバーのまま残り、保持する権限の組は管理者の組になる。O は所有者でない。「メンバーを一覧する」は、M を所有者、O を役割 `admin` で返す |
| TC-workspace-transferOwnership-03 | なし | M を新しい所有者に指定して変更する | 操作履歴に `workspace.owner_changed` を一件残す。実行者は O、対象は M。変更前の所有者に O、変更後の所有者に M を残す |
| TC-workspace-transferOwnership-05 | 実行者 X は、W の、すべての権限を持つ、所有者でないメンバー | X が、M を新しい所有者に指定して変更する | `ForbiddenError`（理由 `OWNER_ONLY`）。W の所有者は O のまま。操作履歴を残さない |
| TC-workspace-transferOwnership-06 | なし | O を新しい所有者に指定して変更する | `BusinessRuleError`（`AlreadyOwner`）。W の所有者は O のまま。操作履歴を残さない |
| TC-workspace-transferOwnership-07 | W は多要素認証を必須にしている。O の多要素認証は `enabled`、M の多要素認証は `disabled` | M を新しい所有者に指定して変更する | 正常に終わる。W の所有者は M になる。W は多要素認証を必須にしたまま。M の W への次の操作の認可は `ForbiddenError`（理由 `MFA_SETUP_REQUIRED`）になり、M が多要素認証を設定すると `granted` になる |
| TC-workspace-transferOwnership-08 | M は、「メンバーの利用を解除する」で解除済み | M を新しい所有者に指定して変更する | `NotFoundError`。W の所有者は O のまま。操作履歴を残さない |
| TC-workspace-transferOwnership-09 | M への所有者の変更と、M の解除が並行し、解除が先に確定する | 所有者の変更を確定する | `ConflictError`（理由 `OWNER_NOT_MEMBER`）。対象がメンバーでないことを示す。W の所有者は O のまま。O が保持する権限の組は変わらない。操作履歴を残さない |
| TC-workspace-transferOwnership-10 | O が手動発行したトークン T がある | M を新しい所有者に指定して変更する | T は変わらず、失効しない |
