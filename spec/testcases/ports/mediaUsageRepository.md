# MediaUsageRepository

「上限」は、その節のメソッドの契約が定める、一度に渡せる件数・個数、または `limit` の上限を指す。値は、ポートを定めるドメインのファイルが持つ。

契約は [media.md](../../domains/media.md) の「ポート」の `MediaUsageRepository` と `MediaRepository`、[common.md](../../domains/common.md) の「TransactionalRepository」の共通の規則、「Pagination と PaginationResult」「WriteScope」「UnitOfWorkProvider」が定める。前提と確認には、Media のポートだけを使う。

## 記法

- W、W1、W2 はワークスペース、M、M1、M2 は `MediaId`、H、H1、H2 は `holderId`、J、J1、J2 は `stagingId`（空でない文字列）。リポジトリは、そのワークスペースの区画のもの。書き込みでは、W の UnitOfWork（`{ kind: "workspace", workspaceId: W, condition: { kind: "none" } }` を渡した `UnitOfWorkProvider.run`）の文脈から、読み取りでは `PartitionReader.workspace(W)` から得る。メソッドは `workspaceId` を取らない。「W に」「W の区画で」は、W の区画のリポジトリを使うことを指す。W1 と W2 を使うケースは、それぞれの区画のリポジトリを使う。区画に `Workspace` を確定しておく必要は無い
- メディアは `Media` の振る舞いが返す値を、`MediaRepository.insert` と `save` で保存して作る
- 保護は、`MediaRepository.delete(M, v)` が `MEDIA_IN_USE` になるか、`Media.unpublish` の結果の `MediaRepository.save` が `MEDIA_IN_PUBLIC_USE` になるかで確かめる。「M は削除できる」は、`delete(M, v)` が確定することを指す。「M は削除できない」は、`ConflictError`（`MEDIA_IN_USE`）になることを指す。「M は非公開にできない」は、`ConflictError`（`MEDIA_IN_PUBLIC_USE`）になることを指す
- 前提条件の書き込みは、W の UnitOfWork で呼び、確定済み。操作の書き込みは、W の新しい UnitOfWork で呼ぶ。読み取りは UnitOfWork の外で呼ぶ
- 「`ConflictError`（理由）になる」は、メソッドの呼び出しか確定のどちらかでエラーが現れ、`run` がそのエラーを投げ、その UnitOfWork の書き込みが一つも反映されないことを指す
- v は、直前の `MediaRepository.findById` が返した `expectedVersion`
- `use(M)` は `{ mediaId: M, public: false }`、`publicUse(M)` は `{ mediaId: M, public: true }`。`H: [...]` は `{ holderId: H, uses: [...] }`
- `replace(H, [...])` と書いた場合は `replaceMany([H: [...]])`、`clear(H)` と書いた場合は `replaceMany([H: []])`
- `usages(M)` と書いた場合は、`findPageByMedia(M, { page: 1, limit: 100 })`。`status(…)` と書いた場合は、`findStatusByMedia([…])`。結果は `MediaUsageStatus`（`unused`、`in_use`、`in_public_use`）

## 対象にしない

- 空の文字列の `holderId` と `stagingId` を渡した場合の結果。契約は、空でない文字列を使う側に求め、結果を約束しない
- 保存先の障害と、デコードできない保存値（common.md の共通の規則）

