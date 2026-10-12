# 設置を更新する（`updateInstallation`）

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-installation-updateInstallation-01 | `operator`。更新が必要な設置。資源は足りている。バックアップは一つも無い | 設置を更新する | バックアップの有無を確かめない。保存済みのデータを実行中の版へ更新し、更新後の版と、更新が完了したことを返す。`Installation` の保存済みのデータの版は実行中の版になり、更新の進行は `idle`。復旧の記録は変わらない。設置は稼働中になり、以後の起動は、更新が必要なことを示さずに起動する |
| TC-installation-updateInstallation-02 | `operator`。更新が必要な設置。更新先の版が必要とする、データの保存容量が不足している | 設置を更新する | `BusinessRuleError`（`UpdateNotReady`）。不足する資源（`resource_short` と、その資源の使用量と上限）を添える。更新を始めず、データと保存済みの版は変わらず、更新の進行は `idle` のまま。CMS を更新前の版の CMS に戻して起動すると、起動して稼働中になる |
| TC-installation-updateInstallation-03 | `operator`。更新を完了した後に、CMS を更新前の版の CMS に戻した設置（保存済みのデータの版は更新先の版。実行中の CMS は、その版からの更新を持たない）。資源は足りている | 設置を更新する | `BusinessRuleError`（`UpdateNotSupported`）。保存済みの版と実行中の版を添える。更新を始めず、データと保存済みの版は変わらず、更新の進行は `idle` のまま。CMS を、保存済みのデータの版を扱う CMS に入れ替えて起動すると、稼働中になる |
| TC-installation-updateInstallation-04 | `operator`。更新できる設置。データの更新が失敗する | 設置を更新する | `SystemError`（理由 `UPDATE_FAILED`）。理由を示し、更新を完了と扱わない。保存済みのデータの版は進まず、`Installation` の更新の進行は、更新先の版を持つ `updating` で残る。設置は更新の途中。利用者の要求を受け付けない |
| TC-installation-updateInstallation-05 | `operator`。更新できる設置。更新の開始の保存の後、データの更新の結果を受ける前に中断する | 設置を更新する | `Installation` の更新の進行は `updating` で残り、保存済みのデータの版は進まない。設置は更新の途中 |
| TC-installation-updateInstallation-06 | `operator`。更新中の設置。CMS を更新前の版の CMS に戻している | 設置を更新する | `BusinessRuleError`（`CommandNotAccepted`、理由 `update_in_progress`）。更新を始めて完了していないことと、更新先の版を示す。データと `Installation` は変わらない |
| TC-installation-updateInstallation-07 | `operator`。更新中の設置。原因を解消している。実行中の CMS は更新先の版 | 設置をもう一度更新する | 更新が成功し、更新後の版と完了を返す。保存済みのデータの版は実行中の版になり、更新の進行は `idle` になる。設置は稼働中になる |
| TC-installation-updateInstallation-08 | `operator`。更新中の設置。実行中の CMS は更新先の版。更新先の版が必要とする資源が不足している | 設置をもう一度更新する | `BusinessRuleError`（`UpdateNotReady`）。不足する資源を添える。データは変わらず、更新の進行は `updating` のまま |
| TC-installation-updateInstallation-09 | `operator`。データの更新は成功し、更新の完了の保存が確定しなかった設置（更新の進行が `updating`、保存済みのデータは更新先の版の形） | 設置をもう一度更新する | データを変えずに、保存済みのデータの版を実行中の版に、更新の進行を `idle` にし、更新後の版と完了を返す |
| TC-installation-updateInstallation-10 | `operator`。更新中の設置。一つ目の実行のデータの更新が進行中 | 二つ目の実行が、設置を更新する | 二つ目の実行は `ConflictError`（理由 `UPDATE_IN_PROGRESS`）。データと `Installation` を変えず、更新が進行中であることを示す。一つ目の実行は、完了して、更新後の版を返す |
| TC-installation-updateInstallation-11 | `operator`。更新が必要な設置（更新の進行は `idle`）。二つの実行が、どちらも、更新の進行が `idle` の状態を確認している。一つ目の実行が、更新の開始を確定する | 二つ目の実行が、更新の開始の UnitOfWork を行う | 二つ目の実行は、何も書かずに `ConflictError`（理由 `UPDATE_IN_PROGRESS`）を返し、更新が進行中であることを示す。データの更新を呼ばない。`Installation` は、一つ目の実行が保存した `updating` のまま。一つ目の実行は、続けてデータを更新し、更新後の版と完了を返す |
| TC-installation-updateInstallation-12 | `operator`。更新中の設置。二つの実行のデータの更新が、順に成功を返している。一つ目の実行が、更新の完了を確定する | 二つ目の実行が、更新の完了の UnitOfWork を行う | 二つ目の実行は、何も書かずに `ConflictError`（理由 `STALE_VERSION`）を返す。`Installation` は、一つ目の実行が保存した値のまま（保存済みのデータの版は更新先の版、更新の進行は `idle`）。一つ目の実行は、更新後の版と完了を返している |
| TC-installation-updateInstallation-13 | `operator`。稼働中の設置（保存済みのデータの版が実行中の版と同じ） | 設置を更新する | エラーにせず、更新が不要なことを返す。データと `Installation` は変わらない |
| TC-installation-updateInstallation-14 | `operator`。更新が必要な設置。データの保存容量とメディアの保管先の容量は足りている。通知用の空きを取得できない（`readResource` が `unavailable` を返す） | 設置を更新する | 更新を妨げず、保存済みのデータを実行中の版へ更新し、更新後の版と、更新が完了したことを返す。設置は稼働中になる |
