# バックアップを一覧する（`listBackups`）

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-installation-listBackups-01 | `operator`。取得日時の異なるバックアップの記録が 3 件ある | 一覧する | 3 件を取得日時の新しい順に返し、総件数 3 を返す。記録ごとに、ID、取得日時、データの版、整合性の確認の結果、保持・削除期限、期限を過ぎているかどうか、含むワークスペースを返す |
| TC-installation-listBackups-02 | `operator`。整合性の確認に失敗したバックアップの記録がある | 一覧する | その記録を、整合性 `failed` として返す |
| TC-installation-listBackups-03 | `operator`。保持・削除期限を過ぎたバックアップの記録がある | 一覧する | その記録を、期限を過ぎたものとして返す。バックアップと記録は削除されない |
| TC-installation-listBackups-04 | `operator`。運営者が保管先で削除したバックアップと、途中で失敗・中止した取得がある | 一覧する | どちらも返さず、総件数に数えない |
| TC-installation-listBackups-06 | `operator`。バックアップが一つも無い | 一覧する | 空の列と総件数 0 を返す。エラーにしない |
| TC-installation-listBackups-08 | `operator`。データの保存先の設定は正しい。保存先の障害で、設置の区画を読めない設置。バックアップが 2 件ある | 一覧する | エラーにせず、2 件を返す。設置のデータを読まない |
