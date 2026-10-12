# ContentRepository

「上限」は、その節のメソッドの契約が定める、一度に渡せる件数・個数、または `limit` の上限を指す。値は、ポートを定めるドメインのファイルが持つ。

契約は [content.md](../../domains/content.md) の「ポート」「一覧の条件」「一度に扱う対象」「取得と検索の規則」「参照の記録と保護」と、[common.md](../../domains/common.md) の `TransactionalRepository`・`UnitOfWorkProvider` が定める。

## 記法

- `W` と `W2` はワークスペース、`M1`、`M2` はモデルの ID。断りが無いコンテンツは、`W`・`M1` の複数件モデルの、通常の管理対象にある未公開のコンテンツ。`W0` は、何も書き込んでいないワークスペース。`W2` の書き込みは、`workspaceId` が `W2` の UnitOfWork の `contentRepository` で、`W2` の読み取りは `PartitionReader.workspace(W2)` の `contentRepository` で呼ぶ
- UnitOfWork は、`{ kind: "workspace", workspaceId: W, condition: { kind: "none" } }` を渡した `UnitOfWorkProvider.run` を指す。書き込みは、その `WorkspaceContext` の `contentRepository` で呼ぶ。UnitOfWork の外の読み取りは、`PartitionReader.workspace(W)` の `contentRepository` で呼ぶ。`W0` の読み取りは、`PartitionReader.workspace(W0)` の `contentRepository` で呼ぶ
- 前提条件のコンテンツは、`Content` の振る舞いで作った値を `insert` と `save` で保存し、記した状態にする。単一モデルのコンテンツを作る `Content.create` と、ごみ箱から復元する `Content.restore` は、`occupant` に `null` を渡して呼ぶ。前提条件の書き込みは、`UnitOfWorkProvider.run` の UnitOfWork で呼び、確定済み
- 操作の書き込みは、新しい UnitOfWork で呼び、UnitOfWork を確定する。読み取りは、UnitOfWork の外で呼ぶ
- 「`ConflictError`（理由）になる」は、メソッドの呼び出しか UnitOfWork の確定のどちらかでエラーが現れ、`run` がそのエラーを投げ、その UnitOfWork の書き込みが 1 件も反映されないことを指す
- 「版がある」は、`RevisionRepository.insertMany` を確定した状態。「`RevisionRepository.deleteMany`（R）」は、版 R の ID を渡す呼び出しを指す
- 「`findInbound`（対象 T）」は、`PartitionReader.workspace(W)` の `ContentReferenceQuery.findInbound` を、`targets` が T、`holders` と `strengths` がすべての種類、`page` が 1、`limit` が 100 で呼んだ結果を指す
- ID の大小は [common.md](../../domains/common.md) の「TransactionalRepository」の順序に、文字列の大小は [content.md](../../domains/content.md) の「文字列の比較」に従う
- コンテンツID として書いた `abc`、`x`、`t` などの短い文字列は、`IdGenerator` の形式の有効な ID を表す。`abc` と `ab` は、ID と、その ID の先頭の一部である文字列の関係を表す。アダプターが保存できない形式の文字列を、ID として渡さない
- `range`（`min` …、`max` …）は、`FilterRange` の `bounds` の値を指す。`findPage` に渡す条件は、`ContentListQuery` の `filters`（`ContentFilter`）
- 「`SystemError`（`INVALID_QUERY`）になる」は、呼び出しがそのエラーを返すことを指す。書き込みでは、`run` がそのエラーを投げ、その UnitOfWork の書き込みが 1 件も反映されない
- 参照先として挙げたコンテンツは、断りが無い限り存在する
- `findPage` と `findCandidates` の項目は一意でなく、内容の `uniqueValues` は空。`findByUniqueValue`、`findUniqueHolders`、「確定の時点の整合性」、「区画」の `title` と `slug` は一意な項目で、設定した値が内容の `uniqueValues` に入る
- 「公開する」は、`Content.publish` を適用した値を `save` すること。「公開を取り消す」は、`Content.unpublish` を適用した値を `save` すること
- `findPage` のモデル `M1` は、直下に `title`（テキスト）、`price`（数値）、`flag`（真偽値）、`day`（日付）、`at`（日時）、`kind`（単一選択）、`tags`（複数選択）、`ref`（コンテンツ参照・一件）、`refs`（コンテンツ参照・複数件）を持つ。断りが無い `findPage` は、`scope` が `management`（`status` はすべて `null`、`search` は `null`）、`filters` は空、`sort` は `createdAt` の昇順、`page` は 1、`limit` は 100

## insert

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-01 | コンテンツが無い | 未公開のコンテンツ A を `insert` する | `findById` が、渡した A とすべてのフィールドが等しいコンテンツと `ExpectedVersion` を返す |
| TC-ports-contentRepository-02 | A がある | A と同じ ID のコンテンツを `insert` する | `ConflictError`（`ALREADY_EXISTS`）になる。`findById` は前提条件の A を返す |
| TC-ports-contentRepository-03 | A を `deleteMany` で削除済み | A と同じ ID のコンテンツを `insert` する | 成功する。`findById` が新しいコンテンツを返す |
| TC-ports-contentRepository-04 | `M1` の通常の管理対象に、単一モデルのコンテンツ S がある | `M1` に、別の ID の単一モデルのコンテンツを `insert` する | `ConflictError`（`SINGLE_EXISTS`）になる。`findSingle` は S を返す |
| TC-ports-contentRepository-05 | `M1` に単一モデルのコンテンツ S がある | `M2` に単一モデルのコンテンツを `insert` する | 成功する |
| TC-ports-contentRepository-06 | `M1` のごみ箱に、単一モデルのコンテンツ S がある。通常の管理対象には無い | `M1` に、別の ID の単一モデルのコンテンツ S2 を `insert` する | 成功する。`findSingle`（`M1`）は S2 を返す |
| TC-ports-contentRepository-07 | `M1` のごみ箱に単一モデルのコンテンツ S、通常の管理対象に単一モデルのコンテンツ S2 がある | ごみ箱から復元した S を `save` する | `ConflictError`（`SINGLE_EXISTS`）になる。`findSingle`（`M1`）は S2 を返す。`findById`（S）の `location` は `trashed` |
| TC-ports-contentRepository-08 | `M1` のごみ箱に単一モデルのコンテンツ S、通常の管理対象に単一モデルのコンテンツ S2 がある | 一つの UnitOfWork で、ごみ箱へ移動した S2 の `save` と、ごみ箱から復元した S の `save` を呼ぶ | 成功する。`findSingle`（`M1`）は S を返す |
| TC-ports-contentRepository-09 | `M1` に単一モデルのコンテンツが無い | 一つの UnitOfWork で、`M1` の単一モデルのコンテンツを、別の ID で二つ `insert` する | `ConflictError`（`SINGLE_EXISTS`）になる。`findSingle`（`M1`）は `null` を返す |
| TC-ports-contentRepository-10 | `M1` に複数件モデルのコンテンツ A がある | `M1` に複数件モデルのコンテンツ B を `insert` する | 成功する。`findById` が A と B をそれぞれ返す |
| TC-ports-contentRepository-11 | コンテンツ T がある | 下書きが T への必須の参照を持つ A を `insert` する | `findInbound`（対象 T）が、参照元 A、`holder` が `draft`、`strength` が `required` の記録を返す |
| TC-ports-contentRepository-12 | ID が `t` のコンテンツは無い。ごみ箱のコンテンツ T がある。A は無い | 下書きが、`t` への任意の参照、`t` への必須の参照、`t` への本文のリンク（`link`）、T への必須の参照、A 自身への必須の参照を持つ A を `insert` する | 成功する。`findById` が、それらの参照を持つ A を返す |
| TC-ports-contentRepository-17 | A も B も無い | 一つの UnitOfWork で、下書きが互いへの必須の参照を持つ A と B を `insert` する | 成功する。`findById` が A と B を返す |