## replaceMany

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaUsageRepository-01 | W に M の非公開のメディアがある | `replace(H, [use(M)])` を呼ぶ | 確定する。M は削除できない |
| TC-ports-mediaUsageRepository-02 | W に M の公開のメディアがある | `replace(H, [publicUse(M)])` を呼ぶ | 確定する。M は非公開にできない。M は削除できない |
| TC-ports-mediaUsageRepository-03 | W に M の公開のメディアがある | `replace(H, [use(M)])` を呼ぶ | 確定する。`unpublish` の結果の `save` は確定する。M は削除できない |
| TC-ports-mediaUsageRepository-04 | W に M1 と M2 のメディアがあり、`replace(H, [use(M1)])` を確定済み | `replace(H, [use(M2)])` を呼ぶ | 確定する。M1 は削除できる。M2 は削除できない |
| TC-ports-mediaUsageRepository-05 | W に M1 と M2 のメディアがある | `replace(H, [use(M1), use(M2)])` を呼ぶ | 確定する。M1 も M2 も削除できない |
| TC-ports-mediaUsageRepository-06 | W に M のメディアがあり、`replace(H, [use(M)])` を確定済み | `replace(H, [])` を呼ぶ | 確定する。M は削除できる。空の `uses` は、その持ち主の使用箇所を消す |
| TC-ports-mediaUsageRepository-07 | H の使用箇所が無い | `replace(H, [])` を呼ぶ | 確定する |
| TC-ports-mediaUsageRepository-08 | W に M のメディアがあり、H1 と H2 が、それぞれ M の使用箇所を持つ | `clear(H1)` を呼ぶ | M は削除できない |
| TC-ports-mediaUsageRepository-09 | W に M のメディアがあり、`UnitOfWorkCapacity.workspaceWrites` を超える数の持ち主が M を使う | そのすべての持ち主を、空の `uses` で、一度の `replaceMany` に渡す | 確定する。M は削除できる |
| TC-ports-mediaUsageRepository-10 | W1 の区画と W2 の区画に、それぞれ M1、M2 のメディアがあり、同じ `holderId` H が、それぞれの使用箇所を持つ | W1 の区画で `clear(H)` を呼ぶ | M1 は削除できる。M2 は削除できない |
| TC-ports-mediaUsageRepository-11 | W に M の公開のメディアがあり、`replace(H, [publicUse(M)])` を確定済み | `clear(H)` を呼び、確定の後に `unpublish` の結果を `save` する | 確定する |
| TC-ports-mediaUsageRepository-12 | W に M の公開のメディアがある | `replace(H, [use(M), publicUse(M)])` を呼ぶ | 確定する。M は非公開にできない。同じ `mediaId` は一つの使用箇所にまとまり、`public` は真 |
| TC-ports-mediaUsageRepository-13 | W に M の公開のメディアがあり、`replace(H, [publicUse(M)])` を確定済み | `replace(H, [use(M)])` を呼ぶ | 確定する。`unpublish` の結果の `save` は確定する |
| TC-ports-mediaUsageRepository-14 | W に M のメディアがあり、`replace(H1, [use(M)])` を確定済み | `replace(H2, [use(M)])` を呼び、確定の後に `clear(H1)` を確定する | M は削除できない。使う側ごとに別の使用箇所 |
| TC-ports-mediaUsageRepository-15 | W に M1、M2、M3 のメディアがあり、`replace(H3, [use(M3)])` を確定済み | 一度の `replaceMany([H1: [use(M1)], H2: [use(M2)]])` を呼ぶ | 確定する。M1、M2、M3 のどれも削除できない。`entries` に無い H3 の使用箇所は変わらない |
| TC-ports-mediaUsageRepository-16 | W に M1 と M2 のメディアがあり、`replace(H1, [use(M1)])` と `replace(H2, [use(M2)])` を確定済み | 一度の `replaceMany([H1: [], H2: [use(M1)]])` を呼ぶ | 確定する。M1 は削除できない。M2 は削除できる。持ち主ごとに置き換える |
| TC-ports-mediaUsageRepository-17 | W に M1 と M2 のメディアがある | 一度の `replaceMany([H: [use(M1)], H: [use(M2)]])` を呼ぶ | 確定する。M1 は削除できる。M2 は削除できない。同じ `holderId` は、列の最後の要素を使う |
| TC-ports-mediaUsageRepository-18 | W に M のメディアがあり、`replace(H, [use(M)])` を確定済み | `replaceMany([])` を呼ぶ | 確定する。M は削除できない。何も変えない |
| TC-ports-mediaUsageRepository-19 | W に、`UnitOfWorkCapacity.workspaceWrites` を超える数のメディアがある | その全件の `use` を、一つの持ち主の `uses` にして `replaceMany` に渡す | 確定する。どのメディアも削除できない |
| TC-ports-mediaUsageRepository-20 | W に、`UnitOfWorkCapacity.workspaceWrites` を超える数のメディアがある | メディアと同じ数の持ち主が、それぞれ一件の `use` を持つ `entries` を、一度の `replaceMany` に渡す | 確定する。どのメディアも削除できない |
| TC-ports-mediaUsageRepository-21 | W に M9 のメディアが無い | `replace(H, [use(M9)])` を呼ぶ | `ConflictError`（`MEDIA_NOT_FOUND`）になる |
| TC-ports-mediaUsageRepository-22 | W1 の区画に M のメディアがある | W2 の区画で `replace(H, [use(M)])` を呼ぶ | `ConflictError`（`MEDIA_NOT_FOUND`）になる。W1 の区画の M は削除できる |
| TC-ports-mediaUsageRepository-23 | W に M1 のメディアがある。M9 のメディアは無い | `replace(H, [use(M1), use(M9)])` を呼ぶ | `ConflictError`（`MEDIA_NOT_FOUND`）になる。M1 は削除できる |
| TC-ports-mediaUsageRepository-24 | W に M1 と M2 のメディアがあり、`replace(H, [use(M1)])` を確定済み。M9 のメディアは無い | `replace(H, [use(M2), use(M9)])` を呼ぶ | `ConflictError`（`MEDIA_NOT_FOUND`）になる。M1 は削除できない。M2 は削除できる。前の使用箇所のまま |
| TC-ports-mediaUsageRepository-25 | W に M1 のメディアがある。M9 のメディアは無い | 一度の `replaceMany([H1: [use(M1)], H2: [use(M9)]])` を呼ぶ | `ConflictError`（`MEDIA_NOT_FOUND`）になる。M1 は削除できる。一つでも拒否されれば、どの持ち主も置き換えない |
| TC-ports-mediaUsageRepository-26 | W に M の非公開のメディアがある | `replace(H, [publicUse(M)])` を呼ぶ | `ConflictError`（`MEDIA_NOT_PUBLIC`）になる。M は削除できる |
| TC-ports-mediaUsageRepository-27 | W に M の非公開のメディアがある | 一つの UnitOfWork で、`MediaRepository.publishMany([M], t)` と、`replace(H, [publicUse(M)])` を呼ぶ | 確定する。メディアは `public` で、非公開にできない。判定は、確定後の状態に対して行う |
| TC-ports-mediaUsageRepository-28 | W に M の公開のメディアがある | 一つの UnitOfWork で、`Media.unpublish` の結果の `MediaRepository.save` と、`replace(H, [publicUse(M)])` を呼ぶ | `ConflictError` になる。理由は `MEDIA_NOT_PUBLIC` または `MEDIA_IN_PUBLIC_USE`。メディアは `public` のまま、M は削除できる |
| TC-ports-mediaUsageRepository-29 | W に M のメディアがある | 一つの UnitOfWork で、`MediaRepository.delete(M, v)` と、`replace(H, [use(M)])` を呼ぶ | `ConflictError` になる。理由は `MEDIA_NOT_FOUND` または `MEDIA_IN_USE`。`MediaRepository.findById(M)` はメディアを返す |
| TC-ports-mediaUsageRepository-30 | W1 の区画と W2 の区画に、それぞれ M1、M2 のメディアがある。W1 の区画で `replace(H, [use(M1)])` を確定済み | W2 の区画で、同じ `holderId` の `replace(H, [use(M2)])` を呼ぶ | 確定する。M1 も M2 も削除できない。区画が違えば別の使う側 |

