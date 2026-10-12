# SubmissionNotificationRepository

`spec/domains/form.md` の `SubmissionNotificationRepository` の契約と、`spec/domains/common.md` のリポジトリに共通する規則を、すべてのアダプターが通すケースで定める。

- 「UoW」は `UnitOfWorkProvider.run` の `fn`。「確定」は `fn` の正常終了を指す
- UoW の `WriteScope` は、どのケースも、対象のワークスペースの `{ kind: "workspace", workspaceId, condition: { kind: "none" } }`。区画と条件は、`unitOfWorkProvider.md` が確かめる
- リポジトリは、ワークスペースの区画に結び付く。断りが無い限り、UoW は W1 の UoW、読み取りは `PartitionReader.workspace(W1)` のリポジトリで、通知は W1 の通知。W2 の通知の書き込みは W2 の UoW で、読み取りは W2 の区画のリポジトリで行う
- 前提条件の書き込みは、別の UoW で確定済み。期待結果の読み取りは、UoW の外で行う
- 下の「`findByIds` の上限」は、`spec/domains/form.md` の「SubmissionNotificationRepository」の `findByIds` の契約が定める、`findByIds` が一度に受ける ID の数の上限を指す。「止まった」は、`SubmissionNotification.stalled(notification, now)` が `null` でないことを指す
- W1 と W2 は別の、確定済みのワークスペース。F1 と F2 は別の `FormId`（F1 < F2）。S1、S2、S3 は別の `SubmissionId`（S1 < S2 < S3）
- 通知は、`Submission.receive` が返した `notification` を `insert` し、同じ UoW で、その `submission` を `SubmissionRepository.insert` で保存して作る
- 通知の状態は、`SubmissionNotification` の振る舞いの結果を `save` して作る。振る舞いに渡す設定は、断りが無い限り、通知の `notifiedTo` を有効な通知先として持つ `active` の `FormSettings`

通知の呼び名と、その状態と作り方は次のとおり。

- 送信待ちの通知: `attempt` は `initial`、`state` は `waiting`。`insert` したまま
- 送信中の通知: `attempt` は `initial`、`state` は `sending`。`start` の `send` の結果
- 送信成功の通知: `succeeded`。全宛先が受理の `complete` の結果
- 送信失敗の通知: `failed`（`delivery`、未対処）。失敗を含む `complete` の結果
- 宛先なしの通知: `failed`（`noRecipients`、未対処）。有効な通知先が無い設定を渡した `start` の `record` の結果
- 対処済みの通知: `failed`（対処済み）。`markHandled` が返した `changed` の `value`
- 再試行の送信待ちの通知: `attempt` は `retry` の `round` 1、`state` は `waiting`。送信失敗の通知への `requestRetry` の結果
- 再試行の送信中の通知: `attempt` は `retry` の `round` 1、`state` は `sending`。再試行の送信待ちの通知への `start` の `send` の結果

## insert

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-submissionNotificationRepository-01 | 通知が無い | UoW で、回答と通知を `insert` して確定する | `findById` が、渡した通知と `submissionId`、`formId`、`receivedAt`、`notifiedTo`、`attempt`（`initial`）、`state`（`waiting` と `queuedAt`）、`deliveries`（空）、`version`（0）が等しい `entity` と、`expectedVersion` を返す |
| TC-ports-submissionNotificationRepository-02 | 通知が無い | `notifiedTo` が 3 件の通知と、1 件の通知を `insert` して確定する | `findById` が返す `notifiedTo` は、渡した順序と件数を保つ |
| TC-ports-submissionNotificationRepository-03 | S1 の通知がある（`insert` したまま、または `save` で `failed` まで進んでいる） | UoW で、`submissionId` が S1 の `waiting` の通知を `insert` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。`findById(S1)` は元の通知を返す |
| TC-ports-submissionNotificationRepository-05 | 通知が無い | 二つの UoW が並行して、同じ `submissionId` の回答と通知を `insert` する | 一方だけが確定する。他方の `run` は `ConflictError`（理由 `ALREADY_EXISTS`）を投げる |

