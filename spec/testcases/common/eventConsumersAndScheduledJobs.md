# イベントの消費者と定期ジョブ

[../../domains/index.md](../../domains/index.md) の「イベントの消費者と定期ジョブ」の代表ケース。どの消費者と定期ジョブにも同じ規則で当たるので、ユースケースごとのファイルに置かない。一件の失敗の後に残りの対象を続けるかどうかは、各ユースケースのテストが持つ。

- 断りが無ければ、ワークスペース W は利用できる。実行者は `system`

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-common-eventConsumersAndScheduledJobs-01 | W のフォームの回答 S の `form.submission_accepted` が配送の対象にある。S は、回答の削除で、通知とともに削除済み | イベントを消費する（Form の `sendSubmissionNotification`） | エラーにせずに終える。何も書かず、`MailSender` を呼ばない。イベントは処理を終えたものになる |
| TC-common-eventConsumersAndScheduledJobs-02 | W の `preparing` の反映のジョブ J の `schema.apply_requested` が配送の対象にある。W の区画の `Workspace` は `closing` | イベントを消費する（Schema の `prepareSchemaApply`） | エラーにせずに終える。J、`WorkspaceSchema`、準備領域は変わらない |
| TC-common-eventConsumersAndScheduledJobs-03 | W の `preparing` の反映のジョブ J の `schema.apply_requested` が配送の対象にある。消費が J を読んだ後、確定の前に、W の利用終了の開始が確定する（確定が `ConflictError`（`WORKSPACE_NOT_ACTIVE`）で拒否される） | イベントを消費する（Schema の `prepareSchemaApply`） | エラーにせずに終える。J と準備領域は、消費の前のまま |
| TC-common-eventConsumersAndScheduledJobs-04 | W のフォームの回答 S の `form.submission_accepted` が配送の対象にある。一つ目の UnitOfWork の確定が `SystemError` で失敗する | イベントを消費する（Form の `sendSubmissionNotification`） | 消費がエラーを投げて終える。S の通知は `waiting` のまま。イベントは配送の対象に残り、再配送された消費が通知を送る |
| TC-common-eventConsumersAndScheduledJobs-05 | 名簿で開設済みのワークスペース W1 と W2 に、保持方針の期限を過ぎた履歴の版がある。W1 の削除の確定が `SystemError` で失敗する | 定期ジョブ「履歴の版の整理」を起動する（Content の `cleanUpExpiredRevisions`） | W2 の版を削除する。W1 の、確定しなかった削除の版は残り、次の起動が削除する |
| TC-common-eventConsumersAndScheduledJobs-06 | W の `WorkspaceSchema` はスキーマの反映の適用中。W に、保持方針の期限を過ぎた履歴の版がある | 定期ジョブ「履歴の版の整理」を起動する（Content の `cleanUpExpiredRevisions`） | エラーにせず、何も書かずに、W の処理を終える。版は残り、適用中でなくなった後の起動が削除する |
