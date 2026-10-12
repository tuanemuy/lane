# RevisionRepository

「上限」は、その節のメソッドの契約が定める、一度に渡せる件数・個数、または `limit` の上限を指す。値は、ポートを定めるドメインのファイルが持つ。

契約は [content.md](../../domains/content.md) の「Revision」「ポート」「一度に扱う対象」「参照の記録と保護」と、[common.md](../../domains/common.md) の「TransactionalRepository」の共通の規則・「UnitOfWorkProvider」が定める。

## 記法

- リポジトリは、ワークスペース `W1` の区画に結び付く。メソッドは、ワークスペースの ID を引数に取らない。`W0` は、区画に何も確定していないワークスペース
- UnitOfWork は、`{ kind: "workspace", workspaceId: W1, condition: { kind: "none" } }` を渡した `UnitOfWorkProvider.run` を指す。書き込みは、その `WorkspaceContext` のリポジトリで呼ぶ。UnitOfWork の外の読み取りは、`PartitionReader.workspace(W1)` のリポジトリで呼ぶ
- A、B、T はコンテンツで、`ContentRepository.insert` を確定した状態。R1、R2 などは版
- 「`deleteMany`（R1 と R2）」は、`ids` に、R1 と R2 の ID を渡す呼び出しを指す
- 前提条件の版は、`Revision.record` で作った値を `insertMany` で保存する。前提条件の書き込みは、UnitOfWork で呼び、確定済み
- 操作の書き込みは、新しい UnitOfWork で呼び、UnitOfWork を確定する。読み取りは、UnitOfWork の外で呼ぶ
- 「`ConflictError` になる」は、メソッドの呼び出しか UnitOfWork の確定のどちらかでエラーが現れ、`run` がそのエラーを投げ、その UnitOfWork の書き込みが 1 件も反映されないことを指す
- 「`SystemError`（`INVALID_QUERY`）になる」は、呼び出しがそのエラーを返すことを指す。書き込みでは、`run` がそのエラーを投げ、その UnitOfWork の書き込みが 1 件も反映されない
- ID の大小は [common.md](../../domains/common.md) の「TransactionalRepository」の順序に従う。`cutoff` は時点 `t`
- 「組」は、読み取りが返す、版の ID とコンテンツID の `RevisionTarget`
- 断りが無い `findInbound` は、`ContentReferenceQuery.findInbound` を、`holders` が `published`・`draft`・`revision`、`strengths` が `required`・`optional`・`link`、`page` が 1、`limit` が 100 で呼ぶ

## insertMany

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-revisionRepository-01 | A がある。版は無い | A の版 R1 を `insertMany` に渡す | `findById`（R1）が、渡した R1 とすべてのフィールドが等しい版を返す |
| TC-ports-revisionRepository-02 | A がある。版は無い | 空の `revisions` を `insertMany` に渡す | 成功する。`count` は 0 |
| TC-ports-revisionRepository-03 | A がある。版は無い | `origin` が `published` の R1 と、`origin` が `draft` の R2 を、一度の `insertMany` に渡す | `findById` が R1 と R2 をそれぞれ返す。`findPage`（A）の `count` は 2 |
| TC-ports-revisionRepository-04 | R1 がある | 新しい版 R2 と、R1 と同じ `id` の版を、一度の `insertMany` に渡す | `ConflictError`（`ALREADY_EXISTS`）になる。`findById`（R1）は前提条件の R1 を、`findById`（R2）は `null` を返す |
| TC-ports-revisionRepository-06 | A がある。版は無い | 同じ `id` の版を二つ、一度の `insertMany` に渡す | `ConflictError`（`ALREADY_EXISTS`）になる。`findPage`（A）の `count` は 0 |
| TC-ports-revisionRepository-07 | A と T がある | T への必須の参照を持つ A の版 R1 を `insertMany` に渡す | `findInbound`（対象 T）が、参照元 A、`holder` が `revision` で `revisionId` が R1、`strength` が `required` の記録を返す |
| TC-ports-revisionRepository-08 | A がある。ID が `x` のコンテンツは無い | `x` への必須の参照を持つ R1、`x` への本文のリンク（`link`）を持つ R2、`x` への任意の参照を持つ R3 を、A の版として一度の `insertMany` に渡す | 成功する。`findById` が R1、R2、R3 を返す |
| TC-ports-revisionRepository-11 | A がある。版は無い | `ContentBatch.maxContents` の 2 倍の件数の版を、一度の `insertMany` に渡す | 成功する。`count` は、渡した件数 |
| TC-ports-revisionRepository-12 | A がある。版は無い | `ContentBatch.maxContents` の 2 倍を 1 件超える件数の版を、一度の `insertMany` に渡す | `SystemError`（`INVALID_QUERY`）になる。`count` は 0 |