## findById

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-submissionNotificationRepository-06 | 区画が無いワークスペース（存在しない ID） | その区画のリポジトリで、`findById(S1)`、`findByIds([S1])`、`countUnhandledFailures()`、`summarizeByForm()`、`findUnsettled(FormSweep.maxLimit)`、`findStalled(now, FormSweep.maxLimit)` を呼ぶ | `findById` は `null`、`countUnhandledFailures` は 0、ほかは空の列 |
| TC-ports-submissionNotificationRepository-07 | 通知が無い | `findById(S1)` を呼ぶ | `null` |
| TC-ports-submissionNotificationRepository-08 | S1 の通知がある | `findById(S1)` を呼ぶ | S1 の通知と `expectedVersion` を返す |
| TC-ports-submissionNotificationRepository-09 | W1 に S1、W2 に S2 の通知がある | W1 の区画のリポジトリで `findById(S2)` と `findByIds([S1, S2])` を、W2 の区画のリポジトリで `findById(S2)` を呼ぶ | W1 の区画の `findById` は `null`、`findByIds` は S1 の 1 件だけ。W2 の区画は S2 の通知を返す |
| TC-ports-submissionNotificationRepository-10 | S1 の通知がある | UoW の外で `findById(S1)` を呼ぶ | UoW を開かずに通知を返す |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-submissionNotificationRepository-11 | 送信待ちの通知を `findById` で読む | UoW で、`start` の `send` の結果を、読んだ `expectedVersion` で `save` して確定する | `findById` が、`state` が `sending`（`startedAt` と、送信中の宛先の `recipients`）、`deliveries` が空、`version` が 1 の通知を返す |
| TC-ports-submissionNotificationRepository-12 | 送信中の通知を `findById` で読む | UoW で、全宛先が受理の `complete` の結果を `save` して確定する | `findById` が、`state` が `succeeded`（`settledAt`）、宛先ごとの結果が `accepted`（`at`）の通知を返す |
| TC-ports-submissionNotificationRepository-13 | 送信中の通知を `findById` で読む | UoW で、受理と失敗が混ざる `complete` の結果を `save` して確定する | `findById` が、`state` が `failed`（`delivery`、`handling` は `unhandled`）、宛先ごとの `accepted` と `failed`（`at` と理由）の通知を返す |
| TC-ports-submissionNotificationRepository-14 | 送信待ちの通知を `findById` で読む | UoW で、有効な通知先が無い設定を渡した `start` の `record` の結果を `save` して確定する | `findById` が、`state` が `failed`（`noRecipients`、`handling` は `unhandled`）、`deliveries` が空の通知を返す |
| TC-ports-submissionNotificationRepository-15 | 送信失敗の通知を `findById` で読む | UoW で、`markHandled` が返した `changed` の `value`を `save` して確定する | `findById` が返す `handling` は `handled`。`by` と `at` は渡した値 |
| TC-ports-submissionNotificationRepository-16 | 送信失敗の通知と、再試行の送信中の通知が失敗を含む `complete` で `failed` になった通知を、`findById` で読む | UoW で、それぞれ `requestRetry` の結果を `save` して確定する | `findById` が返す `attempt` は、前者が `retry` の `round` 1、後者が `round` 2。どちらも `state` は `waiting`（`queuedAt`）で、`deliveries` は前の回の状態を保つ |
| TC-ports-submissionNotificationRepository-18 | 送信中の通知を `findById` で読む | UoW で、`abandon` の結果を `save` して確定する | `findById` が、`state` が `failed`、`recipients` にあった宛先の結果が `failed`（`unconfirmed`）の通知を返す |
| TC-ports-submissionNotificationRepository-19 | 通知を `save` して確定した | 保存の後に `findById` で読んだ `expectedVersion` で、次の変更を `save` して確定する | 成功する |
| TC-ports-submissionNotificationRepository-20 | 通知を `save` して確定した | 回答の `SubmissionRepository.findById` を呼ぶ | 回答の内容は変わらない |
| TC-ports-submissionNotificationRepository-21 | 送信待ちの通知を二度 `findById` で読み、一方の変更を `save` して確定した | UoW で、他方の変更を、最初に読んだ `expectedVersion` で `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById` は先に確定した内容を返す |
| TC-ports-submissionNotificationRepository-22 | 送信待ちの通知と、送信失敗の通知を `findById` で読む | 二つの UoW が並行して、送信待ちの通知に、同じ `expectedVersion` で `start` の結果と `abandon` の結果を `save` する。別の二つの UoW が並行して、送信失敗の通知に、同じ `expectedVersion` で `requestRetry` の結果を `save` する | どちらの組も、一方だけが確定する。他方の `run` は `ConflictError`（理由 `STALE_VERSION`）を投げる。送信失敗だった通知の `attempt` は `retry` の `round` 1 |
| TC-ports-submissionNotificationRepository-24 | 通知が無い | UoW で、存在しない `submissionId` の通知を任意の `expectedVersion` で `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById` は `null` |
| TC-ports-submissionNotificationRepository-26 | S1 と S2 の通知を読み、S2 は別の変更を `save` して確定した | 一つの UoW で、S1 の `save` と、最初に読んだ `expectedVersion` での S2 の `save` を行う | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。S1 は変更前のまま |

