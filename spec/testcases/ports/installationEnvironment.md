# InstallationEnvironment

`spec/domains/installation.md` の `InstallationEnvironment` の契約を、すべてのアダプターが通すケースで定める。

- 「設置の設定」は、`InstallationSetting` の五つの項目
- 「UnitOfWork」は、`WriteScope` に `{ kind: "installation" }` を渡した `UnitOfWorkProvider.run` の `fn`
- 保存先、メディアの保管先、資源の状態は、試験用の相手（`spec/domains/common.md`）に指定する。試験用の相手は、操作ごとの成否を指定でき、受けた呼び出しと、残っている値を観測できる
- 「正しい設置」は、五つの設定がすべて正しく、試験用の相手がすべての操作を受け付ける設置
- 「秘密の値」は、設置の設定に指定した `secret_key` の値
- V0、V1、V2 は別の `DataVersion`。試験用の相手に、実行中の CMS が更新を持つ版を指定する

## readResource

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-installationEnvironment-01 | 試験用の相手に、資源ごとに別の使用量と上限を指定した | `data_storage`、`media_storage`、`notification_queue` のそれぞれで `readResource` を呼ぶ | それぞれ、`kind` がその資源、`status` が `measured` の値を返す。`used` と `limit` は、その資源に指定した値（0 以上の整数）に等しい |
| TC-ports-installationEnvironment-05 | 試験用の相手が、三つの資源の状態の取得を失敗させる | 三つの `kind` で `readResource` を呼ぶ | どれも例外を投げない。`kind` がその資源、`status` が `unavailable` の値と、空でない `reason` を返す。`reason` は秘密の値を含まない |
| TC-ports-installationEnvironment-08 | 設置にデータがある | `readResource` を三つの `kind` で呼び、その前後で各集約を読む | 設置のデータは変わらない |

## inspectSettings

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-installationEnvironment-09 | 正しい設置 | `inspectSettings` を呼ぶ | 5 件を返す。`setting` は `data_store`、`media_store`、`secret_key`、`public_url`、`system_mail` が一件ずつ。すべて `valid` |
| TC-ports-installationEnvironment-10 | `system_mail` を宣言していない設置 | `inspectSettings` を呼ぶ | 5 件を返す。`system_mail` は `not_declared`、ほかは `valid` |
| TC-ports-installationEnvironment-11 | `data_store`、`media_store`、`secret_key`、`public_url` のどれか一つを宣言していない設置 | それぞれの設置で `inspectSettings` を呼ぶ | 例外を投げない。5 件を返す。宣言していない設定は `not_declared` |
| TC-ports-installationEnvironment-15 | `data_store`、`media_store`、`system_mail` のどれか一つの、指定の形式に誤りがある、または必要な指定が欠けている設置 | それぞれの設置で `inspectSettings` を呼ぶ | 例外を投げない。該当する設定は `invalid` と、空でない `reason`。ほかの設定は、それぞれの状態を返す |
| TC-ports-installationEnvironment-18 | `secret_key` の指定の形式に誤りがある設置 | `inspectSettings` を呼ぶ | `secret_key` は `invalid` と `reason`。`reason` は、指定した鍵の値を含まない |
| TC-ports-installationEnvironment-20 | `public_url` に、絶対 URL でない値を指定した設置 | `inspectSettings` を呼ぶ | `public_url` は `invalid` と `reason` |
| TC-ports-installationEnvironment-21 | `public_url` に、`http` と `https` のどちらでもない URL、クエリを持つ URL、フラグメントを持つ URL を指定した設置 | それぞれの設置で `inspectSettings` を呼ぶ | どれも、`public_url` は `invalid` と `reason` |
| TC-ports-installationEnvironment-22 | 五つの設定のうち、二つが `invalid`、一つが `not_declared` になる設置 | `inspectSettings` を呼ぶ | 5 件を返す。該当する三つを、すべてそれぞれの状態で返す |
| TC-ports-installationEnvironment-23 | 五つの設定の指定の形は正しい。試験用の相手が、保存先への書き込み、メディアの保存、メールの送信を失敗させる | `inspectSettings` を呼ぶ | すべて `valid`。試験用の相手は、呼び出しを受けていない |
| TC-ports-installationEnvironment-25 | 正しい設置。`Installation` が無い設置と、データがある設置 | `inspectSettings` を呼び、その前後で `InstallationRepository.find` と各集約を読む | 設置のデータは変わらない。`Installation` が無い設置では、`find` は `null` のまま。試験用の相手は、書き込みを受けていない |

