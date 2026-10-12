# 自分に紐付くトークンを失効させる

- 実行者は、入口が「示されたトークンを確かめる」で作った `Actor`

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-token-revokeOwnToken-01 | スクリプトの認証で発行した `active` のトークンがある。実行者は、そのトークンの `memberToken` | 失効させる | 正常に終わる。トークンは `revoked` になる。`token.revoked` の操作履歴を、実行者を `memberToken`（そのトークンと、紐付くメンバー）として残す。操作履歴はトークンの値を含まない |
| TC-token-revokeOwnToken-02 | スクリプトの認証で発行した `active` のトークンがあり、紐付くメンバーは解除済み。実行者は、そのトークンの `memberToken` | 失効させる | 正常に終わる。トークンは `revoked` になる。`token.revoked` の操作履歴を残す |
| TC-token-revokeOwnToken-05 | 手動発行した `active` のトークンがある。実行者は、そのトークンの `appToken` | 失効させる | `ForbiddenError`（`ACTOR_NOT_ALLOWED`）。トークンは `active` のまま。操作履歴を追記しない |
| TC-token-revokeOwnToken-06 | プレビューの引き換えで発行した `active` のトークンがある。実行者は、そのトークンの `memberToken` | 失効させる | `ForbiddenError`（`SELF_REVOCATION_NOT_ALLOWED`）。トークンは `active` のまま。操作履歴を追記しない |
| TC-token-revokeOwnToken-07 | スクリプトの認証で発行した `active` のトークン。入口が確かめた後に、発行元のワークスペースが利用終了の途中になった。区画の `Workspace` は `active` のまま。実行者は、そのトークンの `memberToken` | 失効させる | 正常に終わる。トークンは `revoked` になる。`token.revoked` の操作履歴を残す |
