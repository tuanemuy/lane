# BackupStore

`spec/domains/installation.md` の `BackupStore` の契約を、すべてのアダプターが通すケースで定める。

- 「UnitOfWork」は `UnitOfWorkProvider.run` の `fn`。書く区画の `WriteScope`（設置の区画は `{ kind: "installation" }`、ワークスペースの区画は条件が `none` の `workspace`）を渡す。「確定」は `fn` の正常終了を指す
- 設置のデータは、各集約のリポジトリの書き込みを、その区画の UnitOfWork で確定して作る
- どのケースも、「空の設置」と断るものを除いて、`Installation`（初期構成は `settled`）がある設置から始める。「空の設置」は、`InstallationRepository.find` が `null` を返し、どの区画にもデータが無い設置
- 「`findLatest` は X」「`findLatest` の記録」は、`findLatest` が `status` が `read` の結果を返し、その `record` が X であること、その `record` を指す
- B1、B2、B3 は別の `BackupRecordId`（B1 < B2 < B3）。W1、W2、W3 は別のワークスペース（W1 < W2 < W3）。V1 と V2 は別の `DataVersion`
- 「設置に W1 がある」は、`WorkspaceDirectory.register` で W1 を名簿に登録して開設済みにし、W1 の区画に `Workspace` を確定した状態を指す
- 「B1 を取得する」は、`capture(B1)`、`verify(B1)`、`BackupRecord.record` の結果を渡した `complete` を順に終えることを指す。「取得中の B1」は、`capture(B1)` を終え、`complete` を呼んでいない状態
- 「`restore(B1)`」は、`backupId` が B1 の `RestoringRecovery`（B1 の記録を渡した `Recovery.begin` が返す記録）を渡した `restore` を指す。`RecoveryJournal` を使わない
- 期待結果の読み取りは、各集約のリポジトリの読み取り（`PartitionReader`）で行う
- 保管先は試験用の相手（`spec/domains/common.md`）。取得済みのバックアップを削除でき、`restore` の置き換えと、`capture` の写しの取得を、進行中のまま止められる。`restore` の置き換えを、始めた後に失敗させられる（どこまで置き換えた後に失敗するかは、指定しない）。配送を繰り返しても処理できないイベントの隔離は、`spec/domains/common.md` の「QuarantinedEventQuery」が定める試験用の消費者で起こす。置き換えの打ち切りの時間（`Recovery.restoreTimeoutMs`）の経過を、指定できる。打ち切った `remove` を、指定した時点で保管先へ届けられる
- メディアの実体の読み取りは、`MediaStorage`（`spec/domains/media.md`）で行う
- 「確定の前で止めている」UnitOfWork は、`fn` の中で待たせ、テストが `fn` を正常終了させるまで確定させないものを指す。止める時点と進める時点は、テストが `fn` の中で決める。アダプターの内部の割り込みの順序に依らない