## stage

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaUsageRepository-31 | W に M のメディアがある。使用箇所は無い | `stage(J, [H: [use(M)]])` を呼び、確定の後に `usages(M)` と `status(M)` を呼ぶ | 確定する。`usages(M)` は空で `count` は 0、`status(M)` は `unused`。M は削除できる。準備領域の内容は使用箇所でない |
| TC-ports-mediaUsageRepository-32 | W に M の公開のメディアがある。使用箇所は無い | `stage(J, [H: [publicUse(M)]])` を呼び、確定の後に `unpublish` の結果を `save` する | どちらも確定する。準備領域の内容は、非公開化を妨げない |
| TC-ports-mediaUsageRepository-33 | W に M9 のメディアが無い | `stage(J, [H: [use(M9)]])` を呼ぶ | 確定する。メディアの存在を確かめない |
| TC-ports-mediaUsageRepository-34 | W に M の非公開のメディアがある | `stage(J, [H: [publicUse(M)]])` を呼ぶ | 確定する。公開範囲を確かめない |
| TC-ports-mediaUsageRepository-35 | W に M1 のメディアがあり、`replace(H, [use(M1)])` を確定済み | `stage(J, [H: []])` を呼ぶ | 確定する。M1 は削除できない。使用箇所を変えない |
| TC-ports-mediaUsageRepository-36 | W に、`UnitOfWorkCapacity.workspaceWrites` を超える数のメディアがある | メディアと同じ数の持ち主が、それぞれ一件の `use` を持つ `entries` を、一度の `stage` に渡す | 確定する |
| TC-ports-mediaUsageRepository-37 | 準備領域が無い | `stage(J, [])` を呼ぶ | 確定する |
| TC-ports-mediaUsageRepository-38 | W に M のメディアがある | `stagingId` を `apply:7/事前確認` にして、`stage`、`promoteStaged` を順に確定する | 確定する。M は削除できない。`stagingId` の中身を解釈しない |
| TC-ports-mediaUsageRepository-39 | W に M1 と M2 のメディアがある | 一度の `stage(J, [H: [use(M1)], H: [use(M2)]])` を呼び、確定の後に `promoteStaged(J)` を確定する | M1 は削除できる。M2 は削除できない。同じ `holderId` は、列の最後の要素を使う |
| TC-ports-mediaUsageRepository-40 | W に M のメディアがある。`stage(J, [H: [use(M)]])` を確定済み | 同じ `stage(J, [H: [use(M)]])` をもう一度確定し、`promoteStaged(J)` を確定した後に `usages(M)` を呼ぶ | `items` は H の 1 件、`count` は 1。繰り返しても結果は変わらない |
| TC-ports-mediaUsageRepository-41 | W に M のメディアがある。`stage(J, [H: [use(M)]])` を確定済み | `replace(H, [])` を確定し、その後に `promoteStaged(J)` を確定する | M は削除できない。`replaceMany` は、準備領域の内容を変えない |

