# PublicationLedgerRepository

「上限」は、その節のメソッドの契約が定める、一度に渡せる件数・個数、または `limit` の上限を指す。値は、ポートを定めるドメインのファイルが持つ。

契約は [content.md](../../domains/content.md) の「PublicationLedger」「ポート」と、[common.md](../../domains/common.md) の「PublishedChangeNotice」・「TransactionalRepository」の共通の規則・「WriteScope」・「UnitOfWorkProvider」が定める。

## 記法

- リポジトリは、ワークスペース `W1` の区画に結び付く。メソッドは、ワークスペースの ID を引数に取らない。`W0` は、区画に何も確定していないワークスペース
- UnitOfWork は、`{ kind: "workspace", workspaceId: W1, condition: { kind: "none" } }` を渡した `UnitOfWorkProvider.run` を指す。書き込みは、その `WorkspaceContext` のリポジトリで呼ぶ。読み取りは、UnitOfWork の外で、`PartitionReader.workspace(W1)` のリポジトリで呼ぶ
- `L` は、`PublicationLedger.create()` で作った `PublicationLedger`。「`L` がある」は、`L` の `insert` を確定した状態
- X、Y、Z は操作の ID。「X のドラフト」は、`PublicationLedger.record` で作った、`operationId` が X の `PublicationRecordDraft`。断りが無ければ、`changes` は、区画に無いコンテンツの変更 1 件
- `append` に渡す `modelIds` は、`{ m: モデルの ID }`。区画のコンテンツは、どれもこのモデルに属する
- 「X を追記する」は、新しい UnitOfWork で X のドラフトを `append` に渡し、UnitOfWork を確定することを指す。「X を追記した」は、その確定が済んだ状態
- 「X の参照元」は、`findByOperationId`（X）が返す記録の `changeList.referrers`
- 「X の `sequence`」は、`findByOperationId`（X）が返す記録の `sequence`
- 前提条件の書き込みは、UnitOfWork で呼び、確定済み。前提条件に挙げた順に、前の UnitOfWork の確定の後で次の UnitOfWork を始める。操作の書き込みは、新しい UnitOfWork で呼び、UnitOfWork を確定する
- 「`ConflictError`（理由）になる」は、メソッドの呼び出しか UnitOfWork の確定のどちらかでエラーが現れ、`run` がそのエラーを投げ、その UnitOfWork の書き込みが 1 件も反映されないことを指す
- 「`SystemError`（`INVALID_QUERY`）になる」は、呼び出しがそのエラーを返すことを指す。書き込みでは、`run` がそのエラーを投げ、その UnitOfWork の書き込みが 1 件も反映されない
- A、B はコンテンツで、`ContentRepository.insert` を確定した状態。「R が A を公開内容から参照する」は、公開中の R の公開内容の `references` が、A へのコンテンツの参照を持つ状態

## insert

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-publicationLedgerRepository-01 | `PublicationLedger` が無い | `L` を `insert` する | 成功する。続けて X を追記すると確定し、X の `sequence` は 1 |
| TC-ports-publicationLedgerRepository-02 | `L` がある。X、Y を追記した | `PublicationLedger.create()` で作った値を `insert` する | `ConflictError`（`ALREADY_EXISTS`）になる。`findByOperationId` は、X と Y の記録を返す。続けて Z を追記すると、Z の `sequence` は 3 |
| TC-ports-publicationLedgerRepository-04 | `PublicationLedger` が無い | UnitOfWork の中で `L` を `insert` した後、`fn` が例外を投げる | `run` はその例外を投げる。続けて X のドラフトを `append` に渡した UnitOfWork は `ConflictError`（`STALE_VERSION`）になる |

