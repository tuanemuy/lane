# SubmissionRepository

`spec/domains/form.md` の `SubmissionRepository` の契約と、`spec/domains/common.md` のリポジトリに共通する規則を、すべてのアダプターが通すケースで定める。通知の読み書きのケースは [submissionNotificationRepository.md](submissionNotificationRepository.md) が定める。

- 「UoW」は `UnitOfWorkProvider.run` の `fn`。「確定」は `fn` の正常終了を指す
- UoW の `WriteScope` は、どのケースも、対象のワークスペースの `{ kind: "workspace", workspaceId, condition: { kind: "none" } }`。区画と条件は、`unitOfWorkProvider.md` が確かめる
- リポジトリは、ワークスペースの区画に結び付く。断りが無い限り、UoW は W1 の UoW、読み取りは `PartitionReader.workspace(W1)` のリポジトリで、回答は W1 の回答。W2 の回答の書き込みは W2 の UoW で、読み取りは W2 の区画のリポジトリで行う
- 前提条件の書き込みは、別の UoW で確定済み。期待結果の読み取りは、UoW の外で行う
- W1 と W2 は別の、確定済みのワークスペース。F1 と F2 は別の `FormId`。S1、S2、S3、S4 は別の `SubmissionId`
- 「`SystemError`（`INVALID_QUERY`）になる」は、呼び出しがそのエラーを返すことを指す。書き込みでは、`run` がそのエラーを投げ、その UoW の書き込みが 1 件も反映されない
- 「回答」は、`Submission.receive` が返した `submission` を `insert` し、同じ UoW で、その `notification` を `SubmissionNotificationRepository.insert` で保存したもの
- 通知の状態は、`SubmissionNotification` の振る舞いの結果を `SubmissionNotificationRepository.save` で保存して作る

回答の呼び名と、その通知の状態と作り方は次のとおり。

- 送信待ちの回答: `attempt` は `initial`、`state` は `waiting`。受け付けたまま
- 送信中の回答: `attempt` は `initial`、`state` は `sending`。`start` の `send` の結果
- 送信成功の回答: `succeeded`。全宛先が受理の `complete` の結果
- 送信失敗の回答: `failed`（未対処）。失敗を含む `complete` の結果
- 対処済みの回答: `failed`（対処済み）。`markHandled` が返した `changed` の `value`
- 再試行の送信待ちの回答: `attempt` は `retry` の `round` 1、`state` は `waiting`。送信失敗の回答の通知への `requestRetry` の結果
- 再試行の送信中の回答: `attempt` は `retry` の `round` 1、`state` は `sending`。再試行の送信待ちの回答の通知への `start` の `send` の結果

## insert

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-submissionRepository-01 | 回答が無い | UoW で、回答を `insert` して確定する | `findById` が、渡した回答と `id`、`formId`、`receivedAt`、`form`、`template`、`notifiedTo`、`retention` が等しい回答を返す |
| TC-ports-submissionRepository-02 | 回答が無い | `FormFieldValue` のすべての種類の値と、問い合わせ対象を持つ回答を `insert` して確定する | `findById` が返す `content` を `Confidential.reveal` した値は、渡した `values` と `inquiryTargets` に等しい |
| TC-ports-submissionRepository-03 | 回答が無い | 項目、選択肢、表示名称、順序を持つ `form` と、参照と `ifAnswered` を含む `template` の回答を `insert` して確定する | `findById` が返す `form` と `template` は、渡した値に等しい。項目と選択肢の順序を保つ |
| TC-ports-submissionRepository-04 | 回答が無い | `notifiedTo` が 3 件の回答と、1 件の回答を `insert` して確定する | `findById` が返す `notifiedTo` は、渡した順序と件数を保つ |
| TC-ports-submissionRepository-05 | W1 に S1 がある | UoW で、W1 に `id` が S1 の回答を `insert` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。`findById(S1)` は元の回答を返す |
| TC-ports-submissionRepository-06 | W1 に S1 がある | 一つの UoW で、S2 の `insert` と、`id` が S1 の回答の `insert` を行う | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。`findById(S2)` は `null` |
| TC-ports-submissionRepository-07 | 回答が無い | 二つの UoW が並行して、`id` が S1 の回答を `insert` する | 一方だけが確定する。他方の `run` は `ConflictError`（理由 `ALREADY_EXISTS`）を投げる |
| TC-ports-submissionRepository-08 | W1 に回答がある | UoW で、入力内容が同じで `id` が別の回答を `insert` して確定する | 成功する。二件が別の回答として残る |
| TC-ports-submissionRepository-09 | S1 を `deleteMany` で削除して確定した | UoW で、`id` が S1 で入力内容が別の回答を `insert` して確定する | 成功する。`findById(S1)` は新しい入力内容を返す |

