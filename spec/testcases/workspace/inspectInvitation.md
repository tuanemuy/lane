# 招待の内容と状態を確かめる

ユースケースは [../../usecases/workspace.md](../../usecases/workspace.md) が定める。

- I は、「メンバーを招待する」が `active` のワークスペース W に作った、宛先 e、編集者の権限の組の招待。断りが無ければ、`pending` で期限内

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-workspace-inspectInvitation-01 | e のアカウントが無い。ログインしていない | I の値で確かめる | `acceptable`、W の名称、宛先 e、編集者の権限の組、有効期限、宛先のアカウントが無いことを返す。I は `pending` のまま変わらない |
| TC-workspace-inspectInvitation-02 | e のアカウント A がある。A でログインしている | I の値と A で確かめる | `acceptable` と、宛先のアカウントが既にあることを返す |
| TC-workspace-inspectInvitation-03 | e のアカウント A がある。別のメールアドレスのアカウント B でログインしている | I の値と B で確かめる | `email_mismatch`、W の名称、宛先 e、編集者の権限の組、有効期限を返す。同じ状態の「招待を受諾する」は `InvitationEmailMismatch` を返す |
| TC-workspace-inspectInvitation-04 | e のアカウント A がある。別のメールアドレスのアカウント B は、W のメンバー。B でログインしている | I の値と B で確かめる | `email_mismatch` を返す。`invalid` にしない |
| TC-workspace-inspectInvitation-05 | N は、作成したばかりで、初期所有者が未参加のワークスペース（所有者がいない `active`。名簿で開設済み）。K は、N の、`pending` で期限内の初期所有者の招待 | K の値で確かめる | `acceptable`、N の名称、所有者に指定すること、有効期限を返す |
| TC-workspace-inspectInvitation-06 | I は `pending` で、有効期限を過ぎている | I の値で確かめる | `expired` だけを返す。W の名称と付与内容を返さない。エラーにしない |
| TC-workspace-inspectInvitation-07 | I は `accepted` | I の値で確かめる | `accepted` だけを返す。W の名称と付与内容を返さない。エラーにしない |
| TC-workspace-inspectInvitation-08 | I は `revoked` | I の値で確かめる | `revoked` だけを返す。W の名称と付与内容を返さない。エラーにしない |
| TC-workspace-inspectInvitation-09 | 入力する値に該当する招待が無い | その値で確かめる | `invalid` だけを返す。エラーにしない。応答は、入力した招待の値を含まない |
| TC-workspace-inspectInvitation-12 | I は `pending` で期限内。W は、所有者がいない（`vacant`）ワークスペース | I の値で確かめる | `acceptable` と、W の名称、編集者の権限の組を返す |
| TC-workspace-inspectInvitation-13 | J は、利用終了の途中のワークスペース Cの、`pending` で期限内の招待 | J の値で確かめる | `invalid` だけを返す。C の名称と付与内容を返さない。該当する招待が無い値と同じ応答を返す |
| TC-workspace-inspectInvitation-15 | e のアカウント A がある。A は、初期所有者の招待の受諾で W の所有者になった。I は `pending` で期限内。A でログインしている | I の値と A で確かめる | `invalid` だけを返す。同じ状態の「招待を受諾する」は `InvitationInvalid` を返す |
| TC-workspace-inspectInvitation-16 | e のアカウント A がある。A は、初期所有者の招待の受諾で W の所有者になった。I は `pending` で期限内。ログインしていない | I の値で確かめる | `acceptable` と、宛先のアカウントが既にあることを返す |