## append

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-publicationLedgerRepository-05 | `L` がある | X を追記する | `append` は値を返さない。`findByOperationId`（X）が、`sequence` が 1 で、`operationId`、`operation`、`actor`、`schemaVersion` がドラフトと等しく、`changeList.changes` がドラフトの `changes` と等しく、`changeList.referrers` が空の記録を返す |
| TC-ports-publicationLedgerRepository-06 | `L` がある | X、Y、Z を、この順に追記する | X、Y、Z の `sequence` は 1、2、3 |
| TC-ports-publicationLedgerRepository-07 | `L` がある | `operation` が `published` で、`changes` が、`change` が `published` と `updated` の 2 件（`uniqueValues` は、空のものと、`before` が `null` のもの）の X のドラフトと、`operation` が `migrated` で、`changes` が、`change` が `updated` と `unpublished`（`uniqueValues` の `after` が `null`）の 2 件の Y のドラフトを追記する | X と Y の記録の `operation` は、それぞれ `published` と `migrated`。`changeList.changes` は、要素と順が、それぞれのドラフトの `changes` と等しい |
| TC-ports-publicationLedgerRepository-09 | `L` がある。区画に、公開中のコンテンツが 1,000 件あり、それぞれを公開内容から参照する公開中のコンテンツが、別に 1,000 件ある | 前者の 1,000 件の変更を `changes` に持つ X のドラフトを追記する | 成功する。`findByOperationId`（X）の `changeList.changes` は 1,000 件、X の参照元は 1,000 件 |
| TC-ports-publicationLedgerRepository-10 | `L` と A がある。X を追記した | 一つの UnitOfWork で、内容を変えた A の `ContentRepository.save` と、`operation` と `changes` が異なる、`operationId` が X のドラフトの `append` を呼ぶ。続けて Y を追記する | 前者は `ConflictError`（`ALREADY_EXISTS`）になる。`findByOperationId`（X）は先に追記した記録を、`ContentRepository.findById`（A）は前提条件の A を返す。Y の `sequence` は 2 |
| TC-ports-publicationLedgerRepository-13 | `PublicationLedger` が無い。A がある | 一つの UnitOfWork で、内容を変えた A の `ContentRepository.save` と、X のドラフトの `append` を呼ぶ | `ConflictError`（`STALE_VERSION`）になる。`findByOperationId`（X）は `null`、`ContentRepository.findById`（A）は前提条件の A を返す |
| TC-ports-publicationLedgerRepository-15 | `L` がある。X を追記した | UnitOfWork の中で Y のドラフトを `append` に渡した後、`fn` が例外を投げる。続けて Z を追記する | `run` はその例外を投げる。`findByOperationId`（Y）は `null` を返す。Z の `sequence` は 2 |
| TC-ports-publicationLedgerRepository-16 | `L` と A がある | 一つの UnitOfWork で、X のドラフトの `append` と、A と同じ ID のコンテンツの `ContentRepository.insert` を呼ぶ | `ConflictError`（`ALREADY_EXISTS`）になる。`findByOperationId`（X）は `null` を返す。続けて Y を追記すると、Y の `sequence` は 1 |
| TC-ports-publicationLedgerRepository-18 | `L` がある。`fn` が例外を投げて確定しなかった UnitOfWork で X のドラフトを `append` に渡した | X を追記する | 成功する。X の `sequence` は 1 |

## appendStaged

`appendStaged` の契約と、`append` と共有する「通し番号」「参照元」の例。`changes` の並び順は、[content.md](../../domains/content.md) の「PublicationLedger」の例。

