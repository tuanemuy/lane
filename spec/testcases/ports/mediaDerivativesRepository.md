# MediaDerivativesRepository

「上限」は、その節のメソッドの契約が定める、一度に渡せる件数・個数、または `limit` の上限を指す。値は、ポートを定めるドメインのファイルが持つ。

契約は [media.md](../../domains/media.md) の「ポート」と、[common.md](../../domains/common.md) の「TransactionalRepository」「WriteScope」「UnitOfWorkProvider」が定める。

## 記法

- W、W1、W2 はワークスペース、M、M1、M2 は `MediaId`。リポジトリは、そのワークスペースの区画のもの。書き込みでは、W の UnitOfWork（`{ kind: "workspace", workspaceId: W, condition: { kind: "none" } }` を渡した `UnitOfWorkProvider.run`）の文脈から、読み取りでは `PartitionReader.workspace(W)` から得る。メソッドは `workspaceId` を取らない。「W に」「W の区画で」は、W の区画のリポジトリを使うことを指す。W1 と W2 を使うケースは、それぞれの区画のリポジトリを使う。区画に `Workspace` を確定しておく必要は無い。「W の区画に何も確定していない」は、W の UnitOfWork で一度も確定していないことを指す
- 派生形式の記録は `MediaDerivatives` の振る舞い（`create`、`applyDefinitions`、`complete`、`fail`、`retry`、`restartStalled`）が返す値で作り、`insert` と `save` で保存する
- 「M の画像と派生形式の記録がある」は、画像の `MediaRepository.insert` と、その `MediaDerivatives` の `insert` を、同じ UnitOfWork で確定した状態
- 前提条件の書き込みは、W の UnitOfWork で呼び、確定済み。操作の書き込みは、W の新しい UnitOfWork で呼ぶ。読み取りは UnitOfWork の外で呼ぶ
- `complete` の結果は、`completed` の `derivatives`
- 「定義を変えた `applyDefinitions`」は、同じキーで定義が異なる候補を渡した `applyDefinitions` を指す。候補は、定義のキーをキーにした `DerivativeTarget.propose` の結果で、世代は `IdGenerator.generate` の値
- 「`ConflictError`（理由）になる」は、メソッドの呼び出しか確定のどちらかでエラーが現れ、`run` がそのエラーを投げ、その UnitOfWork の書き込みが一つも反映されないことを指す
- v は、直前の `findById` が返した `expectedVersion`
- `generating` の `requestedAt` は古い順に r1 < r2 < r3 < r4

## 対象にしない

- 画像の `Media` の `insert` と別の UnitOfWork で、または画像の `Media` が無い状態で `insert` した場合の結果。契約は、同じ UnitOfWork で呼ぶことを呼び出し側に求め、結果を約束しない。このスイートの前提は、どれも画像の `Media` とともに作る
- 保存先の障害と、デコードできない保存値（common.md の共通の規則）