## capture

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-backupStore-01 | 設置に W1 と W2 がある | `capture(B1)` を呼ぶ | `BackupCapture` を返す。`workspaces` は W1 と W2 の `id` と `name` を、ID の昇順で持つ |
| TC-ports-backupStore-02 | 名簿に、準備中の W1、開設済みの W2、終了中の W3 がある | `capture(B1)` を呼ぶ | `workspaces` は W1、W2、W3 を持つ |
| TC-ports-backupStore-03 | 設置に W1 がある。名簿に登録していない W3 の区画に、集約を確定した | `capture(B1)` を呼び、`complete` を終えて、`restore(B1)` を呼ぶ | `workspaces` は W1 だけを持つ。復旧の後、W3 の区画の集約は一件も読めない |
| TC-ports-backupStore-04 | 設置の区画にアカウント X を、W1 の区画に集約 Y を確定した後に、B1 を取得した。その後に X と Y を削除して確定した | `restore(B1)` を呼ぶ | X と Y は、どちらも取得の時点の内容を返す |
| TC-ports-backupStore-05 | ワークスペースが無い | `capture(B1)` を呼ぶ | `workspaces` が空の `BackupCapture` を返す |
| TC-ports-backupStore-06 | 設置に W1 がある。試験用の `Clock` は、時刻 t を返す | `capture(B1)` を呼ぶ | `BackupCapture.takenAt` は t |
| TC-ports-backupStore-07 | 設置に W1 がある | `capture(B1)` を呼ぶ | `BackupCapture.sizeBytes` は、0 以上の整数 |
| TC-ports-backupStore-08 | 設置にデータがある | `capture(B1)` を呼び、続けて `verify(B1)` を呼ぶ | `intact: true` |
| TC-ports-backupStore-09 | `Installation` の `update` が `updating` の設置（`Installation.beginUpdate` の結果を保存した）。取得中の B2 がある | `capture(B1)` を呼ぶ | `ConflictError`（理由 `UPDATE_IN_PROGRESS`）。`verify(B1)` は `NotFoundError`。取得中の B2 は残り、`verify(B2)` は `intact: true` を返す |
| TC-ports-backupStore-10 | 上の設置で、`Installation.completeUpdate` の結果を保存した | `capture(B1)` を呼ぶ | `BackupCapture` を返す。`dataVersion` は、更新の後の `Installation` の `dataVersion` |
| TC-ports-backupStore-11 | 試験用の相手が、`capture(B1)` の写しの取得を進行中のまま止めている | `InstallationEnvironment.updateData` を呼び、その後に、止めていた `capture(B1)` を進める | `updateData` は `ok: false`（`failed`）を返し、データを変えない。`capture(B1)` は完了する |
| TC-ports-backupStore-12 | 集約 X を確定した後に B1 を取得し、その後に集約 Y を確定した | `restore(B1)` を呼ぶ | X は読める。Y は読めない |
| TC-ports-backupStore-13 | 集約 X を確定した後に B1 を取得し、その後に X を変更して確定した | `restore(B1)` を呼ぶ | X は、取得の時点の内容を返す |
| TC-ports-backupStore-15 | `capture(B1)` と、W1 の区画の複数の集約を書いて確定する一つの UnitOfWork を、並行に実行し、`complete` を終えた | `restore(B1)` を呼ぶ | その UnitOfWork の書き込みは、すべて読めるか、一件も読めないかのどちらかになる |
| TC-ports-backupStore-16 | `capture(B1)` と、設置の区画の複数の集約を書く UnitOfWork、W1 の区画の複数の集約を書く UnitOfWork を、並行に実行し、`complete` を終えた | `restore(B1)` を呼ぶ | UnitOfWork ごとに、その書き込みは、すべて読めるか、一件も読めないかのどちらかになる。二つの UnitOfWork の結果がそろうことは確かめない |
| TC-ports-backupStore-17 | 設置にデータがある | `capture(B1)` と、集約を書く UnitOfWork を、並行に実行する | UnitOfWork は確定する。`capture` も完了する |
| TC-ports-backupStore-18 | 取得中の B1 がある | `capture(B1)` を呼ぶ | `ConflictError`（理由 `ALREADY_EXISTS`） |
| TC-ports-backupStore-19 | B1 を取得した | `capture(B1)` を呼ぶ | `ConflictError`（理由 `ALREADY_EXISTS`）。`verify(B1)` は `intact: true` を返し、`restore(B1)` は最初の取得の内容へ戻す |
| TC-ports-backupStore-20 | B1 を取得し、その後にデータを変更して確定した | B2 を取得する | 成功する。`restore(B2)` は変更後の内容へ、`restore(B1)` は変更前の内容へ戻す |
| TC-ports-backupStore-21 | 設置にデータがある | `capture(B1)` を呼び、その前後で各集約を読む | 設置のデータは変わらない |
| TC-ports-backupStore-22 | 設置にデータがある | UnitOfWork の中で `capture(B1)` を呼び、その後に `fn` が例外を投げる | `verify(B1)` は `intact: true` を返す |
| TC-ports-backupStore-23 | `Installation` の `dataVersion` が `InstallationEnvironment.dataVersion` の値と同じ設置 | `capture(B1)` を呼ぶ | `BackupCapture.dataVersion` は、その値に等しい |
| TC-ports-backupStore-24 | `Installation` の `dataVersion`（保存済みの版）が、`InstallationEnvironment.dataVersion` の値（実行中の版）と違う設置 | `capture(B1)` を呼ぶ | `BackupCapture.dataVersion` は、`Installation` の `dataVersion` に等しい。実行中の版ではない |