- `J1`、`J2` は `SchemaApplyJobId`。`H(X)` は、`operationId` が X の `MigrationRecordHeader`。「記録」は、`findByOperationId`（X）が返す記録
- コンテンツ C の「C の結果」「`publication` が `updated` の C の結果」「C の削除の結果」は、`ContentStagingArea.putContents` で `J` の準備領域に書いた結果で、記法は [contentStagingArea.md](contentStagingArea.md) と同じ。断りが無い結果は、`modelIds` の `m` のモデルに属する。`M2` は、`modelIds` の `n` のモデル
- 「`J` の確定の段を呼ぶ」は、一つの UnitOfWork で、`appendStaged`（`J`、`H(X)`、`modelIds`）と `ContentStagingArea.promote`（`J`）を呼ぶことを指す。断りが無い確定の段は、`J1` と `H(X)` のもの。「`H(Y)` で確定の段を呼ぶ」は、`H(X)` の代わりに `H(Y)` を渡す。この節の `modelIds` は `{ m: モデルの ID, n: M2 }`
- 断りが無い前提条件は、`L` があり、公開の変更の記録が無い。断りが無いコンテンツは公開中で、公開内容からの参照は任意の参照。「区画に…がある」は、区画のコンテンツが、挙げたものだけであることを指す

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-publicationLedgerRepository-19 | 区画に A がある。`J1` の準備領域に、`publication` が `updated` の A の結果がある | 確定の段を呼ぶ | `recorded` が真の結果を返す。確定の後、`findByOperationId`（X）が、`sequence` が 1、`operation` が `migrated`、`operationId` が X、`actor` と `schemaVersion` が `H(X)` の値の記録を返す。記録の `changeList.changes` は、A の報告から `ContentMigration.noticeOf` が作る値の 1 件。`changeList.referrers` は空 |
| TC-ports-publicationLedgerRepository-20 | 区画に、T と、公開内容が T を参照する A と、公開内容が T を参照しない B がある。`J1` の準備領域に、`publication` が `updated` の T の結果、公開内容が T を参照しない A の結果、公開内容が T を参照し、`M2` に属する B の結果がある | 確定の段を呼ぶ | 記録の `changeList.changes` は T の 1 件。`changeList.referrers` は、B のコンテンツID と `n` の 1 件で、A を含まない |
| TC-ports-publicationLedgerRepository-21 | 区画に、T と、公開内容が T を参照する A がある。`J1` の準備領域に、`publication` が `updated` の T の結果と、`publication` が `updated` の A の結果がある | 確定の段を呼ぶ | 記録の `changeList.changes` は、コンテンツID の昇順に、A と T の 2 件。`changeList.referrers` は空 |
| TC-ports-publicationLedgerRepository-22 | 区画に、公開中の B と、公開内容が B を任意の参照で参照する A がある。`J1` の準備領域に、公開内容が B を任意の参照で参照する A の結果と、`publication` が `unpublished` の B の削除の結果（`result` は `null`）がある | 確定の段を呼ぶ | `recorded` が真の結果を返す。`ContentRepository.findById`（B）は `null`。記録の `changeList.changes` は、B の `unpublished` の 1 件。`changeList.referrers` は、A の 1 件 |
| TC-ports-publicationLedgerRepository-23 | 区画に A がある。`J1` の準備領域に A の結果がある | 確定の段を呼ぶ | `recorded` が偽の結果を返す。`findByOperationId`（X）は `null` を返す |
| TC-ports-publicationLedgerRepository-24 | 区画に A がある。`J1` の準備領域に、`publication` が `updated` の A の結果がある | UnitOfWork の中で確定の段を呼んだ後、`fn` が例外を投げる | `run` はその例外を投げる。`findByOperationId`（X）は `null`、`ContentRepository.findById`（A）は前提条件の A を返す |
| TC-ports-publicationLedgerRepository-26 | 区画に A がある。Y、Z を、この順に追記した。`J1` の準備領域に、`publication` が `updated` の A の結果がある | 確定の段を呼ぶ | 記録の `sequence` は 3。Y と Z の `sequence` は、1 と 2 のまま |
| TC-ports-publicationLedgerRepository-27 | 区画に A がある。`J1` の準備領域に、`publication` が `updated` の A の結果がある | 確定の段を確定した後、Y を追記する | Y の `sequence` は 2 |
| TC-ports-publicationLedgerRepository-28 | 区画に A がある。`J1` の準備領域に、`publication` が `updated` の A の結果がある。確定の段を呼んだ UnitOfWork が、`fn` の例外で確定しなかった | Y を追記する | Y の `sequence` は 1 |
| TC-ports-publicationLedgerRepository-29 | 区画に A がある。Y を追記した。`J1` の準備領域に、`publication` が `updated` の A の結果がある。確定の段を呼んだ UnitOfWork が、`fn` の例外で確定しなかった | 新しい UnitOfWork で確定の段を呼ぶ | 成功する。記録の `sequence` は 2 |
| TC-ports-publicationLedgerRepository-30 | 区画に、下書きの `title` が `old` の A がある。X を追記した。`J1` の準備領域に、`title` が `new` で、`publication` が `updated` の A の結果がある | 確定の段を呼ぶ。続けて `H(Y)` で確定の段を呼ぶ | 一度目は `ConflictError`（`ALREADY_EXISTS`）になる。その後、`ContentRepository.findById`（A）は `old` の A を、`findByOperationId`（X）は先に追記した記録を返し、`ContentStagingArea.cursor`（`J1`）の `contents` は 1。二度目は成功し、Y の `sequence` は 2 |
| TC-ports-publicationLedgerRepository-32 | 区画に A がある。X を追記した。`J1` の準備領域に A の結果がある（`publication` は `null`） | 確定の段を呼ぶ | 成功し、`recorded` が偽の結果を返す。`findByOperationId`（X）は、先に追記した記録を返す |
| TC-ports-publicationLedgerRepository-33 | `PublicationLedger` が無い。区画に、下書きの `title` が `old` の A がある。`J1` の準備領域に、`title` が `new` で、`publication` が `updated` の A の結果がある | 確定の段を呼ぶ | `ConflictError`（`STALE_VERSION`）になる。`ContentRepository.findById`（A）は `old` の A を返す。`findByOperationId`（X）は `null`。`ContentStagingArea.cursor`（`J1`）の `contents` は 1 |
| TC-ports-publicationLedgerRepository-34 | `PublicationLedger` が無い。区画に A がある。`J1` の準備領域に A の結果がある（`publication` は `null`） | 確定の段を呼ぶ | 成功し、`recorded` が偽の結果を返す。`ContentRepository.findById`（A）は `result` と等しい A を返す |
| TC-ports-publicationLedgerRepository-35 | 区画に A がある。`J1` の準備領域に、`publication` が `updated` の A の結果がある | 一方の UnitOfWork が確定の段を、他方の UnitOfWork が Y のドラフトの `append` を、並行に呼ぶ | どちらも確定する。X と Y の `sequence` は、1 と 2 を一つずつ |