## findById

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-revisionRepository-13 | 版が無い | `findById`（`r`）を呼ぶ | `null` を返す |
| TC-ports-revisionRepository-14 | R1 がある | `findById`（R1 の `id`）を呼ぶ | `contentId`、`origin`、`values`、`references`、`updatedAt`、`produced`、`migrated`、`retired` が、保存した値と等しい版を返す |

## findByIds

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-revisionRepository-15 | A の版 R1 と、B の版 R2 がある。`id` が `r` の版は無い | `findByIds`（R1、R2、`r`）を呼ぶ | R1 と R2 だけを返す |
| TC-ports-revisionRepository-16 | 版の `id` が `r1` < `r2` < `r3` | `ids` に `r3`、`r1`、`r2` の順で渡す | `r1`、`r2`、`r3` の順に返す |
| TC-ports-revisionRepository-17 | R1 がある | 空の `ids` で `findByIds` を呼ぶ | 空の列を返す |
| TC-ports-revisionRepository-18 | R1 がある | `ids` に R1 の `id` を 2 回渡す | R1 を 1 件だけ返す |
| TC-ports-revisionRepository-19 | A の版が `ContentBatch.maxRevisions` 件ある | そのすべての `id` を渡す | すべての版を返す |
| TC-ports-revisionRepository-20 | A の版が `ContentBatch.maxRevisions` 件ある | そのすべての `id` に、そのうち 1 個の重複を加えて渡す | すべての版を、1 件ずつ返す |
| TC-ports-revisionRepository-21 | R1 がある | 重複の無い、`ContentBatch.maxRevisions` を 1 個超える数の `id` を渡す | `SystemError`（`INVALID_QUERY`）になる |

## findPage

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-revisionRepository-22 | A がある。A の版は無い | `findPage`（A、`page` 1、`limit` 100）を呼ぶ | 空の一覧を返す。`count` は 0 |
| TC-ports-revisionRepository-23 | A の版 R1 がある | `findPage`（A、`page` 1、`limit` 100）を呼ぶ | R1 だけを返す。`count` は 1 |
| TC-ports-revisionRepository-24 | A の版が 3 件ある。`retired.at` は R1 が最も古く、R3 が最も新しい | `findPage`（A、`page` 1、`limit` 100）を呼ぶ | R3、R2、R1 の順に返す |
| TC-ports-revisionRepository-25 | A の版が 3 件あり、`retired.at` が同じ。`id` は `a` < `b` < `c` | `findPage`（A、`page` 1、`limit` 100）を呼ぶ | `c`、`b`、`a` の順に返す |
| TC-ports-revisionRepository-26 | A の版 R1 と、B の版 R2 がある | `findPage`（A、`page` 1、`limit` 100）を呼ぶ | R1 だけを返す。`count` は 1 |
| TC-ports-revisionRepository-27 | A の版が 5 件ある | `findPage`（A、`page` 2、`limit` 2）と、`findPage`（A、`page` 3、`limit` 2）を呼ぶ | 前者は並び順の 3 番目と 4 番目、後者は 5 番目だけを返す。`count` はどちらも 5 |
| TC-ports-revisionRepository-29 | A の版が 5 件ある | `findPage`（A、`page` 4、`limit` 2）を呼ぶ | 空の一覧を返す。`count` は 5 |

