# MediaRepository

「上限」は、その節のメソッドの契約が定める、一度に渡せる件数・個数、または `limit` の上限を指す。値は、ポートを定めるドメインのファイルが持つ。

契約は [media.md](../../domains/media.md) の「ポート」と、[common.md](../../domains/common.md) の「TransactionalRepository」「Pagination と PaginationResult」「WriteScope」「UnitOfWorkProvider」が定める。使用箇所は、`MediaUsageRepository` が記録する。

## 記法

- W、W1、W2 はワークスペース、M、M1、M2 は `MediaId`。リポジトリは、そのワークスペースの区画のもの。書き込みでは、W の UnitOfWork（`{ kind: "workspace", workspaceId: W, condition: { kind: "none" } }` を渡した `UnitOfWorkProvider.run`）の文脈から、読み取りでは `PartitionReader.workspace(W)` から得る。メソッドは `workspaceId` を取らない。「W に」「W の区画で」は、W の区画のリポジトリを使うことを指す。W1 と W2 を使うケースは、それぞれの区画のリポジトリを使う。区画に `Workspace` を確定しておく必要は無い。「W の区画に何も確定していない」は、W の UnitOfWork で一度も確定していないことを指す
- メディアは `Media` の振る舞い（`register`、`unpublish`）が返す値で作り、`insert` と `save` で保存する。公開のメディアは、`publishMany` で公開する。メディアを `insert` する UnitOfWork は、`MediaProfile.create` の値を `MediaProfileRepository.insert` で保存する。「`name` が X のメディア」は、`MediaProfile.describe` で名称を X にした管理情報を、`MediaProfileRepository.save` で保存したメディア
- 「派生形式の記録がある」は、`MediaDerivatives` の振る舞いが返す値を、`MediaDerivativesRepository.insert` と `save` で保存した状態
- H、H1、H2 は `holderId`。「H が M を公開使用する」は、`MediaUsageRepository.replaceMany([{ holderId: H, uses: [{ mediaId: M, public: true }] }])` を確定した状態。「H が M を使う」は、`public` を偽にして確定した状態。「H の使用箇所を消す」は、`replaceMany([{ holderId: H, uses: [] }])`。J は `stagingId`（文字列）。「`unpublish` の結果」は、`Media.unpublish` が返す `PrivateMedia`
- 前提条件の書き込みは、W の UnitOfWork で呼び、確定済み。操作の書き込みは、W の新しい UnitOfWork で呼ぶ。読み取りは UnitOfWork の外で呼ぶ
- 「`ConflictError`（理由）になる」は、メソッドの呼び出しか確定のどちらかでエラーが現れ、`run` がそのエラーを投げ、その UnitOfWork の書き込みが一つも反映されないことを指す
- v は、直前の `findById` が返した `expectedVersion`。`createdAt` は古い順に c1 < c2 < c3
- 絞り込まない `filter` は `{ text: null, derivativeStatuses: null, definitions: [] }`。`derivativeStatuses` を指定する `filter` の `definitions` は、断りが無い限り、前提の画像の派生形式の記録が持つキーごとに、その状態の定義と等しい定義の列。`findPage()` と書いた場合は、絞り込まない `filter` と `{ page: 1, limit: 100 }`

## 対象にしない

- `findPage` と並行した登録・削除で、ページをまたいで起こり得る重複と欠落。契約は、起こり得ることだけを述べ、結果を約束しない
- 利用者の入力の検証（`MediaListFilter.create`、`Pagination.create`）。ポートの前に済む
- 保存先の障害と、デコードできない保存値（common.md の共通の規則）

