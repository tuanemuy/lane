# 送信接続設定を取得する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-getMailConnection-01 | 送信接続設定が登録済み | `notification:manage` を持つメンバーが取得する | 接続先と接続条件と、`credentialsRegistered` が真の `MailConnectionView` を返す。認証情報の値（`identity` と秘密）を含まない |
| TC-form-getMailConnection-02 | 送信接続設定が未登録 | `notification:manage` を持つメンバーが取得する | 未登録であることを返す |
| TC-form-getMailConnection-03 | 送信接続設定が登録済み | `notification:manage` を持たないメンバーが、取得する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `notification:manage` |
