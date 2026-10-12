# ObjectRemovalRepository

契約は [media.md](../../domains/media.md) の「ポート」と「ObjectRemoval」、[common.md](../../domains/common.md) の「TransactionalRepository」の共通の規則、「WriteScope」「UnitOfWorkProvider」が定める。

## 記法

- W、W1、W2 はワークスペース、T、T1、T2 は、そのワークスペースの `StorageTarget`。リポジトリは、そのワークスペースの区画のもの。書き込みでは、W の UnitOfWork（`{ kind: "workspace", workspaceId: W, condition: { kind: "none" } }` を渡した `UnitOfWorkProvider.run`）の文脈から、読み取りでは `PartitionReader.workspace(W)` から得る。メソッドは `workspaceId` を取らない。「W に」「W の区画で」は、W の区画のリポジトリを使うことを指す。W1 と W2 を使うケースは、それぞれの区画のリポジトリを使う。区画に `Workspace` を確定しておく必要は無い
- 記録は `ObjectRemoval` の `ofMedia`、`ofObjects` が返す値で作り、`enqueue` で記録する
- 前提条件の書き込みは、W の UnitOfWork で呼び、確定済み。操作の書き込みは、W の新しい UnitOfWork で呼ぶ。`findPending` と `count` は UnitOfWork の外で呼ぶ
- `requestedAt` は古い順に t1 < t2 < t3
- 「T（t1）の記録」は、`target` が T で `requestedAt` が t1 の `ObjectRemoval`。「読んだ記録」は、`findPending` または `enqueue` が返した `QueuedRemoval`
- 引数なしで `findPending()` と書いた場合は `findPending(null, 10)`
- g1、g2 は、互いに違う世代（`DerivativeGeneration`）

## 対象にしない

- 記録が指す実体の消去。ポートは記録だけを扱い、消去は `MediaStorage` が行う
- 保存先の障害と、デコードできない保存値（common.md の共通の規則）

## enqueue

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-objectRemovalRepository-01 | 記録が無い | `findPending()` を呼ぶ | 空の列を返す |
| TC-ports-objectRemovalRepository-02 | 記録が無い | `target` が `media` の記録を 1 件 `enqueue` し、確定の後に `findPending()` を呼ぶ | `enqueue` は、渡した `removal` と `ticket` を持つ 1 件を返す。`findPending()` は、`target`（`mediaId`）、`requestedAt`、`ticket` が等しい 1 件を返す |
| TC-ports-objectRemovalRepository-03 | 記録が無い | `target` が `object` の記録を 1 件 `enqueue` し、確定の後に `findPending()` を呼ぶ | 1 件を返す。`object` の `mediaId`、`variant`（`key` と `generation`）を保つ |
| TC-ports-objectRemovalRepository-04 | 記録が無い | 空の列を `enqueue` する | 確定し、空の列を返す。`findPending()` は空の列を返す |
| TC-ports-objectRemovalRepository-05 | 記録が無い | `target` の異なる 3 件を一度の `enqueue` に渡し、`findPending()` を呼ぶ | `enqueue` は 3 件を返し、`ticket` は互いに違う。`findPending()` は 3 件を返す |
| TC-ports-objectRemovalRepository-06 | T（t1）の記録がある。別の前提では T（t3）の記録がある | T（t1）の記録には T（t3）の記録と同じ T（t1）の記録を、T（t3）の記録には T（t1）の記録を、それぞれ `enqueue` する | どれも確定する。`findPending()` は、T の記録を、渡した `requestedAt` で 1 件だけ返す。`ticket` は、前の記録の値と違う |
| TC-ports-objectRemovalRepository-09 | 記録が無い | T（t1）の記録と T（t3）の記録を、この順に一度の `enqueue` に渡す | `enqueue` は 1 件を返す。`findPending()` は、列の最後の、`requestedAt` が t3 の 1 件だけを返す |
| TC-ports-objectRemovalRepository-10 | 同じメディアの、`target` が `media` の記録と、キー k・世代 g1 の派生形式を指す `object` の記録がある | キー k・世代 g2 の派生形式を指す `object` の記録を `enqueue` する | `findPending()` は 3 件を返す |
| TC-ports-objectRemovalRepository-12 | 記録が無い | 並行する二つの UnitOfWork が、それぞれ同じ `target` の記録を `enqueue` する | 両方が確定する。`findPending()` は、その `target` の記録を 1 件だけ返す |
| TC-ports-objectRemovalRepository-13 | T（t1）の記録を `enqueue` し、`complete` を確定済み | T（t3）の記録を `enqueue` する | `findPending()` は、`requestedAt` が t3 の 1 件を返す |

