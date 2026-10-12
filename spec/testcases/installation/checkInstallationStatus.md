# 状態を確認する（`checkInstallationStatus`）

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-installation-checkInstallationStatus-01 | `operator`。資源は足りている。開設済みのワークスペース 2 つに、未送信・未対処の送信失敗の通知、適用中の反映と、未解消の失敗した反映が無い。どちらにも所有者がいる。隔離されたイベントが無い。`BackupPolicy.intervalMs` の内に取得した、整合性の確認に成功したバックアップがある | 状態を確認する | 資源ごとの使用量、上限、判定 `sufficient` を返す。ワークスペースごとに、各ドメインが「状態確認の項目」に公開するワークスペースごとの項目を、件数 0、対処は不要として返す。設置の全体の項目（作成の途中、利用終了の途中、所有者がいないワークスペース）は、件数 0 で、対処は不要。隔離されたイベントの件数は 0。最後のバックアップは `taken` で、取得日時と整合性 `verified` を持ち、`overdue` は偽。必要な対処はどの項目も無い。報告は、回答の入力内容、連絡先、秘密情報を含まない |
| TC-installation-checkInstallationStatus-02 | `operator`。メディアの保存容量が不足している | 状態を確認する | その資源を `short` と示し、必要な対処 `expand_capacity` を示す |
| TC-installation-checkInstallationStatus-03 | `operator`。資源の一つの状態を取得できない | 状態を確認する | エラーにしない。その資源を取得できなかったことと理由を報告に載せ、必要な対処 `rerun_check` を示す。不足なしと示さない |
| TC-installation-checkInstallationStatus-04 | `operator`。あるワークスペースの二つのフォームに、未送信の通知が 1 件と 2 件、未対処の送信失敗の通知が 1 件、対処済みの送信失敗の通知が 1 件ある | 状態を確認する | そのワークスペースの項目として、Form が公開する未送信の通知の項目を 3 件（対処は不要）、未対処の送信失敗の通知の項目を 1 件（対処が要る）で、別の項目として返す。ほかのワークスペースの項目と分けて示す |
| TC-installation-checkInstallationStatus-05 | `operator`。あるワークスペースに、未対処の表示アプリへの通知の失敗と、送信待ちの通知がある | 状態を確認する | そのワークスペースの項目として、Webhook が公開する未対処の失敗の項目を、失敗の件数で、対処が要る項目として返し、送信待ちの項目を、送信待ちの件数で、対処は不要として返す。二つは別の項目 |
| TC-installation-checkInstallationStatus-06 | `operator`。あるワークスペースに、適用中の反映がある。別のワークスペースに、未解消の失敗した反映がある | 状態を確認する | 前者のワークスペースの、Schema が公開する適用中の反映の項目は 1 件で、対処は不要。後者のワークスペースの、未解消の失敗した反映の項目は 1 件で、対処が要る |
| TC-installation-checkInstallationStatus-07 | `operator`。開設済みのワークスペース 2 つに、隔離されたイベントが 1 件ずつある | 状態を確認する | 隔離されたイベントの件数 2 と、必要な対処 `redeliver_quarantined_events` を示す。件数は、ワークスペースごとの件数を足した値で、ワークスペースで分けない。エラーにしない |
| TC-installation-checkInstallationStatus-08 | `operator`。隔離されたイベントが 1 件ある開設済みのワークスペースの利用を終了し、利用終了が完了している | 状態を確認する | 隔離されたイベントの件数は 0。利用を終了したワークスペースのイベントを数えない |
| TC-installation-checkInstallationStatus-09 | `operator`。開設済みのワークスペースのほかに、作成の途中で名簿で準備中のワークスペースと、名簿で終了中のワークスペースがある。終了中のワークスペースに、未対処の送信失敗の通知がある | 状態を確認する | ワークスペースごとの項目の列は、開設済みのワークスペースだけを持つ。準備中のワークスペースは、作成の途中の項目（1 件、対処が要る）に、終了中のワークスペースは、利用終了の途中の項目（1 件、対処は不要）に、ID と名称で示す。どちらも、添える値は `null`。終了中のワークスペースの通知の件数を示さない |
| TC-installation-checkInstallationStatus-10 | `operator`。作成が、名簿への登録の後、区画の生成の前に止まったワークスペースと、区画の生成の後、名簿を開設済みにする前に止まったワークスペースがある | 状態を確認する | 二つとも、作成の途中の項目（2 件、対処が要る）に、ID と名称で示す。区画を読まない |
| TC-installation-checkInstallationStatus-11 | `operator`。利用終了の実行が、名簿の状態の変更の後、開始の前に止まったワークスペースがある（名簿で終了中、区画の `Workspace` は `active`） | 状態を確認する | そのワークスペースを、利用終了の途中の項目に示す。ワークスペースごとの項目の列に含めない |
| TC-installation-checkInstallationStatus-12 | `operator`。開設済みのワークスペースの一つが、復旧で所有者を失っている（所有者の指定が `vacant` の `active`）。`pending` の初期所有者の招待は無い | 状態を確認する | 所有者がいないワークスペースの項目を、1 件、対処が要る項目（`actionRequired` が真）として返し、そのワークスペースの ID と名称を示し、初期所有者の招待の状態 `none` を `detail` の `invitation` に添える。そのワークスペースの、ほかのドメインの項目も返す |
| TC-installation-checkInstallationStatus-13 | `operator`。作成を終え、初期所有者が未参加のワークスペースが 2 つある。一方の初期所有者の招待は期限内、もう一方は期限を過ぎている | 状態を確認する | 所有者がいないワークスペースの項目を、2 件、対処が要る項目として返す。前者に初期所有者の招待の状態 `usable` を、後者に `expired` を添える。招待の宛先と値を示さない |
| TC-installation-checkInstallationStatus-14 | `operator`。作成が、名簿を開設済みにした後、初期所有者の招待の発行の前に止まったワークスペースがある（名簿で開設済み、所有者の指定が `vacant`、初期所有者の招待は無い） | 状態を確認する | そのワークスペースを、作成の途中の項目に示さず、所有者がいないワークスペースの項目に、ID と名称と初期所有者の招待の状態 `none` で示す |
| TC-installation-checkInstallationStatus-15 | `operator`。初期所有者が招待を受諾して参加したワークスペース | 状態を確認する | そのワークスペースを、所有者がいないワークスペースの項目に示さない |
| TC-installation-checkInstallationStatus-16 | `operator`。利用終了が完了したワークスペース（名簿に項目が無い）がある | 状態を確認する | そのワークスペースを、どの列にも示さない |
| TC-installation-checkInstallationStatus-17 | `operator`。バックアップが一つも無い | 状態を確認する | 最後のバックアップは `none`。必要な対処 `take_backup` を示す |
| TC-installation-checkInstallationStatus-18 | `operator`。最後のバックアップの取得から `BackupPolicy.intervalMs` 以上が過ぎている | 状態を確認する | 最後のバックアップは `taken` で、`overdue` は真。必要な対処 `take_backup` を示す |
| TC-installation-checkInstallationStatus-19 | `operator`。最後のバックアップが整合性の確認に失敗している | 状態を確認する | 最後のバックアップは `taken` で、整合性 `failed` を持つ。必要な対処 `take_backup` を示す |
| TC-installation-checkInstallationStatus-20 | `operator`。ワークスペースが無い設置 | 状態を確認する | ワークスペースごとの列が空で、隔離されたイベントの件数が 0 の報告を返す。設置の全体の項目は、どれも件数 0。資源と最後のバックアップを示す |
| TC-installation-checkInstallationStatus-23 | `operator`。資源は足りている。バックアップの保管先へ接続できない | 状態を確認する | エラーにしない。最後のバックアップは `unavailable` で、理由と必要な対処 `rerun_check` を示し、`take_backup` を示さない。資源、各ドメインの項目、隔離されたイベントの件数は示す |