## findById

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-submissionRepository-10 | 区画が無いワークスペース（存在しない ID） | その区画のリポジトリで、`findById(S1)` と、すべての条件が `null` の `filter` の `findPage` を呼ぶ | `findById` は `null`。`findPage` は空の列と `count` 0 |
| TC-ports-submissionRepository-11 | 回答が無い | `findById(S1)` を呼ぶ | `null` |
| TC-ports-submissionRepository-12 | W1 に S1 がある | `findById(S1)` を呼ぶ | S1 を返す |
| TC-ports-submissionRepository-13 | W1 に、期限を過ぎた S1 と、過ぎていない S3、W2 に、期限を過ぎた S2 がある | W1 の区画のリポジトリで `findById(S2)`、すべての条件が `null` の `filter` の `findPage`、`countExpired(now)` を、W2 の区画のリポジトリで `findById(S2)` を呼ぶ | W1 の区画の `findById` は `null`、`findPage` は S1 と S3 と `count` 2、`countExpired` は 1。W2 の区画は S2 を返す |
| TC-ports-submissionRepository-14 | W1 に S1 がある | UoW の外で `findById(S1)` を呼ぶ | UoW を開かずに回答を返す |

## deleteMany

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-submissionRepository-15 | W1 に S1、S2 がある | UoW で `deleteMany([S1])` を呼び、確定する | `findById(S1)` は `null`。`findById(S2)` は S2 を返す |
| TC-ports-submissionRepository-16 | W1 に `SubmissionBatch.maxTargets` 件ある | UoW で、そのすべての ID を渡して `deleteMany` を呼び、確定する | すべてが `findById` で `null` になる |
| TC-ports-submissionRepository-17 | W1 に送信待ち、送信中、送信成功、送信失敗、再試行の送信待ちの回答が 1 件ずつある | UoW で、5 件の ID を渡して `deleteMany` を呼び、確定する | 5 件すべてを、その通知とともに削除する |
| TC-ports-submissionRepository-18 | W1 に S1、S2 がある。S3 は無い。S4 は、一度 `deleteMany` で削除して確定した | UoW で `deleteMany([S1, S2, S3, S4])` を呼び、確定する。続けて、回答の無い区画で `deleteMany([S1])` を呼び、確定する | どちらも例外を投げない。`findById(S1)` と `findById(S2)` は `null`。区画に無い ID は、削除済みとして扱い、何も書かない |
| TC-ports-submissionRepository-20 | W1 に S1、W2 に S2 がある | W1 の UoW で `deleteMany([S1, S2])` を呼び、確定する | S1 は `null`。W2 の S2 は残る。ほかの区画の回答は、区画に無い回答として扱う |
| TC-ports-submissionRepository-22 | W1 に S1、S2 がある | UoW で `deleteMany([S1, S1])` を呼び、確定する | S1 を削除し、S2 は残る |
| TC-ports-submissionRepository-23 | W1 に S1、S2、S3 がある | 二つの UoW が並行して、`deleteMany([S1, S2])` と `deleteMany([S2, S3])` を行う | どちらの `run` も確定し、`ConflictError` にならない。S1、S2、S3 はどれも `null` |
| TC-ports-submissionRepository-24 | W1 に、保持期限を過ぎた S1 と、過ぎていない S2 がある | 二つの UoW が並行して、`deleteMany([S1, S2])` と `deleteExpired(now, BoundedDeletion.maxLimit)` を行う | どちらの `run` も確定し、`ConflictError` にならない。S1 と S2 はどちらも `null` |
| TC-ports-submissionRepository-25 | W1 に S1、S2 がある | 一つの UoW で、別の回答の `insert` と `deleteMany([S1, S2])` を行い、`fn` が例外を投げる | `insert` した回答は `null`。S1 と S2 はどちらも残る。一部だけを削除した結果は無い |
| TC-ports-submissionRepository-26 | W1 に、`SubmissionBatch.maxTargets` を 1 件超える件数の回答がある | UoW で `deleteMany([])` を呼ぶ。別の UoW で、重複の無い、そのすべての ID を渡して `deleteMany` を呼ぶ | どちらも `SystemError`（`INVALID_QUERY`）になる。どの回答も残る。上限を超える入力を切り詰めない |
| TC-ports-submissionRepository-28 | W1 に `SubmissionBatch.maxTargets` 件ある | UoW で、そのすべての ID に、そのうち 1 個の重複を加えて `deleteMany` を呼び、確定する | すべてを削除する。上限は、重複を除いた数に適用する |

