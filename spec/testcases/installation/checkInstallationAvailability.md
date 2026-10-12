# 設置の状態を確かめる（`checkInstallationAvailability`）

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-installation-checkInstallationAvailability-01 | 稼働中の設置（初期化を終えた `Installation` があり、復旧の記録が無く、保存済みのデータの版が実行中の版と同じ） | 利用者の要求、ワーカーの処理、運用コマンドのそれぞれについて確かめる | 利用者の要求とワーカーの処理を受け付ける。運用コマンドは、利用再開前の確認と利用の再開を除くすべてを受け付ける |
| TC-installation-checkInstallationAvailability-02 | 稼働中の設置。設置の公開の URL の設定に、誤りがある | 利用者の要求、ワーカーの処理、起動のほかの運用コマンドのそれぞれについて確かめる | 利用者の要求とワーカーの処理を受け付けない。結果は、理由 `settings` と、該当する設定と理由を持ち、設置の状態を持たない。入口は、要求に `UnavailableError`（理由 `INSTALLATION_NOT_OPERATING`）を返し、ワーカーの処理を始めない。運用コマンドのうち、ワークスペースの作成、初期所有者の招待の発行し直し、管理者復旧を受け付けない。結果は、理由 `settings` と、`public_url` の問題だけを持ち、設置の状態を持たない。ほかの運用コマンドは、設定に問題が無い稼働中の設置と同じに受け付ける |
| TC-installation-checkInstallationAvailability-03 | 稼働中の設置。秘密情報の鍵、またはメディアの保管先の設定に、誤りがある | 起動のほかの運用コマンドのそれぞれについて確かめる | バックアップの取得と復旧を受け付けない。結果は、理由 `settings` と、誤りがある設定（`secret_key` または `media_store`）の問題だけを持ち、設置の状態を持たない。ほかの運用コマンドは、設定に問題が無い稼働中の設置と同じに受け付ける |
| TC-installation-checkInstallationAvailability-04 | 利用再開前の設置。メディアの保管先の設定を宣言していない | 利用者の要求と、利用の再開について確かめる | 利用者の要求を受け付けない。結果は、理由 `settings` と該当する設定を持ち、設置の状態を持たない。利用の再開は受け付ける |
| TC-installation-checkInstallationAvailability-05 | データの保存先の設定に、不足または誤りがある設置 | 利用者の要求、ワーカーの処理、運用コマンド（バックアップの一覧、復旧、状態確認、管理者復旧）のそれぞれについて確かめる | どれも受け付けない。`Installation` と復旧の記録を読まない。結果は、理由 `settings` と該当する設定と理由を持ち、設置の状態を持たない。運用コマンドの結果が持つ設定は `data_store` だけ。運用コマンドの入口は `BusinessRuleError`（`CommandNotAccepted`）を返し、設置の状態を示さず、該当する設定と理由を示す |
| TC-installation-checkInstallationAvailability-06 | データの保存先とメディアの保管先の両方の設定に誤りがある設置 | 利用者の要求と、状態確認について確かめる | 利用者の要求の結果は、二つの設定の問題を持つ。状態確認の結果は、`data_store` の問題だけを持つ。どちらも設置の状態を持たない |
| TC-installation-checkInstallationAvailability-07 | データの保存先の設定は正しい。保存先の障害で、設置の区画を読めない設置 | 利用者の要求とワーカーの処理について確かめる | `SystemError` を返す。入口は処理を始めず、利用者の要求には `SystemError` を返して保存先を読めないことを示し、ワーカーの処理ではイベントを配送の対象に残す |
| TC-installation-checkInstallationAvailability-08 | 初期化の前の設置（`Installation` と復旧の記録が無い） | 利用者の要求、ワーカーの処理、運用コマンドのそれぞれについて確かめる | 受け付けるのは、起動、バックアップの一覧、復旧だけ。利用者の要求とワーカーの処理を受け付けない。受け付けない結果は、設置の状態（初期化の前）を持つ |
| TC-installation-checkInstallationAvailability-09 | 保存先が空で、完了した復旧の記録（`restored`）だけが残る設置 | 同上 | 設置の状態は、利用再開前でなく、初期化の前。受け付けるのは、起動、バックアップの一覧、復旧だけ。利用の再開を受け付けない |
| TC-installation-checkInstallationAvailability-10 | 読み込みの途中の設置（`Installation` の初期構成が `loading`） | 同上 | 受け付けるのは、起動、バックアップの一覧、復旧だけ。利用者の要求とワーカーの処理を受け付けない |
| TC-installation-checkInstallationAvailability-11 | 更新が必要な設置（保存済みのデータの版が、実行中の CMS が扱う版と違う。更新の進行は `idle`） | 同上 | 受け付けるのは、設置の更新、バックアップの取得と一覧、復旧だけ。起動、利用者の要求、ワーカーの処理、管理者復旧を受け付けない |
| TC-installation-checkInstallationAvailability-12 | 更新中の設置（更新の進行が、更新先の版を持つ `updating`）。実行中の CMS は更新先の版 | 同上 | 設置の状態は更新の途中（`updating.target`）。受け付けるのは、設置の更新、バックアップの一覧、復旧だけ。起動とバックアップの取得を受け付けず、結果は、理由 `update_in_progress` と、更新先の版を持つ。利用者の要求とワーカーの処理を受け付けない |
| TC-installation-checkInstallationAvailability-13 | 更新中の設置。実行中の CMS は、更新先の版と違う版を扱う CMS（更新前の版の CMS を含む） | 利用者の要求、ワーカーの処理、運用コマンドのそれぞれについて確かめる | 設置の状態は更新の途中（`updating.other`）で、稼働中にならない。受け付けるのは、バックアップの一覧、復旧だけ。設置の更新、起動、バックアップの取得を受け付けず、結果は、理由 `update_in_progress` と、更新先の版を持つ |
| TC-installation-checkInstallationAvailability-14 | 復旧中の設置。復旧の記録が `restoring` で、現在時刻が、占有の期限（`leaseUntil`。開始から `Recovery.leaseMs` の後）より前 | 同上 | 設置の状態は復旧の進行中。受け付けるのは、起動、バックアップの一覧だけ。復旧を受け付けず、結果は、理由 `restore_occupied` と、始め直せる時期（占有の期限）を持つ。利用者の要求とワーカーの処理を受け付けない |
| TC-installation-checkInstallationAvailability-15 | 復旧中の設置。復旧の記録が `restoring` で、現在時刻が、占有の期限以降（結果が分からないまま中断した復旧） | 同上 | 設置の状態は復旧の失敗。起動、バックアップの一覧、復旧を受け付ける。利用者の要求、ワーカーの処理、利用再開前の確認、利用の再開を受け付けない |
| TC-installation-checkInstallationAvailability-16 | 復旧中の設置。復旧の記録が `failed`（失敗を示した復旧）。失敗の直後 | 同上 | 設置の状態は復旧の失敗。起動、バックアップの一覧、復旧を受け付ける。復旧は、占有の期限を待たずに受け付ける。利用者の要求とワーカーの処理を受け付けない |
| TC-installation-checkInstallationAvailability-17 | 利用再開前の設置（`Installation` があり、復旧の記録が `restored`。保存済みのデータの版が実行中の版と同じ） | 同上 | 設置の状態は `awaiting_resume.version_matched`。受け付けるのは、起動、利用の再開、バックアップの一覧、復旧だけ。利用者の要求とワーカーの処理、バックアップの取得、ワークスペースの作成と終了、管理者復旧を受け付けない |
| TC-installation-checkInstallationAvailability-18 | `Installation` が無く、復旧の記録が `restoring` または `failed` の設置 | 起動について確かめる | 受け付ける。設置の状態は、初期化の前でなく、復旧の進行中または復旧の失敗 |
| TC-installation-checkInstallationAvailability-19 | 利用再開前の設置。保存済みのデータの版が、実行中の CMS が扱う版と違う | 運用コマンドのそれぞれについて確かめる | 起動、バックアップの一覧、復旧を受け付ける。利用の再開を受け付けず、結果は、理由 `data_version_mismatch` と、保存済みの版・実行中の版を持つ。設置の更新とバックアップの取得を受け付けず、結果は、同じ理由 `data_version_mismatch` を持つ |
| TC-installation-checkInstallationAvailability-20 | 稼働中でない設置 | 管理画面の要求、アプリ向けの HTTP API の要求、メディアの配信の入口から確かめる | 入口は `UnavailableError`（理由 `INSTALLATION_NOT_OPERATING`）を返し、取得、フォーム受付、管理画面の利用を始めない。フォームの送信に受付完了を返さない |
| TC-installation-checkInstallationAvailability-21 | 稼働中でない設置 | 定期ジョブとイベントの消費の入口から確かめる | 入口は処理を始めず、通知を送らない。イベントは配送の対象に残る |
| TC-installation-checkInstallationAvailability-22 | 稼働中でない設置 | 運用コマンドの入口が、管理者復旧について確かめる | 入口は `BusinessRuleError`（`CommandNotAccepted`）を返し、設置の状態を示す。Identity のユースケースを呼ばず、回復の値を発行しない |
