# スクリプトの認証を承認しない

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-token-denyDeviceAuthorization-01 | `pending` の依頼。ログインしたアカウント（どのワークスペースにも所属しない） | 承認の値を指定して、承認しないことを選ぶ | 正常に終わる。依頼は、理由が `denied_by_account` の拒否になる。トークンは発行されない |
| TC-token-denyDeviceAuthorization-03 | 期限を過ぎた `pending` の依頼。ログインしたアカウント | 承認しないことを選ぶ | 正常に終わる。依頼は、理由が `denied_by_account` の拒否になる |
| TC-token-denyDeviceAuthorization-04 | ログインしたアカウント | どの依頼にも該当しない承認の値を指定する | `NotFoundError` |
| TC-token-denyDeviceAuthorization-05 | `approved` の依頼と、拒否の依頼 | それぞれ、承認しないことを選ぶ | どちらも `BusinessRuleError`（`DeviceAuthorizationNotPending`）。依頼は変わらない |
