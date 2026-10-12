# ContentStagingArea

「上限」は、その節のメソッドの契約が定める、一度に渡せる件数・個数、または `limit` の上限を指す。値は、ポートを定めるドメインのファイルが持つ。

契約は [content.md](../../domains/content.md) の「ContentMigration」「ポート」「参照の記録と保護」と、[common.md](../../domains/common.md) の「PublishedChangeNotice」・「TransactionalRepository」の共通の規則・「UnitOfWorkProvider」が定める。

## 記法

- ポートは、ワークスペース `W1` の区画に結び付く。メソッドは、ワークスペースの ID を引数に取らない。`J1`、`J2` は `SchemaApplyJobId`。`M1`、`M2` はモデルの ID。`W0` は、区画に何も確定していないワークスペース
- UnitOfWork は、`{ kind: "workspace", workspaceId: W1, condition: { kind: "none" } }` を渡した `UnitOfWorkProvider.run` を指す。書き込みは、その `WorkspaceContext` のリポジトリで呼ぶ。UnitOfWork の外の読み取りは、`PartitionReader.workspace(W1)` のリポジトリで呼ぶ
- A、B、C、T、U は区画のコンテンツで、`ContentRepository.insert` と `save` で保存する。R は版で、`RevisionRepository.insertMany` で保存する。「区画に…がある」は、区画のコンテンツと版が、挙げたものだけであることを指す
- 「X の結果」は、X を指す `StagedContent`（`StagedRevision`）。断りが無い結果は、`result` が X を変換した値で、`result` の ID が X を指す。「X の削除の結果」は、`result` が `null` で、`contentId`（版では `target`）が X を指す結果
- 断りが無い結果の報告は、`publication` が `null`。「`publication` が `updated` の A の結果」は、報告の `publication` の `change` が `updated` である、A の結果を指す。断りが無い結果は `M1` に属し、報告の `model` は `m`。`M2` に属する結果の報告の `model` は `n`
- 断りが無い `promote` は、`promote`（`J1`）
- 前提条件の書き込みは、UnitOfWork で呼び、確定済み。操作の書き込みは、新しい UnitOfWork で呼び、UnitOfWork を確定する。読み取りは、UnitOfWork の外で呼ぶ
- 「`ConflictError`（理由）になる」は、メソッドの呼び出しか UnitOfWork の確定のどちらかでエラーが現れ、`run` がそのエラーを投げ、その UnitOfWork の書き込みが 1 件も反映されないことを指す
- 「`SystemError`（`INVALID_QUERY`）になる」は、呼び出しがそのエラーを返すことを指す。書き込みでは、`run` がそのエラーを投げ、その UnitOfWork の書き込みが 1 件も反映されない
- ID の大小は [common.md](../../domains/common.md) の「TransactionalRepository」の順序に、文字列の大小は [content.md](../../domains/content.md) の「文字列の比較」に従う
- `slug` と `code` は一意な項目で、設定した値が内容の `uniqueValues` に入る。`title` は一意でない項目
- 断りが無い `findInbound` は、`ContentReferenceQuery.findInbound` を、`holders` が `published`・`draft`・`revision`、`strengths` が `required`・`optional`・`link`、`page` が 1、`limit` が 100 で呼ぶ

