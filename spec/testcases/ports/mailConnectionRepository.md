# MailConnectionRepository

`spec/domains/form.md` の `MailConnectionRepository` の契約と、`spec/domains/common.md` の「SingletonRepository」とリポジトリに共通する規則を、すべてのアダプターが通すケースで定める。

- 「UoW」は `UnitOfWorkProvider.run` の `fn`。「確定」は `fn` の正常終了を指す
- UoW の `WriteScope` は、どのケースも、対象のワークスペースの `{ kind: "workspace", workspaceId, condition: { kind: "none" } }`。区画と条件は、`unitOfWorkProvider.md` が確かめる
- リポジトリは、ワークスペースの区画に結び付く。W1 の送信接続設定の書き込みは W1 の UoW で、読み取りは `PartitionReader.workspace(W1)` のリポジトリで行う。W2 も同じ。メソッドは、ワークスペースの ID を取らない。「`find()`（W1）」は、W1 の区画のリポジトリの `find()` を指す
- 前提条件の書き込みは、別の UoW で確定済み。期待結果の読み取りは、UoW の外で行う
- W1 と W2 は別の、確定済みのワークスペース
- 一つの UoW の二つ目の書き込みには、同じ区画の `formSettingsRepository` の `applySchemaChange`（`created` に、`FormSettings.createForNewForm` の結果 G だけを渡す）を使う。下の「G の保存」は、この呼び出しを指す
- 送信接続設定は `MailConnection.register` の結果。変更は `MailConnection.update` が返した `changed` の `value`

