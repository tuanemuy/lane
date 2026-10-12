# プレビュー先の表示アプリの URL を設定する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-token-setPreviewBaseUrl-01 | アプリ連携の権限を持つメンバー。URL は未設定 | `https://app.example.com` を設定する | 保存した URL を返す。設定は URL を持つ状態になる。`preview.url_changed` の操作履歴を残す。`changes` の `baseUrl` は、変更前が `null`、変更後が設定した URL |
| TC-token-setPreviewBaseUrl-05 | アプリ連携の権限を持つメンバー。URL は設定済み | 形式が正しくない URL（スキームが `http` と `https` のどちらでもない）を設定する | `ValidationError`（`InvalidPreviewBaseUrl`）。理由を示す。設定は変わらない。操作履歴を追記しない |
| TC-token-setPreviewBaseUrl-06 | アプリ連携の権限を持たないメンバー（編集権限とメンバー管理の権限を持つ） | URL を設定する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限（`integration:manage`）を示す。設定は変わらない。操作履歴を追記しない |