## putContents、putRevisions、cursor

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentStagingArea-01 | `J1` の準備領域が無い | `cursor`（`J1`）を呼ぶ | `contents` は 0、`revisions` は 0、`lastContentId` と `lastRevisionId` は `null` |
| TC-ports-contentStagingArea-02 | A がある。`J1` の準備領域に A の結果がある | 空の `items` を `putContents`（`J1`）に渡し、続けて空の `items` を `putRevisions`（`J1`）に渡す | どちらも成功する。`cursor`（`J1`）の `contents` は 1、`revisions` は 0 |
| TC-ports-contentStagingArea-03 | A、B がある。ID は A < B | A と B の結果を `putContents`（`J1`）に渡す | `cursor`（`J1`）の `contents` は 2、`lastContentId` は B、`revisions` は 0、`lastRevisionId` は `null` |
| TC-ports-contentStagingArea-04 | ID が `a` < `b` < `c` のコンテンツがある | `c`、`a`、`b` の順に並べた結果を `putContents` に渡す | `cursor` の `lastContentId` は `c` |
| TC-ports-contentStagingArea-05 | `J1` の準備領域に `a` の結果がある | `b` の結果を `putContents` に渡す | `cursor` の `contents` は 2、`lastContentId` は `b` |
| TC-ports-contentStagingArea-06 | `J1` の準備領域に A の結果がある。その報告の `draftHasIssues` は偽 | 報告の `draftHasIssues` が真の、A の結果を `putContents` に渡す | `cursor` の `contents` は 1。`listContentReports` が返す A の報告の `draftHasIssues` は真 |
| TC-ports-contentStagingArea-07 | A、B がある | A と B の結果を `putContents` に渡して確定し、同じ `items` をもう一度 `putContents` に渡す | `cursor` と `listContentReports` の結果は、一度目の後と同じ |
| TC-ports-contentStagingArea-08 | A がある | A の削除の結果（`result` が `null` で、`removedBy` を持つ `removed` の報告）を `putContents` に渡す | `cursor` の `contents` は 1。`listContentReports` が、その報告を返す |
| TC-ports-contentStagingArea-09 | 上限の件数のコンテンツがある | 上限の件数の結果を、一度の `putContents` に渡す | 成功する。`cursor` の `contents` は、上限の件数 |
| TC-ports-contentStagingArea-10 | 上限より 1 件多いコンテンツがある | 上限より 1 件多い結果を、一度の `putContents` に渡す | `SystemError`（`INVALID_QUERY`）になる。`cursor` の `contents` は 0 |
| TC-ports-contentStagingArea-11 | A の版 R1、R2 がある。`id` は R1 < R2 | R1 と R2 の結果を `putRevisions`（`J1`）に渡す | `cursor`（`J1`）の `revisions` は 2、`lastRevisionId` は R2、`contents` は 0、`lastContentId` は `null` |
| TC-ports-contentStagingArea-12 | `J1` の準備領域に R1 の結果がある。その報告の `losses` は空 | 報告の `losses` が空でない、R1 の結果を `putRevisions` に渡す | `cursor` の `revisions` は 1。`listRevisionReports` が返す R1 の報告の `losses` は、後に渡した値 |
| TC-ports-contentStagingArea-13 | R1、R2 がある | R1 と R2 の結果を `putRevisions` に渡して確定し、同じ `items` をもう一度 `putRevisions` に渡す | `cursor` と `listRevisionReports` の結果は、一度目の後と同じ |
| TC-ports-contentStagingArea-14 | 上限の件数の版がある | 上限の件数の結果を、一度の `putRevisions` に渡す | 成功する。`cursor` の `revisions` は、上限の件数 |
| TC-ports-contentStagingArea-15 | 上限より 1 件多い版がある | 上限より 1 件多い結果を、一度の `putRevisions` に渡す | `SystemError`（`INVALID_QUERY`）になる。`cursor` の `revisions` は 0 |
| TC-ports-contentStagingArea-16 | A と、A の版 R がある | 一つの UnitOfWork で、A の結果の `putContents` と、R の結果の `putRevisions` を呼ぶ | `cursor` の `contents` は 1、`revisions` は 1 |