## findPage

### 絞り込み

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-submissionRepository-29 | W1 に回答が無い | すべての条件が `null` の `filter` で `findPage(filter, { page: 1, limit: 20 })` を呼ぶ | 空の列と `count` 0 |
| TC-ports-submissionRepository-31 | W1 に、F1 の回答が 2 件、F2 の回答が 1 件ある | すべての条件が `null` の `filter` で `findPage` を呼ぶ | 3 件すべてと `count` 3 |
| TC-ports-submissionRepository-34 | W1 に、F1 の回答が 2 件、F2 の回答が 1 件ある | `formId` が F1 の `filter` で `findPage` を呼ぶ | F1 の 2 件と `count` 2 |
| TC-ports-submissionRepository-35 | W1 に F1 の回答がある | `formId` が、回答を持たない `FormId` の `filter` で `findPage` を呼ぶ | 空の列と `count` 0 |
| TC-ports-submissionRepository-36 | W1 に、F1 の回答が 1 件、F2 の回答が 1 件ある。同じ区画の `FormSettingsRepository` で、F1 の設定を `removed` にして確定した | すべての条件が `null` の `filter` と、`formId` が F1 の `filter` で `findPage` を呼ぶ | 条件なしは 2 件、F1 は F1 の 1 件を返す。削除済みフォームの回答を含む |
| TC-ports-submissionRepository-37 | W1 に、`receivedAt` が T1 より前、T1 と T2 の間、T2 より後の回答が 1 件ずつある（T1 < T2） | `receivedFrom` が T1 の `filter` で `findPage` を呼ぶ | T1 と T2 の間の回答と、T2 より後の回答の 2 件 |
| TC-ports-submissionRepository-38 | 同上 | `receivedTo` が T2 の `filter` で `findPage` を呼ぶ | T1 より前の回答と、T1 と T2 の間の回答の 2 件 |
| TC-ports-submissionRepository-39 | 同上 | `receivedFrom` が T1、`receivedTo` が T2 の `filter` で `findPage` を呼ぶ | T1 と T2 の間の 1 件 |
| TC-ports-submissionRepository-40 | W1 に、送信待ち、送信中、送信成功、送信失敗、対処済み、再試行の送信待ち、再試行の送信中の回答が 1 件ずつある | `notification` が `{ result: "unsent" }` の `filter` で `findPage` を呼ぶ | 送信待ち、送信中、再試行の送信待ち、再試行の送信中の 4 件 |
| TC-ports-submissionRepository-41 | 同上 | `notification` が `{ result: "succeeded" }` の `filter` で `findPage` を呼ぶ | 送信成功の 1 件 |
| TC-ports-submissionRepository-42 | 同上 | `notification` が `{ result: "failed", handling: null }` の `filter` で `findPage` を呼ぶ | 送信失敗と対処済みの 2 件。再試行の送信待ちと再試行の送信中を含まない |
| TC-ports-submissionRepository-43 | 同上 | `notification` が `{ result: "failed", handling: "unhandled" }` の `filter` と、`{ result: "failed", handling: "handled" }` の `filter` で `findPage` を呼ぶ | `unhandled` は送信失敗の 1 件。`handled` は対処済みの 1 件。`failed` でない通知の回答は、どちらにも現れない |
| TC-ports-submissionRepository-44 | W1 の対処済みの回答の通知に、`requestRetry` の結果を保存した | `notification` が `{ result: "failed", handling: "handled" }` の `filter` と、`{ result: "unsent" }` の `filter` で `findPage` を呼ぶ | `handled` は返さない。`unsent` はその回答を返す |
| TC-ports-submissionRepository-45 | W1 に、F1 の送信成功と送信失敗、F2 の送信失敗の回答がある | `formId` が F1、`notification` が `{ result: "failed", handling: null }` の `filter` で `findPage` を呼ぶ | F1 の送信失敗の 1 件 |

