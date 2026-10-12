# スクリプトの認証の結果を受け取る

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-token-redeemDeviceAuthorization-01 | ワークスペース W のメンバー M が承認した、スキーマ反映の許可の `approved` の依頼。M は在籍し、スキーマ変更権限を持つ | 受け取りの値を指定する | 発行済みと、トークンの値、有効期限、対象のワークスペース W を返す。値は `TokenValue.prefixes.server` で始まり、W の ID を含む。トークンは W の区画にあり、`active`、サーバー用、M に紐付き、発行の経路は `device`、許可は依頼の許可、範囲はワークスペース全体、期限は承認の時刻の `TokenLifetimePolicy.deviceTokenMs` 後。依頼は `redeemed` になり、発行したトークンの ID を記録する。W の操作履歴に `token.issued` を、実行者を M、対象を発行の経路 `device` のトークンとして残す。操作履歴はトークンの値を含まない |
| TC-token-redeemDeviceAuthorization-03 | `pending` の依頼 | 受け取りの値を指定する | 承認待ちを返す。トークンを発行しない。依頼は変わらない |
| TC-token-redeemDeviceAuthorization-04 | メンバーが承認しなかった拒否の依頼と、権限が足りない承認で拒否になった依頼 | それぞれの受け取りの値を指定する | どちらも拒否を返す。`denial` は、前者が `denied_by_account`、後者が `insufficient_permissions` で不足する権限を持つ。トークンを発行しない |
| TC-token-redeemDeviceAuthorization-06 | 期限を過ぎた `pending` の依頼と、承認の後に期限を過ぎた `approved` の依頼（期限の前の受け取りが拒否を返した依頼を含む） | それぞれの受け取りの値を指定する | どちらも、エラーにせず、期限切れを返す。依頼は変わらず、トークンも操作履歴も残らない |
| TC-token-redeemDeviceAuthorization-08 | `redeemed` の依頼。先の受け取りがトークンを受け取った、または、トークンの発行は確定したが応答がコマンドに届かなかった | 同じ受け取りの値をもう一度指定する | 受け取り済みを返す。トークンの値を含まない。二つ目のトークンを発行しない。先に発行したトークンは `active` のまま残る |
| TC-token-redeemDeviceAuthorization-09 | 無し | どの依頼にも該当しない受け取りの値を指定する | `NotFoundError` |
| TC-token-redeemDeviceAuthorization-10 | `approved` の依頼。承認の後に、承認したメンバーが解除された、または、多要素認証を必須にしたワークスペースで多要素認証を解除した | 受け取りの値を二度指定する | 二度とも、エラーにせず、拒否（`refusal` が `member_unavailable`）を返す。依頼は `approved` のまま変わらない。トークンも操作履歴も残らない |
| TC-token-redeemDeviceAuthorization-11 | `approved` の依頼。承認の後に、ワークスペースが利用終了の途中になった | 受け取りの値を指定する | 拒否（`refusal` が `member_unavailable`）を返す。依頼は `approved` のまま変わらない。トークンも操作履歴も残らない |
| TC-token-redeemDeviceAuthorization-13 | スキーマ反映の許可の `approved` の依頼。承認の後に、承認したメンバーがスキーマ変更権限を失った | 受け取りの値を指定する | 拒否（`refusal` が `insufficient_permissions`）と、不足する権限（`schema:change`）を返す。依頼は `approved` のまま変わらない。トークンも操作履歴も残らない |
| TC-token-redeemDeviceAuthorization-14 | スキーマ反映の許可の `approved` の依頼。承認の後に、承認したメンバーがスキーマ変更権限を失い、受け取りが拒否を返した。その後、依頼の期限内に、メンバーがスキーマ変更権限を得た | 同じ受け取りの値を指定する | 発行済みと、トークンの値を返す。依頼は `redeemed` になる |
| TC-token-redeemDeviceAuthorization-16 | `approved` の依頼。二つの受け取りが、どちらも、依頼を `redeemed` にする UnitOfWork の中で `approved` の依頼を読んだ | 二つの受け取りが、それぞれ確定を試みる | 一方だけが、発行済みとトークンの値を返す。もう一方は `ConflictError`（`STALE_VERSION`）で、トークンも操作履歴も残さない。依頼から発行されたトークンは一つ |
| TC-token-redeemDeviceAuthorization-18 | `approved` の依頼。依頼を `redeemed` にする確定の後、トークンの発行の確定の前に、次のいずれかが起きた。承認したメンバーが解除された、権限が縮小された、多要素認証を必須にしたワークスペースで多要素認証を解除した。ワークスペースが利用終了の途中になった。区画の `Workspace` が `active` でなくなり、トークンの発行の確定が `ConflictError`（`WORKSPACE_NOT_ACTIVE`）で拒否された。受け取りが止まった | 受け取りの値を指定する。止まった場合は、同じ受け取りの値をもう一度指定する | どの場合も、受け取り済みを返す。トークンの値を含まない。依頼は `redeemed` のまま残る。トークンも操作履歴も残らない |
| TC-token-redeemDeviceAuthorization-19 | ワークスペース W の `approved` の依頼 D。D の受け取りの後段が、復旧の置き換えの完了の後に確定し、W の区画にトークン（ID は D の ID）が残った。設置の区画の D は、置き換えで `approved` の写しに戻り、期限の前にある | D の受け取りの値を指定する | 受け取り済みを返す。トークンの値を含まない。後段のトークンの `insert` は `ConflictError`（`ALREADY_EXISTS`）で拒否され、W に二つ目のトークンも操作履歴も残らない。D は `redeemed` のまま残る |
