# PreviewSettingsRepository

契約は [token.md](../../domains/token.md) の「ポート」と「トランザクション境界」、[common.md](../../domains/common.md) の `SingletonRepository`、`TransactionalRepository` の共通の規則、`UnitOfWorkProvider` が定める。

## 前提

- 設定は `PreviewSettings.create(now)` で作り、`setUrl` / `clearUrl` で変える。区画に一つの集約で、区画が識別する。ワークスペースの ID のフィールドを持たない。「W1 の設定」は、W1 の区画に保存する設定を指す
- 書き込み（`insert` / `save` / `deleteAll`）は `UnitOfWorkProvider.run` の中で、文脈のリポジトリで呼ぶ。`WriteScope` は、そのワークスペースの `workspace` で、条件は `none`。断りが無ければ、一つの操作を一つの UnitOfWork で確定し、読み取りは UnitOfWork の外で呼ぶ
- 「保存済み」は、`insert` を確定した状態を指す。`v` は、直前の `find` が返した `expectedVersion` を指す
- 「並行して」は、二つの UnitOfWork を同時に始め、どちらも確定まで進めることを指す
- 競合と違反は、メソッドの呼び出しか確定のどちらかで現れる。`run` が `ConflictError` を投げることと、その UnitOfWork の書き込みが一つも反映されないことで確かめる
- リポジトリは、ワークスペースの区画に結び付き、メソッドはワークスペースの ID を引数に取らない。W1、W2 は別のワークスペース。S1、S2 は、それぞれのワークスペースの設定。断りの無い呼び出しは、W1 の区画のリポジトリに対して行う。S2 の読み書きと、「W2 の区画」と書いた呼び出しは、W2 の区画のリポジトリに対して行う
- 同じ区画のほかの書き込みには、`tokenRepository` を使う。T は、W1 のトークン