### 検索

A は F1 の `text` の項目のキー、B は F1 の別の `text` の項目のキー、N は F1 の `number` の項目のキー。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-submissionRepository-46 | W1 に、A の値が検索の文字列を含む F1 の回答と、含まない F1 の回答がある | `search` の `listFields` が `{ F1: [A] }` の `filter` で `findPage` を呼ぶ | 含む回答だけ |
| TC-ports-submissionRepository-47 | W1 に、A の値の途中に検索の文字列を含む F1 の回答がある | `listFields` が `{ F1: [A] }` の `filter` で `findPage` を呼ぶ | その回答を返す（部分一致） |
| TC-ports-submissionRepository-48 | W1 に、A の値が検索の文字列と、大文字・小文字だけ違う F1 の回答と、全角・半角だけ違う F1 の回答がある | `listFields` が `{ F1: [A] }` の `filter` で `findPage` を呼ぶ | 2 件とも返す |
| TC-ports-submissionRepository-50 | W1 に、B の値だけが検索の文字列を含む F1 の回答がある | `listFields` が `{ F1: [A] }` の `filter` で `findPage` を呼ぶ | 空の列 |
| TC-ports-submissionRepository-51 | W1 に、B の値だけが検索の文字列を含む F1 の回答がある | `listFields` が `{ F1: [A, B] }` の `filter` で `findPage` を呼ぶ | その回答を返す |
| TC-ports-submissionRepository-52 | W1 に、A の値が検索の文字列を含む F1 の回答がある | `listFields` が、受付時のフォームに無いキーだけを持つ `filter` で `findPage` を呼ぶ | 空の列 |
| TC-ports-submissionRepository-53 | W1 に、N の値を文字列にすると検索の文字列を含む F1 の回答がある | `listFields` が `{ F1: [N] }` の `filter` で `findPage` を呼ぶ | 空の列。検索は、テキストの項目の値だけを照合する |
| TC-ports-submissionRepository-54 | W1 に、A の値が検索の文字列を含む F1 の回答がある | `listFields` が F1 の対応を持たない `filter` と、`{ F1: [] }` の `filter` で `findPage` を呼ぶ | どちらも空の列と `count` 0 |
| TC-ports-submissionRepository-56 | W1 に、A が未入力の F1 の回答がある | `listFields` が `{ F1: [A] }` の `filter` で `findPage` を呼ぶ | 空の列 |
| TC-ports-submissionRepository-57 | W1 に、F1 の回答が 2 件ある | `search.text` が空文字列の `filter` で `findPage` を呼ぶ | 2 件すべて |
| TC-ports-submissionRepository-58 | W1 に、A の値が検索の文字列を含む F1 の回答が、`receivedAt` が T の前と後に 1 件ずつある | `search` と `receivedFrom`（T）を指定した `filter` で `findPage` を呼ぶ | T の後の 1 件 |
| TC-ports-submissionRepository-59 | W1 の、A の値が検索の文字列を含む回答を `deleteMany` で削除して確定した | 同じ `search` の `filter` で `findPage` を呼ぶ | 空の列と `count` 0 |

