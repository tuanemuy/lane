# プレビュー先の表示アプリの URL を取り除く

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-token-clearPreviewBaseUrl-01 | アプリ連携の権限を持つメンバー。URL は `https://app.example.com` | URL を取り除く | 正常に終わる。設定は未設定になる。`preview.url_changed` の操作履歴を残す。`changes` の `baseUrl` は、変更前が取り除いた URL、変更後が `null` |
| TC-token-clearPreviewBaseUrl-04 | アプリ連携の権限を持たないメンバー。URL は設定済み | URL を取り除く | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限（`integration:manage`）を示す。設定は変わらない。操作履歴を追記しない |