## 取得中のバックアップ

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-backupStore-25 | 取得中の B1 がある | `findById(B1)`、`findLatest`、`findPage` を呼ぶ | `findById` は `null`、`findLatest` は `null`、`findPage` は空の列と `count` 0 |
| TC-ports-backupStore-26 | 取得中の B1 がある | `BackupRecord.record` の B1 の記録から作った `RestoringRecovery` を渡して `restore` を呼ぶ | `NotFoundError`。設置のデータは変わらない |
| TC-ports-backupStore-27 | 取得中の B1 がある | `capture(B2)` を呼び、続けて `verify(B1)` を呼ぶ | `capture(B2)` は成功する。`verify(B1)` は `NotFoundError` |
| TC-ports-backupStore-28 | 取得中の B1 がある。`capture(B2)` を終えた | B1 の記録を `complete` に渡す | `NotFoundError`。`findById(B1)` は `null` |
| TC-ports-backupStore-29 | 取得中の B1 がある。`capture(B2)` を終えた | `capture(B1)` を呼ぶ | 成功する。前の取得中の写しは残っていない |
| TC-ports-backupStore-30 | B1 を取得した。取得中の B2 がある | `capture(B3)` を呼ぶ | 成功する。`findById(B1)` は B1 の記録を返し、`verify(B1)` は `intact: true`。取得済みのバックアップは消えない |
| TC-ports-backupStore-31 | 設置にデータがある | 二つの取得（`capture` から `complete` まで）を、B1 と B2 で並行に実行する | `verify` または `complete` が `NotFoundError` になった側は、`findById` が `null` を返す。`findPage` の `count` は、`complete` が成功した数に等しい |

## complete

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-backupStore-32 | 取得中の B1 がある | `integrity` が `verified` の B1 の記録を `complete` に渡す | 完了の直後の `findById(B1)` が、渡した記録と全フィールド（`takenAt`、`integrity`、`retainUntil`、`dataVersion`、`secretKeyId`、`workspaces`、`sizeBytes`）が等しい記録を返す |
| TC-ports-backupStore-33 | 取得中の B1 がある | `integrity` が `failed`（理由を持つ）の B1 の記録を `complete` に渡す | `findById(B1)` が返す `integrity` は `failed`。`checkedAt` と `reason` は渡した値 |
| TC-ports-backupStore-34 | 取得中の B1 がある | `workspaces` が 3 件の B1 の記録を `complete` に渡す | `findById(B1)` が返す `workspaces` は、渡した `id` と `name` を、渡した順に持つ |
| TC-ports-backupStore-35 | B1 のバックアップが無い | B1 の記録を `complete` に渡す | `NotFoundError`。`findById(B1)` は `null`。`findPage` の `count` は 0 |
| TC-ports-backupStore-36 | B1 を取得した | 内容が別の B1 の記録を `complete` に渡す | `ConflictError`（理由 `ALREADY_EXISTS`）。`findById(B1)` は元の記録を返す |
| TC-ports-backupStore-37 | 取得中の B1 がある | 同じ B1 の記録を、並行して二つ `complete` に渡す | 一方だけが成功する。他方は `ConflictError`（理由 `ALREADY_EXISTS`）。`findPage` の `count` は 1 |
| TC-ports-backupStore-38 | 取得中の B1 がある | UnitOfWork の中で `complete` を呼び、その後に `fn` が例外を投げる | `findById(B1)` は B1 の記録を返す |

