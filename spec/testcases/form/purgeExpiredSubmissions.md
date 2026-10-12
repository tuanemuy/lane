# 保持期限を過ぎた回答を自動整理する

断りが無い限り、回答は、定期ジョブが対象にするワークスペース W のもの。「定期ジョブが自動整理を実行する」は、W を対象に、このユースケースを呼ぶことを指す。回答が残らない削除済みフォームの運用設定の削除は、[disposeRemovedFormSettings.md](disposeRemovedFormSettings.md) が持つ。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-purgeExpiredSubmissions-01 | W に、保持期限を過ぎた回答 S1 と、保持期限を過ぎていない回答 S2 | 定期ジョブが、W を対象に自動整理を実行する | 1 を返す。S1 が、その通知とともに無くなり、S2 は残る |
| TC-form-purgeExpiredSubmissions-02 | 保持期限を過ぎた回答が無い | 定期ジョブが自動整理を実行する | 0 を返す。どの回答も削除しない |
| TC-form-purgeExpiredSubmissions-04 | 保持期限を過ぎた回答 S1 の通知は最初の通知の送信待ち（`attempt` は `initial`、状態は `waiting`）、S2 の通知は送信中（`sending`） | 定期ジョブが自動整理を実行する | S1 と S2 が、その通知とともに無くなる |
| TC-form-purgeExpiredSubmissions-05 | W は利用終了の途中。保持期限を過ぎた回答 S1 がある | 定期ジョブが、W を対象に自動整理を実行する | 1 を返す。S1 が無くなる |
| TC-form-purgeExpiredSubmissions-06 | 削除済みフォーム F の `removed` の設定があり、F の回答は、保持期限を過ぎた S1 だけ | 定期ジョブが自動整理を実行する | S1 が無くなる。F の設定は変わらずに残る |
| TC-form-purgeExpiredSubmissions-07 | W に、保持期限を過ぎた回答が `BoundedDeletion.maxLimit` 件より多くある。前の起動が、一部の UnitOfWork を確定した後に中断した | 定期ジョブが自動整理を実行する | 前の起動が確定した削除は残る。残りの保持期限を過ぎた回答が、その通知とともに無くなり、この起動で削除した件数を返す |