## findById

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-19 | コンテンツが無い | `findById`（`x`）を呼ぶ | `null` を返す |
| TC-ports-contentRepository-20 | ごみ箱のコンテンツ A がある | `findById`（A）を呼ぶ | A を返す。`location` は `trashed` で、`trashedAt` と `trashedBy` が保存した値と等しい |
| TC-ports-contentRepository-21 | 公開中で下書きを持つ A がある。公開内容は `violations`、下書きは `issues` を持つ | `findById`（A）を呼ぶ | 公開内容と下書きの `values`・`references`・`uniqueValues`・`updatedAt`・`produced`・`migrated`、`violations`、`issues`、`cardinality`、`createdAt`、`lastChange`、`version` が、保存した値と等しい |
| TC-ports-contentRepository-22 | A がある | `findById`（A）が返した `expectedVersion` で、A を変えた内容を `save` する | 成功する |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-23 | 未公開の A がある | `findById` の `expectedVersion` で、下書きを置き換えた A を `save` する | `findById` が、置き換えた後の A を返す |
| TC-ports-contentRepository-24 | 未公開の A がある | `findById` の `expectedVersion` で、公開した A を `save` する | `findById` が、公開内容を持ち下書きを持たない A を返す |
| TC-ports-contentRepository-25 | A の `expectedVersion` を読んだ後に、A を一度 `save` した | 先に読んだ `expectedVersion` で、A を `save` する | `ConflictError`（`STALE_VERSION`）になる。`findById` は一度目の `save` の内容を返す |
| TC-ports-contentRepository-26 | A を `deleteMany` で削除済み。削除の前に `expectedVersion` を読んだ | その `expectedVersion` で、A を `save` する | `ConflictError`（`STALE_VERSION`）になる。`findById` は `null` を返す |
| TC-ports-contentRepository-27 | コンテンツ T1、T2 がある。A の下書きは T1 への必須の参照を持つ | 下書きを、T2 への必須の参照を持つ内容に置き換えた A を `save` する | `findInbound`（対象 T1）は A の記録を返さない。`findInbound`（対象 T2）は、参照元 A、`holder` が `draft` の記録を返す |
| TC-ports-contentRepository-28 | 公開中のコンテンツ T がある。未公開の A の下書きは T への必須の参照を持つ | 公開した A を `save` する | `findInbound`（対象 T）は、参照元 A の記録を、`holder` が `published` の 1 件だけ返す |
| TC-ports-contentRepository-29 | コンテンツ T がある。公開中の A の公開内容は参照を持たず、下書きは T への必須の参照を持つ | 下書きを破棄した A を `save` する | `findInbound`（対象 T）は A の記録を返さない |
| TC-ports-contentRepository-30 | A がある。ID が `t` のコンテンツは無い | 下書きを、`t` への必須の参照、`t` への本文のリンク（`link`）、`t` への任意の参照、A 自身への必須の参照を持つ内容に置き換えた A を `save` する | 成功する。`findById` が、それらの参照を持つ A を返す |
| TC-ports-contentRepository-34 | A がある。B が無い | 一つの UnitOfWork で、B の `insert` と、下書きを B への必須の参照を持つ内容に置き換えた A の `save` を呼ぶ | 成功する |
| TC-ports-contentRepository-35 | A と B がある。B の下書きは A への任意の参照を持つ。A を `deleteMany` で削除済み | 下書きの、A への参照でない項目を変えた B を `save` する | 成功する |

## saveMany

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-36 | A、B がある | `findByIds` の `expectedVersion` で、下書きを置き換えた A と B を、一度の `saveMany` に渡す | `findById` が、置き換えた後の A と B を返す |
| TC-ports-contentRepository-37 | A、B がある。B の `expectedVersion` を読んだ後に、B を一度 `save` した | A と、先に読んだ `expectedVersion` の B を、一度の `saveMany` に渡す | `ConflictError`（`STALE_VERSION`）になる。`findById`（A）は前提条件の A を返す |
| TC-ports-contentRepository-38 | A がある。B を `deleteMany` で削除済み。削除の前に B の `expectedVersion` を読んだ | A と、その `expectedVersion` の B を、一度の `saveMany` に渡す | `ConflictError`（`STALE_VERSION`）になる。`findById`（A）は前提条件の A を返す |
| TC-ports-contentRepository-39 | 未公開の A と B がある。下書きが互いへの必須の参照を持つ | 公開した A と B を、一度の `saveMany` に渡す | 成功する。`findById` が、公開中の A と B を返す |
| TC-ports-contentRepository-40 | 公開中のコンテンツ T がある。A の下書きは T への必須の参照を持つ | 下書きを、参照を持たない内容に置き換えた A を、`saveMany` に渡す | `findInbound`（対象 T）は A の記録を返さない |
| TC-ports-contentRepository-41 | 同じモデルの未公開の A と B がある。下書きの `slug` は、どちらも `x` | 公開した A と B を、一度の `saveMany` に渡す | `ConflictError`（`UNIQUE_VALUE_TAKEN`）になる。A と B は未公開のまま |
| TC-ports-contentRepository-42 | `ContentBatch.maxContents` 件のコンテンツがある | その全件を一度の `saveMany` に渡す | 成功する |
| TC-ports-contentRepository-43 | `ContentBatch.maxContents` を 1 件超える件数のコンテンツがある | その全件を一度の `saveMany` に渡す | `SystemError`（`INVALID_QUERY`）になる。どのコンテンツも変わらない |
| TC-ports-contentRepository-44 | A がある | 空の `items` で `saveMany` を呼ぶ | 成功する。A は変わらない |
| TC-ports-contentRepository-45 | A、B がある | 下書きを置き換えた A を二つと B を、一度の `saveMany` に渡す | `SystemError`（`INVALID_QUERY`）になる。A と B は変わらない |

## findByIds

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-46 | A がある | 空の `ids` で `findByIds` を呼ぶ | 空の列を返す |
| TC-ports-contentRepository-47 | A、B がある。ID が `x` のコンテンツは無い | `ids` に A、B、`x` を渡す | A と B だけを、`ExpectedVersion` とともに返す |
| TC-ports-contentRepository-49 | ごみ箱のコンテンツ A がある | `ids` に A を渡す | A を返す |
| TC-ports-contentRepository-50 | ID が `a` < `b` < `c` のコンテンツがある | `ids` に `c`、`a`、`b` の順で渡す | `a`、`b`、`c` の順に返す |
| TC-ports-contentRepository-51 | A がある | `ids` に A の ID を 2 回渡す | A を 1 件だけ返す |
| TC-ports-contentRepository-52 | 上限の件数のコンテンツがある | 上限の個数の ID を渡す | 上限の件数を返す |
| TC-ports-contentRepository-53 | 上限の件数のコンテンツがある | 上限の個数の ID に、そのうち 1 個の重複を加えた上限より 1 個多い数を渡す | 上限の件数を返す |
| TC-ports-contentRepository-54 | A がある | 重複の無い上限より 1 個多い ID を渡す | `SystemError`（`INVALID_QUERY`）になる |
| TC-ports-contentRepository-55 | A がある | `findByIds` が返した `expectedVersion` で、A を変えた内容を `save` する | 成功する |

## findSingle

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-56 | `M1` に単一モデルのコンテンツ S がある。`M2` にコンテンツが無い | `findSingle`（`M2`）を呼ぶ | `null` を返す |
| TC-ports-contentRepository-57 | `M1` に単一モデルのコンテンツ S がある | `findSingle`（`M1`）を呼ぶ | S と `ExpectedVersion` を返す |
| TC-ports-contentRepository-59 | `M1` のごみ箱にだけ、単一モデルのコンテンツ S がある | `findSingle`（`M1`）を呼ぶ | `null` を返す |
| TC-ports-contentRepository-60 | `M1` のごみ箱に単一モデルのコンテンツ S、通常の管理対象に単一モデルのコンテンツ S2 がある | `findSingle`（`M1`）を呼ぶ | S2 を返す |
| TC-ports-contentRepository-61 | `M1` に単一モデルのコンテンツ S がある | `findSingle` が返した `expectedVersion` で、S を変えた内容を `save` する | 成功する |

## findPage: 対象

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-62 | `M1` に、公開中で下書きを持たない A、公開中で下書きを持つ B、未公開の C、ごみ箱の D がある。`M2` に公開中の E がある | `scope` が `public` の `findPage` を呼ぶ | A と B を返す。`count` は 2 |
| TC-ports-contentRepository-63 | 公開中の A がある。公開内容の `title` は `x`、下書きの `title` は `y` | `scope` が `public`、`filters` が `title` の `equals` `y` と、`equals` `x` で、それぞれ呼ぶ | `y` では空の一覧を返し、`count` は 0。`x` では A を返す |
| TC-ports-contentRepository-65 | 公開中で下書きを持つ A、B がある。公開内容の `updatedAt` は A が B より前、下書きの `updatedAt` は B が A より前 | `scope` が `public`、`sort` が `updatedAt` の昇順 | A、B の順に返す |
| TC-ports-contentRepository-66 | `M1` に、公開中で下書きを持たない A、入力エラーを持たない下書きの未公開の B、制約違反だけを持つ下書きの C、`missingRequired` を持つ下書きの未公開の D、公開中で `referenceMismatch` を持つ下書きの E、ごみ箱の F がある | `scope` が `preview` の `findPage` を呼ぶ | A、B、C を返す。`count` は 3 |
| TC-ports-contentRepository-67 | 公開中の A がある。公開内容の `title` は `x`、下書きの `title` は `y` | `scope` が `preview`、`filters` が `title` の `equals` `y` と、`equals` `x` で、それぞれ呼ぶ | `y` では A を返す。`x` では空の一覧を返す |
| TC-ports-contentRepository-69 | 公開中で下書きを持つ A、B がある。公開内容の `updatedAt` は A が B より前、下書きの `updatedAt` は B が A より前 | `scope` が `preview`、`sort` が `updatedAt` の昇順 | B、A の順に返す |
| TC-ports-contentRepository-70 | 通常の管理対象の A と、`missingRequired` を持つ下書きの B、ごみ箱の C がある | `scope` が `management` | A と B を返し、C を含めない。`count` は 2 |
| TC-ports-contentRepository-71 | 公開中の A がある。公開内容の `title` は `x`、下書きの `title` は `y` | `scope` が `management`、`filters` が `title` の `equals` `y` | A を返す |
| TC-ports-contentRepository-72 | `M1` に A、`M2` に B がある | `modelId` が `M1` の `findPage` を呼ぶ | A だけを返す |