## findByOperationId

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-publicationLedgerRepository-36 | `PublicationLedger` が無い | `findByOperationId`（X）を呼ぶ | `null` を返す |
| TC-ports-publicationLedgerRepository-37 | `L` がある。X を追記した | `findByOperationId`（Y）を呼ぶ | `null` を返す |
| TC-ports-publicationLedgerRepository-38 | `L` がある。X、Y を追記した | `findByOperationId`（Y）を呼ぶ | Y の記録を返す。`sequence` は 2 |
| TC-ports-publicationLedgerRepository-39 | `W1` に `L` があり、X を追記した | `PartitionReader.workspace(W0)` のリポジトリで、`findByOperationId`（X）を呼ぶ | `null` を返す |

## 参照元

前提条件のコンテンツは、どれも公開中。X のドラフトは、断りが無ければ、A の `updated` の変更 1 件を `changes` に持つ。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-publicationLedgerRepository-40 | `L` と A がある。R が A を公開内容から参照する | X を追記する | X の参照元は、R のコンテンツID とモデルのキー `m` の 1 件 |
| TC-ports-publicationLedgerRepository-41 | `L` と A がある。A はモデル `M1`、R はモデル `M2` に属する。R が A を公開内容から参照する | `modelIds` に `{ m: M1, n: M2 }` を渡して、X を追記する | X の参照元は、R のコンテンツID と `n` の 1 件 |
| TC-ports-publicationLedgerRepository-42 | `L` と A がある。A を参照する内容は無い | X を追記する | X の参照元は空 |
| TC-ports-publicationLedgerRepository-43 | `L` と A がある。R1 は必須の参照、R2 は任意の参照、R3 は本文のリンクで、A を公開内容から参照する | X を追記する | X の参照元は、R1、R2、R3 の 3 件 |
| TC-ports-publicationLedgerRepository-44 | `L` と A がある。R1 と R2 が A を公開内容から参照する。R2 のコンテンツID は R1 より小さい | X を追記する | X の参照元は、R2、R1 の順 |
| TC-ports-publicationLedgerRepository-45 | `L` と A がある。D は、公開内容では A を参照せず、下書きで A を参照する。E の履歴の版が A を参照する | X を追記する | X の参照元は空 |
| TC-ports-publicationLedgerRepository-46 | `L` と A、B がある。R が、A と B の両方を公開内容から参照する | A と B の変更を `changes` に持つ X のドラフトを追記する | X の参照元は、R の 1 件 |
| TC-ports-publicationLedgerRepository-47 | `L` と A、B がある。A と B は、公開内容で互いを参照し、A は A 自身も参照する。R が A を公開内容から参照する | A と B の変更を `changes` に持つ X のドラフトを追記する | X の参照元は、R の 1 件。A と B を含まない |
| TC-ports-publicationLedgerRepository-49 | `L` と A がある。R1 は A を参照しない。R2 が A を公開内容から参照する | 一つの UnitOfWork で、公開内容が A を参照するように変えた R1 と、公開内容が A を参照しないように変えた R2 の `ContentRepository.save` と、X のドラフトの `append` を呼ぶ | X の参照元は、R1 の 1 件 |
| TC-ports-publicationLedgerRepository-51 | `L` と A がある。R が A を公開内容から参照する。X を追記した | 公開内容が A を参照しないように変えた R の `ContentRepository.save` を確定してから、`findByOperationId`（X）を呼ぶ | X の参照元は、R の 1 件のまま。追記した記録は変わらない |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-publicationLedgerRepository-52 | `PublicationLedger` が無い | `deleteAll`（10）を呼ぶ | 0 を返す |
| TC-ports-publicationLedgerRepository-53 | `L` がある。記録は無い | `deleteAll`（10）を呼ぶ | 1 を返す。続けて X のドラフトを `append` に渡した UnitOfWork は `ConflictError`（`STALE_VERSION`）になる |
| TC-ports-publicationLedgerRepository-54 | `L` がある。X、Y を追記した | `deleteAll`（10）を呼ぶ | 3 を返す。`findByOperationId` は、X と Y のどちらにも `null` を返す。続けて Z のドラフトを `append` に渡した UnitOfWork は `ConflictError`（`STALE_VERSION`）になる |
| TC-ports-publicationLedgerRepository-55 | `L` がある。X、Y、Z を追記した | `deleteAll`（2）を、呼び出しごとに別の UnitOfWork で、返る件数が 2 より少なくなるまで繰り返す | 2、2、0 の順に返す。一度目の後、`findByOperationId` は、X と Y に `null`、Z に Z の記録を返し、`PublicationLedger.create()` で作った値を `insert` した UnitOfWork は `ConflictError`（`ALREADY_EXISTS`）になる |
| TC-ports-publicationLedgerRepository-56 | `L` がある。X、Y を追記した | `deleteAll`（2）を、呼び出しごとに別の UnitOfWork で、返る件数が 2 より少なくなるまで繰り返す | 2、1 の順に返す。一度目の後、`findByOperationId` は、X と Y のどちらにも `null` を返し、`PublicationLedger.create()` で作った値を `insert` した UnitOfWork は `ConflictError`（`ALREADY_EXISTS`）になる |
| TC-ports-publicationLedgerRepository-59 | `L` がある。X、Y を追記した後、`deleteAll`（10）で削除した | `L` を `insert` し、X を追記する | どちらも成功する。X の `sequence` は 1 |
| TC-ports-publicationLedgerRepository-60 | `L` がある。X を追記した | UnitOfWork の中で `deleteAll`（10）を呼んだ後、`fn` が例外を投げる | `findByOperationId`（X）は X の記録を返す。続けて Y を追記すると、Y の `sequence` は 2 |
| TC-ports-publicationLedgerRepository-61 | `L` がある。X を追記した | `deleteAll`（0）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる。`findByOperationId`（X）は X の記録を返す |
| TC-ports-publicationLedgerRepository-62 | `L` がある。X を追記した | `deleteAll`（上限より 1 大きい `limit`）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる。`findByOperationId`（X）は X の記録を返す |

