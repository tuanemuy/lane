# 復旧前の確認を示す（`previewRestore`）

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-installation-previewRestore-01 | `operator`。ワークスペース A、B を含むバックアップがある。現在の設置にはワークスペース A、C がある | そのバックアップを指定して復旧前の確認を示す | 復旧点はバックアップの取得日時。失われる可能性のある期間は、取得日時から現在時刻まで。戻るワークスペースは A（現在ある）と B（現在無い）。無くなるワークスペースは C。どれも ID と名称で示す。元に戻る操作の種類は、`RevertibleOperationKind` のすべての値。秘密情報の復旧条件として、バックアップが必要とする別に管理する秘密情報の識別子を返す。バックアップの大きさ（記録の `sizeBytes`）と、復旧できる大きさの上限（`InstallationEnvironment.restorableSizeLimit` の値）を返す。設置のデータと状態は変わらない |
| TC-installation-previewRestore-02 | `operator`。運営者が空の保存先を用意した設置（初期化の前）。ワークスペース A、B を含むバックアップがある | そのバックアップを指定して復旧前の確認を示す | 戻るワークスペースは A と B で、どちらも現在無い。無くなるワークスペースは無い |
| TC-installation-previewRestore-03 | `operator` | 存在しないバックアップを指定する | `NotFoundError` |
| TC-installation-previewRestore-05 | `operator`。復旧の進行中の設置（占有の期限の内） | バックアップを指定して復旧前の確認を示す | `BusinessRuleError`（`CommandNotAccepted`、理由 `restore_occupied`）。確認を返さない |