## insert と findById

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaDerivativesRepository-01 | M の派生形式の記録が無い | `findById(M)` を呼ぶ | `null` を返す |
| TC-ports-mediaDerivativesRepository-02 | M のメディアが無い | 一つの UnitOfWork で、画像の `MediaRepository.insert` と、`create` した派生形式の記録の `insert` を呼び、確定の後に `findById(M)` を呼ぶ | `states` が空で、`insert` した値と全フィールドが等しい `entity` と、`expectedVersion` を返す |
| TC-ports-mediaDerivativesRepository-03 | M の画像と、`create` した派生形式の記録がある | 候補を 2 つ渡した `applyDefinitions` の結果を v とともに `save` し、確定の後に `findById(M)` を呼ぶ | 保存した値と全フィールドが等しい `entity`（`source`、各 `generating` の `target`（`definition` と `generation`）・`requestedAt`・`restarts`・`previous` を含む）を返す |
| TC-ports-mediaDerivativesRepository-04 | M の文書が `MediaRepository` にある | `findById(M)` を呼ぶ | `null` を返す |
| TC-ports-mediaDerivativesRepository-05 | M の画像と派生形式の記録がある | 同じ `mediaId` の、内容が異なる派生形式の記録を `insert` する | `ConflictError`（`ALREADY_EXISTS`）になる。`findById(M)` は、先に `insert` した値を返す |
| TC-ports-mediaDerivativesRepository-06 | M のメディアが無い | 並行する二つの UnitOfWork が、それぞれ、ID M の画像の `MediaRepository.insert` と、その派生形式の記録の `insert` を呼ぶ | 一方だけが確定し、もう一方は `ConflictError`（`ALREADY_EXISTS`）になる。`findById(M)` は、確定した側の値を返す |
| TC-ports-mediaDerivativesRepository-07 | M の画像と派生形式の記録が W1 の区画にある | W1 の区画と W2 の区画で、それぞれ `findById(M)` を呼ぶ | W1 の区画は `entity` を返す。W2 の区画は `null` を返す |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaDerivativesRepository-08 | `generating` の派生形式を持つ派生形式の記録がある | `complete` の結果を v とともに `save` し、確定の後に `findById` を呼ぶ | その派生形式が `ready` で、`current` の `definition`・`generation`・`width`・`height`・`generatedAt` が保存した値の `entity` を返す |
| TC-ports-mediaDerivativesRepository-09 | `generating` の派生形式を持つ派生形式の記録がある | `fail` の結果を `save` し、`findById` を呼ぶ | その派生形式が `failed` で、`target`・`failure`・`failedAt`・`previous` が保存した値の `entity` を返す |
| TC-ports-mediaDerivativesRepository-10 | `ready` の派生形式を持つ派生形式の記録がある | 定義を変えた `applyDefinitions` の結果を `save` し、`findById` を呼ぶ | その派生形式が `generating` で、`previous` が前の `current`、`restarts` が 0、`target.generation` が保存した値の `entity` を返す |
| TC-ports-mediaDerivativesRepository-11 | 派生形式を 2 つ持つ派生形式の記録がある | 候補を 1 つにした `applyDefinitions` の結果を `save` し、`findById` を呼ぶ | `states` は、残した 1 つのキーだけを持つ |
| TC-ports-mediaDerivativesRepository-12 | 派生形式を 1 つ持つ派生形式の記録がある | 空の候補を渡した `applyDefinitions` の結果を `save` し、`findById` を呼ぶ | `states` が空の `entity` を返す |
| TC-ports-mediaDerivativesRepository-13 | `restarts` が 0 の `generating` の派生形式を持つ派生形式の記録がある | `restartStalled` の結果を `save` し、`findById` を呼ぶ | その派生形式が `generating` で、`restarts` が 1、`requestedAt` が保存した値、`target` と `previous` が前と同じ `entity` を返す |
| TC-ports-mediaDerivativesRepository-14 | `restarts` が 3 の `generating` の派生形式を持つ派生形式の記録がある | `restartStalled` の結果を `save` し、`findById` を呼ぶ | その派生形式が `failed` で、`failure.reason` が `stalled`、`failedAt`・`target`・`previous` が保存した値の `entity` を返す |
| TC-ports-mediaDerivativesRepository-15 | 派生形式の記録がある。`findById` で `entity` と v を得ている | 読んだ `entity` をそのまま、v で `save` し、確定の後に `findById` を呼ぶ | 確定する。`findById` は、全フィールドが前と等しい `entity` と、v と等しい `expectedVersion` を返す。値と `version` を変えない |
| TC-ports-mediaDerivativesRepository-16 | 派生形式の記録がある。`entity` と v を得た後、別の UnitOfWork が、定義を変えた `applyDefinitions` の結果の `save` を確定した | 一つの UnitOfWork で、先に読んだ `entity` をそのまま v で `save` し、`StorageReservationRepository.settle` を呼ぶ | `ConflictError`（`STALE_VERSION`）になる。保管の予約は残る。`findById` は、別の UnitOfWork が保存した値を返す |
| TC-ports-mediaDerivativesRepository-17 | M の画像と派生形式の記録がある。`entity` と v を得た後、別の UnitOfWork が `MediaRepository.delete(M)` を確定した | 先に読んだ `entity` をそのまま、または変えた値を、v で `save` する | `ConflictError`（`STALE_VERSION`）になる。`findById(M)` は `null` を返す |
| TC-ports-mediaDerivativesRepository-18 | `failed` の派生形式を持つ派生形式の記録がある | `retry` の結果を `save` し、`findById` を呼ぶ | その派生形式が `generating` で、`restarts` が 0 の `entity` を返す |
| TC-ports-mediaDerivativesRepository-19 | 派生形式の記録がある。v を得た後、別の UnitOfWork が `save` を確定した | v とともに `save` する | `ConflictError`（`STALE_VERSION`）になる。`findById` は、先に確定した値を返す |
| TC-ports-mediaDerivativesRepository-20 | `generating` の派生形式を 2 つ持つ派生形式の記録がある。二つの処理が、同じ `expectedVersion` を得た | 並行する二つの UnitOfWork が、一方は 1 つ目の `complete` の結果を、もう一方は 2 つ目の `complete` の結果を `save` する | 一方だけが確定し、もう一方は `ConflictError`（`STALE_VERSION`）になる。`findById` は、確定した側の値を返す |
| TC-ports-mediaDerivativesRepository-21 | 派生形式の記録を `save` 済み | `findById` で得た新しい `expectedVersion` とともに、次の変更を `save` する | 確定する。`findById` は次の変更の値を返す |
| TC-ports-mediaDerivativesRepository-22 | M の画像と派生形式の記録がある。二つの処理が、それぞれの `expectedVersion` を得た | 並行する二つの UnitOfWork の、一方が派生形式の記録の `save` を、もう一方が、M の公開使用の記録と `MediaRepository.publishMany([M], t)` を呼ぶ | 両方が確定する。`Media` の保存とは競合しない |
| TC-ports-mediaDerivativesRepository-23 | M の画像と派生形式の記録、その管理情報がある。二つの処理が、それぞれの `expectedVersion` を得た | 並行する二つの UnitOfWork の、一方が派生形式の記録の `save` を、もう一方が `MediaProfileRepository.save`（`MediaProfile.describe` の結果）を呼ぶ | 両方が確定する。`MediaProfile` の保存とは競合しない |
| TC-ports-mediaDerivativesRepository-25 | M の画像と派生形式の記録がある。v を得た後、`MediaRepository.deleteAll` を確定した | M の派生形式の記録を v とともに `save` する | `ConflictError`（`STALE_VERSION`）になる |
| TC-ports-mediaDerivativesRepository-26 | M の画像と派生形式の記録がある。それぞれの `expectedVersion` を得た | 並行する二つの UnitOfWork の、一方が派生形式の記録の `save` を、もう一方が `MediaRepository.delete` を呼ぶ | 削除の側は確定する。保存の側は、確定するか、`ConflictError`（`STALE_VERSION`）になるかのどちらか。どちらでも、`findById(M)` は `null` を返す |

