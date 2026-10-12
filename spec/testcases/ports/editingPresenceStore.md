# EditingPresenceStore

「上限」は、その節のメソッドの契約が定める、一度に渡せる件数・個数、または `limit` の上限を指す。値は、ポートを定めるドメインのファイルが持つ。

契約は [content.md](../../domains/content.md) の「EditingPresence」「ポート」と、[common.md](../../domains/common.md) の「TransactionalRepository」の共通の規則（件数を区切る削除、日時で区切る削除）・「UnitOfWorkProvider」が定める。

## 記法

- `W1`、`W2` はワークスペースの ID。`C1`、`C2` はコンテンツID。`m1`、`m2`、`m3` はメンバーの ID。ポートは区画に結び付かず、対象のワークスペースを引数で受ける
- 「（`W1`、`C1`、`m1`）の記録」は、`workspaceId` が `W1`、`contentId` が `C1`、`memberId` が `m1` の `EditingPresence`。記録は `put` で置く。コンテンツとメンバーは、どのポートにも保存しない
- 読み取りも書き込みも、`UnitOfWorkProvider.run` の UnitOfWork の外で呼ぶ。前提条件の書き込みは、呼び出しを完了済み
- `deleteExpired` の `now` は時点 `t`
- 記録が有効かどうかの判定と、`list` が返す順序は、このファイルで確かめない
- バックアップと復旧での記録の扱いの例は、[backupStore.md](backupStore.md) が持つ

## put と find

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-editingPresenceStore-01 | 記録が無い | `find`（`W1`、`C1`、`m1`）を呼ぶ | `null` を返す |
| TC-ports-editingPresenceStore-02 | 記録が無い | （`W1`、`C1`、`m1`）の記録を `put` する | `find`（`W1`、`C1`、`m1`）が、`workspaceId`、`contentId`、`memberId`、`memberName`、`since`、`expiresAt` が渡した値と等しい記録を返す |
| TC-ports-editingPresenceStore-03 | （`W1`、`C1`、`m1`）の記録がある | `since` と `expiresAt` が異なる、（`W1`、`C1`、`m1`）の記録を `put` する | `find`（`W1`、`C1`、`m1`）が、後に渡した記録を返す。`list`（`W1`、`C1`）が返す `m1` の記録は 1 件 |
| TC-ports-editingPresenceStore-04 | （`W1`、`C1`、`m1`）の記録がある | （`W1`、`C1`、`m2`）、（`W1`、`C2`、`m1`）の記録と、`since` と `expiresAt` が異なる（`W2`、`C1`、`m1`）の記録を `put` する | 4 つの組の `find` が、それぞれの組に置いた記録を返す |
| TC-ports-editingPresenceStore-07 | （`W1`、`C1`、`m1`）の記録がある | `find`（`W2`、`C1`、`m1`）を呼ぶ | `null` を返す |
| TC-ports-editingPresenceStore-08 | （`W1`、`C1`、`m1`）の記録がある。`expiresAt` は `t` より前 | `find`（`W1`、`C1`、`m1`）を呼ぶ | その記録を返す |
| TC-ports-editingPresenceStore-09 | `C1` のコンテンツを `ContentRepository` に保存していない | （`W1`、`C1`、`m1`）の記録を `put` する | 成功する。`find`（`W1`、`C1`、`m1`）がその記録を返す |
| TC-ports-editingPresenceStore-10 | 記録が無い | （`W1`、`C1`、`m1`）の記録を `put` し、呼び出しの完了の直後に `find` と `list` を呼ぶ | どちらも、置いた記録を返す |
| TC-ports-editingPresenceStore-11 | 記録が無い | （`W1`、`C1`、`m1`）の、内容が異なる二つの記録を、並行に `put` する | どちらも成功する。`find`（`W1`、`C1`、`m1`）は、二つのどちらか一方と等しい記録を返す。`list`（`W1`、`C1`）が返す `m1` の記録は 1 件 |

## list