## findPage: 管理の条件

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-73 | 公開中で下書きを持たない A、公開中で下書きを持つ B、未公開の C がある | `status.publication` を `published` と `unpublished` にして、それぞれ呼ぶ | `published` では A と B を、`unpublished` では C だけを返す |
| TC-ports-contentRepository-75 | 公開中で下書きを持たない A、公開中で下書きを持つ B、未公開の C がある | `status.hasDraft` を `true` と `false` にして、それぞれ呼ぶ | `true` では B と C を、`false` では A だけを返す |
| TC-ports-contentRepository-77 | 公開中で下書きを持たない A、`issues` が空の下書きの B、制約違反を持つ下書きの C、`missingRequired` を持つ下書きの D がある | `status.draftHasIssues` を `true` と `false` にして、それぞれ呼ぶ | `true` では C と D を、`false` では A と B を返す |
| TC-ports-contentRepository-79 | 未公開の A、`violations` が空の公開中の B、`violations` を持つ公開中の C がある | `status.publishedHasViolations` を `true` と `false` にして、それぞれ呼ぶ | `true` では C だけを、`false` では A と B を返す |
| TC-ports-contentRepository-81 | 公開中で下書きを持たない A、公開中で下書きを持つ B、未公開の C がある | `status.publication` が `published`、`status.hasDraft` が `true` | B だけを返す |
| TC-ports-contentRepository-82 | A、B がある。B の ID は `abc`。どの項目の値も `ab` を含まない | `textFields` を空にして、`search.term` を `abc` と `ab` で、それぞれ呼ぶ | `abc` では B だけを返す。`ab` では空の一覧を返す |
| TC-ports-contentRepository-84 | A の `title` は `Hello world`、B の `title` は `Goodbye` | `search.term` が `lo wo`、`textFields` が `title` | A だけを返す |
| TC-ports-contentRepository-85 | A の `title` は `Hello` | `search.term` が `hello`、`textFields` が `title` | 空の一覧を返す |
| TC-ports-contentRepository-86 | A の `title` は未設定 | `search.term` が `a`、`textFields` が `title` | 空の一覧を返す |
| TC-ports-contentRepository-87 | `textFields` が指すテキストの項目が二つある。A は一つ目、B は二つ目の値に `foo` を含む。C は、`textFields` に無いテキストの項目の値だけが `foo` を含む | `search.term` が `foo`、`textFields` がその二つ | A と B を返す |
| TC-ports-contentRepository-89 | 公開中の A がある。公開内容の `title` は `old`、下書きの `title` は `new` | `search.term` が `old`、`textFields` が `title` | 空の一覧を返す |
| TC-ports-contentRepository-90 | A の `title` は `foo` で `price` は 1、B の `title` は `foo` で `price` は 2 | `search.term` が `foo`（`textFields` は `title`）、`filters` が `price` の `equals` 2 | B だけを返す |

## findPage: 絞り込み

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-91 | A の `title` は `abc`、B の `title` は `abcd` | `title` の `equals` `abc` | A だけを返す |
| TC-ports-contentRepository-92 | A の `title` は `ABC` | `title` の `equals` `abc` | 空の一覧を返す |
| TC-ports-contentRepository-93 | A の `title` は前後に空白を持つ ` abc ` | `title` の `equals` `abc` | 空の一覧を返す |
| TC-ports-contentRepository-94 | A の `title` は合成済みの文字（U+00E9）、B の `title` は分解した並び（U+0065 U+0301） | `title` の `equals` U+00E9 | A だけを返す |
| TC-ports-contentRepository-95 | A の `price` は 10、B の `price` は 10.5 | `price` の `equals` 10 | A だけを返す |
| TC-ports-contentRepository-96 | A の `flag` は `true`、B の `flag` は `false` | `flag` の `equals` `false` | B だけを返す |
| TC-ports-contentRepository-97 | A の `day` は 2026-01-01、B の `day` は 2026-01-02 | `day` の `equals` 2026-01-01 | A だけを返す |
| TC-ports-contentRepository-98 | A の `at` は 2026-01-01T00:00:00.000Z、B の `at` は 2026-01-01T00:00:00.001Z | `at` の `equals` 2026-01-01T00:00:00.000Z | A だけを返す |
| TC-ports-contentRepository-99 | A の `kind` は選択肢 `a`、B の `kind` は選択肢 `b` | `kind` の `equals` `a` | A だけを返す |
| TC-ports-contentRepository-100 | `price` が 9、10、20、21 のコンテンツがある | `price` の `range` を（`min` 10、`max` 20）、（`min` 10、`max` は `null`）、（`min` は `null`、`max` 20）にして、それぞれ呼ぶ | それぞれ、10 と 20、10 と 20 と 21、9 と 10 と 20 のコンテンツを返す |
| TC-ports-contentRepository-103 | `price` が 2、10 のコンテンツがある | `price` の `range`（`min` 3、`max` は `null`） | 10 のコンテンツだけを返す |
| TC-ports-contentRepository-104 | `price` が 10 のコンテンツがある | `price` の `range`（`min` 10、`max` 10） | そのコンテンツを返す |
| TC-ports-contentRepository-105 | `day` が 2025-12-31、2026-01-01、2026-01-31、2026-02-01 のコンテンツがある | `day` の `range`（`min` 2026-01-01、`max` 2026-01-31） | 2026-01-01 と 2026-01-31 のコンテンツを返す |
| TC-ports-contentRepository-106 | `at` が 2026-01-01T00:00:00.000Z、2026-01-01T00:00:00.001Z のコンテンツがある | `at` の `range`（`min` は `null`、`max` 2026-01-01T00:00:00.000Z） | 000Z のコンテンツだけを返す |
| TC-ports-contentRepository-107 | A の `tags` は `a` と `b`、B の `tags` は `b`、C の `tags` は `c` | `tags` の `includes` `b` | A と B を返す |
| TC-ports-contentRepository-108 | コンテンツ T がある。A の `ref` は T、B の `ref` は別のコンテンツ | `ref` の `references` T | A だけを返す |
| TC-ports-contentRepository-109 | コンテンツ T がある。A の `refs` は T と U、B の `refs` は U だけ | `refs` の `references` T | A だけを返す |
| TC-ports-contentRepository-110 | ID が `t` のコンテンツは無い。A の `ref`（任意）は `t` | `ref` の `references` `t` | A を返す |
| TC-ports-contentRepository-111 | 未公開のコンテンツ T がある。公開中の A の公開内容の `ref`（任意）は T | `scope` が `public`、`ref` の `references` T | A を返す |
| TC-ports-contentRepository-112 | A は `title`、`price`、`flag`、`day`、`at`、`kind`、`tags`、`ref`、`refs` が未設定 | 各項目の `equals`、`price`・`day`・`at` の `range`（`min` だけ、または `max` だけ）、`tags` の `includes`、`ref` と `refs` の `references` を、一つずつ指定する | どの条件でも、空の一覧を返す |
| TC-ports-contentRepository-115 | A の `title` は `x` で `price` は 1、B の `title` は `x` で `price` は 2、C の `title` は `y` で `price` は 2 | `title` の `equals` `x` と、`price` の `equals` 2 | B だけを返す |
| TC-ports-contentRepository-116 | A の `price` は 5 | A が満たす条件を `ContentDelivery.limits.filters` 個指定して呼び、A が満たす条件を `ContentDelivery.limits.filters` より 1 個少なく、満たさない条件を 1 個指定して呼ぶ | 前者は A を返す。後者は空の一覧を返す |

## findPage: 並び替え

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-118 | `title` が `a`、`B`、`c` のコンテンツがある | `sort` を `title` の昇順と降順にして、それぞれ呼ぶ | 昇順では `B`、`a`、`c` の順に、降順では `c`、`a`、`B` の順に返す |
| TC-ports-contentRepository-120 | `title` が U+FF5E の一文字と、U+2000B の一文字のコンテンツがある | `sort` が `title` の昇順 | U+FF5E、U+2000B の順に返す |
| TC-ports-contentRepository-121 | `title` が `ab`、`a`、`abc` のコンテンツがある | `sort` が `title` の昇順 | `a`、`ab`、`abc` の順に返す |
| TC-ports-contentRepository-122 | `price` が 10、9、-1、0.5 のコンテンツがある | `sort` を `price` の昇順と降順にして、それぞれ呼ぶ | 昇順では -1、0.5、9、10 の順に、降順では 10、9、0.5、-1 の順に返す |
| TC-ports-contentRepository-124 | `day` が 2026-02-01、2025-12-31、2026-01-15 のコンテンツがある | `sort` が `day` の昇順 | 2025-12-31、2026-01-15、2026-02-01 の順に返す |
| TC-ports-contentRepository-125 | `at` が 2026-01-01T00:00:00.001Z、2026-01-01T00:00:00.000Z のコンテンツがある | `sort` が `at` の昇順 | 000Z、001Z の順に返す |
| TC-ports-contentRepository-126 | `kind` が選択肢 `b`、`a`、`C` のコンテンツがある | `sort` が `kind` の昇順 | `C`、`a`、`b` の順に返す |
| TC-ports-contentRepository-127 | `createdAt` が異なる A、B、C がある | `sort` を `createdAt` の昇順と降順にして、それぞれ呼ぶ | 昇順では `createdAt` の古い順に、降順では新しい順に返す |
| TC-ports-contentRepository-129 | 下書きの `updatedAt` が異なる未公開の A、B、C がある | `scope` が `management`、`sort` が `updatedAt` の降順 | 下書きの `updatedAt` の新しい順に返す |
| TC-ports-contentRepository-130 | 公開中で下書きを持たない A と、未公開の B がある。A の公開内容の `updatedAt` は、B の下書きの `updatedAt` より前 | `scope` が `management`、`sort` が `updatedAt` の昇順 | A、B の順に返す |
| TC-ports-contentRepository-131 | `price` が 1 の A、2 の B、未設定の C がある | `sort` を `price` の昇順と降順にして、それぞれ呼ぶ | 昇順では A、B、C の順に、降順では B、A、C の順に返す |
| TC-ports-contentRepository-133 | `title` が未設定の A と、`x` の B がある | `sort` が `title` の昇順、および降順 | どちらも B、A の順に返す |
| TC-ports-contentRepository-134 | `price` が同じ 3 件がある。ID は `a` < `b` < `c` | `sort` を `price` の昇順と降順にして、それぞれ呼ぶ | どちらも `a`、`b`、`c` の順に返す |
| TC-ports-contentRepository-136 | `price` が未設定の 3 件がある。ID は `a` < `b` < `c`。`price` が 1 の `z` がある | `sort` が `price` の降順 | `z`、`a`、`b`、`c` の順に返す |
| TC-ports-contentRepository-137 | `createdAt` が同じ 2 件がある。ID は `a` < `b` | `sort` が `createdAt` の降順 | `a`、`b` の順に返す |
| TC-ports-contentRepository-138 | 5 件のコンテンツがある | 同じ条件の `findPage` を、書き込みを挟まずに 2 回呼ぶ | 2 回とも同じ順序で返す |
| TC-ports-contentRepository-139 | 公開中で下書きを持つ A、B がある。公開内容の `title` は A が `a`、B が `b`。下書きの `title` は A が `z`、B が `y` | `scope` が `public`、`sort` が `title` の昇順 | A、B の順に返す |
| TC-ports-contentRepository-140 | 公開中で下書きを持つ A、B がある。公開内容の `updatedAt` は A が B より前、下書きの `updatedAt` は B が A より前 | `scope` が `management`、`sort` が `updatedAt` の昇順 | B、A の順に返す |

