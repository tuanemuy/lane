# ContentReferenceQuery

「上限」は、その節のメソッドの契約が定める、一度に渡せる件数・個数、または `limit` の上限を指す。値は、ポートを定めるドメインのファイルが持つ。

契約は [content.md](../../domains/content.md) の「ポート」「参照の記録と保護」と、[common.md](../../domains/common.md) の「TransactionalRepository」の共通の規則が定める。記録は、`ContentRepository`、`RevisionRepository`、`ContentStagingArea` の書き込みで作る。

## 記法

- `ContentReferenceQuery` は、ワークスペース `W1` の区画に結び付き、`PartitionReader.workspace(W1)` から得る。`findInbound` は、ワークスペースの ID を引数に取らない。`W0` は、区画に何も確定していないワークスペース
- T、U は参照先、A、B、C は参照元のコンテンツ
- UnitOfWork は、`{ kind: "workspace", workspaceId: W1, condition: { kind: "none" } }` を渡した `UnitOfWorkProvider.run` を指す。書き込みは、その `WorkspaceContext` のリポジトリで呼ぶ
- 前提条件のコンテンツは `ContentRepository.insert` と `save`、版は `RevisionRepository.insertMany` で保存する。前提条件の書き込みは、UnitOfWork で呼び、確定済み
- 操作の書き込みは、新しい UnitOfWork で呼び、UnitOfWork を確定する。読み取りは、UnitOfWork の外で呼ぶ
- 断りが無い `findInbound` は、`holders` が `published`・`draft`・`revision`、`strengths` が `required`・`optional`・`link`、`page` は 1、`limit` は 100
- ID の大小は [common.md](../../domains/common.md) の「TransactionalRepository」の順序に従う
- 参照先として挙げたコンテンツは、断りが無い限り存在する
- 公開内容からの参照は、断りが無い限り任意の参照。公開内容からの必須の参照と本文のリンクの参照先は、公開中のコンテンツ
- `targets` は、コンテンツID の集合
- 準備領域の結果は、`result` の ID がそのコンテンツを指す `StagedContent` で、報告の `publication` は `null`

## findInbound: 記録の内容

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentReferenceQuery-01 | T がある。T を参照する内容は無い | `findInbound`（対象 T）を呼ぶ | 空の一覧を返す。`count` は 0 |
| TC-ports-contentReferenceQuery-02 | 公開中の T がある。公開中の A（モデル `M1`）の公開内容は、項目 `ref` に T への必須の参照を持つ | `findInbound`（対象 T）を呼ぶ | 1 件を返す。`target` は T、`source.contentId` は A、`source.modelId` は `M1`、`source.location` は `active`、`source.holder` は `published`、`strength` は `required`、`path` は `ref` |
| TC-ports-contentReferenceQuery-03 | T がある。未公開の A の下書きは T への任意の参照を持つ | `findInbound`（対象 T）を呼ぶ | 1 件を返す。`source.holder` は `draft`、`strength` は `optional` |
| TC-ports-contentReferenceQuery-04 | T がある。A の下書きは、リッチテキストに T へのリンクを持つ | `findInbound`（対象 T）を呼ぶ | 1 件を返す。`strength` は `link` |
| TC-ports-contentReferenceQuery-05 | T がある。通常の管理対象の A の版 R は T への必須の参照を持つ。A の公開内容と下書きは T を参照しない | `findInbound`（対象 T）を呼ぶ | 1 件を返す。`source.contentId` は A、`source.holder` は `revision` で `revisionId` は R、`source.location` は `active` |
| TC-ports-contentReferenceQuery-06 | T がある。ごみ箱の A の下書きは T への必須の参照を持つ | `findInbound`（対象 T）を呼ぶ | 1 件を返す。`source.holder` は `draft`、`source.location` は `trashed` |
| TC-ports-contentReferenceQuery-07 | T がある。ごみ箱の A の版 R は T への必須の参照を持つ | `findInbound`（対象 T）を呼ぶ | 1 件を返す。`source.holder` は `revision` で `revisionId` は R、`source.location` は `trashed` |
| TC-ports-contentReferenceQuery-09 | T がある。A の下書きは、複数件の参照の項目 `refs` に T と U を持つ | `findInbound`（対象 T）を呼ぶ | 1 件を返す。`target` は T、`path` は `refs` と T の位置 |
| TC-ports-contentReferenceQuery-10 | T がある。A の下書きは、項目 `ref1` と `ref2` に T への参照を持つ | `findInbound`（対象 T）を呼ぶ | `path` が `ref1` と `ref2` の 2 件を返す。`count` は 2 |
| TC-ports-contentReferenceQuery-11 | 公開中の T がある。公開中の A は、公開内容と下書きの両方に T への必須の参照を持つ | `findInbound`（対象 T）を呼ぶ | `source.holder` が `published` の記録、`draft` の記録の順に、2 件を返す |
| TC-ports-contentReferenceQuery-12 | A の下書きは A 自身への必須の参照を持つ | `findInbound`（対象 A）を呼ぶ | `source.contentId` と `target` が A の記録を返す |
| TC-ports-contentReferenceQuery-13 | T と U がある。T の下書きは U への必須の参照を持つ | `findInbound`（対象 T と U）を呼ぶ | 参照元 T、`target` が U の記録を返す |
| TC-ports-contentReferenceQuery-14 | T と U がある。A の下書きは T へ、B の下書きは U へ、C の下書きは別のコンテンツへの参照を持つ | `findInbound`（対象 T と U）を呼ぶ | A と B の記録を返す。C の記録を返さない。`count` は 2 |
| TC-ports-contentReferenceQuery-15 | ID が `t` のコンテンツは無い。A の下書きは `t` への任意の参照を、B の下書きは `t` への必須の参照を持つ | `findInbound`（対象 `t`）を呼ぶ | 参照元 A で `strength` が `optional` の記録と、参照元 B で `strength` が `required` の記録を返す。どちらも `holder` は `draft` |