### 並び順とページング

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-submissionRepository-60 | W1 に、`receivedAt` が互いに異なる 3 件を、`receivedAt` の昇順に `insert` した | `findPage` を呼ぶ | `receivedAt` の降順で返す |
| TC-ports-submissionRepository-61 | W1 に、`receivedAt` が同じ 2 件と、それより新しい 1 件がある | `findPage` を呼ぶ | 新しい 1 件、続けて同じ日時の 2 件を `id` の昇順で返す |
| TC-ports-submissionRepository-63 | W1 に 3 件ある | `{ page: 1, limit: 1 }` と `{ page: 2, limit: 2 }` で `findPage` を呼ぶ | 前者は並び順の先頭の 1 件、後者は 3 件目の 1 件。どちらも `count` 3 |
| TC-ports-submissionRepository-65 | W1 に 3 件ある | `{ page: 3, limit: 2 }` で `findPage` を呼ぶ | 空の列と `count` 3 |
| TC-ports-submissionRepository-66 | W1 に、F1 の回答が 3 件、F2 の回答が 2 件ある | `formId` が F1 の `filter` と `{ page: 1, limit: 2 }` で `findPage` を呼ぶ | F1 の先頭 2 件と `count` 3 |

## previewRetention

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-submissionRepository-67 | F1 の回答が無い | `previewRetention(F1, policy, now)` を呼ぶ | `total` 0、`oldestReceivedAt` と `newestReceivedAt` は `null`、`changing` 0、`expiring` 0 |
| TC-ports-submissionRepository-68 | F1 に、`receivedAt` が互いに異なる 3 件がある。別に、F2 に 1 件だけがある | `previewRetention(F1, policy, now)` と `previewRetention(F2, policy, now)` を呼ぶ | F1 は `total` 3、`oldestReceivedAt` は最も古い、`newestReceivedAt` は最も新しい `receivedAt`。F2 は `total` 1、どちらの日時もその回答の `receivedAt` |
| TC-ports-submissionRepository-70 | F1 に、`retention` が 30 日の回答が 2 件、90 日の回答が 1 件ある。F2 に、`retention` が 90 日の回答だけがある | `policy` が 90 日の `previewRetention` を、F1 と F2 について呼ぶ | F1 は `changing` 2、F2 は `changing` 0 |
| TC-ports-submissionRepository-72 | F1 に、`policy` では `now` に保持期限を過ぎる回答が 2 件、過ぎない回答が 1 件ある | `previewRetention(F1, policy, now)` を呼ぶ | `expiring` 2 |
| TC-ports-submissionRepository-73 | F1 に、現在の `retention` では期限を過ぎているが、`policy` では過ぎない回答がある | `previewRetention(F1, policy, now)` を呼ぶ | `expiring` 0 |
| TC-ports-submissionRepository-74 | F1 に 2 件、F2 に 3 件ある | `previewRetention(F1, policy, now)` を呼ぶ | `total` 2。F2 の回答を数えない |
| TC-ports-submissionRepository-75 | F1 に、`retention` が 30 日の回答がある | `policy` が 90 日の `previewRetention` を呼んだ後、`findById` を呼ぶ | `retention` は 30 日のまま |