## findById・findLatest

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-backupStore-39 | バックアップが無い | `findById(B1)`、`findLatest` を呼ぶ | `findById` は `null`、`findLatest` は `null` |
| TC-ports-backupStore-40 | B1 を取得した | `findById(B1)` と `findById(B2)` を呼ぶ | B1 の記録と `null` |
| TC-ports-backupStore-41 | `takenAt` が互いに異なる B1、B2、B3 を取得した | `findLatest` を呼ぶ | `takenAt` が最も新しい記録。取得の順によらない |
| TC-ports-backupStore-42 | `takenAt` が最も新しい記録の `integrity` が `failed`、それより古い記録が `verified` | `findLatest` を呼ぶ | `integrity` が `failed` の、最も新しい記録 |
| TC-ports-backupStore-43 | `takenAt` が同じ B1、B2 の記録と、それより古い B3 の記録がある | `findLatest` を呼ぶ | B2 の記録 |
| TC-ports-backupStore-44 | 記録が 3 件ある | `findLatest` と `{ page: 1, limit: 1 }` の `findPage` を呼ぶ | `findLatest` の記録は、`findPage` の先頭の記録と等しい |
| TC-ports-backupStore-45 | B1 と、`takenAt` がより新しい B2 を取得し、保管先で B2 のバックアップを削除した設置と、B1 だけを取得し、保管先で B1 のバックアップを削除した設置 | `findById`、`findLatest`、`findPage` を呼ぶ | 削除したバックアップは、`findById` が `null` を返し、`findLatest` と `findPage` に現れず、`count` に数えない。前者では、`findLatest` は B1 の記録、`findPage` は B1 の 1 件と `count` 1。後者では、`findLatest` は `null`、`findPage` は空の列と `count` 0 |

## findPage

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-backupStore-47 | バックアップが無い | `{ page: 1, limit: 20 }` で `findPage` を呼ぶ | 空の列と `count` 0 |
| TC-ports-backupStore-48 | B1 を取得した | `{ page: 1, limit: 20 }` で `findPage` を呼ぶ | B1 の記録と `count` 1 |
| TC-ports-backupStore-49 | `takenAt` が互いに異なる 3 件を、`takenAt` の昇順に取得した | `{ page: 1, limit: 20 }` で `findPage` を呼ぶ | `takenAt` の降順で返す |
| TC-ports-backupStore-50 | `takenAt` が同じ 3 件がある | `{ page: 1, limit: 20 }` で `findPage` を呼ぶ | ID の降順で返す |
| TC-ports-backupStore-51 | `integrity` が `verified` の記録と `failed` の記録がある | `{ page: 1, limit: 20 }` で `findPage` を呼ぶ | 両方を返す |
| TC-ports-backupStore-52 | 3 件ある | `{ page: 1, limit: 1 }` で `findPage` を呼ぶ | 並び順の先頭の 1 件と `count` 3 |
| TC-ports-backupStore-53 | 3 件ある | `{ page: 2, limit: 2 }` で `findPage` を呼ぶ | 並び順の 3 件目の 1 件と `count` 3 |
| TC-ports-backupStore-54 | 3 件ある | `{ page: 3, limit: 2 }` で `findPage` を呼ぶ | 空の列と `count` 3 |
| TC-ports-backupStore-55 | 101 件ある | `{ page: 1, limit: 100 }` と `{ page: 2, limit: 100 }` で `findPage` を呼ぶ | 1 ページ目は並び順の先頭から 100 件、2 ページ目は最後の 1 件。どちらも `count` は 101。重複も欠落も無い |

