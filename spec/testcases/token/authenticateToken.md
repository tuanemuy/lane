# 示されたトークンを確かめる

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-token-authenticateToken-01 | ワークスペース W で手動発行した `active` のサーバー用トークン。最終利用日時は未記録 | トークンの値を示して確かめる | `Actor` の `appToken`（そのトークンと発行者）と、発行元のワークスペース W を返す。最終利用日時を要求の時刻で記録する |
| TC-token-authenticateToken-02 | スクリプトの認証で発行した `active` のトークン | トークンの値を示して確かめる | `Actor` の `memberToken`（そのトークンと、紐付くメンバー）と、発行元のワークスペースを返す |
| TC-token-authenticateToken-04 | 無し | トークンを示さない要求、種類の接頭辞で始まらない値、ワークスペースの ID を取り出せず宣言したトークンの索引にも無い値を、それぞれ示して確かめる | どれも `UnauthenticatedError`（`TOKEN_UNKNOWN`） |
| TC-token-authenticateToken-05 | ワークスペース W で手動発行した `active` のトークン T。W と V は名簿で開設済み | W の ID を含み W のどのトークンにも該当しない値と、T の値のワークスペースの ID の部分を V の ID に置き換えた値を、それぞれ示して確かめる | どちらも `UnauthenticatedError`（`TOKEN_UNKNOWN`）。T の最終利用日時を記録しない |
| TC-token-authenticateToken-06 | W で手動発行したトークンと、スクリプトの認証で発行したトークン。W の利用終了を完了し、名簿に W の項目が無い | それぞれの値と、名簿に項目が無いワークスペースの ID を含む値を示して確かめる | どれも `NotFoundError`。ワークスペースの情報を示さない |
| TC-token-authenticateToken-08 | ワークスペース W の初期構成で宣言した `active` のトークン。値はワークスペースの ID を含まない | 宣言した値を示して確かめる | `Actor` の `appToken`（そのトークンと、発行者である初期構成の所有者）と、発行元のワークスペース W を返す。最終利用日時を記録する |
| TC-token-authenticateToken-10 | W の初期構成で宣言した `active` のトークン。宣言したトークンの索引に項目があり、W の区画にトークンが無い（初期構成の読み込みが、索引の登録の後、区画のトークンの作成の前で止まっている） | 宣言した値を示して確かめる | `NotFoundError`（W は名簿で準備中）。読み込みの続きが確定した後は、`Actor` を返す |
| TC-token-authenticateToken-12 | W で手動発行した `active` のトークン A と、失効したトークン B、スクリプトの認証で発行した `active` のトークン C。W は利用終了の途中で、区画の `Workspace` は `active` のまま（利用終了が、名簿の変更の確定の直後に止まっている） | それぞれの値を示して確かめる | どれも `NotFoundError`。ワークスペースの情報を示さない。B に `TOKEN_REVOKED` を返さない。最終利用日時も利用量も記録しない |
| TC-token-authenticateToken-18 | W の初期構成で宣言したトークン。W の利用終了を完了し、名簿の項目と、宣言したトークンの索引の項目が無い | 宣言した値を示して確かめる | `UnauthenticatedError`（`TOKEN_UNKNOWN`）。値からワークスペースの ID を求められない |
| TC-token-authenticateToken-20 | 失効したトークン | トークンの値を示して確かめる | `UnauthenticatedError`（`TOKEN_REVOKED`）。最終利用日時も利用量も記録しない |
| TC-token-authenticateToken-22 | 期限を過ぎたトークン | トークンの値を示して確かめる | `UnauthenticatedError`（`TOKEN_EXPIRED`）。最終利用日時を記録しない |
| TC-token-authenticateToken-23 | 利用元を `https://example.com` に限定したブラウザ用トークン | 利用元 `https://other.example` を示す要求、利用元を示さない要求、利用元 `https://example.com` を示す要求で、それぞれ確かめる | 一つ目と二つ目は `ForbiddenError`（`ORIGIN_NOT_ALLOWED`）で、最終利用日時も利用量も記録しない。三つ目は `Actor` を返す |
| TC-token-authenticateToken-26 | 利用量の制限を持つブラウザ用トークン。期間の中の利用は上限に達していない | トークンの値を示して確かめる | `Actor` を返す。利用を一つ記録する |
| TC-token-authenticateToken-27 | 利用量の制限を持つブラウザ用トークン。期間の中の利用が上限に達している | トークンの値を示して確かめる | `RateLimitedError`（`TOKEN_USAGE_LIMITED`）。`retryAt` を示す。利用も最終利用日時も記録しない |
| TC-token-authenticateToken-28 | 利用量の制限を持つブラウザ用トークンで確かめた要求。続く操作が、操作ごとの認可で拒否される、または反映されない | 確かめた後に、操作が失敗する | 最終利用日時と利用の記録は残る |
| TC-token-authenticateToken-29 | 失効から `OneTimeValueRetention.ms` を過ぎた、手動発行のトークンと、初期構成で宣言したトークン。定期ジョブが削除したかどうかによらない | それぞれの値を示して確かめる | どちらも `UnauthenticatedError`（`TOKEN_UNKNOWN`）。`TOKEN_REVOKED` にしない |