断りが無い記録は、`W1` の `C1` の記録。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-editingPresenceStore-12 | 記録が無い | `list`（`W1`、`C1`）を呼ぶ | 空の列を返す |
| TC-ports-editingPresenceStore-13 | `m1`、`m2`、`m3` の記録がある | `list`（`W1`、`C1`）を呼ぶ | 3 件の記録を返す。それぞれ、`put` した値と等しい |
| TC-ports-editingPresenceStore-14 | `expiresAt` が `t` より後の `m1` と、`t` より前の `m2` の記録がある | `list`（`W1`、`C1`）を呼ぶ | `m1` と `m2` の記録を、どちらも返す |
| TC-ports-editingPresenceStore-15 | （`W1`、`C1`、`m1`）、（`W1`、`C2`、`m2`）、（`W2`、`C1`、`m3`）の記録がある | `list`（`W1`、`C1`）を呼ぶ | （`W1`、`C1`、`m1`）の記録だけを返す |

## remove

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-editingPresenceStore-17 | （`W1`、`C1`、`m1`）、（`W1`、`C1`、`m2`）、（`W1`、`C2`、`m1`）、（`W2`、`C1`、`m1`）の記録がある | `remove`（`W1`、`C1`、`m1`）を呼ぶ | `find`（`W1`、`C1`、`m1`）は `null`。ほかの三つの組の `find` は、それぞれの記録を返す |
| TC-ports-editingPresenceStore-18 | 記録が無い | `remove`（`W1`、`C1`、`m1`）を 2 回呼ぶ | 2 回とも成功する |
| TC-ports-editingPresenceStore-21 | （`W1`、`C1`、`m1`）の記録がある | `remove`（`W1`、`C1`、`m1`）の完了の直後に `find`（`W1`、`C1`、`m1`）と `list`（`W1`、`C1`）を呼ぶ | `find` は `null`、`list` は空の列を返す |

## deleteExpired

断りが無い記録は、`W1` の `C1` の記録。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-editingPresenceStore-22 | 記録が無い | `deleteExpired`（`t`、1000）を呼ぶ | 0 を返す |
| TC-ports-editingPresenceStore-23 | `expiresAt` が `t` より前の `m1`、`t` と等しい `m2`、`t` より後の `m3` の記録がある | `deleteExpired`（`t`、1000）を呼ぶ | 2 を返す。`find`（`W1`、`C1`、`m1`）と `find`（`W1`、`C1`、`m2`）は `null`。`find`（`W1`、`C1`、`m3`）は記録を返す |
| TC-ports-editingPresenceStore-25 | （`W1`、`C1`、`m1`）と（`W2`、`C2`、`m2`）の記録がある。どちらも `expiresAt` は `t` より前 | `deleteExpired`（`t`、1000）を呼ぶ | 2 を返す。どちらの `find` も `null` を返す |
| TC-ports-editingPresenceStore-26 | `expiresAt` が `t` より前の記録が 2 件ある | `deleteExpired`（`t`、2）を 2 回呼ぶ | 2、0 の順に返す |
| TC-ports-editingPresenceStore-27 | `expiresAt` が `t` より前の記録が 3 件、`t` より後の記録が 1 件ある | `deleteExpired`（`t`、2）を呼ぶ | 2 を返す。`expiresAt` が `t` より前の記録は 1 件残る。`t` より後の記録は残る |
| TC-ports-editingPresenceStore-28 | `expiresAt` が `t` より前の記録が 5 件ある | `deleteExpired`（`t`、2）を、返る件数が 2 より少なくなるまで繰り返す | 2、2、1 の順に返す。`expiresAt` が `t` より前の記録は残らない |
| TC-ports-editingPresenceStore-30 | `expiresAt` が `t` より前の記録が 1 件ある | `deleteExpired`（`t`、0）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる。記録は残る |
| TC-ports-editingPresenceStore-31 | `expiresAt` が `t` より前の記録が 1 件ある | `deleteExpired`（`t`、上限より 1 大きい `limit`）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる。記録は残る |

