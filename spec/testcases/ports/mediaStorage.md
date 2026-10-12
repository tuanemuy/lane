# MediaStorage

契約は [media.md](../../domains/media.md) の「ポート」と、[common.md](../../domains/common.md) の共通の規則（`limit` の範囲）が定める。[common.md](../../domains/common.md) が定める試験用の相手に対して実行する。このポートは区画の外にあり、すべてのワークスペースの実体を一つのポートで扱う。

## 記法

- `ref` は `MediaObjectRef`（`MediaObjectRef.of` で作る）。ワークスペース W のメディア M の元のファイルを指す `ref` を O(W, M)、キー k・世代 g の派生形式を指す `ref` を D(W, M, k, g) と書く。g1、g2、g3 は、互いに違う世代（`DerivativeGeneration`）
- 「試み」は、`put` が返した `StoredAttempt`。「`ref` に内容 a を保管済み」は、内容 a の `put` と、返った試みの `commit` を完了していることを指す。「`ref` に内容 a の試みがある」は、`put` を完了し、`commit` していないことを指す
- 「実体が無い」は、その `ref` に、実体も試みも無いことを指す
- 「打ち切った `put` の内容が届いている」は、`put` に応答せず、打ち切りの後に書き込みを届けるよう指定した試験用の相手に対して、内容 a の `put` が打ち切りで失敗し、その後に試験用の相手が書き込みを届けた状態を指す。「まだ届いていない」は、同じ `put` が打ち切りで失敗し、試験用の相手が書き込みを届ける前の状態を指す
- 「打ち切った `commit` がまだ届いていない」は、最初の `commit` の呼び出しにだけ応答せず、打ち切りの後に確定を届けるよう指定した試験用の相手に対して、試みの `commit` が打ち切りで失敗し、試験用の相手が確定を届ける前の状態を指す。後の呼び出しには、指定の無い試験用の相手と同じに応答する
- 「打ち切った消去がまだ届いていない」は、最初の消去の呼び出しにだけ応答せず、打ち切りの後に、その消去を届けるよう指定した試験用の相手に対して、その呼び出しが打ち切りで失敗し、試験用の相手が消去を届ける前の状態を指す。後の呼び出しには、指定の無い試験用の相手と同じに応答する
- 打ち切りを確かめるケースは、打ち切りの時間を短い値にしたアダプターを使う。期限を確かめるケースは、試みの期限を短い値にしたアダプターを使い、「期限を過ぎた」は、その時間が過ぎたことを指す
- `UnitOfWorkProvider.run` は、W の `WriteScope`（`{ kind: "workspace", workspaceId: W, condition: { kind: "none" } }`）で呼ぶ

## 対象にしない

- 契約がアダプターごとの検証の対象にする項目
- 既定の打ち切りの時間（`StoragePolicy.callTimeoutMs`）と、既定の試みの期限（`StoragePolicy.attemptLifetimeMs`）そのものの長さ

## put と commit と get

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaStorage-01 | 実体が無い | `get(O(W, M))` を呼ぶ | `null` を返す |
| TC-ports-mediaStorage-02 | 実体が無い | `put(O(W, M), { bytes })` を呼び、完了の後に `get(O(W, M))` を呼ぶ | `put` は、`ref` が O(W, M) の試みを返す。`get` は `null` を返す。確定していない試みの内容は、実体でない |
| TC-ports-mediaStorage-03 | O(W, M) に内容 a の試みがある | 試みを `commit` し、完了の後に `get(O(W, M))` を呼ぶ | `bytes` がバイト列として等しい実体を返す |
| TC-ports-mediaStorage-04 | 実体が無い | D(W, M, k, g1) に `put` して `commit` し、`get(D(W, M, k, g1))` を呼ぶ | 渡した `bytes` の実体を返す |
| TC-ports-mediaStorage-05 | 実体が無い | 0x00 から 0xFF のすべての値を含む `bytes` を保管し、`get` する | 内容が等しい `bytes` を返す |
| TC-ports-mediaStorage-06 | O(W, M) に保管済み | `get` を 2 回呼ぶ | どちらも同じ内容を返す |
| TC-ports-mediaStorage-07 | O(W, M) に保管済み。確定に使った試みがある | 同じ試みをもう一度 `commit` する | 成功する。`get` は同じ内容を返す |
| TC-ports-mediaStorage-08 | `Media` の記録が無い | `MediaRepository` を呼ばずに O(W, M) に保管し、`get` する | 渡した内容を返す。`Media` の記録の有無を確かめない |

