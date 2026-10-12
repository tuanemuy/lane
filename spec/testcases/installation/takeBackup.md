# バックアップを取得する（`takeBackup`）

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-installation-takeBackup-01 | `operator`。ワークスペースが 2 つある稼働中の設置 | バックアップを取得する | バックアップの記録を返す。取得日時（設置の区画の写しの時点）、名簿にある 2 つのワークスペース、整合性 `verified`、取得日時に `BackupPolicy.retentionMs` を加えた保持・削除期限、設置のデータの版、取得の時点の別に管理する秘密情報の識別子、写しの大きさを持つ。秘密情報の値を含まない。記録は一覧に現れる。設置のデータは変わらない |
| TC-installation-takeBackup-02 | `operator`。更新が必要な設置（保存済みのデータの版が、実行中の CMS が扱う版と違う。更新の進行は `idle`） | バックアップを取得する | バックアップの記録を返す。記録のデータの版は、保存済みの版 |
| TC-installation-takeBackup-03 | `operator`。更新中の設置（更新の進行が `updating`） | バックアップを取得する | `BusinessRuleError`（`CommandNotAccepted`、理由 `update_in_progress`）。更新を始めて完了していないことと、更新先の版を示す。取得を始めず、バックアップは残らない |
| TC-installation-takeBackup-06 | `operator`。取得したバックアップの整合性の確認が失敗する | バックアップを取得する | エラーにせず、整合性が `failed` で理由を持つ記録を返す。この記録を指定した復旧は `BusinessRuleError`（`BackupNotRestorable`、理由 `integrity_failed`）になる |
| TC-installation-takeBackup-07 | `operator`。バックアップの保管先の容量が足りない | バックアップを取得する | `SystemError`（理由 `BACKUP_CAPACITY_EXCEEDED`）。途中で失敗した取得は一覧に現れない |
| TC-installation-takeBackup-08 | `operator`。バックアップの保管先へ接続できない | バックアップを取得する | `SystemError`（理由 `BACKUP_DESTINATION_UNREACHABLE`）。一覧に現れない |
| TC-installation-takeBackup-09 | `operator`。写しの取得の後、取得を完了する前に中断する | バックアップを取得する | 中断した取得は一覧に現れず、復旧に指定すると `NotFoundError` になる |
| TC-installation-takeBackup-10 | `operator`。前の取得が完了する前に中断している | バックアップをもう一度取得する | 新しいバックアップの記録を返す。一覧は、新しいバックアップだけを返す。中断した取得の写しは保管先に残らない |