## findInbound: 絞り込み

公開中の T を、A の公開内容が必須の参照で、B の下書きが任意の参照で、C の版 R が `link` で参照する。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentReferenceQuery-17 | 上記 | `holders` を `published`、`draft`、`revision` の一つずつにして、3 回呼ぶ | それぞれ、A の記録だけ、B の記録だけ、C の版 R の記録だけを返す。`count` はどれも 1 |
| TC-ports-contentReferenceQuery-20 | 上記 | `holders` が `published` と `revision` | A と C の記録を返す。`count` は 2 |
| TC-ports-contentReferenceQuery-21 | 上記 | `strengths` を `required`、`optional` の一つずつにして、2 回呼ぶ | それぞれ、A の記録だけ、B の記録だけを返す |
| TC-ports-contentReferenceQuery-23 | 上記 | `strengths` が `required` と `link` | A と C の記録を返す |
| TC-ports-contentReferenceQuery-24 | 上記 | `holders` が `draft`、`strengths` が `required` | 空の一覧を返す。`count` は 0 |
| TC-ports-contentReferenceQuery-25 | 上記 | `holders` を空にして呼び、`strengths` を空にして呼ぶ | どちらも空の一覧を返す。`count` は 0 |

## findInbound: 順序とページ分割

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentReferenceQuery-27 | T を、ID が `a` < `b` < `c` の 3 件のコンテンツの下書きが、それぞれ参照する | `findInbound`（対象 T）を呼ぶ | 参照元が `a`、`b`、`c` の順に返す |
| TC-ports-contentReferenceQuery-28 | T を、A の公開内容、A の下書き、A の版 R が、それぞれ参照する | `findInbound`（対象 T）を呼ぶ | `holder` が `published`、`draft`、`revision` の順に返す |
| TC-ports-contentReferenceQuery-29 | T を、`a` の版、`b` の公開内容が参照する。ID は `a` < `b` | `findInbound`（対象 T）を呼ぶ | `a` の版の記録、`b` の公開内容の記録の順に返す |
| TC-ports-contentReferenceQuery-30 | T を、A の版 3 件が参照する。`revisionId` は `r1` < `r2` < `r3`。`retired.at` は `r3` が最も古い | `findInbound`（対象 T）を呼ぶ | `r1`、`r2`、`r3` の順に返す |
| TC-ports-contentReferenceQuery-31 | T を、A の下書きの項目 `b`、`a`、`c` が参照する | `findInbound`（対象 T）を呼ぶ | `path` が `a`、`b`、`c` の順に返す |
| TC-ports-contentReferenceQuery-32 | T と U がある。A の下書きは、複数件の参照の項目 `refs` の位置 2 に T、位置 10 に U を持つ | `findInbound`（対象 T と U）を呼ぶ | 位置 10 の記録、位置 2 の記録の順に返す |
| TC-ports-contentReferenceQuery-33 | T を、A の下書きの、項目のまとまり `a` の中の項目 `z` と、直下の項目 `ab` が参照する | `findInbound`（対象 T）を呼ぶ | `a` の中の `z` の記録、`ab` の記録の順に返す |
| TC-ports-contentReferenceQuery-34 | T と U がある。ID は T < U。A の下書きは、リッチテキストの項目 `body` に、U へのリンクと T へのリンクを持つ | `findInbound`（対象 T と U）を呼ぶ | `target` が T の記録、U の記録の順に返す |
| TC-ports-contentReferenceQuery-35 | T を、A の下書きの項目 `ref1`、`ref2`、`ref3` が参照する | `page` 2、`limit` 2 | `path` が `ref3` の記録だけを返す。`count` は 3 |
| TC-ports-contentReferenceQuery-36 | T を、別々のコンテンツの下書き 1 件が参照する | `page` 1、`limit` 1 | その 1 件を返す。`count` は 1 |
| TC-ports-contentReferenceQuery-38 | T を、別々のコンテンツの下書き 5 件が参照する | `page` 4、`limit` 2 | 空の一覧を返す。`count` は 5 |
| TC-ports-contentReferenceQuery-39 | 上限の件数の対象のそれぞれを、別々のコンテンツの下書きが参照する | 上限の件数の対象で `findInbound` を呼ぶ | 上限の件数を返す。`count` は、上限の件数 |
| TC-ports-contentReferenceQuery-40 | 上限より 1 件多いコンテンツがある | 重複の無い上限より 1 個多い `targets` で `findInbound` を呼ぶ | `SystemError`（`INVALID_QUERY`）になる |
| TC-ports-contentReferenceQuery-41 | T を、A の下書きが参照する | `targets` に T の ID を 2 回渡す | A の記録を 1 件だけ返す。`count` は 1 |
| TC-ports-contentReferenceQuery-42 | 上限の件数の対象のそれぞれを、別々のコンテンツの下書きが参照する | 上限の個数の ID に、そのうち 1 個の重複を加えた上限より 1 個多い `targets` で呼ぶ | 上限の件数を返す。`count` は、上限の件数 |
| TC-ports-contentReferenceQuery-43 | T を、A の下書きが参照する。ID が `x` のコンテンツは無く、`x` を参照する内容も無い | `targets` に T と `x` を渡す | A の記録だけを返す |
| TC-ports-contentReferenceQuery-44 | T を、A の下書きが参照する | `PartitionReader.workspace(W0)` の `ContentReferenceQuery` で、`findInbound`（対象 T）を呼ぶ | 空の一覧を返す。`count` は 0 |

