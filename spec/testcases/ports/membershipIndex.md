# MembershipIndex

「上限」は、その節のメソッドの契約が定める、一度に渡せる件数・個数、または `limit` の上限を指す。値は、ポートを定めるドメインのファイルが持つ。

`MembershipIndex` の契約（[../../domains/workspace.md](../../domains/workspace.md)）を、すべてのアダプターが通すケースで定める。

- 所属の索引は、設置の区画にある。書き込み（`beginJoining`、`confirm`、`remove`、`markLastUsed`、`removeWorkspace`）は `UnitOfWorkProvider.run` の UnitOfWork の中で呼ぶ。`WriteScope` は `{ kind: "installation" }`。「確定する」は、`fn` が正常に終わり `run` が値を返すことを指す。「失敗する」は、`run` がそのエラーを投げることを指す
- 読み取りは、断りが無ければ UnitOfWork の外で、`PartitionReader.installation()` の `membershipIndex` で呼ぶ
- 「並行した二つの UnitOfWork」は、並行に実行する二つの `run` を指す
- A、B、C は、互いに違うアカウント。`Account.create` で作り、断りが無ければ、同じ設置の区画の `accountRepository.insert` で確定済みとする。ID は `IdGenerator.generate` の値を使う
- W、V は、互いに違うワークスペースの ID。`IdGenerator.generate` の値から作る。名簿とワークスペースの区画に何も確定していなくてよい
- `remove([(A, W)])` は、断りが無ければ、直前に `find((A, W))` で読んだ項目を `remove` に渡すことを指す
- 「(A, W)」は、A の ID と W の組を指す。「(A, W) が `joining`」は、`beginJoining([(A, W)], t)` を確定した状態、「(A, W) が `joined`」は、`confirm([(A, W)])` を確定した状態を指す。t、u は、互いに違う固定の日時
- 同じ区画のほかの書き込みには、`accountRepository` を使う
- 「世代」は、項目の `generation` を指す。ケースは、二つの項目の世代が同じか違うかだけを確かめる
- 各ケースは、索引に項目が無い状態から始める

## beginJoining

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-membershipIndex-01 | なし | `beginJoining([(A, W)], t)` を呼び、確定する | `find((A, W))` が、`joining` で `since` が t の項目を返す |
| TC-ports-membershipIndex-02 | なし | `beginJoining([(A, W)], t)` の後、`fn` が例外を投げる | `find((A, W))` が `null` を返す |
| TC-ports-membershipIndex-03 | (A, W) が `joining`（`since` は t）。`find((A, W))` で項目を読んでいる | `beginJoining([(A, W)], u)` を呼び、確定する | `find((A, W))` が、`joining` で `since` が u の、読んだ項目と世代が違う項目を返す。`findByAccount(A)` が一件を返す |
| TC-ports-membershipIndex-04 | (A, W) が `joined`。`find((A, W))` で項目を読んでいる | `beginJoining([(A, W)], u)` を呼び、確定する | `run` が確定する。`find((A, W))` が、`joining` で `since` が u の、読んだ項目と世代が違う項目を返す。`findByAccount(A)` が一件を返す |
| TC-ports-membershipIndex-05 | (A, W) が `joined` で、`markLastUsed((A, W))` を確定している | `beginJoining([(A, W)], u)` を確定した後、`confirm([(A, W)])` を確定する | `find((A, W))` が `joined` の項目を返す。`findLastUsed(A)` は W のまま |
| TC-ports-membershipIndex-06 | (A, W) が `joining`。`find((A, W))` で項目を読み、`remove([(A, W)])` を確定している | `beginJoining([(A, W)], u)` を呼び、確定する | `find((A, W))` が、外した項目と世代が違う項目を返す |
| TC-ports-membershipIndex-07 | なし | `beginJoining([(A, W), (A, V), (B, W)], t)` を一回呼び、確定する | 三つの組の `find` が、どれも `joining` で `since` が t の項目を返す。三つの項目の世代は、互いに違う |
| TC-ports-membershipIndex-08 | なし | `beginJoining([(A, W), (A, W)], t)` を呼び、確定する | `run` が確定する。`findByAccount(A)` が一件を返す |
| TC-ports-membershipIndex-09 | なし | 空の列で `beginJoining` を呼び、確定する | `run` が確定する。項目は無いまま |
| TC-ports-membershipIndex-10 | なし | 上限の数の組を一回の `beginJoining` に渡し、確定する | `run` が確定する。渡した組の `find` が項目を返す |
| TC-ports-membershipIndex-11 | なし | 上限より 1 組多い組を一回の `beginJoining` に渡す | `run` が `SystemError`（理由 `INVALID_QUERY`）で失敗する。どの項目も置かれない |
| TC-ports-membershipIndex-12 | なし | 一つの UnitOfWork で、`accountRepository.insert` でアカウント D を保存し、`beginJoining([(D, W)], t)` を呼び、確定する | `run` が確定する。`find((D, W))` が `joining` の項目を返す |
| TC-ports-membershipIndex-13 | A の `findById` で読んだ `expectedVersion` を持つ | `beginJoining([(A, W)], t)` を確定した後、読んだ `expectedVersion` で A を `accountRepository.save` し、確定する | `save` の `run` が確定する |

