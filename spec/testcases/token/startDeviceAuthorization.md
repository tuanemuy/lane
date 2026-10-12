# スクリプトの認証を始める

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-token-startDeviceAuthorization-01 | セッションもトークンも示さない | スキーマ反映とコンテンツ管理の許可を指定して認証を始める | 承認の値、`UserCode.format` の形の確認用のコード、受け取りの値、依頼の期限を返す。依頼は `pending` で、指定した許可を持ち、ワークスペースを持たない。期限は作成の `TokenLifetimePolicy.deviceAuthorizationMs` 後 |
| TC-token-startDeviceAuthorization-03 | セッションもトークンも示さない | 語彙にない許可を指定した要求と、許可を一つも指定しない要求で、それぞれ認証を始める | それぞれ `ValidationError`（`UnknownGrant`、`EmptyGrants`）。依頼を作らない |
