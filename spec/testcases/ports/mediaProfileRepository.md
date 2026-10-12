# MediaProfileRepository

「上限」は、その節のメソッドの契約が定める、一度に渡せる件数・個数の上限を指す。値は、ポートを定めるドメインのファイルが持つ。

契約は [media.md](../../domains/media.md) の「ポート」と、[common.md](../../domains/common.md) の「TransactionalRepository」「WriteScope」「UnitOfWorkProvider」が定める。

## 記法

- W、W1、W2 はワークスペース、M、M1、M2 は `MediaId`。リポジトリは、そのワークスペースの区画のもの。書き込みでは、W の UnitOfWork（`{ kind: "workspace", workspaceId: W, condition: { kind: "none" } }` を渡した `UnitOfWorkProvider.run`）の文脈から、読み取りでは `PartitionReader.workspace(W)` から得る。メソッドは `workspaceId` を取らない。区画に `Workspace` を確定しておく必要は無い。「W の区画に何も確定していない」は、W の UnitOfWork で一度も確定していないことを指す
- 管理情報は `MediaProfile` の振る舞い（`create`、`describe`）が返す値で作り、`insert` と `save` で保存する
- 「M のメディアと管理情報がある」は、`MediaRepository.insert` と、その `MediaProfile.create` の値の `insert` を、同じ UnitOfWork で確定した状態
- 前提条件の書き込みは、W の UnitOfWork で呼び、確定済み。操作の書き込みは、W の新しい UnitOfWork で呼ぶ。読み取りは UnitOfWork の外で呼ぶ
- 「`ConflictError`（理由）になる」は、メソッドの呼び出しか確定のどちらかでエラーが現れ、`run` がそのエラーを投げ、その UnitOfWork の書き込みが一つも反映されないことを指す
- v は、直前の `findById` が返した `expectedVersion`

## 対象にしない

- `Media` の `insert` と別の UnitOfWork で、または `Media` が無い状態で `insert` した場合の結果。契約は、同じ UnitOfWork で呼ぶことを呼び出し側に求め、結果を約束しない
- 保存先の障害と、デコードできない保存値（common.md の共通の規則）

## insert と findById

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaProfileRepository-01 | M の管理情報が無い | `findById(M)` を呼ぶ | `null` を返す |
| TC-ports-mediaProfileRepository-02 | M のメディアが無い | 一つの UnitOfWork で、`MediaRepository.insert` と、`create` した管理情報の `insert` を呼び、確定の後に `findById(M)` を呼ぶ | `insert` した値と全フィールドが等しい `entity`（`name`、`searchInfo`、`sharedDescription` を含む）と、`expectedVersion` を返す |
| TC-ports-mediaProfileRepository-03 | M のメディアと管理情報がある | 同じ `mediaId` の、内容が異なる管理情報を `insert` する | `ConflictError`（`ALREADY_EXISTS`）になる。`findById(M)` は、先に `insert` した値を返す |
| TC-ports-mediaProfileRepository-04 | M のメディアと管理情報が W1 の区画にある | W1 の区画と W2 の区画で、それぞれ `findById(M)` を呼ぶ | W1 の区画は `entity` を返す。W2 の区画は `null` を返す |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaProfileRepository-05 | M のメディアと管理情報がある | `describe` の結果を v とともに `save` し、確定の後に `findById(M)` を呼ぶ | `name`、`searchInfo`、`sharedDescription`（`alt` と `caption`）、`updatedAt` が保存した値の `entity` を返す |
| TC-ports-mediaProfileRepository-06 | M のメディアと管理情報がある。v を得た後、別の UnitOfWork が `save` を確定した | v とともに `save` する | `ConflictError`（`STALE_VERSION`）になる。`findById(M)` は、先に確定した値を返す |
| TC-ports-mediaProfileRepository-07 | M のメディアと管理情報がある。二つの処理が、同じ `expectedVersion` を得た | 並行する二つの UnitOfWork が、それぞれ名称の違う `describe` の結果を `save` する | 一方だけが確定し、もう一方は `ConflictError`（`STALE_VERSION`）になる。`findById(M)` は、確定した側の値を返す |
| TC-ports-mediaProfileRepository-08 | M のメディアと管理情報がある。v を得た後、`MediaRepository.delete` で M を削除した | v とともに `save` する | `ConflictError`（`STALE_VERSION`）になる。`findById(M)` は `null` を返す |
| TC-ports-mediaProfileRepository-09 | M の非公開のメディアと管理情報がある。二つの処理が、`Media` と管理情報の `expectedVersion` をそれぞれ得た | 並行する二つの UnitOfWork の、一方が管理情報の `save` を、もう一方が M の公開使用の記録と `MediaRepository.publishMany([M], t)` を呼ぶ | 両方が確定する。`Media` の保存とは競合しない |
| TC-ports-mediaProfileRepository-10 | W の区画に `Workspace` を確定していない。M のメディアと管理情報がある | W の UnitOfWork（条件は `none`）で、`describe` の結果を `save` する | 確定する。リポジトリは `Workspace` の状態を見ない |

## findByIds

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaProfileRepository-11 | W に M1、M2、M3 のメディアと管理情報がある | `findByIds([M1, M3])` を呼ぶ | M1 と M3 の管理情報の 2 件を返す。順序は問わない |
| TC-ports-mediaProfileRepository-12 | W に M1 のメディアと管理情報がある。M9 は無い | `findByIds([M1, M9])` を呼ぶ | M1 の 1 件だけを返す |
| TC-ports-mediaProfileRepository-13 | W1 の区画に M1、W2 の区画に M2 のメディアと管理情報がある | W1 の区画で `findByIds([M1, M2])` を呼ぶ | M1 の 1 件だけを返す |
| TC-ports-mediaProfileRepository-14 | W の区画に何も確定していない | `findByIds([M1])` を呼ぶ | 空の列を返す |
| TC-ports-mediaProfileRepository-15 | W に M1 と M2 のメディアと管理情報がある | `findByIds([M1, M1, M2])` を呼ぶ | M1 と M2 の 2 件を返す。M1 を一度だけ返す |
| TC-ports-mediaProfileRepository-16 | W にメディアと管理情報が、上限の件数だけある | その全件の ID を渡して `findByIds` を呼ぶ | 上限の件数を返す |
| TC-ports-mediaProfileRepository-17 | W にメディアと管理情報が、上限より 1 件多くある | その全件の ID を渡して `findByIds` を呼ぶ | `SystemError`（`INVALID_QUERY`）を投げる。上限の件数に切り詰めない |

## 削除

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaProfileRepository-18 | M のメディアと管理情報がある。使用箇所は無い | `MediaRepository.delete(M, v)` を確定し、`findById(M)` を呼ぶ | `null` を返す |
| TC-ports-mediaProfileRepository-19 | W にメディアと管理情報が 2 件ある | `MediaRepository.deleteAll(10)` を確定し、それぞれの `findById` を呼ぶ | どちらも `null` を返す |
| TC-ports-mediaProfileRepository-20 | M のメディアと管理情報がある | `MediaRepository.delete(M, v)` を呼んだ後、`fn` が例外を投げる | `findById(M)` は管理情報を返す |