## ref と実体の対応

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaStorage-09 | O(W, M) と D(W, M, k, g1)、D(W, M, k, g1) と D(W, M, k, g2)、D(W, M, k1, g1) と D(W, M, k2, g1)、O(W, M1) と O(W, M2) の組ごとに、一方に内容 a、もう一方に内容 b を保管済み | それぞれ `get` する | どの組も、それぞれ a、b を返す |
| TC-ports-mediaStorage-13 | O(W1, M) に内容 a を保管済み | `get(O(W2, M))` を呼ぶ | `null` を返す |
| TC-ports-mediaStorage-14 | D(W, M, k, g1) に保管済み | `get(D(W, M, k, g2))` と `get(O(W, M))` を呼ぶ | どちらも `null` を返す |

## 試み

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaStorage-15 | 実体が無い | O(W, M) に、内容 a の `put` と内容 b の `put` を順に呼ぶ | どちらも成功し、別の試みを返す。`get` は `null` を返す |
| TC-ports-mediaStorage-16 | O(W, M) に内容 a の試みと内容 b の試みがある | b の試みを `commit` し、`get` する | 内容 b を返す |
| TC-ports-mediaStorage-17 | O(W, M) に内容 a の試みと内容 b の試みがある。b の試みを `commit` 済み | a の試みを `commit` し、`get` する | `commit` は `ConflictError`（理由 `OBJECT_EXISTS`）を投げる。`get` は内容 b を返す |
| TC-ports-mediaStorage-18 | 上の `OBJECT_EXISTS` の後。W に、ほかの実体も試みも無い | `removeByWorkspace(W, 10)` を呼ぶ | 1 を返す。取り除いたのは内容 b の実体だけで、拒否した試みの内容は残っていない |
| TC-ports-mediaStorage-19 | D(W, M, k, g1) に内容 a を保管済み | 同じ `ref` に、内容 b を `put` する | `put` は成功する。`get` は、内容 a を返す |
| TC-ports-mediaStorage-20 | O(W, M) に内容 a の試みと内容 b の試みがある。b の試みを `commit` 済みで、a の試みを `discard` 済み | a の試みを `commit` する | `ConflictError`（理由 `OBJECT_EXISTS`）を投げる。実体がある `ref` では、試みの内容が無くても `OBJECT_EXISTS` を先に返す。`get` は内容 b を返す |
| TC-ports-mediaStorage-21 | D(W, M, k, g1) に内容 a を保管済み。同じ `ref` に、同じ内容 a の試みがある | その試みを `commit` する | `ConflictError`（理由 `OBJECT_EXISTS`）を投げる。内容が同じでも拒否する。`get` は内容 a を返す |
| TC-ports-mediaStorage-22 | D(W, M, k, g1) に保管済み | D(W, M, k, g2) に `put` して `commit` する | 成功する。`ref` が異なれば拒否しない |
| TC-ports-mediaStorage-23 | O(W, M) に内容 a の試みがある | 試みを `discard` し、同じ試みを `commit` する | `discard` は成功する。`commit` は `ConflictError`（理由 `ATTEMPT_NOT_FOUND`）を投げる。`get` は `null` を返す |
| TC-ports-mediaStorage-24 | O(W, M) に内容 a の試みと内容 b の試みがある | a の試みを `discard` し、b の試みを `commit` して `get` する | 内容 b を返す。`discard` は、ほかの試みを変えない |
| TC-ports-mediaStorage-25 | O(W, M) に内容 a を保管済み。確定に使った試みがある | その試みを `discard` し、`get` する | `discard` は成功する。`get` は内容 a を返す。確定済みの試みの `discard` は、実体を取り除かない |
| TC-ports-mediaStorage-26 | O(W, M) に内容 a の試みがあり、`discard` 済み | 同じ試みをもう一度 `discard` する | 成功する |

