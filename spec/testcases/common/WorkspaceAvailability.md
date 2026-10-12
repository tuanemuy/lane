# 利用できるワークスペース

[../../domains/index.md](../../domains/index.md) の「利用できるワークスペース」の代表ケース。どのユースケースにも同じ規則で当たるので、ユースケースごとのファイルに置かない。

- 「利用終了の途中の C」は、名簿の項目が終了中（`retiring`）のワークスペース。断りが無ければ、区画の `Workspace` は `active` のまま
- 「名簿に項目が無い C」は、利用終了を完了し、名簿から外したワークスペース
- 値からワークスペースを求める入口と、可否を決める振る舞いごとの結果は、その入口のユースケースのテストが持つ。示されたトークンを確かめる段は [../token/authenticateToken.md](../token/authenticateToken.md)（`TC-token-authenticateToken-06`、`-12`）、トークンの認可は [../token/authorizeTokenOperation.md](../token/authorizeTokenOperation.md)（`TC-token-authorizeTokenOperation-18`、`-22`）、プレビューの引き換えは [../token/redeemPreviewExchange.md](../token/redeemPreviewExchange.md)（`TC-token-redeemPreviewExchange-08`、`-09`）、スクリプトの認証の受け取りは [../token/redeemDeviceAuthorization.md](../token/redeemDeviceAuthorization.md)（`TC-token-redeemDeviceAuthorization-11`）、招待の内容の確認と受諾は [../workspace/inspectInvitation.md](../workspace/inspectInvitation.md) と [../workspace/acceptInvitation.md](../workspace/acceptInvitation.md)、メディアの配信は [../media/deliverMedia.md](../media/deliverMedia.md)、フォームの受付の応答は [../app-api/httpEntry.md](../app-api/httpEntry.md) の「利用できないワークスペース」

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-common-WorkspaceAvailability-01 | 実行者は、C の、すべての権限を持つメンバー。C に未公開のコンテンツ A がある。C は、利用終了の途中（区画が `active`）、利用終了の途中（区画が `closing`）、名簿で準備中（区画が `active`）、名簿に項目が無い、の四つの場合 | A の下書きを保存する（Content の `saveDraft`） | どの場合も `NotFoundError`。C の情報を示さない。A は変わらず、操作履歴も残らない |
| TC-common-WorkspaceAvailability-02 | 実行者は、C の `content:edit` を持つメンバー。C は利用終了の途中で、スキーマの反映の適用中 | C のコンテンツの下書きを保存する（Content の `saveDraft`） | `NotFoundError`。`UnavailableError`（`SCHEMA_APPLYING`）にしない |
| TC-common-WorkspaceAvailability-03 | 実行者は、利用できるワークスペース W の、`content:edit` を持たないメンバー。W はスキーマの反映の適用中 | W のコンテンツの下書きを保存する（Content の `saveDraft`） | `ForbiddenError`（`PERMISSION_MISSING`）。`UnavailableError`（`SCHEMA_APPLYING`）にしない |
| TC-common-WorkspaceAvailability-04 | C は利用終了の途中。C のフォームの回答 S の通知は `waiting`。送信接続設定は登録済み | S の `form.submission_accepted` を消費する（Form の `sendSubmissionNotification`） | エラーにせずに終える。`MailSender` を呼ばない。S の通知は `waiting` のまま。送信失敗も記録しない |
| TC-common-WorkspaceAvailability-05 | アカウント A は、利用できるワークスペース W と、利用終了の途中の C のメンバー。A の W と C の項目は `joined` | 所属するワークスペースを一覧する（Workspace の `listMyWorkspaces`） | W だけを示す。C を示さず、エラーにしない |