## promoteStaged

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaUsageRepository-42 | W に M のメディアがある。`stage(J, [H: [use(M)]])` を確定済み | `promoteStaged(J)` を呼び、確定の後に `usages(M)` を呼ぶ | 確定する。`usages(M)` は `{ holderId: H, public: false }` の 1 件。M は削除できない |
| TC-ports-mediaUsageRepository-43 | W に M1 と M2 のメディアがあり、`replace(H, [use(M1)])` を確定済み。`stage(J, [H: [use(M2)]])` を確定済み | `promoteStaged(J)` を呼ぶ | 確定する。M1 は削除できる。M2 は削除できない。持ち主の使用箇所を、準備領域の内容に置き換える |
| TC-ports-mediaUsageRepository-44 | W に M のメディアがあり、`replace(H, [use(M)])` を確定済み。`stage(J, [H: []])` を確定済み | `promoteStaged(J)` を呼ぶ | 確定する。M は削除できる。空の `uses` の持ち主の使用箇所を取り除く |
| TC-ports-mediaUsageRepository-45 | W に M1 と M2 のメディアがあり、`replace(H1, [use(M1)])` を確定済み。`stage(J, [H2: [use(M2)]])` を確定済み | `promoteStaged(J)` を呼ぶ | 確定する。M1 も M2 も削除できない。準備領域に無い H1 の使用箇所は変わらない |
| TC-ports-mediaUsageRepository-46 | W に M1 と M2 のメディアがある。`stage(J, [H: [use(M1)]])` の後に `stage(J, [H: [use(M2)]])` を確定済み | `promoteStaged(J)` を呼ぶ | 確定する。M1 は削除できる。M2 は削除できない。同じ `stagingId` の同じ `holderId` は、後に書いた内容に置き換わっている |
| TC-ports-mediaUsageRepository-47 | W に M1 と M2 のメディアがある。`stage(J, [H1: [use(M1)]])` と、別の UnitOfWork の `stage(J, [H2: [use(M2)]])` を確定済み | `promoteStaged(J)` を呼ぶ | 確定する。M1 も M2 も削除できない。複数回に分けて書いた準備領域を、一回で切り替える |
| TC-ports-mediaUsageRepository-48 | W に M の公開のメディアがある。`stage(J, [H: [use(M), publicUse(M)]])` を確定済み | `promoteStaged(J)` を呼び、確定の後に `usages(M)` を呼ぶ | `{ holderId: H, public: true }` の 1 件。同じ `mediaId` は一つにまとまり、`public` は真 |
| TC-ports-mediaUsageRepository-49 | W に M1 と M2 のメディアがある。`stage(J1, [H: [use(M1)]])` と `stage(J2, [H: [use(M2)]])` を確定済み | `promoteStaged(J1)` を呼ぶ | 確定する。M1 は削除できない。M2 は削除できる。別の `stagingId` の準備領域は影響しない |
| TC-ports-mediaUsageRepository-50 | W に M のメディアがある。`stage(J, [H: [use(M)]])` と `promoteStaged(J)` を確定済み。その後に `clear(H)` を確定済み | もう一度 `promoteStaged(J)` を呼ぶ | 確定する。M は削除できる。切り替えた準備領域は無くなっている |
| TC-ports-mediaUsageRepository-51 | 準備領域が無い | `promoteStaged(J)` を呼ぶ | 確定する。何も変えない |
| TC-ports-mediaUsageRepository-52 | W に M9 のメディアが無い。`stage(J, [H: [use(M9)]])` を確定済み | `promoteStaged(J)` を呼ぶ | `ConflictError`（`MEDIA_NOT_FOUND`）になる |
| TC-ports-mediaUsageRepository-53 | W に M のメディアがある。`stage(J, [H: [use(M)]])` を確定済み。その後に `MediaRepository.delete(M, v)` を確定済み | `promoteStaged(J)` を呼ぶ | `ConflictError`（`MEDIA_NOT_FOUND`）になる |
| TC-ports-mediaUsageRepository-54 | W に M の公開のメディアがある。`stage(J, [H: [publicUse(M)]])` を確定済み。その後に `unpublish` の結果の `save` を確定済み | `promoteStaged(J)` を呼ぶ | `ConflictError`（`MEDIA_NOT_PUBLIC`）になる。M は削除できる |
| TC-ports-mediaUsageRepository-55 | W に M1 のメディアがあり、`replace(H1, [use(M1)])` を確定済み。M9 のメディアは無い。`stage(J, [H1: [], H2: [use(M9)]])` を確定済み | `promoteStaged(J)` を呼ぶ | `ConflictError`（`MEDIA_NOT_FOUND`）になる。M1 は削除できない。どの持ち主も切り替えない |
| TC-ports-mediaUsageRepository-56 | W に M の非公開のメディアがある。`stage(J, [H: [publicUse(M)]])` を確定済み | 一つの UnitOfWork で、`MediaRepository.publishMany([M], t)` と、`promoteStaged(J)` を呼ぶ | 確定する。M は非公開にできない。判定は、確定後の状態に対して行う |
| TC-ports-mediaUsageRepository-57 | W に M のメディアがある。`UnitOfWorkCapacity.workspaceWrites` を超える数の持ち主が M を使う内容を、`stage(J, …)` で確定済み | `promoteStaged(J)` を呼び、確定の後に `usages(M)` を呼ぶ | 確定する。`count` は、持ち主の数 |