## findPage: ページ分割と総件数

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-141 | `M1` にコンテンツが無い | `page` 1、`limit` 100 | 空の一覧を返す。`count` は 0 |
| TC-ports-contentRepository-142 | コンテンツが 1 件ある | `page` 1、`limit` 1 | その 1 件を返す。`count` は 1 |
| TC-ports-contentRepository-143 | コンテンツが 1 件ある | `page` 2、`limit` 1 | 空の一覧を返す。`count` は 1 |
| TC-ports-contentRepository-144 | コンテンツが 5 件ある | `limit` 2 で、`page` を 2、3、4 にして、それぞれ呼ぶ | それぞれ、並び順の 3 番目と 4 番目、5 番目だけ、空の一覧を返す。`count` はどれも 5 |
| TC-ports-contentRepository-147 | `price` が 1 のコンテンツが 3 件、2 のコンテンツが 4 件ある | `filters` が `price` の `equals` 2、`page` 1、`limit` 2 | 2 件を返す。`count` は 4 |
| TC-ports-contentRepository-148 | `scope` が `preview` の対象が 3 件、`missingRequired` を持つ下書きのコンテンツが 2 件ある | `scope` が `preview`、`page` 1、`limit` 2 | 2 件を返す。`count` は 3 |

## findPreviewExcluded

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-149 | `M1` にコンテンツが無い | `findPreviewExcluded`（`M1`）を呼ぶ | `total` は 0、`items` は空 |
| TC-ports-contentRepository-150 | `M1` に、`missingRequired` を持つ下書きの未公開の A、公開中で `referenceMismatch` を持つ下書きの B、制約違反だけを持つ下書きの C、公開中で下書きを持たない D、ごみ箱にあり `missingRequired` を持つ下書きの E がある | `findPreviewExcluded`（`M1`）を呼ぶ | `total` は 2、`items` は A と B の ID |
| TC-ports-contentRepository-151 | `M2` にだけ、`missingRequired` を持つ下書きのコンテンツがある | `findPreviewExcluded`（`M1`）を呼ぶ | `total` は 0、`items` は空 |
| TC-ports-contentRepository-152 | `missingRequired` を持つ下書きのコンテンツが 3 件ある。ID は `a` < `b` < `c` | `findPreviewExcluded` を呼ぶ | `items` は `a`、`b`、`c` の順 |
| TC-ports-contentRepository-153 | `missingRequired` を持つ下書きのコンテンツが `SchemaLimits.sampleLimit` 件ある | `findPreviewExcluded` を呼ぶ | `total` は `SchemaLimits.sampleLimit`、`items` はその全件 |
| TC-ports-contentRepository-154 | `missingRequired` を持つ下書きのコンテンツが、`SchemaLimits.sampleLimit` を 1 件超える件数ある | `findPreviewExcluded` を呼ぶ | `total` は `SchemaLimits.sampleLimit` より 1 多い件数、`items` は ID の昇順で先頭の `SchemaLimits.sampleLimit` 件 |

## findCandidates

断りが無い `CandidateQuery` は、`models` が `M1`（`textFields` は `title`）、`term` は `null`、`location` は `active`、`publication` と `hasDraft` は `null`、`page` は 1、`limit` は 100。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-155 | コンテンツが無い | `findCandidates` を呼ぶ | 空の一覧を返す。`count` は 0 |
| TC-ports-contentRepository-156 | `M1` に A、`M2` に B、別のモデル `M3` に C がある | `models` が `M1` と `M2` | A と B を返す。`count` は 2 |
| TC-ports-contentRepository-157 | 通常の管理対象の A と、ごみ箱の B がある | `location` を `active` と `trashed` にして、それぞれ呼ぶ | `active` では A だけを、`trashed` では B だけを返す |
| TC-ports-contentRepository-159 | 公開中で下書きを持たない A、公開中で下書きを持つ B、未公開の C がある | `publication` を `published` と `unpublished` にして、それぞれ呼ぶ | `published` では A と B を、`unpublished` では C だけを返す |
| TC-ports-contentRepository-161 | 公開中で下書きを持たない A、公開中で下書きを持つ B、未公開の C がある | `hasDraft` を `true` と `false` にして、それぞれ呼ぶ | `true` では B と C を、`false` では A だけを返す |
| TC-ports-contentRepository-163 | 公開中で下書きを持たない A、公開中で下書きを持つ B、未公開の C がある | `publication` が `published`、`hasDraft` が `true` | B だけを返す |
| TC-ports-contentRepository-164 | A の ID は `abc`。B の `title` は `xabcx`。C はどちらにも当たらない | `term` が `abc` | A と B を返す |
| TC-ports-contentRepository-165 | A の ID は `abc`。どの `title` も `ab` を含まない | `term` が `ab` | 空の一覧を返す |
| TC-ports-contentRepository-166 | A の `title` は `Hello` | `term` が `hello` | 空の一覧を返す |
| TC-ports-contentRepository-167 | 公開中の A がある。公開内容の `title` は `old`、下書きの `title` は `new` | `term` を `new` と `old` にして、それぞれ呼ぶ | `new` では A を返す。`old` では空の一覧を返す |
| TC-ports-contentRepository-169 | `M1` の A は `title` に `foo` を含む。`M2` の B は `title` に `foo` を含み、`name` に含まない | `models` が `M1`（`textFields` は `title`）と `M2`（`textFields` は `name`）、`term` が `foo` | A だけを返す |
| TC-ports-contentRepository-170 | 最新の内容の `updatedAt` が異なる A、B、C がある | `findCandidates` を呼ぶ | `updatedAt` の新しい順に返す |
| TC-ports-contentRepository-171 | 公開中で下書きを持つ A と、未公開の B がある。A の公開内容の `updatedAt` は B の下書きより後、A の下書きの `updatedAt` は B の下書きより前 | `findCandidates` を呼ぶ | B、A の順に返す |
| TC-ports-contentRepository-172 | 最新の内容の `updatedAt` が同じ 3 件がある。ID は `a` < `b` < `c` | `findCandidates` を呼ぶ | `a`、`b`、`c` の順に返す |
| TC-ports-contentRepository-173 | 該当するコンテンツが 5 件ある | `page` 2、`limit` 2 | 並び順の 3 番目と 4 番目を返す。`count` は 5 |
| TC-ports-contentRepository-174 | 該当するコンテンツが 5 件ある | `page` 4、`limit` 2 | 空の一覧を返す。`count` は 5 |
| TC-ports-contentRepository-175 | `M1` に A がある | `models` が空 | 空の一覧を返す。`count` は 0 |
| TC-ports-contentRepository-176 | `missingRequired` を持つ下書きの A がある | `findCandidates` を呼ぶ | A を返す。`count` は 1 |

## findByUniqueValue

