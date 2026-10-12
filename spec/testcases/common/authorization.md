# 認可

[../../domains/index.md](../../domains/index.md) の「認可」の代表ケース。どのユースケースにも同じ規則で当たるので、ユースケースごとのファイルに置かない。利用できないワークスペースへの結果は [WorkspaceAvailability.md](WorkspaceAvailability.md) が、メンバーの認可の結果を求める振る舞いは [../workspace/resolveMemberAccess.md](../workspace/resolveMemberAccess.md) が、トークンの認可の手続き（`GRANT_MISSING`、`OUT_OF_SCOPE` を含む）は [../token/authorizeTokenOperation.md](../token/authorizeTokenOperation.md) が持つ。

- 断りが無ければ、ワークスペース W は利用できる

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-common-authorization-01 | W に送信接続設定が登録済み。実行者は、すべての権限を持つメンバーが手動発行した、W の `appToken` | 送信接続設定を取得する（Form の `getMailConnection`） | `ForbiddenError`（`ACTOR_NOT_ALLOWED`）。送信接続設定を返さない |
| TC-common-authorization-02 | W に未公開のコンテンツ A がある。`content:edit` を持つメンバー M と、M が手動発行した、コンテンツ管理の許可を持ち範囲が A のモデルを含む `appToken` がある | M と、そのトークンの実行者が、それぞれ同じ値で A の下書きを保存する（Content の `saveDraft`） | どちらも保存する。結果の種類と、A に残る内容は同じ |
| TC-common-authorization-03 | 実行者は、W の `content:edit` だけを持つメンバー。W に下書きを持つ未公開のコンテンツ A がある。A の下書きは、未公開のコンテンツ T を必須の参照で参照するので、公開できない | A を公開する（Content の `publishContent`） | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限 `content:publish` を添える。公開できない理由を返さない。A は変わらない |
| TC-common-authorization-04 | W は多要素認証を必須にしている。実行者は、多要素認証を設定していない、W のすべての権限を持つメンバー | W のメンバーを一覧する（Workspace の `listMembers`） | `ForbiddenError`（`MFA_SETUP_REQUIRED`） |
| TC-common-authorization-05 | 実行者は、W の `content:edit` と `content:publish` を持つメンバー M。M が A を一度公開した後、「メンバーの権限を変更する」が M の権限の組を `content:edit` にした。別の場合では、「メンバーの利用を解除する」が M を解除した。M のセッションは残っている | M が、公開を取り消した A をもう一度公開する（Content の `publishContent`） | 権限の変更の後は `ForbiddenError`（`PERMISSION_MISSING`）、解除の後は `NotFoundError`。A は変わらない |