## 失敗と打ち切り

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaStorage-27 | 実体が無い。試験用の相手を、`put` の途中で失敗するよう指定している | `put(O(W, M), …)` を呼び、失敗の後に `get(O(W, M))` を呼ぶ | `put` は試みを返さずに失敗する。`get` は `null` を返す |
| TC-ports-mediaStorage-28 | 上の失敗の後 | 指定を解いて、同じ `ref` に保管し、`get` する | 成功する。`get` は、後に渡した内容を返す |
| TC-ports-mediaStorage-29 | 実体が無い。応答しないよう指定した試験用の相手がある | `put(O(W, M), …)` を呼び、失敗の後に `get(O(W, M))` を呼ぶ | `put` は、打ち切りの時間の後に `SystemError`（理由 `STORAGE_TIMED_OUT`）で失敗し、試みを返さない。`get` は `null` を返す |
| TC-ports-mediaStorage-30 | O(W, M) に、打ち切った `put` の内容が届いている | `get(O(W, M))` を呼ぶ | `null` を返す。打ち切った `put` の内容は、後から届いても実体にならない |
| TC-ports-mediaStorage-31 | O(W, M) に、打ち切った `put` の内容が届いている | 指定を解いて、同じ `ref` に内容 b を保管し、`get` する | 成功する。`get` は内容 b を返す |
| TC-ports-mediaStorage-32 | O(W, M) に、打ち切った `put` の内容が届いている。ほかに W の実体も試みも無い | `remove(O(W, M))` または `removeAll(W, M)` を呼び、続けて `removeByWorkspace(W, 10)` を呼ぶ | `removeByWorkspace` は 0 を返す |
| TC-ports-mediaStorage-34 | 同上 | `removeByWorkspace(W, 10)` を 2 回呼ぶ | 1 回目は 1、2 回目は 0 を返す。届いた内容は、試み一つの内容として一件に数える |
| TC-ports-mediaStorage-35 | O(W, M) への打ち切った `put` の内容が、まだ届いていない。ほかに W の実体も試みも無い | `remove(O(W, M))` または `removeAll(W, M)` を呼び、完了の後に試験用の相手が書き込みを届けてから、`get(O(W, M))` と `removeByWorkspace(W, 10)` を呼ぶ | `get` は `null` を返す。`removeByWorkspace` は 1 を返す |
| TC-ports-mediaStorage-37 | 同上 | `removeByWorkspace(W, 10)` を呼び、完了の後に試験用の相手が書き込みを届けてから、`get(O(W, M))` と、もう一度 `removeByWorkspace(W, 10)` を呼ぶ | 1 回目の `removeByWorkspace` は 0 を返す。`get` は `null` を返す。2 回目の `removeByWorkspace` は 1 を返す |
| TC-ports-mediaStorage-38 | O(W, M) に内容 a を保管済み。`get` に応答しないよう指定した試験用の相手がある | `get(O(W, M))` を呼ぶ | 打ち切りの時間の後に `SystemError`（理由 `STORAGE_TIMED_OUT`）で失敗する。実体を返さない |
| TC-ports-mediaStorage-39 | O(W, M) に内容 a を保管済み。消去に応答しないよう指定した試験用の相手がある | `remove(O(W, M))`、`removeAll(W, M)`、`removeByWorkspace(W, 10)` を、それぞれ呼ぶ | どれも、打ち切りの時間の後に `SystemError`（理由 `STORAGE_TIMED_OUT`）で失敗する |
| TC-ports-mediaStorage-40 | 上の打ち切りの後 | 指定を解いて、`remove(O(W, M))` をもう一度呼び、`get(O(W, M))` を呼ぶ | `remove` は成功する。`get` は `null` を返す。打ち切った消去は、同じ呼び出しでやり直せる |
| TC-ports-mediaStorage-41 | O(W, M) に内容 a の試みがある。`discard` に応答しないよう指定した試験用の相手がある | 試みを `discard` する | 打ち切りの時間の後に `SystemError`（理由 `STORAGE_TIMED_OUT`）で失敗する。指定を解いた後の `discard` は成功し、その試みの `commit` は `ConflictError`（理由 `ATTEMPT_NOT_FOUND`）を投げる |
| TC-ports-mediaStorage-42 | O(W, M) に内容 a の試みがある。`commit` に応答しないよう指定した試験用の相手がある | 試みを `commit` する | 打ち切りの時間の後に `SystemError`（理由 `STORAGE_TIMED_OUT`）で失敗する |
| TC-ports-mediaStorage-43 | O(W, M) に内容 a の試みがある。`commit` に応答せず、打ち切りの後に確定を届けるよう指定した試験用の相手がある | `commit` が打ち切りで失敗した後、試験用の相手が確定を届けてから `get(O(W, M))` を呼ぶ | 内容 a を返す。打ち切った `commit` が後から成り立った結果は、成り立つ時点に呼んだ `commit` と同じ |
| TC-ports-mediaStorage-44 | O(W, M) に内容 a の試みがある。打ち切った `commit` がまだ届いていない | `remove(O(W, M))`、`removeAll(W, M)`、`removeByWorkspace(W, 10)` のいずれかを呼び、完了の後に試験用の相手が確定を届けてから `get(O(W, M))` を呼ぶ | どれも `null` を返す |
| TC-ports-mediaStorage-47 | O(W, M) に内容 a の試みがある。打ち切った `commit` がまだ届いていない。ほかに W の実体も試みも無い | 同じ `ref` に内容 b を保管し、完了の後に試験用の相手が確定を届けてから、`get(O(W, M))` と `removeByWorkspace(W, 10)` を呼ぶ | `get` は内容 b を返す。`removeByWorkspace` は 1 を返す。後から成り立った `commit` は、`OBJECT_EXISTS` と同じに、保管済みの実体を変えず、その試みの内容を取り除く |
| TC-ports-mediaStorage-48 | O(W, M) に内容 a を保管済み。`remove(O(W, M))`、`removeAll(W, M)`、`removeByWorkspace(W, 10)` のいずれかの打ち切った消去がまだ届いていない | 同じ消去をもう一度呼び、完了の後に同じ `ref` に内容 b を保管し、試験用の相手が消去を届けてから `get(O(W, M))` を呼ぶ | どれも内容 b を返す |
| TC-ports-mediaStorage-51 | O(W, M) に内容 a の試みがある。`remove(O(W, M))` の打ち切った消去がまだ届いていない | その試みを `commit` し、完了の後に試験用の相手が消去を届けてから `get(O(W, M))` を呼ぶ | `commit` は成功する。`get` は `null` を返す。呼び出しの時点でメディアの保管先にあった試みが後から確定した実体は、後から成り立った消去が取り除く |