## insert と findById

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaRepository-01 | M のメディアが無い | `findById(M)` を呼ぶ | `null` を返す |
| TC-ports-mediaRepository-02 | M のメディアが無い | `register` した画像を `insert` し、確定の後に `findById(M)` を呼ぶ | `insert` した値と全フィールドが等しい `entity`（`file` の `format`・`byteSize`・`width`・`height` を含む）と、`expectedVersion` を返す |
| TC-ports-mediaRepository-03 | M のメディアが無い | `register` した文書を `insert` し、確定の後に `findById(M)` を呼ぶ | `insert` した値と全フィールドが等しい `entity` を返す |
| TC-ports-mediaRepository-04 | M のメディアがある | 同じ ID M の、内容が異なるメディアを `insert` する | `ConflictError`（`ALREADY_EXISTS`）になる。`findById(M)` は、先に `insert` した値を返す |
| TC-ports-mediaRepository-05 | M のメディアが無い | 並行する二つの UnitOfWork が、それぞれ ID M のメディアを `insert` する | 一方だけが確定し、もう一方は `ConflictError`（`ALREADY_EXISTS`）になる。`findById(M)` は、確定した側の値を返す |
| TC-ports-mediaRepository-06 | M のメディアが W1 の区画にある | W1 の区画と W2 の区画で、それぞれ `findById(M)` を呼ぶ | W1 の区画は `entity` を返す。W2 の区画は `null` を返す |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaRepository-07 | M の公開のメディアがある。使用箇所は無い | `unpublish` の結果を `save` し、`findById(M)` を呼ぶ | `visibility` が `private` の `entity` を返す |
| TC-ports-mediaRepository-08 | M のメディアがある。v を得た後、別の UnitOfWork が `save` を確定した | v とともに `save` する | `ConflictError`（`STALE_VERSION`）になる。`findById(M)` は、先に確定した値を返す |
| TC-ports-mediaRepository-09 | M の公開のメディアがある。使用箇所は無い。二つの処理が、同じ `expectedVersion` を得た | 並行する二つの UnitOfWork が、それぞれ `unpublish` の結果を `save` する | 一方だけが確定し、もう一方は `ConflictError`（`STALE_VERSION`）になる。`findById(M)` は、確定した側の値を返す |
| TC-ports-mediaRepository-10 | M のメディアを `save` 済み | `findById(M)` で得た新しい `expectedVersion` とともに、次の変更を `save` する | 確定する。`findById(M)` は次の変更の値を返す |
| TC-ports-mediaRepository-11 | M のメディアを `insert` し、v を得た後、`delete` を確定済み | M のメディアを v とともに `save` する | `ConflictError`（`STALE_VERSION`）になる。`findById(M)` は `null` を返す |
| TC-ports-mediaRepository-12 | M の画像と、その派生形式の記録がある。二つの処理が、それぞれの `expectedVersion` を得た | 並行する二つの UnitOfWork の、一方が `Media` の `save` を、もう一方が `MediaDerivativesRepository.save` を呼ぶ | 両方が確定する |

## save の公開使用の保護

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaRepository-13 | M の公開のメディアがあり、H が M を公開使用する | `unpublish` の結果を `save` する | `ConflictError`（`MEDIA_IN_PUBLIC_USE`）になる。`findById(M)` は `public` のメディアを返す |
| TC-ports-mediaRepository-14 | M の公開のメディアがあり、H が M を使う。公開使用は無い | `unpublish` の結果を `save` する | 確定する。`findById(M)` は `private` のメディアを返す |
| TC-ports-mediaRepository-15 | M の公開のメディアがあり、H1 が M を公開使用し、H2 が M を使う。その後、H1 の使用箇所を消した、または H1 の使用箇所を、M を使わない内容に `replaceMany` 済み | `unpublish` の結果を `save` する | 確定する |
| TC-ports-mediaRepository-17 | M1 と M2 の公開のメディアがあり、H が M2 だけを公開使用する | M1 の `unpublish` の結果を `save` する | 確定する |
| TC-ports-mediaRepository-18 | M の公開のメディアがある。使用箇所は無い。`MediaUsageRepository.stage(J, …)` で、H が M を公開使用する内容を、準備領域に確定済み | `unpublish` の結果を `save` する | 確定する。準備領域の使用箇所を判定に含めない |
| TC-ports-mediaRepository-19 | M の公開のメディアがあり、H が M を公開使用する | 一つの UnitOfWork で、H の使用箇所を消す `replaceMany` と、`unpublish` の結果の `save` を呼ぶ | 確定する。判定は、確定後の状態に対して行う |
| TC-ports-mediaRepository-20 | M の公開のメディアがあり、H が M を使う | 一つの UnitOfWork で、H が M を公開使用する `replaceMany` と、`unpublish` の結果の `save` を呼ぶ | `ConflictError` になる。理由は `MEDIA_IN_PUBLIC_USE` または `MEDIA_NOT_PUBLIC`。メディアは `public` のまま、H の使用箇所は前のまま |
| TC-ports-mediaRepository-21 | M の公開のメディアがある。使用箇所は無い | 並行する二つの UnitOfWork の、一方が `unpublish` の結果の `save` を、もう一方が、H が M を公開使用する `replaceMany` を呼ぶ | 一方だけが確定する。記録の側が負けた場合は `ConflictError`（`MEDIA_NOT_PUBLIC`）になり、メディアは `private`。非公開化の側が負けた場合は `ConflictError`（`MEDIA_IN_PUBLIC_USE`）になり、メディアは `public` のまま |
| TC-ports-mediaRepository-22 | M の非公開のメディアと、その管理情報がある。管理情報の `expectedVersion` を得た | 並行する二つの UnitOfWork の、一方が、H が M を公開使用する `replaceMany` と `publishMany([M], t)` を、もう一方が、`MediaProfile.describe` の結果の `MediaProfileRepository.save` を呼ぶ | 両方が確定する。メディアは `public` で、H の公開使用が記録され、管理情報は保存した値 |

