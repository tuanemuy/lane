# バックアップから設置全体を復旧する（`restoreInstallation`）

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-installation-restoreInstallation-01 | `operator`。稼働中の設置。整合性の確認に成功し、実行中の CMS が扱うデータの版と別に管理する秘密情報が一致するバックアップがある。取得の後に、コンテンツの作成とワークスペースの追加を行っている | そのバックアップを指定して復旧する | 復旧点（バックアップの取得日時）と、設置が利用再開前であることを返す。設置全体のデータはバックアップの内容になり、取得の後に作成したコンテンツとワークスペースは残らない。復旧の記録は `restored` で、復旧に使ったバックアップと復旧点を持つ。`Installation` は、バックアップの中の値になる。利用者の要求の入口からの設置の状態の確認は、受け付けない。バックアップは変わらない |
| TC-installation-restoreInstallation-02 | `operator`。バックアップの取得の後に、コンテンツの個別の削除、メンバーの解除、権限の縮小、トークンの失効を行っている | そのバックアップを指定して復旧する | 削除したコンテンツ、解除したメンバー、縮小する前の権限、失効する前のトークンが、バックアップの時点の状態に戻る |
| TC-installation-restoreInstallation-03 | `operator`。復旧した設置 | 初期構成の宣言を添えて起動する | 初期構成を読み込まない。設定はバックアップの内容のまま |
| TC-installation-restoreInstallation-04 | `operator`。初期化の前の設置（運営者が空の保存先を用意した設置。`Installation` と復旧の記録が無い）。復旧できるバックアップがある | そのバックアップを指定して復旧する | 設置全体のデータはバックアップの内容になる。`Installation` はバックアップの中の値になり、復旧の記録は `restored`。設置は利用再開前になる |
| TC-installation-restoreInstallation-05 | `operator`。保存先が空で、完了した復旧の記録（`restored`）だけが残る設置。復旧できるバックアップがある | そのバックアップを指定して復旧する | 復旧を受け付ける。設置全体のデータはバックアップの内容になり、復旧の記録は、そのバックアップの `restored` になる |
| TC-installation-restoreInstallation-06 | `operator`。データの保存先の設定は正しい。保存先の障害で、設置の区画を読めない設置。復旧の記録は無い | バックアップを指定して復旧する | `SystemError`。復旧を始めず、復旧の記録を作らない。運営者が空の保存先を用意した後の復旧は、成功する |
| TC-installation-restoreInstallation-07 | `operator`。初期化の前の設置。データの置き換えが失敗する | 復旧する | `SystemError`（理由 `RESTORE_FAILED`）。復旧の記録は `failed` で残る。設置は復旧の失敗で、起動しても初期化しない |
| TC-installation-restoreInstallation-08 | `operator` | 存在しないバックアップを指定して復旧する | `NotFoundError`。設置のデータと状態は変わらない |
| TC-installation-restoreInstallation-09 | `operator`。整合性の確認に失敗したバックアップがある | そのバックアップを指定して復旧する | `BusinessRuleError`（`BackupNotRestorable`、理由 `integrity_failed`）。復旧を実行せず、設置のデータは変わらない。設置は稼働中のまま |
| TC-installation-restoreInstallation-10 | `operator`。データの版が、実行中の CMS が扱うデータの版と違うバックアップがある | そのバックアップを指定して復旧する | `BusinessRuleError`（`BackupNotRestorable`、理由 `incompatible_version`）。設置のデータと状態は変わらない |
| TC-installation-restoreInstallation-11 | `operator`。設置の鍵（`SecretCipher.currentKeyId` の値）が、バックアップの `secretKeyId` と違う | そのバックアップを指定して復旧する | `BusinessRuleError`（`BackupNotRestorable`、理由 `secret_unavailable`）。設置のデータと状態は変わらない |
| TC-installation-restoreInstallation-12 | `operator`。秘密情報の鍵の設定の形は正しく、鍵を読めない設置 | バックアップを指定して復旧する | `SystemError`（理由 `SECRET_UNAVAILABLE`）。復旧を始めず、復旧の記録を作らない。設置のデータと状態は変わらない |
| TC-installation-restoreInstallation-13 | `operator`。稼働中の設置。ワークスペースが 2 つある。データの置き換えが、一方のワークスペースの区画を置き換えた後に失敗する | 復旧する | `SystemError`（理由 `RESTORE_FAILED`）。失敗と理由を示し、復旧を完了と扱わない。復旧の記録は `failed` になり、設置は復旧の失敗。利用者の要求を受け付けず、利用再開前の確認と利用の再開を受け付けない |
| TC-installation-restoreInstallation-14 | `operator`。上の失敗の直後の設置（一方の区画だけがバックアップの内容。復旧の記録が `failed`）。または、置き換えの打ち切り（`RESTORE_TIMED_OUT`）の直後の設置で、置き換えが時間内に終わる資源を用意している | すぐに、同じバックアップ、または別のバックアップを指定して復旧する | 占有の期限を待たずに、復旧が始まって成功する。設置全体のデータは、前の復旧が置き換えた区画を含めて、指定したバックアップの内容になる。復旧の記録は、そのバックアップの `restored` になる |
| TC-installation-restoreInstallation-16 | `operator`。稼働中の設置。データの置き換えが、打ち切りの時間（`Recovery.restoreTimeoutMs`）の内に終わらない | 復旧する | `SystemError`（理由 `RESTORE_TIMED_OUT`）。時間内に終えられなかったことを理由とする失敗として示し、復旧を完了と扱わない。復旧の記録は `failed` になり、設置は復旧の失敗 |
| TC-installation-restoreInstallation-18 | `operator`。稼働中の設置。データの置き換えは成功し、復旧の完了の記録の保存が、記録の保存先の障害で失敗する | 復旧する | `SystemError`。設置のデータはバックアップの内容で、復旧の記録は `restoring` のまま残る。占有の期限を過ぎた後に、同じバックアップで復旧し直すと、`restored` になる |
| TC-installation-restoreInstallation-19 | `operator`。結果が分からないまま中断した復旧の設置（復旧の記録が `restoring`）。復旧の開始から `Recovery.leaseMs` が過ぎていない | バックアップを指定して復旧する | `BusinessRuleError`（`CommandNotAccepted`、理由 `restore_occupied`）。復旧を始めず、復旧中であることと、始め直せる時期を示す。復旧の記録と設置のデータは変わらない |
| TC-installation-restoreInstallation-20 | `operator`。結果が分からないまま中断した復旧の設置。復旧の開始から `Recovery.leaseMs` が過ぎている | 同じバックアップ、または別のバックアップを指定して復旧する | 復旧が成功する。設置全体のデータは指定したバックアップの内容になり、復旧の記録は、そのバックアップの `restored` になる |
| TC-installation-restoreInstallation-21 | `operator`。利用再開前の設置（`restored`） | 別のバックアップを指定して復旧する | 設置全体のデータは指定したバックアップの内容になる。復旧の記録は、新しい復旧点の `restored` になる |
| TC-installation-restoreInstallation-22 | `operator`。更新中の設置（更新の進行が `updating`）。CMS を更新前の版の CMS に戻している。更新の前に取得したバックアップがある | そのバックアップを指定して復旧する | 復旧を受け付け、設置は利用再開前になる。`Installation` はバックアップの中の値になり、更新の進行は `idle`。利用再開前の確認と利用の再開の後、設置は、更新前の版で稼働中になる |
| TC-installation-restoreInstallation-23 | `operator`。更新中の設置。実行中の CMS は更新先の版。更新の前に取得したバックアップがある | そのバックアップを指定して復旧する | `BusinessRuleError`（`BackupNotRestorable`、理由 `incompatible_version`）。設置のデータと `Installation` は変わらない |
| TC-installation-restoreInstallation-24 | `operator`。稼働中の設置。復旧の記録は無い。バックアップの取得が進行中（`capture` の呼び出しが進む間） | 復旧できるバックアップを指定して復旧する | `ConflictError`（理由 `BACKUP_IN_PROGRESS`）。開始を取り下げ、復旧の記録は無いまま。設置のデータは変わらず、設置は稼働中。取得が完了した後に、同じバックアップを指定して実行し直した復旧は、成功する |
| TC-installation-restoreInstallation-25 | `operator`。復旧の記録が `failed` の設置。指定したバックアップを、開始の保存の確定の後、`BackupStore.restore` の前に、運営者が保管先で削除する | そのバックアップを指定して復旧する | `NotFoundError`。開始を取り下げ、復旧の記録は前の `failed` の内容に戻る。設置のデータは変わらず、設置は復旧の失敗のまま。続けて「バックアップを一覧する」は、削除したバックアップを返さない。一覧が返したバックアップを指定し直した復旧は、成功する |
| TC-installation-restoreInstallation-26 | `operator`。稼働中の設置。復旧できるバックアップが 2 件ある。一つ目の復旧が、開始の保存を確定している | 二つ目の復旧が、開始の前に読んだ状態（復旧の記録が無い）から、別のバックアップを指定して開始の保存をする | 二つ目の復旧は `ConflictError`（理由 `STALE_VERSION`）になり、設置のデータを変えない。設置全体のデータは、一つ目の復旧のバックアップの内容になり、復旧の記録は、そのバックアップの `restored` になる |
| TC-installation-restoreInstallation-27 | `operator`。稼働中の設置。バックアップの取得の後に、メディアを登録し、ワークスペースを作成してメディアを登録している | そのバックアップを指定して復旧する | 取得の後に保管した実体と、復旧で無くなったワークスペースの実体は、メディアの保管先に残らない。バックアップが含むメディアの実体は、取得の時点の内容で取り出せる |
| TC-installation-restoreInstallation-28 | `operator`。稼働中の設置。復旧できるバックアップがある | 運用コマンドの入口でそのバックアップを指定して復旧を実行し、入口が示した復旧前の確認に、確認を返さない | 入口は「バックアップから設置全体を復旧する」を呼ばない。復旧の記録を作らず、設置のデータと状態は変わらない |
| TC-installation-restoreInstallation-29 | `operator`。稼働中の設置。記録の `sizeBytes` が `InstallationEnvironment.restorableSizeLimit` の値を超える、復旧できるバックアップがある | そのバックアップを指定して復旧する | 大きさで拒否せずに復旧する。復旧点を返し、設置全体のデータはバックアップの内容になる。復旧の記録は `restored` で、設置は利用再開前になる |
