# 操作履歴の一件を確かめる（`getAuditLogEntry`）

断りが無い限り、実行者は `workspace:manage` を持つメンバーで、記録は実行者のワークスペースのもの。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-audit-getAuditLogEntry-01 | メンバー B の権限を変えた `member.permissions_changed` の記録がある。B は現在もメンバー | 記録を確かめる | 対象が B で、`changes` の `permissions` が変更前後の権限の組である記録と、対象が現在もあることを返す。操作履歴の記録は増えない |
| TC-audit-getAuditLogEntry-02 | 表示アプリへの通知の宛先の、秘密情報を置き換えた `webhook.settings_changed` の記録がある。宛先は現在もある | 記録を確かめる | `changes` の `secret` は、変更した事実だけを持ち、値を持たない。対象が現在もあることを返す |
| TC-audit-getAuditLogEntry-03 | メンバー B の `member.permissions_changed` の記録がある。B は、その後に利用を解除されている | 記録を確かめる | エラーにしない。記録を、記録の時点の B の名称とともに返し、対象が現在は無いことを示す |
| TC-audit-getAuditLogEntry-04 | トークン T1、T2、T3 の `token.issued` の記録がある。T1 は有効、T2 は失効している、T3 は有効期限を過ぎている | それぞれの記録を確かめる | どれも、エラーにせず、`changes` を持たない記録を、記録の時点のトークンの名称とともに返す。T1 の記録は対象が現在もあること、T2 と T3 の記録は対象が現在は無いことを示す |
| TC-audit-getAuditLogEntry-07 | フォーム contact とフォーム inquiry の `form.recipients_changed` の記録がある。contact は、その後のスキーマの反映で削除され、同じキー contact のフォームが、スキーマの反映で加えられている。inquiry は現行スキーマに残っている | それぞれの記録を確かめる | どちらも、エラーにせず記録を返す。contact の記録は対象が現在は無いことを、inquiry の記録は対象が現在もあることを示す |
| TC-audit-getAuditLogEntry-09 | 宛先の `webhook.settings_changed` の記録がある。その宛先は、削除されている | 記録を確かめる | 記録を返し、対象が現在は無いことを示す |
| TC-audit-getAuditLogEntry-10 | `schema.migrations_applied` の記録（対象はスキーマ）と、`mail_connection.changed` の記録（対象はワークスペース）がある | それぞれの記録を確かめる | どちらも、対象が現在もあることを返す |
| TC-audit-getAuditLogEntry-11 | その ID の記録が無い | 記録を確かめる | `NotFoundError`（対象の種類は `audit_log_entry`） |
| TC-audit-getAuditLogEntry-13 | 実行者のメンバーは `workspace:manage` を持たない | 記録を確かめる | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限 `workspace:manage` を添える。記録を返さない |
| TC-audit-getAuditLogEntry-15 | 招待 I、J、K、L の `member.invited` の記録がある。I は期限内で受諾されていない。J は、その後に取り消されている。K は受諾されないまま期限を過ぎている。L は受諾されている | それぞれの記録を確かめる | I の記録は対象が現在もあることを、J、K、L の記録は対象が現在は無いことを示す。どれもエラーにしない |