## findByIds

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-submissionNotificationRepository-27 | S1、S2、S3 の通知がある | `findByIds([S3, S1, S2])` を呼ぶ | S1、S2、S3 の順で返す |
| TC-ports-submissionNotificationRepository-28 | S1 の通知が `save` で `failed` まで進んでいる | `findByIds([S1])` を呼び、返った `expectedVersion` で `save` する | `findById(S1)` と等しい `entity` と `expectedVersion` を返す。その `expectedVersion` で `save` できる |
| TC-ports-submissionNotificationRepository-29 | S1 の通知がある | `findByIds([])` を呼ぶ | 空の列 |
| TC-ports-submissionNotificationRepository-30 | S1 の通知がある。S2 は無い | `findByIds([S1, S2])` を呼ぶ | S1 の通知だけ |
| TC-ports-submissionNotificationRepository-32 | S1、S2 の通知がある | `findByIds([S2, S1, S2])` を呼ぶ | S1、S2 の 2 件 |
| TC-ports-submissionNotificationRepository-34 | `findByIds` の上限と同じ件数の通知がある | そのすべての ID を渡して `findByIds` を呼ぶ | すべてを `submissionId` の昇順で返す |
| TC-ports-submissionNotificationRepository-36 | `findByIds` の上限と同じ件数の通知がある | そのすべての ID に、そのうち 1 個の重複を加えて `findByIds` を呼ぶ | すべてを 1 件ずつ返す。上限は、重複を除いた数に適用する |
| TC-ports-submissionNotificationRepository-37 | S1 の通知がある | 重複の無い、`findByIds` の上限を 1 個超える数の ID を渡して `findByIds` を呼ぶ | `SystemError`（理由 `INVALID_QUERY`）になる。上限を超える入力を切り詰めない |