## バックアップの対象

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-backupStore-56 | W1 と W2 の区画の、`WorkspaceRepositories`（`spec/domains/common.md`）のどのリポジトリにも、集約または記録が一件以上ある。準備領域（`ContentStagingArea` の変換結果と、`MediaUsageRepository` の準備領域の使用箇所）にも書いてある。B1 を取得し、その後にすべてを消去して確定した | `restore(B1)` を呼ぶ | どのリポジトリの読み取りも、取得の時点の内容を返す |
| TC-ports-backupStore-57 | 設置の区画の、`InstallationRepositories`（`spec/domains/common.md`）のどのリポジトリにも、集約または記録が一件以上ある。B1 を取得し、その後に、`Installation` のほかを削除し、名簿の項目の状態を変えて確定した | `restore(B1)` を呼ぶ | どのリポジトリの読み取りも、取得の時点の内容を返す |
| TC-ports-backupStore-58 | 回答が、未送信、送信成功、送信失敗、対処済みの状態で 1 件ずつある。B1 を取得し、その後に回答を削除して確定した | `restore(B1)` を呼ぶ | 回答は、取得の時点の入力内容、通知の進行、宛先ごとの送信結果、`handling` を返す |
| TC-ports-backupStore-59 | `FormSettings` が `removed` のフォームの回答がある。B1 を取得し、その後に回答を削除して確定した | `restore(B1)` を呼ぶ | 回答と `removed` の `FormSettings` は、取得の時点の内容を返す |
| TC-ports-backupStore-60 | メディアの元のファイルと、生成済みの派生形式の実体がある。B1 を取得し、その後にメディアを削除して実体を消去した | `restore(B1)` を呼ぶ | 元のファイルと派生形式の実体を取り出せ、取得の時点の内容に等しい |
| TC-ports-backupStore-61 | W1 のメディアの実体 M1 がある状態で B1 を取得し、その後に W1 に実体 M2 を保管して記録を確定した | `restore(B1)` を呼ぶ | M1 は取り出せる。M2 は、保管先に残らず、取り出せない |
| TC-ports-backupStore-62 | W1 がある状態で B1 を取得し、その後に W2 を作って、W2 に実体を保管して記録を確定した | `restore(B1)` を呼ぶ | W2 の実体は、保管先に一件も残らない |
| TC-ports-backupStore-63 | B1 を取得し、その後に、どのメディアの記録も指さない実体を W1 の保管先に保管した | `restore(B1)` を呼ぶ | その実体は、保管先に残らない |
| TC-ports-backupStore-64 | W1 のメディアの実体 M1 がある状態で B1 を取得した。`restore(B1)` の前に、M1 の `remove` を打ち切った。試験用の相手が、その `remove` を、`restore(B1)` が M1 を保管した後に保管先へ届ける | `restore(B1)` を呼ぶ | 復旧の後、M1 を取り出せ、取得の時点の内容に等しい |
| TC-ports-backupStore-65 | コンテンツの編集中の記録 P がある状態で B1 を取得し、その後に P を取り除き、別の編集中の記録 Q を置いた | `restore(B1)` を呼び、`EditingPresenceStore` で読む | Q を返し、P を返さない。復旧の前と同じ内容 |
| TC-ports-backupStore-66 | 利用量の制限の記録がある状態で B1 を取得し、その後に利用を記録した | `restore(B1)` を呼び、`UsageLimiter` で判定する | 判定は、復旧の前と同じ |
| TC-ports-backupStore-67 | `MailConnection` と `WebhookEndpoints` が、保護した秘密情報を持つ。B1 を取得し、その後に秘密情報を置き換えて確定した | `restore(B1)` を呼ぶ | 各集約の秘密情報は、取得の時点の保護した値に等しい。`SecretCipher.reveal` は、取得の時点の元の値を返す |
| TC-ports-backupStore-68 | 確定済みで配送を待つドメインイベントがある。B1 を取得し、その後にイベントを配送し終えた | `restore(B1)` を呼ぶ | 取得の時点で配送を待っていたイベントは、復旧後も配送を待つ |
| TC-ports-backupStore-69 | W1 に、隔離されたドメインイベントが 1 件ある。B1 を取得し、その後に、W1 の区画のイベントを `deleteEvents` で消して確定した | `restore(B1)` を呼び、`PartitionReader.workspace(W1).quarantinedEventQuery.count()` を呼ぶ | 1 を返す |
| TC-ports-backupStore-70 | メディアの消去待ちの実体の記録と、保管の予約がある。B1 を取得し、その後に消去待ちの記録を取り除いて確定した | `restore(B1)` を呼ぶ | `ObjectRemovalRepository.findPending(null, 10)` は、取得の時点の消去待ちを返す。`StorageReservationRepository.count()` は 0。保管の予約は戻らない |
| TC-ports-backupStore-71 | メディア M の使用箇所の記録がある。B1 を取得し、その後に使用箇所を `MediaUsageRepository.replaceMany`（空の `uses`）で取り除いて確定した | `restore(B1)` を呼び、`MediaRepository.delete(M, v)` を呼ぶ | `ConflictError`（理由 `MEDIA_IN_USE`）。使用箇所の記録は、取得の時点の内容に戻る |
| TC-ports-backupStore-72 | B1 を取得し、その後に B2 を取得した | `restore(B1)` を呼ぶ | `findById(B2)` は B2 の記録、`findLatest` は B2 の記録、`findPage` は 2 件と `count` 2 を返す。B1 と B2 の記録の全フィールドは、復旧の前と等しい |