## discardStaged

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaUsageRepository-58 | W に M のメディアがある。`stage(J, [H: [use(M)]])` を確定済み | `discardStaged(J)` を呼び、確定の後に `promoteStaged(J)` を確定する | どちらも確定する。M は削除できる。破棄した準備領域は、何も書いていない準備領域と同じ |
| TC-ports-mediaUsageRepository-59 | W に M1 のメディアがあり、`replace(H, [use(M1)])` を確定済み。`stage(J, [H: []])` を確定済み | `discardStaged(J)` を呼ぶ | 確定する。M1 は削除できない。使用箇所を変えない |
| TC-ports-mediaUsageRepository-60 | 準備領域が無い | `discardStaged(J)` を呼ぶ | 確定する |
| TC-ports-mediaUsageRepository-61 | W に M1 と M2 のメディアがある。`stage(J1, [H: [use(M1)]])` と `stage(J2, [H: [use(M2)]])` を確定済み | `discardStaged(J1)` を呼び、確定の後に `promoteStaged(J2)` を確定する | M1 は削除できる。M2 は削除できない。別の `stagingId` の準備領域を破棄しない |
| TC-ports-mediaUsageRepository-62 | W に M1 と M2 のメディアがある。`stage(J, [H: [use(M1)]])` と `discardStaged(J)` を確定済み | `stage(J, [H: [use(M2)]])` と、別の UnitOfWork の `promoteStaged(J)` を確定する | M1 は削除できる。M2 は削除できない。破棄した `stagingId` に、新しく書ける |

