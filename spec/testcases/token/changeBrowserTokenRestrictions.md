# ブラウザ用トークンの利用元の限定と利用量の制限を変える

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-token-changeBrowserTokenRestrictions-01 | アプリ連携の権限を持つメンバー。利用元を `https://a.example` に限定し、利用量の制限が無い、手動発行のブラウザ用トークン T | 利用元を `https://b.example`、利用量の制限を上限 100・期間 60 秒にして保存する | 変更後のトークンの内容を返す。T の利用元の限定と利用量の制限が変わり、トークンの値は変わらない。`token.restrictions_changed` の操作履歴を、対象の名称を T の名称として残す。`changes` は、`origins` が `["https://a.example"]` から `["https://b.example"]`、`usageLimitMax` が `null` から 100、`usageLimitWindowMs` が `null` から 60000 |
| TC-token-changeBrowserTokenRestrictions-02 | 上の変更を保存した T | 同じ値を示して、利用元 `https://b.example` と `https://a.example` から、それぞれ次の要求で「示されたトークンを確かめる」を行う | `https://b.example` からの要求は `Actor` を返す。`https://a.example` からの要求は `ForbiddenError`（`ORIGIN_NOT_ALLOWED`） |
| TC-token-changeBrowserTokenRestrictions-03 | 利用量の制限を持ち、期間の中の利用が上限に達しているブラウザ用トークン T | 上限を大きくして保存し、同じ値を示して次の要求で「示されたトークンを確かめる」を行う | `Actor` を返す。新しい制限で利用を数える |
| TC-token-changeBrowserTokenRestrictions-04 | アプリ連携の権限を持つメンバー。利用元の限定と利用量の制限を持つブラウザ用トークン T | 利用元を空の列、利用量の制限を無しにして保存する | T は利用元を限定せず、利用量の制限を持たない |
| TC-token-changeBrowserTokenRestrictions-06 | アプリ連携の権限を持つメンバー。ブラウザ用トークン T | `WebOrigin` にならない利用元と、上限 0 の利用量の制限を入力して保存する | `ValidationError`。利用元（`InvalidOrigin`）と利用量の制限（`InvalidUsageLimit`）を、項目ごとの理由で示す。T は変わらない。操作履歴を追記しない |
| TC-token-changeBrowserTokenRestrictions-07 | アプリ連携の権限を持つメンバー。他のメンバーの操作で既に失効したブラウザ用トークン T | 利用元の限定を変えて保存する | 成功として、失効済みであることを示して返す。T は変わらない。操作履歴を追記しない |
| TC-token-changeBrowserTokenRestrictions-08 | アプリ連携の権限を持つメンバー。手動発行したサーバー用トークン | 利用元の限定を指定して保存する | `BusinessRuleError`（`RestrictionNotChangeable`）。トークンは変わらない |
| TC-token-changeBrowserTokenRestrictions-12 | アプリ連携の権限を持つメンバー | 存在しないトークンの ID を指定して保存する | `NotFoundError`。操作履歴を追記しない |
| TC-token-changeBrowserTokenRestrictions-10 | アプリ連携の権限を持たないメンバー。ブラウザ用トークン T | 利用元の限定を変えて保存する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限（`integration:manage`）を示す。T は変わらない |