## findTargetsByContents

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-revisionRepository-30 | A がある。版は無い | `findTargetsByContents`（A、`null`、1000）を呼ぶ | 空の列を返す |
| TC-ports-revisionRepository-31 | A の版 R1、R2 と、B の版 R3 と、コンテンツ C の版 R4 がある | `findTargetsByContents`（A と B、`null`、1000）を呼ぶ | R1 と A、R2 と A、R3 と B の組を返す。R4 の組を返さない |
| TC-ports-revisionRepository-32 | A に、`origin` が `published` の版と `draft` の版が 1 件ずつある | `findTargetsByContents`（A、`null`、1000）を呼ぶ | 2 件の組を返す |
| TC-ports-revisionRepository-33 | A の版 `b`、`c` と、B の版 `a` がある | `findTargetsByContents`（A と B、`null`、1000）を呼ぶ | `a` と B、`b` と A、`c` と A の順に返す |
| TC-ports-revisionRepository-34 | A の版が 5 件ある。`id` は `a` < `b` < `c` < `d` < `e` | `findTargetsByContents`（A、`null`、2）と、`findTargetsByContents`（A、`b`、2）を呼ぶ | 前者は `a`、`b` の組、後者は `c`、`d` の組を返す |
| TC-ports-revisionRepository-36 | A の版が 5 件ある。`id` は `a` < `b` < `c` < `d` < `e` | `findTargetsByContents`（A、`e`、2）を呼ぶ | 空の列を返す |
| TC-ports-revisionRepository-37 | A の版 R1 がある | `contentIds` に A を 2 回渡す | R1 の組を 1 件だけ返す |
| TC-ports-revisionRepository-38 | A の版 R1 がある。ID が `x` のコンテンツも、その版も無い | `findTargetsByContents`（A と `x`、`null`、1000）を呼ぶ | R1 の組だけを返す |
| TC-ports-revisionRepository-39 | A の版 R1 がある | 空の `contentIds` で呼ぶ | 空の列を返す |
| TC-ports-revisionRepository-40 | `ContentBatch.maxContents` 件のコンテンツが、版を 1 件ずつ持つ | そのすべてのコンテンツID を渡す（`null`、上限の `limit`） | すべての版の組を返す |
| TC-ports-revisionRepository-41 | A の版 R1 がある | 重複の無い、`ContentBatch.maxContents` を 1 個超える数のコンテンツID を渡す | `SystemError`（`INVALID_QUERY`）になる |
| TC-ports-revisionRepository-42 | A の版が、上限より 1 件多くある | `findTargetsByContents`（A、`null`、上限の `limit`）を呼び、最後の `revisionId` を `after` にしてもう一度呼ぶ | 一度目は `id` の昇順で先頭から上限の件数、二度目は残りの 1 件を返す |
| TC-ports-revisionRepository-43 | A の版が 1 件ある | `findTargetsByContents`（A、`null`、上限より 1 大きい `limit`）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる |
| TC-ports-revisionRepository-44 | A の版が 1 件ある | `findTargetsByContents`（A、`null`、0）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる |

## findExpired と countExpired

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-revisionRepository-45 | 版が無い | `findExpired`（`t`、10）を呼ぶ | 空の列を返す |
| TC-ports-revisionRepository-46 | `retired.at` が `t` より前の R1、`t` と等しい R2、`t` より後の R3 がある | `findExpired`（`t`、10）を呼ぶ | R1 と R2 の組だけを返す |
| TC-ports-revisionRepository-48 | A の版 R1 と、B の版 R2 がある。どちらも `retired.at` が `t` より前 | `findExpired`（`t`、10）を呼ぶ | R1 と A、R2 と B の組を返す |
| TC-ports-revisionRepository-49 | `retired.at` が `t-1` の R1、`t-3` の R2、`t-2` の R3 がある | `findExpired`（`t`、10）を呼ぶ | R2、R3、R1 の順に返す |
| TC-ports-revisionRepository-50 | `retired.at` が同じで `t` より前の版が 3 件ある。`id` は `a` < `b` < `c` | `findExpired`（`t`、10）を呼ぶ | `a`、`b`、`c` の順に返す |
| TC-ports-revisionRepository-51 | `retired.at` が `t-3` の R1、`t-2` の R2、`t-1` の R3 がある | `findExpired`（`t`、2）を、間に書き込みを挟まずに 2 回呼ぶ | 2 回とも R1、R2 の組を返す |
| TC-ports-revisionRepository-53 | `retired.at` が `t` より前の版が 5 件ある | `findExpired`（`t`、2）を呼び、返った組の版の ID を別の UnitOfWork の `deleteMany` に渡すことを、返る件数が 2 より少なくなるまで繰り返す | 2 件、2 件、1 件の順に返す。どの `deleteMany` も成功する。`countExpired`（`t`）は 0 |
| TC-ports-revisionRepository-54 | `retired.at` が `t` より前の版が、`ContentBatch.maxRevisions` を 1 件超える件数ある | `findExpired`（`t`、`ContentBatch.maxRevisions`）を呼ぶ | `retired.at` の早い順で先頭の、`ContentBatch.maxRevisions` 件を返す |
| TC-ports-revisionRepository-55 | `retired.at` が `t` より前の版が 1 件ある | `limit` に `ContentBatch.maxRevisions` を 1 超える数を渡して `findExpired` を呼ぶ | `SystemError`（`INVALID_QUERY`）になる |
| TC-ports-revisionRepository-56 | `retired.at` が `t` より前の版が 1 件ある | `findExpired`（`t`、0）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる |
| TC-ports-revisionRepository-57 | 版が無い | `countExpired`（`t`）を呼ぶ | 0 を返す |
| TC-ports-revisionRepository-58 | `retired.at` が `t` より前の R1、`t` と等しい R2、`t` より後の R3 がある | `countExpired`（`t`）を呼ぶ | 2 を返す |

## deleteMany

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-revisionRepository-59 | A の版 R1、R2、R3 がある | `deleteMany`（R1 と R2）を呼ぶ | `findById`（R1、R2）は `null`。`findById`（R3）は R3 を返す。`findPage`（A）の `count` は 1 |
| TC-ports-revisionRepository-60 | A の版 R1 がある | 空の `ids` を `deleteMany` に渡す | 成功する。`findById`（R1）は R1 を返す |
| TC-ports-revisionRepository-61 | 公開中で下書きを持つ A と、A の版 R1 がある | `deleteMany`（R1）を呼ぶ | `ContentRepository.findById`（A）は、前提条件と同じ公開内容・下書き・`version` の A を返す |
| TC-ports-revisionRepository-62 | A の版 R1 がある。`id` が `r` の版は無い | `deleteMany`（R1 と `r`）を呼ぶ | `ConflictError`（`REVISION_MISSING`）になり、`r` を示す。`findById`（R1）は R1 を返す |
| TC-ports-revisionRepository-63 | A の版 R1 と、B の版 R2 がある | `deleteMany`（R1 と R2）を呼ぶ | 成功する。`findById`（R1、R2）は `null` を返す |
| TC-ports-revisionRepository-64 | A と T がある。A の版 R1 だけが T への必須の参照を持つ | `deleteMany`（R1）を確定し、別の UnitOfWork で T を `ContentRepository.deleteMany` に渡す | どちらも成功する。`deleteMany`（R1）の確定の後、`findInbound`（対象 T）は R1 の記録を返さない |
| TC-ports-revisionRepository-66 | A と B がある。A の版 R1 がある。B の下書きは A への必須の参照を持つ | `deleteMany`（R1）を呼ぶ | 成功する。`findById`（R1）は `null` を返す |
| TC-ports-revisionRepository-67 | A の版 R1、R2 がある | `deleteMany`（R1 を 2 回）を呼ぶ | 成功する。`findById`（R1）は `null`、`findById`（R2）は R2 を返す |
| TC-ports-revisionRepository-68 | A の版が `ContentBatch.maxRevisions` 件ある | そのすべての版を、一度の `deleteMany` に渡す | 成功する。`findPage`（A）の `count` は 0 |
| TC-ports-revisionRepository-69 | A の版が、`ContentBatch.maxRevisions` を 1 件超える件数ある | そのすべての版を、一度の `deleteMany` に渡す | `SystemError`（`INVALID_QUERY`）になる。`count` は、前提条件の件数 |