## findPageByMedia

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaUsageRepository-63 | W に M のメディアがある。使用箇所は無い | `usages(M)` を呼ぶ | `items` は空、`count` は 0 |
| TC-ports-mediaUsageRepository-64 | W に M9 のメディアが無い | `usages(M9)` を呼ぶ | `items` は空、`count` は 0 |
| TC-ports-mediaUsageRepository-65 | W の区画に何も確定していない | `usages(M)` を呼ぶ | `items` は空、`count` は 0 |
| TC-ports-mediaUsageRepository-66 | W に M のメディアがあり、`replace(H, [use(M)])` を確定済み | `usages(M)` を呼ぶ | `items` は `{ holderId: H, public: false }` の 1 件、`count` は 1 |
| TC-ports-mediaUsageRepository-67 | W に M の公開のメディアがあり、`replace(H, [publicUse(M)])` を確定済み | `usages(M)` を呼ぶ | `items` は `{ holderId: H, public: true }` の 1 件 |
| TC-ports-mediaUsageRepository-68 | W に M の公開のメディアがあり、`replace(H, [use(M), publicUse(M)])` を確定済み | `usages(M)` を呼ぶ | `items` は `{ holderId: H, public: true }` の 1 件、`count` は 1 |
| TC-ports-mediaUsageRepository-69 | W に M の公開のメディアがあり、H1 が `publicUse(M)` を、H2 が `use(M)` を記録済み | `usages(M)` を呼ぶ | `items` は 2 件。H1 は `public` が真、H2 は偽。`count` は 2 |
| TC-ports-mediaUsageRepository-70 | W に M のメディアがあり、`holderId` が `b`、`a`、`c` の使う側が、この順に `use(M)` を記録済み | `usages(M)` を呼ぶ | `items` は `holderId` が `a`、`b`、`c` の順 |
| TC-ports-mediaUsageRepository-71 | W に M のメディアがあり、`holderId` が `content:C1/draft` と `版-7` の使う側が `use(M)` を記録済み | `usages(M)` を呼ぶ | 渡した文字列と等しい `holderId` を、コードポイントの昇順で返す。中身を解釈しない |
| TC-ports-mediaUsageRepository-72 | W に M1 と M2 のメディアがあり、H1 が M1 を、H2 が M2 を使う | `usages(M1)` を呼ぶ | `items` は H1 の 1 件だけ、`count` は 1 |
| TC-ports-mediaUsageRepository-73 | W に M1 と M2 のメディアがあり、`replace(H, [use(M1)])` の後に `replace(H, [use(M2)])` を確定済み | `usages(M1)` と `usages(M2)` を呼ぶ | M1 は空で `count` は 0。M2 は H の 1 件 |
| TC-ports-mediaUsageRepository-74 | W に M の公開のメディアがあり、`replace(H, [publicUse(M)])` の後に `replace(H, [use(M)])` を確定済み | `usages(M)` を呼ぶ | `items` は `{ holderId: H, public: false }` の 1 件 |
| TC-ports-mediaUsageRepository-75 | W に M のメディアがあり、`replace(H, [use(M)])` の後に `clear(H)` を確定済み | `usages(M)` を呼ぶ | `items` は空、`count` は 0 |
| TC-ports-mediaUsageRepository-76 | W1 の区画と W2 の区画に、それぞれ M1、M2 のメディアがあり、同じ `holderId` H が、それぞれの使用箇所を持つ | W1 の区画で `usages(M1)` と `usages(M2)` を呼ぶ | M1 は H の 1 件。M2 は空で `count` は 0 |
| TC-ports-mediaUsageRepository-77 | W に M のメディアがあり、5 つの使う側が `use(M)` を記録済み | `{ page: 1, limit: 2 }`、`page: 2`、`page: 3`、`page: 4` を順に呼ぶ | `items` は順に 2 件、2 件、1 件、0 件で、通して `holderId` の昇順。重複も欠落も無い。`count` はどれも 5 |

## findStatusByMedia

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaUsageRepository-78 | W に M1、M2、M3 のメディアがある。M3 は公開。H が `use(M2)` と `publicUse(M3)` を記録済み | `status(M1, M2, M3)` を呼ぶ | M1 は `unused`、M2 は `in_use`、M3 は `in_public_use` |
| TC-ports-mediaUsageRepository-79 | W に M の公開のメディアがあり、H1 が `publicUse(M)` を、H2 が `use(M)` を記録済み | `status(M)` を呼ぶ | `in_public_use` |
| TC-ports-mediaUsageRepository-80 | W に M9 のメディアが無い | `status(M9)` を呼ぶ | M9 をキーに持ち、`unused` を返す |
| TC-ports-mediaUsageRepository-81 | W の区画に何も確定していない | `status(M)` を呼ぶ | M をキーに持ち、`unused` を返す |
| TC-ports-mediaUsageRepository-82 | W に M のメディアがある | `findStatusByMedia([])` を呼ぶ | 空の結果を返す |
| TC-ports-mediaUsageRepository-83 | W に M1 と M2 のメディアがあり、H が M1 を使う | `findStatusByMedia([M1, M1, M2])` を呼ぶ | キーは M1 と M2 の二つ |
| TC-ports-mediaUsageRepository-84 | W に M の公開のメディアがあり、`replace(H, [publicUse(M)])` の後に `replace(H, [use(M)])` を確定済み | `status(M)` を呼ぶ | `in_use` |
| TC-ports-mediaUsageRepository-85 | W に M の公開のメディアがあり、`replace(H, [publicUse(M)])` の後に `clear(H)` を確定済み | `status(M)` を呼ぶ | `unused` を返す |
| TC-ports-mediaUsageRepository-86 | W1 の区画に M のメディアがあり、H が M を使う | W2 の区画で `status(M)` を呼ぶ | `unused` を返す |
| TC-ports-mediaUsageRepository-87 | W に M1 と M2 のメディアがある。M2 は公開。`status(M1, M2)` が、M1 に `in_use` を、M2 に `in_public_use` を返す | `MediaRepository.delete(M1, v)` と、M2 の `unpublish` の結果の `save` を、それぞれ呼ぶ | 削除は `ConflictError`（`MEDIA_IN_USE`）、非公開化は `ConflictError`（`MEDIA_IN_PUBLIC_USE`）になる。読み取りと保護は同じ記録から決まる |
| TC-ports-mediaUsageRepository-88 | W にメディアが、上限の件数だけある | その全件の ID を渡して呼ぶ | 上限の個数のキーを返す |
| TC-ports-mediaUsageRepository-89 | W にメディアが、上限の件数だけある | その全件の ID に、同じ ID の重複を 1 個加えた上限より 1 個多い数を渡して呼ぶ | 上限の個数のキーを返す。上限は、重複を除いた数に適用する |
| TC-ports-mediaUsageRepository-90 | W にメディアが、上限より 1 件多くある | その全件の ID を渡して呼ぶ | `SystemError`（`INVALID_QUERY`）を投げる。上限の件数に切り詰めない |