## verify

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-backupStore-74 | B1 を取得した | `verify(B1)` を二度呼ぶ | どちらも `intact: true` |
| TC-ports-backupStore-75 | B1 を取得し、その後に設置のデータを変更して確定した | `verify(B1)` を呼ぶ | `intact: true` |
| TC-ports-backupStore-76 | B1 を取得した | `verify(B1)` を呼び、その前後で各集約と `InstallationRepository.find` を読む | 設置のデータと `Installation` の `version` は変わらない |
| TC-ports-backupStore-77 | バックアップが無い設置と、B1 だけを取得した設置 | `verify(B2)` を呼ぶ | どちらも `NotFoundError` |

## restore

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-backupStore-79 | W1 がある状態で B1 を取得し、その後に W2 を作って確定した | `restore(B1)` を呼ぶ | W1 とその集約は取得の時点の内容を返す。W2 とその集約は一件も読めない |
| TC-ports-backupStore-80 | W1 と W2 がある状態で B1 を取得し、その後に W1 だけを変更して確定した | `restore(B1)` を呼ぶ | W1 と W2 のどちらも、取得の時点の内容を返す |
| TC-ports-backupStore-81 | W1 と W2 がある状態で B1 を取得し、その後に W2 のデータを消去して `Workspace` を `closed` にした | `restore(B1)` を呼ぶ | W2 とその集約は、取得の時点の内容を返す |
| TC-ports-backupStore-82 | B1 を取得し、その後に `Installation.completeUpdate` の結果を保存して `dataVersion` を進めた | `restore(B1)` を呼ぶ | `InstallationRepository.find` が返す `Installation` は、`initialConfiguration` と `dataVersion` が取得の時点の値 |
| TC-ports-backupStore-83 | B1 を取得した | `restore(B1)` を呼び、完了の直後に各集約を読む | 待たずに、どの読み取りも復旧後の内容を返す |
| TC-ports-backupStore-84 | B1 を取得した。B2 のバックアップは無い | `backupId` が B2 の `RestoringRecovery` を渡して `restore` を呼ぶ | `NotFoundError`。設置のデータは呼び出しの前の内容のまま |
| TC-ports-backupStore-85 | B1 を取得し、その後にデータを変更して確定した | `RecoveryJournal` に記録を保存せずに、`restore(B1)` を呼ぶ | 成功する。設置のデータは B1 の内容になる |
| TC-ports-backupStore-86 | B1 を取得した | `restore(B1)` を完了した後、もう一度 `restore(B1)` を呼ぶ | 成功する。設置のデータは B1 の内容になる |
| TC-ports-backupStore-87 | 空の設置。別の設置で取得した B1 が保管先にある | `restore(B1)` を呼ぶ | 成功する。設置のデータは B1 の内容になり、`InstallationRepository.find` は、B1 の中の `Installation` を返す |
| TC-ports-backupStore-88 | W1 と W2 がある状態で B1 を取得し、その後に W1 と W2 を変更して確定した。試験用の相手が、`restore(B1)` の置き換えを、始めた後に失敗させる | `restore(B1)` を呼ぶ | `SystemError`（理由 `RESTORE_FAILED`）で失敗する。設置の区画、W1 の区画、W2 の区画は、それぞれ、変更後の内容か、B1 の内容のどちらか。その間の内容の区画は無い |
| TC-ports-backupStore-89 | 同上の失敗の後 | 試験用の相手の指定を解いて、もう一度 `restore(B1)` を呼ぶ | 成功する。設置のデータは、W1 と W2 のどちらも B1 の内容に等しい。`RESTORE_IN_PROGRESS` にならない |
| TC-ports-backupStore-90 | B1 を取得し、データを変更して B2 を取得した。試験用の相手が、`restore(B1)` の置き換えを、始めた後に失敗させた | 試験用の相手の指定を解いて、`restore(B2)` を呼ぶ | 成功する。設置のデータは、すべての区画が B2 の内容に等しい |
| TC-ports-backupStore-91 | B1 を取得した | UnitOfWork の中で `restore(B1)` を呼び、その後に `fn` が例外を投げる | 設置のデータは B1 の内容になる。置き換えは取り消されない |
| TC-ports-backupStore-92 | B1 と B2 を取得した。試験用の相手が、`restore(B1)` の置き換えを進行中のまま止めている | `restore(B2)` を呼ぶ | `ConflictError`（理由 `RESTORE_IN_PROGRESS`）。設置のデータを変えない |
| TC-ports-backupStore-93 | 同上 | 試験用の相手が、止めていた `restore(B1)` を進める | `restore(B1)` は成功する。設置のデータは B1 の内容に等しい |
| TC-ports-backupStore-94 | B1 を取得した。取得中の B2 がある。試験用の相手が、`restore(B1)` の置き換えを進行中のまま止めている | `capture(B3)` を呼ぶ | `ConflictError`（理由 `RESTORE_IN_PROGRESS`）。`verify(B3)` は `NotFoundError`。取得中の B2 は消えず、`verify(B2)` は `intact: true` |
| TC-ports-backupStore-95 | 同上 | 試験用の相手が、止めていた `restore(B1)` を進め、完了の後に `capture(B3)` を呼ぶ | `restore(B1)` と `capture(B3)` は、どちらも成功する。B3 の写しは、B1 の内容に等しい |
| TC-ports-backupStore-96 | B1 を取得した後に、データを変更して確定した。試験用の相手が、`capture(B2)` の写しの取得を進行中のまま止めている | `restore(B1)` を呼ぶ | `ConflictError`（理由 `BACKUP_IN_PROGRESS`）。設置のデータは、変更後の内容のまま |
| TC-ports-backupStore-97 | 同上 | 試験用の相手が、止めていた `capture(B2)` を進め、`complete` を終えて、`restore(B2)` を呼ぶ | `capture(B2)` は成功する。復旧の後のデータは、変更後の内容に等しい。拒否した `restore` は、写しを変えていない |
| TC-ports-backupStore-98 | B1 を取得した。取得中の B2 がある（`capture(B2)` は終えている） | `restore(B1)` を呼ぶ | 成功する |
| TC-ports-backupStore-99 | B1 を取得し、その後にデータを変更して確定した。試験用の相手が、`restore(B1)` の置き換えを、`Recovery.restoreTimeoutMs` が過ぎるまで終えさせない | `restore(B1)` を呼ぶ | `Recovery.restoreTimeoutMs` の内に、`SystemError`（理由 `RESTORE_TIMED_OUT`）で失敗する。置き換えは進行中で残らない。試験用の相手の指定を解いて、続けて呼んだ `restore(B1)` は、`RESTORE_IN_PROGRESS` にならずに成功し、設置のデータは B1 の内容に等しい |
| TC-ports-backupStore-100 | B1 と B2 を取得した | `restore(B1)` の完了の後に、`restore(B2)` を呼ぶ | どちらも成功する。設置のデータは B2 の内容に等しい |