## replaceRetentionByForm

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-submissionRepository-76 | F1 の回答が無い。または、F1 に、`retention` が 90 日の回答だけが 2 件ある | UoW で `replaceRetentionByForm(F1, 90 日, FormSweep.maxLimit)` を呼び、確定する | 0 を返す。どの回答も変わらない |
| TC-ports-submissionRepository-78 | F1 に、`retention` が 30 日の回答が 2 件、90 日の回答が 1 件ある | UoW で `replaceRetentionByForm(F1, 90 日, FormSweep.maxLimit)` を呼び、確定する | 2 を返す。3 件の `retention` は 90 日 |
| TC-ports-submissionRepository-80 | F1 に、`retention` が 30 日の回答が 2 件、F2 に 1 件ある | UoW で `replaceRetentionByForm(F1, 90 日, FormSweep.maxLimit)` を呼び、確定する | F2 の回答の `retention` は 30 日のまま |
| TC-ports-submissionRepository-81 | F1 に、送信待ち、送信中、送信成功、送信失敗の回答が 1 件ずつある（`retention` は 30 日） | UoW で `replaceRetentionByForm(F1, 90 日, FormSweep.maxLimit)` を呼び、確定する | 4 を返す。通知は変わらない |
| TC-ports-submissionRepository-82 | F1 に、`receivedAt` が互いに異なる、`retention` が 30 日の回答が 3 件ある。F2 に、`receivedAt` が同じ、`retention` が 30 日の回答が 3 件ある | UoW で `replaceRetentionByForm(F1, 90 日, 2)` と `replaceRetentionByForm(F2, 90 日, 2)` を呼び、確定する | どちらも 2 を返す。F1 は `receivedAt` が古い 2 件が 90 日になり、最も新しい 1 件は 30 日のまま。F2 は `id` が最も大きい 1 件が 30 日のまま |
| TC-ports-submissionRepository-84 | F1 に、`retention` が 30 日の回答が 5 件ある | 呼び出しごとに別の UoW で、`replaceRetentionByForm(F1, 90 日, 2)` を、返る件数が 2 より少なくなるまで繰り返す | 2、2、1 の順に返す。5 件すべてが 90 日 |
| TC-ports-submissionRepository-85 | F1 に、`retention` が 30 日の回答が 3 件ある | UoW で `replaceRetentionByForm(F1, 90 日, FormSweep.maxLimit)` を呼び、`fn` が例外を投げる | 3 件の `retention` は変わらない |
| TC-ports-submissionRepository-86 | F1 に、30 日の `retention` では `now` に期限を過ぎていない回答がある | UoW で `replaceRetentionByForm(F1, 1 日, FormSweep.maxLimit)` を呼び、確定する。1 日の `retention` では期限を過ぎる | `countExpired(now)` がその回答を数える |
| TC-ports-submissionRepository-87 | F1 に、30 日の `retention` で `now` に期限を過ぎた回答がある | UoW で `replaceRetentionByForm(F1, 365 日, FormSweep.maxLimit)` を呼び、確定する | `countExpired(now)` はその回答を数えない。`deleteExpired(now, BoundedDeletion.maxLimit)` は削除しない |
| TC-ports-submissionRepository-88 | F1 に、`retention` が 30 日の回答がある | UoW で `replaceRetentionByForm(F1, 90 日, 0)` を呼ぶ。別の UoW で、`FormSweep.maxLimit` を超える `limit` で呼ぶ | どちらも `SystemError`（`INVALID_QUERY`）になる。回答の `retention` は 30 日のまま |

### 並行性

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-submissionRepository-90 | F1 に、`retention` が 30 日の回答がある | 二つの UoW が並行して、`replaceRetentionByForm(F1, 60 日, FormSweep.maxLimit)` と `replaceRetentionByForm(F1, 90 日, FormSweep.maxLimit)` を行う | どちらの `run` も確定する。`ConflictError` にならない。回答の `retention` は、後に確定した側の保持方針（60 日か 90 日のどちらか）になる。30 日のまま残らない |
| TC-ports-submissionRepository-91 | F1 に、`retention` が 30 日の回答 S1 がある | 二つの UoW が並行して、`replaceRetentionByForm(F1, 90 日, FormSweep.maxLimit)` と `deleteMany([S1])` を行う | `deleteMany` が確定した後、`findById(S1)` は `null` |
| TC-ports-submissionRepository-92 | F1 に、`retention` が 30 日の送信待ちの回答 S1 がある。S1 の通知を読む | 二つの UoW が並行して、`replaceRetentionByForm(F1, 90 日, FormSweep.maxLimit)` と、S1 の通知の `start` の結果の `SubmissionNotificationRepository.save` を行う | どちらの `run` も確定し、`ConflictError` にならない。S1 の `retention` は 90 日で、通知は `sending` |

## countExpired

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-submissionRepository-93 | 回答が無い。または、期限を過ぎた回答が 2 件、過ぎていない回答が 1 件ある | `countExpired(now)` を呼ぶ | 前者は 0、後者は 2 |
| TC-ports-submissionRepository-96 | 期限を過ぎた、送信待ち、送信中、送信成功、送信失敗の回答が 1 件ずつある | `countExpired(now)` を呼ぶ | 4 |
| TC-ports-submissionRepository-97 | 期限を過ぎた回答を `deleteExpired` で削除して確定した | `countExpired(now)` を呼ぶ | 0 |

