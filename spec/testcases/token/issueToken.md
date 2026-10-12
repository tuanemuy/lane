# トークンを手動発行する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-token-issueToken-01 | アプリ連携の権限を持つメンバー。現行スキーマにモデル A がある | 名称、サーバー用、公開取得の許可、ワークスペース全体の範囲、将来の日時の期限を指定して発行する | 発行したトークンの内容とトークンの値を返す。値は `TokenValue.prefixes.server` で始まり、ワークスペースの ID を含む。トークンは `active`、アプリに紐付き、発行の経路は `manual`、発行者は実行したメンバー。`token.issued` の操作履歴を、実行者をそのメンバー、対象の名称をトークンの名称として残す。操作履歴はトークンの値を含まない |
| TC-token-issueToken-02 | アプリ連携の権限を持つメンバー | サーバー用トークンを、期限なしを指定して発行する | 期限なしのトークンを発行する |
| TC-token-issueToken-03 | アプリ連携の権限を持つメンバー。現行スキーマに、キー `article` のモデル A と、キー `contact` のフォーム F がある | 範囲に、モデルのキー `article` とフォームのキー `contact` を指定して発行する | 範囲が、モデル A の ID とフォーム F の ID の、紐付け済みの要素からなるトークンを発行する |
| TC-token-issueToken-04 | アプリ連携の権限を持つメンバー | ブラウザ用、公開取得とフォーム送信の許可、利用元の限定、利用量の制限を指定して発行する | トークンの値を返す。値は `TokenValue.prefixes.browser` で始まる。トークンは、指定した利用元の限定と利用量の制限を持つ。発行の後に「ブラウザ用トークンの値を確かめる」で同じ値を得られる |
| TC-token-issueToken-05 | アプリ連携の権限と、編集権限だけを持つメンバー | サーバー用、コンテンツ管理の許可を指定して発行する | トークンを発行する。トークンは、発行時にメンバーが持っていた権限の組を保持する |
| TC-token-issueToken-06 | アプリ連携の権限を持つメンバー | 名称を空にし、許可を一つも選ばず、有効期限も期限なしも選ばずに発行する | `ValidationError`。名称（`InvalidTokenName`）、許可（`EmptyGrants`）、有効期限（`ExpiryNotSpecified`）を、項目ごとの理由で示す。トークンも操作履歴も残らない |
| TC-token-issueToken-07 | アプリ連携の権限を持つメンバー | ブラウザ用に、プレビュー取得、スキーマ反映、コンテンツ管理のいずれかの許可を指定して発行する | `ValidationError`（`GrantNotAllowedForKind`）。項目と理由を示す。トークンも操作履歴も残らない |
| TC-token-issueToken-09 | アプリ連携の権限を持つメンバー | サーバー用、公開取得の許可を指定し、利用元の限定を指定して発行する | `BusinessRuleError`（`RestrictionNotAllowedForKind`）。トークンも操作履歴も残らない |
| TC-token-issueToken-10 | アプリ連携の権限とスキーマ変更権限を持つメンバー。現行スキーマにモデル A がある | サーバー用、スキーマ反映の許可を指定し、範囲にモデル A のキーを指定して発行する | `BusinessRuleError`（`ScopeNotAllowedForGrant`）。トークンも操作履歴も残らない |
| TC-token-issueToken-11 | アプリ連携の権限を持ち、スキーマ変更権限を持たないメンバー | サーバー用、スキーマ反映の許可を指定して発行する | `BusinessRuleError`（`GrantExceedsIssuerPermissions`）。与えられない許可（スキーマ反映）を示す。トークンも操作履歴も残らない |
| TC-token-issueToken-13 | アプリ連携の権限を持つメンバー。範囲に指定できる対象として返したモデル A（キー `article`）が、発行がスキーマを読む前に、スキーマの反映で削除されている | 範囲にモデルのキー `article` を指定して発行する | `BusinessRuleError`（`ScopeTargetNotFound`）。該当する対象（モデルのキー `article`）を示す。トークンも操作履歴も残らない |
| TC-token-issueToken-14 | アプリ連携の権限を持つメンバー。発行がスキーマを読んだ後、確定の前に、モデル A を削除するスキーマの反映が確定する場合と、反映の確定の UnitOfWork が範囲を集合で持つトークンを読んだ後、反映の確定より先に発行が確定する場合 | 範囲にモデル A のキーを指定して発行する | 前者は `ConflictError`（`SCHEMA_CHANGED`）で、トークンも操作履歴も残らない。後者は発行し、範囲はモデル A の ID の要素を持ったまま残る。反映の後、その要素はどの対象にも当たらず、「トークンを一覧する」はその要素を範囲に示さない |
| TC-token-issueToken-15 | アプリ連携の権限を持つメンバー。スキーマの反映が適用中。現行スキーマに、キー `article` のモデル A がある | 範囲にモデルのキー `article` を指定した発行と、ワークスペース全体の範囲の発行を、それぞれ行う | どちらもトークンを発行する。前者の範囲は、モデル A の ID の紐付け済みの要素 |
| TC-token-issueToken-20 | アプリ連携の権限を持ち、スキーマ変更権限を持たないメンバー | 範囲に、現行スキーマに無いモデルのキーを指定して、次のそれぞれを発行する。名称が空の入力。利用元の限定を持つサーバー用の入力。スキーマ反映の許可を持つサーバー用の入力 | それぞれ `ValidationError`（`InvalidTokenName`）、`BusinessRuleError`（`RestrictionNotAllowedForKind`）、`BusinessRuleError`（`ScopeNotAllowedForGrant`）。`ScopeTargetNotFound` にしない。トークンも操作履歴も残らない |
| TC-token-issueToken-17 | アプリ連携の権限を持たないメンバー（編集権限とメンバー管理の権限を持つ） | トークンを発行する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限（`integration:manage`）を示す。トークンも操作履歴も残らない |
