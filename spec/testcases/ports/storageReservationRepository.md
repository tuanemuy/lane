# StorageReservationRepository

契約は [media.md](../../domains/media.md) の「ポート」と「StorageReservation」、[common.md](../../domains/common.md) の「TransactionalRepository」の共通の規則、「WriteScope」「UnitOfWorkProvider」が定める。

## 記法

- W、W1、W2 はワークスペース、R、R1、R2、R3 は、ID が互いに違う保管の予約。T、T1、T2 は、互いに違う `StorageTarget`。リポジトリは、そのワークスペースの区画のもの。書き込みでは、W の UnitOfWork（`{ kind: "workspace", workspaceId: W, condition: { kind: "none" } }` を渡した `UnitOfWorkProvider.run`）の文脈から、読み取りでは `PartitionReader.workspace(W)` から得る。メソッドは `workspaceId` を取らない。「W に」「W の区画で」は、W の区画のリポジトリを使うことを指す。W1 と W2 を使うケースは、それぞれの区画のリポジトリを使う。区画に `Workspace` を確定しておく必要は無い
- 記録は `StorageReservation.reserve` が返す値で作り、`insert` で保存する。ID は `IdGenerator.generate` の値
- 前提条件の書き込みは、W の UnitOfWork で呼び、確定済み。操作の書き込みは、W の新しい UnitOfWork で呼ぶ。`findById`、`findCollectable`、`count` は UnitOfWork の外で呼ぶ
- 「`ConflictError`（理由）になる」は、メソッドの呼び出しか確定のどちらかでエラーが現れ、`run` がそのエラーを投げ、その UnitOfWork の書き込みが一つも反映されないことを指す
- `expiresAt` は古い順に e1 < e2 < e3 < e4
- 引数なしで `findCollectable()` と書いた場合は `findCollectable(e4, null, 10)`

## 対象にしない

- 保存先の障害と、デコードできない保存値（common.md の共通の規則）

## insert と findById

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-storageReservationRepository-01 | 記録が無い | R の ID で `findById` を呼ぶ | `null` を返す |
| TC-ports-storageReservationRepository-02 | 記録が無い | `target` が `media` の R1 と、派生形式の `object` の R2 を `insert` し、確定の後に、それぞれの ID で `findById` を呼ぶ | `id`、`target`（`object` ではメディア、キー、世代）、`expiresAt` が等しい記録を返す |
| TC-ports-storageReservationRepository-04 | R の記録がある | 同じ ID で、`target` と `expiresAt` が異なる記録を `insert` する | `ConflictError`（`ALREADY_EXISTS`）になる。`findById` は、先に `insert` した記録を返す |
| TC-ports-storageReservationRepository-05 | `target` が T の R1 の記録がある | `target` が同じ T の R2 を `insert` する | 確定する。`findById` は R1 と R2 をそれぞれ返す。同じ `target` の予約は複数持てる |
| TC-ports-storageReservationRepository-06 | W1 の区画に R の記録がある | W2 の区画で、R の ID の `findById` と、`findCollectable()`、`count()` を呼ぶ | `null`、空の列、0 を返す。ほかの区画の記録を返さない |
| TC-ports-storageReservationRepository-07 | 記録が無い | 並行する二つの UnitOfWork が、それぞれ同じ ID の記録を `insert` する | 一方だけが確定し、もう一方は `ConflictError`（`ALREADY_EXISTS`）になる |
| TC-ports-storageReservationRepository-08 | `expiresAt` が e1 の R の記録がある | R の ID で `findById` を呼ぶ | 記録を返す。期限によらない |

## settle

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-storageReservationRepository-09 | W に R の記録がある | R の ID で `settle` を呼び、確定の後に `findById` を呼ぶ | 確定する。`findById` は `null` を返す |
| TC-ports-storageReservationRepository-10 | 記録が無い | R の ID の `settle` と、`MediaRepository.insert` を呼ぶ | `ConflictError`（`RESERVATION_NOT_FOUND`）になる。`MediaRepository.findById` は `null` を返す |
| TC-ports-storageReservationRepository-11 | W に R の記録を `insert` し、`settle` を確定済み | もう一度 R の ID で `settle` を呼ぶ | `ConflictError`（`RESERVATION_NOT_FOUND`）になる |
| TC-ports-storageReservationRepository-12 | W1 の区画に R の記録がある | W2 の区画で、R の ID の `settle` を呼ぶ | `ConflictError`（`RESERVATION_NOT_FOUND`）になる。W1 の区画の `findById` は記録を返す |
| TC-ports-storageReservationRepository-13 | W に、`target` が同じ T の R1 と R2 の記録がある | R1 の ID で `settle` を呼ぶ | `findById` は、R1 に `null`、R2 に記録を返す |
| TC-ports-storageReservationRepository-14 | W に、`expiresAt` が e1 の R の記録がある | R の ID で `settle` を呼ぶ | 確定する。期限で拒否しない |
| TC-ports-storageReservationRepository-15 | W に R の記録がある | 並行する二つの UnitOfWork の、一方が R の ID の `settle` と `MediaRepository.insert` を、もう一方が R の ID の `settle` と `ObjectRemovalRepository.enqueue` を呼ぶ | 一方だけが確定し、もう一方は `ConflictError`（`RESERVATION_NOT_FOUND`）になる。メディアの `insert` と消去待ちの記録は、確定した側のものだけが残る |