## count、lastId、scan

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-revisionRepository-70 | 版が無い | `count` を呼ぶ | 0 を返す |
| TC-ports-revisionRepository-71 | A の版が 2 件、B の版が 1 件ある | `count` を呼ぶ | 3 を返す |
| TC-ports-revisionRepository-72 | 版が無い | `lastId` を呼ぶ | `null` を返す |
| TC-ports-revisionRepository-73 | A の版 `b`、B の版 `c`、A の版 `a` を、この順に追加した | `lastId` を呼ぶ | `c` を返す |
| TC-ports-revisionRepository-74 | `id` が `a` < `b` の 2 件がある。`b` を `deleteMany` で削除した | `lastId` を呼ぶ | `a` を返す |
| TC-ports-revisionRepository-75 | `id` が `a` < `b` < `c` の 3 件がある | `lastId` と、`scan`（`null`、100）を呼ぶ | `lastId` は、`scan` が最後に返した版の `id` と等しい |
| TC-ports-revisionRepository-76 | 版が無い | `scan`（`null`、100）を呼ぶ | 空の列を返す |
| TC-ports-revisionRepository-77 | A の版 `b`、B の版 `a`、A の版 `c` がある | `scan`（`null`、100）を呼ぶ | `a`、`b`、`c` の順に返す |
| TC-ports-revisionRepository-78 | `id` が `a` < `b` < `c` < `d` < `e` の 5 件がある | `scan`（`null`、2）と、`scan`（`b`、2）を呼ぶ | 前者は `a`、`b` を、後者は `c`、`d` を返す |
| TC-ports-revisionRepository-80 | `id` が `a` < `b` < `c` < `d` < `e` の 5 件がある | `scan`（`e`、2）を呼ぶ | 空の列を返す |
| TC-ports-revisionRepository-81 | 上限より 1 件多くある | `scan`（`null`、上限の `limit`）を呼び、最後の `id` を `after` にしてもう一度呼ぶ | 一度目は `id` の昇順で先頭から上限の件数、二度目は残りの 1 件を返す |
| TC-ports-revisionRepository-83 | 1 件がある | `scan`（`null`、上限より 1 大きい `limit`）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる |
| TC-ports-revisionRepository-84 | 1 件がある | `scan`（`null`、0）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる |
| TC-ports-revisionRepository-85 | `W1` に版が 1 件ある | `PartitionReader.workspace(W0)` のリポジトリで、`findById`、`findByIds`、`findPage`、`findTargetsByContents`、`findExpired`、`countExpired`、`scan`、`count`、`lastId` を呼ぶ | `findById` と `lastId` は `null`、`findPage` は空の一覧で `count` が 0、`count` と `countExpired` は 0、ほかは空の列を返す |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-revisionRepository-86 | 版が無い | `deleteAll`（10）を呼ぶ | 0 を返す |
| TC-ports-revisionRepository-87 | 版が 3 件ある | `deleteAll`（10）を呼ぶ | 3 を返す。`count` は 0 |
| TC-ports-revisionRepository-88 | 版が 5 件ある | `deleteAll`（2）を、呼び出しごとに別の UnitOfWork で、返る件数が 2 より少なくなるまで繰り返す | 2、2、1 の順に返す。`count` は 0 |
| TC-ports-revisionRepository-89 | 版が 2 件ある | `deleteAll`（2）を呼び、別の UnitOfWork でもう一度呼ぶ | 2、0 の順に返す |
| TC-ports-revisionRepository-90 | A と T がある。A の下書きと、A の版 R1 は T への参照を持つ | `deleteAll`（10）を呼ぶ | `findInbound`（対象 T）は、`holder` が `draft` の記録だけを返す |
| TC-ports-revisionRepository-91 | A と、A の版が 2 件ある | `deleteAll`（10）を呼ぶ | 2 を返す。`ContentRepository.findById`（A）は A を返す |
| TC-ports-revisionRepository-92 | 版が 1 件ある | `deleteAll`（0）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる。`count` は 1 |
| TC-ports-revisionRepository-93 | 版が 1 件ある | `deleteAll`（上限より 1 大きい `limit`）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる。`count` は 1 |