## 準備領域の分離

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentStagingArea-17 | A、B がある。`J1` の準備領域に A の結果がある | B の結果を `putContents`（`J2`）に渡す | `cursor`（`J1`）の `contents` は 1 で `lastContentId` は A。`cursor`（`J2`）の `contents` は 1 で `lastContentId` は B |
| TC-ports-contentStagingArea-18 | `J1` と `J2` の準備領域に、報告が異なる A の結果がある | `listContentReports`（`J1`）と `listContentReports`（`J2`）を呼ぶ | それぞれ、自分のジョブに書いた報告を返す |
| TC-ports-contentStagingArea-19 | `J1` と `J2` の準備領域に A の結果がある | `discard`（`J1`）を呼ぶ | `cursor`（`J1`）の `contents` は 0。`cursor`（`J2`）の `contents` は 1 |
| TC-ports-contentStagingArea-20 | 未公開の A と B がある。A の下書きの `title` は `old` | 下書きの `title` が `new` の A の結果と、B の削除の結果を `putContents`（`J1`）に渡す | `ContentRepository.findById` は、`title` が `old` の A と、B を返す。`ContentRepository.findPage` と `scan` も、`old` の A と B を返す。`ContentRepository.count` は 2 |
| TC-ports-contentStagingArea-22 | A の版 R がある | `values` を変えた R の結果を `putRevisions`（`J1`）に渡す | `RevisionRepository.findById`（R）は、前提条件の R を返す |
| TC-ports-contentStagingArea-23 | A と T がある。A の下書きは参照を持たない | 下書きが T への参照を持つ A の結果を `putContents`（`J1`）に渡す | `findInbound`（対象 T）は空の一覧を返す |
| TC-ports-contentStagingArea-24 | 区画に A と B がある | 一方の UnitOfWork が A の結果を `putContents`（`J1`）に、他方の UnitOfWork が B の結果を `putContents`（`J2`）に、並行に渡す | どちらも確定する。`cursor`（`J1`）と `cursor`（`J2`）の `contents` は、どちらも 1 |
| TC-ports-contentStagingArea-25 | `W1` の `J1` の準備領域に、A の結果と、版 R の結果がある | `PartitionReader.workspace(W0)` のポートで、`cursor`（`J1`）、`listContentReports`（`J1`、`null`、上限の `limit`）、`listRevisionReports`（`J1`、`null`、上限の `limit`）、`findDuplicates`（`J1`、`M1`、項目）を呼ぶ | `cursor` は件数 0 と `null` を、ほかは空を返す |

## listContentReports

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentStagingArea-26 | `J1` の準備領域に、版の結果だけがある | `listContentReports`（`J1`、`null`、100）を呼ぶ | 空の列を返す |
| TC-ports-contentStagingArea-27 | `J1` の準備領域に、ID が `b`、`a`、`c` の順に書いた 3 件の結果がある | `listContentReports`（`J1`、`null`、100）を呼ぶ | `a`、`b`、`c` の順に、`contentId` が対象の ID で、ほかが書いた `report` と等しい報告を返す |
| TC-ports-contentStagingArea-28 | `J1` の準備領域に、ID が `a` < `b` < `c` < `d` < `e` の 5 件の結果がある | `listContentReports`（`J1`、`null`、2）と、`listContentReports`（`J1`、`b`、2）を呼ぶ | 前者は `a`、`b` の報告を、後者は `c`、`d` の報告を返す |
| TC-ports-contentStagingArea-30 | `J1` の準備領域に、ID が `a` < `b` < `c` < `d` < `e` の 5 件の結果がある | `listContentReports`（`J1`、`e`、2）を呼ぶ | 空の列を返す |
| TC-ports-contentStagingArea-31 | `J1` の準備領域に、`publication` が `null` の `a`、`updated` の `b`、`null` の `c`、`unpublished` の `d` の結果がある | `listContentReports`（`J1`、`null`、100）を呼ぶ | `a`、`b`、`c`、`d` の報告を返す |
| TC-ports-contentStagingArea-32 | `J1` の準備領域に、上限より 1 件多い結果がある | `listContentReports`（`J1`、`null`、上限の `limit`）を呼び、最後の ID を `after` にしてもう一度呼ぶ | 一度目は ID の昇順で先頭から上限の件数、二度目は残りの 1 件を返す |
| TC-ports-contentStagingArea-34 | `J1` の準備領域に、1 件の結果がある | `listContentReports`（`J1`、`null`、上限より 1 大きい `limit`）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる |
| TC-ports-contentStagingArea-35 | `J1` の準備領域に、1 件の結果がある | `listContentReports`（`J1`、`null`、0）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる |
| TC-ports-contentStagingArea-36 | `J1` の準備領域が無い | `listContentReports`（`J1`、`null`、上限の `limit`）を呼ぶ | 空の列を返す |