## findCollectable

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-storageReservationRepository-16 | 記録が無い | `findCollectable()` を呼ぶ | 空の列を返す |
| TC-ports-storageReservationRepository-17 | `expiresAt` が e1、e2、e3 の記録がある | `findCollectable(e2, null, 10)` を呼ぶ | e1 と e2 の 2 件を返す。`StorageReservation.isCollectable` が真の記録だけを返す |
| TC-ports-storageReservationRepository-18 | `expiresAt` が e2 と e3 の記録がある | `findCollectable(e1, null, 10)` を呼ぶ | 空の列を返す |
| TC-ports-storageReservationRepository-19 | `expiresAt` が e2、e3、e1 の記録を、この順に `insert` 済み | `findCollectable()` を呼ぶ | `expiresAt` が e1、e2、e3 の順で返す |
| TC-ports-storageReservationRepository-20 | `expiresAt` が同じ e1 で、ID が異なる記録が 3 件ある | `findCollectable()` を呼ぶ | ID のコードポイントの昇順で返す |
| TC-ports-storageReservationRepository-21 | W1 の区画と W2 の区画に、`expiresAt` が e1 の記録が 1 件ずつある | W1 の区画で `findCollectable()` を呼ぶ | W1 の 1 件だけを返す |
| TC-ports-storageReservationRepository-22 | 区画に `Workspace` を確定していない W に、`expiresAt` が e1 の記録がある | `findCollectable()` を呼ぶ | その記録を返す。ワークスペースの状態を見ない |
| TC-ports-storageReservationRepository-23 | `expiresAt` が e1、e2、e3 の記録がある | `findCollectable(e4, null, 2)` を呼ぶ | e1 と e2 の 2 件を返す |
| TC-ports-storageReservationRepository-24 | `expiresAt` が e1、e2、e3 の記録がある | `findCollectable(e4, null, 2)` が返した最後の記録を `after` に渡して、`findCollectable(e4, after, 2)` を呼ぶ | e3 の 1 件を返す。返す件数が `limit` より少ない |
| TC-ports-storageReservationRepository-25 | `expiresAt` が同じ e1 で、ID が異なる記録が 3 件ある | ID の昇順で 2 番目の記録を `after` に渡して `findCollectable` を呼ぶ | 3 番目の 1 件を返す |
| TC-ports-storageReservationRepository-26 | `expiresAt` が e1、e2、e3 の記録がある。e1 の記録を読んだ後に、その記録を `settle` して確定済み | 読んだ e1 の記録を `after` に渡して `findCollectable` を呼ぶ | e2 と e3 の 2 件を返す。`after` の記録が無くても、その位置より後を返す |
| TC-ports-storageReservationRepository-27 | `expiresAt` が e1、e2 の記録がある | e2 の記録を `after` に渡して `findCollectable` を呼ぶ | 空の列を返す |
| TC-ports-storageReservationRepository-28 | 記録が 1 件ある | `limit` に、範囲外の値（1 より小さい値、`BoundedDeletion.maxLimit` を超える値）を渡して `findCollectable` を呼ぶ | どちらも `SystemError`（`INVALID_QUERY`）を投げる |

## count

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-storageReservationRepository-29 | 記録が無い | `count()` を呼ぶ | 0 を返す |
| TC-ports-storageReservationRepository-30 | W の区画に何も確定していない | `count()` を呼ぶ | 0 を返す |
| TC-ports-storageReservationRepository-31 | `expiresAt` が e1 の記録と、e3 の記録がある | `count()` を呼ぶ | 2 を返す。期限によらない |
| TC-ports-storageReservationRepository-32 | `target` が同じ T の記録が 2 件ある | `count()` を呼ぶ | 2 を返す |
| TC-ports-storageReservationRepository-33 | 記録が 2 件ある | 1 件を `settle` し、確定の後に `count()` を呼ぶ | 1 を返す |
| TC-ports-storageReservationRepository-34 | W1 の区画に 2 件、W2 の区画に 3 件の記録がある | W1 の区画で `count()` を呼ぶ | 2 を返す |

## 可視性と UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-storageReservationRepository-35 | 記録が無い | R の `insert` を確定した直後に、`findById`、`findCollectable()`、`count()` を呼ぶ | 記録、記録を含む列、1 を返す |
| TC-ports-storageReservationRepository-36 | W に R の記録がある | R の ID の `settle` を確定した直後に、`findById`、`findCollectable()`、`count()` を呼ぶ | `null`、空の列、0 を返す |
| TC-ports-storageReservationRepository-37 | 記録が無い | `insert` を呼んだ後、`fn` が例外を投げる | `findById` は `null` を返す |
| TC-ports-storageReservationRepository-38 | W に R の記録がある | R の ID の `settle` を呼んだ後、`fn` が例外を投げる | `findById` は記録を返す |
| TC-ports-storageReservationRepository-39 | W の区画に `Workspace` を確定していない。W に R の記録がある | W の UnitOfWork（条件は `none`）で、R の ID の `settle` と `ObjectRemovalRepository.enqueue` を呼ぶ | 確定する。`count()` は 0 を返す。リポジトリは `Workspace` の状態を見ない |
| TC-ports-storageReservationRepository-40 | W に R の記録がある | 一つの UnitOfWork で、R の ID の `settle` と、`ObjectRemovalRepository.enqueue` を呼んだ後、`fn` が例外を投げる | `findById` は記録を返す。`ObjectRemovalRepository.count()` は 0 を返す |