## countUnhandledFailures

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-submissionNotificationRepository-39 | W1 に通知が無い。または、送信待ち、送信中、送信成功の通知が 1 件ずつある | `countUnhandledFailures()` を呼ぶ | 0 |
| TC-ports-submissionNotificationRepository-42 | W1 に、送信失敗の通知が 2 件、宛先なしの通知が 1 件ある | `countUnhandledFailures()` を呼ぶ | 3 |
| TC-ports-submissionNotificationRepository-43 | W1 に、送信失敗の通知が 2 件、対処済みの通知が 1 件ある | `countUnhandledFailures()` を呼ぶ | 2 |
| TC-ports-submissionNotificationRepository-44 | W1 に、再試行の送信待ちの通知と、再試行の送信中の通知がある | `countUnhandledFailures()` を呼ぶ | 0 |
| TC-ports-submissionNotificationRepository-45 | W1 の再試行の送信中の通知が、失敗を含む `complete` で `failed` になった | `countUnhandledFailures()` を呼ぶ | 1 |
| TC-ports-submissionNotificationRepository-46 | W1 に F1 の送信失敗の通知が 1 件、W2 に F2 の送信失敗の通知が 2 件ある | W1 の区画のリポジトリで `countUnhandledFailures()` と `summarizeByForm()` を呼ぶ | `countUnhandledFailures` は 1。`summarizeByForm` は F1 の一件だけ |

## summarizeByForm

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-submissionNotificationRepository-48 | W1 に通知が無い | `summarizeByForm()` を呼ぶ | 空の列 |
| TC-ports-submissionNotificationRepository-50 | W1 に、F1 の送信待ち、送信中、送信成功、送信失敗、宛先なし、対処済み、再試行の送信待ち、再試行の送信中の通知が 1 件ずつある | `summarizeByForm()` を呼ぶ | F1 の一件。`unsent` 4、`succeeded` 1、`failed` 3 |
| TC-ports-submissionNotificationRepository-51 | W1 に、F2 の送信失敗の通知が 1 件、F1 の送信成功の通知が 2 件を、この順に `insert` した | `summarizeByForm()` を呼ぶ | F1（`unsent` 0、`succeeded` 2、`failed` 0）、F2（`unsent` 0、`succeeded` 0、`failed` 1）の順で返す |

## findUnsettled

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-submissionNotificationRepository-55 | 通知が無い。または、送信成功、送信失敗、宛先なし、対処済みの通知だけがある | `findUnsettled(FormSweep.maxLimit)` を呼ぶ | 空の列 |
| TC-ports-submissionNotificationRepository-57 | 送信待ち、送信中、再試行の送信待ち、再試行の送信中、送信成功、送信失敗の通知が 1 件ずつある | `findUnsettled(FormSweep.maxLimit)` を呼ぶ | 送信待ち、送信中、再試行の送信待ち、再試行の送信中の 4 件の ID |
| TC-ports-submissionNotificationRepository-58 | `receivedAt` が互いに異なる送信待ちの通知 3 件を、`receivedAt` の降順に `insert` した。別に、`receivedAt` が同じ送信待ちの通知が 2 件ある | `findUnsettled(FormSweep.maxLimit)` を呼ぶ | `receivedAt` の昇順で返す。同じ `receivedAt` は `submissionId` の昇順 |
| TC-ports-submissionNotificationRepository-60 | W1 に、止まった送信待ちの通知が 2 件、W2 に 1 件ある | W1 の区画のリポジトリで `findUnsettled(FormSweep.maxLimit)` と `findStalled(now, FormSweep.maxLimit)` を呼ぶ | どちらも W1 の 2 件の ID だけ |
| TC-ports-submissionNotificationRepository-61 | 送信待ちの通知が 3 件ある | `findUnsettled(3)`、`findUnsettled(2)`、`findUnsettled(1)` を呼ぶ | 順に、3 件、並び順の先頭 2 件、先頭 1 件 |
| TC-ports-submissionNotificationRepository-62 | 送信待ちの通知に `abandon` を適用して `save` した | `findUnsettled(FormSweep.maxLimit)` を呼ぶ | その通知を含まない |
| TC-ports-submissionNotificationRepository-63 | 送信待ちの止まった通知が 1 件ある | `findUnsettled(0)`、`FormSweep.maxLimit` を超える `limit` の `findUnsettled`、`findStalled(now, 0)`、`FormSweep.maxLimit` を超える `limit` の `findStalled` を呼ぶ | どれも `SystemError`（理由 `INVALID_QUERY`）になる |