## confirm

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-membershipIndex-14 | (A, W) が `joining`。`find((A, W))` で項目を読んでいる | `confirm([(A, W)])` を呼び、確定する | `find((A, W))` が、`joined` の、読んだ項目と世代が同じ項目を返す |
| TC-ports-membershipIndex-15 | (B, W) が `joining`。(A, W) の項目が無い | `confirm([(A, W)])` を呼び、確定する | `find((A, W))` が、`joined` の項目を返す。その世代は、(B, W) の項目の世代と違う |
| TC-ports-membershipIndex-16 | (A, W) が `joined` で、`markLastUsed((A, W))` を確定している。`find((A, W))` で項目を読んでいる | `confirm([(A, W)])` を呼び、確定する | `find((A, W))` が、`joined` の、読んだ項目と世代が同じ項目を返す。`findLastUsed(A)` が W を返す |
| TC-ports-membershipIndex-18 | (A, W) が `joining` | `confirm([(A, W)])` の後、`fn` が例外を投げる | `find((A, W))` が `joining` の項目を返す |
| TC-ports-membershipIndex-19 | (A, W) と (B, W) が `joining` | `confirm([(A, W), (B, W)])` を一回呼び、確定する | 二つの組の `find` が、どちらも `joined` の項目を返す |
| TC-ports-membershipIndex-20 | (A, W) が `joining` | `confirm([(A, W), (A, W)])` を呼び、確定する | `run` が確定する。`find((A, W))` が `joined` の項目を返す。`findByAccount(A)` が一件を返す |
| TC-ports-membershipIndex-21 | なし | 空の列で `confirm` を呼び、確定する | `run` が確定する。項目は無いまま |
| TC-ports-membershipIndex-22 | (A, W) が `joining` | 上限の数の組を一回の `confirm` に渡し、確定する | `run` が確定する。`find((A, W))` が `joined` の項目を返す |
| TC-ports-membershipIndex-23 | なし | 上限より 1 組多い組を一回の `confirm` に渡す | `run` が `SystemError`（理由 `INVALID_QUERY`）で失敗する |

