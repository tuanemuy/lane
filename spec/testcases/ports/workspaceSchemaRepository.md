# WorkspaceSchemaRepository

契約は [schema.md](../../domains/schema.md) の「ポート」「WriteScope とスキーマ」と、[common.md](../../domains/common.md) の「SingletonRepository」「TransactionalRepository」「WriteScope」「UnitOfWorkProvider」が定める。

## 記法

- W、W1、W2 は、互いに違うワークスペース。リポジトリは、ワークスペースの区画ごとにあり、メソッドはワークスペースの ID を取らない。「W の `find`」のように書いたワークスペースは、呼ぶリポジトリが結び付く区画を指す。`WorkspaceSchema` は `WorkspaceSchema` の振る舞い（`create`、`beginApply`、`commit`、`endApply`、`clearFailedApply`、`recordApplied`）が返す値で作り、`insert` と `save` で保存する。振る舞いに渡すジョブは、`Preparation` と `SchemaApplyJob` の振る舞いで作る。`WorkspaceSchema` は、ワークスペースの ID を持たない
- 前提条件の書き込みは、そのワークスペースの UnitOfWork（`UnitOfWorkProvider.run` に `{ kind: "workspace", workspaceId, condition }` を渡した UnitOfWork。[unitOfWorkProvider.md](unitOfWorkProvider.md)）の `WorkspaceContext` のリポジトリで呼び、確定済み。操作の書き込みは、新しい UnitOfWork で呼ぶ。読み取りは UnitOfWork の外で、`PartitionReader.workspace` のリポジトリで呼ぶ。断りの無い UnitOfWork の条件は `none`
- 「`ConflictError`（理由）になる」は、メソッドの呼び出しか確定のどちらかでエラーが現れ、`run` がそのエラーを投げ、その UnitOfWork の書き込みが一つも反映されないことを指す
- 「ジョブの `insert`」は、同じ UnitOfWork で呼ぶ、事前確認のジョブの `SchemaApplyJobRepository.insert` を指す。条件の UnitOfWork が確定したかどうかを、`SchemaApplyJobRepository.findById` で確かめるために使う
- v は、直前の `find` が返した `expectedVersion`
- 「`stableSchema(W, n)` の UnitOfWork」は、`WriteScope` が `{ kind: "workspace", workspaceId: W, condition: { kind: "stableSchema", schemaVersion: n } }` の UnitOfWork。「`schemaVersion(W, n)` の UnitOfWork」も同じ形で、`condition` の `kind` が `schemaVersion`。「条件」の二つの節では、W、W1、W2 の区画は `active` の `Workspace`（`WorkspaceRepository` で確定済み）を持つ
- 「`applying` にする」は `beginApply` の結果の `save`、「確定する」は `commit` の結果（`CommittedApply.workspaceSchema`）の `save`、「失敗で終える」は `endApply` の結果の `save` を指す。「未解消の失敗した反映」は `failedApply` を指す。「上限」は、`deleteAll` の `BoundedDeletion.maxLimit`