## findInbound: 書き込みの反映

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentReferenceQuery-45 | T と U がある。A の下書きは T への参照を持つ | 下書きを U への参照を持つ内容に置き換えた A を `save` する | `findInbound`（対象 T）は空。`findInbound`（対象 U）は A の `draft` の記録を返す |
| TC-ports-contentReferenceQuery-46 | 公開中の T がある。未公開の A の下書きは T への必須の参照を持つ | 公開した A を `save` する | `findInbound`（対象 T）は、`holder` が `published` の 1 件だけを返す |
| TC-ports-contentReferenceQuery-47 | T がある。公開中の A の公開内容は参照を持たず、下書きは T への参照を持つ | 下書きを破棄した A を `save` する | `findInbound`（対象 T）は空の一覧を返す |
| TC-ports-contentReferenceQuery-48 | T がある。通常の管理対象の未公開の A の下書きと、A の版 R は T への参照を持つ | ごみ箱へ移動した A を `save` する | `findInbound`（対象 T）の、A の下書きの記録と R の記録の `source.location` は、どちらも `trashed` |
| TC-ports-contentReferenceQuery-49 | T がある。ごみ箱の A の下書きと、A の版 R は T への参照を持つ | 復元した A を `save` する | `findInbound`（対象 T）の、A の下書きの記録と R の記録の `source.location` は、どちらも `active` |
| TC-ports-contentReferenceQuery-50 | T がある。A の公開内容、下書き、版 R は T への参照を持つ | A を `ContentRepository.deleteMany` に渡す | `findInbound`（対象 T）は空の一覧を返す |
| TC-ports-contentReferenceQuery-51 | T がある。A の版 R1、R2 は T への参照を持つ | `RevisionRepository.deleteMany`（R1）を呼ぶ | `findInbound`（対象 T）は、R2 の記録だけを返す |
| TC-ports-contentReferenceQuery-52 | T がある。A の下書きは T への参照を持つ | `ContentRepository.deleteAll`（100）を呼ぶ | `findInbound`（対象 T）は空の一覧を返す |
| TC-ports-contentReferenceQuery-53 | T がある。A の下書きと、A の版 R は T への参照を持つ | `RevisionRepository.deleteAll`（100）を呼ぶ | `findInbound`（対象 T）は、`holder` が `draft` の記録だけを返す |
| TC-ports-contentReferenceQuery-56 | T と A がある | UnitOfWork の中で、下書きを T への参照を持つ内容に置き換えた A を `save` した後、`fn` が例外を投げる | `findInbound`（対象 T）は空の一覧を返す |
| TC-ports-contentReferenceQuery-57 | T と A がある。A の下書きは参照を持たない。ジョブ `J` の準備領域に、下書きが T への参照を持つ A の変換結果を `ContentStagingArea.putContents` で書いた | `findInbound`（対象 T）を呼ぶ | 空の一覧を返す |
| TC-ports-contentReferenceQuery-58 | 区画のコンテンツは T と A だけで、版は無い。A の下書きは T への必須の参照を持つ。ジョブ `J` の準備領域に、T の変換結果と、下書きが参照を持たない A の変換結果がある | `ContentStagingArea.promote`（`J`）を確定する | `findInbound`（対象 T）は空の一覧を返す |
| TC-ports-contentReferenceQuery-59 | 区画のコンテンツは T と A だけで、版は無い。A の下書きは参照を持たない。ジョブ `J` の準備領域に、T の変換結果と、下書きが T への任意の参照を持つ A の変換結果がある | `ContentStagingArea.promote`（`J`）を確定する | `findInbound`（対象 T）は、A の `draft`、`strength` が `optional` の記録を返す |