## delete

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaRepository-23 | M のメディアがある。使用箇所は無い | `delete(M, v)` を呼び、確定の後に `findById(M)` を呼ぶ | `null` を返す |
| TC-ports-mediaRepository-24 | M のメディアがある。v を得た後、別の UnitOfWork が `save` を確定した | `delete(M, v)` を呼ぶ | `ConflictError`（`STALE_VERSION`）になる。`findById(M)` は、`save` した値を返す |
| TC-ports-mediaRepository-25 | M のメディアを `insert` し、v を得た後、`delete` を確定済み | もう一度 `delete(M, v)` を呼ぶ | `ConflictError`（`STALE_VERSION`）になる |
| TC-ports-mediaRepository-26 | M の画像と、その派生形式の記録がある | `delete(M, v)` を呼び、確定の後に `MediaDerivativesRepository.findById(M)` を呼ぶ | `null` を返す。派生形式の記録を同時に削除する |
| TC-ports-mediaRepository-27 | M1 と M2 の画像と、それぞれの派生形式の記録がある | `delete(M1, v)` を呼び、確定の後に `MediaDerivativesRepository.findById(M2)` を呼ぶ | M2 の派生形式の記録を返す |
| TC-ports-mediaRepository-28 | M の文書がある。派生形式の記録は無い | `delete(M, v)` を呼ぶ | 確定する。`findById(M)` は `null` を返す |
| TC-ports-mediaRepository-29 | M のメディアと、その管理情報がある | `delete(M, v)` を呼び、確定の後に `MediaProfileRepository.findById(M)` を呼ぶ | `null` を返す。管理情報を同時に削除する |
| TC-ports-mediaRepository-30 | M のメディアがあり、H が M を使う。非公開のメディアを `public` が偽で、公開のメディアを `public` が真で、派生形式の記録を持つ画像を `public` が偽で、それぞれ確かめる | `delete(M, v)` を呼ぶ | `ConflictError`（`MEDIA_IN_USE`）になる。`findById(M)` はメディアを、`MediaDerivativesRepository.findById(M)` は派生形式の記録を返す |
| TC-ports-mediaRepository-33 | M のメディアがあり、H1 と H2 が M を使う。H1 の使用箇所を消した | `delete(M, v)` を呼ぶ | `ConflictError`（`MEDIA_IN_USE`）になる |
| TC-ports-mediaRepository-34 | M のメディアがあり、H が M を使う。その後、H の使用箇所を、M を使わない内容に `replaceMany` 済み、または消した | `delete(M, v)` を呼ぶ | 確定する。`findById(M)` は `null` を返す |
| TC-ports-mediaRepository-36 | M1 と M2 のメディアがあり、H が M2 だけを使う | `delete(M1, v)` を呼ぶ | 確定する |
| TC-ports-mediaRepository-37 | M のメディアがある。使用箇所は無い。`MediaUsageRepository.stage(J, …)` で、H が M を使う内容を、準備領域に確定済み | `delete(M, v)` を呼ぶ | 確定する。準備領域の使用箇所を判定に含めない |
| TC-ports-mediaRepository-38 | M のメディアがあり、H が M を使う | 一つの UnitOfWork で、H の使用箇所を消す `replaceMany` と、`delete(M, v)` を呼ぶ | 確定する。判定は、確定後の状態に対して行う |
| TC-ports-mediaRepository-39 | M のメディアがある。使用箇所は無い | 並行する二つの UnitOfWork の、一方が `delete(M, v)` を、もう一方が、H が M を使う `replaceMany` を呼ぶ | 一方だけが確定する。削除が確定した場合、もう一方は `ConflictError`（`MEDIA_NOT_FOUND`）になる。記録が確定した場合、もう一方は `ConflictError`（`MEDIA_IN_USE`）になり、メディアは残る |

