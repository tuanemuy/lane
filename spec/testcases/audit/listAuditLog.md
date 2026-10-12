# 操作履歴を一覧する（`listAuditLog`）

断りが無い限り、実行者は `workspace:manage` を持つメンバー。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-audit-listAuditLog-01 | ワークスペースに、メンバー A による `member.invited`、アプリに紐付くトークン T（発行者 A）による `schema.migrations_applied`、ホスティング運営者（`operator`）による `invitation.created` の記録が、この順の日時である | 条件なしで一覧する | 三件を日時の新しい順に返し、総件数は 3。T の記録の実行者は `appToken` で、T の ID と名称と、発行者 A の名前を持つ。A の記録の実行者は A の `MemberId` を持つ。変更の内容を含まない。操作履歴の記録は増えず、ドメインイベントを記録しない |
| TC-audit-listAuditLog-02 | メンバー B に紐付くトークン U（スクリプトの認証で発行）による `schema.pushed` の記録がある | 一覧する | 記録の実行者は `memberToken` で、U の発行の経路 `device` と、紐付くメンバー B の名前を持つ |
| TC-audit-listAuditLog-04 | 1 日、2 日、3 日の正午の記録が一件ずつある | 期間の開始に 2 日の 0 時、終了に 3 日の 0 時を指定して一覧する | 2 日の記録だけを返し、総件数は 1 |
| TC-audit-listAuditLog-05 | `member.invited`、`token.issued`、`schema.pushed` の記録が一件ずつある | 操作の種類に `token.issued` と `schema.pushed` を指定して一覧する | その二件を返し、総件数は 2 |
| TC-audit-listAuditLog-06 | メンバー A が実行者の記録、A が発行したトークン T が実行者の記録、メンバー B が実行者の記録、B に紐付くトークン U が実行者の記録がある | 実行者にメンバー A を指定して一覧する。続けて、実行者にメンバー B を指定して一覧する | A の指定は、A が実行者の記録だけを返し、T が実行者の記録を含まない。B の指定は、B が実行者の記録だけを返し、U が実行者の記録を含まない |
| TC-audit-listAuditLog-07 | 同上 | 実行者にトークン T を指定して一覧する | T が実行者の記録だけを返す |
| TC-audit-listAuditLog-09 | メンバー A が実行者の記録と、ホスティング運営者（`operator`）による `invitation.created` の記録がある | 実行者にホスティング運営者を指定して一覧する | `invitation.created` の記録だけを返す |
| TC-audit-listAuditLog-10 | A による `member.invited`、A による `token.issued`、別のメンバー B による `token.issued` がある | 操作の種類に `token.issued`、実行者に A を指定して一覧する | A による `token.issued` だけを返す |
| TC-audit-listAuditLog-11 | メンバー A が実行者の記録がある。A は、その後に利用を解除されている | 一覧する | 記録を返す。実行者は、記録の時点の A の名前を持つ |
| TC-audit-listAuditLog-12 | 記録がある | 期間の開始に、終了より後の日時を指定して一覧する | `ValidationError`（`InvalidPeriod`）。記録を返さない |
| TC-audit-listAuditLog-13 | 記録がある | 操作の種類に、`AuditOperationType` でない値を指定して一覧する | `ValidationError`（`UnknownOperationType`） |
| TC-audit-listAuditLog-15 | 記録がある | 操作の種類に空の集合を指定して一覧する | エラーにしない。空の列と 0 を返す |
| TC-audit-listAuditLog-17 | 実行者のメンバーは `workspace:manage` を持たない | 一覧する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限 `workspace:manage` を添える。記録を返さない |
