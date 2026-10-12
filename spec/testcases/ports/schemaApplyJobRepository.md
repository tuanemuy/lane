# SchemaApplyJobRepository

契約は [schema.md](../../domains/schema.md) の「ポート」と、[common.md](../../domains/common.md) の「TransactionalRepository」「Pagination と PaginationResult」「WriteScope」「UnitOfWorkProvider」が定める。

## 記法

- W、W1、W2 は、互いに違うワークスペース、J、J1、J2 は `SchemaApplyJobId`。リポジトリは、ワークスペースの区画ごとにあり、メソッドはワークスペースの ID を取らない。断りの無い呼び出しは、W の区画のリポジトリに対して行う。「W1 の `findById`」のように書いたワークスペースは、呼ぶリポジトリが結び付く区画を指す
- ジョブ（`SchemaApplyJob`）は、`Preparation` と `SchemaApplyJob` の振る舞い（`accept`、`advance`、`complete`、`failIfSchemaChanged`、`fail`）と `WorkspaceSchema.commit` が返す値で作り、`insert` と `save` で保存する。ジョブの ID は `preparation.jobId`。ジョブは、ワークスペースの ID を持たない。「反映のジョブ」は `mode` が `commit`、「事前確認のジョブ」は `mode` が `dryRun` のジョブ
- 前提条件の書き込みは、そのワークスペースの UnitOfWork（`UnitOfWorkProvider.run` に `{ kind: "workspace", workspaceId, condition }` を渡した UnitOfWork。[unitOfWorkProvider.md](unitOfWorkProvider.md)）の `WorkspaceContext` のリポジトリで呼び、確定済み。操作の書き込みは、新しい UnitOfWork で呼ぶ。読み取りは UnitOfWork の外で、`PartitionReader.workspace` のリポジトリで呼ぶ。UnitOfWork の条件は `none`
- 「`ConflictError`（理由）になる」は、メソッドの呼び出しか確定のどちらかでエラーが現れ、`run` がそのエラーを投げ、その UnitOfWork の書き込みが一つも反映されないことを指す
- v は、直前の `findById` が返した `expectedVersion`
- 「全状態のジョブ」は、反映の `preparing`・`prepared`・`succeeded`・`failed` と、事前確認の `preparing`・`checked`・`failed` の 7 件
- `progressedAt` は `preparation.progressedAt` を指し、古い順に p1 < p2 < p3。`finishedAt` は古い順に f1 < f2 < f3
- 引数を書かない `findInProgress` は `findInProgress({ progressedBefore: null, after: null }, 10)`。引数を書かない `findApplyResultsPage` の `pagination` は `{ page: 1, limit: 10 }`
- 「上限」は、`findInProgress` では `SchemaLimits.inProgressLimit`、`findApplyResultsPage` では 1 ページの件数 10、`deleteAll` では `BoundedDeletion.maxLimit`