`unique` の項目は `title`。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-177 | `title` が `x` の内容を持つコンテンツが無い | `view` が `published`、および `latest` で、`title` が `x` を探す | どちらも空の列を返す |
| TC-ports-contentRepository-178 | 公開中の A がある。公開内容の `title` は `x`、下書きの `title` は `y` | `view` が `published` で、`x` と `y` をそれぞれ探す | `x` では A を返す。`y` では空の列を返す |
| TC-ports-contentRepository-180 | 公開中の A がある。公開内容の `title` は `x`、下書きの `title` は `y` | `view` が `latest` で、`y` と `x` をそれぞれ探す | `y` では A を返す。`x` では空の列を返す |
| TC-ports-contentRepository-182 | 公開中で下書きを持たない A がある。公開内容の `title` は `x` | `view` が `latest` で `x` を探す | A を返す |
| TC-ports-contentRepository-183 | 未公開の A がある。下書きの `title` は `x` | `view` が `published` で `x` を探す | 空の列を返す |
| TC-ports-contentRepository-184 | ごみ箱の A がある。下書きの `title` は `x` | `view` が `latest` で `x` を探す | 空の列を返す |
| TC-ports-contentRepository-185 | `M2` にだけ、下書きの `title` が `x` のコンテンツがある | `M1` で、`view` が `latest` で `x` を探す | 空の列を返す |
| TC-ports-contentRepository-186 | A の下書きの `title` は `X`、B の下書きの `title` は ` x ` | `view` が `latest` で `x` を探す | 空の列を返す |
| TC-ports-contentRepository-187 | 下書きの `title` が `x` の未公開のコンテンツが 3 件ある。ID は `a` < `b` < `c` | `view` が `latest` で `x` を探す | `a`、`b`、`c` の順に返す |
| TC-ports-contentRepository-188 | 下書きの `title` が `x` の未公開のコンテンツが `ContentDelivery.limits.uniqueMatches` 件ある | `view` が `latest` で `x` を探す | その全件を返す |
| TC-ports-contentRepository-189 | 下書きの `title` が `x` の未公開のコンテンツが、`ContentDelivery.limits.uniqueMatches` を 1 件超える件数ある | `view` が `latest` で `x` を探す | ID の昇順で先頭の `ContentDelivery.limits.uniqueMatches` 件を返す |

## findUniqueHolders

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-190 | 公開中の A がある | 空の `values` で呼ぶ | 空の列を返す |
| TC-ports-contentRepository-191 | 公開中の A がある。公開内容の `title` は `x` | A の値を含む、項目と値がすべて違う上限の個数の `values` に、そのうち 1 個と項目も値も同じ要素を加えた上限より 1 個多い数を渡す | 成功する。A の 1 件を返す |
| TC-ports-contentRepository-192 | 公開中の A がある。公開内容の `title` は `x` | `values` が `title` の `x` | `modelId` が `M1`、`field` が `title`、`value` が `x`、`contentId` が A の 1 件を返す |
| TC-ports-contentRepository-193 | `M1` の公開中の A と、`M2` の公開中の B がある。どちらも公開内容の `title` は `x` | `M1` で、`values` が `title` の `x` | A の 1 件を返す。B の組を返さない |
| TC-ports-contentRepository-194 | 公開中の A がある。公開内容の `title` は `x`、下書きの `title` は `y`。下書きの `title` が `y` の未公開の B と、ごみ箱の C がある | `values` が `title` の `y` | 空の列を返す |
| TC-ports-contentRepository-196 | 公開中の A（`title` は `x`）、B（`title` は `y`）、C（`title` は `z`）がある | `values` が `title` の `x` と `title` の `y` | A と B の組を返す。C の組を返さない。順序は確かめない |
| TC-ports-contentRepository-197 | 公開中の A と B がある。A の公開内容の `title` は `x`、B の公開内容の `slug` は `y` | `values` が `title` の `x` と `slug` の `y` | `field` が `title`、`value` が `x`、`contentId` が A の組と、`field` が `slug`、`value` が `y`、`contentId` が B の組の 2 件を返す。順序は確かめない |
| TC-ports-contentRepository-198 | 公開中の A がある。公開内容の `title` は `x`、`slug` は `y` | `values` が `title` の `x` と `slug` の `y` | `contentId` が A の組を、`title` と `slug` の 2 件返す。順序は確かめない |
| TC-ports-contentRepository-199 | 公開中の A がある。公開内容の `title` は `x`、別の一意な項目 `slug` は `y` | `values` が `slug` の `x` | 空の列を返す |
| TC-ports-contentRepository-200 | 公開中の A がある。公開内容の `title` は `X` | `values` が `title` の `x` | 空の列を返す |
| TC-ports-contentRepository-202 | 公開中の A がある。公開内容の `title` は `x` | A の値を含む上限の個数の `values` で呼ぶ | A の組を返す |
| TC-ports-contentRepository-203 | 公開中の A がある | 上限より 1 個多い `values` で呼ぶ | `SystemError`（`INVALID_QUERY`）になる |

## deleteMany

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-204 | 公開中で下書きを持つ A と、A の版 R1、R2 がある | A を `deleteMany` に渡す | `findById`（A）は `null`。`RevisionRepository.findPage`（A）の `count` は 0。`RevisionRepository.findById`（R1、R2）は `null` |
| TC-ports-contentRepository-205 | コンテンツ T がある。A の下書きは T への必須の参照を持つ | A を `deleteMany` に渡す | `findInbound`（対象 T）は A の記録を返さない |
| TC-ports-contentRepository-206 | A、B、C がある | A と B を一度の `deleteMany` に渡す | `findById`（A、B）は `null`。`findById`（C）は C を返す |
| TC-ports-contentRepository-207 | A がある | 空の `targets` を `deleteMany` に渡す | 成功する。`findById`（A）は A を返し、`version` は変わらない |
| TC-ports-contentRepository-208 | A、B がある | A を二度と B を含む `targets` を `deleteMany` に渡す | `SystemError`（`INVALID_QUERY`）になる。`findById` は A と B を返す |
| TC-ports-contentRepository-209 | ごみ箱の A がある | A を `deleteMany` に渡す | `findById`（A）は `null` を返す |
| TC-ports-contentRepository-210 | A、B がある。B の `expectedVersion` を読んだ後に、B を `save` した | A の現在の `expectedVersion` と、B の先に読んだ `expectedVersion` を渡す | `ConflictError`（`STALE_VERSION`）になる。`findById` は A と B を返す |
| TC-ports-contentRepository-211 | A がある。B を `deleteMany` で削除済み。削除の前に B の `expectedVersion` を読んだ | A と、その `expectedVersion` の B を渡す | `ConflictError`（`STALE_VERSION`）になる。`findById` は A を返す |
| TC-ports-contentRepository-212 | 公開中の A と B がある。B の公開内容は A への必須の参照を持つ | A を `deleteMany` に渡す | `ConflictError`（`PUBLISHED_REFERENCE_REQUIRED`）になる。`findById` は A を返す |
| TC-ports-contentRepository-213 | 公開中の A と B がある。B の公開内容は A への本文のリンク（`link`）を持つ | A を `deleteMany` に渡す | `ConflictError`（`PUBLISHED_REFERENCE_REQUIRED`）になる |
| TC-ports-contentRepository-214 | 公開中の A と B がある。B の公開内容は A への必須の参照を持つ | A と B を一度の `deleteMany` に渡す | 成功する。`findById`（A、B）は `null` |
| TC-ports-contentRepository-215 | 公開中の B がある。B の公開内容は、未公開の A への任意の参照を持つ | A を `deleteMany` に渡す | 成功する |
| TC-ports-contentRepository-216 | 公開中の A がある。公開内容の `slug` は `x` | A を `deleteMany` に渡して確定し、別の UnitOfWork で、`slug` が `x` の B を公開する | どちらも成功する。`findUniqueHolders`（`slug` の `x`）は B の組だけを返す |
| TC-ports-contentRepository-217 | A と B がある。B の下書きは A への必須の参照を持つ | A を `deleteMany` に渡す | `ConflictError`（`REFERENCED`）になる |
| TC-ports-contentRepository-218 | A と B がある。B の下書きは A への本文のリンク（`link`）を持つ | A を `deleteMany` に渡す | `ConflictError`（`REFERENCED`）になる |
| TC-ports-contentRepository-219 | A と B がある。B の版 R は A への必須の参照を持つ。B の公開内容と下書きは A を参照しない | A を `deleteMany` に渡す | `ConflictError`（`REFERENCED`）になる |
| TC-ports-contentRepository-220 | A と、ごみ箱の B がある。B の下書きは A への必須の参照を持つ | A を `deleteMany` に渡す | `ConflictError`（`REFERENCED`）になる |
| TC-ports-contentRepository-221 | A と、ごみ箱の B がある。B の版 R は A への `link` の参照を持つ | A を `deleteMany` に渡す | `ConflictError`（`REFERENCED`）になる |
| TC-ports-contentRepository-222 | A、B、C と、A の版 R がある。C の下書きは B への必須の参照を持つ | A と B を一度の `deleteMany` に渡す | `ConflictError`（`REFERENCED`）になる。`findById` は A と B を返す。`RevisionRepository.findById`（R）は R を返す |
| TC-ports-contentRepository-223 | A と B がある。B の下書きは A への任意の参照を持つ | A を `deleteMany` に渡す | 成功する。`findById`（A）は `null`、`findById`（B）は B を返す |
| TC-ports-contentRepository-224 | A と B がある。A の下書きは A 自身と B への必須の参照を、B の下書きは A への必須の参照を持つ | A と B を一度の `deleteMany` に渡す | 成功する。`findById`（A、B）は `null` |
| TC-ports-contentRepository-226 | A の版 R は A への必須の参照を持つ | A を `deleteMany` に渡す | 成功する。`RevisionRepository.findById`（R）は `null` |
| TC-ports-contentRepository-227 | A と B がある。A の版 R は B への必須の参照を持つ | A と B を一度の `deleteMany` に渡す | 成功する |
| TC-ports-contentRepository-228 | A と B がある。B の下書きは A への必須の参照を持っていたが、参照を持たない内容に置き換えて `save` した | A を `deleteMany` に渡す | 成功する |
| TC-ports-contentRepository-229 | A と B がある。B の版 R だけが A への必須の参照を持っていたが、R を `RevisionRepository.deleteMany` で削除した | A を `deleteMany` に渡す | 成功する |
| TC-ports-contentRepository-230 | A と B がある。B の版 R だけが A への必須の参照を持つ | 一つの UnitOfWork で、`RevisionRepository.deleteMany`（R）と、A の `deleteMany` を呼ぶ | 成功する。`findById`（A）と `RevisionRepository.findById`（R）は `null`。`findById`（B）は B を返す |
| TC-ports-contentRepository-231 | A と B がある。B の版 R1 と R2 が A への必須の参照を持つ | 一つの UnitOfWork で、`RevisionRepository.deleteMany`（R1）と、A の `deleteMany` を呼ぶ | `ConflictError`（`REFERENCED`）になる。`findById`（A）は A、`RevisionRepository.findById`（R1）は R1 を返す |
| TC-ports-contentRepository-232 | A と B がある。A の下書きは B への必須の参照を持つ | A を `deleteMany` に渡して確定し、別の UnitOfWork で B を `deleteMany` に渡す | どちらも成功する |
| TC-ports-contentRepository-233 | `ContentBatch.maxContents` 件のコンテンツがある | その全件を一度の `deleteMany` に渡す | 成功する。`count` は 0 |
| TC-ports-contentRepository-234 | `ContentBatch.maxContents` を 1 件超える件数のコンテンツがある | その全件を一度の `deleteMany` に渡す | `SystemError`（`INVALID_QUERY`）になる。`count` は、渡した件数 |
| TC-ports-contentRepository-235 | A と、`ContentBatch.maxRevisions` を 1 件超える件数の A の版がある | A を `deleteMany` に渡す | 成功する。`findById`（A）は `null`。`RevisionRepository.findPage`（A）の `count` は 0 |