## listRevisionReports

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentStagingArea-37 | `J1` の準備領域に、コンテンツの結果だけがある | `listRevisionReports`（`J1`、`null`、100）を呼ぶ | 空の列を返す |
| TC-ports-contentStagingArea-38 | `J1` の準備領域に、`id` が `b`、`a`、`c` の順に書いた 3 件の版の結果がある | `listRevisionReports`（`J1`、`null`、100）を呼ぶ | `a`、`b`、`c` の順に、版の ID とコンテンツID が対象のもので、`losses` が書いた値と等しい報告を返す |
| TC-ports-contentStagingArea-39 | A の版 R1、R2 がある。`J1` の準備領域に、R1 の結果（`losses` は空でない）と、R2 の削除の結果（`result` は `null`）がある | `listRevisionReports`（`J1`、`null`、100）を呼ぶ | `id` の昇順に、`contentId` が A、`revisionId` がそれぞれの版の ID で、`losses` が書いた値と等しい報告を 2 件返す |
| TC-ports-contentStagingArea-40 | `J1` の準備領域に、`id` が `a` < `b` < `c` < `d` < `e` の 5 件の版の結果がある | `listRevisionReports`（`J1`、`null`、2）と、`listRevisionReports`（`J1`、`b`、2）を呼ぶ | 前者は `a`、`b` の報告を、後者は `c`、`d` の報告を返す |
| TC-ports-contentStagingArea-42 | `J1` の準備領域に、上限より 1 件多い版の結果がある | `listRevisionReports`（`J1`、`null`、上限の `limit`）を呼び、最後の `id` を `after` にしてもう一度呼ぶ | 一度目は `id` の昇順で先頭から上限の件数、二度目は残りの 1 件を返す |
| TC-ports-contentStagingArea-44 | `J1` の準備領域に、1 件の版の結果がある | `listRevisionReports`（`J1`、`null`、上限より 1 大きい `limit`）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる |
| TC-ports-contentStagingArea-45 | `J1` の準備領域に、1 件の版の結果がある | `listRevisionReports`（`J1`、`null`、0）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる |
| TC-ports-contentStagingArea-46 | `J1` の準備領域が無い | `listRevisionReports`（`J1`、`null`、上限の `limit`）を呼ぶ | 空の列を返す |

## findDuplicates

`field` は `slug`、`modelId` は `M1`。断りが無い結果は、`M1` に属する公開中のコンテンツ。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentStagingArea-47 | `J1` の準備領域に、公開内容の `slug` が `x` の A と、`y` の B の結果がある | `findDuplicates`（`J1`、`M1`、`slug`）を呼ぶ | `total` と `contentCount` は 0、`items` は空 |
| TC-ports-contentStagingArea-48 | `J1` の準備領域に、公開内容の `slug` が `x` の `a` と `b`、`y` の `c` の結果がある | `findDuplicates` を呼ぶ | `total` は 1、`contentCount` は 2。`items` は、`value` が `x`、`contents` が `a`、`b` の順の 1 組 |
| TC-ports-contentStagingArea-49 | `J1` の準備領域に、公開内容の `slug` が `x` の 3 件（`c`、`a`、`b` の順に書いた）の結果がある | `findDuplicates` を呼ぶ | `total` は 1、`contentCount` は 3。組の `contents` は `a`、`b`、`c` の順 |
| TC-ports-contentStagingArea-50 | `J1` の準備領域に、公開内容の `slug` が `x` の `a`、下書きの `slug` が `x` の未公開の `b`、公開内容の `slug` が `y` で下書きの `slug` が `x` の `c` の結果がある | `findDuplicates` を呼ぶ | `total` は 0、`items` は空 |
| TC-ports-contentStagingArea-52 | `J1` の準備領域に、公開内容の `slug` が `x` の `a` の結果と、`b` の削除の結果がある。保存済みの `b` の公開内容の `slug` は `x` | `findDuplicates` を呼ぶ | `total` は 0、`items` は空 |
| TC-ports-contentStagingArea-53 | `J1` の準備領域に、`M1` の `a` と、`M2` の `b` の結果がある。どちらも公開内容の `slug` は `x` | `findDuplicates`（`J1`、`M1`、`slug`）を呼ぶ | `total` は 0、`items` は空 |
| TC-ports-contentStagingArea-54 | `J1` の準備領域に `a` の結果、`J2` の準備領域に `b` の結果がある。どちらも公開内容の `slug` は `x` | `findDuplicates`（`J1`、`M1`、`slug`）を呼ぶ | `total` は 0、`items` は空 |
| TC-ports-contentStagingArea-55 | `J1` の準備領域に、公開内容の `slug` が `x` の `a` と、`X` の `b` の結果がある | `findDuplicates` を呼ぶ | `total` は 0、`items` は空 |
| TC-ports-contentStagingArea-56 | `J1` の準備領域に、公開内容の `slug` が未設定の `a` と `b` の結果がある | `findDuplicates` を呼ぶ | `total` は 0、`items` は空 |
| TC-ports-contentStagingArea-57 | `J1` の準備領域が無い | `findDuplicates` を呼ぶ | `total` は 0、`items` は空 |
| TC-ports-contentStagingArea-58 | `J1` の準備領域に、公開内容の `slug` が `x` で `code` が異なる `a` と `b` の結果がある | `findDuplicates`（`J1`、`M1`、`code`）を呼ぶ | `total` は 0、`items` は空 |
| TC-ports-contentStagingArea-59 | `J1` の準備領域に、公開内容の `title` が `x` の `a` と `b` の結果がある | `findDuplicates`（`J1`、`M1`、`title`）を呼ぶ | `total` は 0、`items` は空 |
| TC-ports-contentStagingArea-60 | `J1` の準備領域に、`slug` が `x` の `c` と `d`、`y` の `a` と `e` の結果がある | `findDuplicates` を呼ぶ | `total` は 2、`contentCount` は 4。`items` は、`y` の組（最小の ID は `a`）、`x` の組（最小の ID は `c`）の順 |
| TC-ports-contentStagingArea-61 | `J1` の準備領域に、同じ値を 2 件ずつが持つ組が、`items` の上限の数だけある | `findDuplicates` を呼ぶ | `total` と、`items` の組の数は、どちらも上限の数 |
| TC-ports-contentStagingArea-62 | `J1` の準備領域に、同じ値を 2 件ずつが持つ組が、上限より 1 組多くある | `findDuplicates` を呼ぶ | `total` は、上限より 1 多い数。`contentCount` は、その 2 倍。`items` は、組の中で最小のコンテンツID の昇順で、先頭から上限の数の組 |
| TC-ports-contentStagingArea-63 | `J1` の準備領域に、公開内容の `slug` が `x` の上限より 1 件多い結果がある | `findDuplicates` を呼ぶ | `total` は 1、`contentCount` は上限より 1 多い数。組の `contents` は、ID の昇順で先頭から上限の件数 |