## insert と findById

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-schemaApplyJobRepository-01 | J のジョブが無い | `findById(J)` を呼ぶ | `null` を返す |
| TC-ports-schemaApplyJobRepository-02 | J のジョブが無い | `accept` の結果の反映のジョブを `insert` し、確定の後に `findById(J)` を呼ぶ | `insert` した値と全フィールドが等しい `entity`（`preparation` の `mode`、`plan`、`phase`、`progressedAt`、`requestDigest`、`requestedAt`、`actor`、`actorNames` を含む。`plan` の `migrations`、`units`、`lossyOperations` の順序を保つ）と、`expectedVersion` を返す |
| TC-ports-schemaApplyJobRepository-03 | J のジョブが無い | `plan.source` が `direct` の反映のジョブと、事前確認のジョブを、別の ID で `insert` し、それぞれ `findById` で読む | それぞれ、保存した `mode` と `plan`（`operations` を含む）を保つ |
| TC-ports-schemaApplyJobRepository-04 | ジョブが無い | `actor` が `appToken` のジョブと `memberToken` のジョブを `insert` し、それぞれ `findById` で読む | それぞれ、保存した `actor` と `actorNames` を保つ |
| TC-ports-schemaApplyJobRepository-05 | J のジョブがある | 同じ ID J の、内容が異なるジョブを `insert` する | `ConflictError`（`ALREADY_EXISTS`）になる。`findById(J)` は、先に `insert` した値を返す |
| TC-ports-schemaApplyJobRepository-06 | J の反映のジョブがある | ID J の事前確認のジョブを `insert` する | `ConflictError`（`ALREADY_EXISTS`）になる。`findById(J)` は、反映のジョブを返す |
| TC-ports-schemaApplyJobRepository-07 | J のジョブは終端（`succeeded`） | ID J のジョブを `insert` する | `ConflictError`（`ALREADY_EXISTS`）になる。`findById(J)` は、`succeeded` のジョブを返す |
| TC-ports-schemaApplyJobRepository-08 | J のジョブが W1 にある | 同じ ID J の、W2 のジョブを、W2 のリポジトリで `insert` する | 確定する。W1 の `findById(J)` は W1 のジョブを、W2 の `findById(J)` は W2 のジョブを返す。ID の一意性は、区画の中に限る |
| TC-ports-schemaApplyJobRepository-09 | J のジョブが無い | 並行する二つの UnitOfWork が、一方は ID J の反映のジョブを、もう一方は ID J の事前確認のジョブを `insert` する | 一方だけが確定し、もう一方は `ConflictError`（`ALREADY_EXISTS`）になる。`findById(J)` は、確定した側の値を返す |
| TC-ports-schemaApplyJobRepository-10 | J のジョブが W1 にだけある | W1 と W2 の `findById(J)` を呼ぶ | W1 はそのジョブを、W2 は `null` を返す |
| TC-ports-schemaApplyJobRepository-11 | J のジョブを `insert` し、`deleteAll` で消去済み | ID J のジョブを `insert` する | 確定する。`findById(J)` は新しいジョブを返す |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-schemaApplyJobRepository-12 | `preparing` のジョブがある | `Preparation.fixTarget` の結果を渡した `Preparation.recordConverted` の結果を、`advance` に渡し、その結果を v とともに `save` して、確定の後に `findById` を呼ぶ | `preparation` の `phase`（`phase.target` の `lastContentId`、`lastRevisionId`、`total` と、`phase.processed` を含む）と `progressedAt` が保存した値の `entity` を返す |
| TC-ports-schemaApplyJobRepository-13 | 準備の `phase` が `tallying` のジョブがある | `Preparation.recordTallied` の結果を渡した `advance` の結果を `save` し、`findById` を呼ぶ | `phase.total`、`phase.tally`（`Sampled` の `total` と `items` を含む）、`phase.cursor` が保存した値の `entity` を返す |
| TC-ports-schemaApplyJobRepository-14 | 準備の `phase` が `tallying` の反映のジョブがある | `complete` の結果（`prepared`）、続けて `SchemaApplyJob.succeed` の結果（`succeeded`）を、それぞれ新しい `expectedVersion` とともに順に `save` し、その都度 `findById` を呼ぶ | `prepared` では `effects`（`Sampled` の `total` と `items` を含む）を、`succeeded` では `success`（`operationId`、`fingerprints`、`appliedMigrations` の順序、`effects`）と `finishedAt` を保った `entity` を返す |
| TC-ports-schemaApplyJobRepository-15 | 準備の `phase` が `tallying` の事前確認のジョブがある | `complete` の結果を `save` し、`findById` を呼ぶ | `blockers`（`total` と `items`）と `effects` を保った `checked` の `entity` を返す |
| TC-ports-schemaApplyJobRepository-16 | 反映のジョブがある | `failure` が `blocked`、`capacityExceeded`（準備の途中の `preparation`）、`interrupted`（準備を終えた `preparation`）、`commitRejected` の `failed` のジョブを、別の ID のジョブに `save` し、それぞれ `findById` で読む | それぞれ、保存した値と全フィールドが等しい `entity` を返す。`blockers` の `total` と `items`、`preparation.phase`、`effects` の有無、`cause` を保つ |
| TC-ports-schemaApplyJobRepository-17 | `preparing` の事前確認のジョブがある | `failIfSchemaChanged` が返す `failed` のジョブを `save` し、`findById` を呼ぶ | `failure` が `schemaChanged` の `entity` を返す |
| TC-ports-schemaApplyJobRepository-18 | `preparing` のジョブがある。v を得た後、別の UnitOfWork が `save` を確定した | v とともに `save` する | `ConflictError`（`STALE_VERSION`）になる。`findById` は、先に確定した値を返す |
| TC-ports-schemaApplyJobRepository-19 | `preparing` のジョブがある。二つの処理が、同じ `expectedVersion` を得た | 並行する二つの UnitOfWork が、一方は `advance` の結果を、もう一方は `fail` が返すジョブを `save` する | 一方だけが確定し、もう一方は `ConflictError`（`STALE_VERSION`）になる。`findById` は、確定した側の値を返す |
| TC-ports-schemaApplyJobRepository-20 | `prepared` のジョブがある。二つの処理が、同じ `expectedVersion` を得た | 並行する二つの UnitOfWork が、一方は `succeed` の結果のジョブを、もう一方は `fail` が返すジョブを `save` する | 一方だけが確定し、もう一方は `ConflictError`（`STALE_VERSION`）になる |
| TC-ports-schemaApplyJobRepository-21 | J のジョブを `insert` し、v を得た後、`deleteAll` で消去済み | J のジョブを v とともに `save` する | `ConflictError`（`STALE_VERSION`）になる。`findById(J)` は `null` を返す |