## findTrashedBefore

`cutoff` は時点 `t`。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-236 | ごみ箱のコンテンツが無い。通常の管理対象の A がある | `after` が `null`、`limit` 100 で呼ぶ | 空の列を返す |
| TC-ports-contentRepository-237 | ごみ箱に、`trashedAt` が `t` より前の A、`t` と等しい B、`t` より後の C がある | `after` が `null`、`limit` 100 で呼ぶ | A と B を、`ExpectedVersion` とともに返す |
| TC-ports-contentRepository-238 | ごみ箱に、`trashedAt` が `t-3` の C、`t-2` の A、`t-1` の B がある | `after` が `null`、`limit` 100 で呼ぶ | C、A、B の順に返す |
| TC-ports-contentRepository-239 | ごみ箱に、`trashedAt` が同じ 3 件がある。ID は `a` < `b` < `c` | `after` が `null`、`limit` 100 で呼ぶ | `a`、`b`、`c` の順に返す |
| TC-ports-contentRepository-240 | ごみ箱に、順序が 1 番目から 5 番目の 5 件がある | `after` が `null`、`limit` 2 で呼び、`after` が 2 番目の `trashedAt` と `id`、`limit` 2 で呼ぶ | 前者は 1 番目と 2 番目を、後者は 3 番目と 4 番目を返す |
| TC-ports-contentRepository-242 | ごみ箱に、順序が 1 番目から 5 番目の 5 件がある | `after` が 5 番目の `trashedAt` と `id` で呼ぶ | 空の列を返す |
| TC-ports-contentRepository-243 | ごみ箱に、`trashedAt` が同じ 3 件がある。ID は `a` < `b` < `c` | `after` がその `trashedAt` と `a`、`limit` は上限で呼ぶ | `b`、`c` の順に返す |
| TC-ports-contentRepository-244 | ごみ箱に、順序が 1 番目から 3 番目の 3 件がある。1 番目を `after` にする位置を控えた後、1 番目を `deleteMany` で削除した | `after` が控えた位置で呼ぶ | 2 番目と 3 番目を返す |
| TC-ports-contentRepository-245 | ごみ箱に、対象が、上限より 1 件多くある | `after` が `null`、`limit` は上限で呼び、最後の `trashedAt` と `id` を `after` にしてもう一度呼ぶ | 一度目は先頭から上限の件数、二度目は残りの 1 件を返す |
| TC-ports-contentRepository-247 | ごみ箱に、他から参照されていない A がある | `findTrashedBefore` が返した `expectedVersion` で、A を `deleteMany` に渡す | 成功する |
| TC-ports-contentRepository-248 | ごみ箱に A がある | `limit` は上限より 1 大きい値で呼ぶ | `SystemError`（`INVALID_QUERY`）になる |
| TC-ports-contentRepository-249 | ごみ箱に A がある | `limit` 0 で呼ぶ | `SystemError`（`INVALID_QUERY`）になる |
| TC-ports-contentRepository-250 | `M1` のごみ箱に A、`M2` のごみ箱に B がある。どちらも `trashedAt` は `t` より前 | `after` が `null`、`limit` 100 で呼ぶ | A と B を返す |

## count、lastId、countByPublication、countTrashedBefore

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-251 | コンテンツが無い | `count` を呼ぶ | 0 を返す |
| TC-ports-contentRepository-252 | `M1` の通常の管理対象が 2 件、`M2` の通常の管理対象が 1 件、ごみ箱が 1 件ある | `count` を呼ぶ | 4 を返す |
| TC-ports-contentRepository-253 | コンテンツが無い | `lastId` を呼ぶ | `null` を返す |
| TC-ports-contentRepository-254 | `M1` の `b`、`M2` の `c`、`M1` の `a` を、この順に `insert` した | `lastId` を呼ぶ | `c` を返す |
| TC-ports-contentRepository-255 | 通常の管理対象の `a` と、ごみ箱の `b` がある | `lastId` を呼ぶ | `b` を返す |
| TC-ports-contentRepository-256 | ID が `a` < `b` の 2 件がある。`b` を `deleteMany` で削除した | `lastId` を呼ぶ | `a` を返す |
| TC-ports-contentRepository-257 | ID が `a` < `b` < `c` の 3 件がある | `lastId` と、`scan`（`null`、100）を呼ぶ | `lastId` は、`scan` が最後に返したコンテンツの ID と等しい |
| TC-ports-contentRepository-258 | コンテンツが無い | `countByPublication` を呼ぶ | `published` は 0、`unpublished` は 0 |
| TC-ports-contentRepository-259 | 公開中で下書きを持たない A、公開中で下書きを持つ B、未公開の C、ごみ箱の D がある | `countByPublication` を呼ぶ | `published` は 2、`unpublished` は 2 |
| TC-ports-contentRepository-260 | ごみ箱のコンテンツが無い。通常の管理対象の A がある | `countTrashedBefore`（`t`）を呼ぶ | 0 を返す |
| TC-ports-contentRepository-261 | ごみ箱に、`trashedAt` が `t` より前の A、`t` と等しい B、`t` より後の C がある | `countTrashedBefore`（`t`）を呼ぶ | 2 を返す |

## 何も書き込んでいないワークスペース

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-262 | なし | `W0` で、`findById`、`findByIds`、`findSingle`、`findPage`、`findPreviewExcluded`、`findCandidates`、`findByUniqueValue`、`findUniqueHolders`、`findTrashedBefore`、`scan`、`count`、`lastId`、`countByPublication`、`countTrashedBefore` を呼ぶ | `findById`、`findSingle`、`lastId` は `null`、`findPage` と `findCandidates` は空の一覧で `count` は 0、`findPreviewExcluded` は `total` が 0 で `items` は空、`count` と `countTrashedBefore` は 0、`countByPublication` は `published` が 0 で `unpublished` が 0、ほかは空の列を返す |

