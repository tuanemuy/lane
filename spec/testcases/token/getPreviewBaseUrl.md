# プレビュー先の表示アプリの URL を確かめる

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-token-getPreviewBaseUrl-01 | アプリ連携の権限を持つメンバー。URL は設定済み | 現在の URL を確かめる | 設定済みの URL を返す |
| TC-token-getPreviewBaseUrl-02 | アプリ連携の権限を持つメンバー。URL は未設定 | 現在の URL を確かめる | 未設定であることを返す |
| TC-token-getPreviewBaseUrl-03 | アプリ連携の権限を持たないメンバー（編集権限とメンバー管理の権限を持つ） | 現在の URL を確かめる | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限（`integration:manage`）を示す |