## findInProgress

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-schemaApplyJobRepository-22 | ジョブが無い | `findInProgress` を呼ぶ | 空の列を返す |
| TC-ports-schemaApplyJobRepository-23 | `preparing` のジョブが 1 件ある | `findInProgress` を呼ぶ | その 1 件を、`findById` が返すものと等しい `expectedVersion` とともに返す |
| TC-ports-schemaApplyJobRepository-24 | 全状態のジョブがある | `findInProgress` を呼ぶ | 反映の `preparing`・`prepared` と、事前確認の `preparing` の 3 件だけを返す |
| TC-ports-schemaApplyJobRepository-25 | W1 と W2 に、`preparing` のジョブが 1 件ずつある | W1 の `findInProgress` を呼ぶ | W1 の 1 件だけを返す |
| TC-ports-schemaApplyJobRepository-26 | W の `Workspace`（`WorkspaceRepository` で確定済み）は `closing`。`preparing` のジョブが 1 件ある | `findInProgress` を呼ぶ | その 1 件を返す。ワークスペースの状態を見ない |
| TC-ports-schemaApplyJobRepository-27 | W の区画に何も確定していない | `findInProgress` を呼ぶ | 空の列を返す |
| TC-ports-schemaApplyJobRepository-28 | `progressedAt` が p2、p3、p1 の進行中のジョブを、この順に `insert` 済み | `findInProgress` を呼ぶ | `progressedAt` が p1、p2、p3 の順で返す |
| TC-ports-schemaApplyJobRepository-29 | `progressedAt` が同じ p1 で、ID が異なる進行中のジョブが 3 件ある | `findInProgress` を呼ぶ | ジョブの ID のコードポイントの昇順で返す |
| TC-ports-schemaApplyJobRepository-30 | `progressedAt` が p1、p2、p3 の進行中のジョブがある | `findInProgress({ progressedBefore: p2, after: null }, 10)` を呼ぶ | p1 と p2 の 2 件を返す。`progressedBefore` と同じ日時のジョブを含む |
| TC-ports-schemaApplyJobRepository-31 | `progressedAt` が p2 と p3 の進行中のジョブがある | `findInProgress({ progressedBefore: p1, after: null }, 10)` を呼ぶ | 空の列を返す |
| TC-ports-schemaApplyJobRepository-32 | `progressedAt` が p1 の `preparing` のジョブに、`advance` で `progressedAt` を p3 にした結果を `save` 済み | `findInProgress({ progressedBefore: p2, after: null }, 10)` を呼ぶ | 空の列を返す |
| TC-ports-schemaApplyJobRepository-33 | `progressedAt` が p1 の `preparing` のジョブに、`fail` が返すジョブを `save` 済み | `findInProgress` を呼ぶ | そのジョブを返さない |
| TC-ports-schemaApplyJobRepository-34 | `progressedAt` が p1、p2、p3 の進行中のジョブがある | `findInProgress({ progressedBefore: null, after: null }, 2)` を呼び、返った列の最後のジョブの `progressedAt` と ID を `after` に渡して、もう一度呼ぶ | 1 回目は p1 と p2 の 2 件、2 回目は p3 の 1 件を返す。重複も欠落も無い |
| TC-ports-schemaApplyJobRepository-35 | `progressedAt` が同じ p1 で、ID が異なる進行中のジョブが 3 件ある | `limit` を 2 にして呼び、返った列の最後のジョブの位置を `after` に渡して、もう一度呼ぶ | 1 回目は ID の昇順で先頭の 2 件、2 回目は残りの 1 件を返す |
| TC-ports-schemaApplyJobRepository-36 | `progressedAt` が p1、p2、p3 の進行中のジョブがある。p1 のジョブの位置を `after` に渡す読み取りの前に、p1 のジョブに `fail` が返すジョブを `save` 済み | p1 のジョブの位置を `after` に渡して呼ぶ | p2 と p3 の 2 件を返す。位置のジョブが進行中でなくても、その位置より後ろを返す |
| TC-ports-schemaApplyJobRepository-37 | 進行中のジョブが、上限より 1 件多くある | `limit` を上限にして呼ぶ | 契約の順で先頭から、上限の件数を返す |
| TC-ports-schemaApplyJobRepository-38 | `preparing` のジョブが 1 件ある | `limit` を 0 にして呼ぶ | `SystemError`（`INVALID_QUERY`）を投げる |
| TC-ports-schemaApplyJobRepository-39 | `preparing` のジョブが 1 件ある | `limit` を、上限より 1 大きい値にして呼ぶ | `SystemError`（`INVALID_QUERY`）を投げる |
| TC-ports-schemaApplyJobRepository-40 | `findInProgress` が返したジョブと `expectedVersion` がある | `fail` が返すジョブを、その `expectedVersion` とともに `save` する | 確定する。`findById` は `failed` のジョブを返す |

