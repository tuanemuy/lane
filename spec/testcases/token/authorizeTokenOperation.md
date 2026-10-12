# トークンの要求を操作ごとに認可する

断りが無い限り、実行者は「示されたトークンを確かめる」が返した `Actor` で、トークンは `active`、発行元のワークスペースは、名簿で開設済みで、区画の `Workspace` が `active`。利用終了の段の行は、入口がトークンを確かめた後に、その段になった要求を指す。操作は、この手続きを呼ぶユースケースを通して行う。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-token-authorizeTokenOperation-01 | 手動発行したサーバー用トークン。公開取得の許可を持ち、範囲はワークスペース全体 | 公開取得の許可とモデル A を必要とする操作を行う | `GrantedTokenAccess` を返す。`actor` は `appToken`。`designation` はトークンの名称、`relatedAccountId` は発行者のアカウントの ID。最終利用日時を変えない |
| TC-token-authorizeTokenOperation-02 | 範囲がモデル A の紐付け済みの要素である、公開取得の許可を持つトークン | モデル A を必要とする操作と、モデル B を必要とする操作を行う | A の操作は `GrantedTokenAccess` を返す。B の操作は `ForbiddenError`（`OUT_OF_SCOPE`）で、対象（モデル B）を示す |
| TC-token-authorizeTokenOperation-03 | コンテンツ管理の許可を持ち、範囲がモデル A の紐付け済みの要素であるトークン。発行者は編集権限を持つ | メディアの操作（対象を範囲で絞らない）を行う | `GrantedTokenAccess` を返す |
| TC-token-authorizeTokenOperation-04 | 公開取得の許可だけを持つサーバー用トークンと、公開取得とフォーム送信の許可を持つブラウザ用トークン | それぞれの実行者で、プレビュー取得の許可を必要とする操作を行う | どちらも `ForbiddenError`（`GRANT_MISSING`）。必要な許可（プレビュー取得）を示す |
| TC-token-authorizeTokenOperation-07 | 範囲に選んだモデル・フォームがすべて削除され、範囲が空のトークン | モデル B を必要とする操作を行う | `ForbiddenError`（`OUT_OF_SCOPE`）。対象を示す。トークンは `active` のまま |
| TC-token-authorizeTokenOperation-08 | 編集権限だけを持つ発行者が発行した、コンテンツ管理の許可を持つアプリに紐付くトークン | 編集権限を必要とするコンテンツ管理の操作と、公開権限を必要とするコンテンツ管理の操作を行う | 編集権限の操作は `GrantedTokenAccess` を返し、権限の組は `content:edit` だけを持つ。公開権限の操作は `ForbiddenError`（`PERMISSION_MISSING`）で、不足する権限（`content:publish`）を示す |
| TC-token-authorizeTokenOperation-10 | スキーマ反映の許可を持つ、手動発行したトークン（範囲はワークスペース全体）。発行の後に、発行者が、スキーマ変更権限を失った、利用を解除された、または、ワークスペースが多要素認証を必須にし、発行者は多要素認証を設定していない | スキーマ反映の操作（対象はワークスペース全体）を行う | どの場合も `GrantedTokenAccess` を返す。権限の組は `schema:change` を含む。`relatedAccountId` は発行者のアカウントの ID |
| TC-token-authorizeTokenOperation-13 | スクリプトの認証で発行した、スキーマ反映の許可を持つトークン（範囲はワークスペース全体）。紐付くメンバーはスキーマ変更権限を持つ | スキーマ反映の操作（対象はワークスペース全体）を行う | `GrantedTokenAccess` を返す。`actor` は `memberToken`。`designation` は発行の経路 `device`、`relatedAccountId` は紐付くメンバーのアカウントの ID |
| TC-token-authorizeTokenOperation-14 | スクリプトの認証で発行した、スキーマ反映の許可を持つトークン。発行の後に、紐付くメンバーがスキーマ変更権限を失った | スキーマ反映の操作を行う | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限（`schema:change`）を示す |
| TC-token-authorizeTokenOperation-15 | メンバーに紐付くトークン。発行の後に、紐付くメンバーが解除された（同じアカウントが再び参加した場合を含む） | 解除前のトークンの実行者で操作を行う | `UnauthenticatedError`（`TOKEN_MEMBER_REMOVED`）。トークンは変わらない |
| TC-token-authorizeTokenOperation-17 | メンバーに紐付くトークン。ワークスペースが多要素認証を必須にし、紐付くメンバーは多要素認証を設定していない | 操作を行う | `ForbiddenError`（`MFA_SETUP_REQUIRED`） |
| TC-token-authorizeTokenOperation-18 | 手動発行したトークンと、スクリプトの認証で発行したトークン（紐付くメンバーは解除済み）。発行元のワークスペースは利用終了の途中で、区画の `Workspace` は `active` のまま | それぞれの実行者で、許可と範囲にある操作を行う | どちらも `NotFoundError`。ワークスペースの情報を示さない。`TOKEN_MEMBER_REMOVED` にしない。操作を確定しない |
| TC-token-authorizeTokenOperation-22 | 手動発行したトークンと、スクリプトの認証で発行したトークン。発行元のワークスペースの利用終了を完了し、名簿に項目が無く、区画の `Workspace` は `closed` | それぞれの実行者で操作を行う | どちらも `NotFoundError` |
| TC-token-authorizeTokenOperation-26 | トークンを確かめた後、操作の認可の前に、そのトークンが失効した | モデル A を必要とする操作と、現行スキーマに無いモデルのキーを指定した操作を、それぞれ行う | どちらも `UnauthenticatedError`（`TOKEN_REVOKED`）。対象が存在しない結果にしない。操作を確定しない |
| TC-token-authorizeTokenOperation-27 | トークンを確かめた後、操作の認可の前に、そのトークンの期限を過ぎた | 操作を行う | `UnauthenticatedError`（`TOKEN_EXPIRED`）。操作を確定しない |
| TC-token-authorizeTokenOperation-28 | 手動発行したトークンの実行者（`appToken`）と、スクリプトの認証で発行したトークンの実行者（`memberToken`） | 実行者を `member` だけに限るユースケース（「トークンを手動発行する」）を、それぞれの実行者で要求する | どちらも `ForbiddenError`（`ACTOR_NOT_ALLOWED`）。この手続きを呼ばない。何も変わらず、操作履歴もイベントも残らない |
| TC-token-authorizeTokenOperation-30 | 公開取得の許可だけを持ち、範囲がモデル A の紐付け済みの要素であるトークン | 現行スキーマに無いモデルのキーを指定して、プレビュー取得の操作を行う | 呼んだユースケースの、対象が存在しない結果を返す。`GRANT_MISSING` にも `OUT_OF_SCOPE` にもしない |