## findPending

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-objectRemovalRepository-14 | W1 の区画と W2 の区画に、記録が 1 件ずつある | W1 の区画で `findPending()` を呼ぶ | W1 の 1 件だけを返す |
| TC-ports-objectRemovalRepository-15 | W の区画に何も確定していない | `findPending()` を呼ぶ | 空の列を返す |
| TC-ports-objectRemovalRepository-16 | 区画に `Workspace` を確定していない W に、記録がある | `findPending()` を呼ぶ | その記録を返す。ワークスペースの状態を見ない |
| TC-ports-objectRemovalRepository-17 | 記録を `enqueue` した直後 | `findPending()` を呼ぶ | その記録を返す。記録からの経過によらない |
| TC-ports-objectRemovalRepository-18 | `requestedAt` が t2、t3、t1 の記録を、この順に `enqueue` 済み | `findPending()` を呼ぶ | `requestedAt` が t1、t2、t3 の順で返す |
| TC-ports-objectRemovalRepository-19 | `requestedAt` が同じ t1 で、`target` が `media` の記録と、同じメディアの二つの派生形式を指す `object` の記録がある | `findPending()` を呼ぶ | `StorageTarget.key` の文字列のコードポイントの昇順で返す |
| TC-ports-objectRemovalRepository-20 | `requestedAt` が t1、t2、t3 の記録がある | `findPending(null, 2)` を呼ぶ | t1 と t2 の 2 件を返す |
| TC-ports-objectRemovalRepository-21 | `requestedAt` が t1、t2、t3 の記録がある | `findPending(null, 2)` が返した最後の記録の `removal` を `after` に渡して、`findPending(after, 2)` を呼ぶ | t3 の 1 件を返す。返す件数が `limit` より少ない |
| TC-ports-objectRemovalRepository-22 | `requestedAt` が同じ t1 で、`target` が異なる記録が 3 件ある | `StorageTarget.key` の昇順で 2 番目の記録の `removal` を `after` に渡して `findPending` を呼ぶ | 3 番目の 1 件を返す |
| TC-ports-objectRemovalRepository-23 | `requestedAt` が t1、t2、t3 の記録がある。t1 の記録を読んだ後に、その記録を `complete` に渡して確定済み | 読んだ t1 の記録の `removal` を `after` に渡して `findPending` を呼ぶ | t2 と t3 の 2 件を返す。`after` の記録が無くても、その位置より後を返す |
| TC-ports-objectRemovalRepository-24 | T1（t1）と T2（t2）の記録がある。T1 の記録を読んだ後に、T1（t3）の記録を `enqueue` 済み | 先に読んだ T1（t1）の記録の `removal` を `after` に渡して `findPending` を呼ぶ | T2（t2）、T1（t3）の順で 2 件を返す。改めて記録された消去待ちは、新しい `requestedAt` の位置にある |
| TC-ports-objectRemovalRepository-25 | `requestedAt` が t1、t2 の記録がある | t2 の記録の `removal` を `after` に渡して `findPending` を呼ぶ | 空の列を返す |
| TC-ports-objectRemovalRepository-26 | 記録が 1 件ある | `findPending()` を 2 回呼ぶ | どちらも、同じ `ticket` の記録を返す |
| TC-ports-objectRemovalRepository-27 | 記録が 1 件ある | `limit` に、範囲外の値（1 より小さい値、`BoundedDeletion.maxLimit` を超える値）を渡して `findPending` を呼ぶ | どちらも `SystemError`（`INVALID_QUERY`）を投げる |