## findByIds

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaRepository-40 | W に M1、M2、M3 のメディアがある | `findByIds([M1, M3])` を呼ぶ | M1 と M3 の 2 件を返す。順序は問わない |
| TC-ports-mediaRepository-41 | W に M1 のメディアがある | `findByIds([])` を呼ぶ | 空の列を返す |
| TC-ports-mediaRepository-42 | W に M1 のメディアがある。M9 のメディアは無い | `findByIds([M1, M9])` を呼ぶ | M1 の 1 件だけを返す |
| TC-ports-mediaRepository-43 | W1 の区画に M1、W2 の区画に M2 のメディアがある | W1 の区画で `findByIds([M1, M2])` を呼ぶ | M1 の 1 件だけを返す |
| TC-ports-mediaRepository-44 | W の区画に何も確定していない | `findByIds([M1])` を呼ぶ | 空の列を返す |
| TC-ports-mediaRepository-45 | W に M1 と M2 のメディアがある | `findByIds([M1, M1, M2])` を呼ぶ | M1 と M2 の 2 件を返す。M1 を一度だけ返す |
| TC-ports-mediaRepository-46 | W にメディアが、上限の件数だけある | その全件の ID を渡して `findByIds` を呼ぶ | 上限の件数を返す |
| TC-ports-mediaRepository-47 | W にメディアが、上限の件数だけある | その全件の ID に、同じ ID の重複を 1 個加えた上限より 1 個多い数を渡して `findByIds` を呼ぶ | 上限の件数を返す。上限は、重複を除いた数に適用する |
| TC-ports-mediaRepository-48 | W にメディアが、上限より 1 件多くある | その全件の ID を渡して `findByIds` を呼ぶ | `SystemError`（`INVALID_QUERY`）を投げる。上限の件数に切り詰めない |
| TC-ports-mediaRepository-49 | W に、画像と文書、公開と非公開のメディアがある | `findByIds` の結果を `Media.facts` に渡す | メディアごとに、保存した値に合う `kind`（`image` または `document`）と `visibility` を返す |

## publishMany