## insert と find

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-workspaceSchemaRepository-01 | W の `WorkspaceSchema` が無い | W の `find` を呼ぶ | `null` を返す |
| TC-ports-workspaceSchemaRepository-02 | W の `WorkspaceSchema` が無い | W のリポジトリで、`WorkspaceSchema.create(now)` の結果を `insert` し、確定の後に `find` を呼ぶ | `insert` した値と全フィールドが等しい `entity` と、`expectedVersion` を返す |
| TC-ports-workspaceSchemaRepository-03 | W の `WorkspaceSchema` がある | W の `WorkspaceSchema` を `insert` する | `ConflictError`（`ALREADY_EXISTS`）になる。`find` は、先に `insert` した値を返す |
| TC-ports-workspaceSchemaRepository-04 | W1 の `WorkspaceSchema` がある | W2 の `WorkspaceSchema` を W2 のリポジトリで `insert` し、W1 と W2 の `find` を呼ぶ | それぞれ、自分のワークスペースの `WorkspaceSchema` を返す |
| TC-ports-workspaceSchemaRepository-05 | W の `WorkspaceSchema` が無い | 並行する二つの UnitOfWork が、それぞれ W の `WorkspaceSchema` を `insert` する | 一方だけが確定し、もう一方は `ConflictError`（`ALREADY_EXISTS`）になる。`find` は、確定した側の値を返す |
| TC-ports-workspaceSchemaRepository-06 | `schema`・`modelIds`・`formIds`・`appliedMigrations`・`migratedStructure` が空でなく、`state` がジョブ J で `applying` の `WorkspaceSchema` を保存済み | `find` を呼ぶ | 保存した値と全フィールドが等しい `entity` を返す。`state.jobId`、`appliedMigrations` の順序、`schema` の配列の順序を保つ |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-workspaceSchemaRepository-07 | W の `WorkspaceSchema` がある | `beginApply` の結果を v とともに `save` し、確定の後に `find` を呼ぶ | `state` が `applying` の、保存した値を返す |
| TC-ports-workspaceSchemaRepository-08 | W が `applying` | `commit` の結果を v とともに `save` し、`find` を呼ぶ | 保存した値と全フィールドが等しい `entity` を返す。`idle` で、`schemaVersion` が 1 進んでいる |
| TC-ports-workspaceSchemaRepository-09 | W の `WorkspaceSchema` が `idle`。J のジョブは `SchemaApplyJobRepository` に無い | `state` が J で `applying` の値を v とともに `save` する | 確定する。`jobId` が指すジョブの有無を確かめない |
| TC-ports-workspaceSchemaRepository-10 | W が `applying` | `endApply` の結果を v とともに `save` し、`find` を呼ぶ | `idle` で、`schemaVersion` が前と同じ `entity` を返す。未解消の失敗した反映は、渡したジョブの ID |
| TC-ports-workspaceSchemaRepository-11 | W が `idle` で、未解消の失敗した反映を持つ | `clearFailedApply` の結果を v とともに `save` し、`find` を呼ぶ | 未解消の失敗した反映が `null` で、`schemaVersion` と `state` が前と同じ `entity` を返す |
| TC-ports-workspaceSchemaRepository-12 | W が、未解消の失敗した反映を持ち、別のジョブで `applying` | `find` を呼ぶ | `state.jobId` と未解消の失敗した反映を、保存した値のまま返す |
| TC-ports-workspaceSchemaRepository-13 | W の `WorkspaceSchema` がある。v を得た後、別の UnitOfWork が `save` を確定した | 別の変更を v とともに `save` する | `ConflictError`（`STALE_VERSION`）になる。`find` は、先に確定した値を返す |
| TC-ports-workspaceSchemaRepository-14 | W の `WorkspaceSchema` が `idle`。二つの処理が、同じ `expectedVersion` を得た | 並行する二つの UnitOfWork が、それぞれ別の反映の `beginApply` の結果を `save` する | 一方だけが確定し、もう一方は `ConflictError`（`STALE_VERSION`）になる。`find` は、確定した側の反映で `applying` の値を返す |
| TC-ports-workspaceSchemaRepository-15 | W の `WorkspaceSchema` を `insert` し、v を得た後、`deleteAll(1)` を確定済み | W の `WorkspaceSchema` を v とともに `save` する | `ConflictError`（`STALE_VERSION`）になる。`find` は `null` を返す |
| TC-ports-workspaceSchemaRepository-16 | W の `WorkspaceSchema` が一度も無い。W2 の `find` で得た `expectedVersion` がある | W の `WorkspaceSchema` を、その `expectedVersion` とともに `save` する | `ConflictError`（`STALE_VERSION`）になる。`find` は `null` を返す |