## promote: 切り替え

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentStagingArea-64 | 区画に、下書きの `title` が `old` の A と、A の版 R がある。`J1` の準備領域に、下書きの `title` が `new` の A の結果と、`values` を変えた R の結果がある | `promote` を呼ぶ | `ContentRepository.findById`（A）は、`result` と等しい A を返す。`RevisionRepository.findById`（R）は、`result` と等しい R を返す |
| TC-ports-contentStagingArea-65 | 区画に A と B、B の版 R がある。`J1` の準備領域に、A の結果、B の削除の結果、R の削除の結果がある | `promote` を呼ぶ | `ContentRepository.findById`（B）と `RevisionRepository.findById`（R）は `null`。`ContentRepository.count` は 1、`RevisionRepository.count` は 0 |
| TC-ports-contentStagingArea-66 | 区画に、公開中で下書きを持つ A と、ごみ箱の B がある。`J1` の準備領域に、A と B の結果がある | `promote` を呼ぶ | `ContentRepository.findById` が、A と B のそれぞれについて、`result` と等しい公開内容・下書き・置き場所・`version` のコンテンツを返す |
| TC-ports-contentStagingArea-67 | 区画に A、T、U がある。A の下書きは T への必須の参照を持つ。`J1` の準備領域に、T と U の結果と、下書きが U への任意の参照を持つ A の結果がある | `promote` を呼ぶ | `findInbound`（対象 T）は空の一覧を返す。`findInbound`（対象 U）は、参照元 A、`holder` が `draft`、`strength` が `optional` の記録を返す |
| TC-ports-contentStagingArea-68 | 区画に A、T と、A の版 R がある。R は T への必須の参照を持つ。`J1` の準備領域に、A と T の結果と、参照を持たない R の結果がある | `promote` を確定し、別の UnitOfWork で T を `ContentRepository.deleteMany` に渡す | どちらも成功する |
| TC-ports-contentStagingArea-69 | 区画に A、B がある。`J1` の準備領域に A と B の結果を書いた後、A の削除の結果を `putContents` に渡して置き換えた | `promote` を呼ぶ | 成功する。`ContentRepository.findById`（A）は `null`、`findById`（B）は `result` と等しい B を返す |
| TC-ports-contentStagingArea-70 | 区画に A と、A の版 R がある。`J1` の準備領域に、A と R の結果がある | `promote` を呼ぶ | 確定の後、`cursor`（`J1`）の `contents` と `revisions` は 0、`lastContentId` と `lastRevisionId` は `null` |
| TC-ports-contentStagingArea-71 | 区画に A がある。`J1` と `J2` の準備領域に、A の結果がある | `promote` を呼ぶ | `cursor`（`J2`）の `contents` は 1 |
| TC-ports-contentStagingArea-72 | 区画にコンテンツも版も無い。`J1` の準備領域に結果を書いていない | `promote` を呼ぶ | 成功する。`ContentRepository.count` と `RevisionRepository.count` は 0 |
| TC-ports-contentStagingArea-73 | 区画に A がある。`J1` の準備領域に A の結果がある | `promote` を確定した後、`ContentRepository.findById`（A）の `expectedVersion` で、A を変えた内容を `save` する | 成功する |
| TC-ports-contentStagingArea-74 | 区画に、同じモデルの公開中の A（公開内容の `slug` は `x`）と未公開の B がある。`J1` の準備領域に、公開内容の `slug` を `z` にした A の結果と、B の結果がある | `promote` を確定した後、`ContentRepository.findById`（B）の `expectedVersion` で、下書きの `slug` を `z` にして公開した B を `ContentRepository.save` する。続けて、同じ `expectedVersion` で、下書きの `slug` を `x` にして公開した B を `save` する | 一度目は `ConflictError`（`UNIQUE_VALUE_TAKEN`）になる。二度目は成功し、`ContentRepository.findUniqueHolders`（`slug` の `x`）は B の組だけを返す |