`publishMany` に渡す `now` は、固定の日時 t。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaRepository-50 | W に非公開の M1 と M2 がある | `publishMany([M1, M2], t)` を呼ぶ | 確定する。`findById` は、M1 と M2 を、`Media.publish` の結果と等しい値（`public`、`version` が 1 進み、`updatedAt` が t）で返す。ほかのフィールドは変わらない |
| TC-ports-mediaRepository-51 | W に公開の M がある | `publishMany([M], t)` を呼ぶ | 確定する。`findById(M)` は、前と同じ値と、前と等しい `expectedVersion` を返す。公開のメディアを変えない |
| TC-ports-mediaRepository-52 | W に非公開の M1 と、公開の M2 がある | `publishMany([M1, M2], t)` を呼ぶ | 確定する。M1 は `public` になり、M2 は前と同じ値のまま |
| TC-ports-mediaRepository-53 | W に M のメディアがある | 空の列を `publishMany` に渡す | 確定する。`findById(M)` は前と同じ値と、前と等しい `expectedVersion` を返す |
| TC-ports-mediaRepository-54 | W に非公開の M がある | `publishMany([M, M], t)` を呼ぶ | 確定する。M は `public` で、`version` は 1 だけ進む |
| TC-ports-mediaRepository-55 | W に非公開の M1 がある。M9 のメディアは無い | `publishMany([M1, M9], t)` を呼ぶ | `ConflictError`（`MEDIA_NOT_FOUND`）になる。M1 は `private` のまま |
| TC-ports-mediaRepository-56 | W1 の区画に非公開の M1、W2 の区画に M2 のメディアがある | W1 の区画で `publishMany([M1, M2], t)` を呼ぶ | `ConflictError`（`MEDIA_NOT_FOUND`）になる。M1 は `private` のまま |
| TC-ports-mediaRepository-57 | W に非公開の M がある。`findById(M)` の結果を得た後、別の UnitOfWork が、M の公開使用の記録と `publishMany([M], t)` を確定した | `publishMany([M], t)` を呼ぶ | 確定する。読んだときの `version` に依らない。M は `public` で、`version` はもう進まない |
| TC-ports-mediaRepository-58 | W に非公開の M がある | 並行する二つの UnitOfWork が、それぞれ `publishMany([M], t)` を呼ぶ | 両方が確定する。M は `public` |
| TC-ports-mediaRepository-59 | W に非公開の M がある | 一つの UnitOfWork で、`publishMany([M], t)` と、H が M を公開使用する `replaceMany` を呼ぶ | 確定する。M は `public` で、非公開にできない |
| TC-ports-mediaRepository-60 | W に非公開の M がある。`findById(M)` で `expectedVersion` を得た後、`publishMany([M], t)` を確定した | 得た `expectedVersion` で `delete(M, v)` を呼ぶ | `ConflictError`（`STALE_VERSION`）になる。M は `public` のまま |
| TC-ports-mediaRepository-61 | W に公開の M がある。`findById(M)` で `expectedVersion` を得た後、`publishMany([M], t)` を確定した | 得た `expectedVersion` で、M の `unpublish` の結果を `save` する | 確定する。`publishMany` は公開のメディアの `version` を進めていない。M は `private` になる |
| TC-ports-mediaRepository-62 | W に非公開の M がある | `publishMany([M], t)` を呼んだ後、`fn` が例外を投げる | M は `private` のまま |
| TC-ports-mediaRepository-63 | W に、非公開のメディアが `MediaLimits.maxPublishedAtOnce` 件ある。この件数は `UnitOfWorkCapacity.workspaceWrites` を超える | 一つの UnitOfWork で、その全件の ID を一度の `publishMany` に渡し、H がその全件を公開使用する `replaceMany` を呼ぶ | 確定する。どのメディアも `public` で、非公開にできない。`publishMany` は、件数によらず書き込み一回に数える |
| TC-ports-mediaRepository-64 | W にメディアがある | `MediaLimits.maxPublishedAtOnce` を超える数の、重複の無い ID を `publishMany` に渡す | `SystemError`（`INVALID_QUERY`）を投げる。どのメディアも前の値のまま |