## remove

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-membershipIndex-24 | (A, W) が `joined`。(A, V) が `joined` | `remove([(A, W)])` を呼び、確定する | `find((A, W))` が `null` を返す。`find((A, V))` が `joined` の項目を返す |
| TC-ports-membershipIndex-26 | (A, W) が `joining`。`find((A, W))` で項目を読み、`remove([(A, W)])` を確定している | 読んだ項目を、もう一度 `remove` に渡し、確定する | `run` が確定する。項目は無いまま |
| TC-ports-membershipIndex-27 | (A, W) が `joining`（`since` は t）。`find((A, W))` で項目を読んだ後に、`beginJoining([(A, W)], u)` を確定している | 読んだ項目を `remove` に渡し、確定する | `run` が確定する。`find((A, W))` が、`joining` で `since` が u の項目を返す |
| TC-ports-membershipIndex-29 | (A, W) が `joined`。`find((A, W))` で項目を読んだ後に、`beginJoining([(A, W)], u)` と `confirm([(A, W)])` を確定している | 読んだ `joined` の項目を `remove` に渡し、確定する | `run` が確定する。`find((A, W))` が `joined` の項目を返す |
| TC-ports-membershipIndex-30 | (A, W) が `joining`。`find((A, W))` で項目を読んだ後に、`confirm([(A, W)])` を確定している | 読んだ `joining` の項目を `remove` に渡し、確定する | `find((A, W))` が `null` を返す |
| TC-ports-membershipIndex-31 | (A, W) が `joined`。(B, W) が `joining`。二つの項目を `find` で読んだ後に、`beginJoining([(B, W)], u)` を確定している | 読んだ二つの項目を、一回の `remove` に渡し、確定する | `find((A, W))` が `null` を、`find((B, W))` が `joining` の項目を返す |
| TC-ports-membershipIndex-32 | (A, W) が `joined` | `remove([(A, W)])` の後、`fn` が例外を投げる | `find((A, W))` が `joined` の項目を返す |
| TC-ports-membershipIndex-33 | (A, W) と (B, W) が `joined` | `remove([(A, W), (B, W)])` を一回呼び、確定する | 二つの組の `find` が、どちらも `null` を返す |
| TC-ports-membershipIndex-34 | (A, W) が `joined`。(A, V) が `joined` | `find((A, W))` で読んだ項目を二つ重ねて、一回の `remove` に渡し、確定する | `run` が確定する。`find((A, W))` が `null` を、`find((A, V))` が `joined` の項目を返す |
| TC-ports-membershipIndex-35 | (A, W) が `joined` | `remove([(A, W)])` を確定した後、`findById(A.id)`（`accountRepository`）を呼ぶ | A を返す |
| TC-ports-membershipIndex-36 | なし | 空の列で `remove` を呼び、確定する | `run` が確定する。項目は無いまま |
| TC-ports-membershipIndex-37 | (A, W) が `joined` | 上限の数の項目（(A, W) の読んだ項目を含む）を一回の `remove` に渡し、確定する | `run` が確定する。`find((A, W))` が `null` を返す |
| TC-ports-membershipIndex-38 | なし | 上限より 1 件多い項目を一回の `remove` に渡す | `run` が `SystemError`（理由 `INVALID_QUERY`）で失敗する |

## markLastUsed と findLastUsed

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-membershipIndex-39 | (A, W) が `joined`、(A, V) が `joined` | `findLastUsed(A)` を呼ぶ | `null` を返す |
| TC-ports-membershipIndex-40 | (A, W) が `joined`、(A, V) が `joined` | `markLastUsed((A, W))` を呼び、確定する | `findLastUsed(A)` が W を返す |
| TC-ports-membershipIndex-41 | (A, W) と (A, V) が `joined` で、`markLastUsed((A, V))` を確定している | `markLastUsed((A, W))` を呼び、確定する | `findLastUsed(A)` が W を返す |
| TC-ports-membershipIndex-42 | (A, W) が `joined`、(B, W) が `joined` で、`markLastUsed((B, W))` を確定している | `markLastUsed((A, W))` を呼び、確定する | `findLastUsed(A)` と `findLastUsed(B)` が、どちらも W を返す |
| TC-ports-membershipIndex-43 | (A, W) が `joined` で、`markLastUsed((A, W))` を確定している | もう一度 `markLastUsed((A, W))` を呼び、確定する | `run` が確定する。`findLastUsed(A)` は W のまま |
| TC-ports-membershipIndex-44 | 項目が無い | `markLastUsed((A, W))` を呼び、確定する | `run` が確定する。`find((A, W))` は `null` のまま。`findLastUsed(A)` が W を返す |
| TC-ports-membershipIndex-46 | (A, W) が `joined` | `markLastUsed((A, W))` の後、`fn` が例外を投げる | `findLastUsed(A)` が `null` を返す |
| TC-ports-membershipIndex-47 | (A, W) と (A, V) が `joined` で、`markLastUsed((A, W))` を確定している | `remove([(A, W)])` を確定する | `findLastUsed(A)` は W のまま |
| TC-ports-membershipIndex-48 | (A, W) が `joined` で、`markLastUsed((A, W))` を確定している | `removeWorkspace(W)` を確定する | `findLastUsed(A)` は W のまま |
| TC-ports-membershipIndex-49 | (A, W) と (A, V) が `joined` | 並行した二つの UnitOfWork が、一方は `markLastUsed((A, W))`、他方は `markLastUsed((A, V))` を呼ぶ | 両方の `run` が確定する。`findLastUsed(A)` は、W または V の一つを返す |