## 条件（stableSchema）

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-workspaceSchemaRepository-17 | W の `WorkspaceSchema` が `idle` で、`schemaVersion` が 0 | `stableSchema(W, 0)` の UnitOfWork で、ジョブの `insert` を呼ぶ | 確定する。ジョブを読める。`find` は、前と同じ `entity` と、前と等しい `expectedVersion` を返す |
| TC-ports-workspaceSchemaRepository-18 | W の `WorkspaceSchema` が `idle` で、`schemaVersion` が 0 | `stableSchema(W, 0)` の UnitOfWork で、書き込みを行わない | 確定する。`find` は、前と同じ `entity` と `expectedVersion` を返す |
| TC-ports-workspaceSchemaRepository-19 | W の `WorkspaceSchema` が `idle`。`recordApplied` の結果を `save` 済みで、`schemaVersion` が 1 | `stableSchema(W, 0)` の UnitOfWork と、`stableSchema(W, 2)` の UnitOfWork で、それぞれジョブの `insert` を呼ぶ | どちらも `ConflictError`（`SCHEMA_CHANGED`）になる。ジョブは `null` |
| TC-ports-workspaceSchemaRepository-20 | 同上 | `stableSchema(W, 1)` の UnitOfWork で、ジョブの `insert` を呼ぶ | 確定する |
| TC-ports-workspaceSchemaRepository-22 | W を `applying` にした。`schemaVersion` は 0 | `stableSchema(W, 0)` の UnitOfWork で、ジョブの `insert` を呼ぶ | `ConflictError`（`SCHEMA_CHANGED`）になる。ジョブは `null` |
| TC-ports-workspaceSchemaRepository-24 | W を `applying` にし、確定した。`schemaVersion` が 1 で `idle` | `stableSchema(W, 0)` の UnitOfWork と、`stableSchema(W, 1)` の UnitOfWork を、順に実行する | 前者は `ConflictError`（`SCHEMA_CHANGED`）になる。後者は確定する |
| TC-ports-workspaceSchemaRepository-25 | W を `applying` にし、失敗で終えた。`schemaVersion` は 0 のまま、`version` は 2 進んでいる | `stableSchema(W, 0)` の UnitOfWork で、ジョブの `insert` を呼ぶ | 確定する。`version` を見ない |
| TC-ports-workspaceSchemaRepository-26 | W の `WorkspaceSchema` が `idle` で、未解消の失敗した反映を持つ。`schemaVersion` が 0 | 並行する二つの UnitOfWork の、一方が `stableSchema(W, 0)` でジョブの `insert` を、もう一方が `clearFailedApply` の結果の `save` を呼ぶ | 両方が確定する |
| TC-ports-workspaceSchemaRepository-27 | W の `WorkspaceSchema` が無い | `stableSchema(W, 0)` の UnitOfWork で、ジョブの `insert` を呼ぶ | `ConflictError`（`SCHEMA_CHANGED`）になる。ジョブは `null` |
| TC-ports-workspaceSchemaRepository-29 | W の区画に `Workspace` も `WorkspaceSchema` も無い | `stableSchema(W, 0)` の UnitOfWork と、`schemaVersion(W, 0)` の UnitOfWork で、それぞれジョブの `insert` を呼ぶ | どちらも `ConflictError`（`WORKSPACE_NOT_ACTIVE`）になる。`SCHEMA_CHANGED` にならない。ジョブは `null` |
| TC-ports-workspaceSchemaRepository-30 | W1 の `schemaVersion` が 1、W2 の `schemaVersion` が 0 | `stableSchema(W2, 0)` の UnitOfWork と、`stableSchema(W2, 1)` の UnitOfWork を、順に実行する | 前者は確定する。後者は `ConflictError`（`SCHEMA_CHANGED`）になる |
| TC-ports-workspaceSchemaRepository-31 | W の `WorkspaceSchema` が `idle` で、`schemaVersion` が 0 | 並行する二つの `stableSchema(W, 0)` の UnitOfWork が、それぞれ ID の異なるジョブの `insert` を呼ぶ | 両方が確定し、両方のジョブを読める |
| TC-ports-workspaceSchemaRepository-32 | W の `WorkspaceSchema` が `idle` で、`schemaVersion` が 0 | 並行する二つの UnitOfWork の、一方が `stableSchema(W, 0)` でジョブの `insert` を、もう一方が `beginApply` の結果の `save` を呼ぶ | `save` の側は確定する。`stableSchema` の側は、確定するか、`ConflictError`（`SCHEMA_CHANGED`）になるかのどちらか。後者では、ジョブは `null` |
| TC-ports-workspaceSchemaRepository-33 | W の `WorkspaceSchema` が `idle` で、`schemaVersion` が 0 | 並行する二つの UnitOfWork の、一方が `stableSchema(W, 0)` でジョブの `insert` を、もう一方が `recordApplied` の結果の `save` を呼ぶ | `save` の側は確定する。`stableSchema` の側は、確定するか、`ConflictError`（`SCHEMA_CHANGED`）になるかのどちらか |