## countInProgress

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-schemaApplyJobRepository-41 | ジョブが無い | `countInProgress()` を呼ぶ | `commit` も `dryRun` も 0 |
| TC-ports-schemaApplyJobRepository-42 | W の区画に何も確定していない | `countInProgress()` を呼ぶ | `commit` も `dryRun` も 0 |
| TC-ports-schemaApplyJobRepository-43 | 全状態のジョブがある | `countInProgress()` を呼ぶ | `commit` は 2（`preparing` と `prepared`）、`dryRun` は 1 |
| TC-ports-schemaApplyJobRepository-44 | W1 に `preparing` の事前確認のジョブが 2 件、W2 に 1 件ある | W1 の `countInProgress()` を呼ぶ | `dryRun` は 2 |
| TC-ports-schemaApplyJobRepository-45 | W の `Workspace`（`WorkspaceRepository` で確定済み）は `closing`。`preparing` の事前確認のジョブが 1 件ある | `countInProgress()` を呼ぶ | `dryRun` は 1 |
| TC-ports-schemaApplyJobRepository-46 | `preparing` の事前確認のジョブが、`findInProgress` の上限より 1 件多くある | `countInProgress()` を呼ぶ | `dryRun` は、その件数。上限で頭打ちにならない |
| TC-ports-schemaApplyJobRepository-47 | `preparing` の事前確認のジョブが 2 件ある。一件に、`fail` が返すジョブを `save` 済み | `countInProgress()` を呼ぶ | `dryRun` は 1 |
| TC-ports-schemaApplyJobRepository-48 | `preparing` の事前確認のジョブが 1 件ある | 一つの UnitOfWork で、`fail` が返すジョブを `save` した後、`fn` が例外を投げる | `countInProgress()` の `dryRun` は 1 |

## findApplyResultsPage

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-schemaApplyJobRepository-49 | ジョブが無い | `findApplyResultsPage` を呼ぶ | `items` は空、`count` は 0 |
| TC-ports-schemaApplyJobRepository-50 | W の区画に何も確定していない | `findApplyResultsPage` を呼ぶ | `items` は空、`count` は 0 |
| TC-ports-schemaApplyJobRepository-51 | 全状態のジョブがある | `findApplyResultsPage` を呼ぶ | `items` は、反映の `succeeded` と `failed` の 2 件、`count` は 2。進行中のジョブと、事前確認のジョブ（`checked`、`failed`）を含まない |
| TC-ports-schemaApplyJobRepository-52 | `finishedAt` が f2、f1、f3 の終端の反映のジョブを、この順に終端にして `save` 済み | `findApplyResultsPage` を呼ぶ | `items` は `finishedAt` が f3、f2、f1 の順、`count` は 3 |
| TC-ports-schemaApplyJobRepository-53 | `finishedAt` が同じ f1 で、ID が異なる終端の反映のジョブが 3 件ある | `findApplyResultsPage` を呼ぶ | `items` はジョブの ID のコードポイントの昇順 |
| TC-ports-schemaApplyJobRepository-54 | `finishedAt` が f1 の `failed` の反映のジョブと、f2 の `succeeded` の反映のジョブがある | `findApplyResultsPage` を呼ぶ | `items` は 2 件。成功も失敗も返す |
| TC-ports-schemaApplyJobRepository-55 | 終端の反映のジョブが 5 件ある | `{ page: 1, limit: 2 }`、`page: 2`、`page: 3`、`page: 4` を順に呼ぶ | `items` は順に 2 件、2 件、1 件、0 件で、通して契約の順。重複も欠落も無い。`count` はどれも 5 |
| TC-ports-schemaApplyJobRepository-56 | 終端の反映のジョブが、上限より 1 件多くある | `limit` を上限にして、`page` が 1 と 2 を呼ぶ | `items` は、上限の件数と 1 件。`count` はどちらも、その件数 |
| TC-ports-schemaApplyJobRepository-57 | W1 に終端の反映のジョブが 2 件、W2 に 3 件ある | W1 の `findApplyResultsPage` を呼ぶ | `items` は W1 の 2 件だけ、`count` は 2 |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-schemaApplyJobRepository-58 | ジョブが無い | `deleteAll(10)` を呼ぶ | 0 を返す |
| TC-ports-schemaApplyJobRepository-59 | 全状態のジョブがある | `deleteAll(10)` を呼び、確定の後に各ジョブの `findById` を呼ぶ | `limit` より少ない 7 を返す。進行中のジョブも消去する。`findById` はどれも `null`。`findInProgress` は空の列を、`findApplyResultsPage` は `count` が 0 を返す |
| TC-ports-schemaApplyJobRepository-61 | ジョブが 5 件ある | `deleteAll(2)` を、呼び出しごとに別の UnitOfWork で、返す件数が 2 より少なくなるまで繰り返す | 返す値は順に 2、2、1。最後は、どのジョブの `findById` も `null` |
| TC-ports-schemaApplyJobRepository-62 | W1 にジョブが 2 件、W2 にジョブが 3 件ある | W1 の `deleteAll(10)` を呼び、確定の後に W2 の各ジョブの `findById` を呼ぶ | 2 を返す。W2 の 3 件は残る |
| TC-ports-schemaApplyJobRepository-63 | `insert` の後に `save` を 2 回確定したジョブがある。`expectedVersion` を読んでいない | `deleteAll(10)` を呼ぶ | 1 を返し、確定する。楽観ロックを使わない |
| TC-ports-schemaApplyJobRepository-64 | ジョブが 3 件ある | `limit` を上限にして呼ぶ | 3 を返す |
| TC-ports-schemaApplyJobRepository-65 | ジョブが 3 件ある | `deleteAll(0)` を呼ぶ | `SystemError`（`INVALID_QUERY`）を投げる。3 件は残る |
| TC-ports-schemaApplyJobRepository-66 | ジョブが 3 件ある | `limit` を、上限より 1 大きい値にして呼ぶ | `SystemError`（`INVALID_QUERY`）を投げる。3 件は残る |

## UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-schemaApplyJobRepository-67 | ジョブが無い | 一つの UnitOfWork で、ジョブの `insert` を呼び、`schema.apply_requested` を `collectEvents` に渡して、正常に終える | `findById` はジョブを返す。イベントは、試験用の消費者に届く |
| TC-ports-schemaApplyJobRepository-68 | ジョブが無い | 一つの UnitOfWork で、ジョブの `insert` を呼んだ後、`fn` が例外を投げる | `run` はその例外を投げる。`findById` は `null` を返す |
| TC-ports-schemaApplyJobRepository-69 | W の `WorkspaceSchema` が `idle`。J のジョブが無い | 一つの UnitOfWork で、ID J の反映のジョブの `insert` と、`beginApply` の結果の `WorkspaceSchemaRepository.save` を呼んだ後、`fn` が例外を投げる | `findById(J)` は `null`。`WorkspaceSchemaRepository.find` は `idle` の値を返す |
| TC-ports-schemaApplyJobRepository-70 | W の `WorkspaceSchema` が、反映のジョブ J で `applying` | 一つの UnitOfWork で、`fail` が返すジョブの `save` と、`endApply` の結果の `WorkspaceSchemaRepository.save` を呼び、正常に終える | `findById(J)` は `failed` のジョブを、`WorkspaceSchemaRepository.find` は `idle` の値を返す |
| TC-ports-schemaApplyJobRepository-71 | 同上 | 一つの UnitOfWork で、同じ二つを呼んだ後、`fn` が例外を投げる | `run` はその例外を投げる。`findById(J)` は進行中のジョブを、`WorkspaceSchemaRepository.find` は `applying` の値を返す |
| TC-ports-schemaApplyJobRepository-72 | 同上。ジョブの v を得た後、別の UnitOfWork が、ジョブの `advance` の結果の `save` を確定した | 一つの UnitOfWork で、v とともに `fail` が返すジョブの `save` と、`endApply` の結果の `WorkspaceSchemaRepository.save` を呼ぶ | `ConflictError`（`STALE_VERSION`）になる。`WorkspaceSchemaRepository.find` は `applying` の値を返す |
| TC-ports-schemaApplyJobRepository-73 | `preparing` のジョブがある | `advance` の結果を `save` した後、`fn` が例外を投げる | `findById` は、保存の前の値と、前と等しい `expectedVersion` を返す |
| TC-ports-schemaApplyJobRepository-74 | ジョブが 3 件ある | `deleteAll(10)` を呼んだ後、`fn` が例外を投げる | 3 件は残る |