## 試みの期限

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaStorage-52 | O(W, M) に内容 a の試みがあり、期限の内 | 試みを `commit` し、`get` する | 成功する。`get` は内容 a を返す |
| TC-ports-mediaStorage-53 | O(W, M) に内容 a の試みがあり、期限を過ぎた | 試みを `commit` し、`get` する | `commit` は `ConflictError`（理由 `ATTEMPT_NOT_FOUND`）を投げる。`get` は `null` を返す |
| TC-ports-mediaStorage-54 | O(W, M) に内容 a の試みがあり、期限を過ぎた。ほかに W の実体も試みも無い | `removeByWorkspace(W, 10)` を呼ぶ | 0 を返す。期限を過ぎた試みの内容は、件数に入らない |
| TC-ports-mediaStorage-55 | O(W, M) に内容 a の試みがあり、期限を過ぎた | 試みを `discard` する | 成功する |
| TC-ports-mediaStorage-56 | O(W, M) に内容 a の試みと内容 b の試みがある。b の試みを `commit` 済みで、a の試みだけが期限を過ぎた | a の試みを `commit` する | `ConflictError`（理由 `OBJECT_EXISTS`）を投げる。`get` は内容 b を返す |
| TC-ports-mediaStorage-57 | O(W, M) に、打ち切った `put` の内容が届いていて、届いてから期限を過ぎた。ほかに W の実体も試みも無い | `removeByWorkspace(W, 10)` を呼ぶ | 0 を返す |
| TC-ports-mediaStorage-58 | O(W, M) に内容 a を保管済み。保管してから、試みの期限と同じ時間が過ぎた | `get(O(W, M))` と `removeByWorkspace(W, 10)` を呼ぶ | `get` は内容 a を返す。`removeByWorkspace` は 1 を返す。実体は期限を持たない |
| TC-ports-mediaStorage-59 | O(W, M) に内容 a の試みと内容 b の試みがある。a の試みだけが期限を過ぎた | b の試みを `commit` し、`get` する | 成功する。`get` は内容 b を返す |