## removeWorkspace

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-membershipIndex-50 | (A, W) が `joined`。(B, W) が `joining`、(B, V) が `joined`。(C, V) が `joined` | `removeWorkspace(W)` を呼び、確定する | `find((A, W))` と `find((B, W))` が `null` を返す。`find((B, V))` と `find((C, V))` が `joined` の項目を返す |
| TC-ports-membershipIndex-51 | W の項目が無い | `removeWorkspace(W)` を呼び、確定する | `run` が確定する。何も変わらない |
| TC-ports-membershipIndex-52 | (A, W) が `joined` | `removeWorkspace(W)` の後、`fn` が例外を投げる | `find((A, W))` が `joined` の項目を返す |
| TC-ports-membershipIndex-53 | 1,200 のアカウントに、W の `joined` の項目がある | `removeWorkspace(W)` を一回呼び、確定する | `run` が確定する。`findPageByWorkspace(W, { after: null, limit: 100 })` が空の列を返す |

## find

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-membershipIndex-55 | (A, W) が `joined` | `find((A, W))` を二度呼ぶ | どちらも、`accountId` が A の ID、`workspaceId` が W、`status` が `joined` の項目を返す。二つの項目の世代は同じ |
| TC-ports-membershipIndex-56 | (A, W) が `joined` | `find((A, V))` と `find((B, W))` を呼ぶ | どちらも `null` を返す |

## findByAccount

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-membershipIndex-58 | (A, W) が `joined`、(A, V) が `joining`、(A, U) が `joined`（U は、W、V と違うワークスペースの ID）。(B, W) が `joined` | `findByAccount(A)` を呼ぶ | A の三件を、`workspaceId` の昇順で、状態によらず返す。B の項目を含めない |
| TC-ports-membershipIndex-59 | A の項目が無い | `findByAccount(A)` を呼ぶ | 空の列を返す |
| TC-ports-membershipIndex-60 | A に、150 のワークスペースの `joined` の項目がある | `findByAccount(A)` を呼ぶ | 150 件を、`workspaceId` の昇順で返す |

## findPageByWorkspace

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-membershipIndex-61 | (A, W) が `joined`、(B, W) が `joining`。(C, V) が `joined` | `findPageByWorkspace(W, { after: null, limit: 100 })` を呼ぶ | A と B の項目を、`accountId` の昇順で、状態によらず返す。`next` は `null`。C の項目を含めない |
| TC-ports-membershipIndex-62 | W の項目が無い | `findPageByWorkspace(W, { after: null, limit: 100 })` を呼ぶ | 空の列と、`next` に `null` を返す |
| TC-ports-membershipIndex-63 | 三つのアカウントに W の項目がある | `findPageByWorkspace(W, { after: null, limit: 2 })` を呼び、返った `next` を `after` に渡してもう一度呼ぶ | 一度目は、`accountId` の昇順で先頭の二件と、`next` に二件目の `accountId` を返す。二度目は、残りの一件と、`next` に `null` を返す |
| TC-ports-membershipIndex-64 | 二つのアカウントに W の項目がある | `findPageByWorkspace(W, { after: null, limit: 2 })` を呼ぶ | 二件と、`next` に `null` を返す |
| TC-ports-membershipIndex-65 | 三つのアカウントに W の項目がある。`limit: 2` で一度目を読んだ | 一度目に返った先頭の項目を `remove` で確定してから、`next` を `after` に渡して呼ぶ | 残りの一件を返す |
| TC-ports-membershipIndex-66 | W の項目がある | `limit` に 0、または上限より 1 大きい値を渡して呼ぶ | `SystemError`（理由 `INVALID_QUERY`） |