## promote: 整合性

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentStagingArea-76 | 区画に A、B がある。`J1` の準備領域に A の結果だけがある | `promote` を呼ぶ | `ConflictError`（`STAGING_INCOMPLETE`）になる。`ContentRepository.findById`（A、B）は、前提条件の A と B を返す。`cursor`（`J1`）の `contents` は 1 |
| TC-ports-contentStagingArea-77 | 区画に A と、A の版 R1、R2 がある。`J1` の準備領域に、A と R1 の結果だけがある | `promote` を呼ぶ | `ConflictError`（`STAGING_INCOMPLETE`）になる。`RevisionRepository.findById`（R1）は、前提条件の R1 を返す |
| TC-ports-contentStagingArea-78 | 区画に A がある。`J1` の準備領域に、A の結果と、区画に無いコンテンツの結果がある | `promote` を呼ぶ | `ConflictError`（`STAGING_INCOMPLETE`）になる。`ContentRepository.findById`（A）は、前提条件の A を返す |
| TC-ports-contentStagingArea-79 | 区画に A がある。`J1` の準備領域に結果を書いていない | `promote` を呼ぶ | `ConflictError`（`STAGING_INCOMPLETE`）になる |
| TC-ports-contentStagingArea-81 | 区画に A、B がある。`J1` の準備領域に、下書きが B への必須の参照、本文のリンク（`link`）、任意の参照を持つ未公開の A の結果と、B の削除の結果がある | `promote` を呼ぶ | 成功する。`ContentRepository.findById`（B）は `null`、`findById`（A）は、それらの参照を持つ `result` と等しい A を返す |
| TC-ports-contentStagingArea-83 | 区画に A、B と、A の版 R がある。`J1` の準備領域に、A の結果、B への必須の参照を持つ R の結果、B の削除の結果がある | `promote` を呼ぶ | 成功する。`RevisionRepository.findById`（R）は `result` と等しい R を返す |
| TC-ports-contentStagingArea-84 | 区画に A がある。ID が `t` のコンテンツは無い。`J1` の準備領域に、下書きが `t` への必須の参照を持つ A の結果がある | `promote` を呼ぶ | 成功する |
| TC-ports-contentStagingArea-86 | 区画に A、B がある。`J1` の準備領域に、下書きが B への必須の参照を持つ A の結果と、B の結果がある | `promote` を呼ぶ | 成功する |
| TC-ports-contentStagingArea-87 | 区画に A、B がある。`J1` の準備領域に、下書きが B への必須の参照を持つ A の削除の結果と、B の削除の結果がある | `promote` を呼ぶ | 成功する。`ContentRepository.count` は 0 |
| TC-ports-contentStagingArea-88 | 区画に A、B がある。`J1` の準備領域に、公開内容が B への必須の参照を持つ公開中の A の結果と、未公開の B の結果がある | `promote` を呼ぶ | `ConflictError`（`REFERENCE_NOT_PUBLISHED`）になる。`ContentRepository.findById`（A、B）は、前提条件の A と B を返す |
| TC-ports-contentStagingArea-89 | 区画に A、B がある。`J1` の準備領域に、公開内容が B への本文のリンク（`link`）を持つ公開中の A の結果と、ごみ箱の B の結果がある | `promote` を呼ぶ | `ConflictError`（`REFERENCE_NOT_PUBLISHED`）になる |
| TC-ports-contentStagingArea-90 | 区画に A、B がある。`J1` の準備領域に、公開内容が B への必須の参照を持つ公開中の A の結果と、B の削除の結果がある | `promote` を呼ぶ | `ConflictError`（`REFERENCE_NOT_PUBLISHED`）になる。`ContentRepository.findById`（A、B）は、前提条件の A と B を返す。`cursor`（`J1`）の `contents` は 2 |
| TC-ports-contentStagingArea-91 | 区画に A、B がある。`J1` の準備領域に、公開内容が B への必須の参照を持つ公開中の A の結果と、公開中の B の結果がある | `promote` を呼ぶ | 成功する |
| TC-ports-contentStagingArea-92 | 区画に A、B がある。`J1` の準備領域に、公開内容が B への任意の参照を持つ公開中の A の結果と、未公開の B の結果がある | `promote` を呼ぶ | 成功する |
| TC-ports-contentStagingArea-93 | 区画に、同じモデルの A、B がある。`J1` の準備領域に、どちらも公開内容の `slug` が `x` の、公開中の A と B の結果がある | `promote` を呼ぶ | `ConflictError`（`UNIQUE_VALUE_TAKEN`）になる。`ContentRepository.findById`（A、B）は、前提条件の A と B を返す。`cursor`（`J1`）の `contents` は 2 |
| TC-ports-contentStagingArea-94 | 区画に、`M1` の A と `M2` の B がある。`J1` の準備領域に、どちらも公開内容の `slug` が `x` の、公開中の A と B の結果がある | `promote` を呼ぶ | 成功する |
| TC-ports-contentStagingArea-95 | 区画に、同じモデルの A、B がある。`J1` の準備領域に、公開内容の `slug` が `x` の公開中の A の結果と、下書きの `slug` が `x` の未公開の B の結果がある | `promote` を呼ぶ | 成功する |