## 条件（schemaVersion）

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-workspaceSchemaRepository-34 | W の `WorkspaceSchema` が `idle` で、`schemaVersion` が 0 | `schemaVersion(W, 0)` の UnitOfWork で、ジョブの `insert` を呼ぶ | 確定する。`find` は、前と同じ `entity` と `expectedVersion` を返す |
| TC-ports-workspaceSchemaRepository-35 | W の `WorkspaceSchema` が `applying` で、`schemaVersion` が 0 | `schemaVersion(W, 0)` の UnitOfWork で、ジョブの `insert` を呼ぶ | 確定する。適用中かどうかを見ない |
| TC-ports-workspaceSchemaRepository-36 | W の `WorkspaceSchema` の `schemaVersion` が 1 | `schemaVersion(W, 0)` の UnitOfWork と、`schemaVersion(W, 2)` の UnitOfWork で、それぞれジョブの `insert` を呼ぶ | どちらも `ConflictError`（`SCHEMA_CHANGED`）になる。ジョブは `null` |
| TC-ports-workspaceSchemaRepository-38 | W を `applying` にし、失敗で終えた。`schemaVersion` は 0 のまま | `schemaVersion(W, 0)` の UnitOfWork で、ジョブの `insert` を呼ぶ | 確定する。`version` を見ない |
| TC-ports-workspaceSchemaRepository-39 | W の `WorkspaceSchema` が無い | `schemaVersion(W, 0)` の UnitOfWork で、ジョブの `insert` を呼ぶ | `ConflictError`（`SCHEMA_CHANGED`）になる。ジョブは `null` |
| TC-ports-workspaceSchemaRepository-40 | W の `WorkspaceSchema` が `idle` で、`schemaVersion` が 0 | `schemaVersion(W, 0)` の UnitOfWork で、書き込みを行わない | 確定する |
| TC-ports-workspaceSchemaRepository-41 | W の `WorkspaceSchema` が `idle` で、`schemaVersion` が 0 | 並行する二つの `schemaVersion(W, 0)` の UnitOfWork が、それぞれ ID の異なるジョブの `insert` を呼ぶ | 両方が確定する |
| TC-ports-workspaceSchemaRepository-42 | W の `WorkspaceSchema` が `idle` で、`schemaVersion` が 0 | 並行する `schemaVersion(W, 0)` の UnitOfWork と `stableSchema(W, 0)` の UnitOfWork が、それぞれジョブの `insert` を呼ぶ | 両方が確定する |
| TC-ports-workspaceSchemaRepository-43 | W の `WorkspaceSchema` が `idle` で、`schemaVersion` が 0 | 並行する二つの UnitOfWork の、一方が `schemaVersion(W, 0)` でジョブの `insert` を、もう一方が `beginApply` の結果の `save` を呼ぶ | 両方が確定する。スキーマの版を進めない保存とは両立する |
| TC-ports-workspaceSchemaRepository-44 | W の `WorkspaceSchema` が `applying`。`schemaVersion` が 0 | 並行する二つの UnitOfWork の、一方が `schemaVersion(W, 0)` でジョブの `insert` を、もう一方が `commit` の結果の `save` を呼ぶ | `save` の側は確定する。`schemaVersion` の側は、確定するか、`ConflictError`（`SCHEMA_CHANGED`）になるかのどちらか。後者では、ジョブは `null` |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-workspaceSchemaRepository-45 | W の `WorkspaceSchema` がある | `deleteAll(1)` を呼び、確定の後に `find` を呼ぶ | 1 を返す。`find` は `null` を返す |
| TC-ports-workspaceSchemaRepository-46 | W の `WorkspaceSchema` が無い | `deleteAll(1)` を呼ぶ | 0 を返し、確定する |
| TC-ports-workspaceSchemaRepository-47 | W1 と W2 の `WorkspaceSchema` がある | W1 の `deleteAll(1)` を呼び、確定の後に W2 の `find` を呼ぶ | W2 の `WorkspaceSchema` を返す |
| TC-ports-workspaceSchemaRepository-48 | W の `WorkspaceSchema` を `insert` した後、`save` を 2 回確定済み。`expectedVersion` を読んでいない | `deleteAll(1)` を呼ぶ | 1 を返し、確定する。楽観ロックを使わない |
| TC-ports-workspaceSchemaRepository-49 | W の `WorkspaceSchema` が `applying` | `deleteAll(1)` を呼ぶ | 1 を返し、確定する |
| TC-ports-workspaceSchemaRepository-50 | W の `WorkspaceSchema` がある | `limit` を上限にして `deleteAll` を呼ぶ | `limit` より少ない 1 を返す。`find` は `null` を返す |
| TC-ports-workspaceSchemaRepository-51 | W の `WorkspaceSchema` がある | `deleteAll(0)` を呼ぶ | `SystemError`（`INVALID_QUERY`）を投げる。`find` は `WorkspaceSchema` を返す |
| TC-ports-workspaceSchemaRepository-52 | W の `WorkspaceSchema` がある | `limit` を、上限より 1 大きい値にして `deleteAll` を呼ぶ | `SystemError`（`INVALID_QUERY`）を投げる。`find` は `WorkspaceSchema` を返す |

## UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-workspaceSchemaRepository-53 | W の `WorkspaceSchema` が、準備済みのジョブ J で `applying` | 一つの UnitOfWork で、`commit` の結果の `save`、`SchemaVersionRecordRepository.append`、`SchemaApplyJob.succeed` の結果のジョブの `SchemaApplyJobRepository.save` を呼び、正常に終える | `find` は保存した値を返す。`SchemaVersionRecordRepository.findPage` は追記した版の記録を、`SchemaApplyJobRepository.findById(J)` は `succeeded` のジョブを返す |
| TC-ports-workspaceSchemaRepository-54 | 同上 | 一つの UnitOfWork で、同じ三つを呼んだ後、`fn` が例外を投げる | `run` はその例外を投げる。`find` は保存の前の値と、前と等しい `expectedVersion` を返す。版の記録は無く、ジョブ J は `prepared` のまま |
| TC-ports-workspaceSchemaRepository-55 | W の `WorkspaceSchema` が、ジョブ J で `applying`。v を得た後、別の UnitOfWork が `save` を確定した | 一つの UnitOfWork で、v とともに `endApply` の結果を `save` し、`fail` が返すジョブの `SchemaApplyJobRepository.save` を呼ぶ | `ConflictError`（`STALE_VERSION`）になる。ジョブ J は進行中のまま |
| TC-ports-workspaceSchemaRepository-56 | W の `WorkspaceSchema` が無い | `insert` を呼んだ後、`fn` が例外を投げる | `find` は `null` を返す |
| TC-ports-workspaceSchemaRepository-57 | W の `WorkspaceSchema` がある | `deleteAll(1)` を呼んだ後、`fn` が例外を投げる | `find` は `WorkspaceSchema` を返す |