## findByIds

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaDerivativesRepository-27 | W に M1、M2、M3 の画像と派生形式の記録がある | `findByIds([M1, M3])` を呼ぶ | M1 と M3 の派生形式の記録の 2 件を返す。順序は問わない |
| TC-ports-mediaDerivativesRepository-28 | W に M1 の画像と派生形式の記録がある | `findByIds([])` を呼ぶ | 空の列を返す |
| TC-ports-mediaDerivativesRepository-29 | W に M1 の画像と派生形式の記録がある。M9 のメディアは無い | `findByIds([M1, M9])` を呼ぶ | M1 の 1 件だけを返す |
| TC-ports-mediaDerivativesRepository-30 | W に、M1 の画像と派生形式の記録、M2 の文書がある | `findByIds([M1, M2])` を呼ぶ | M1 の 1 件だけを返す。文書の ID は結果に含めない |
| TC-ports-mediaDerivativesRepository-31 | W1 の区画に M1、W2 の区画に M2 の画像と派生形式の記録がある | W1 の区画で `findByIds([M1, M2])` を呼ぶ | M1 の 1 件だけを返す |
| TC-ports-mediaDerivativesRepository-32 | W の区画に何も確定していない | `findByIds([M1])` を呼ぶ | 空の列を返す |
| TC-ports-mediaDerivativesRepository-33 | W に M1 と M2 の画像と派生形式の記録がある | `findByIds([M1, M1, M2])` を呼ぶ | 2 件を返す。M1 を一度だけ返す |
| TC-ports-mediaDerivativesRepository-34 | W に画像と派生形式の記録が、上限の件数だけある | その全件の ID を渡して `findByIds` を呼ぶ | 上限の件数を返す |
| TC-ports-mediaDerivativesRepository-35 | W に画像と派生形式の記録が、上限の件数だけある | その全件の ID に、同じ ID の重複を 1 個加えた上限より 1 個多い数を渡して呼ぶ | 上限の件数を返す |
| TC-ports-mediaDerivativesRepository-36 | W に画像と派生形式の記録が、上限より 1 件多くある | その全件の ID を渡して `findByIds` を呼ぶ | `SystemError`（`INVALID_QUERY`）を投げる。上限の件数に切り詰めない |
| TC-ports-mediaDerivativesRepository-37 | W に、`ready`、`generating`（`previous` あり）、`failed`（`previous` なし）の派生形式を持つ派生形式の記録がある | `findByIds` の結果を `MediaDerivatives.generated` に渡す | `ready` の `current` と、`generating` の `previous` を返す。保存した値に合う |