## remove

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaStorage-60 | D(W, M, k, g1) に保管済み | `remove(D(W, M, k, g1))` を呼び、`get` する | `remove` は成功する。`get` は `null` を返す |
| TC-ports-mediaStorage-61 | O(W, M) の実体が無い。一度も保管していない場合と、`remove` 済みの場合を確かめる | `remove(O(W, M))` を呼ぶ | 成功する |
| TC-ports-mediaStorage-63 | O(W, M)、D(W, M, k, g1)、D(W, M, k, g2) に保管済み | `remove(D(W, M, k, g1))` を呼ぶ | `get(O(W, M))` と `get(D(W, M, k, g2))` は、保管した内容を返す |
| TC-ports-mediaStorage-64 | D(W, M, k, g1) に内容 a を保管し、`remove` 済み | 同じ `ref` に内容 b を保管し、`get` する | 成功する。`get` は内容 b を返す |
| TC-ports-mediaStorage-65 | D(W, M, k, g1) に内容 a の試みがある | `remove(D(W, M, k, g1))` を呼び、その試みを `commit` する | `commit` は `ConflictError`（理由 `ATTEMPT_NOT_FOUND`）を投げる。`get` は `null` を返す。`remove` は、確定していない試みの内容も取り除く |
| TC-ports-mediaStorage-66 | D(W, M, k, g1) に内容 a を保管済み。確定に使った試みがある | `remove(D(W, M, k, g1))` を呼び、その試みをもう一度 `commit` する | `ConflictError`（理由 `ATTEMPT_NOT_FOUND`）を投げる。`get` は `null` を返す。取り除いた実体は、同じ試みから戻らない |
| TC-ports-mediaStorage-67 | D(W, M, k, g1) と D(W, M, k, g2) に、それぞれ試みがある | `remove(D(W, M, k, g1))` を呼び、D(W, M, k, g2) の試みを `commit` する | `commit` は成功する。`remove` は、ほかの `ref` の試みを取り除かない |

## removeAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaStorage-68 | O(W, M)、D(W, M, k1, g1)、D(W, M, k1, g2)、D(W, M, k2, g3) に保管済み | `removeAll(W, M)` を呼び、4 つの `ref` を `get` する | `removeAll` は成功する。`get` はどれも `null` を返す |
| TC-ports-mediaStorage-69 | D(W, M, k, g1) だけに保管済み。元のファイルの実体は無い | `removeAll(W, M)` を呼び、`get(D(W, M, k, g1))` を呼ぶ | `null` を返す |
| TC-ports-mediaStorage-70 | M の実体が無い。一度も保管していない場合と、`removeAll(W, M)` を完了済みの場合を確かめる | `removeAll(W, M)` を呼ぶ | 成功する |
| TC-ports-mediaStorage-72 | O(W, M1) と D(W, M1, k, g1)、O(W, M2) と D(W, M2, k, g1) に保管済み | `removeAll(W, M1)` を呼ぶ | M1 の `get` はどちらも `null`。M2 の `get` は、保管した内容を返す |
| TC-ports-mediaStorage-73 | O(W1, M) と O(W2, M) に保管済み | `removeAll(W1, M)` を呼ぶ | `get(O(W1, M))` は `null`、`get(O(W2, M))` は保管した内容を返す |
| TC-ports-mediaStorage-74 | O(W, M) に内容 a を保管し、`removeAll(W, M)` を完了済み | O(W, M) に内容 b を保管し、`get` する | 成功する。`get` は内容 b を返す |
| TC-ports-mediaStorage-75 | O(W, M) と D(W, M, k, g1) に、それぞれ試みがある | `removeAll(W, M)` を呼び、二つの試みを `commit` する | どちらの `commit` も `ConflictError`（理由 `ATTEMPT_NOT_FOUND`）を投げる。`get` はどちらも `null` を返す |
| TC-ports-mediaStorage-76 | O(W, M1) と O(W, M2) に、それぞれ試みがある | `removeAll(W, M1)` を呼び、O(W, M2) の試みを `commit` する | `commit` は成功する |

