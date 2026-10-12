# 多要素認証の必須を解除する

ユースケースは [../../usecases/workspace.md](../../usecases/workspace.md) が定める。

- W は `active` のワークスペースで、多要素認証を必須にしている。O は W の所有者。断りが無ければ、実行者は O

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-workspace-liftMfaRequirement-01 | W に、多要素認証が `disabled` のメンバー M2 がいる | 必須を解除する | 正常に終わる。W は多要素認証を必須にしなくなる。M2 の W での次の操作の認可は `granted` になる |
| TC-workspace-liftMfaRequirement-02 | W に、多要素認証が `enabled` のメンバー M3 がいる | 必須を解除する | M3 と O のアカウントの多要素認証は `enabled` のまま変わらない |
| TC-workspace-liftMfaRequirement-03 | 実行者 X は、W の、すべての権限を持つ、所有者でないメンバー | X が必須を解除する | `ForbiddenError`（理由 `OWNER_ONLY`）。W は多要素認証を必須にしたまま |
