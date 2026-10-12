# 利用を再開する（`resumeInstallation`）

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-installation-resumeInstallation-01 | `operator`。復旧を終えた利用再開前の設置（`restored`）。保持期間を過ぎた保存回答が 2 件、復旧点で送信を終えていなかった通知（最初の送信を待つ通知、再試行を待つ通知、送信中の通知）を持つ保持期間内の回答が 1 件ずつ、復旧点で適用中だった反映が 1 件ある。設置の区画とワークスペースの区画に食い違いは無い | 利用を再開する | 再開した後の設置の状態（稼働中）を返す。保持期間を過ぎた回答 2 件とその通知は無くなる。送信を終えていなかった通知 3 件は送信失敗になり、送信しない。適用中だった反映は失敗（理由 `interrupted`）になり、ワークスペースの適用中が解け、準備領域は破棄される。復旧の記録が無くなって設置は稼働中になり、利用者の要求を受け付ける |
| TC-installation-resumeInstallation-02 | `operator`。`restored` の設置。済ませる処理の対象が無い | 利用を再開する | 再開した後の設置の状態（稼働中）を返す。設置のデータは変わらず、復旧の記録が無くなる |
| TC-installation-resumeInstallation-03 | `operator`。`restored` の設置。復旧の後に、CMS を、扱うデータの版が違う版へ入れ替えている。保持期間を過ぎた保存回答がある | 利用を再開する | `BusinessRuleError`（`CommandNotAccepted`、理由 `data_version_mismatch`）。保存済みの版と実行中の版、CMS を、復旧したデータの版を扱う CMS に合わせることを示す。済ませる処理を始めず、回答と復旧の記録は変わらない。CMS を戻した後の再開は、成功する |
| TC-installation-resumeInstallation-04 | `operator`。`restored` の設置。名簿で準備中で、区画に `Workspace` があるワークスペース（復旧点で、区画まで作り終えていた作成）と、名簿で開設済みで、区画の `Workspace` が `closing` のワークスペースがある | 利用を再開する | 前者は名簿で開設済みに、後者は終了中になる。設置は稼働中になる。前者は、所有者がいないワークスペースで、再開の後の状態確認は、作成の途中でなく、所有者がいないワークスペースの項目に示す。その招待は届いていない。終了中になったワークスペースは、再開の後に、定期ジョブ「利用終了の完了」が完了させる |
| TC-installation-resumeInstallation-05 | `operator`。`restored` の設置。開設済みのワークスペースに、所属の索引に `joined` の項目が無いメンバーと、メンバーが無い所属の索引の項目がある | 利用を再開する | 前者のメンバーの組は `joined` の項目になり、後者の項目は外れる。外れた結果、所属が無くなったアカウントは残る。設置は稼働中になる |
| TC-installation-resumeInstallation-06 | `operator`。`restored` の設置。開設済みのワークスペースの所有者のメンバーと、ほかのメンバー 1 人に、アカウントが無い。ワークスペースは、多要素認証を必須にしている | 利用を再開する | アカウントが無い二人のメンバーは削除される。ワークスペースは `active` のまま、所有者の指定が `vacant` になり、多要素認証の必須は変わらない。設置は稼働中になる。再開の後、残るメンバーは利用を続けられ、配信、受付、通知は止まらない。所有者だけの操作は、誰も行えない |
| TC-installation-resumeInstallation-07 | `operator`。`restored` の設置。保持期間を過ぎた保存回答が、送信済みか判断できない通知を持つ | 利用を再開する | その回答と通知を削除する。削除した回答の通知を、送信失敗として残さない |
| TC-installation-resumeInstallation-08 | `operator`。`restored` の設置。利用終了の途中のワークスペースに、送信済みか判断できない通知と、適用中だった反映がある | 利用を再開する | 利用終了の途中のワークスペースの通知も送信失敗に、反映も失敗にする。設置は稼働中になる |
| TC-installation-resumeInstallation-09 | `operator`。`restored` の設置。復旧で、メンバーの解除、権限の縮小、トークンの失効、個別の削除が元に戻っている。表示アプリへの通知に送信待ちがある | 利用を再開する | 元に戻ったメンバー、権限、トークン、コンテンツは変わらない。フォームの通知を再送しない。表示アプリへの通知の送信待ちは変わらない |
| TC-installation-resumeInstallation-10 | `operator`。`restored` の設置。復旧点より後に非公開にしたメディアが、復旧で公開に戻っている | 利用を再開する | メディアは変わらない。設置は稼働中になる。再開の後の配信は、復旧点の公開範囲で決まり、公開に戻ったメディアを配信する |
| TC-installation-resumeInstallation-11 | `operator`。`restored` の設置。名簿で準備中で、区画に `Workspace` が無いワークスペース（復旧点で、区画を作り終えていなかった作成）がある | 利用を再開する | そのワークスペースは準備中のまま残る。CMS は作成を進めない。設置は稼働中になる。再開の後の状態確認は、作成の途中の項目に示す |
| TC-installation-resumeInstallation-12 | `operator`。`restored` の設置。開設済みのワークスペースに、初期所有者が未参加の（所有者の指定が `vacant` の）ものがある | 利用を再開する | そのワークスペースは変わらない。設置は稼働中になる。再開の後の状態確認は、所有者がいないワークスペースの項目に、初期所有者の招待の状態を添えて示す |
| TC-installation-resumeInstallation-13 | `operator`。`restored` の設置。所属の索引に `joined` の項目が無いメンバーが 1 人ある。その所属の索引の確定は、確定しない | 利用を再開する | 確定しなかった対象を読み飛ばし、同じ実行でやり直さずに、残りの処理を行う。`BusinessRuleError`（`ResumeCleanupIncomplete`）を返し、残件数は、Workspace の、所属の索引で確定する対象の項目が 1 で、ほかの項目は 0。設置は利用再開前のまま残り、利用者の要求を受け付けない。確定するようになった後の実行で、再開する |
| TC-installation-resumeInstallation-17 | `operator`。`restored` の設置。送信を終えていない通知が、Form の利用再開前の処理が一度に読む件数より多くある | 利用を再開し、`ResumeCleanupIncomplete` を返した後に、もう一度再開する | 一度目は、一度に読んだ分だけを送信失敗にし、`BusinessRuleError`（`ResumeCleanupIncomplete`）を返す。一度目の後の、Form の、送信済みか判断できない通知の項目の残件数は、読まなかった通知の件数。設置は利用再開前のまま。残件数が 0 になった実行で、設置は稼働中になる |
| TC-installation-resumeInstallation-18 | `operator`。`restored` の設置。名簿で終了中のワークスペースの区画は、利用終了の区画の消去が回答の削除の途中で止まった時点の写しで、送信を終えていない通知を持つ回答が残っている | 利用を再開する | その通知は送信失敗になり、送信しない。設置は稼働中になる |
| TC-installation-resumeInstallation-19 | `operator`。`restored` の設置。回答の削除を終えた後、通知を送信失敗にする処理の途中で失敗・中断する | 利用を再開する | 削除した回答は戻らない。設置は利用再開前のまま残り、利用者の要求を受け付けない |
| TC-installation-resumeInstallation-20 | `operator`。再開が途中で失敗し、利用再開前のまま残る `restored` の設置 | 利用をもう一度再開する | Workspace、Form、Schema は、済んだ分をやり直さず、残りの処理だけを行う。設置は稼働中になる |
| TC-installation-resumeInstallation-21 | `operator`。`restored` の設置。保持期間を過ぎた保存回答がある。再開が、済ませる処理の後のもう一度の状態の確認で復旧の記録を読み直した後、記録を消す前に、並行した復旧の開始が、開始の記録を保存する | 再開の残りを行う | `ConflictError`（理由 `STALE_VERSION`）。復旧の記録は消えず、並行した復旧の `restoring` のまま。削除した回答は戻らない |
| TC-installation-resumeInstallation-23 | `operator`。`restored` の設置。保持期間を過ぎた保存回答がある。再開が済ませる処理を行う間に、並行した復旧の開始が、開始の記録を保存する | 再開の残りを行う | 処理の後の設置の状態の確認が受け付けず、`BusinessRuleError`（`CommandNotAccepted`）を返す。復旧の記録は消えず、並行した復旧の `restoring` のまま。削除した回答は戻らない |
| TC-installation-resumeInstallation-22 | `operator`。復旧を終えた利用再開前の設置（`restored`）。保持期間を過ぎた保存回答がある | 運用コマンドの入口で利用の再開を実行し、入口が示した利用再開前の確認に、確認を返さない（対話できない環境では、確認の指定を添えない） | 入口は「利用を再開する」を呼ばない。回答は残り、復旧の記録は変わらず、設置は利用再開前のまま |