## probe

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-installationEnvironment-27 | 正しい設置 | `probe("data_store")` を呼ぶ | `status` が `passed`。完了後、試験用の相手に確認用の値は残らない |
| TC-ports-installationEnvironment-28 | 試験用の相手が、書いた値と違う値を、保存先の読み出しで返す | `probe("data_store")` を呼ぶ | 例外を投げない。`status` が `failed` と `reason`。完了後、確認用の値は残らない |
| TC-ports-installationEnvironment-29 | 正しい設置 | `probe("media_store")` を呼ぶ | `status` が `passed`。完了後、試験用の相手に確認用の実体は残らない |
| TC-ports-installationEnvironment-30 | 試験用の相手が、保存した実体と違う内容を、メディアの保管先の取り出しで返す | `probe("media_store")` を呼ぶ | 例外を投げない。`status` が `failed` と `reason`。完了後、確認用の実体は残らない |
| TC-ports-installationEnvironment-31 | 五つの設定の指定の形は正しい。試験用の相手が、保存先への書き込みを失敗させる | `probe("data_store")` を呼ぶ | 例外を投げない。`status` が `failed` と、空でない `reason`。`reason` は秘密の値を含まない |
| TC-ports-installationEnvironment-32 | 試験用の相手が、メディアの保管先への保存を失敗させる | `probe("media_store")` を呼ぶ | 例外を投げない。`status` が `failed` と `reason`。`reason` は秘密の値を含まない |
| TC-ports-installationEnvironment-33 | W1 にデータとメディアがある設置と、`Installation` が無い正しい設置 | `probe("data_store")` を呼び、その前後で各集約と `InstallationRepository.find` を読む | 各集約の一覧と件数は変わらない。確認用の値は、どの集約の読み取りにも現れない。`Installation` が無い設置では、`find` は `null` のまま |
| TC-ports-installationEnvironment-34 | W1 にメディアがある | `probe("media_store")` を呼び、その前後で W1 のメディアの記録と実体を読む | W1 のメディアの一覧、件数、実体は変わらない。確認用の実体は、メディアの読み取りに現れない |
| TC-ports-installationEnvironment-36 | 正しい設置 | `probe("data_store")` と `probe("media_store")` を、それぞれ 3 回続けて呼ぶ | すべて `passed`。前の呼び出しの確認用の値と実体は、次の呼び出しを妨げない |

## dataVersion

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-installationEnvironment-38 | 正しい設置 | `dataVersion` を呼ぶ | 空でない `DataVersion` を返す |
| TC-ports-installationEnvironment-39 | 正しい設置 | `dataVersion` を二度呼ぶ | 同じ値を返す |
| TC-ports-installationEnvironment-40 | 試験用の相手が、保存先の読み取りを失敗させる設置、`Installation` の `dataVersion` が別の値の設置、`Installation` が無い設置 | それぞれで `dataVersion` を呼ぶ | どれも、保存済みのデータを読まずに、実行中の CMS が扱う同じ値を返す |
| TC-ports-installationEnvironment-42 | 設置にデータがある | `dataVersion` を呼び、その前後で各集約を読む | 設置のデータは変わらない |

## supportsUpdateFrom

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-installationEnvironment-43 | `dataVersion()` は V2。実行中の CMS は、V1 からの更新を持ち、V0 からの更新を持たない | `supportsUpdateFrom(V1)` と `supportsUpdateFrom(V0)` を呼ぶ | 真と偽 |
| TC-ports-installationEnvironment-44 | `dataVersion()` は V2 | `supportsUpdateFrom(V2)` を呼ぶ | 真 |
| TC-ports-installationEnvironment-45 | 同上 | `supportsUpdateFrom(V1)` を二度呼ぶ | 同じ値を返す |
| TC-ports-installationEnvironment-46 | 試験用の相手が、保存先の読み取りを失敗させる | `supportsUpdateFrom(V1)` を呼ぶ | 保存済みのデータを読まずに、同じ値を返す |

## publicUrl

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-installationEnvironment-48 | `public_url` に `https://cms.example.com`、または経路を持つ URL（`https://example.com/cms/`）を宣言した設置 | `publicUrl` を呼ぶ | 宣言した値をそのまま返す |
| TC-ports-installationEnvironment-50 | 正しい設置 | `publicUrl` を二度呼ぶ | 同じ値を返す |
| TC-ports-installationEnvironment-51 | 試験用の相手が、保存先の読み取りを失敗させる | `publicUrl` を呼ぶ | 保存済みのデータを読まずに、宣言した値を返す |
| TC-ports-installationEnvironment-52 | `public_url` を宣言していない設置と、`public_url` の指定の形式に誤りがある設置 | `publicUrl` を呼ぶ | どちらも `SystemError` |