## scan

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaDerivativesRepository-38 | W の派生形式の記録が無い | `scan(null, 10)` を呼ぶ | 空の列を返す |
| TC-ports-mediaDerivativesRepository-39 | W の区画に何も確定していない | `scan(null, 10)` を呼ぶ | 空の列を返す |
| TC-ports-mediaDerivativesRepository-40 | W に画像と派生形式の記録が 1 件ある | `scan(null, 10)` を呼ぶ | その 1 件を返す |
| TC-ports-mediaDerivativesRepository-41 | W に、画像と派生形式の記録が 3 件、文書が 2 件ある | `scan(null, 10)` を呼ぶ | 派生形式の記録の 3 件だけを、`mediaId` のコードポイントの昇順で返す |
| TC-ports-mediaDerivativesRepository-42 | W に、派生形式の状態が `generating`、`ready`、`failed` の派生形式の記録と、`states` が空の派生形式の記録がある | `scan(null, 10)` を呼ぶ | 4 件とも返す。状態で絞らない |
| TC-ports-mediaDerivativesRepository-43 | W1 の区画に 2 件、W2 の区画に 3 件の画像と派生形式の記録がある | W1 の区画で `scan(null, 10)` を呼ぶ | W1 の 2 件だけを返す |
| TC-ports-mediaDerivativesRepository-44 | W に画像と派生形式の記録が 5 件ある。別の前提では 4 件ある | `scan(null, 2)` を呼び、返った最後の `mediaId` を `after` に渡して、返す件数が 2 より少なくなるまで繰り返す | 返す件数は順に 2、2、1（4 件では 2、2、0）。通して `mediaId` の昇順で、全件を重複も欠落も無く返す |
| TC-ports-mediaDerivativesRepository-46 | W に画像と派生形式の記録が 3 件ある。`mediaId` の昇順で 2 番目を X とする | `scan(X, 10)` を呼ぶ | 3 番目の 1 件だけを返す。X の派生形式の記録は返さない |
| TC-ports-mediaDerivativesRepository-47 | W に画像と派生形式の記録が 3 件ある。`mediaId` の昇順で最後を Z とする | `scan(Z, 10)` を呼ぶ | 空の列を返す |
| TC-ports-mediaDerivativesRepository-48 | W に画像と派生形式の記録が 3 件ある。どの `mediaId` とも異なり、昇順で 1 番目と 2 番目の間に入る ID を X とする | `scan(X, 10)` を呼ぶ | 2 番目と 3 番目の 2 件を返す |
| TC-ports-mediaDerivativesRepository-49 | W に画像と派生形式の記録が、上限の件数だけある | `scan`（null、上限の `limit`）を呼ぶ | 上限の件数を返す |
| TC-ports-mediaDerivativesRepository-50 | W に画像と派生形式の記録が、上限より 1 件多くある | `scan`（null、上限の `limit`）を呼び、返った最後の `mediaId` を `after` に渡してもう一度呼ぶ | 上限の件数、続けて 1 件を返す |
| TC-ports-mediaDerivativesRepository-51 | W に画像と派生形式の記録が 3 件ある | `scan(null, 1)` を呼ぶ | `mediaId` の昇順で先頭の 1 件を返す |
| TC-ports-mediaDerivativesRepository-52 | W に画像と派生形式の記録が 3 件ある | `limit` に、範囲外の値（0、上限より 1 大きい値）を渡して `scan` を呼ぶ | どちらも `SystemError`（`INVALID_QUERY`）を投げる |
| TC-ports-mediaDerivativesRepository-54 | W に画像と派生形式の記録が 3 件ある。`limit` 1 で 1 件目を読んだ後、`mediaId` の昇順で 3 番目の派生形式の記録に `applyDefinitions` の結果を `save` した | 1 件目の `mediaId` を `after` に渡して続きを読む | 2 番目と、保存した後の値の 3 番目を返す |
| TC-ports-mediaDerivativesRepository-55 | W に画像と派生形式の記録が 2 件ある。`limit` 1 で 1 件目を読んだ後、`mediaId` の昇順で最後になる画像と派生形式の記録を `insert` した | 1 件目の `mediaId` を `after` に渡して続きを読む | 後から加わった派生形式の記録を含めて返す |
| TC-ports-mediaDerivativesRepository-56 | W に画像と派生形式の記録が 3 件ある。`limit` 1 で 1 件目を読んだ後、2 番目のメディアを `MediaRepository.delete` で削除した | 1 件目の `mediaId` を `after` に渡して続きを読む | 3 番目の 1 件だけを返す |
| TC-ports-mediaDerivativesRepository-57 | W に画像と派生形式の記録が 1 件ある | `scan(null, 10)` が返した `expectedVersion` で、`applyDefinitions` の結果を `save` する | 確定する。`scan` は、`findById` と同じ `expectedVersion` を返す |
| TC-ports-mediaDerivativesRepository-58 | W に画像と派生形式の記録が 1 件ある。`scan(null, 10)` で読んだ後、別の UnitOfWork がその派生形式の記録の `save` を確定した | `scan` が返した `expectedVersion` で `save` する | `ConflictError`（`STALE_VERSION`）になる |