## removeByWorkspace

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaStorage-77 | W の実体が無い | `removeByWorkspace(W, 10)` を呼ぶ | 0 を返す |
| TC-ports-mediaStorage-78 | W に、O(W, M1)、D(W, M1, k, g1)、O(W, M2) を保管済み | `removeByWorkspace(W, 10)` を呼び、3 つの `ref` を `get` する | 3 を返す。`get` はどれも `null` を返す |
| TC-ports-mediaStorage-79 | W に実体を 5 件保管済み | `removeByWorkspace(W, 2)` を、0 が返るまで繰り返す | 返す値は順に 2、2、1、0。1 回目の後に `get` で読める実体は 3 件、最後は 0 件 |
| TC-ports-mediaStorage-80 | W1 に実体を 2 件、W2 に実体を 3 件保管済み | `removeByWorkspace(W1, 10)` を呼ぶ | 2 を返す。W2 の 3 件は `get` で読める |
| TC-ports-mediaStorage-81 | `MediaRepository` に記録が無い W のメディアの実体を保管済み | `removeByWorkspace(W, 10)` を呼ぶ | 1 を返す。`get` は `null` を返す |
| TC-ports-mediaStorage-82 | W に実体を保管し、`remove` 済み | `removeByWorkspace(W, 10)` を呼ぶ | 0 を返す |
| TC-ports-mediaStorage-83 | W に、実体を 1 件保管済み。別の `ref` に試みが 2 件ある | `removeByWorkspace(W, 10)` を呼び、二つの試みを `commit` する | 3 を返す。どちらの `commit` も `ConflictError`（理由 `ATTEMPT_NOT_FOUND`）を投げる。試みの `ref` の `get` は `null` を返す |
| TC-ports-mediaStorage-84 | W1 と W2 に、それぞれ試みがある | `removeByWorkspace(W1, 10)` を呼び、W2 の試みを `commit` する | `commit` は成功する |
| TC-ports-mediaStorage-85 | W に実体を 1 件保管済み | `limit` に、範囲外の値（1 より小さい値、`BoundedDeletion.maxLimit` を超える値）を渡して `removeByWorkspace` を呼ぶ | どちらも `SystemError`（理由 `INVALID_QUERY`）を投げる。`get` は実体を返す |
| TC-ports-mediaStorage-86 | W に実体を保管し、`removeByWorkspace(W, 10)` を完了済み | 同じ `ref` に保管する | 成功する |

## 並行性

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaStorage-87 | 実体が無い | 異なる `ref` への保管（`put` と `commit`）を、並行して 10 件行う | すべて成功する。それぞれの `get` は、その `ref` に渡した内容を返す |
| TC-ports-mediaStorage-88 | 実体が無い | 同じ `ref` に、内容 a の `put` と内容 b の `put` を並行して呼ぶ | どちらも成功し、別の試みを返す |
| TC-ports-mediaStorage-89 | O(W, M) に内容 a の試みと内容 b の試みがある | 二つの試みの `commit` を並行して呼び、両方の完了の後に `get` する | 一方の `commit` だけが成功し、もう一方は `ConflictError`（理由 `OBJECT_EXISTS`）を投げる。`get` は、成功した側の内容を返す |
| TC-ports-mediaStorage-90 | O(W, M) に内容 a の試みがある | 試みの `commit` と、`remove(O(W, M))` または `removeAll(W, M)` を並行して呼び、両方の完了の後に `get` する | `get` は `null` を返す。`commit` は、成功するか、`ConflictError`（理由 `ATTEMPT_NOT_FOUND`）を投げるかのどちらか |
| TC-ports-mediaStorage-92 | O(W, M) に内容 a の試みがある。ほかに W の実体も試みも無い | 試みの `commit` と `removeByWorkspace(W, 10)` を並行して呼び、両方の完了の後に `get` と、もう一度 `removeByWorkspace(W, 10)` を呼ぶ | `get` は `null` を返す。2 回目の `removeByWorkspace` は 0 を返す。`commit` は、成功するか、`ConflictError`（理由 `ATTEMPT_NOT_FOUND`）を投げるかのどちらか |
| TC-ports-mediaStorage-93 | D(W, M, k, g1) に保管済み。D(W, M, k, g2) に内容 b の試みがある | `remove(D(W, M, k, g1))` と、D(W, M, k, g2) の試みの `commit` を並行して呼ぶ | どちらも成功する。`get(D(W, M, k, g1))` は `null`、`get(D(W, M, k, g2))` は内容 b を返す |

## UnitOfWork との関係

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaStorage-94 | 実体が無い | `UnitOfWorkProvider.run` の `fn` の中で `put` と `commit` を呼んだ後、`fn` が例外を投げる | `get` は、保管した実体を返す |
| TC-ports-mediaStorage-95 | 実体を保管済み | `UnitOfWorkProvider.run` の `fn` の中で `remove` を呼んだ後、`fn` が例外を投げる | `get` は `null` を返す |
| TC-ports-mediaStorage-96 | 実体が無い | `UnitOfWorkProvider.run` の外で `put`、`commit`、`discard`、`get`、`remove`、`removeAll`、`removeByWorkspace` を呼ぶ | どれも成功する |
