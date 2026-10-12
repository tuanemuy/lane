# RecoveryJournal

`spec/domains/installation.md` の `RecoveryJournal` の契約を、すべてのアダプターが通すケースで定める。

- 記録の保存先は試験用の相手（`spec/domains/common.md`）。各ケースは、復旧の記録が無い状態から始める
- R、R1、R2 は、`Recovery` の振る舞いが返す値。R は、そのケースの操作の欄が、どの振る舞いの結果かを示す。B1、B2 は別の `BackupRecordId`。e は、直前の `find` が返した `expectedVersion`
- 「R と全フィールドが等しい」は、`find` が返す `entity` が、`save` に渡した R と、`phase` の変種のフィールドを含むすべてのフィールドで等しいことを指す
- 「B1 の `restoring` の記録」は、`Recovery.begin` に B1 の記録を渡して返った記録。占有の期限（`leaseUntil`）を持つ
- 「UnitOfWork」は、`WriteScope` に `{ kind: "installation" }` を渡した `UnitOfWorkProvider.run` の `fn`

## find

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-recoveryJournal-01 | 復旧の記録が無い | `find` を呼ぶ | `null` |
| TC-ports-recoveryJournal-02 | B1 の `restoring` の記録を保存した | `find` を呼ぶ | 渡した値と全フィールド（`backupId`、`restorePoint`、`startedAt`、`leaseUntil`、`phase`、`version`）が等しい `entity` と、`expectedVersion` を返す |
| TC-ports-recoveryJournal-03 | 占有の期限（`leaseUntil`）が、現在時刻より前の `restoring` の記録を保存した | `find` を呼ぶ | 保存したままの記録を返す |
| TC-ports-recoveryJournal-04 | `restoring` の記録の `Recovery.fail` の結果 R を `save(R, e)` で保存した | `find` を呼ぶ | R と全フィールドが等しい記録を返す。`phase` は `failed` で、`failedAt` を持ち、`leaseUntil` を持たない |
| TC-ports-recoveryJournal-05 | `restoring` の記録の `Recovery.complete` の結果 R を `save(R, e)` で保存した | `find` を呼ぶ | R と全フィールドが等しい記録を返す。`phase` は `restored` で、`restoredAt` を持つ |
| TC-ports-recoveryJournal-06 | 記録がある | `find` を二度呼ぶ | 同じ内容と、同じ `expectedVersion` を返す。`find` は記録を変えない |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-recoveryJournal-07 | 復旧の記録が無い | B1 の `restoring` の記録を `save(R1, null)` で保存し、直後に `find` を呼ぶ | 待たずに、R1 を返す |
| TC-ports-recoveryJournal-10 | `failed`、`restored`、または B1 の `restoring` の記録がある | その記録を `current` に、B2 の記録を渡した `Recovery.begin` の結果 R を `save(R, e)` で保存する | 成功する。`find` は、R と全フィールドが等しい記録を返す |
| TC-ports-recoveryJournal-11 | `restored` の記録 P を読み、B2 の記録を渡した `Recovery.begin` の結果 R を保存した | `Recovery.withdraw(R, P)` の結果 R1 を `save` で保存する（`expectedVersion` は、R の保存の後の `find` の値） | 成功する。`find` は、R1 と全フィールドが等しい記録を返す |
| TC-ports-recoveryJournal-14 | 復旧の記録がある | `save(R2, null)` を呼ぶ | `ConflictError`（理由 `STALE_VERSION`）。`find` は元の記録を返す |
| TC-ports-recoveryJournal-15 | 復旧の記録が無い | `null` でない `expectedVersion` で `save` を呼ぶ | `ConflictError`（理由 `STALE_VERSION`）。`find` は `null` |
| TC-ports-recoveryJournal-16 | 記録を二度 `find` で読み、一方の変更を保存した | 他方の変更を、最初に読んだ `expectedVersion` で `save` に渡す | `ConflictError`（理由 `STALE_VERSION`）。`find` は先に保存した内容を返す |
| TC-ports-recoveryJournal-17 | 復旧の記録が無い設置と、`restoring` の記録がある設置 | 記録が無い設置では、並行して二つの `save(R, null)` を呼ぶ。記録がある設置では、並行して、同じ `expectedVersion` で、`Recovery.complete` の結果の `save` と、`Recovery.begin` の結果の `save` を呼ぶ | どちらの設置でも、一方だけが成功する。他方は `ConflictError`（理由 `STALE_VERSION`）。`find` は成功した側の値を返す |
| TC-ports-recoveryJournal-19 | 存在しない `BackupRecordId` を `backupId` に持つ `restoring` の記録 | `save(R, null)` を呼ぶ | 成功する |

## clear

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-recoveryJournal-20 | `restored` の記録がある | `clear(e)` を呼び、直後に `find` を呼ぶ | `null` |
| TC-ports-recoveryJournal-21 | `restoring` の記録がある | `clear(e)` を呼ぶ | 成功する。`find` は `null` |
| TC-ports-recoveryJournal-22 | 記録を `find` で読んだ後、別の保存が確定した | 最初に読んだ `expectedVersion` で `clear` を呼ぶ | `ConflictError`（理由 `STALE_VERSION`）。記録は残る |
| TC-ports-recoveryJournal-23 | 復旧の記録が無い | 任意の `expectedVersion` で `clear` を呼ぶ | `ConflictError`（理由 `STALE_VERSION`） |
| TC-ports-recoveryJournal-24 | `restored` の記録がある | 並行して、同じ `expectedVersion` で、`Recovery.begin` の結果の `save` と、`clear` を呼ぶ | 一方だけが成功する。他方は `ConflictError`（理由 `STALE_VERSION`） |
| TC-ports-recoveryJournal-25 | `clear` で記録を消した | `save(R1, null)` を呼ぶ | 成功する |

## 設置のデータと UnitOfWork

「B1 を取得する」と `restore` は、`backupStore.md` の定義による。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-recoveryJournal-26 | B1 を取得した後に、B1 の `restoring` の記録を保存した | その記録を渡して `BackupStore.restore` を呼び、`find` を呼ぶ | `restore` の前と同じ内容（B1 の `restoring`）と、同じ `expectedVersion` を返す |
| TC-ports-recoveryJournal-28 | 設置の区画に `Installation` が無い | `save(R1, null)` を呼び、`find` を呼ぶ | R1 を返す |
| TC-ports-recoveryJournal-29 | 復旧の記録が無い設置と、`restored` の記録がある設置 | UnitOfWork の中で、`save(R1, null)`（記録が無い設置）または `clear(e)`（記録がある設置）を呼び、その後に `fn` が例外を投げる | `find` は、`save` では R1、`clear` では `null` を返す |
