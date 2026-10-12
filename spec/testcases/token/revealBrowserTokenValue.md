# ブラウザ用トークンの値を確かめる

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-token-revealBrowserTokenValue-01 | アプリ連携の権限を持つメンバー。手動発行したブラウザ用トークンがある | そのトークンの ID を指定する | 発行時に示した値と同じトークンの値を返す |
| TC-token-revealBrowserTokenValue-02 | アプリ連携の権限を持つメンバー。サーバー用トークンがある | そのトークンの ID を指定する | `BusinessRuleError`（`TokenValueNotStored`） |
| TC-token-revealBrowserTokenValue-03 | アプリ連携の権限を持つメンバー | 存在しないトークンの ID を指定する | `NotFoundError` |
| TC-token-revealBrowserTokenValue-05 | アプリ連携の権限を持たないメンバー。ブラウザ用トークンがある | そのトークンの ID を指定する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限（`integration:manage`）を示す。値を返さない |
| TC-token-revealBrowserTokenValue-06 | アプリ連携の権限を持つメンバー。失効済みのブラウザ用トークンがある | そのトークンの ID を指定する | 値を返さず、失効済みであることを返す |
| TC-token-revealBrowserTokenValue-07 | アプリ連携の権限を持つメンバー。期限から `OneTimeValueRetention.ms` を過ぎた、手動発行のブラウザ用トークンがある。定期ジョブはまだ削除していない | そのトークンの ID を指定する | `NotFoundError`。値を返さない |
| TC-token-revealBrowserTokenValue-08 | アプリ連携の権限を持つメンバー。期限を過ぎ、期限から `OneTimeValueRetention.ms` を過ぎていない、手動発行のブラウザ用トークンがある | そのトークンの ID を指定する | トークンの値を返す |