## 並行性

二つの UnitOfWork を並行に実行する。どちらの `run` も、相手の結果を待たずに呼ぶ。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-publicationLedgerRepository-63 | `L` と A、B がある | 一方の UnitOfWork が A の `ContentRepository.save` と X のドラフトの `append` を、他方の UnitOfWork が B の `ContentRepository.save` と Y のドラフトの `append` を呼ぶ | どちらも確定する。`ContentRepository.findById` は、A と B のどちらも保存した内容を返す。X と Y の `sequence` は、1 と 2 を一つずつ。続けて Z を追記すると、Z の `sequence` は 3 |
| TC-ports-publicationLedgerRepository-65 | `L` がある | 二つの UnitOfWork が、それぞれ、`changes` が異なる、`operationId` が X のドラフトを `append` に渡す | 一方だけが確定する。他方は `ConflictError`（`ALREADY_EXISTS`）になる。`findByOperationId`（X）は、確定した側のドラフトと等しい、`sequence` が 1 の記録を返す。続けて Y を追記すると、Y の `sequence` は 2 |
| TC-ports-publicationLedgerRepository-66 | `L` と A がある。A の `expectedVersion` を読んでいる | 一方の UnitOfWork が、その `expectedVersion` での A の `ContentRepository.save` と X のドラフトの `append` を、他方の UnitOfWork が、同じ `expectedVersion` での A の `ContentRepository.save` と Y のドラフトの `append` を呼ぶ | 一方だけが確定し、他方は `ConflictError`（`STALE_VERSION`）になる。`findByOperationId` は、確定した側に `sequence` が 1 の記録を、他方に `null` を返す。続けて Z を追記すると、Z の `sequence` は 2 |
| TC-ports-publicationLedgerRepository-67 | `PublicationLedger` が無い | 二つの UnitOfWork が、それぞれ `PublicationLedger.create()` で作った値を `insert` する | 一方だけが確定する。他方は `ConflictError`（`ALREADY_EXISTS`）になる。続けて X を追記すると、X の `sequence` は 1 |

## 可視性と UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-publicationLedgerRepository-68 | `L` と A がある。R が A を公開内容から参照する | A の変更を持つ X を追記して確定した直後に、`findByOperationId`（X）を呼ぶ | `sequence` が 1 で、参照元に R を持つ X の記録を返す |
| TC-ports-publicationLedgerRepository-69 | `L` と A がある | 一つの UnitOfWork で、内容を変えた A の `ContentRepository.save` と、X のドラフトの `append` を呼び、確定する | `ContentRepository.findById`（A）は保存した内容を返す。X の `sequence` は 1 |
| TC-ports-publicationLedgerRepository-70 | `PublicationLedger` が無い | 一つの UnitOfWork で、`L` の `insert` と、`ContentRetentionPolicyRepository.insert` を呼び、確定する | `ContentRetentionPolicyRepository.find()` は保存した値を返す。続けて X を追記すると、X の `sequence` は 1 |
