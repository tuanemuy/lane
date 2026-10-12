# スクリプトの認証を承認する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-token-approveDeviceAuthorization-01 | スキーマ反映の許可を要求した `pending` の依頼。ワークスペース W と V に所属し、W でスキーマ変更権限を持つメンバー | 承認の値と、対象に選んだ W を指定して承認する | 承認した依頼の内容（ワークスペース W、許可、トークンの有効期限）を返す。出力はトークンの値を含まない。依頼は `approved` になり、W、承認したメンバーとアカウント、承認の時刻の `TokenLifetimePolicy.deviceTokenMs` 後の有効期限を記録する。トークンは発行されていない。W の操作履歴に記録を残さない |
| TC-token-approveDeviceAuthorization-02 | プレビュー取得の許可を要求した `pending` の依頼。どの権限も持たない役割の、W のメンバー | W を指定して承認する | 依頼は `approved` になる |
| TC-token-approveDeviceAuthorization-03 | スキーマ反映の許可を要求した `pending` の依頼。承認の画面が W を選べると示した後に、メンバーは W のスキーマ変更権限を失った | W を指定して承認する | エラーにしない。拒否になったことと、不足する権限（`schema:change`）を返す。依頼は、理由が `insufficient_permissions` の拒否になり、不足する権限を持つ |
| TC-token-approveDeviceAuthorization-04 | 上の承認で拒否になった依頼（メンバーは、スキーマ変更権限を持つ V にも所属する）と、`approved` の依頼 | それぞれ、メンバーが所属するワークスペースを指定して承認する | どちらも `BusinessRuleError`（`DeviceAuthorizationNotPending`）。依頼は変わらない |
| TC-token-approveDeviceAuthorization-05 | 期限を過ぎた `pending` の依頼。W のメンバー | W を指定して承認する | `BusinessRuleError`（`DeviceAuthorizationExpired`）。依頼は変わらない |
| TC-token-approveDeviceAuthorization-07 | W のメンバー | どの依頼にも該当しない承認の値を指定して承認する | `NotFoundError` |
| TC-token-approveDeviceAuthorization-08 | `pending` の依頼。ワークスペース W に所属しないアカウント | W を指定して承認する | `NotFoundError`。依頼は `pending` のまま |
| TC-token-approveDeviceAuthorization-12 | スキーマ反映の許可を要求した `pending` の依頼。W でスキーマ変更権限を持つメンバー M。承認の認可の読み取りの後、確定の前に、M が W の利用を解除された | W を指定して承認する | 依頼は `approved` で確定する。続く受け取りは、拒否（`refusal` が `member_unavailable`）を返す |