## deleteExpired

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-submissionRepository-98 | 回答が無い。または、期限を過ぎていない回答だけがある | UoW で `deleteExpired(now, BoundedDeletion.maxLimit)` を呼び、確定する | 0 を返す。回答は残る |
| TC-ports-submissionRepository-100 | 期限を過ぎた回答が 2 件、過ぎていない回答が 1 件ある | UoW で `deleteExpired(now, BoundedDeletion.maxLimit)` を呼び、確定する | 2 を返す。過ぎていない回答だけが残る |
| TC-ports-submissionRepository-102 | `expiresAt` が互いに異なる、期限を過ぎた回答が 3 件ある | UoW で `deleteExpired(now, 2)` を呼び、確定する。別の同じ前提で、`deleteExpired(now, 1)` を呼び、確定する | 前者は 2 を返し、`expiresAt` が最も遅い 1 件だけが残る。後者は 1 を返し、`expiresAt` が最も早い 1 件だけを削除する |
| TC-ports-submissionRepository-104 | `receivedAt` が新しいほうの回答の `retention` が短く、`expiresAt` は早い、期限を過ぎた回答が 2 件ある | UoW で `deleteExpired(now, 1)` を呼び、確定する | `expiresAt` が早いほうを削除する |
| TC-ports-submissionRepository-105 | `expiresAt` が同じ、期限を過ぎた回答が 3 件ある | UoW で `deleteExpired(now, 2)` を呼び、確定する | 2 を返す。`id` が最も大きい 1 件だけが残る |
| TC-ports-submissionRepository-106 | 期限を過ぎた回答が 5 件ある | 呼び出しごとに別の UoW で、`deleteExpired(now, 2)` を、返る件数が 2 より少なくなるまで繰り返す | 2、2、1 の順に返す。`countExpired(now)` は 0 |
| TC-ports-submissionRepository-107 | W1 に期限を過ぎた回答が 2 件、W2 に期限を過ぎた回答が 1 件と、ほかの回答が 1 件ある | W1 の UoW で `deleteExpired(now, BoundedDeletion.maxLimit)` を呼んで確定し、続けて別の W1 の UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼んで確定する | `deleteExpired` は 2 を返す。W2 の 2 件の回答と、その通知は残る |
| TC-ports-submissionRepository-108 | W1 に期限を過ぎた回答があり、`WorkspaceRepository` で W1 の区画を `closing` にして確定した | UoW で `deleteExpired(now, BoundedDeletion.maxLimit)` を呼び、確定する | ワークスペースの状態を確かめずに、その回答を削除する |
| TC-ports-submissionRepository-109 | 期限を過ぎた、送信待ち、送信中、送信成功、送信失敗、対処済み、再試行の送信待ちの回答が 1 件ずつある | UoW で `deleteExpired(now, BoundedDeletion.maxLimit)` を呼び、確定する | 6 を返す。すべて削除する |
| TC-ports-submissionRepository-110 | 期限を過ぎた回答が 2 件ある | UoW で `deleteExpired(now, BoundedDeletion.maxLimit)` を呼び、`fn` が例外を投げる | 2 件とも残る。通知も残る |
| TC-ports-submissionRepository-111 | 期限を過ぎた回答が 2 件ある | UoW で `deleteExpired(now, 0)` を呼ぶ。別の UoW で、`BoundedDeletion.maxLimit` を超える `limit` で `deleteExpired(now, limit)` を呼ぶ | どちらも `SystemError`（`INVALID_QUERY`）になる。`countExpired(now)` は 2 |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-submissionRepository-113 | W1 に回答が無い | UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼び、確定する | 0 を返す |
| TC-ports-submissionRepository-114 | W1 に 3 件ある | UoW で `deleteAll(3)` を呼び、確定する | 3 を返す。`findPage(…)` は空の列と `count` 0 |
| TC-ports-submissionRepository-115 | W1 に 3 件ある | UoW で `deleteAll(2)` を呼び、確定する。別の同じ前提で、`deleteAll(1)` を呼び、確定する | 前者は 2 を返し、`findPage(…)` の `count` は 1。後者は 1 を返し、`count` は 2 |
| TC-ports-submissionRepository-117 | W1 に 5 件ある | 呼び出しごとに別の UoW で、`deleteAll(2)` を、返る件数が 2 より少なくなるまで繰り返す | 2、2、1 の順に返す。W1 に回答は残らない |
| TC-ports-submissionRepository-119 | W1 に、送信待ち、送信中、送信成功、送信失敗、再試行の送信待ちの回答が 1 件ずつある | UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼び、確定する | 5 を返す |
| TC-ports-submissionRepository-120 | W1 の区画の `Workspace` は `active`。W1 に 2 件ある | UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼び、確定する | ワークスペースの状態を確かめずに 2 を返す |
| TC-ports-submissionRepository-121 | W1 に 2 件ある | UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼び、`fn` が例外を投げる | 2 件とも残る。通知も残る |
| TC-ports-submissionRepository-122 | W1 に 2 件ある | UoW で `deleteAll(0)` を呼ぶ。別の UoW で、`BoundedDeletion.maxLimit` を超える `limit` で `deleteAll` を呼ぶ | どちらも `SystemError`（`INVALID_QUERY`）になる。`findPage(…)` の `count` は 2 |