## findPage

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaRepository-65 | W のメディアが無い | `findPage()` を呼ぶ | `items` は空、`count` は 0 |
| TC-ports-mediaRepository-66 | W の区画に何も確定していない | `findPage()` を呼ぶ | `items` は空、`count` は 0 |
| TC-ports-mediaRepository-67 | W にメディアが 1 件ある | `findPage()` を呼ぶ | `items` はその 1 件、`count` は 1 |
| TC-ports-mediaRepository-68 | W に、`createdAt` が c2、c1、c3 のメディアを、この順に `insert` 済み | `findPage()` を呼ぶ | `items` は `createdAt` が c3、c2、c1 の順、`count` は 3 |
| TC-ports-mediaRepository-69 | W に、`createdAt` が同じ c1 で、ID が異なるメディアが 3 件ある | `findPage()` を呼ぶ | `items` は ID のコードポイントの降順 |
| TC-ports-mediaRepository-70 | W に、`createdAt` が c1 のメディアがあり、`publishMany` で公開して `updatedAt` を進めた。`createdAt` が c2 の別のメディアがある | `findPage()` を呼ぶ | `items` は c2、c1 の順。更新日時で並べない |
| TC-ports-mediaRepository-71 | W1 の区画に 2 件、W2 の区画に 3 件のメディアがある | W1 の区画で `findPage()` を呼ぶ | `items` は W1 の 2 件だけ、`count` は 2 |
| TC-ports-mediaRepository-72 | W に、画像と文書、公開と非公開のメディアがある | `findPage()` を呼ぶ | すべてを返す |
| TC-ports-mediaRepository-73 | W に、`name` が `Summer Photo`、`winter`、`logo` のメディアがある | `text` を `photo` にして `findPage` を呼ぶ | `Summer Photo` の 1 件だけを返す。`count` は 1 |
| TC-ports-mediaRepository-74 | W に、`name` が `logo` で `searchInfo` が `Campaign 2026` のメディアと、`searchInfo` が空のメディアがある | `text` を `campaign` にして `findPage` を呼ぶ | `searchInfo` が一致する 1 件だけを返す |
| TC-ports-mediaRepository-75 | W に、`name` が `banner` のメディアがある | `text` を `BANNER` にして `findPage` を呼ぶ | そのメディアを返す |
| TC-ports-mediaRepository-76 | W に、`name` が `banner` のメディアがある | `text` を、前後に空白を付けた ` ann ` にして `findPage` を呼ぶ | そのメディアを返す |
| TC-ports-mediaRepository-77 | W に、`name` が `春の写真` のメディアがある | `text` を `の写` にして `findPage` を呼ぶ | そのメディアを返す |
| TC-ports-mediaRepository-78 | W にメディアが 3 件ある | `text` を、空白だけの文字列にして `findPage` を呼ぶ | 3 件を返す。`count` は 3 |
| TC-ports-mediaRepository-79 | W にメディアが 3 件ある | `text` を、どの `name` と `searchInfo` にも含まれない語にして `findPage` を呼ぶ | `items` は空、`count` は 0 |
| TC-ports-mediaRepository-80 | W に、代替テキストが `hero` で、`name` と `searchInfo` が `hero` を含まないメディアがある | `text` を `hero` にして `findPage` を呼ぶ | `items` は空。共有する説明は対象にしない |
| TC-ports-mediaRepository-81 | W に、派生形式がすべて `ready` の画像 A、すべて `generating` の画像 B、すべて `failed` の画像 C、文書 D がある | `derivativeStatuses` を `["failed"]`、`["ready"]` にして、それぞれ `findPage` を呼ぶ | それぞれ C の 1 件、A の 1 件だけを返す。`count` は 1 |
| TC-ports-mediaRepository-82 | 同上 | `derivativeStatuses` を `["generating", "failed"]` にして `findPage` を呼ぶ | B と C の 2 件を返す。`count` は 2 |
| TC-ports-mediaRepository-84 | 同上 | `derivativeStatuses` を `["generating", "ready", "failed"]` にして `findPage` を呼ぶ | A、B、C の 3 件を返す。文書 D は返さない |
| TC-ports-mediaRepository-85 | 同上 | `derivativeStatuses` を `null` にして `findPage` を呼ぶ | A、B、C、D の 4 件を返す |
| TC-ports-mediaRepository-86 | 同上 | `derivativeStatuses` を空の集合（`[]`）にして `findPage` を呼ぶ | `items` は空、`count` は 0 |
| TC-ports-mediaRepository-87 | W に、`ready` と `failed` の派生形式を一つずつ持つ画像がある | `derivativeStatuses` を `["ready"]` にして、次に `["failed"]` にして `findPage` を呼ぶ | どちらもその画像を返す |
| TC-ports-mediaRepository-88 | W に、派生形式の記録の `states` が空の画像がある | `definitions` を空にし、`derivativeStatuses` を `["generating", "ready", "failed"]` にして `findPage` を呼ぶ。次に、`definitions` を派生形式 a の定義だけにし、`derivativeStatuses` を `["generating"]` にして `findPage` を呼ぶ | 一回目は、その画像を返さない。二回目は、その画像を返す |
| TC-ports-mediaRepository-89 | W に、`stalled` で `failed` の派生形式を持つ画像がある | `derivativeStatuses` を `["failed"]` にして `findPage` を呼ぶ | その画像を返す |
| TC-ports-mediaRepository-90 | W に、`failed` の派生形式を持つ画像があり、`MediaDerivatives.retry` の結果を `MediaDerivativesRepository.save` 済み | `derivativeStatuses` を `["failed"]` にして `findPage` を呼ぶ | その画像を返さない |
| TC-ports-mediaRepository-91 | W に、`name` が `banner` で `failed` を持つ画像、`name` が `banner` で `failed` を持たない画像、`name` が `logo` で `failed` を持つ画像がある | `text` を `banner`、`derivativeStatuses` を `["failed"]` にして `findPage` を呼ぶ | 両方を満たす 1 件だけを返す。`count` は 1 |
| TC-ports-mediaRepository-92 | W にメディアが 5 件ある | 絞り込まずに `{ page: 1, limit: 2 }`、`page: 2`、`page: 3`、`page: 4` を順に呼ぶ | `items` は順に 2 件、2 件、1 件、0 件で、通して契約の順。重複も欠落も無い。`count` はどれも 5 |
| TC-ports-mediaRepository-122 | W に、派生形式 a が定義 A で `ready` の画像がある | `definitions` を、キー a の定義 B（A と異なる）だけにし、`derivativeStatuses` を `["generating"]`、次に `["ready"]` にして、それぞれ `findPage` を呼ぶ | 一回目は、その画像を返す。二回目は返さない |
| TC-ports-mediaRepository-93 | W に、`text` に一致するメディアが 3 件、一致しないメディアが 4 件ある | `text` を指定して `{ page: 1, limit: 2 }` を呼ぶ | `items` は 2 件、`count` は 3 |