## scan

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-263 | コンテンツが無い | `scan`（`null`、100）を呼ぶ | 空の列を返す |
| TC-ports-contentRepository-264 | `M1` の `b`、`M2` の `a`、ごみ箱の `c` がある | `scan`（`null`、100）を呼ぶ | `a`、`b`、`c` の順に返す |
| TC-ports-contentRepository-265 | ID が `a` < `b` < `c` < `d` < `e` の 5 件がある | `scan`（`null`、2）と、`scan`（`b`、2）を呼ぶ | 前者は `a`、`b` を、後者は `c`、`d` を返す |
| TC-ports-contentRepository-267 | ID が `a` < `b` < `c` < `d` < `e` の 5 件がある | `scan`（`e`、2）を呼ぶ | 空の列を返す |
| TC-ports-contentRepository-268 | ID が `a`、`c` の 2 件がある。`b` は無い | `scan`（`b`、上限の `limit`）を呼ぶ | `c` だけを返す |
| TC-ports-contentRepository-269 | 上限より 1 件多くある | `scan`（`null`、上限の `limit`）を呼び、最後の ID を `after` にしてもう一度呼ぶ | 一度目は ID の昇順で先頭から上限の件数、二度目は残りの 1 件を返す |
| TC-ports-contentRepository-271 | 1 件がある | `scan`（`null`、上限より 1 大きい `limit`）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる |
| TC-ports-contentRepository-272 | 1 件がある | `scan`（`null`、0）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる |
| TC-ports-contentRepository-273 | 1 件がある | `scan`（`null`、1.5）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-274 | コンテンツが無い | `deleteAll`（10）を呼ぶ | 0 を返す |
| TC-ports-contentRepository-275 | 通常の管理対象が 2 件、ごみ箱が 1 件ある | `deleteAll`（10）を呼ぶ | 3 を返す。`count` は 0 |
| TC-ports-contentRepository-276 | 5 件ある | `deleteAll`（2）を、呼び出しごとに別の UnitOfWork で、0 が返るまで繰り返す | 2、2、1、0 の順に返す。一度目の後の `count` は 3、最後の `count` は 0 |
| TC-ports-contentRepository-278 | 2 件ある | `deleteAll`（2）を呼び、別の UnitOfWork でもう一度呼ぶ | 2、0 の順に返す |
| TC-ports-contentRepository-279 | 1 件ある | `deleteAll`（上限の `limit`）を呼ぶ | 1 を返す。`count` は 0 |
| TC-ports-contentRepository-280 | 1 件ある | `deleteAll`（0）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる。`count` は 1 |
| TC-ports-contentRepository-281 | 1 件ある | `deleteAll`（上限より 1 大きい `limit`）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる。`count` は 1 |
| TC-ports-contentRepository-282 | 2 件ある | 一つの UnitOfWork で `deleteAll`（10）を呼んだ後、`fn` が例外を投げる | `run` はその例外を投げる。`count` は 2 |
| TC-ports-contentRepository-283 | A、B と、公開中の T がある。B の下書きは A への必須の参照を持つ。A は公開中で、公開内容は T への必須の参照を持つ | `deleteAll`（1）を、別の UnitOfWork で 3 回呼ぶ | 3 回とも 1 を返す。`count` は 0 |
| TC-ports-contentRepository-285 | ID が `t` のコンテンツは無い。A の下書きは `t` への必須の参照を持つ | `deleteAll`（10）を呼ぶ | `findInbound`（対象 `t`）は空の一覧を返す |
| TC-ports-contentRepository-286 | 公開内容の `slug` が `x` の公開中の A がある | `deleteAll`（10）を確定する。別の UnitOfWork で、下書きの `slug` が `x` の B を `insert` して確定し、さらに別の UnitOfWork で B を公開する | どれも成功する。`findUniqueHolders`（`slug` の `x`）は B の組だけを返す |
| TC-ports-contentRepository-287 | A と、A の版 R がある | `deleteAll`（10）を呼ぶ | 1 を返す。`findById`（A）は `null`、`RevisionRepository.findById`（R）は R を返す |

## 確定の時点の整合性: 一意な値

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-288 | 公開中の A がある。公開内容の `slug` は `x`。B の下書きの `slug` は `x` | B を公開する | `ConflictError`（`UNIQUE_VALUE_TAKEN`）になる。`findById`（B）は未公開の B を返す |
| TC-ports-contentRepository-289 | 公開中の A と B がある。A の公開内容の `slug` は `x`、B の公開内容の `slug` は `y` | 公開内容の `slug` を `x` に置き換えた B を `save` する | `ConflictError`（`UNIQUE_VALUE_TAKEN`）になる |
| TC-ports-contentRepository-290 | 公開中の A と B がある。A の公開内容の `slug` は `x`、B の公開内容の `slug` は `y` | 下書きの `slug` が `x` の C を `insert` し、別の UnitOfWork で、`slug` が `x` の下書きを加えた B を `save` する | どちらも成功する |
| TC-ports-contentRepository-292 | 公開中の A がある。公開内容の `slug` は `x`。B の下書きの `title` は `x`、`slug` は `y` | B を公開する | 成功する |
| TC-ports-contentRepository-293 | 公開中の A がある。公開内容の `slug` は `x`。B の下書きの `slug` は `X` | B を公開する | 成功する |
| TC-ports-contentRepository-294 | `M1` の公開中の A がある。公開内容の `slug` は `x`。`M2` の B の下書きの `slug` は `x` | B を公開する | 成功する |
| TC-ports-contentRepository-295 | 公開中の A がある。公開内容の `slug` は未設定で、`uniqueValues` は空。B の下書きも同じ | B を公開する | 成功する |
| TC-ports-contentRepository-296 | A と B がある。どちらも下書きの `slug` は `x` | 一つの UnitOfWork で、A と B を公開する | `ConflictError`（`UNIQUE_VALUE_TAKEN`）になる。`findById` は未公開の A と B を返す |
| TC-ports-contentRepository-297 | 公開中の A がある。公開内容の `slug` は `x`。B の下書きの `slug` は `x` | 一つの UnitOfWork で、A の公開を取り消し、B を公開する | 成功する。`findUniqueHolders`（`slug` の `x`）は B の組だけを返す |
| TC-ports-contentRepository-298 | 公開中の A と B がある。A の公開内容の `slug` は `x`、B の公開内容の `slug` は `y` | 一つの UnitOfWork で、公開内容の `slug` を `y` に置き換えた A と、`x` に置き換えた B を `save` する | 成功する |
| TC-ports-contentRepository-299 | 公開中の A がある。公開内容の `slug` は `x`。A の公開を取り消して確定した | 下書きの `slug` が `x` の B を公開する | 成功する |
| TC-ports-contentRepository-300 | 公開中の A がある。公開内容の `slug` は `x` | 公開内容の `slug` を `y` に置き換えた A を `save` して確定し、別の UnitOfWork で、下書きの `slug` が `x` の B を公開する | どちらも成功する |

## 確定の時点の整合性: 公開内容からの参照

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-301 | 未公開の T がある。A の下書きは T への必須の参照を持つ | A を公開する | `ConflictError`（`REFERENCE_NOT_PUBLISHED`）になる。`findById`（A）は未公開の A を返す |
| TC-ports-contentRepository-302 | 未公開の T がある。A の下書きは T への本文のリンク（`link`）を持つ | A を公開する | `ConflictError`（`REFERENCE_NOT_PUBLISHED`）になる |
| TC-ports-contentRepository-303 | ごみ箱の T がある。A の下書きは T への必須の参照を持つ | A を公開する | `ConflictError`（`REFERENCE_NOT_PUBLISHED`）になる |
| TC-ports-contentRepository-304 | 未公開の T と、公開中の A がある | 公開内容を、T への必須の参照を持つ内容に置き換えた A を `save` する | `ConflictError`（`REFERENCE_NOT_PUBLISHED`）になる。`findById`（A）は前提条件の A を返す |
| TC-ports-contentRepository-305 | 公開中の T がある。A の下書きは T への必須の参照を持つ | A を公開する | 成功する |
| TC-ports-contentRepository-306 | 公開中で下書きを持つ T がある。A の下書きは T への本文のリンク（`link`）を持つ | A を公開する | 成功する |
| TC-ports-contentRepository-307 | 未公開の T がある。ID が `t` のコンテンツは無い。A の下書きは、T と `t` への任意の参照を持つ | A を公開する | 成功する |
| TC-ports-contentRepository-309 | 未公開の T と、公開中の A がある | T への必須の参照を持つ下書きを加えた A を `save` する | 成功する |
| TC-ports-contentRepository-310 | 未公開の T がある。A の下書きは T への、T の下書きは A への必須の参照を持つ | 一つの UnitOfWork で、T と A を公開する | 成功する。`findById` は公開中の A と T を返す |
| TC-ports-contentRepository-312 | A の下書きは A 自身への必須の参照を持つ | A を公開する | 成功する |
| TC-ports-contentRepository-313 | ID が `t` のコンテンツは無い。公開中の A がある | 公開内容を、`t` への必須の参照を持つ内容に置き換えた A を `save` する | `ConflictError`（`REFERENCE_NOT_PUBLISHED`）になる。`findById`（A）は前提条件の A を返す |
| TC-ports-contentRepository-314 | 未公開の T がある。A の下書きは T への必須の参照を持つ | 公開した A を `saveMany` に渡す | `ConflictError`（`REFERENCE_NOT_PUBLISHED`）になる。`findById`（A）は未公開の A を返す |

## 確定の時点の整合性: 公開内容をなくす変更