## 回答と通知の対応

「通知が現れない」は、`SubmissionNotificationRepository` の `findById` が `null`、`findByIds` が含まず、`countUnhandledFailures` と `summarizeByForm` が数えず、`findUnsettled` と `findStalled` が返さないことを指す。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-submissionRepository-124 | W1 に、送信失敗の S1 と、止まった通知（`SubmissionNotification.stalled` が `waiting` を返す）の S2 がある | UoW で `deleteMany([S1, S2])` を呼び、確定する | S1 と S2 の通知が現れない |
| TC-ports-submissionRepository-125 | 同上。S1 と S2 は保持期限を過ぎている | UoW で `deleteExpired(now, BoundedDeletion.maxLimit)` を呼び、確定する | S1 と S2 の通知が現れない |
| TC-ports-submissionRepository-126 | 同上 | UoW で `deleteAll(BoundedDeletion.maxLimit)` を呼び、確定する | S1 と S2 の通知が現れない |
| TC-ports-submissionRepository-127 | W1 に S1、S2 がある | UoW で `deleteMany([S1])` を呼び、確定する | S2 の通知は、`SubmissionNotificationRepository.findById(S2)` が返す |
| TC-ports-submissionRepository-128 | S1 の通知を `SubmissionNotificationRepository.findById` で読み、その後に `deleteMany([S1])` を確定した | UoW で、読んだ `expectedVersion` で通知を `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。回答と通知のどちらも `null` のまま |
| TC-ports-submissionRepository-129 | S1 の通知を読む | 二つの UoW が並行して、通知の `save` と `deleteMany([S1])` を行う | `deleteMany` が確定した後、回答と通知のどちらも `null`。通知だけが残る結果は無い |

## 可視性と UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-submissionRepository-130 | 回答が無い | UoW で回答と通知を `insert` し、確定の直後に `findById`、`findPage`、`previewRetention` を呼ぶ | 待たずに、どれもその回答を反映する |
| TC-ports-submissionRepository-131 | W1 に、検索に該当する値を持つ F1 の S1 がある | UoW で `deleteMany([S1])` を呼び、確定する | `findById(S1)` は `null`。`findPage`（条件なし、`formId`、`search`）は S1 を含まず、`count` に数えない。`previewRetention(F1, …)` と `countExpired` は S1 を数えない |
| TC-ports-submissionRepository-132 | W1 に S1 がある | `deleteExpired`、`deleteAll` のそれぞれで S1 を削除して確定する | 前の行と同じく、どの読み取りにも S1 が現れない |
| TC-ports-submissionRepository-133 | 回答が無い | 一つの UoW で、回答の `insert` と、その通知の `insert` を行い、`fn` が例外を投げる | `run` はその例外を投げる。回答も通知も `null` |
| TC-ports-submissionRepository-134 | W1 に S1 がある | 一つの UoW で、S2 の回答と通知の `insert` と、S1 と同じ `submissionId` の通知の `SubmissionNotificationRepository.insert` を行う | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。S2 の回答も通知も `null` |
| TC-ports-submissionRepository-135 | W1 に S1 と S2 がある | 一つの UoW で、S3 の `insert` と `deleteMany([S2])` を行い、確定する | 二つの書き込みがすべて反映される |
| TC-ports-submissionRepository-136 | W1 に S1 と S2 がある | 一つの UoW で、S3 の `insert` と `deleteMany([S2])` を行い、`fn` が例外を投げる | `run` はその例外を投げる。S3 は `null`、S2 は残る |