## 並行性

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaUsageRepository-91 | W に M のメディアがある。使用箇所は無い | 並行する二つの UnitOfWork の、一方が `replace(H, [use(M)])` を、もう一方が `MediaRepository.delete(M, v)` を呼ぶ | 一方だけが確定する。削除が確定した場合、記録の側は `ConflictError`（`MEDIA_NOT_FOUND`）になる。記録が確定した場合、削除の側は `ConflictError`（`MEDIA_IN_USE`）になる |
| TC-ports-mediaUsageRepository-92 | W に M の公開のメディアがある。使用箇所は無い | 並行する二つの UnitOfWork の、一方が `replace(H, [publicUse(M)])` を、もう一方が `Media.unpublish` の結果の `MediaRepository.save` を呼ぶ | 一方だけが確定する。非公開化が確定した場合、記録の側は `ConflictError`（`MEDIA_NOT_PUBLIC`）になる。記録が確定した場合、非公開化の側は `ConflictError`（`MEDIA_IN_PUBLIC_USE`）になる |
| TC-ports-mediaUsageRepository-93 | W に M のメディアがある。使用箇所は無い。`stage(J, [H: [use(M)]])` を確定済み | 並行する二つの UnitOfWork の、一方が `promoteStaged(J)` を、もう一方が `MediaRepository.delete(M, v)` を呼ぶ | 一方だけが確定する。削除が確定した場合、切り替えの側は `ConflictError`（`MEDIA_NOT_FOUND`）になる。切り替えが確定した場合、削除の側は `ConflictError`（`MEDIA_IN_USE`）になる |
| TC-ports-mediaUsageRepository-94 | W に M1 と M2 のメディアがある | 並行する二つの UnitOfWork が、それぞれ `replace(H, [use(M1)])` と `replace(H, [use(M2)])` を呼ぶ | 両方が確定する。M1 と M2 のうち、一方だけが削除できない |
| TC-ports-mediaUsageRepository-95 | W に M1 と M2 のメディアがある。`stage(J, [H: [use(M1)]])` を確定済み | 並行する二つの UnitOfWork が、それぞれ `promoteStaged(J)` と `replace(H, [use(M2)])` を呼ぶ | 両方が確定する。M1 と M2 のうち、一方だけが削除できない。H の使用箇所は、後に確定した側の内容 |
| TC-ports-mediaUsageRepository-96 | W に M のメディアがある | 並行する二つの UnitOfWork が、それぞれ `replace(H1, [use(M)])` と `replace(H2, [use(M)])` を呼ぶ | 両方が確定する。`clear(H1)` を確定した後も、M は削除できない |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaUsageRepository-97 | W の使用箇所も準備領域も無い | `deleteAll(10)` を呼ぶ | 0 を返す |
| TC-ports-mediaUsageRepository-98 | W に M1 と M2 のメディアがあり、H1 が M1 と M2 を、H2 が M1 を使う | `deleteAll(10)` を呼び、確定の後に `usages(M1)` と `status(M1, M2)` を呼ぶ | `limit` より少ない 3 を返す。`usages(M1)` は空で `count` は 0。`status` はどちらも `unused`。M1 も M2 も削除できる |
| TC-ports-mediaUsageRepository-99 | 同上 | `deleteAll(2)` を、呼び出しごとに別の UnitOfWork で、返す件数が 2 より少なくなるまで繰り返す | 返す値は順に 2、1。M1 も M2 も削除できる |
| TC-ports-mediaUsageRepository-100 | W1 の区画と W2 の区画に、それぞれ M1、M2 のメディアと使用箇所がある | W1 の区画で `deleteAll(10)` を呼ぶ | 1 を返す。M1 は削除できる。M2 は削除できない |
| TC-ports-mediaUsageRepository-102 | W に M の公開のメディアがあり、`replace(H, [publicUse(M)])` を確定済み | `deleteAll(10)` を呼び、確定の後に `unpublish` の結果を `save` する | 確定する |
| TC-ports-mediaUsageRepository-103 | W に M のメディアがある。使用箇所は無い。`stage(J1, [H1: [use(M)], H2: []])` と `stage(J2, [H1: [use(M)]])` を確定済み | `deleteAll(10)` を呼び、確定の後に `promoteStaged(J1)` と `promoteStaged(J2)` を確定する | `deleteAll` は、準備領域の持ち主の数の 3 を返す。切り替えの後も、M は削除できる。準備領域の内容を削除する |
| TC-ports-mediaUsageRepository-104 | W の使用箇所が 1 件ある | `limit` に、範囲外の値（0、上限より 1 大きい値）を渡して `deleteAll` を呼ぶ | どちらも `SystemError`（`INVALID_QUERY`）を投げる |

