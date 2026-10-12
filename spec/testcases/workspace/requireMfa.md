# 多要素認証を必須にする

ユースケースは [../../usecases/workspace.md](../../usecases/workspace.md) が定める。

- W は `active` のワークスペースで、多要素認証を必須にしていない。O は W の所有者。断りが無ければ、実行者は O で、O の多要素認証は `enabled`

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-workspace-requireMfa-01 | W に、多要素認証が `disabled` のメンバー M2 と、`enabled` のメンバー M3 がいる | 必須にする | 正常に終わる。W は多要素認証を必須にする。「メンバーを一覧する」が、W が多要素認証を必須にしていることを返す。M2 と M3 のメンバー、権限の組、アカウントは変わらない |
| TC-workspace-requireMfa-02 | W に、多要素認証が `disabled` のメンバー M2 と、`enabled` のメンバー M3 がいる。M2 の `active` のセッションがある | 必須にした後、M2、M3、O が W で次の操作を行う | M2 の認可は `ForbiddenError`（理由 `MFA_SETUP_REQUIRED`）になる。M3 と O の認可は `granted` になる |
| TC-workspace-requireMfa-03 | O の多要素認証は `disabled` | 必須にする | `BusinessRuleError`（`OwnerMfaNotConfigured`）。W は多要素認証を必須にしないまま |
| TC-workspace-requireMfa-04 | 実行者 X は、W の、すべての権限を持つ、所有者でないメンバー。X の多要素認証は `enabled` | X が必須にする | `ForbiddenError`（理由 `OWNER_ONLY`）。W は多要素認証を必須にしないまま |
| TC-workspace-requireMfa-05 | W に、アプリに紐付くトークン T がある | 必須にする | T は変わらない |