## complete

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-objectRemovalRepository-28 | T（t1）の記録がある | `findPending` で読んだ記録、または `enqueue` が返した記録を `complete` に渡し、確定の後に `findPending()` を呼ぶ | 確定する。`findPending()` は空の列を返す |
| TC-ports-objectRemovalRepository-30 | T（t1）の記録を `findPending` で読んだ後、T（t3）の記録、または同じ T（t1）の記録を、改めて `enqueue` 済み | 先に読んだ記録を `complete` に渡す | 確定する。`findPending()` は、改めて記録した T の記録を返す |
| TC-ports-objectRemovalRepository-32 | 記録が無い | 前に読んだ T の記録を `complete` に渡す | 確定する |
| TC-ports-objectRemovalRepository-33 | T1 と T2 の記録がある。別の前提では、同じメディアの、`target` が `media` の記録と、派生形式を指す `object` の記録がある | T1、または `media` の読んだ記録を `complete` に渡す | `findPending()` は、T2、または `object` の記録だけを返す |
| TC-ports-objectRemovalRepository-35 | T（t1）の記録がある | 並行する二つの UnitOfWork が、それぞれ同じ読んだ記録を `complete` に渡す | 両方が確定する。`findPending()` は空の列を返す |
| TC-ports-objectRemovalRepository-36 | T（t1）の記録がある | 並行する二つの UnitOfWork の、一方が読んだ記録を `complete` に渡し、もう一方が T（t3）の記録を `enqueue` する | 両方が確定する。`findPending()` は、`enqueue` が返した `ticket` の記録を返す |

## count

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-objectRemovalRepository-37 | 記録が無い | `count()` を呼ぶ | 0 を返す |
| TC-ports-objectRemovalRepository-38 | W の区画に何も確定していない | `count()` を呼ぶ | 0 を返す |
| TC-ports-objectRemovalRepository-39 | `target` が `media` の記録が 2 件、`object` の記録が 1 件ある | `count()` を呼ぶ | 3 を返す |
| TC-ports-objectRemovalRepository-40 | T（t1）の記録がある | T（t3）の記録を `enqueue` し、確定の後に `count()` を呼ぶ | 1 を返す。同じ `target` の記録は一つ |
| TC-ports-objectRemovalRepository-41 | 記録が 2 件ある | 1 件の読んだ記録を `complete` に渡し、確定の後に `count()` を呼ぶ | 1 を返す |
| TC-ports-objectRemovalRepository-42 | T（t1）の記録を `findPending` で読んだ後、T（t3）の記録を `enqueue` 済み | 先に読んだ記録を `complete` に渡し、確定の後に `count()` を呼ぶ | 1 を返す |
| TC-ports-objectRemovalRepository-43 | W1 の区画に 2 件、W2 の区画に 3 件の記録がある | W1 の区画で `count()` を呼ぶ | 2 を返す |

## UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-objectRemovalRepository-44 | 記録が無い | `enqueue` を確定した直後に、`findPending()` と `count()` を呼ぶ | 記録と 1 を返す |
| TC-ports-objectRemovalRepository-45 | M のメディアが `MediaRepository` にある | 一つの UnitOfWork で、`MediaRepository.delete` と `enqueue`（`ObjectRemoval.ofMedia(M, …)` の結果）を呼び、正常に終える | `MediaRepository.findById(M)` は `null`、`findPending()` は記録を返す |
| TC-ports-objectRemovalRepository-46 | 同上 | 一つの UnitOfWork で、`MediaRepository.delete` と `enqueue` を呼んだ後、`fn` が例外を投げる | `run` はその例外を投げる。`MediaRepository.findById(M)` はメディアを返し、`findPending()` は空の列を返す |
| TC-ports-objectRemovalRepository-47 | M のメディアがある。その `expectedVersion` を得た後、別の UnitOfWork が `MediaRepository.save` を確定した | 一つの UnitOfWork で、古い `expectedVersion` での `MediaRepository.save` と、`enqueue` を呼ぶ | `ConflictError`（`STALE_VERSION`）になる。`findPending()` は空の列を返す |
| TC-ports-objectRemovalRepository-48 | T（t1）の記録がある | 読んだ記録を `complete` に渡した後、`fn` が例外を投げる | `findPending()` は記録を返す |
| TC-ports-objectRemovalRepository-49 | 区画に `Workspace` を確定していない W | W の UnitOfWork（条件は `none`）で、W の `target` の記録を `enqueue` する | 確定する。`findPending()` はその記録を返す。リポジトリは `Workspace` の状態を見ない |