## 可視性と UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaUsageRepository-106 | W に M のメディアがある | `replace(H, [use(M)])` を確定した直後に、`MediaRepository.delete(M, v)` を呼ぶ | `ConflictError`（`MEDIA_IN_USE`）になる |
| TC-ports-mediaUsageRepository-107 | W に M のメディアがある | `replace(H, [use(M)])` を確定した直後に、`usages(M)` と `status(M)` を呼ぶ | `usages(M)` は H の 1 件、`status(M)` は `in_use` |
| TC-ports-mediaUsageRepository-108 | W に M のメディアがある。`stage(J, [H: [use(M)]])` を確定済み | `promoteStaged(J)` を確定した直後に、`usages(M)` と `MediaRepository.delete(M, v)` を呼ぶ | `usages(M)` は H の 1 件。削除は `ConflictError`（`MEDIA_IN_USE`）になる |
| TC-ports-mediaUsageRepository-109 | W に M のメディアがある | `replace(H, [use(M)])` を呼んだ後、`fn` が例外を投げる | `run` はその例外を投げる。M は削除できる。`usages(M)` は空を返す |
| TC-ports-mediaUsageRepository-110 | W に M のメディアがあり、`replace(H, [use(M)])` を確定済み | `clear(H)` を呼んだ後、`fn` が例外を投げる | M は削除できない |
| TC-ports-mediaUsageRepository-111 | W に M のメディアがある | `stage(J, [H: [use(M)]])` を呼んだ後、`fn` が例外を投げる。その後に `promoteStaged(J)` を確定する | M は削除できる。準備領域に何も書かれていない |
| TC-ports-mediaUsageRepository-112 | W に M のメディアがある。`stage(J, [H: [use(M)]])` を確定済み | `promoteStaged(J)` を呼んだ後、`fn` が例外を投げる。その後に、もう一度 `promoteStaged(J)` を確定する | 例外の後、M は削除できる。二度目の確定の後、M は削除できない。準備領域は、確定するまで残る |
| TC-ports-mediaUsageRepository-113 | W に M のメディアがある。`stage(J, [H: [use(M)]])` を確定済み | `discardStaged(J)` を呼んだ後、`fn` が例外を投げる。その後に `promoteStaged(J)` を確定する | M は削除できない。準備領域は残っている |
| TC-ports-mediaUsageRepository-114 | W に M のメディアがあり、`replace(H, [use(M)])` を確定済み | `deleteAll(10)` を呼んだ後、`fn` が例外を投げる | M は削除できない |
| TC-ports-mediaUsageRepository-115 | W の区画に `Workspace` を確定していない。W に M のメディアがある | W の UnitOfWork（条件は `none`）で、`replace(H, [use(M)])` を呼ぶ | 確定する。リポジトリは `Workspace` の状態を見ない |