## promote: UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentStagingArea-96 | 区画に、下書きの `title` が `old` の A と、A の版 R がある。`J1` の準備領域に、A と R の結果がある | `promote` を確定した直後に、`ContentRepository.findById`、`findPage`、`scan`、`RevisionRepository.findById`、`ContentReferenceQuery.findInbound` を呼ぶ | どの読み取りも、変換結果を返す |
| TC-ports-contentStagingArea-97 | 区画に、下書きの `title` が `old` の A がある。`J1` の準備領域に、`title` が `new` の A の結果がある | UnitOfWork の中で `promote` を呼んだ後、`fn` が例外を投げる | `run` はその例外を投げる。`ContentRepository.findById`（A）は `old` の A を返す。`cursor`（`J1`）の `contents` は 1 |
| TC-ports-contentStagingArea-98 | 区画に A がある。`J1` の準備領域に A の結果がある。`promote` を呼んだ UnitOfWork が、`fn` の例外で確定しなかった | 新しい UnitOfWork で `promote` を呼ぶ | 成功する。`ContentRepository.findById`（A）は `result` と等しい A を返す |

## discard

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentStagingArea-99 | `J1` の準備領域に、同じ `slug` の公開内容を持つ A と B の結果と、A の版 R の結果がある | `discard`（`J1`）を呼ぶ | 確定の後、`cursor`（`J1`）の `contents` と `revisions` は 0、`lastContentId` と `lastRevisionId` は `null`。`listContentReports` と `listRevisionReports` は空の列、`findDuplicates` は `total` が 0 で空の `items` を返す |
| TC-ports-contentStagingArea-100 | `J1` の準備領域が無い | `discard`（`J1`）を、別の UnitOfWork で 2 回呼ぶ | 2 回とも成功する |
| TC-ports-contentStagingArea-102 | 区画に、下書きの `title` が `old` の A がある。`J1` の準備領域に、`title` が `new` の A の結果がある | `discard`（`J1`）を呼ぶ | `ContentRepository.findById`（A）は `old` の A を返す |
| TC-ports-contentStagingArea-103 | `J1` の準備領域に A の結果がある | UnitOfWork の中で `discard`（`J1`）を呼んだ後、`fn` が例外を投げる | `cursor`（`J1`）の `contents` は 1。`listContentReports`（`J1`）は A の報告を返す |
| TC-ports-contentStagingArea-105 | `J1` の準備領域に A の結果があり、`discard`（`J1`）を確定した | B の結果を `putContents`（`J1`）に渡す | 成功する。`cursor`（`J1`）の `contents` は 1、`lastContentId` は B。`listContentReports`（`J1`）は B の報告だけを返す |
| TC-ports-contentStagingArea-106 | 区画に A がある。`J1` の準備領域に A の結果があり、`discard`（`J1`）を確定した | `promote` を呼ぶ | `ConflictError`（`STAGING_INCOMPLETE`）になる |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentStagingArea-107 | 準備領域が無い | `deleteAll`（10）を呼ぶ | 0 を返す |
| TC-ports-contentStagingArea-108 | `J1` の準備領域にコンテンツの結果が 2 件と版の結果が 1 件、`J2` の準備領域にコンテンツの結果が 2 件ある | `deleteAll`（10）を呼ぶ | 5 を返す。`cursor`（`J1`）と `cursor`（`J2`）の `contents` と `revisions` は 0。`listContentReports` と `listRevisionReports` は空の列を返す |
| TC-ports-contentStagingArea-109 | `J1` の準備領域にコンテンツの結果が 2 件と版の結果が 1 件、`J2` の準備領域にコンテンツの結果が 2 件ある | `deleteAll`（2）を、呼び出しごとに別の UnitOfWork で、返る件数が 2 より少なくなるまで繰り返す | 2、2、1 の順に返す。`cursor`（`J1`）と `cursor`（`J2`）の `contents` と `revisions` は 0 |
| TC-ports-contentStagingArea-111 | `J1` の準備領域に、A の削除の結果が 1 件ある | `deleteAll`（10）を呼ぶ | 1 を返す |
| TC-ports-contentStagingArea-112 | 区画に A がある。`J1` の準備領域に A の結果がある | `deleteAll`（10）を呼ぶ | `ContentRepository.findById`（A）は、前提条件の A を返す |
| TC-ports-contentStagingArea-113 | `J1` の準備領域にコンテンツの結果が 1 件ある | UnitOfWork の中で `deleteAll`（10）を呼んだ後、`fn` が例外を投げる | `cursor`（`J1`）の `contents` は 1 |
| TC-ports-contentStagingArea-114 | `J1` の準備領域にコンテンツの結果が 1 件ある | `deleteAll`（0）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる。`cursor`（`J1`）の `contents` は 1 |
| TC-ports-contentStagingArea-115 | `J1` の準備領域にコンテンツの結果が 1 件ある | `deleteAll`（上限より 1 大きい `limit`）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる。`cursor`（`J1`）の `contents` は 1 |

## 可視性と UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentStagingArea-116 | A、B がある。`J1` の準備領域が無い | A の結果を `putContents` に渡して確定した直後に、`cursor`、`listContentReports`、`findDuplicates` を呼ぶ | どの読み取りも、A の結果を反映する |
| TC-ports-contentStagingArea-117 | `J1` の準備領域に A の結果がある。B と、B の版 R がある | 一つの UnitOfWork で、B の結果の `putContents` と、R の結果の `putRevisions` を呼んだ後、`fn` が例外を投げる | `run` はその例外を投げる。`cursor`（`J1`）の `contents` は 1、`lastContentId` は A、`revisions` は 0 |
| TC-ports-contentStagingArea-119 | A と B がある。`J1` の準備領域が無い。B の `expectedVersion` を読んだ後に、B を `save` した | 一つの UnitOfWork で、A の結果の `putContents` と、先に読んだ `expectedVersion` での B の `ContentRepository.save` を呼ぶ | `ConflictError`（`STALE_VERSION`）になる。`cursor`（`J1`）の `contents` は 0 |