## findStalled

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaDerivativesRepository-59 | 派生形式の記録が無い | `findStalled(r4, 10)` を呼ぶ | 空の列を返す |
| TC-ports-mediaDerivativesRepository-60 | `requestedAt` が r1 の `generating` の派生形式を持つ派生形式の記録が 1 件ある | `findStalled(r2, 10)` を呼ぶ | その 1 件を返す |
| TC-ports-mediaDerivativesRepository-61 | `requestedAt` が r2 の `generating` の派生形式を持つ派生形式の記録がある | `findStalled(r2, 10)` を呼ぶ | その派生形式の記録を返す。`requestedBefore` と同じ日時を含む |
| TC-ports-mediaDerivativesRepository-62 | `requestedAt` が r3 の `generating` の派生形式を持つ派生形式の記録がある | `findStalled(r2, 10)` を呼ぶ | 空の列を返す |
| TC-ports-mediaDerivativesRepository-63 | r1 に作り、派生形式がすべて `ready` の派生形式の記録、すべて `failed` の派生形式の記録、`states` が空の派生形式の記録がある | `findStalled(r4, 10)` を呼ぶ | 空の列を返す |
| TC-ports-mediaDerivativesRepository-64 | `ready` の派生形式と、`requestedAt` が r1 の `generating` の派生形式を持つ派生形式の記録がある | `findStalled(r2, 10)` を呼ぶ | その派生形式の記録を返す |
| TC-ports-mediaDerivativesRepository-65 | `requestedAt` が r1 の `generating` と、r3 の `generating` を持つ派生形式の記録がある | `findStalled(r2, 10)` を呼ぶ | その派生形式の記録を返す。一つ以上が該当すれば返す |
| TC-ports-mediaDerivativesRepository-66 | W1 の区画と W2 の区画に、`requestedAt` が r1 の `generating` を持つ派生形式の記録が 1 件ずつある | W1 の区画で `findStalled(r2, 10)` を呼ぶ | W1 の 1 件だけを返す |
| TC-ports-mediaDerivativesRepository-67 | W の区画に何も確定していない | `findStalled(r4, 10)` を呼ぶ | 空の列を返す |
| TC-ports-mediaDerivativesRepository-68 | 区画に `Workspace` を確定していない W に、`requestedAt` が r1 の `generating` を持つ派生形式の記録がある | `findStalled(r2, 10)` を呼ぶ | その派生形式の記録を返す。ワークスペースの状態を見ない |
| TC-ports-mediaDerivativesRepository-69 | `generating` の `requestedAt` が r3 の派生形式の記録 A、r1 の派生形式の記録 B、r2 の派生形式の記録 C がある | `findStalled(r4, 10)` を呼ぶ | B、C、A の順で返す |
| TC-ports-mediaDerivativesRepository-70 | `generating` の `requestedAt` が r2 と r3 の派生形式の記録 A、r1 と r4 の派生形式の記録 B がある | `findStalled(r4, 10)` を呼ぶ | B、A の順で返す。派生形式の記録ごとの最も古い値で並べる |
| TC-ports-mediaDerivativesRepository-71 | `generating` の `requestedAt` が同じ r1 で、`mediaId` が異なる派生形式の記録が 3 件ある | `findStalled(r2, 10)` を呼ぶ | `mediaId` のコードポイントの昇順で返す |
| TC-ports-mediaDerivativesRepository-72 | `generating` の `requestedAt` が r1、r2、r3 の派生形式の記録が 1 件ずつある | `findStalled(r4, 2)` を呼ぶ | r1 と r2 の 2 件を返す |
| TC-ports-mediaDerivativesRepository-73 | `restarts` が 3 の `generating`（`requestedAt` は r1）を持つ派生形式の記録がある | `findStalled(r2, 10)` を呼ぶ | その派生形式の記録を返す。`restarts` で絞らない |
| TC-ports-mediaDerivativesRepository-74 | `generating`（`requestedAt` は r1）を持つ派生形式の記録に、`restartStalled` で `requestedAt` を r3 にした結果を `save` 済み | `findStalled(r2, 10)` を呼ぶ | 空の列を返す |
| TC-ports-mediaDerivativesRepository-75 | `restarts` が 3 の `generating`（`requestedAt` は r1）を持つ派生形式の記録に、`restartStalled` で `failed`（`stalled`）にした結果を `save` 済み | `findStalled(r4, 10)` を呼ぶ | 空の列を返す |
| TC-ports-mediaDerivativesRepository-76 | `generating`（`requestedAt` は r1）を持つ派生形式の記録に、`complete` の結果を `save` 済み | `findStalled(r2, 10)` を呼ぶ | 空の列を返す |
| TC-ports-mediaDerivativesRepository-77 | `generating`（`requestedAt` は r1）を持つ派生形式の記録がある。そのメディアを `MediaRepository.delete` で削除済み | `findStalled(r2, 10)` を呼ぶ | 空の列を返す |
| TC-ports-mediaDerivativesRepository-78 | `generating` の `requestedAt` が r1 の派生形式の記録が、上限の件数だけある | `findStalled`（r2、上限の `limit`）を呼ぶ | 上限の件数を返す |
| TC-ports-mediaDerivativesRepository-79 | `generating` の `requestedAt` が r1 の派生形式の記録が、上限より 1 件多くある | `findStalled`（r2、上限の `limit`）を呼ぶ | `mediaId` の昇順で先頭から上限の件数を返す |
| TC-ports-mediaDerivativesRepository-80 | `generating` の `requestedAt` が r1 の派生形式の記録が 1 件ある | `findStalled(r2, 1)` を呼ぶ | その 1 件を返す |
| TC-ports-mediaDerivativesRepository-81 | `generating` の `requestedAt` が r1 の派生形式の記録が 1 件ある | `limit` に、範囲外の値（0、上限より 1 大きい値）を渡して `findStalled(r2, limit)` を呼ぶ | どちらも `SystemError`（`INVALID_QUERY`）を投げる |
| TC-ports-mediaDerivativesRepository-83 | `generating` の `requestedAt` が r1 の派生形式の記録が 1 件ある | `findStalled(r2, 10)` が返した `expectedVersion` で、`restartStalled` の結果を `save` する | 確定する。`findStalled` は、`findById` と同じ `expectedVersion` を返す |
| TC-ports-mediaDerivativesRepository-84 | `generating` の `requestedAt` が r1 の派生形式の記録が 1 件ある。`findStalled(r2, 10)` で読んだ後、別の UnitOfWork がその派生形式の記録の `save` を確定した | `findStalled` が返した `expectedVersion` で `save` する | `ConflictError`（`STALE_VERSION`）になる |

## UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaDerivativesRepository-85 | M のメディアが無い | 一つの UnitOfWork で、画像の `MediaRepository.insert` と派生形式の記録の `insert` を呼んだ後、`fn` が例外を投げる | `run` はその例外を投げる。`findById(M)` と `MediaRepository.findById(M)` は `null` を返す |
| TC-ports-mediaDerivativesRepository-86 | M の画像と派生形式の記録がある | 一つの UnitOfWork で、`complete` の結果の `save` と、`ObjectRemovalRepository.enqueue`（`ObjectRemoval.ofObjects` に `superseded` を渡した結果）を呼び、正常に終える | `findById(M)` は保存した値を、`ObjectRemovalRepository.findPending(null, 10)` は記録を返す |
| TC-ports-mediaDerivativesRepository-87 | 同上 | 一つの UnitOfWork で、同じ二つを呼んだ後、`fn` が例外を投げる | `findById(M)` は保存の前の値と、前と等しい `expectedVersion` を返す。`findPending(null, 10)` は空の列を返す |
| TC-ports-mediaDerivativesRepository-88 | M の画像と派生形式の記録がある。v を得た後、別の UnitOfWork が `save` を確定した | 一つの UnitOfWork で、v での `save` と、`ObjectRemovalRepository.enqueue` を呼ぶ | `ConflictError`（`STALE_VERSION`）になる。`findPending(null, 10)` は空の列を返す |