## deleteByWorkspace

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-editingPresenceStore-32 | `W1` の記録が無い | `deleteByWorkspace`（`W1`、10）を呼ぶ | 0 を返す |
| TC-ports-editingPresenceStore-33 | `W1` の `C1` に、`expiresAt` が `t` より後の記録が 2 件、`W1` の `C2` に、`t` より前の記録が 1 件ある | `deleteByWorkspace`（`W1`、10）を呼ぶ | 3 を返す。`W1` のどの `find` も `null` を返す |
| TC-ports-editingPresenceStore-35 | `W1` に記録が 5 件ある | `deleteByWorkspace`（`W1`、2）を、返る件数が 2 より少なくなるまで繰り返す | 2、2、1 の順に返す。`W1` のどの `find` も `null` を返す |
| TC-ports-editingPresenceStore-36 | （`W1`、`C1`、`m1`）、（`W1`、`C1`、`m2`）、（`W2`、`C1`、`m1`）の記録がある | `deleteByWorkspace`（`W1`、10）を呼ぶ | 2 を返す。`find`（`W2`、`C1`、`m1`）は記録を返す |
| TC-ports-editingPresenceStore-37 | `W1` に記録が 2 件ある | `deleteByWorkspace`（`W1`、2）を 2 回呼ぶ | 2、0 の順に返す |
| TC-ports-editingPresenceStore-38 | `W1` に記録が 1 件ある | `deleteByWorkspace`（`W1`、0）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる。記録は残る |
| TC-ports-editingPresenceStore-39 | `W1` に記録が 1 件ある | `deleteByWorkspace`（`W1`、上限より 1 大きい `limit`）を呼ぶ | `SystemError`（`INVALID_QUERY`）になる。記録は残る |
| TC-ports-editingPresenceStore-40 | （`W1`、`C1`、`m1`）の記録を `deleteByWorkspace`（`W1`、10）で削除済み | `expiresAt` が `t` より前の（`W1`、`C1`、`m1`）の記録を `put` する | 成功する。`find`（`W1`、`C1`、`m1`）と `list`（`W1`、`C1`）が、置いた記録を返す |
| TC-ports-editingPresenceStore-41 | （`W1`、`C1`、`m1`）の記録を `deleteByWorkspace`（`W1`、10）で削除した後、`expiresAt` が `t` より前の（`W1`、`C1`、`m1`）の記録を `put` した | `deleteExpired`（`t`、10）を呼ぶ | 1 を返す。`find`（`W1`、`C1`、`m1`）は `null` を返す |

## UnitOfWork

[common.md](../../domains/common.md) の「UnitOfWorkProvider」の、UnitOfWork に属さないポートの例。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-editingPresenceStore-42 | 記録が無い | `UnitOfWorkProvider.run` の `fn` の中で（`W1`、`C1`、`m1`）の記録を `put` した後、`fn` が例外を投げる | `run` はその例外を投げる。`find`（`W1`、`C1`、`m1`）は、置いた記録を返す |
| TC-ports-editingPresenceStore-43 | （`W1`、`C1`、`m1`）の記録がある | `UnitOfWorkProvider.run` の `fn` の中で `remove`（`W1`、`C1`、`m1`）を呼んだ後、`fn` が例外を投げる | `run` はその例外を投げる。`find`（`W1`、`C1`、`m1`）は `null` を返す |
| TC-ports-editingPresenceStore-44 | `expiresAt` が `t` より前の記録が 1 件ある | `UnitOfWorkProvider.run` の `fn` の中で `deleteExpired`（`t`、10）を呼んだ後、`fn` が例外を投げる | `run` はその例外を投げる。その記録の `find` は `null` を返す |
| TC-ports-editingPresenceStore-45 | `W1` に記録が 1 件ある | `UnitOfWorkProvider.run` の `fn` の中で `deleteByWorkspace`（`W1`、10）を呼んだ後、`fn` が例外を投げる | `run` はその例外を投げる。その記録の `find` は `null` を返す |
