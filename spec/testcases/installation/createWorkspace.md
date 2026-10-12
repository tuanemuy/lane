# ワークスペースを作成し、初期所有者の招待を発行する（`createWorkspace`）

作成と並行した利用終了の結果は、`spec/domains/workspace.md` の「作成と利用終了の並行」が定める。「ID」は、作成の入力の `WorkspaceId`。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-installation-createWorkspace-01 | `operator`。設置全体のメール送信が動く稼働中の設置。既存のワークスペースが 1 つある | 新しい ID と、名称と、初期所有者のメールアドレスを指定して作成する | 作成したワークスペース（その ID と名称）、招待の宛先、招待の有効期限、メールを送信できたことを返す。名簿で開設済みの、所有者の指定が `vacant` の `active` のワークスペース、作成が作るワークスペースに一つの集約、`pending` の初期所有者の招待ができる。指定した宛先に、設置の公開の URL と招待の値から作った招待の URL を本文に持つメールを送る。操作履歴に `invitation.created`（実行者は `operator`）を 1 件残す。既存のワークスペースのスキーマ、コンテンツ、メンバー、設定は変わらない |
| TC-installation-createWorkspace-02 | `operator`。設置全体のメール送信を宣言していない設置 | 作成する | メールを送らず、一度だけ使える招待の値を結果に返す。URL を返さない。ワークスペースと招待ができ、操作履歴に `invitation.created` を残す |
| TC-installation-createWorkspace-03 | `operator`。設置全体のメール送信が失敗する設置 | 作成する | 送信の失敗と理由を結果に示す。ワークスペース、ワークスペースに一つの集約、招待、操作履歴は残る |
| TC-installation-createWorkspace-04 | `operator`。作成が、名簿への登録の後、区画の生成の前に止まり、名簿で準備中のまま残るワークスペースがある | 同じ ID、名称、初期所有者のメールアドレスで作成する | 続きから作成する。作成したワークスペース（同じ ID）、招待の宛先と期限、メールを送信できたことを返す。ワークスペースは名簿で開設済みになる |
| TC-installation-createWorkspace-05 | `operator`。作成が、区画の生成の後、名簿を開設済みにする前に止まったワークスペースがある。区画に `Workspace` と、作成が作った集約がある。招待は無い | 同じ ID、名称、初期所有者のメールアドレスで作成する | ワークスペースに一つの集約は作り直さない。ワークスペースは名簿で開設済みになり、指定した初期所有者へ招待を発行して、メールを送る。招待の宛先と期限を返す。操作履歴に `invitation.created` を 1 件残す |
| TC-installation-createWorkspace-06 | `operator`。作成が、名簿を開設済みにした後、初期所有者の招待の発行の前に止まったワークスペースがある（名簿で開設済み、所有者の指定が `vacant`、初期所有者の招待は無い） | 同じ ID、名称、初期所有者のメールアドレスで作成する | 区画の集約を作り直さずに、初期所有者へ招待を発行してメールを送る。招待の宛先と期限を返す。操作履歴に `invitation.created` を 1 件残す |
| TC-installation-createWorkspace-07 | `operator`。作成が途中で止まり、名簿で準備中のまま残るワークスペースがある | 定期ジョブが一巡した後に、状態を確認する | ワークスペースは準備中のまま。CMS は作成を進めない。状態確認は、作成の途中の項目に、ID と名称で示す |
| TC-installation-createWorkspace-09 | `operator` | 規則を満たさない名称を指定して作成する | `ValidationError`（理由 `InvalidWorkspaceName`）。ワークスペースを作成しない |
| TC-installation-createWorkspace-10 | `operator`。同じ名称の、名簿で開設済みのワークスペースと、名簿で終了中のワークスペースがある | 別の ID と、その名称を指定して作成する | 作成する。名称の重複で拒否しない。既存の二つのワークスペースは変わらない |
| TC-installation-createWorkspace-11 | `operator`。作成が途中で止まり、名簿で準備中のまま残るワークスペースがある | 同じ ID と、違う名称、または前の実行と違う初期所有者のメールアドレスを指定して作成する | `ConflictError`（理由 `ALREADY_EXISTS`）。何も書かず、招待を発行しない |
| TC-installation-createWorkspace-13 | `operator`。作成を終えたワークスペースがある。初期所有者の招待は期限内で、受諾されていない | 同じ ID、名称、初期所有者のメールアドレスで作成する | 作成済みであることと、その ID と名称を返す。何も書かない。招待を発行せず、メールを送らず、前の招待は有効なまま残る。操作履歴を残さない |
| TC-installation-createWorkspace-14 | `operator`。利用終了が完了したワークスペース（名簿に項目が無い）がある | その ID を指定して作成する | `ConflictError`（理由 `WORKSPACE_CLOSING`）。何も書かず、招待を発行しない |
| TC-installation-createWorkspace-15 | `operator`。名簿で準備中のワークスペースの作成の続きが、名簿の項目を読んだ後、区画の段の前に、そのワークスペースの利用終了が、開始を確定している（区画の `Workspace` は `closing`） | 作成の区画の段を行う | 何も書かずに `ConflictError`（理由 `WORKSPACE_CLOSING`）。招待を発行せず、メールを送らない。区画の `Workspace` は `closing` のまま |
| TC-installation-createWorkspace-16 | `operator`。名簿で準備中で、区画に `Workspace` が無いワークスペースの作成の続きが、区画の `Workspace` が無いことを読んだ後、区画の段の確定の前に、そのワークスペースの利用終了の開始が確定する | 作成の区画の段を確定させる | `ConflictError`（理由 `ALREADY_EXISTS`）。区画の `Workspace` は `closing` のまま。区画に、作成が作る集約は残らない。招待を発行せず、メールを送らない |
| TC-installation-createWorkspace-17 | `operator`。作成の、区画の段の確定の後、名簿を開設済みにする段の前に、そのワークスペースの利用終了が、名簿の項目を終了中にした | 作成の残りの段を行う | `ConflictError`（理由 `WORKSPACE_CLOSING`）。招待を発行せず、メールを送らず、招待の値を返さない。名簿の項目は終了中のまま。その後の「ワークスペースの利用終了を完了させる」が、作成が作った集約を消して、名簿から外す |
| TC-installation-createWorkspace-20 | `operator`。作成の、名簿を開設済みにする段の確定の後、招待の発行の対象の確認の前に、そのワークスペースの利用終了が、最後の段まで完了した（名簿に項目が無い。区画の `Workspace` は `closed`） | 作成の残りの段を行う | `NotFoundError`。招待を発行せず、メールを送らず、何も書き足さない |