## findStalled

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-submissionNotificationRepository-66 | 通知が無い。または、`settledAt` から `SubmissionNotification.stallAfterMs` 以上が過ぎた、送信成功、送信失敗、対処済みの通知だけがある | `findStalled(now, FormSweep.maxLimit)` を呼ぶ | 空の列 |
| TC-ports-submissionNotificationRepository-67 | `queuedAt` から `SubmissionNotification.stallAfterMs` 以上が過ぎた送信待ちの通知と、過ぎていない送信待ちの通知がある | `findStalled(now, FormSweep.maxLimit)` を呼ぶ | 過ぎた通知の ID だけ |
| TC-ports-submissionNotificationRepository-68 | `queuedAt` から `SubmissionNotification.stallAfterMs` 以上が過ぎた再試行の送信待ちの通知と、過ぎていない再試行の送信待ちの通知がある | `findStalled(now, FormSweep.maxLimit)` を呼ぶ | 過ぎた通知の ID だけ |
| TC-ports-submissionNotificationRepository-69 | `startedAt` から `SubmissionNotification.stallAfterMs` 以上が過ぎた送信中の通知と、過ぎていない送信中の通知がある | `findStalled(now, FormSweep.maxLimit)` を呼ぶ | 過ぎた通知の ID だけ |
| TC-ports-submissionNotificationRepository-70 | `receivedAt` からは `SubmissionNotification.stallAfterMs` 以上が過ぎ、`startedAt` からは過ぎていない、送信中の通知がある | `findStalled(now, FormSweep.maxLimit)` を呼ぶ | 空の列 |
| TC-ports-submissionNotificationRepository-71 | `receivedAt` からは `SubmissionNotification.stallAfterMs` 以上が過ぎ、`requestRetry` の `queuedAt` からは過ぎていない、再試行の送信待ちの通知がある | `findStalled(now, FormSweep.maxLimit)` を呼ぶ | 空の列 |
| TC-ports-submissionNotificationRepository-74 | `receivedAt` が互いに異なる、止まった通知が 3 件と、`receivedAt` が同じ、止まった通知が 2 件ある | `findStalled(now, FormSweep.maxLimit)` を呼ぶ | `receivedAt` の昇順で返す。同じ `receivedAt` は `submissionId` の昇順 |
| TC-ports-submissionNotificationRepository-77 | 止まった通知が 3 件ある | `findStalled(now, 3)`、`findStalled(now, 2)`、`findStalled(now, 1)` を呼ぶ | 順に、3 件、並び順の先頭 2 件、先頭 1 件 |
| TC-ports-submissionNotificationRepository-78 | 止まった送信中の通知に `abandon` を適用して `save` した | `findStalled(now, FormSweep.maxLimit)` を呼ぶ | その通知を含まない |

## 可視性と UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-submissionNotificationRepository-82 | 通知が無い | UoW で回答と通知を `insert` し、確定の直後に `findById`、`findByIds`、`summarizeByForm`、`findUnsettled` を呼ぶ | 待たずに、どれもその通知を反映する |
| TC-ports-submissionNotificationRepository-83 | 送信中の通知がある | UoW で、失敗を含む `complete` の結果を `save` し、確定の直後に `countUnhandledFailures` と `findUnsettled` を呼ぶ | `countUnhandledFailures` は数え、`findUnsettled` は返さない |
| TC-ports-submissionNotificationRepository-84 | 送信待ちの通知がある | UoW で `start` の結果を `save` し、`fn` が例外を投げる | `run` はその例外を投げる。`findById` は送信待ちの通知を返す |
| TC-ports-submissionNotificationRepository-85 | 通知が無い | 一つの UoW で、回答と通知の `insert` を行い、`fn` が例外を投げる | 通知も回答も `null` |
| TC-ports-submissionNotificationRepository-86 | S1 と S2 の通知がある | 一つの UoW で、S1 と S2 の `requestRetry` の結果を `save` して確定する | 両方が、`attempt` が `retry` の `round` 1 の `waiting` になる |