## countByVisibility

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaRepository-94 | W のメディアが無い | `countByVisibility()` を呼ぶ | `{ public: 0, private: 0 }` を返す |
| TC-ports-mediaRepository-95 | W の区画に何も確定していない | `countByVisibility()` を呼ぶ | `{ public: 0, private: 0 }` を返す |
| TC-ports-mediaRepository-96 | W に、非公開のメディアが 3 件、公開のメディアが 2 件ある | `countByVisibility()` を呼ぶ | `{ public: 2, private: 3 }` を返す |
| TC-ports-mediaRepository-97 | W1 の区画に非公開が 1 件、W2 の区画に公開が 4 件ある | W1 の区画で `countByVisibility()` を呼ぶ | `{ public: 0, private: 1 }` を返す |
| TC-ports-mediaRepository-98 | W に非公開のメディアが 1 件ある | そのメディアを `publishMany` で公開し、確定の後に `countByVisibility()` を呼ぶ | `{ public: 1, private: 0 }` を返す |
| TC-ports-mediaRepository-99 | W に公開のメディアが 2 件ある | 1 件を `delete` し、確定の後に `countByVisibility()` を呼ぶ | `{ public: 1, private: 0 }` を返す |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaRepository-100 | W のメディアが無い | `deleteAll(10)` を呼ぶ | 0 を返す |
| TC-ports-mediaRepository-101 | W にメディアが 3 件ある | `deleteAll(10)` を呼び、確定の後に `findPage()` と各メディアの `findById` を呼ぶ | `limit` より少ない 3 を返す。`count` は 0、`findById` はどれも `null` |
| TC-ports-mediaRepository-102 | W に、画像が 2 件とその派生形式の記録、文書が 1 件ある | `deleteAll(10)` を呼び、確定の後に `MediaDerivativesRepository.scan(null, 10)` と、各メディアの `MediaProfileRepository.findById` を呼ぶ | メディアの数の 3 を返す。`scan` は空の列、`MediaProfileRepository.findById` はどれも `null` を返す |
| TC-ports-mediaRepository-103 | W にメディアが 3 件ある | `deleteAll(3)` を呼び、確定の後に、別の UnitOfWork でもう一度呼ぶ | 1 回目は 3、2 回目は 0 を返す |
| TC-ports-mediaRepository-104 | W にメディアが 5 件ある | `deleteAll(2)` を、呼び出しごとに別の UnitOfWork で、返す件数が 2 より少なくなるまで繰り返す | 返す値は順に 2、2、1。最後の `count` は 0 |
| TC-ports-mediaRepository-105 | W1 の区画に画像が 2 件、W2 の区画に画像が 3 件と、それぞれの派生形式の記録がある | W1 の区画で `deleteAll(10)` を呼び、確定の後に、W2 の区画で `findPage()` と `MediaDerivativesRepository.scan(null, 10)` を呼ぶ | 2 を返す。W2 の `count` は 3、`scan` は 3 件を返す |
| TC-ports-mediaRepository-106 | W に M の公開のメディアがあり、H1 が M を公開使用し、H2 が M を使う | `deleteAll(10)` を呼ぶ | 1 を返し、確定する。使用箇所の記録によらず削除する |
| TC-ports-mediaRepository-107 | W に、`insert` の後に `save` を 2 回確定したメディアがある。`expectedVersion` を読んでいない | `deleteAll(10)` を呼ぶ | 1 を返し、確定する。楽観ロックを使わない |
| TC-ports-mediaRepository-108 | W に M のメディアがあり、`MediaStorage` の `put` と `commit` で M の元のファイルの実体を保管済み | `deleteAll(10)` を呼び、確定の後に `MediaStorage.get` を呼ぶ | `MediaStorage.get` は、保管した実体を返す。実体を消去しない |
| TC-ports-mediaRepository-109 | W にメディアが 3 件ある | `deleteAll(1)` を呼ぶ | 1 を返す。`findPage()` の `count` は 2 |
| TC-ports-mediaRepository-110 | W にメディアが 3 件ある | `deleteAll`（上限の `limit`）を呼ぶ | 3 を返す |
| TC-ports-mediaRepository-111 | W にメディアが 3 件ある | `limit` に、範囲外の値（0、上限より 1 大きい値）を渡して `deleteAll` を呼ぶ | どちらも `SystemError`（`INVALID_QUERY`）を投げる。`findPage()` の `count` は 3 |

## UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaRepository-113 | `StorageReservation.reserve` の記録 R を `StorageReservationRepository.insert` 済み | 一つの UnitOfWork で、画像の `insert`、`MediaDerivativesRepository.insert`、R の ID の `StorageReservationRepository.settle` を呼び、正常に終える | `findById` はメディアを、`MediaDerivativesRepository.findById` は派生形式の記録を返す。R の ID の `StorageReservationRepository.findById` は `null` を返す |
| TC-ports-mediaRepository-114 | 同上 | 一つの UnitOfWork で、同じ三つを呼んだ後、`fn` が例外を投げる | `run` はその例外を投げる。`findById` と `MediaDerivativesRepository.findById` は `null`、`StorageReservationRepository.findById` は R を返す |
| TC-ports-mediaRepository-115 | 保管の予約の記録が無い | 一つの UnitOfWork で、画像の `insert`、`MediaDerivativesRepository.insert`、`StorageReservationRepository.settle` を呼ぶ | `ConflictError`（`RESERVATION_NOT_FOUND`）になる。`findById` と `MediaDerivativesRepository.findById` は `null` を返す |
| TC-ports-mediaRepository-116 | M1 と M2 のメディアがある。M2 の `expectedVersion` を得た後、別の UnitOfWork が M2 の `save` を確定した | 一つの UnitOfWork で、M1 の `save` と、古い `expectedVersion` での M2 の `save` を呼ぶ | `ConflictError`（`STALE_VERSION`）になる。M1 は保存の前の値のまま |
| TC-ports-mediaRepository-117 | M のメディアがある。使用箇所は無い | 一つの UnitOfWork で、`delete(M, v)` と `ObjectRemovalRepository.enqueue`（`ObjectRemoval.ofMedia(M, …)` の結果）を呼び、正常に終える | `findById(M)` は `null`、`ObjectRemovalRepository.findPending(null, 10)` はその記録を返す |
| TC-ports-mediaRepository-118 | M のメディアがあり、H が M を使う | 一つの UnitOfWork で、`delete(M, v)` と `ObjectRemovalRepository.enqueue` を呼ぶ | `ConflictError`（`MEDIA_IN_USE`）になる。`findPending(null, 10)` は空の列を返す |
| TC-ports-mediaRepository-119 | M の画像と、その派生形式の記録がある | `delete(M, v)` を呼んだ後、`fn` が例外を投げる | `findById(M)` はメディアを、`MediaDerivativesRepository.findById(M)` は派生形式の記録を返す |
| TC-ports-mediaRepository-120 | W にメディアが 3 件ある | `deleteAll(10)` を呼んだ後、`fn` が例外を投げる | `findPage()` の `count` は 3 |
| TC-ports-mediaRepository-121 | W の区画に `Workspace` を確定していない。M の公開のメディアがある | W の UnitOfWork（条件は `none`）で、`unpublish` の結果を `save` する | 確定する。リポジトリは `Workspace` の状態を見ない |