## scan

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-membershipIndex-67 | (A, W) が `joined`、(A, V) が `joining`。(B, W) が `joined` | `scan({ after: null, limit: 100 })` を呼ぶ | 三件を、`accountId` の昇順、同じアカウントは `workspaceId` の昇順で、状態とワークスペースによらず返す。`next` は `null` |
| TC-ports-membershipIndex-68 | 項目が無い | `scan({ after: null, limit: 100 })` を呼ぶ | 空の列と、`next` に `null` を返す |
| TC-ports-membershipIndex-69 | 索引に三つの項目がある | `scan({ after: null, limit: 2 })` を呼び、返った `next` を `after` に渡してもう一度呼ぶ | 一度目は、順の先頭の二件と、`next` に二件目の組を返す。二度目は、残りの一件と、`next` に `null` を返す |
| TC-ports-membershipIndex-70 | 索引に三つの項目がある。`limit: 2` で一度目を読んだ | 一度目に返った先頭の項目を `remove` で確定してから、`next` を `after` に渡して呼ぶ | 残りの一件を返す |
| TC-ports-membershipIndex-71 | 項目がある | `limit` に 0、または上限より 1 大きい値を渡して呼ぶ | `SystemError`（理由 `INVALID_QUERY`） |

## 参照整合性

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-membershipIndex-72 | D は、`accountRepository` に確定していないアカウントの ID | `beginJoining([(D, W)], t)` を呼ぶ | `run` が `ConflictError`（理由 `MEMBERSHIP_ACCOUNT_MISSING`）で失敗する。`find((D, W))` が `null` を返す |
| TC-ports-membershipIndex-73 | D は、確定していないアカウントの ID | `confirm([(D, W)])` を呼ぶ | `run` が `ConflictError`（理由 `MEMBERSHIP_ACCOUNT_MISSING`）で失敗する。`find((D, W))` が `null` を返す |
| TC-ports-membershipIndex-74 | D は、確定していないアカウントの ID | `beginJoining([(A, W), (D, W)], t)` を呼ぶ | `run` が `ConflictError`（理由 `MEMBERSHIP_ACCOUNT_MISSING`）で失敗する。`find((A, W))` が `null` を返す |
| TC-ports-membershipIndex-75 | W は、名簿に項目が無く、区画に何も確定していない | `beginJoining([(A, W)], t)` を呼び、確定する | `run` が確定する |

## 並行性

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-membershipIndex-76 | 項目が無い | 並行した二つの UnitOfWork が、どちらも `beginJoining([(A, W)], t)` を呼ぶ | 両方の `run` が確定する。`findByAccount(A)` が一件を返す |
| TC-ports-membershipIndex-77 | (A, W) が `joining`。`find((A, W))` で項目を読んでいる | 並行した二つの UnitOfWork が、一方は `beginJoining([(A, W)], u)`、他方は、読んだ項目の `remove` を呼ぶ | 両方の `run` が確定する。確定の順によらず、`find((A, W))` は、`joining` で `since` が u の、読んだ項目と世代が違う項目を返す |

## 可視性

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-membershipIndex-78 | なし | `beginJoining([(A, W)], t)` を確定した直後に、UnitOfWork の外で `find((A, W))`、`findByAccount(A)`、`findPageByWorkspace(W, { after: null, limit: 100 })`、`scan({ after: null, limit: 100 })` を呼ぶ | どの読み取りも、確定した項目を反映する |
| TC-ports-membershipIndex-79 | (A, W) が `joined` | `removeWorkspace(W)` を確定した直後に、同じ四つの読み取りを呼ぶ | `find` は `null`、三つの一覧は空の列 |