## insert

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mailConnectionRepository-01 | 送信接続設定が無い | UoW で、W1 の送信接続設定を `insert` して確定する | `find()`（W1） が、渡した値と全フィールドが等しい `entity` と `expectedVersion` を返す |
| TC-ports-mailConnectionRepository-02 | 送信接続設定が無い | 認証情報の `identity` が `null` の設定を `insert` して確定する | `find()` が返す `credentials.identity` は `null` |
| TC-ports-mailConnectionRepository-03 | 送信接続設定が無い | 接続条件を 3 組持つ設定と、接続条件を 1 組持つ設定を、W1 と W2 に、それぞれの UoW で `insert` して確定する | `find()` が返す `endpoint` は、それぞれ渡した `destination` と、名前と値の組が一致する `conditions` |
| TC-ports-mailConnectionRepository-04 | 送信接続設定が無い | UoW で、認証情報の `identity` を持つ W1 の送信接続設定を `insert` して確定する | `find()`（W1） が返す `credentials` の `identity` と `secret` は、それぞれ渡した `ProtectedMailSecret` と同じ値 |
| TC-ports-mailConnectionRepository-05 | W1 の送信接続設定がある | UoW で、W1 の送信接続設定を `insert` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。`find()`（W1） は元の設定を返す |
| TC-ports-mailConnectionRepository-06 | W1 の送信接続設定がある | W2 の UoW で、W2 の送信接続設定を `insert` して確定する | 成功する。`find()`（W1）と `find()`（W2）が、それぞれの設定を返す |
| TC-ports-mailConnectionRepository-07 | W1 の送信接続設定がある | W1 の一つの UoW で、G の保存と、W1 の送信接続設定の `insert` を行う | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。G は保存されない |
| TC-ports-mailConnectionRepository-08 | 送信接続設定が無い | 二つの UoW が並行して、W1 の送信接続設定を `insert` する | 一方だけが確定する。他方の `run` は `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。`find()`（W1） は確定した側の値を返す |
| TC-ports-mailConnectionRepository-09 | W1 の送信接続設定を `deleteAll` で削除して確定した | UoW で、W1 の送信接続設定を `insert` して確定する | 成功する。`find()`（W1） は新しい設定を返す |

## find

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mailConnectionRepository-10 | W1 の送信接続設定が無い | `find()`（W1） を呼ぶ | `null` |
| TC-ports-mailConnectionRepository-11 | W1 の送信接続設定がある | `find()`（W1） を呼ぶ | その設定と `expectedVersion` を返す |
| TC-ports-mailConnectionRepository-12 | W1 の送信接続設定がある | `find()`（W2）を呼ぶ | `null` |
| TC-ports-mailConnectionRepository-13 | 区画に何も確定していないワークスペース | その区画のリポジトリで `find()` を呼び、その区画の UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼んで確定する | `find` は `null`、`deleteAll` は 0 を返す |
| TC-ports-mailConnectionRepository-14 | W1 の送信接続設定がある | UoW の外で `find()`（W1） を呼ぶ | UoW を開かずに設定を返す |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mailConnectionRepository-15 | W1 の送信接続設定を `find()` で読む | UoW で、接続先・接続条件・認証情報を置き換えた `update` の結果を、読んだ `expectedVersion` で `save` して確定する | `find()`（W1） が、変更後の全フィールドを返す |
| TC-ports-mailConnectionRepository-16 | W1 の送信接続設定を `find()` で読む | UoW で、`credentials` が `null` の `update` の結果を `save` して確定する | `find()`（W1） が返す `credentials` は、登録時の `credentials` と同じ値 |
| TC-ports-mailConnectionRepository-17 | W1 の送信接続設定を `save` して確定した | 保存の後に `find()` で読んだ `expectedVersion` で、次の変更を `save` して確定する | 成功する |
| TC-ports-mailConnectionRepository-18 | W1 の送信接続設定を二度 `find()` で読み、一方の変更を `save` して確定した | UoW で、他方の変更を、最初に読んだ `expectedVersion` で `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`find()`（W1） は先に確定した内容を返す |
| TC-ports-mailConnectionRepository-19 | W1 の送信接続設定を `find()` で読む | 二つの UoW が並行して、同じ `expectedVersion` で別の変更を `save` する | 一方だけが確定する。他方の `run` は `ConflictError`（理由 `STALE_VERSION`）を投げる |
| TC-ports-mailConnectionRepository-20 | W1 と W2 の送信接続設定がある | W1 の UoW で、W1 の変更を `save` して確定し、続けて別の W1 の UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼んで確定する | `find()`（W2）が返す設定は変わらない |
| TC-ports-mailConnectionRepository-21 | W1 の送信接続設定が無い（一度も登録していない、または `find()` で読んだ後に `deleteAll` で削除して確定した） | UoW で、W1 の送信接続設定を `save` する（読んだ、または任意の `expectedVersion`） | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`find()`（W1） は `null` |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mailConnectionRepository-23 | W1 の送信接続設定が無い（一度も登録していない、または `deleteAll` で削除して確定した） | UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼び、確定する | 0 を返す |
| TC-ports-mailConnectionRepository-24 | W1 の送信接続設定がある | UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼び、確定する | 1 を返す。`find()`（W1） は `null` |
| TC-ports-mailConnectionRepository-28 | W1 の送信接続設定を `save` で 3 回変更して確定した | UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼び、確定する | `expectedVersion` を渡さずに 1 を返す。`find()`（W1） は `null` |
| TC-ports-mailConnectionRepository-29 | W1 の区画の `Workspace` は `active`。W1 の送信接続設定がある | UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼び、確定する | ワークスペースの状態を確かめずに 1 を返す |
| TC-ports-mailConnectionRepository-30 | W1 の送信接続設定がある | UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼び、`fn` が例外を投げる | `find()`（W1） は元の設定を返す |
| TC-ports-mailConnectionRepository-31 | W1 の送信接続設定がある | UoW で `deleteAll(0)` を呼ぶ。別の UoW で、`BoundedDeletion.maxLimit` を超える `limit` で `deleteAll` を呼ぶ | どちらも `run` が `SystemError`（理由 `INVALID_QUERY`）で失敗する。`find()`（W1） は元の設定を返す |

## 可視性と UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mailConnectionRepository-34 | 送信接続設定が無い | UoW で W1 の設定を `insert` し、確定の直後に `find()`（W1） を呼ぶ | 待たずに、その設定を返す |
| TC-ports-mailConnectionRepository-35 | W1 の送信接続設定がある | W1 の一つの UoW で、W1 の `save` と G の保存を行い、確定する | 両方が反映される |
| TC-ports-mailConnectionRepository-36 | W1 の送信接続設定がある | W1 の一つの UoW で、W1 の `save` と G の保存を行い、`fn` が例外を投げる | `run` はその例外を投げる。`find()`（W1） は変更前で、G は保存されない |
| TC-ports-mailConnectionRepository-37 | W1 の送信接続設定を `find()` で読み、その後に別の変更を `save` して確定した | W1 の一つの UoW で、G の保存と、最初に読んだ `expectedVersion` での W1 の `save` を行う | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。G は保存されない |