## restorableSizeLimit

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-installationEnvironment-54 | 正しい設置 | `restorableSizeLimit` を呼ぶ | 0 以上の整数を返す |
| TC-ports-installationEnvironment-55 | 正しい設置 | `restorableSizeLimit` を二度呼ぶ | 同じ値を返す |
| TC-ports-installationEnvironment-56 | 試験用の相手が、保存先の読み取りを失敗させる | `restorableSizeLimit` を呼ぶ | 保存済みのデータを読まずに、同じ値を返す |

## updateData

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-installationEnvironment-57 | 保存済みのデータが版 V1 の形。`dataVersion()` は V2。`supportsUpdateFrom(V1)` は真 | `updateData({ from: V1, to: V2 })` を呼ぶ | `ok: true` |
| TC-ports-installationEnvironment-58 | 同上 | `updateData({ from: V1, to: V2 })` を呼び、その前後で `InstallationRepository.find` を呼ぶ | `Installation` の記録（`dataVersion`、`version`）は変わらない |
| TC-ports-installationEnvironment-59 | `updateData({ from: V1, to: V2 })` が `ok: true` を返した | 同じ `plan` で、もう一度 `updateData` を呼び、その前後で各集約を読む | `ok: true`。データは変わらない |
| TC-ports-installationEnvironment-60 | `dataVersion()` は V2。`supportsUpdateFrom(V0)` は偽 | `plan.to` が V2 でない `updateData`、または `updateData({ from: V0, to: V2 })` を呼び、その前後で各集約を読む | どちらも例外を投げない。`ok: false`（`failed`）と、空でない `reason`。データは変わらない |
| TC-ports-installationEnvironment-62 | 試験用の相手が、データの更新を失敗させる | `updateData({ from: V1, to: V2 })` を呼ぶ | 例外を投げない。`ok: false`（`failed`）と `reason`。`reason` は秘密の値を含まない |
| TC-ports-installationEnvironment-63 | 試験用の相手が、データの更新を一度失敗させ、次から受け付ける | 同じ `plan` で `updateData` を二度呼ぶ | 一度目は `ok: false`（`failed`）、二度目は `ok: true` |
| TC-ports-installationEnvironment-64 | 試験用の相手が、`updateData({ from: V1, to: V2 })` を進行中のまま止めている | 同じ `plan` で、もう一度 `updateData` を呼ぶ | 例外を投げない。`ok: false`（`in_progress`） |
| TC-ports-installationEnvironment-65 | 上の後の呼び出しが `in_progress` を返した | 止めていた `updateData` を進める | `ok: true`。`in_progress` を返した呼び出しは、進行中の更新を妨げていない |
| TC-ports-installationEnvironment-66 | 進行中だった `updateData` が `ok: true` を返した | 同じ `plan` で `updateData` を呼ぶ | `ok: true`。`in_progress` を返さない |
| TC-ports-installationEnvironment-68 | 保存済みのデータが版 V1 の形 | UnitOfWork の中で `updateData({ from: V1, to: V2 })` を呼び、その後に `fn` が例外を投げる | `updateData` は `ok: true` を返す。データは V2 の形のまま |

## 並行性と可視性

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-installationEnvironment-69 | 正しい設置 | `probe("data_store")` と `probe("media_store")` を、それぞれ並行して二つ呼ぶ | どれも `passed` |
| TC-ports-installationEnvironment-71 | 正しい設置 | `readResource`、`inspectSettings`、`probe`、`dataVersion`、`supportsUpdateFrom`、`publicUrl`、`restorableSizeLimit` を並行して呼ぶ | それぞれが、単独で呼んだ場合と同じ種類の結果を返す |
| TC-ports-installationEnvironment-72 | 正しい設置。`dataVersion()` は V2 | UnitOfWork の外で、`readResource`、`inspectSettings`、`probe`、`dataVersion`、`supportsUpdateFrom`、`publicUrl`、`restorableSizeLimit`、`updateData` を呼ぶ | どれも、UnitOfWork を開かずに結果を返す |
| TC-ports-installationEnvironment-73 | 正しい設置 | UnitOfWork の中で `probe("data_store")` を呼び、その後に `fn` が例外を投げる | `probe` は `passed` を返す。確認用の値は、ワークスペースのデータに現れない |

`BackupStore.restore` の置き換えの完了より前に始まった `updateData` が、完了の後にデータを変えないことは、`backupStore.md` の「置き換えと書き込み」が、`BackupStore.capture` の呼び出しが進む間の `updateData` がデータを変えないことは、`backupStore.md` の「capture」が確かめる。