公開中の A の公開内容が、公開中の T を参照する。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-315 | 参照は必須の参照 | T の公開を取り消す | `ConflictError`（`PUBLISHED_REFERENCE_REQUIRED`）になる。`findById`（T）は公開中の T を返す |
| TC-ports-contentRepository-316 | 参照は本文のリンク（`link`） | T の公開を取り消す | `ConflictError`（`PUBLISHED_REFERENCE_REQUIRED`）になる |
| TC-ports-contentRepository-317 | 参照は必須の参照 | ごみ箱へ移動した T を `save` する | `ConflictError`（`PUBLISHED_REFERENCE_REQUIRED`）になる。`findById`（T）は、通常の管理対象の公開中の T を返す |
| TC-ports-contentRepository-318 | 参照は必須の参照 | T を `deleteMany` に渡す | `ConflictError`（`PUBLISHED_REFERENCE_REQUIRED`）になる |
| TC-ports-contentRepository-319 | 参照は任意の参照 | T の公開を取り消す | 成功する |
| TC-ports-contentRepository-320 | 参照は任意の参照 | T を `deleteMany` に渡す | 成功する |
| TC-ports-contentRepository-321 | 参照は必須の参照 | 下書きを加えた T を `save` し、別の UnitOfWork で、公開内容の値を置き換えた T を `save` する | どちらも成功する |
| TC-ports-contentRepository-323 | 参照は必須の参照 | 一つの UnitOfWork で、A と T の公開を取り消す | 成功する。`findById` は未公開の A と T を返す |
| TC-ports-contentRepository-324 | 参照は必須の参照 | 一つの UnitOfWork で、公開内容を T を参照しない内容に置き換えた A の `save` と、T の公開の取り消しを呼ぶ | 成功する |
| TC-ports-contentRepository-325 | 参照は必須の参照 | A の公開を取り消して確定し、別の UnitOfWork で T の公開を取り消す | どちらも成功する |
| TC-ports-contentRepository-326 | 参照は必須の参照 | 公開を取り消した T を `saveMany` に渡す | `ConflictError`（`PUBLISHED_REFERENCE_REQUIRED`）になる。`findById`（T）は公開中の T を返す |
| TC-ports-contentRepository-327 | 参照は必須の参照 | 公開を取り消した A と T を、一度の `saveMany` に渡す | 成功する。`findById` は未公開の A と T を返す |
| TC-ports-contentRepository-328 | 参照は必須の参照。A は下書きを持たない | 一つの UnitOfWork で、A の公開の取り消しと、T の `deleteMany` を呼ぶ | `ConflictError`（`REFERENCED`）になる。`findById` は公開中の A と T を返す |
| TC-ports-contentRepository-329 | 参照は必須の参照。B の下書きも T への必須の参照を持つ。A は、T への本文のリンク（`link`）を持つ下書きも持つ | T を `deleteMany` に渡す | `ConflictError` になる。理由は `PUBLISHED_REFERENCE_REQUIRED` と `REFERENCED` のいずれか。`findById`（T）は公開中の T を返す |

## 区画

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-331 | `W` に A がある | `W2` に、A と同じ ID のコンテンツを `insert` する | 成功する。`W` の `findById`（A）は `W` の A を、`W2` の `findById`（A）は `W2` のコンテンツを返す |
| TC-ports-contentRepository-332 | `W` に A がある | `W2` の `findById`（A）、`findByIds`（A）、`scan`（`null`、100）、`count` を呼ぶ | `findById` は `null`、`findByIds` と `scan` は空の列、`count` は 0 を返す |
| TC-ports-contentRepository-333 | `W` の `M1` の通常の管理対象に、単一モデルのコンテンツ S がある | `W2` の `M1` に、単一モデルのコンテンツを `insert` する | 成功する。`W` の `findSingle`（`M1`）は S を返す |
| TC-ports-contentRepository-334 | `W` に公開中の A がある。公開内容の `slug` は `x` | `W2` で、下書きの `slug` が `x` の B を `insert` して確定し、別の UnitOfWork で B を公開する | どれも成功する。`W2` の `findUniqueHolders`（`slug` の `x`）は B の組だけを、`W` の `findUniqueHolders`（`slug` の `x`）は A の組だけを返す |
| TC-ports-contentRepository-335 | `W` に公開中の T がある。`W2` に、ID が T と同じコンテンツは無く、下書きが T の ID への必須の参照を持つ A がある | `W2` で A を公開する | `ConflictError`（`REFERENCE_NOT_PUBLISHED`）になる |

## 並行性

二つの UnitOfWork を並行に実行する。どちらの `run` も、相手の結果を待たずに呼ぶ。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-336 | A がある。二つの呼び出し元が、同じ `expectedVersion` を読んだ | 二つの UnitOfWork が、それぞれ A を別の内容で `save` する | 一方だけが確定する。他方は `ConflictError`（`STALE_VERSION`）になる。`findById` は、確定した側の内容を返す |
| TC-ports-contentRepository-337 | A がある。二つの呼び出し元が、同じ `expectedVersion` を読んだ | 一方の UnitOfWork が A を `save` し、他方の UnitOfWork が A を `deleteMany` に渡す | 一方だけが確定する。他方は `ConflictError`（`STALE_VERSION`）になる |
| TC-ports-contentRepository-338 | ID が `x` のコンテンツが無い | 二つの UnitOfWork が、それぞれ ID が `x` のコンテンツを `insert` する | 一方だけが確定する。他方は `ConflictError`（`ALREADY_EXISTS`）になる |
| TC-ports-contentRepository-339 | `M1` に単一モデルのコンテンツが無い | 二つの UnitOfWork が、それぞれ別の ID で、`M1` の単一モデルのコンテンツを `insert` する | 一方だけが確定する。他方は `ConflictError`（`SINGLE_EXISTS`）になる。`findSingle` は、確定した側のコンテンツを返す |
| TC-ports-contentRepository-340 | A と T がある。T は参照されていない | 一方の UnitOfWork が、下書きを T への必須の参照を持つ内容に置き換えた A を `save` し、他方の UnitOfWork が T を `deleteMany` に渡す | `save` は確定する。`deleteMany` は、確定するか、`ConflictError`（`REFERENCED`）になる。`deleteMany` が確定した場合、`findById`（A）は、T への参照を持つ A を返し、`findById`（T）は `null` を返す |
| TC-ports-contentRepository-341 | T がある。T は参照されていない | 一方の UnitOfWork が、下書きが T への本文のリンク（`link`）を持つ A を `insert` し、他方の UnitOfWork が T を `deleteMany` に渡す | `insert` は確定する。`deleteMany` は、確定するか、`ConflictError`（`REFERENCED`）になる |
| TC-ports-contentRepository-342 | A と T がある。T は参照されていない | 一方の UnitOfWork が、下書きを T への任意の参照を持つ内容に置き換えた A を `save` し、他方の UnitOfWork が T を `deleteMany` に渡す | どちらも確定する |
| TC-ports-contentRepository-343 | A と B がある。どちらも下書きの `slug` は `x` | 一方の UnitOfWork が A を、他方の UnitOfWork が B を公開する | 一方だけが確定する。他方は `ConflictError`（`UNIQUE_VALUE_TAKEN`）になる。`findUniqueHolders`（`slug` の `x`）は 1 件の組を返す |
| TC-ports-contentRepository-344 | 公開中の T がある。A の下書きは T への必須の参照を持つ | 一方の UnitOfWork が A を公開し、他方の UnitOfWork が T の公開を取り消す | 一方だけが確定する。A の公開が確定した場合、他方は `ConflictError`（`PUBLISHED_REFERENCE_REQUIRED`）になる。T の公開の取り消しが確定した場合、他方は `ConflictError`（`REFERENCE_NOT_PUBLISHED`）になる |
| TC-ports-contentRepository-345 | A の下書きの `slug` は `x`、B の下書きの `slug` は `y` | 一方の UnitOfWork が A を、他方の UnitOfWork が B を公開する | どちらも確定する |
| TC-ports-contentRepository-346 | `M1` のごみ箱に、単一モデルのコンテンツ S がある。通常の管理対象には無い | 一方の UnitOfWork が、ごみ箱から復元した S を `save` し、他方の UnitOfWork が、別の ID の単一モデルのコンテンツを `insert` する | 一方だけが確定する。他方は `ConflictError`（`SINGLE_EXISTS`）になる |

## 可視性と UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-contentRepository-347 | A が無い | A を `insert` して確定した直後に、`findById`、`findByIds`、`findPage`、`scan`、`count` を呼ぶ | どの読み取りも A を反映する |
| TC-ports-contentRepository-348 | コンテンツ T がある。A、B が無い | 一つの UnitOfWork で、A の `insert`、B の `insert`、T の `save` を呼び、確定する | `findById` が、A、B、置き換えた後の T を返す |
| TC-ports-contentRepository-349 | A と B がある | 一つの UnitOfWork で、A の `save`、B の `deleteMany`、C の `insert` を呼んだ後、`fn` が例外を投げる | `run` はその例外を投げる。`findById` は、前提条件の A と B を返し、C には `null` を返す |
| TC-ports-contentRepository-350 | A と、コンテンツ T がある | 一つの UnitOfWork で、下書きを T への必須の参照を持つ内容に置き換えた A を `save` した後、`fn` が例外を投げる | `findInbound`（対象 T）は A の記録を返さない |
| TC-ports-contentRepository-351 | A がある。B が無い | 一つの UnitOfWork で、B の `insert` と、A と同じ ID のコンテンツの `insert` を呼ぶ | `ConflictError`（`ALREADY_EXISTS`）になる。`findById`（B）は `null` を返す |
| TC-ports-contentRepository-352 | A と B がある。B の `expectedVersion` を読んだ後に、B を `save` した | 一つの UnitOfWork で、A の `save`（現在の `expectedVersion`）と、B の `save`（先に読んだ `expectedVersion`）を呼ぶ | `ConflictError`（`STALE_VERSION`）になる。`findById`（A）は前提条件の A を返す |
| TC-ports-contentRepository-353 | A がある | 一つの UnitOfWork で、A の `save` と、A の置き換えた下書きから作った版 R の `RevisionRepository.insertMany` を呼び、確定する | `findById` が置き換えた後の A を返す。`RevisionRepository.findById`（R）が R を返す |
| TC-ports-contentRepository-354 | A と、既にある版 R がある | 一つの UnitOfWork で、A の `save` と、R と同じ `id` の版の `RevisionRepository.insertMany` を呼ぶ | `ConflictError`（`ALREADY_EXISTS`）になる。`findById`（A）は前提条件の A を返す |