## insert

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-previewSettingsRepository-01 | 設定が無い | `unset` の S1 を `insert` する | `find()` が、`status` が `unset` で S1 と等しいエンティティを返す |
| TC-ports-previewSettingsRepository-02 | 設定が無い | `create` に続けて `setUrl` を適用した `configured` の S1 を `insert` する | `find()` が、`status` が `configured` で、`baseUrl` を含めて S1 と等しいエンティティを返す |
| TC-ports-previewSettingsRepository-03 | S1 が保存済み | W1 の設定をもう一つ `insert` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。`find()` は S1 を返す |
| TC-ports-previewSettingsRepository-04 | S1 が保存済み | W2 の区画で、S2 を `insert` する | 確定する。W1 の区画の `find()` は S1、W2 の区画の `find()` は S2 を返す |
| TC-ports-previewSettingsRepository-05 | 設定が無い | W1 の `unset` の設定と、W1 の `configured` の設定を、並行して `insert` する | 一方だけが確定し、他方の `run` は `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。`find()` は確定した方を返す |
| TC-ports-previewSettingsRepository-06 | S1 を保存し、`deleteAll(10)` を確定してある | W1 の設定を `insert` する | 確定する。`find()` はその設定を返す |

## find

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-previewSettingsRepository-07 | W1 に設定が無い。区画に何も確定していないワークスペース Z | W1 の区画と Z の区画で、それぞれ `find()` を呼ぶ | どちらも `null` を返す |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-previewSettingsRepository-11 | `unset` の S1 が保存済み。`find` で S1 と `v` を得ている | `setUrl` の結果を `save` する | `find()` が、`status` が `configured`、`baseUrl` が設定した URL、`version` が S1 より 1 大きいエンティティを返す |
| TC-ports-previewSettingsRepository-13 | `configured` の S1 が保存済み。`find` で S1 と `v` を得ている | `clearUrl` の結果を `save` する | `find()` が、`status` が `unset` で、`baseUrl` を持たないエンティティを返す |
| TC-ports-previewSettingsRepository-14 | S1 と S2 が保存済み | S1 の `setUrl` の結果を `save` する | W2 の区画の `find()` は、変わっていない S2 を返す |
| TC-ports-previewSettingsRepository-15 | S1 が保存済み。`save` を一度確定してある | 確定の後に `find` で得た `v` で、もう一度 `save` する | 確定する |
| TC-ports-previewSettingsRepository-16 | S1 が保存済み。`find` で S1 と `v` を得た後、別の UnitOfWork が S1 の `setUrl` を `save` して確定した | `v` で、別の URL の `setUrl` の結果を `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`find()` の `baseUrl` は、先に確定した URL |
| TC-ports-previewSettingsRepository-17 | `configured` の S1 が保存済み。二つの呼び出し側が、同じ `v` を得ている | 別の URL の `setUrl` の `save` と、`clearUrl` の `save` を、並行して呼ぶ | 一方だけが確定し、他方の `run` は `ConflictError`（理由 `STALE_VERSION`）を投げる。`find()` は確定した方の内容を返す |
| TC-ports-previewSettingsRepository-18 | `unset` の S1 が保存済み。`find` で S1 と `v` を得た後、別の UnitOfWork が `deleteAll(10)` を確定した | `v` で `setUrl` の結果を `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`find()` は `null` |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-previewSettingsRepository-19 | 設定が無い。または、S1 を保存し、`deleteAll(10)` を一度確定した | `deleteAll(10)` を呼ぶ | 0 を返す |
| TC-ports-previewSettingsRepository-20 | S1 が保存済み | `deleteAll(10)` を呼ぶ。別の前提で、`deleteAll(1)` を呼ぶ | どちらも 1 を返す。`find()` は `null` |
| TC-ports-previewSettingsRepository-22 | S1 と S2 が保存済み | W1 の区画で `deleteAll(10)` を呼ぶ | 1 を返す。W2 の区画の `find()` は S2 を返す |
| TC-ports-previewSettingsRepository-23 | `unset` の S1 が保存済み。`find` で `v` を得た後、別の UnitOfWork が S1 の `setUrl` の `save` を確定した | `deleteAll(10)` を呼ぶ | 1 を返す。`find()` は `null` |
| TC-ports-previewSettingsRepository-25 | S1 が保存済み | `deleteAll(BoundedDeletion.maxLimit)` を呼ぶ | 1 を返す。`find()` は `null` |
| TC-ports-previewSettingsRepository-26 | S1 が保存済み | `deleteAll(0)` と、`BoundedDeletion.maxLimit` を超える `limit` の `deleteAll` を、それぞれ呼ぶ | どちらも `SystemError`（理由 `INVALID_QUERY`）になる。`find()` は S1 を返す |

## UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-previewSettingsRepository-28 | `unset` の S1 が保存済み | 一つの UnitOfWork で、T の `tokenRepository.insert` と、S1 の `setUrl` の `save` を呼び、確定する | `tokenRepository.findById(T.id)` は T、`find()` は `configured` の S1 を返す |
| TC-ports-previewSettingsRepository-29 | `unset` の S1 が保存済み | 一つの UnitOfWork で、同じ 2 つの書き込みを呼んだ後、`fn` が例外を投げる | `tokenRepository.findById(T.id)` は `null`、`find()` は `unset` の S1 を返す |
| TC-ports-previewSettingsRepository-30 | S1 が保存済み | 一つの UnitOfWork で、T の `tokenRepository.insert` と、W1 の設定の `insert` を呼ぶ | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。`tokenRepository.findById(T.id)` は `null` |
| TC-ports-previewSettingsRepository-31 | `unset` の S1 が保存済み。`find` で `v` を得た後、別の UnitOfWork が S1 の `setUrl` の `save` を確定した | 一つの UnitOfWork で、T の `tokenRepository.insert` と、`v` での S1 の `setUrl` の `save` を呼ぶ | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`tokenRepository.findById(T.id)` は `null` |
| TC-ports-previewSettingsRepository-32 | S1 が保存済み | 一つの UnitOfWork の中で、`deleteAll(10)` を呼んだ後、`fn` が例外を投げる | `find()` は S1 を返す |