## 置き換えと書き込み

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-backupStore-101 | B1 を取得した。その後に、W1 の区画、または設置の区画の UnitOfWork を開いて集約 X を書き、確定の前で止めている | `restore(B1)` を完了させた後に、止めていた UnitOfWork を確定する | `run` が `ConflictError`（理由 `RESTORE_IN_PROGRESS`）を投げる。X は読めない。設置のデータは B1 の内容に等しい |
| TC-ports-backupStore-103 | B1 を取得した。試験用の相手が、`restore(B1)` の置き換えを進行中のまま止めている | 集約 X を書く UnitOfWork を開いて確定し、その後に、止めていた `restore(B1)` を進める | UnitOfWork は、確定するか、`run` が `ConflictError`（理由 `RESTORE_IN_PROGRESS`）を投げるかのどちらか。`restore(B1)` の完了の後、X は読めず、設置のデータは B1 の内容に等しい |
| TC-ports-backupStore-104 | B1 を取得し、`restore(B1)` を完了した | 完了の後に、集約 X を書く UnitOfWork を開いて確定する | 確定する。X を読める |
| TC-ports-backupStore-106 | B1 を取得した。試験用の相手が、`InstallationEnvironment.updateData` を進行中のまま止めている | `restore(B1)` を完了させた後に、止めていた `updateData` を進める | `updateData` は `ok: false`（`failed`）を返す。設置のデータは B1 の内容に等しい |
| TC-ports-backupStore-107 | 保存済みのデータが版 V1 の形の設置で B1 を取得し、`restore(B1)` を完了した。`InstallationEnvironment.dataVersion()` は V2 で、`supportsUpdateFrom(V1)` は真 | 完了の後に `InstallationEnvironment.updateData({ from: V1, to: V2 })` を呼ぶ | `ok: true` |
