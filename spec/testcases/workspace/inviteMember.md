# メンバーを招待する

ユースケースは [../../usecases/workspace.md](../../usecases/workspace.md) が定める。

- W は `active` のワークスペース。実行者 X は、W の `member:manage` を持つメンバー
- e は、W のメンバーのアカウントのメールアドレスでなく、W の `pending` の招待（メンバーの招待と初期所有者の招待）の宛先でもないメールアドレス
- 断りが無ければ、設置は設置全体のメール送信を使え、送信は `accepted` になる

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-workspace-inviteMember-01 | なし | 宛先 e と、編集者の役割を指定して招待する | 宛先 e、編集者の権限の組、現在時刻の `InvitationPolicy.validMs` 後の有効期限を持つ招待と、送信の結果 `accepted` を返す。招待の値を返さない。W の名称、設置の公開の URL と招待の値から `InvitationLink.url` で作った招待の URL、有効期限を含むメールを、e へ一通送る。「未使用の招待を一覧する」が、この招待を返す |
| TC-workspace-inviteMember-02 | なし | 宛先 e と、編集者の役割を指定して招待する | 操作履歴に `member.invited` を一件残す。実行者は X、対象は作った招待で、名称は e。招待の値を含めない |
| TC-workspace-inviteMember-03 | なし | 宛先 e と、権限の列 `submission:read` を指定して招待する | 招待を返す。権限の組は `submission:read`、役割は無し |
| TC-workspace-inviteMember-04 | なし | 宛先 e と、空の権限の列を指定して招待する | 招待を返す。権限の組は空、役割は `viewer` |
| TC-workspace-inviteMember-05 | 設置全体のメール送信を使えない設置 | 宛先 e と、編集者の役割を指定して招待する | 招待と、送信の結果 `not_configured`、招待の値を返す。URL を返さない。メールを送らない。招待と操作履歴は残る。返した値で「招待の内容と状態を確かめる」が `acceptable` を返す |
| TC-workspace-inviteMember-06 | メールの送信が `failed` になる | 宛先 e と、編集者の役割を指定して招待する | 招待と、送信の結果 `failed` を返す。招待の値を返さない。招待と操作履歴は残る。同じ宛先を招待し直せる。送信の結果とエラー情報は、招待の値を含まない |
| TC-workspace-inviteMember-08 | なし | 宛先 e と、語彙にない権限を含む権限の列を指定して招待する | `ValidationError`（`UnknownPermission`）。招待を作らず、操作履歴を残さず、メールを送らない |
| TC-workspace-inviteMember-09 | m は、W のメンバーのアカウントのメールアドレス | 宛先 m を指定して招待する | `BusinessRuleError`（`AlreadyMember`）。招待を作らず、操作履歴を残さず、メールを送らない |
| TC-workspace-inviteMember-10 | W に、宛先 e の、期限内の `pending` の招待 I がある | 宛先 e と、閲覧者の役割を指定して招待する | 新しい招待と、前の招待を置き換えたことを返す。I は `revoked` になる。新しい招待の URL を含むメールを e へ一通送る。操作履歴に `member.invited` を一件残し、`member.invitation_revoked` を残さない。「未使用の招待を一覧する」は、新しい招待だけを返す。I の値で「招待の内容と状態を確かめる」が `revoked` を返す |
| TC-workspace-inviteMember-11 | W に、宛先 e の、有効期限を過ぎた `pending` の招待 I がある | 宛先 e と、編集者の役割を指定して招待する | 新しい招待を返す。前の招待を置き換えたことを示さない。I は `revoked` になる。操作履歴に `member.invited` を一件残す |
| TC-workspace-inviteMember-13 | W は、所有者がいない（`vacant`）ワークスペース。W に、宛先 e の、期限内の `pending` の初期所有者の招待 K と、宛先 e の、期限内の `pending` のメンバーの招待 I がある | 宛先 e と、閲覧者の役割を指定して招待する | 新しい招待と、前の招待を置き換えたことを返す。I は `revoked` になる。K は `pending` のまま変わらない |
| TC-workspace-inviteMember-15 | 実行者 Y は、W の、`member:manage` を持たないメンバー | Y が、宛先 e を指定して招待する | `ForbiddenError`（理由 `PERMISSION_MISSING`）。不足する権限に `member:manage` を添える。招待を作らず、操作履歴を残さず、メールを送らない |
