# 設置を起動する（`bootInstallation`）

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-installation-bootInstallation-01 | 初期化の前の設置（`Installation` と復旧の記録が無い）。設置の設定と資源は起動できる状態。初期構成が、ワークスペース 1 つ、所有者（メールアドレス、名前、パスワード）、ワークスペース全体の範囲のトークン、キーの集合の範囲のトークン、送信接続設定、表示アプリへの通知設定、プレビュー先の URL、コンテンツの保持方針、フォームの運用設定を宣言している | 起動する | 読み込んだ結果として、ワークスペース、所有者、トークンの名称、運用設定の種類を返す。秘密情報の値を含まない。名簿で開設済みの、`active` のワークスペース、作成が作るワークスペースに一つの集約、宣言した名前の所有者のアカウントとメンバー、所有者の `joined` の所属の索引の項目、所有者を発行者とするトークン、送信接続設定、通知の宛先、プレビュー先の URL、紐付け待ちのフォームの運用設定ができる。キーの集合の範囲は紐付け待ちになる。設置は稼働中で、`Installation` の初期構成は `settled`、保存済みのデータの版は実行中の版。操作履歴に `bootstrap.loaded`（実行者は `operator`）を 1 件と、宣言した 2 つのトークンごとに `token.issued`（実行者は `operator`、対象はそのトークン）を 1 件ずつ残す |
| TC-installation-bootInstallation-02 | 上の読み込みが確定している | 所有者が宣言したパスワードでログインし、宣言したトークンの値で要求する | ログインでき、トークンは宣言した許可と範囲で認可される |
| TC-installation-bootInstallation-03 | 初期化の前の設置。初期構成が、ワークスペース 2 つに同じメールアドレスの所有者を、同じ名前とパスワードで宣言している | 起動する | アカウントは 1 つ、メンバーはワークスペースごとに 1 人できる。`bootstrap.loaded` をワークスペースごとに 1 件、合わせて 2 件残す |
| TC-installation-bootInstallation-04 | 初期化の前の設置。初期構成の宣言が無い | 起動する | 宣言なしで始めた結果を返す。設置は稼働中で、`Installation` の初期構成は `settled`。ワークスペースは無い |
| TC-installation-bootInstallation-05 | 初期構成が、ワークスペース 2 つを宣言している。読み込みが、1 つ目のワークスペースの区画の生成を確定した後に止まっている | 利用者の要求の入口から、設置の状態を確かめる | 受け付けない。設置の状態は読み込みの途中。`Installation` の初期構成は `loading`。2 つ目のワークスペースは、名簿で準備中で、利用できない |
| TC-installation-bootInstallation-06 | 同上 | 同じ宣言で起動する | 続きから読み込み、読み込んだ結果を返す。1 つ目のワークスペースと、その所有者、トークン、運用設定を作り直さず、`bootstrap.loaded` は、ワークスペースごとに 1 件のまま。1 つ目のワークスペースのトークンの `token.issued` も、トークンごとに 1 件のまま。2 つ目のワークスペースができ、どちらも名簿で開設済みになる。`Installation` の初期構成は `settled` になり、設置は稼働中になる |
| TC-installation-bootInstallation-07 | 同上 | 所有者のパスワードだけを変えた宣言で起動する | 同じ宣言として、続きから読み込み、読み込んだ結果を返す。確定済みの所有者のアカウントは変わらず、読み込みを始めた宣言のパスワードでログインできる。`Installation` の初期構成は `settled` になる |
| TC-installation-bootInstallation-08 | 同上 | トークンの値だけを変えた宣言で起動する。または、宣言を添えずに起動する。または、読み込みを始めたデータの版と違う版を扱う CMS で、同じ宣言で起動する | どれも起動せず、読み込みの途中の設置であることと、読み込みを始めた宣言で起動し直すか、保存先を空にして始め直すことを返す（「起動と初期構成」の `refuse`）。設置のデータは変わらず、`Installation` の初期構成は `loading` のまま |
| TC-installation-bootInstallation-11 | 初期構成を読み込み終えた稼働中の設置、または初期構成の宣言なしで始めた稼働中の設置（どちらも `Installation` の初期構成は `settled`）。読み込んだ宣言と同じ、または違う内容の初期構成の宣言がある | 起動する | 読み込まずに起動した結果を返す。`declarationIgnored` は真、設置の状態は稼働中。ワークスペース、メンバー、トークン、運用設定を作らず、変えない。`Installation` の初期構成は `settled` のまま |
| TC-installation-bootInstallation-14 | 初期構成の宣言がある。バックアップから復旧した利用再開前の設置（復旧の記録が `restored`。実行中の CMS が扱うデータの版は、復旧したデータの版と同じ、または違う）、または復旧が失敗した設置（`Installation` があり、復旧の記録が `failed`） | 起動する | 初期構成を読み込まず、読み込まずに起動した結果を返す。`declarationIgnored` は真。設置の状態は、利用再開前、または復旧の失敗（すぐ始め直せる）と示す。`Installation` と復旧の記録は変わらない |
| TC-installation-bootInstallation-17 | 結果を受け取れずに中断した復旧の設置（復旧の記録が `restoring`。占有の期限は現在時刻より後） | 起動する | 読み込まずに起動した結果を返す。設置の状態は復旧の進行中で、始め直せる時期は占有の期限（`leaseUntil`）と示す。復旧の記録は変わらない |
| TC-installation-bootInstallation-18 | `Installation` が無い設置への復旧が進行中、または失敗して、復旧の記録だけが `restoring` または `failed` で残る設置。初期構成の宣言がある | 起動する | 設置を初期化せず、初期構成を読み込まない。読み込まずに起動した結果を返し、設置の状態は復旧中。`Installation` は無いまま、復旧の記録は変わらない |
| TC-installation-bootInstallation-19 | 保存先が空で、完了した復旧の記録（`restored`）だけが残る設置。初期構成の宣言がある | 起動する | 初期化の前の設置として、残った復旧の記録を消してから、初期構成を読み込み、読み込んだ結果を返す。復旧の記録は無くなり、設置は稼働中になる |
| TC-installation-bootInstallation-20 | 保存先が空で、完了した復旧の記録（`restored`）だけが残る設置。初期構成の宣言が無い | 起動する | 残った復旧の記録を消し、宣言なしで始めた結果を返す。設置は稼働中になる |
| TC-installation-bootInstallation-21 | 保存先が空で、完了した復旧の記録（`restored`）だけが残る設置。初期構成に、有効期限も期限なしの明示も無いトークンがある | 起動する | 起動せず、`DeclarationIssue` を返す。復旧の記録は消えず、`Installation` は無いまま |
| TC-installation-bootInstallation-22 | 保存済みのデータの版が、実行中の CMS が扱うデータの版と違う設置（更新が必要） | 起動する | 起動せず、更新が必要なことと、保存済みの版・実行中の版を返す。データと `Installation` は変わらない |
| TC-installation-bootInstallation-23 | 更新中の設置（`Installation` の更新の進行が、更新先の版を持つ `updating`）。データの更新が失敗している | 更新先の版の CMS と、更新前の版の CMS のそれぞれで起動する | どちらも起動せず、更新を始めて完了していないことと、更新先の版を返す。データと `Installation` は変わらない |
| TC-installation-bootInstallation-24 | 更新が必要な設置。更新先の版が必要とする資源が不足していて、設置の更新が始まらなかった（更新の進行は `idle`） | CMS を更新前の版の CMS に戻して起動する | 起動する。設置は稼働中で、更新前の版で利用を再開する |
| TC-installation-bootInstallation-25 | 初期化の前の設置。データの保存先の設定、または設置の公開の URL を宣言していない | 起動する | 起動せず、`setting_missing` と、宣言していない設定（`data_store` または `public_url`）の `BootIssue` を返す。何も保存せず、`Installation` は無いまま |
| TC-installation-bootInstallation-27 | データの保存先、またはメディアの保管先の設定の指定が誤っている設置（読み込みを終えた設置、または初期化の前の設置） | 起動し、設定を直した後にもう一度起動する | 一度目は起動せず、`setting_invalid` と、該当する設定と理由の `BootIssue` を返す。理由は認証情報と鍵の値を含まない。設置のデータは変わらない。設定を直した後の起動は、起動する |
| TC-installation-bootInstallation-29 | 初期化の前の設置。データの保存容量が不足している | 起動する | 起動せず、`resource_short` と該当する資源の `BootIssue` を返す。何も保存しない |
| TC-installation-bootInstallation-30 | 初期化の前の設置。設置全体のメール送信を宣言していない。ほかの設定と資源は起動できる状態 | 起動する | 起動する。外部メールを使えない設置として扱う |
| TC-installation-bootInstallation-31 | 初期化の前の設置。初期構成の記述に、所有者を宣言していないワークスペースがある | 起動する | 境界が、形に合わない記述として `ValidationError` で拒否し、該当する宣言を示す。起動せず、ワークスペース、メンバー、トークン、運用設定のいずれも作らない。`Installation` は無いまま |
| TC-installation-bootInstallation-32 | 初期化の前の設置。初期構成が、ワークスペース 2 つに同じメールアドレスの所有者を、違うパスワードで宣言している | 起動する | 起動せず、`path` が二つ目のワークスペースの `owner`、`code` が Identity の `OwnerDeclarationConflict` の `DeclarationIssue` を返す。何も作らない |
| TC-installation-bootInstallation-33 | 初期化の前の設置。初期構成に、有効期限も期限なしの明示も無いトークンがある | 起動する | 起動せず、`path` が該当するトークンの `expiry`、`code` が Token の `ExpiryNotSpecified` の `DeclarationIssue` を返す。何も作らない |
| TC-installation-bootInstallation-34 | 初期化の前の設置。初期構成に、公開取得とフォーム送信以外の許可を持たせたブラウザ用のトークンがある | 起動する | 起動せず、`path` が該当するトークンの `grants`、`code` が Token の `GrantNotAllowedForKind` の `DeclarationIssue` を返す。何も作らない |
| TC-installation-bootInstallation-35 | 初期化の前の設置。初期構成に、スキーマ反映の許可と、モデルのキーの集合の範囲を組にしたサーバー用のトークンがある | 起動する | 起動せず、`path` が該当するトークンの `scope`、`code` が Token の `ScopeNotAllowedForGrant` の `DeclarationIssue` を返す。何も作らない |
| TC-installation-bootInstallation-36 | 初期化の前の設置。初期構成に、受付中と宣言し、有効な通知先または送信接続設定がそろわないフォームがある | 起動する | 起動せず、`path` が該当するフォームの運用設定の位置、`code` が Form の `ReceptionNotReady` の `DeclarationIssue` を返す。何も作らない |
| TC-installation-bootInstallation-37 | 初期化の前の設置。初期構成に、表示アプリへの通知の宛先を 2 件宣言したワークスペースがある。2 件目の URL は、絶対 URL の形でない | 起動する | 起動せず、`path` が該当する宛先（列の位置 1）の `url`、`code` が Webhook の `InvalidWebhookUrl` の `DeclarationIssue` を返す。何も作らない |
| TC-installation-bootInstallation-38 | 初期化の前の設置。初期構成が、同じ名称のワークスペースを 2 つ宣言している | 起動する | 起動せず、`path` が二つ目のワークスペースの `name`、`code` が Workspace の `WorkspaceNameDuplicated` の `DeclarationIssue` を返す。何も作らない |
| TC-installation-bootInstallation-39 | 初期化の前の設置。初期構成に、規則を満たさない名称のワークスペースと、長さの足りないパスワードの所有者がある | 起動する | 起動せず、`path` が所有者の `password` で `code` が `PasswordPolicyViolation` の `DeclarationIssue` と、`path` がワークスペースの `name` で `code` が `InvalidWorkspaceName` の `DeclarationIssue` を、手順を呼ぶ順（Identity、Workspace）に返す。最初の問題で打ち切らず、すべてのドメインの問題を返す |
| TC-installation-bootInstallation-40 | 不正な初期構成で起動しなかった設置。宣言を正しく直している | 起動する | 初期構成を読み込み、読み込んだ結果を返す |
| TC-installation-bootInstallation-41 | 初期化の前の設置。初期構成の宣言がある。一つ目の起動が、設置の区画の最初の段を確定している | 同じ宣言の二つ目の起動が、初期化の前の設置として読んだ状態から、最初の段を確定しようとする | 二つ目の起動は `ConflictError`（理由は問わない）を返し、何も書かない。一つ目の起動が読み込みを終えた後に起動し直すと、読み込まずに起動する。設置には、読み込みの全体が一つだけあり、ワークスペース、所有者、トークン、`bootstrap.loaded`、`token.issued` は重複しない |
| TC-installation-bootInstallation-42 | 初期化の前の設置。初期構成の宣言がある。設置の区画の最初の段の確定が、保存先の障害で失敗する | 起動し、障害の解消の後に、同じ宣言でもう一度起動する | 一度目は `SystemError`。何も残さず、設置は初期化の前のまま（`Installation`、名簿の項目、アカウントは無い）。二度目の起動は、初期構成を読み込み、読み込んだ結果を返す |
| TC-installation-bootInstallation-43 | 初期化の前の設置。初期構成が、一つのワークスペースに、同じキーのフォームの運用設定を 2 件宣言している | 起動する | 起動せず、`path` が二件目のフォームの `form`、`code` が Form の `FormKeyDuplicated` の `DeclarationIssue` を返す。何も作らない |
| TC-installation-bootInstallation-44 | 初期化の前の設置。初期構成に、表示アプリへの通知の宛先を Webhook の `DeliveryPolicy.maxEndpoints` 件より多く宣言したワークスペースがある | 起動する | 起動せず、`path` がそのワークスペースの `settings.webhookEndpoints`、`code` が Webhook の `TooManyEndpoints` の `DeclarationIssue` を返す。何も作らない |
| TC-installation-bootInstallation-45 | 初期化の前の設置。初期構成に、種類の接頭辞で始まらない値を宣言したトークンがある | 起動する | 起動せず、`path` が該当するトークンの `value`、`code` が Token の `InvalidTokenValue` の `DeclarationIssue` を返す。何も作らない |
| TC-installation-bootInstallation-46 | 初期化の前の設置。初期構成が、二つのワークスペースのトークンに、同じ値を宣言している | 起動する | 起動せず、`path` が二件目のトークンの `value`、`code` が Token の `TokenValueDuplicated` の `DeclarationIssue` を返す。何も作らない |
| TC-installation-bootInstallation-47 | 初期化の前の設置。設置の設定と、データの保存容量とメディアの保管先の容量は起動できる状態。通知用の空きを取得できない（`readResource` が `unavailable` を返す）。初期構成の宣言が無い | 起動する | 起動を妨げず、宣言なしで始めた結果を返す。設置は稼働中になる |
| TC-installation-bootInstallation-48 | 初期化の前の設置。初期構成に、コンテンツの保持方針の履歴を保持する日数を、指定できる範囲の中の整数でない日数（1.5）で宣言したワークスペースがある | 起動する | 起動せず、`path` が該当するワークスペースの `settings.contentRetention.revisionDays`、`code` が Content の `ContentRetentionOutOfRange` の `DeclarationIssue` を返す。何も作らない |