## 並行性

二つの UnitOfWork を並行に実行する。どちらの `run` も、相手の結果を待たずに呼ぶ。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-revisionRepository-94 | A がある。`id` が `r` の版は無い | 二つの UnitOfWork が、それぞれ `id` が `r` の版を `insertMany` に渡す | 一方だけが確定する。他方は `ConflictError`（`ALREADY_EXISTS`）になる |
| TC-ports-revisionRepository-95 | A と T がある。T は参照されていない | 一方の UnitOfWork が、T への必須の参照を持つ A の版を `insertMany` に渡し、他方の UnitOfWork が T を `ContentRepository.deleteMany` に渡す | `insertMany` は確定する。`deleteMany` は、確定するか、`ConflictError`（`REFERENCED`）になる |
| TC-ports-revisionRepository-96 | A と T がある。T は参照されていない | 一方の UnitOfWork が、T への任意の参照を持つ A の版を `insertMany` に渡し、他方の UnitOfWork が T を `ContentRepository.deleteMany` に渡す | どちらも確定する |
| TC-ports-revisionRepository-97 | A の版 R1 がある | 二つの UnitOfWork が、それぞれ `deleteMany`（R1）を呼ぶ | 一方だけが確定する。他方は `ConflictError`（`REVISION_MISSING`）になり、R1 の `id` を示す。`findById`（R1）は `null` を返す |

## 可視性と UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-revisionRepository-98 | A がある。版は無い | `retired.at` が `t` より前の R1 を `insertMany` に渡して確定した直後に、`findById`、`findByIds`、`findPage`、`findTargetsByContents`、`findExpired`、`countExpired`、`scan`、`count` を呼ぶ | どの読み取りも R1 を反映する |
| TC-ports-revisionRepository-99 | A の版 R1 がある | 一つの UnitOfWork で、R2 の `insertMany` と `deleteMany`（R1）を呼んだ後、`fn` が例外を投げる | `run` はその例外を投げる。`findById`（R1）は R1、`findById`（R2）は `null` を返す |
| TC-ports-revisionRepository-100 | 版が 2 件ある | 一つの UnitOfWork で `deleteAll`（10）を呼んだ後、`fn` が例外を投げる | `count` は 2 |
| TC-ports-revisionRepository-101 | A の版 R1、R2 がある | A を `ContentRepository.deleteMany` に渡す | `findById`（R1、R2）は `null`。`count` は 0 |
| TC-ports-revisionRepository-102 | A と B がある。A の版 R1 がある。B の下書きは A への必須の参照を持つ | A を `ContentRepository.deleteMany` に渡す | `ConflictError`（`REFERENCED`）になる。`findById`（R1）は R1 を返す |
