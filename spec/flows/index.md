# 動的フロー台帳

複数のユースケースと時間をまたいで生きるフローを定める。状態は [../domains/](../domains/index.md) のエンティティのライフサイクルとポートの契約、遷移の主体は [../usecases/](../usecases/) のユースケースが定める。単一のユースケースの中で完結する処理は載せない。

## 読み方

- この台帳が持つのは、遷移（状態 → 状態）、遷移を起こす主体、途中で止まったときに残る状態と回復の主体、終端。一つのユースケース・振る舞いの中の手順と判定、規則の本文、並行の結果、ポートの契約、ユースケースの出力、定数は、定義元（[../usecases/](../usecases/) と [../domains/](../domains/index.md) の節、[../testcases/](../testcases/) のテスト）の名前で指す
- 経路の「主体」は、次の五つのいずれか
  - 要求のユースケース（メンバー、アプリ、運用コマンドの操作）。「名前（`識別子`）」（ドメイン）と書く
  - イベントの消費者のユースケース。「名前（`識別子`）」（ドメイン）（`イベント` の消費者）と書く
  - 定期ジョブのユースケース。「名前（`識別子`）」（ドメイン）（定期ジョブ「定期ジョブの名前」）と書く
  - ホスティング運営者（運用手順）。CMS の外で行う遷移で、CMS はその遷移のユースケースを持たない
  - 主体は無い。状態として保存せず、現在時刻から求める遷移（期限切れ）
- 複数の主体をまとめた表記（「送信を行う二つの主体」「予約を確定した同じ処理」など）は、その表の前の文、同じ表の前の行、または同じフローの、名前を挙げた経路（「経路（保管の予約）」の形）の表の行が、上の表記で名指しした主体を指す。ほかのフローの遷移の主体は、フローの ID を添えて指し、そのフローの経路が名指しする
- ユースケースの名前と識別子は `spec/usecases/${domain}.md` の見出しと一致する。識別子はドメインの中で一意で、ドメインの名前との組でユースケースを指す（[../domains/index.md](../domains/index.md) の「ユースケースの所属」）
- ドメインイベントと消費者、定期ジョブの定義は、[../domains/index.md](../domains/index.md) の「ドメインイベントで行うもの」「定期ジョブ」の表が持つ。この台帳は、イベントと定期ジョブを表の名前で指し、フローとの対応を「イベント・定期ジョブとフローの対応」に持つ。「起動」「実行」「ジョブ」の語は、同じ文書の「定期ジョブ」と「公開内容の整合性と通し番号」に従う
- 回数、間隔、期限、件数の上限は、ドメインのファイルが定める。この台帳は、値と定数を書かず、定める節の名前で指す
- ドメインイベントの記録、配送、隔離の規則は、[../domains/common.md](../domains/common.md) の「ドメインイベント」が定める。隔離されたイベントから出る遷移の主体は、ホスティング運営者（運用手順）の配送し直し
- 入口が処理を始める前の確認は [../domains/index.md](../domains/index.md) の「設置の状態」が、受け付ける設置と、受け付けない間の消費者と定期ジョブの扱いは [../domains/installation.md](../domains/installation.md) の「設置の状態」が定める。受け付けない間、各フローの消費者と定期ジョブの遷移は起こらない（F-24）
- 一つの UnitOfWork が書く区画と、確定の時点でワークスペースが満たす条件は、[../domains/common.md](../domains/common.md) の「WriteScope」と、各ユースケースのトランザクション境界が定める
- 二つの区画を書く操作の段の順は [../domains/index.md](../domains/index.md) の「区画をまたぐ手順」、各ドメインの手順（初期構成の生成、利用再開前の処理、利用終了の消去）を呼ぶ順は同じ文書の「手順を呼ぶ順」、各段の内容は「手順ごとの規則」が挙げる定義元が定める。この台帳は、段の途中で止まった状態と、その状態から出る遷移を持つ
- 要求のユースケースが確定しなかった UnitOfWork の扱いと、返す結果は、[../domains/common.md](../domains/common.md) の「TransactionalRepository」と [../domains/index.md](../domains/index.md) の「競合の扱い」が定める。各フローは、確定しなかった後に残る状態を書く
- ワークスペースの区画が対象の定期ジョブの、対象にする名簿の状態とワークスペースごとの進め方は、[../domains/index.md](../domains/index.md) の「定期ジョブ」の表と「区画をまたぐ手順」が定める
- イベントの消費者と定期ジョブが、処理を行えない対象（無くなった対象を含む）に当たったときの扱いは、[../domains/index.md](../domains/index.md) の「イベントの消費者と定期ジョブ」が定める。各フローは、その規則で残る状態と、そこから出る遷移を書く
- 利用できないワークスペースの扱いと、入口ごとの結果は、[../domains/index.md](../domains/index.md) の「利用できるワークスペース」が定める。各フローは、その規則で残る状態と、そこから出る遷移を書く
- 「利用終了の途中」「区画が `closing`」「区画が `closed`」の語は、[../domains/index.md](../domains/index.md) の「集約とポートの置き場所」に従う。利用終了の途中のワークスペースの区画に確定する書き込みは、同じ文書の「ワークスペースの作成と利用終了」が定める。各フローは、区画が `closing` になった後に残る状態と、そこから出る遷移を書く。名簿と区画の `Workspace` の組の遷移は、F-23 が持つ
- 期限つきの一度きりの値（セッション、回復依頼、招待、トークン、プレビューの引き換え、スクリプトの認証の依頼）の、使えなくなった後の保持と、保持期間を過ぎた値の扱いは、[../domains/common.md](../domains/common.md) の「期限つきの一度きりの値」が定める。各フローは、使えなくなった状態から出る遷移を書く
- ワークスペースの利用終了の区画の消去は、F-23 が持つ。消去するものは [../domains/workspace.md](../domains/workspace.md) の「利用終了の消去」と、各ドメインの「利用終了の消去」が定める。各フローの経路には、F-23 の区画の消去を書かない。区画の外の実体と、区画に結び付かない記録、設置の区画の集約と記録に及ぶ遷移は、そのフローの経路が持つ

## フローの一覧

| ID | フロー | 対象 |
| --- | --- | --- |
| F-01 | スキーマの反映 | `SchemaApplyJob`（`mode` が `commit`。`preparing` / `prepared` / `succeeded` / `failed`）と、`WorkspaceSchema` の `state`（`idle` / `applying`）、未解消の失敗した反映（`failedApply`） |
| F-02 | 適用の事前確認 | `SchemaApplyJob`（`mode` が `dryRun`。`preparing` / `checked` / `failed`） |
| F-03 | 準備領域 | 進行中の `SchemaApplyJob` ごとの準備領域（`ContentStagingArea` の準備領域と、`MediaUsageRepository` の準備領域） |
| F-04 | フォームの通知の送信と手動再試行 | `SubmissionNotification` の `state` と通知の回（`attempt`） |
| F-05 | 表示アプリへの通知の送信と再試行 | `ChangeNotification` と、通知と宛先の組ごとの送信（`WebhookDelivery`）、送信の要求 `webhook.delivery_requested` |
| F-06 | 派生形式の生成 | `MediaDerivatives` の派生形式ごとの `DerivativeState` |
| F-07 | メディアの実体 | `MediaStorage` の実体、保管の予約（`StorageReservation`）、消去待ちの記録（`ObjectRemoval`） |
| F-08 | バックアップ | `BackupStore` のバックアップ（区画ごとの写しと `BackupRecord`） |
| F-09 | セッション | `Session` |
| F-10 | 回復依頼 | `RecoveryRequest` |
| F-11 | 招待 | `Invitation` |
| F-12 | トークン | `Token`、宣言したトークンの索引（`DeclaredTokenIndex`） |
| F-13 | トークンの範囲の紐付け | `Token` の範囲（`TokenScope`）の要素（`ScopeTarget`） |
| F-14 | プレビューの引き換えの値 | `PreviewExchange` |
| F-15 | スクリプトの認証の依頼 | `DeviceAuthorization` |
| F-16 | 編集中の記録 | `EditingPresence` |
| F-17 | 履歴の版とごみ箱の保持 | `Revision`（保持中 / 保持期限切れ）と、ごみ箱の `Content` の保持。期限は `ContentRetentionPolicy` から決まる |
| F-18 | 回答の保持 | `Submission` |
| F-19 | コンテンツ | `Content`（公開状態、下書きの有無、置き場所）、公開の変更の記録（`PublicationRecord`） |
| F-20 | メディアの公開範囲 | `Media` の公開範囲、メディアの使用箇所の記録（`MediaUsageRepository`） |
| F-21 | フォームの運用設定 | 紐付け待ちの設定 `PendingFormSettings` と、`FormSettings`（`active` / `removed`） |
| F-22 | アカウントと所属の索引 | `Account`（多要素認証）、所属の索引（`MembershipIndex`）の項目 `Membership`（`joining` / `joined`。世代 `generation` を持つ）、アカウントごとの最後に利用したワークスペース |
| F-23 | ワークスペース | 名簿の項目（`WorkspaceDirectoryEntry`。`provisioning` / `open` / `retiring`）と、区画の `Workspace`（`active` / `closing` / `closed`）の組と、`AccessPolicy` の所有者の指定（`designated` / `vacant`） |
| F-24 | 設置 | 設置の状態（`InstallationState`）。`Installation`、復旧の記録 `Recovery`、実行中の CMS が扱うデータの版から求める |
| F-25 | 未対処の失敗 | `SubmissionNotification` と `WebhookDelivery` の `FailureHandling` |

## イベント・定期ジョブとフローの対応

記録するユースケース、消費者、周期、対象は [../domains/index.md](../domains/index.md) の表が定める。この表は、イベントと定期ジョブごとに、それを名前で指すフローを示す。

| ドメインイベント | フロー |
| --- | --- |
| `content.publication_changed` | F-01、F-05、F-19、F-25 |
| `webhook.delivery_requested` | F-05 |
| `form.submission_accepted` | F-04 |
| `form.notification_retry_requested` | F-04 |
| `media.registered` | F-06 |
| `media.derivative_regeneration_requested` | F-06、F-07 |
| `schema.applied` | F-01、F-06 |
| `schema.apply_requested` | F-01、F-02、F-03、F-13、F-17、F-19、F-20、F-21 |

| 定期ジョブ | フロー |
| --- | --- |
| 表示アプリへの通知の再試行 | F-05 |
| 履歴の版の整理 | F-17、F-20 |
| ごみ箱の完全削除 | F-17、F-19、F-20 |
| 回答の自動整理 | F-04、F-18、F-21、F-25 |
| 止まった通知の検出 | F-04 |
| 編集中の記録の掃除 | F-16 |
| 止まった反映の検出 | F-01、F-02、F-03 |
| 止まった派生形式の生成のやり直し | F-06 |
| メディアの実体の消去 | F-07 |
| 所属の索引の修復 | F-22 |
| 利用終了の完了 | F-07、F-12、F-16、F-22、F-23 |
| 期限切れの記録の掃除 | F-09、F-10、F-11、F-12、F-14、F-15 |

## 非同期の処理

### F-01 スキーマの反映

- **対象**: 受け付けた反映の一回を表す `SchemaApplyJob`（`mode` が `commit`。`preparing` / `prepared` / `succeeded` / `failed`）と、`WorkspaceSchema` の `state`（`idle` / `applying`）、未解消の失敗した反映（`failedApply`。`null`、または失敗した反映のジョブの ID）。どちらもワークスペースの区画にある。型と不変条件は [../domains/schema.md](../domains/schema.md) の「WorkspaceSchema」「SchemaApplyJob」が定める。ジョブの経路は、ジョブの状態で書き、`WorkspaceSchema` の `state` を括弧で添える。`failedApply` の経路は、別の表に書く。`WorkspaceSchema` は、ワークスペースの区画を作る UnitOfWork が作る（F-23、F-24）
- **トリガー**: 「反映を要求する（`requestSchemaApply`）」（Schema）、「直接反映を要求する（`requestDirectSchemaApply`）」（Schema）。ジョブを作る要求は [../domains/schema.md](../domains/schema.md) の「SchemaApplyJob」が定める

ジョブの経路。準備の区切りと段階の移り方は [../domains/schema.md](../domains/schema.md) の「Preparation」、消費者がジョブの状態から呼ぶユースケースは [../usecases/schema.md](../usecases/schema.md) の「イベントの消費者と定期ジョブ」が定める。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （ジョブなし。`idle`）→ `preparing`（`applying`） | 「反映を要求する（`requestSchemaApply`）」（Schema）、「直接反映を要求する（`requestDirectSchemaApply`）」（Schema）。同じ UnitOfWork で `schema.apply_requested` を記録する |
| `preparing` → `preparing`（準備の一区切り。`applying` のまま） | 「反映・事前確認を準備する（`prepareSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。続きを始める `schema.apply_requested` を同じ UnitOfWork で記録する。変換の区切りは、準備領域を書く（F-03） |
| `preparing` → `prepared`（`applying` のまま） | 「反映・事前確認を準備する（`prepareSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。準備を終える `SchemaApplyJob.complete` |
| `preparing` → `failed`（`blocked`）（`idle`） | 「反映・事前確認を準備する（`prepareSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。準備を終える `SchemaApplyJob.complete` |
| `preparing` → `failed`（`capacityExceeded`）（`idle`） | 「反映・事前確認を失敗させる（`failSchemaApply`）」（Schema）（`schema.apply_requested` の消費者） |
| `prepared` → `succeeded`（`idle`） | 「反映を確定する（`commitSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。準備済みにした消費が続けて呼ぶ場合と、届いた時点で準備済みのジョブの消費。同じ UnitOfWork で `schema.applied` と、公開内容が変わる場合の `content.publication_changed` を記録する |
| `prepared` → `failed`（`commitRejected` / `capacityExceeded`）（`idle`） | 「反映・事前確認を失敗させる（`failSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。確定の UnitOfWork が、[../domains/schema.md](../domains/schema.md) の「確定できない場合の扱い」が失敗させる理由で確定しなかった場合 |
| `preparing` / `prepared` → `failed`（`interrupted`）（`idle`） | 「止まった反映・事前確認を検出して失敗させる（`failStalledSchemaApplies`）」（Schema）（定期ジョブ「止まった反映の検出」） |
| `preparing` / `prepared` → `failed`（`interrupted`）（`idle`） | 「利用を再開する（`resumeInstallation`）」（Installation）。Schema の「利用再開前の処理」。復旧点で進行中だったジョブ（F-24） |

`failedApply` の経路。`WorkspaceSchema` の `state` と独立に移る。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| `null`、または前に失敗したジョブ → 失敗したジョブ | ジョブの経路で、ジョブを `failed`（`idle`）にする主体。ジョブを `failed` にする UnitOfWork で移る |
| 失敗したジョブ → `null` | 「反映を確定する（`commitSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。次の反映を `succeeded` にする UnitOfWork で移る |
| 失敗したジョブ → `null` | 「反映を要求する（`requestSchemaApply`）」（Schema）、「直接反映を要求する（`requestDirectSchemaApply`）」（Schema）。差分の無い要求。ジョブを作らず、`state` を変えない |

- **参照**: 適用中の反映は、「適用状況を確認する（`getMigrationStatus`）」（Schema）、「反映の適用中かを確かめる（`getSchemaApplyState`）」（Schema）、「受け付けた反映・事前確認の進行と結果を取得する（`getSchemaApplyJob`）」（Schema）が読む。終端のジョブは、「反映の結果を一覧する（`listSchemaApplyResults`）」（Schema）と「受け付けた反映・事前確認の進行と結果を取得する（`getSchemaApplyJob`）」（Schema）が読む。「状態を確認する（`checkInstallationStatus`）」（Installation）は Schema の「状態確認の項目」を、「利用を再開する（`resumeInstallation`）」（Installation）は、確認に、Schema の「利用再開前の処理」の項目を示す
- **終端**
  - 正常: `succeeded`（`idle`）。確定の UnitOfWork は、準備領域の切り替え（F-03）、コンテンツと履歴の版（F-17、F-19）、メディアの使用箇所（F-20）、`FormSettings`（F-21）、トークンの範囲（F-13）を、ともに確定する。確定する書き込みは [../usecases/schema.md](../usecases/schema.md) の「反映を確定する」が定める
  - 異常: `failed`（`idle`）。理由は `ApplyFailure`。スキーマの版は進まず、準備領域は残らない（F-03）
  - 終端のジョブは変わらず、区画の消去まで残る（F-23）
  - `failedApply`: `null`。失敗したジョブを持つ値から出る遷移は、`failedApply` の経路の表が持つ
- **失敗時の扱い**
  - 終了の書き込みが確定しなかった場合: ジョブを `failed` にする UnitOfWork（[../usecases/schema.md](../usecases/schema.md) の「終了の書き込み」）が確定しなければ、ジョブは進行中（`applying`）のまま、準備領域も残る。消費者が行う書き込みは、`schema.apply_requested` の再配送が行い直す
  - 準備の UnitOfWork が確定しなかった場合: 確定済みの区切りと `preparing`（`applying`）のジョブが残る。同じイベントの再配送が、その区切りから進める
  - 確定の UnitOfWork が確定しなかった場合: [../domains/schema.md](../domains/schema.md) の「確定できない場合の扱い」の表で反映を失敗させない行では、`prepared`（`applying`）のジョブと準備領域が残り、イベントの再配送が確定を行う。失敗させる行では、経路の `failed` へ進む
  - 進まない反映: イベントが届かない、隔離された、再配送が確定に至らないジョブは、進行中（`applying`）のまま残る。定期ジョブ「止まった反映の検出」が `interrupted` で失敗させる。対象は [../domains/schema.md](../domains/schema.md) の「SchemaApplyJob」の「中断」、手続きは [../usecases/schema.md](../usecases/schema.md) の「進行中のジョブの中断」が定める。検出の UnitOfWork が確定しなかったジョブは、進行中のまま残り、次の起動が対象にする
  - 並行した受け付けと再送: 確定しなかった側は何も残さない。応答を受け取れなかった要求は、要求した側が同じ `SchemaApplyJobId` で送り直す。規則は [../domains/schema.md](../domains/schema.md) の「受け付けた要求と再送」が定める
  - 差分の無い要求の保存が確定しなかった場合: `failedApply` が残る。送り直した要求が外す
  - 適用中に確定しない書き込み: `applying` の間に確定しない書き込みは、[../domains/schema.md](../domains/schema.md) の「WriteScope とスキーマ」が定める。確定しなかった書き込みの側に残る状態は、F-17、F-19、F-20 が持つ
  - 復旧と利用の再開: 復旧点で進行中だったジョブは、設置の状態が `operating` になるまで、進行中（`applying`）のまま残る（F-24）。経路の「利用を再開する（`resumeInstallation`）」（Installation）が失敗させる。その実行で確定しなかったジョブは、進行中のまま残件数に残り、設置は `awaiting_resume` のまま残る。実行し直した再開が、残りを失敗させる
  - 利用終了の途中のワークスペース: 区画が `closing` になった後、進行中のジョブは進行中（`applying`）のまま残る。出る遷移は、区画の消去（F-23）と、復旧の後の利用再開前の処理（F-24）。区画の消去がジョブを消す手順は、[../domains/schema.md](../domains/schema.md) の「利用終了の消去」が定める
  - 確定で記録したイベント: `schema.applied` の消費は F-06、`content.publication_changed` の消費は F-05 が持つ

### F-02 適用の事前確認

- **対象**: 受け付けた事前確認の一回を表す `SchemaApplyJob`（`mode` が `dryRun`。`preparing` / `checked` / `failed`）。反映（F-01）と同じ集約で、同じ準備（`Preparation`）を行う。ワークスペースの区画にある。`WorkspaceSchema` を変えない
- **トリガー**: 「適用の事前確認を要求する（`requestSchemaDryRun`）」（Schema）。ジョブを作る要求は [../domains/schema.md](../domains/schema.md) の「SchemaApplyJob」が定める

経路。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （ジョブなし）→ `preparing` | 「適用の事前確認を要求する（`requestSchemaDryRun`）」（Schema）。同じ UnitOfWork で `schema.apply_requested` を記録する |
| `preparing` → `preparing`（準備の一区切り） | 「反映・事前確認を準備する（`prepareSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。F-01 の準備の一区切りと同じ |
| `preparing` → `checked` | 「反映・事前確認を準備する（`prepareSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。準備を終える `SchemaApplyJob.complete` |
| `preparing` → `failed`（`schemaChanged`） | 「反映・事前確認を準備する（`prepareSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。`SchemaApplyJob.failIfSchemaChanged` が失敗させたジョブ |
| `preparing` → `failed`（`capacityExceeded`） | 「反映・事前確認を失敗させる（`failSchemaApply`）」（Schema）（`schema.apply_requested` の消費者） |
| `preparing` → `failed`（`interrupted`） | 「止まった反映・事前確認を検出して失敗させる（`failStalledSchemaApplies`）」（Schema）（定期ジョブ「止まった反映の検出」） |
| `preparing` → `failed`（`interrupted`） | 「利用を再開する（`resumeInstallation`）」（Installation）。Schema の「利用再開前の処理」。復旧点で進行中だったジョブ（F-24） |

- **参照**: 「受け付けた反映・事前確認の進行と結果を取得する（`getSchemaApplyJob`）」（Schema）が読む。「利用を再開する（`resumeInstallation`）」（Installation）は、確認に、Schema の「利用再開前の処理」の項目を示す
- **終端**
  - 正常: `checked`
  - 異常: `failed`。理由は `DryRunFailure`
  - どの終端でも、準備領域は残らない（F-03）。終端のジョブは変わらず、区画の消去まで残る（F-23）
- **失敗時の扱い**
  - 準備の UnitOfWork が確定しなかった場合、進まないジョブ、並行した受け付けと再送、復旧と利用の再開は、F-01 と同じ。残る状態は、`preparing` のジョブと準備領域
  - 終了の書き込みが確定しなかった場合: ジョブを `checked` または `failed` にする UnitOfWork（[../usecases/schema.md](../usecases/schema.md) の「終了の書き込み」）が確定しなければ、ジョブは `preparing` のまま、準備領域も残る。`schema.apply_requested` の再配送が行い直す
  - スキーマの版が進んだ後に消費が届かないジョブ: `preparing` のまま残る。定期ジョブ「止まった反映の検出」が `interrupted` で失敗させる
  - 利用終了の途中のワークスペース: 区画が `closing` になった後、ジョブは `preparing` のまま残る。出る遷移は、区画の消去（F-23）と、復旧の後の利用再開前の処理（F-24）。区画の消去がジョブを消す手順は、[../domains/schema.md](../domains/schema.md) の「利用終了の消去」が定める

### F-03 準備領域

- **対象**: 進行中の `SchemaApplyJob`（反映と事前確認）ごとの準備領域。Content の `ContentStagingArea` の準備領域と、Media の `MediaUsageRepository` の準備領域の二つ。どちらもワークスペースの区画にある。二つの識別子は [../usecases/schema.md](../usecases/schema.md) の「準備領域の識別子」が定める。二つは、同じ UnitOfWork で書き、切り替え、破棄するので、同じ遷移をたどる
- **トリガー**: 「反映・事前確認を準備する（`prepareSchemaApply`）」（Schema）の、変換の最初の区切りの書き込み

経路。書き込み、切り替え、破棄に使うポートは [../usecases/schema.md](../usecases/schema.md) の「ほかのドメインとの対応」、契約は [../domains/content.md](../domains/content.md) の「ContentStagingArea」と [../domains/media.md](../domains/media.md) の「MediaUsageRepository」が定める。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （なし）→ 準備領域あり | 「反映・事前確認を準備する（`prepareSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。変換の最初の区切り。ジョブの進行の保存と同じ UnitOfWork |
| 準備領域あり → 準備領域あり（区切りの追加） | 「反映・事前確認を準備する（`prepareSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。変換の区切り |
| 準備領域あり →（なし）（切り替え） | 「反映を確定する（`commitSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。ジョブを `succeeded` にする UnitOfWork（F-01） |
| 準備領域あり →（なし）（破棄） | 「反映・事前確認を準備する（`prepareSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。反映の `blocked`、事前確認の `checked` と `schemaChanged` で、ジョブを終端にする UnitOfWork |
| 準備領域あり →（なし）（破棄） | 「反映・事前確認を失敗させる（`failSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。ジョブを `failed` にする UnitOfWork |
| 準備領域あり →（なし）（破棄） | 「止まった反映・事前確認を検出して失敗させる（`failStalledSchemaApplies`）」（Schema）（定期ジョブ「止まった反映の検出」）。ジョブを `failed` にする UnitOfWork |
| 準備領域あり →（なし）（破棄） | 「利用を再開する（`resumeInstallation`）」（Installation）。Schema の「利用再開前の処理」。ジョブを `failed` にする UnitOfWork（F-24） |

- **参照**: 準備領域を読むのは、同じジョブの「反映・事前確認を準備する（`prepareSchemaApply`）」（Schema）
- **終端**
  - 正常: 反映の確定での切り替え
  - 正常: 事前確認の完了（`checked`）での破棄
  - 異常: 反映または事前確認の失敗での破棄
  - 準備領域とジョブの状態の対応は、[../domains/schema.md](../domains/schema.md) の「SchemaApplyJob」の不変条件が定める
- **失敗時の扱い**
  - 切り替えまたは破棄の UnitOfWork が確定しなかった場合: 準備領域は、進行中のジョブとともに残る。出る遷移は、ジョブの回復の主体が行う切り替えと破棄（F-01、F-02）
  - 準備の UnitOfWork が確定しなかった場合: 確定済みの区切りは残る。`schema.apply_requested` の再配送が、その区切りから書く
  - 進まないジョブの準備領域: ジョブとともに残る。定期ジョブ「止まった反映の検出」が、ジョブの失敗とともに破棄する（F-01、F-02）
  - 復旧と利用の再開: 復旧点で進行中だったジョブの準備領域は、復旧の後も残る。経路の「利用を再開する（`resumeInstallation`）」（Installation）が破棄する。破棄が確定しない間、設置は `awaiting_resume` のまま残る（F-24）
  - 利用終了の途中のワークスペース: 区画が `closing` になった後、進行中のジョブの準備領域は残る。出る遷移は、区画の消去（F-23）と、復旧の後の利用再開前の処理（F-24）。区画の消去は、Content と Media の「利用終了の消去」が行う。順は [../domains/index.md](../domains/index.md) の「手順を呼ぶ順」が定める

### F-04 フォームの通知の送信と手動再試行

- **対象**: `SubmissionNotification` の `state`（`waiting` / `sending` / `succeeded` / `failed`）と、通知の回（`attempt`。`initial`、または `retry` と `round`）
- **トリガー**: 「フォームの送信を受け付ける（`acceptSubmission`）」（Form）（`initial` の回）、「通知を手動で再試行する（`retryNotifications`）」（Form）（`retry` の回）

送信を行う主体は二つ。「回答の通知を送る（`sendSubmissionNotification`）」（Form）（`form.submission_accepted` と `form.notification_retry_requested` の消費者）と、「止まった通知を検出する（`detectStalledNotifications`）」（Form）（定期ジョブ「止まった通知の検出」）。経路の表は、この二つを「送信を行う二つの主体」と書く。どちらも、[../usecases/form.md](../usecases/form.md) の共通の「通知の送信」を行う。経路の表は、遷移を分ける判定を、[../domains/form.md](../domains/form.md) の「通知の送信」が定める結果の名前（`start` の `send` / `record`、`complete` の `recorded`）で指す。止まった通知は、同じ文書の「止まった通知」が定める。

経路。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （なし）→ `waiting`（`attempt` は `initial`） | 「フォームの送信を受け付ける（`acceptSubmission`）」（Form）。回答（F-18）とともに作り、`form.submission_accepted` を記録する |
| `waiting` → `sending` | 送信を行う二つの主体。`start` が `send` を返した回 |
| `waiting` → `failed`（`noRecipients`） | 送信を行う二つの主体。`start` が `record` を返した回 |
| `sending` → `succeeded` | 送信を行う二つの主体のうち、その回を `sending` にした処理。`recorded` が真の `complete` |
| `sending` → `failed`（`delivery`） | 送信を行う二つの主体のうち、その回を `sending` にした処理。`recorded` が真の `complete` |
| `sending` → `failed` | 送信を行う二つの主体。止まった `sending` の通知に、`start` が `record` を返した回（`abandon`） |
| `waiting` / `sending` → `failed` | 「利用を再開する（`resumeInstallation`）」（Installation）。Form の「利用再開前の処理」（F-24） |
| `failed` → `waiting`（`attempt` は次の `retry` の回） | 「通知を手動で再試行する（`retryNotifications`）」（Form）。再試行を受け付けた通知ごとに、`form.notification_retry_requested` を記録する |
| 任意の状態 →（削除） | 回答を削除する遷移の主体（F-18）。通知は、回答とともに消える |

`failed` の通知の対処の状態は F-25 が持つ。

- **参照**
  - 送信は、回答、送信の時点の `FormSettings`（`active` または `removed`。F-21）、送信の時点の `MailConnection` を読む
  - 「回答を一覧・検索する（`searchSubmissions`）」（Form）、「回答を取得する（`getSubmission`）」（Form）、「手動再試行の宛先を確認する（`previewRetryRecipients`）」（Form）、「送信接続設定の更新が影響するフォームを一覧する（`listFormsAffectedByMailConnection`）」（Form）が通知を読む
  - 「状態を確認する（`checkInstallationStatus`）」（Installation）、「ワークスペースの終了前の確認を示す（`previewWorkspaceClosure`）」（Installation）、「設置全体の終了前の確認を示す（`previewInstallationClosure`）」（Installation）は、Form が [../domains/form.md](../domains/form.md) の「状態確認の項目」「利用終了の消去」に公開する項目を、「利用を再開する（`resumeInstallation`）」（Installation）は、確認に、Form が「利用再開前の処理」に公開する項目を示す
- **終端**
  - 正常: `succeeded`。回答の削除まで残る
  - 異常: `failed`。出る遷移は、手動再試行と、回答の削除
  - 削除: 回答の削除（F-18）
- **失敗時の扱い**
  - 回を始めるイベントが届かない、または隔離された通知は、`waiting` のまま残る。定期ジョブ「止まった通知の検出」が進める（経路の、`waiting` から出る送信の行）
  - 送信の途中で止まった処理が残す状態と、そこから進める主体は、[../testcases/form/sendSubmissionNotification.md](../testcases/form/sendSubmissionNotification.md) と [../testcases/form/detectStalledNotifications.md](../testcases/form/detectStalledNotifications.md) が定める。`sending` のまま残った通知は、再配送されたイベントの消費者か、定期ジョブ「止まった通知の検出」が `failed` にする（経路の、止まった `sending` の通知の行）
  - 定期ジョブの一回の起動で扱わなかった通知と、処理が失敗した通知は、変わらずに残り、次の起動が扱う
  - 手動再試行: 受け付けなかった通知と、確定しなかった要求の通知は、`failed` のまま残る。回復の主体はメンバーで、「手動再試行の宛先を確認する（`previewRetryRecipients`）」（Form）で確認し直して要求する。受け付ける条件と結果は、[../domains/form.md](../domains/form.md) の「手動再試行」が定める
  - 利用できないワークスペース: `waiting` と `sending` の通知は、そのまま残る（[../domains/index.md](../domains/index.md) の「利用できるワークスペース」）。利用終了の途中のワークスペースの `waiting` と `sending` から出る遷移は、区画の消去（F-23）、復旧の後の利用再開前の処理（F-24）、定期ジョブ「回答の自動整理」による回答の削除（F-18）の三つ
  - 復旧の後: 「利用を再開する（`resumeInstallation`）」（Installation）の実行で `failed` にならなかった通知は、`waiting` / `sending` のまま、Form の残件数に残る。残件数が残る間、設置は `awaiting_resume` のまま残り、実行し直した再開が残りを処理する（F-24）。一回の実行が扱う範囲と残件数は、[../domains/form.md](../domains/form.md) の「利用再開前の処理」が定める。`failed` にした通知から出る遷移は、手動再試行と、回答の削除

### F-05 表示アプリへの通知の送信と再試行

- **対象**: `ChangeNotification`（公開内容を変えた一つの操作の通知）と、通知と宛先の組ごとの送信 `WebhookDelivery`（`pending` / `succeeded` / `failed`）。`pending` は、次の試行（`attempt`。`first`、`automatic`、`manual`）を持つ。送信の要求 `webhook.delivery_requested`
- **トリガー**: `content.publication_changed` の配送。公開の変更の記録（`PublicationRecord`）の追記と、このイベントの記録は、F-19 と F-01 が持つ

通知を作る主体は、「公開内容の変更の通知を作る（`createChangeNotification`）」（Webhook）（`content.publication_changed` の消費者）。

送信を行う主体は二つ。「送信を求められた通知を送る（`sendRequestedDelivery`）」（Webhook）（`webhook.delivery_requested` の消費者）、「表示アプリへの通知を再試行する（`retryDueDeliveries`）」（Webhook）（定期ジョブ「表示アプリへの通知の再試行」）。経路の表は、この二つを「送信を行う二つの主体」と書く。どちらも、[../usecases/webhook.md](../usecases/webhook.md) の共通の「送信の試行」を、一つの送信ごとに行う。主体ごとの対象の送信は、その節の、主体と対象の表が定める。試行の結果から次の状態を決める規則は、[../domains/webhook.md](../domains/webhook.md) の「WebhookDelivery」が定める。

経路。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| 未消費の `content.publication_changed` →（通知なし） | 「公開内容の変更の通知を作る（`createChangeNotification`）」（Webhook）（`content.publication_changed` の消費者）。通知を作らずに終える消費。当たる場合は、[../domains/webhook.md](../domains/webhook.md) の「ChangeNotification」の `decideCreation`（「通知を作る条件」）が定める |
| 未消費の `content.publication_changed` → 通知と、宛先ごとの `pending`（`first`）と、送信ごとの未消費の `webhook.delivery_requested` | 「公開内容の変更の通知を作る（`createChangeNotification`）」（Webhook）（`content.publication_changed` の消費者）。通知を作る消費。宛先は、[../domains/webhook.md](../domains/webhook.md) の「ChangeNotification」の `decideCreation`（「通知を作る条件」）が定める |
| （通知なし）→ 通知と、宛先ごとの `pending`（`first`）と、送信ごとの未消費の `webhook.delivery_requested` | 「公開内容の変更の通知を作る（`createChangeNotification`）」（Webhook）（`content.publication_changed` の消費者）。通知を作らずに終えた消費の後に、同じイベントが再び届いた消費。当たる場合は、[../domains/webhook.md](../domains/webhook.md) の「ChangeNotification」の `decideCreation`（「通知を作る条件」）が定める |
| `pending` → `succeeded` | 送信を行う二つの主体 |
| `pending`（`first` / `automatic`）→ `pending`（次の `automatic`） | 送信を行う二つの主体 |
| `pending` → `failed` | 送信を行う二つの主体 |
| `failed` → `pending`（`manual`）と、未消費の `webhook.delivery_requested` | 「送信失敗の通知を再試行する（`requestDeliveryRetry`）」（Webhook）。再試行を始めた組ごとに、送信の要求を記録する |

`failed` の送信の対処の状態は F-25 が持つ。

- **参照**
  - 送信は、通知と、送る時点の `WebhookEndpoints` を読む
  - 「通知と宛先の組を一覧する（`listDeliveries`）」（Webhook）と「一つの通知の送信結果を確かめる（`getNotificationDeliveries`）」（Webhook）が、通知と送信を読む
  - 「ワークスペースの終了前の確認を示す（`previewWorkspaceClosure`）」（Installation）と「設置全体の終了前の確認を示す（`previewInstallationClosure`）」（Installation）は、Webhook が [../domains/webhook.md](../domains/webhook.md) の「利用終了の消去」に公開する項目を示す。未対処の送信失敗の項目は F-25 が持つ
- **終端**
  - 正常: `succeeded`
  - 正常: 通知を作らずに終えた消費。出る遷移は、経路の、同じイベントが再び届いた消費
  - 異常: `failed`。出る遷移は、手動再試行
  - 通知と送信は、ワークスペースの利用終了の区画の消去まで残る（F-23）
- **失敗時の扱い**
  - 通知の作成の中断、再配送、並行した消費が残す状態は、[../testcases/webhook/createChangeNotification.md](../testcases/webhook/createChangeNotification.md) が定める。確定の前に止まった作成は何も残さず、イベントは配送の対象に残り、再配送された消費者が作る
  - `content.publication_changed` が届かない、または隔離されている間、通知と送信は作られない。隔離されたイベントは、ホスティング運営者（運用手順）の配送し直しで、通知の作成が始まる
  - `webhook.delivery_requested` が届かない、または隔離された `pending` は、定期ジョブ「表示アプリへの通知の再試行」が送る。定期ジョブが対象にする時期は、[../domains/webhook.md](../domains/webhook.md) の「WebhookDelivery」の `isOverdue` が定める
  - 送信の途中で止まった処理が残す状態は、[../testcases/webhook/sendRequestedDelivery.md](../testcases/webhook/sendRequestedDelivery.md) と [../testcases/webhook/retryDueDeliveries.md](../testcases/webhook/retryDueDeliveries.md) が定める。`pending` のまま残った送信は、再配送された消費者か、定期ジョブの次の起動が送る
  - 手動再試行: 受け付けなかった要求と、確定しなかった要求の組は、`failed` のまま残る。回復の主体はメンバーで、「通知と宛先の組を一覧する（`listDeliveries`）」（Webhook）で確認し直して要求する。受け付ける条件と結果は、[../domains/webhook.md](../domains/webhook.md) の「WebhookDelivery」と「DeliveryBatch」が定める
  - 利用できないワークスペース: 通知は作られず、`pending` は、そのまま残る（[../domains/index.md](../domains/index.md) の「利用できるワークスペース」）。利用終了の途中のワークスペースの `pending` から出る遷移は、区画の消去（F-23）だけ。Webhook は「利用再開前の処理」を持たない
  - 復旧の後: 「利用を再開する（`resumeInstallation`）」（Installation）は、通知と送信を変えない。復旧点で未消費だった `content.publication_changed` は、再開の後に配送されて通知を作る。設置が稼働中でない間の配送で隔離されたイベントは、ホスティング運営者（運用手順）の配送し直しで、通知の作成が始まる。復旧点で `pending` だった送信は、再開の後に、送信を行う二つの主体が送る（F-24）。復旧の後の通し番号は、[../domains/webhook.md](../domains/webhook.md) の「通し番号と復旧」が定める

### F-06 派生形式の生成

- **対象**: `MediaDerivatives`（画像一件）の、派生形式ごとの `DerivativeState`（`generating` / `ready` / `failed`）。`generating` と `failed` は生成の対象（定義と世代 `DerivativeGeneration`）を持つ。型と、状態を変える判定は、[../domains/media.md](../domains/media.md) の「MediaDerivatives」（「定義の適用」「生成の引き受け」「生成の完了」「止まった生成のやり直し」）が定める
- **トリガー**: 「メディアを登録する（`registerMedia`）」（Media）が記録する `media.registered`、「反映を確定する（`commitSchemaApply`）」（Schema）が記録する `schema.applied`、「失敗した派生形式の生成をやり直す（`retryDerivative`）」（Media）、定期ジョブ「止まった派生形式の生成のやり直し」。派生形式ごとの生成は、これらが記録する `media.derivative_regeneration_requested` の消費者が行う。イベントを作る規則は、[../domains/media.md](../domains/media.md) の「生成を起こすイベント」が定める

経路。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （派生形式の記録なし）→ `states` が空の記録 | 「メディアを登録する（`registerMedia`）」（Media）。画像の `Media` を作る UnitOfWork（F-20）。同じ UnitOfWork で `media.registered` を記録する |
| （派生形式なし）/ `ready` / `failed` / `generating` → `generating`（新しい世代） | 「派生形式の定義を登録済みの画像に適用する（`applyDerivativeDefinitions`）」（Media）（`schema.applied` と `media.registered` の消費者）。追加・変更した定義。同じ UnitOfWork で `media.derivative_regeneration_requested` を記録する。指さなくなった実体の位置は F-07 |
| `generating` / `ready` / `failed` →（派生形式なし） | 「派生形式の定義を登録済みの画像に適用する（`applyDerivativeDefinitions`）」（Media）（`schema.applied` と `media.registered` の消費者）。取り除いた定義。指さなくなった実体の位置は F-07 |
| `generating` → `ready` | 「派生形式一つを生成する（`generateDerivative`）」（Media）（`media.derivative_regeneration_requested` の消費者）。実体を保管した生成（F-07） |
| `failed` → `ready` | 「派生形式一つを生成する（`generateDerivative`）」（Media）（`media.derivative_regeneration_requested` の消費者）。生成の間に失敗になった、同じ生成の対象の結果が、後から届いた生成 |
| `generating` → `failed` | 「派生形式一つを生成する（`generateDerivative`）」（Media）（`media.derivative_regeneration_requested` の消費者）。失敗した生成。生成の対象を引き継ぐ |
| `failed` → `generating` | 「失敗した派生形式の生成をやり直す（`retryDerivative`）」（Media）。同じ生成の対象を持つ。同じ UnitOfWork で `media.derivative_regeneration_requested` を記録する |
| `generating` → `generating`（始め直し） | 「止まった派生形式の生成をやり直す（`restartStalledDerivatives`）」（Media）（定期ジョブ「止まった派生形式の生成のやり直し」）。生成の対象を変えない。同じ UnitOfWork で `media.derivative_regeneration_requested` を記録する |
| `generating` → `failed`（`stalled`） | 「止まった派生形式の生成をやり直す（`restartStalledDerivatives`）」（Media）（定期ジョブ「止まった派生形式の生成のやり直し」）。始め直しの回数が上限に達した生成。上限は [../domains/media.md](../domains/media.md) の「止まった生成のやり直し」が定める |
| `generating` / `ready` / `failed` →（派生形式の記録なし） | F-20 のメディアの削除の主体。`Media` とともに派生形式の記録が消える |

- **参照**: 「メディアを一覧・検索する（`listMedia`）」（Media）、「メディアを取得する（`getMedia`）」（Media）、「メディアを配信する（`deliverMedia`）」（Media）
- **終端**
  - 正常: `ready`。定義が変わるまで留まる
  - 異常: `failed`。「失敗した派生形式の生成をやり直す（`retryDerivative`）」（Media）が `generating` に戻す
  - 放棄: 定義の取り除き、メディアの削除
- **失敗時の扱い**
  - 進まない生成: イベントが届かない、隔離された、イベントなしで `generating` になった（「生成を起こすイベント」）、結果を記録する前に処理が止まった派生形式は、`generating` のまま残る。定期ジョブ「止まった派生形式の生成のやり直し」が、始め直すか、`failed`（`stalled`）にする。名簿で開設済みのワークスペースの派生形式は、この経路で `generating` から出る
  - 「生成の引き受け」が `null` を返すイベント: 派生形式の状態は変わらない。その処理が保管した実体は F-07
  - 結果の UnitOfWork が確定しなかった生成と、実体の保管の後に止まった生成: 派生形式は `generating` のまま残る。確定しなかった生成は、イベントの再配送が進める。止まった生成は、定期ジョブの始め直しが進める。場面ごとの残る状態と続きは、[../testcases/media/generateDerivative.md](../testcases/media/generateDerivative.md) が定める。保管した実体は F-07
  - `schema.applied` の消費者が途中で失敗・中断した場合: 適用を終えた派生形式の記録は新しい定義の状態で、残りは前の定義の状態で残る。再配送が、残りに適用する（[../testcases/media/applyDerivativeDefinitions.md](../testcases/media/applyDerivativeDefinitions.md)）
  - `media.registered` の消費が `SCHEMA_CHANGED` で確定しない場合: 消費はエラーを投げて終え、その画像の記録は `states` が空のまま残る。確定した版の `schema.applied` の消費者か、`media.registered` の再配送が、その画像に定義を適用する（[../testcases/media/applyDerivativeDefinitions.md](../testcases/media/applyDerivativeDefinitions.md)）
  - 定期ジョブが確定しなかった一件と、中断した起動の残りは、止まったまま残り、次の起動が対象にする
  - 利用終了の途中のワークスペース: 区画が `closing` になった後、派生形式の記録はその時点の状態で残る。出る遷移は、区画の消去（F-23）。保管した実体は F-07
  - 復旧の後: 派生形式の記録は復旧点の状態に戻る。復旧点で `generating` だった派生形式は、利用の再開の後に、復旧点にあったイベントの配送、または定期ジョブの始め直しが進める（F-24）。復旧の後に割り当てる世代は、[../domains/media.md](../domains/media.md) の「復旧の置き換え」が定める

## ファイル・外部リソース

### F-07 メディアの実体

- **対象**: `MediaStorage` が保管する、元のファイルと派生形式の実体（`MediaObjectRef`）。保管の予約（`StorageReservation`）と、消去待ちの記録（`ObjectRemoval`）。二つの記録はワークスペースの区画に、実体は区画の外にある。実体と、それを指す記録（`Media`、`MediaDerivatives`、保管の予約、消去待ち）の規則と、記録によらずに実体を消去する二つの場合（利用終了、復旧）は、[../domains/media.md](../domains/media.md) の「実体と記録の整合」が定める
- **トリガー**: 「メディアを登録する（`registerMedia`）」（Media）（元のファイル）、「派生形式一つを生成する（`generateDerivative`）」（Media）（派生形式。F-06）
- **有効期限**: 保管の予約が回収できる予約になる時期は、[../domains/media.md](../domains/media.md) の「StorageReservation」が定める。消去待ちの記録は期限を持たない

経路（保管の予約）。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （なし）→ 予約あり | 「メディアを登録する（`registerMedia`）」（Media）、「派生形式一つを生成する（`generateDerivative`）」（Media）（`media.derivative_regeneration_requested` の消費者）。実体の保管より前 |
| 予約あり → 回収できる予約 | 主体は無い。`StorageReservation.isCollectable` が、現在時刻から求める |
| 予約あり / 回収できる予約 →（なし）（結果の確定） | 予約を確定した同じ処理。登録の結果は `Media` の作成（F-20）、生成の結果は派生形式の記録の保存（F-06）。同じ UnitOfWork |
| 予約あり / 回収できる予約 →（なし）（後始末） | 予約を確定した同じ処理。結果を確定できなかった処理が、終える前に行う |
| 回収できる予約 →（なし）（回収） | 「回収できる保管の予約を回収する（`collectStorageReservations`）」（Media）（定期ジョブ「メディアの実体の消去」）。予約の確定の後に止まった処理の予約 |

経路（消去待ちの記録）。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （なし）→ 消去待ち | 経路（保管の予約）の、予約を取り除く三つの確定（結果の確定、後始末、回収）の主体。記録する実体の位置は [../domains/media.md](../domains/media.md) の「StorageReservation」が定める。予約を取り除く確定と同じ UnitOfWork |
| （なし）→ 消去待ち | F-20 のメディアの削除の主体。削除したメディアの、元のファイルとすべての派生形式。`Media` の削除と同じ UnitOfWork |
| （なし）→ 消去待ち | F-06 の、派生形式の記録を保存する主体（定義の適用、生成の完了）。記録する実体の位置は [../domains/media.md](../domains/media.md) の「ObjectRemoval」が定める。派生形式の記録の保存と同じ UnitOfWork |
| 消去待ち →（なし） | 消去待ちを記録した上の主体（その UnitOfWork の確定の後）、「消去待ちの実体を消去する（`purgeRemovedObjects`）」（Media）（定期ジョブ「メディアの実体の消去」）。実体の消去の後に、記録を削除する（[../usecases/media.md](../usecases/media.md) の「消去待ちの消去」） |

経路（実体）。元のファイルと派生形式は、同じ遷移を取る。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （なし）→ 保管済み | 「メディアを登録する（`registerMedia`）」（Media）（元のファイル）、「派生形式一つを生成する（`generateDerivative`）」（Media）（`media.derivative_regeneration_requested` の消費者）（派生形式）。予約の確定の後 |
| 保管済み →（なし） | 経路（消去待ちの記録）の、消去待ちの記録を無くす主体。消去待ちが指す実体 |
| 保管済み →（なし） | 「ワークスペースの利用終了を完了させる（`completeWorkspaceClosures`）」（Installation）（定期ジョブ「利用終了の完了」）。区画の消去（F-23）が `Media` と派生形式の記録を削除した、そのワークスペースの実体。ワークスペースを名簿から外す前 |
| 保管済み →（なし）、（なし）→ 保管済み | 「バックアップから設置全体を復旧する（`restoreInstallation`）」（Installation）。ワークスペースの実体を、バックアップの内容に置き換える。置き換えの手順と、設置の停止の前に始まった消去の呼び出しに対する前提は、[../domains/media.md](../domains/media.md) の「復旧の置き換え」が定める（F-08、F-24） |

- **参照**: 「メディアを配信する（`deliverMedia`）」（Media）（F-20）、「派生形式一つを生成する（`generateDerivative`）」（Media）（元の画像。F-06）。「ワークスペースの利用終了を完了させる（`completeWorkspaceClosures`）」（Installation）は、区画に残る保管の予約と消去待ちの件数を読む（F-23）
- **終端**
  - 正常: 実体の消去と、消去待ちの記録の削除。保管の予約は、結果の確定、後始末、回収のいずれか一つで無くなる
  - 正常: 利用終了を完了したワークスペースの実体は、名簿から外す前の消去で無くなる（F-23）
  - 確定していない試みの内容は実体でない。終端は、[../domains/media.md](../domains/media.md) の「MediaStorage」が定める
  - 配信の入口の先と、CMS の外が保存・複製した内容は、このフローの対象でない
- **失敗時の扱い**
  - 途中で止まった・失敗した場面ごとの、残る状態と続きは、[../testcases/media/registerMedia.md](../testcases/media/registerMedia.md)、[../testcases/media/generateDerivative.md](../testcases/media/generateDerivative.md)、[../testcases/media/collectStorageReservations.md](../testcases/media/collectStorageReservations.md)、[../testcases/media/purgeRemovedObjects.md](../testcases/media/purgeRemovedObjects.md) が、実体と記録の整合の規則は [../domains/media.md](../domains/media.md) の「実体と記録の整合」が定める。回復の主体は三つ。結果を確定できなかった処理は、自分の後始末で予約を取り除く。予約の確定の後に止まった処理の予約は、定期ジョブ「メディアの実体の消去」が回収する。消去に失敗した、または記録の削除の前に止まった消去待ちは、同じ定期ジョブが消去からやり直す
  - 結果を確定できなかった登録は、メディアを登録しない。残った予約と実体は、後始末、または回収と消去待ちの消去で無くなる。要求した側が、登録をやり直す。回収が先に予約を取り除いた派生形式の生成は、再配送が生成する（F-06）
  - 定期ジョブ「メディアの実体の消去」が確定しなかった予約、消去に失敗した記録、中断した起動の残りは、そのまま残り、次の起動が扱う。設置の状態が `operating` でない間は、記録は残る（F-24）
  - 利用終了の途中のワークスペース: 区画が `closing` になった後に残る保管の予約と消去待ちの記録は、区画の消去（F-23）では消えない（[../domains/media.md](../domains/media.md) の「利用終了の消去」）。出る遷移は、処理の後始末と、定期ジョブ「メディアの実体の消去」（経路の表）
  - 利用終了の完了: 「ワークスペースの利用終了を完了させる（`completeWorkspaceClosures`）」（Installation）が実体の消去を始められる条件は、[../domains/media.md](../domains/media.md) の「MediaClosure」が定める。条件を満たさない間、名簿の項目は終了中のまま残り、定期ジョブ「利用終了の完了」の次の起動が確かめ直す。実体の消去が途中で失敗・中断した場合も、次の起動が行い直す（F-23）
  - 復旧: 途中で止まった復旧は、一部のワークスペースの実体だけを置き換えたメディアの保管先を残し得る。設置は復旧中のまま残り、始め直した復旧が、置き換えを初めから行う（F-24）。復旧点にあった消去待ちの記録は、利用の再開の後に、定期ジョブ「メディアの実体の消去」が無くす。保管の予約の扱いは [../domains/media.md](../domains/media.md) の「復旧の置き換え」が定める

### F-08 バックアップ

- **対象**: `BackupStore` が保管先に持つバックアップ（設置のデータの、区画ごとの一つの時点の写しと、その記録 `BackupRecord`）。取得中と、取得済み（`integrity` が `verified` / `failed`）を取る。設置のデータの外にあり、UnitOfWork に属さない。バックアップの対象、取得と復旧の契約は、[../domains/installation.md](../domains/installation.md) の「BackupStore」、記録の型は「BackupRecord」が定める
- **トリガー**: 「バックアップを取得する（`takeBackup`）」（Installation）。定期的な取得は、ホスティング運営者が、自分の環境でこの運用コマンドを実行して行う
- **有効期限**: 保持・削除期限（`retainUntil`）。期限と、運用コマンドを実行する間隔は、[../domains/installation.md](../domains/installation.md) の「BackupPolicy」が定める

経路。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （なし）→ 取得中 | 「バックアップを取得する（`takeBackup`）」（Installation）。写しの取得を始める |
| 取得中 → 取得済み（`integrity` が `verified`） | 「バックアップを取得する（`takeBackup`）」（Installation）。整合性の確認が通った取得 |
| 取得中 → 取得済み（`integrity` が `failed`） | 「バックアップを取得する（`takeBackup`）」（Installation）。整合性の確認が通らなかった取得。復旧に使えないバックアップとして残す |
| 取得中 →（削除） | 次の、または並行して後から始まった「バックアップを取得する（`takeBackup`）」（Installation）。取得済みに至らなかった取得が残した写しを消す |
| 取得済み → 保持・削除期限を過ぎた取得済み | 主体は無い。`BackupRecord.isPastRetention` が現在時刻から求める |
| 取得済み（期限の前後を問わない）→（削除） | ホスティング運営者（運用手順）。保管先でバックアップを削除する |

- **参照**
  - 「バックアップを一覧する（`listBackups`）」（Installation）。受け付ける設置は、[../domains/installation.md](../domains/installation.md) の「設置の状態」が定める
  - 「復旧前の確認を示す（`previewRestore`）」（Installation）、「バックアップから設置全体を復旧する（`restoreInstallation`）」（Installation）。復旧の記録 `Recovery` は、復旧に使うバックアップを `backupId` で指す（F-24）
  - 「状態を確認する（`checkInstallationStatus`）」（Installation）。判定は [../domains/installation.md](../domains/installation.md) の「StatusAssessment」が定める
  - 「ワークスペースの終了前の確認を示す（`previewWorkspaceClosure`）」（Installation）、「設置全体の終了前の確認を示す（`previewInstallationClosure`）」（Installation）（F-23）
- **終端**
  - 正常: ホスティング運営者（運用手順）の、保管先での削除。取得済みのバックアップのライフサイクルは、[../domains/installation.md](../domains/installation.md) の「BackupRecord」が定める
  - 異常: 取得中のまま残った写し。出る遷移は、次の取得による削除と、ホスティング運営者が設置全体の利用終了の手順で行う、保管先ごとの消去（F-24）
- **失敗時の扱い**
  - 取得済みになる前に失敗・中断・中止した取得は、取得中の写しを残す。回復の主体は、ホスティング運営者が実行し直す取得で、やり直しの単位は取得の全体。止まった取得が、その後の取得・復旧・更新を妨げるかどうかは、[../domains/installation.md](../domains/installation.md) の「取得・置き換え・更新の排他」が定める
  - 取得と、復旧の置き換え・設置の更新が重なったときの、バックアップと取得中の写しの結果は、[../domains/installation.md](../domains/installation.md) の「取得・置き換え・更新の排他」が定める。復旧と更新の側に残る状態と、そこから出る遷移は F-24 が持つ

## 有効期限つきのもの

### F-09 セッション

- **対象**: `Session`（`pending_mfa` / `active`）。設置の区画にある。型とライフサイクルは [../domains/identity.md](../domains/identity.md) の「Session」が定める。「保持期間を過ぎた」は、状態として保存しない
- **トリガー**: 「パスワードでログインする（`logInWithPassword`）」（Identity）、「招待を受諾する（`acceptInvitation`）」（Workspace）（アカウントを作った参加）、「新しいパスワードを設定してアクセスを回復する（`completeAccessRecovery`）」（Identity）
- **有効期限**: 期限と起点は [../domains/identity.md](../domains/identity.md) の「Session」、期限の境界と掃除の対象は [../domains/common.md](../domains/common.md) の「期限つきの一度きりの値」が定める

経路。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （なし）→ `active` | 「パスワードでログインする（`logInWithPassword`）」（Identity）。`Authentication.loginWithPassword` が `authenticated` を返したログイン |
| （なし）→ `pending_mfa` | 「パスワードでログインする（`logInWithPassword`）」（Identity）。`Authentication.loginWithPassword` が `mfa_required` を返したログイン |
| （なし）→ `active` | 「招待を受諾する（`acceptInvitation`）」（Workspace）。アカウントを作った参加の、確かめ直しの段。所属の索引の項目を `joined` にする書き込みと同じ UnitOfWork（F-11、F-22） |
| （なし）→ `active` | 「新しいパスワードを設定してアクセスを回復する（`completeAccessRecovery`）」（Identity）。セッションを始める回復。回復依頼の `used` と同じ UnitOfWork（F-10、F-22）。始めるかどうかは [../domains/identity.md](../domains/identity.md) の「LoginAttemptPolicy」が定める |
| `pending_mfa` → `active` | 「確認コードを入力してログインを完了する（`completeMfaLogin`）」（Identity） |
| `pending_mfa` / `active` → 期限切れ | 主体は無い。`Session.standing` が現在時刻から求める |
| `pending_mfa` / `active` / 期限切れ →（削除） | 「ログアウトする（`logOut`）」（Identity） |
| 期限切れ → 保持期間を過ぎた | 主体は無い。`OneTimeValueRetention.isForgotten` が、`Session.lapsedAt` と現在時刻から求める |
| 保持期間を過ぎた →（削除） | 「期限切れのセッションと回復依頼を掃除する（`purgeExpiredSessionsAndRecoveryRequests`）」（Identity）（定期ジョブ「期限切れの記録の掃除」） |

- **参照**: 「セッションからアカウントを確かめる（`resolveSessionAccount`）」（Identity）
- **終端**
  - 正常: 削除（ログアウト、掃除）
  - 期限切れのセッションは、保持期間を過ぎた後に掃除が削除するまで残る。期限切れから出る遷移は、ログアウトと、保持期間を過ぎること
- **失敗時の扱い**
  - 完了しなかった `pending_mfa` のセッションは、`pending_mfa` のまま残り、期限で終わる。ログインの後に多要素認証が `enabled` でなくなったアカウント（F-22）の `pending_mfa` のセッションも同じ。利用者はログインからやり直す
  - 確かめ直しが確定しなかった、アカウントを作った受諾は、セッションを残さない。参加は成立していて、その人は、設定したパスワードでログインする。確かめ直しより前で止まった受諾が残す状態は F-11 と F-22 が持つ
  - セッションを始めなかった回復と、確定しなかった回復は、セッションを残さない。前者の回復の主体は、新しいパスワードでログインする本人。確定しなかった回復に残る状態は F-10 が持つ
  - 途中で止まった掃除: [../domains/common.md](../domains/common.md) の「掃除」に従う
  - 復旧点より後に作ったセッションは、復旧で無くなり、利用者はログインをやり直す（F-24）

### F-10 回復依頼

- **対象**: `RecoveryRequest`（`pending` / `used` / `revoked`。`channel` は `email` と `operator`）。設置の区画にある。型とライフサイクルは [../domains/identity.md](../domains/identity.md) の「RecoveryRequest」が定める。「保持期間を過ぎた」は、状態として保存しない
- **トリガー**: 「アクセス回復を依頼する（`requestAccessRecovery`）」（Identity）、「管理者復旧の回復の値を発行する（`issueOperatorRecoveryToken`）」（Identity）
- **有効期限**: 期限と起点は [../domains/identity.md](../domains/identity.md) の「RecoveryRequest」、期限の境界、保持期間、掃除の対象は [../domains/common.md](../domains/common.md) の「期限つきの一度きりの値」が定める

経路。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （なし）→ `pending`（`email`） | 「アクセス回復を依頼する（`requestAccessRecovery`）」（Identity） |
| （なし）→ `pending`（`operator`） | 「管理者復旧の回復の値を発行する（`issueOperatorRecoveryToken`）」（Identity） |
| `pending` → `revoked` | 「アクセス回復を依頼する（`requestAccessRecovery`）」（Identity）、「管理者復旧の回復の値を発行する（`issueOperatorRecoveryToken`）」（Identity）。同じアカウントの、同じ `channel` の新しい回復依頼の発行。新しい `pending` の作成と同じ UnitOfWork |
| `pending` → `used` | 「新しいパスワードを設定してアクセスを回復する（`completeAccessRecovery`）」（Identity）。`Account` の更新と、セッションを始める回復ではセッションの作成と同じ UnitOfWork（F-09、F-22） |
| `pending` → 期限切れ | 主体は無い。`RecoveryRequest.standing` が現在時刻から求める |
| 期限切れ / `used` / `revoked` → 保持期間を過ぎた | 主体は無い。`OneTimeValueRetention.isForgotten` が、`RecoveryRequest.lapsedAt` と現在時刻から求める |
| 保持期間を過ぎた →（削除） | 「期限切れのセッションと回復依頼を掃除する（`purgeExpiredSessionsAndRecoveryRequests`）」（Identity）（定期ジョブ「期限切れの記録の掃除」） |

- **参照**: 「回復の値の状態を確かめる（`checkRecoveryToken`）」（Identity）
- **終端**
  - 正常: `used`。保持期間を過ぎた後に、掃除が削除する
  - 異常: `revoked`、期限切れ。保持期間を過ぎた後に、掃除が削除する。利用者は依頼からやり直す
- **失敗時の扱い**
  - メールの送信に失敗した回復依頼は `pending` のまま残る。出る遷移は、依頼し直しによる失効と、期限切れ
  - 同じアカウントへの、同じ `channel` の並行した発行が残す状態は、[../testcases/identity/requestAccessRecovery.md](../testcases/identity/requestAccessRecovery.md) と [../testcases/identity/issueOperatorRecoveryToken.md](../testcases/identity/issueOperatorRecoveryToken.md) が定める。確定しなかった管理者復旧の回復の主体は、発行し直す運営者
  - 完了が確定しなかった回復は、アカウントも回復依頼も変えない。回復依頼は `pending` のまま残り、期限まで使える
  - 多要素認証の解除を選んだ回復がアカウントに及ぼす遷移は F-22、確認コード待ちのセッションに残る状態は F-09 が持つ
  - 途中で止まった掃除: [../domains/common.md](../domains/common.md) の「掃除」に従う
  - 復旧点より後に作った回復依頼は、復旧で無くなり、利用者は依頼をやり直す（F-24）

### F-11 招待

- **対象**: `Invitation`（`pending` / `accepted` / `revoked`。`grant` は `member` と `initial_owner`）。ワークスペースの区画にある。型とライフサイクルは [../domains/workspace.md](../domains/workspace.md) の「Invitation」が定める。「保持期間を過ぎた」は、状態として保存しない
- **トリガー**: 「メンバーを招待する（`inviteMember`）」（Workspace）、「ワークスペースを作成し、初期所有者の招待を発行する（`createWorkspace`）」（Installation）、「初期所有者の招待を発行し直す（`reissueInitialOwnerInvitation`）」（Installation）
- **有効期限**: 期限と起点は [../domains/workspace.md](../domains/workspace.md) の「Invitation」、期限の境界、保持期間、掃除の対象は [../domains/common.md](../domains/common.md) の「期限つきの一度きりの値」が定める

経路。名簿と `Workspace` の遷移は F-23 が持つ。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （なし）→ `pending`（`member`） | 「メンバーを招待する（`inviteMember`）」（Workspace） |
| （なし）→ `pending`（`initial_owner`） | 「ワークスペースを作成し、初期所有者の招待を発行する（`createWorkspace`）」（Installation）の招待の発行の段と、「初期所有者の招待を発行し直す（`reissueInitialOwnerInvitation`）」（Installation）。名簿で開設済みの、所有者がいない（`vacant`）ワークスペース |
| `pending` → `accepted` | 「招待を受諾する（`acceptInvitation`）」（Workspace）。参加の後段。`Member` の作成、または既存の `Member` の所有者への指定と同じ UnitOfWork。この確定で参加は成立する。初期所有者の招待の受諾は、同じ UnitOfWork で、所有者の指定を `vacant` から `designated` にする（F-23）。前段と確かめ直しが設置の区画に書くアカウント、所属の索引の項目、セッションの遷移は、F-22 と F-09 が持つ |
| `pending`（`member`）→ `revoked` | 「招待を取り消す（`revokeInvitation`）」（Workspace） |
| `pending`（`member`）→ `revoked` | 「メンバーを招待する（`inviteMember`）」（Workspace）。同じ宛先への新しい招待による置き換え。新しい `pending` の作成と同じ UnitOfWork |
| `pending`（`initial_owner`）→ `revoked` | 上の、初期所有者の招待を発行する二つの主体。前の `pending` の初期所有者の招待の失効。新しい `pending` の作成と同じ UnitOfWork |
| `pending` → 期限切れ | 主体は無い。`Invitation.standing` が現在時刻から求める |
| 期限切れ / `accepted` / `revoked` → 保持期間を過ぎた | 主体は無い。`OneTimeValueRetention.isForgotten` が、`Invitation.lapsedAt` と現在時刻から求める |
| 保持期間を過ぎた →（削除） | 「期限切れの招待を掃除する（`purgeExpiredInvitations`）」（Workspace）（定期ジョブ「期限切れの記録の掃除」） |

招待を受諾できるかどうかと、受諾の結果は、[../domains/workspace.md](../domains/workspace.md) の「InvitationProcess」、置き換えと、メンバーの招待・初期所有者の招待の独立は「Invitation」、取り下げは「MembershipRepair」の「取り下げ」が定める。

- **参照**: 「招待の内容と状態を確かめる（`inspectInvitation`）」（Workspace）、「未使用の招待を一覧する（`listPendingInvitations`）」（Workspace）。「状態を確認する（`checkInstallationStatus`）」（Installation）は、所有者がいないワークスペースに、初期所有者の招待の状態を添えて示す（F-23）
- **終端**
  - 正常: `accepted`。保持期間を過ぎた後に、掃除が削除する
  - 異常: `revoked`、期限切れ。保持期間を過ぎた後に、掃除が削除する
- **失敗時の扱い**
  - メールの送信に失敗した招待は `pending` のまま残る。出る遷移は、招待し直し（メンバーの招待）または発行し直し（初期所有者の招待）による失効と、期限切れ
  - 名簿を開設済みにした後、招待を届ける前に止まった作成は、招待を残さないか、届けていない `pending` の初期所有者の招待を残す。届けていない招待から出る遷移は、同じワークスペースの ID での作成の実行し直しと、発行し直しによる失効と、期限切れ。ワークスペースに残る状態は F-23 が持つ
  - 受諾できない `pending` の招待（[../domains/workspace.md](../domains/workspace.md) の「InvitationProcess」の `availability`）は、`pending` のまま残り、期限で終わる
  - 受諾が確定しなかった招待は、`pending` のまま残る。事前の確認で拒否された受諾、前段または後段が確定しなかった受諾、前段の後に止まった受諾、取り下げられた参加の後段が当たる。回復の主体は、招待された本人で、招待を受諾できる間、招待の URL から受諾をやり直す。止まった場所ごとに残る状態は、[../testcases/workspace/acceptInvitation.md](../testcases/workspace/acceptInvitation.md) が定める。前段が残す所属の索引の項目とアカウントは F-22 が持つ
  - 後段の後に止まった受諾と、確かめ直しが確定しなかった受諾は、成立していて、招待は `accepted`。所属の索引の項目に残る状態は F-22 が持つ
  - 取り下げで止まった参加の招待は、`pending` のまま残る（F-22）
  - 利用終了の途中のワークスペース: 区画が `closing` になった後、招待はその時点の状態で残る。出る遷移は、掃除と、区画の消去（F-23）。確認と受諾の結果は、[../domains/index.md](../domains/index.md) の「利用できるワークスペース」が定める
  - 途中で止まった掃除: [../domains/common.md](../domains/common.md) の「掃除」に従う
  - 復旧の後、招待は、区画の写しの状態で残る。利用再開前の処理が削除したメンバーの `accepted` の招待は `accepted` のまま残る。利用再開前の処理が所有者の指定を `vacant` にしたワークスペースと、開設済みにしたワークスペースは、所有者がいないワークスペースとして、F-23 の終端の異常の対処の対象になる（F-22、F-23、F-24）

### F-12 トークン

- **対象**: `Token`（`active` / `revoked`。アプリに紐付くトークン `AppToken` と、メンバーに紐付くトークン `MemberToken`）。ワークスペースの区画にある。初期構成で宣言したトークンは、設置の区画の索引 `DeclaredTokenIndex` に項目を持つ。「索引だけ」は、索引に項目があり、区画にトークンが無い状態。「保持期間を過ぎた」は、状態として保存しない
- **トリガー**: 「トークンを手動発行する（`issueToken`）」（Token）、「設置を起動する（`bootInstallation`）」（Installation）（初期構成の宣言）、「スクリプトの認証の結果を受け取る（`redeemDeviceAuthorization`）」（Token）、「引き換えの値をトークンに引き換える（`redeemPreviewExchange`）」（Token）
- **有効期限**: 発行の経路ごとの期限は [../domains/token.md](../domains/token.md) の「Token」と「TokenLifetimePolicy」が定める。保持期間の起点は `Token.lapsedAt` が、保持期間は [../domains/common.md](../domains/common.md) の「期限つきの一度きりの値」が定める

経路。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （なし）→ `active` | 「トークンを手動発行する（`issueToken`）」（Token）。アプリに紐付く（F-13） |
| （なし）→ 索引だけ | 「設置を起動する（`bootInstallation`）」（Installation）。初期構成の読み込みの最初の段（F-24） |
| 索引だけ → `active` | 「設置を起動する（`bootInstallation`）」（Installation）。初期構成の読み込みのワークスペースごとの段。アプリに紐付く（F-23、F-13） |
| （なし）→ `active` | 「スクリプトの認証の結果を受け取る（`redeemDeviceAuthorization`）」（Token）。メンバーに紐付く。受け取りの後段（F-15） |
| （なし）→ `active` | 「引き換えの値をトークンに引き換える（`redeemPreviewExchange`）」（Token）。メンバーに紐付く。引き換えを `redeemed` にする UnitOfWork（F-14） |
| `active` → `active`（利用元の限定と利用量の制限の変更） | 「ブラウザ用トークンの利用元の限定と利用量の制限を変える（`changeBrowserTokenRestrictions`）」（Token）。変えるトークンは [../domains/token.md](../domains/token.md) の「Token」の `changeRestrictions` が定める |
| `active` → 期限切れ | 主体は無い。`Token.standing` が現在時刻から求める |
| `active` / 期限切れ → `revoked` | 「トークンを失効させる（`revokeToken`）」（Token） |
| `active` → `revoked` | 「自分に紐付くトークンを失効させる（`revokeOwnToken`）」（Token）。失効させるトークンは [../domains/token.md](../domains/token.md) の「自分に紐付くトークンの失効」が定める |
| `revoked` / 期限切れ → 保持期間を過ぎた | 主体は無い。`OneTimeValueRetention.isForgotten` が、`Token.lapsedAt` と現在時刻から求める |
| 保持期間を過ぎた →（削除） | 「期限切れのプレビューの引き換え・トークンを削除する（`deleteExpiredTokenRecords`）」（Token）（定期ジョブ「期限切れの記録の掃除」）。宣言したトークンは、索引だけになる |
| 索引だけ →（索引からの除外） | 「ワークスペースの利用終了を完了させる（`completeWorkspaceClosures`）」（Installation）（定期ジョブ「利用終了の完了」）。最後の段（F-23）。区画のトークンは、この段より前に、区画の消去が削除している |

宣言したトークンの生成の段は [../domains/token.md](../domains/token.md) の「初期構成の宣言」、利用終了の消去は同じファイルの「利用終了の消去」が定める。

- **参照**: 「示されたトークンを確かめる（`authenticateToken`）」（Token）、「トークンの要求を操作ごとに認可する（`authorizeTokenOperation`）」（Token）、「トークンを一覧する（`listTokens`）」（Token）、「ブラウザ用トークンの値を確かめる（`revealBrowserTokenValue`）」（Token）、「メンバーの利用の解除の影響を確かめる（`previewMemberRemoval`）」（Workspace）
- **終端**
  - 正常: 削除。`revoked` から出る遷移は、保持期間を過ぎた後の削除だけ
  - 正常: 期限なしで `active` のトークンは、失効させるまで残る
  - 正常: 索引からの除外。索引の項目が残る間は [../domains/token.md](../domains/token.md) の「DeclaredTokenIndex」が定める
  - 異常: 最初の段の後に止まった読み込みが残す「索引だけ」。出る遷移は、次の起動の、経路の索引だけ → `active`（F-24）
- **失敗時の扱い**
  - 「索引だけ」の値を示した要求の結果は、[../domains/index.md](../domains/index.md) の「利用できるワークスペース」と [../domains/token.md](../domains/token.md) の「TokenAuthentication」が定める
  - 止まった読み込み: 登録済みの索引の項目と発行済みのトークンは残る。回復の主体は、読み込みの続きを進める「設置を起動する（`bootInstallation`）」（Installation）（F-24）
  - 発行が確定しなかった、または応答が届かなかった受け取りと引き換えに残る状態は、F-15 と F-14 が持つ。応答が届かなかったトークンは `active` のまま残り、出る遷移は、経路の失効と期限切れ
  - メンバーの利用の解除（F-22）、権限の変更、多要素認証の必須化の後、トークンは前の状態のまま残る（[../domains/token.md](../domains/token.md) の「Token」の不変条件）。出る遷移は、経路の失効と期限切れ。そのトークンを示した要求の結果は、同じファイルの「TokenAuthorization」が定める
  - 利用終了の途中のワークスペース: 区画が `closing` になった後、トークンはその時点の状態で残る。出る遷移は、経路の削除と、区画の消去（F-23）。トークンを示した要求の結果は、[../domains/index.md](../domains/index.md) の「利用できるワークスペース」が定める
  - 途中で止まった掃除: [../domains/common.md](../domains/common.md) の「掃除」に従う
  - `PreviewExchange` と `DeviceAuthorization` が持つ `tokenId` は、削除したトークンを指したまま残る。発行が確定しなかった依頼は F-15 が持つ
  - 復旧点以降のトークンの発行と失効は、復旧で元に戻り、利用の再開の後も戻ったまま残る。Workspace の利用再開前の処理が削除したメンバーに紐付くトークンと、そのメンバーが発行したトークンは、解除されたメンバーのトークンと同じ経路をたどる（F-24）

### F-13 トークンの範囲の紐付け

- **対象**: アプリに紐付く `Token` の範囲（`TokenScope` の `selection`）の要素（`ScopeTarget`。紐付け待ちの `pending`＝キー / 紐付け済みの `bound`＝ID）。範囲がワークスペース全体のトークンは、このフローの遷移を持たない
- **トリガー**: 「設置を起動する（`bootInstallation`）」（Installation）（初期構成で、キーの集合で範囲を宣言したトークン）、「トークンを手動発行する（`issueToken`）」（Token）

経路。紐付けと除外の規則は [../domains/common.md](../domains/common.md) の「キーで宣言した設定の紐付け」、範囲を変える振る舞いは [../domains/token.md](../domains/token.md) の「範囲の更新」、手動発行のキーを ID にする規則は同じファイルの「TokenScope」の `resolveKeys` が定める。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （なし）→ `pending`（キー） | 「設置を起動する（`bootInstallation`）」（Installation）。初期構成の読み込みのワークスペースごとの段で、トークンの発行とともに作る（F-12） |
| （なし）→ `bound`（ID） | 「トークンを手動発行する（`issueToken`）」（Token）。トークンの発行とともに作る（F-12） |
| `pending`（キー）→ `bound`（ID） | 「反映を確定する（`commitSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。そのキーのモデル・フォームを作成した反映。確定の UnitOfWork（F-01） |
| `bound`（ID）→（範囲から除外） | 「反映を確定する（`commitSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。そのモデル・フォームを削除した反映。確定の UnitOfWork（F-01） |

- **参照**: 「紐付けを待つトークンの範囲を一覧する（`listPendingTokenScopes`）」（Token）、「紐付けを待つ運用設定を一覧する（`listPendingBindings`）」（Installation）、「トークンの要求を操作ごとに認可する（`authorizeTokenOperation`）」（Token）
- **終端**
  - 正常: `bound`。トークンが削除されるまで残る（F-12）
  - 正常: 範囲からの除外。除いた要素は戻らない。要素を除いたトークンは [../domains/token.md](../domains/token.md) の「範囲の更新」に従う（F-12）
  - 異常: `pending` のまま残る。出る遷移は、そのキーの対象を作成する反映の確定と、トークンの削除（F-12）
- **失敗時の扱い**
  - 失効したトークンの `pending` の要素は、紐付かないまま、トークンとともに削除される（F-12）
  - 確定しなかった反映と、失敗で終わった反映は、どの要素も変えない。`pending` の要素は、次に確定する反映を待つ。確定しなかった後の扱いは [../domains/schema.md](../domains/schema.md) の「確定できない場合の扱い」が定める（F-01）
  - 反映の確定と並行した、範囲が集合の手動発行に残る状態は、[../testcases/token/issueToken.md](../testcases/token/issueToken.md) が定める。削除したモデル・フォームの要素を持って残ったトークンは、その要素を `bound` のまま持ち、トークンの削除まで残る（F-12）。確定しなかった発行は、トークンを残さない。回復の主体は発行者で、「トークンの範囲に指定できる対象を一覧する（`listScopeTargets`）」（Token）が示す対象から選び直して、発行し直す
  - 読み込みが途中で止まったワークスペースは、区画にトークンが無く、`pending` の要素も無い。読み込みの続きを進める起動が、トークンとともに作る（F-12）
  - 利用終了の途中のワークスペース: 区画が `closing` になった後、範囲の要素はその時点の状態で残る。出る遷移は、トークンの削除（F-12）と、区画の消去（F-23）

### F-14 プレビューの引き換えの値

- **対象**: `PreviewExchange`（`pending` / `redeemed`）。ワークスペースの区画にある。「保持期間を過ぎた」は、状態として保存しない
- **トリガー**: 「プレビューを開く（`openPreview`）」（Token）
- **有効期限**: 引き換えの値の有効期間は [../domains/token.md](../domains/token.md) の「TokenLifetimePolicy」が定める。保持期間の起点は `PreviewExchange.lapsedAt` が、期限の境界と保持期間は [../domains/common.md](../domains/common.md) の「期限つきの一度きりの値」が定める

経路。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （なし）→ `pending` | 「プレビューを開く（`openPreview`）」（Token） |
| `pending` → `redeemed` | 「引き換えの値をトークンに引き換える（`redeemPreviewExchange`）」（Token）。トークンの発行と同じ UnitOfWork（F-12） |
| `pending` → 期限切れ | 主体は無い。`PreviewExchange.standing` が現在時刻から求める |
| 期限切れ / `redeemed` → 保持期間を過ぎた | 主体は無い。`OneTimeValueRetention.isForgotten` が、`PreviewExchange.lapsedAt` と現在時刻から求める |
| 保持期間を過ぎた →（削除） | 「期限切れのプレビューの引き換え・トークンを削除する（`deleteExpiredTokenRecords`）」（Token）（定期ジョブ「期限切れの記録の掃除」） |

- **参照**: 引き換えを読むのは、「引き換えの値をトークンに引き換える（`redeemPreviewExchange`）」（Token）。引き換えの結果の種類は [../domains/token.md](../domains/token.md) の「PreviewExchangeProcess」が定める
- **終端**
  - 正常: `redeemed`。保持期間を過ぎた後に、定期ジョブが削除する。発行済みのトークンは、F-12 の経路で残る
  - 異常: 期限切れ。保持期間を過ぎた後に、定期ジョブが削除する
- **失敗時の扱い**
  - 拒否された引き換えと、確定しなかった引き換えは、引き換えを変えず、トークンを残さない。拒否の種類は [../domains/token.md](../domains/token.md) の「PreviewExchangeProcess」が定める。プレビューを開いたメンバーが認可されない引き換えは `pending` のまま残り、出る遷移は、経路の引き換えと期限切れ
  - 利用終了の途中のワークスペース: 区画が `closing` になった後、引き換えはその時点の状態で残る。出る遷移は、経路の削除と、区画の消去（F-23）。値の引き換えの結果は、[../domains/index.md](../domains/index.md) の「利用できるワークスペース」が定める
  - 応答が表示アプリに届かなかった引き換えは `redeemed` のまま残る。回復の主体はメンバーで、「プレビューを開く（`openPreview`）」（Token）が新しい引き換えを作る。発行済みのトークンは `active` のまま残る（F-12）
  - 途中で止まった掃除: [../domains/common.md](../domains/common.md) の「掃除」に従う
  - 復旧点より後に作った引き換えは、復旧で無くなる。復旧点にあった引き換えは、復旧の後も同じ状態で残り、このフローの経路をたどる（F-24）

### F-15 スクリプトの認証の依頼

- **対象**: `DeviceAuthorization`（`pending` / `approved` / `denied` / `redeemed`）。設置の区画にある。承認で選んだワークスペースを、`approval.workspaceId` で参照する。発行するトークンは、そのワークスペースの区画にある（F-12）。「保持期間を過ぎた」は、状態として保存しない
- **トリガー**: 「スクリプトの認証を始める（`startDeviceAuthorization`）」（Token）
- **有効期限**: 依頼の有効期間は [../domains/token.md](../domains/token.md) の「TokenLifetimePolicy」が定める。保持期間の起点は `DeviceAuthorization.lapsedAt` が、期限の境界と保持期間は [../domains/common.md](../domains/common.md) の「期限つきの一度きりの値」が定める

経路。受け取りは、区画をまたぐ手順（設置 → ワークスペース）で、段の順は [../domains/index.md](../domains/index.md) の「区画をまたぐ手順」のスクリプトの認証の受け取りが定める。拒否は二つを書き分ける。`denial` は、依頼が終端の `denied` になる拒否。`refusal` は、受け取りの拒否で、依頼は `approved` のまま残る。どちらの種類も [../domains/token.md](../domains/token.md) の「DeviceAuthorization」と「DeviceAuthorizationProcess」が定める。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （なし）→ `pending` | 「スクリプトの認証を始める（`startDeviceAuthorization`）」（Token） |
| `pending` → `approved` | 「スクリプトの認証を承認する（`approveDeviceAuthorization`）」（Token） |
| `pending` → `denied`（`denial` は `insufficient_permissions`） | 「スクリプトの認証を承認する（`approveDeviceAuthorization`）」（Token） |
| `pending` / 期限切れの `pending` → `denied`（`denial` は `denied_by_account`） | 「スクリプトの認証を承認しない（`denyDeviceAuthorization`）」（Token） |
| `approved` → `redeemed`（トークンが無い） | 「スクリプトの認証の結果を受け取る（`redeemDeviceAuthorization`）」（Token）。受け取りの前段。受け取れる依頼は [../domains/token.md](../domains/token.md) の「DeviceAuthorizationProcess」の `redeem` が定める |
| `redeemed`（トークンが無い）→ `redeemed`（トークンがある） | 「スクリプトの認証の結果を受け取る（`redeemDeviceAuthorization`）」（Token）。前段を確定した同じ受け取りの後段。`approval.workspaceId` のワークスペースの区画で、トークンを発行する（F-12）。依頼を書かない |
| `pending` / `approved` → 期限切れ | 主体は無い。`DeviceAuthorization.standing` が現在時刻から求める |
| 期限切れ / `denied` / `redeemed`（トークンの有無によらない）→ 保持期間を過ぎた | 主体は無い。`OneTimeValueRetention.isForgotten` が、`DeviceAuthorization.lapsedAt` と現在時刻から求める |
| 保持期間を過ぎた →（削除） | 「期限切れのスクリプトの認証の依頼を削除する（`deleteExpiredDeviceAuthorizations`）」（Token）（定期ジョブ「期限切れの記録の掃除」） |

- **参照**: 「スクリプトの認証の内容を確かめる（`getDeviceAuthorization`）」（Token）。依頼を変えない
- **終端**
  - 正常: `redeemed` で、トークンがある依頼
  - 異常: `redeemed` で、トークンが無い依頼。後段が確定しなかった受け取りと、前段の後に止まった受け取りが残す。依頼の `tokenId` は、存在しないトークンを指す
  - 異常: `denied`、期限切れ
  - どの終端の依頼も、出る遷移は、保持期間を過ぎた後の削除だけ。発行済みのトークンは、F-12 の経路で残る
- **失敗時の扱い**
  - 受け取りを繰り返す主体は、スクリプトの実行環境のコマンド。結果ごとの続け方は [../scenario/authentication.md](../scenario/authentication.md) の AUTH-06 が、結果は [../testcases/token/redeemDeviceAuthorization.md](../testcases/token/redeemDeviceAuthorization.md) が定める
  - 途中で止まった受け取りが残す状態は [../testcases/token/redeemDeviceAuthorization.md](../testcases/token/redeemDeviceAuthorization.md) が、以後の受け取りの結果は [../domains/token.md](../domains/token.md) の「DeviceAuthorizationProcess」が定める。`redeemed` でトークンが無い依頼と、応答が届かなかった依頼の回復の主体はスクリプトで、認証を初めからやり直す。応答が届かなかったトークンは `active` のまま残る（F-12）
  - 選べないワークスペースを示された依頼と、承認の認可が拒否した依頼は、`pending` のまま残る。出る遷移は、経路の承認、承認しない操作、期限切れ。`denied` になった依頼の回復の主体はスクリプトで、認証を初めからやり直す
  - `refusal` を受けた依頼は、`approved` のまま残る。出る遷移は、原因が解消した後の、経路の受け取りの前段と、期限切れ。`refusal` の原因は [../domains/token.md](../domains/token.md) の「DeviceAuthorizationProcess」が定める
  - 確定しなかった承認、承認しない操作、受け取りの前段は、依頼を変えない。前段が確定しなかった受け取りは、後段を行わない
  - 利用終了の途中のワークスペースと、名簿から外したワークスペース: そのワークスペースを指す依頼は、設置の区画に残る（F-23）。`approved` の依頼は `refusal` を受けて `approved` のまま残り、出る遷移は期限切れ。受け取りの結果は、[../domains/index.md](../domains/index.md) の「利用できるワークスペース」が定める
  - 復旧点の依頼は、復旧の後も同じ状態で残る。Workspace の利用再開前の処理が削除したメンバーが承認した依頼は、`refusal` を受けて `approved` のまま残り、期限で終わる（F-24）
  - 途中で止まった掃除: [../domains/common.md](../domains/common.md) の「掃除」に従う

### F-16 編集中の記録

- **対象**: `EditingPresence`（ワークスペース、コンテンツ、メンバーの組で一つ。`EditingPresenceStore` に置く。区画に結び付かない）
- **トリガー**: 「編集中であることを知らせる（`announceEditing`）」（Content）
- **有効期限**: `expiresAt`。求め方は [../domains/content.md](../domains/content.md) の「EditingPresence」が定める

経路。有効かどうかの判定は [../domains/content.md](../domains/content.md) の「EditingPresence」が定める。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （なし）→ 有効 | 「編集中であることを知らせる（`announceEditing`）」（Content） |
| 有効 → 有効（`expiresAt` を延ばす） | 「編集中であることを知らせる（`announceEditing`）」（Content） |
| 有効 → 失効 | 主体は無い。現在時刻と `expiresAt` から求める |
| 失効 → 有効 | 「編集中であることを知らせる（`announceEditing`）」（Content） |
| 有効 / 失効 →（削除） | 「編集を終えたことを知らせる（`endEditing`）」（Content） |
| 失効 →（削除） | 「編集中の記録を掃除する（`cleanUpEditingPresence`）」（Content）（定期ジョブ「編集中の記録の掃除」） |
| 有効 / 失効 →（削除） | 「ワークスペースの利用を終了する（`closeWorkspace`）」（Installation）、「ワークスペースの利用終了を完了させる（`completeWorkspaceClosures`）」（Installation）（定期ジョブ「利用終了の完了」）。Content の「利用終了の消去」が、そのワークスペースの記録を削除する（F-23） |

- **参照**: 「編集中のメンバーを確認する（`listEditingMembers`）」（Content）
- **終端**
  - 正常: 削除
- **失敗時の扱い**
  - 延ばされなくなった記録: 失効し、定期ジョブ「編集中の記録の掃除」が削除する
  - 途中で中断した掃除: 残りの失効した記録は、失効のまま残り、定期ジョブ「編集中の記録の掃除」の次の起動が削除する
  - 途中で中断した利用終了の消去: そのワークスペースの記録は残り、F-23 の区画の消去の続きが削除する。消去の後に書かれた記録は、失効し、定期ジョブ「編集中の記録の掃除」が削除する
  - 復旧と利用の再開: 記録は、復旧の前のまま残る（[../domains/content.md](../domains/content.md) の「EditingPresenceStore」）。残った記録は、失効し、利用の再開の後に、定期ジョブ「編集中の記録の掃除」が削除する（F-24）

### F-17 履歴の版とごみ箱の保持

- **対象**: `Revision`（保持中 / 保持期限切れ）と、ごみ箱の `Content` の保持（保持中 / 保持期限切れ）。保持の期限は `ContentRetentionPolicy`（ワークスペースに一つ。ワークスペースの区画を作る UnitOfWork が作る。F-23、F-24）から決まる。コンテンツの公開状態と置き場所の遷移は F-19 が持つ
- **トリガー**: 内容を置き換える・なくす操作（版の作成）、ごみ箱への移動（F-19）
- **有効期限**: 保持方針の期限を過ぎた対象の規則は、[../domains/content.md](../domains/content.md) の「ContentRetentionPolicy」が定める（履歴の版は `revisionCutoff`、ごみ箱のコンテンツは `trashCutoff`）

経路（履歴の版）。どの操作が、どの内容を版にするかは、[../domains/content.md](../domains/content.md) の「操作の種類」の表が定める。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （なし）→ 保持中 | 「下書きを保存する（`saveDraft`）」（Content）、「下書きを破棄する（`discardDraft`）」（Content）、「コンテンツを公開する（`publishContent`）」（Content）、「公開を取り消す（`unpublishContent`）」（Content）、「コンテンツをごみ箱へ移動する（`trashContent`）」（Content）、「値を書き換える（`rewriteValues`）」（Content）、「版を下書きとして復元する（`restoreRevisionAsDraft`）」（Content）。置き換えた、またはなくした内容がある操作（F-19） |
| 保持中 / 保持期限切れ → 同じ状態（変換） | 「反映を確定する（`commitSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。準備領域の切り替え（F-03） |
| 保持中 → 保持期限切れ | 主体は無い。現在時刻と保持方針から求める |
| 保持中 → 保持期限切れ、保持期限切れ → 保持中（日数の置き換え） | 「コンテンツの保持方針を変更する（`changeRetentionPolicy`）」（Content） |
| 保持中 / 保持期限切れ →（削除） | 「履歴の版を削除する（`deleteRevisions`）」（Content）。指定した版 |
| 保持中 / 保持期限切れ →（削除） | 「コンテンツを完全削除する（`purgeContent`）」（Content）。対象のコンテンツの版のすべてと、対象に加えた版（F-19） |
| 保持中 / 保持期限切れ →（削除） | 「履歴の版とともにメディアを削除する（`deleteMediaWithRevisions`）」（Content）。対象に加えた版（F-07、F-20） |
| 保持期限切れ →（削除） | 「期限を過ぎた履歴の版を削除する（`cleanUpExpiredRevisions`）」（Content）（定期ジョブ「履歴の版の整理」） |
| 保持中 / 保持期限切れ →（削除） | 「期限を過ぎたごみ箱のコンテンツを完全削除する（`purgeExpiredTrash`）」（Content）（定期ジョブ「ごみ箱の完全削除」）。完全削除するコンテンツの版のすべて |
| 保持中 / 保持期限切れ →（削除） | 「反映を確定する（`commitSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。モデルを削除した反映の、準備領域の切り替え（F-03、F-19） |

経路（ごみ箱の保持）。ごみ箱に入る遷移と、期限によらずごみ箱から出る遷移（復元、完全削除、モデルの削除）は F-19 が持つ。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （ごみ箱の外）→ ごみ箱（保持中） | F-19 のごみ箱への移動の主体 |
| ごみ箱（保持中）→ ごみ箱（保持期限切れ） | 主体は無い。現在時刻と保持方針から求める |
| ごみ箱（保持中）→ ごみ箱（保持期限切れ）、ごみ箱（保持期限切れ）→ ごみ箱（保持中）（日数の置き換え） | 「コンテンツの保持方針を変更する（`changeRetentionPolicy`）」（Content） |
| ごみ箱（保持期限切れ）→（完全削除） | 「期限を過ぎたごみ箱のコンテンツを完全削除する（`purgeExpiredTrash`）」（Content）（定期ジョブ「ごみ箱の完全削除」）。コンテンツの側の遷移は、この行が持つ（F-19） |

- **参照**: 「履歴を一覧する（`listRevisions`）」（Content）、「履歴の版を取得する（`getRevision`）」（Content）、「内容を比較する（`compareBodies`）」（Content）、「ごみ箱のコンテンツを一覧する（`listTrashedContents`）」（Content）、「コンテンツの保持方針を確認する（`getRetentionPolicy`）」（Content）
- **終端**
  - 正常: 版は削除。ごみ箱のコンテンツは、完全削除、または F-19 の遷移（復元、モデルの削除）
  - 異常: 完全削除を妨げる参照を持たれたごみ箱のコンテンツは、保持期限切れのままごみ箱に残る。残る状態から出る遷移は F-19 が持つ
- **失敗時の扱い**
  - スキーマの反映の適用中（F-01）: 版とごみ箱のコンテンツを削除する Content のユースケースは確定せず、版とごみ箱のコンテンツは保持の状態のまま残る。メンバーとアプリの削除は、反映が終わった後に、要求した側が送り直す。期限を過ぎた対象は、反映が終わった後の、二つの定期ジョブの起動が整理する。適用中に版とごみ箱のコンテンツを変換・削除する主体は、その反映の確定の切り替え（F-03）。規則は [../domains/schema.md](../domains/schema.md) の「WriteScope とスキーマ」が定める
  - 確定しなかった明示的な削除: どの版も削除されず、保持の状態のまま残る。要求した側が、確認を示し直して送り直す。残る状態は [../testcases/content/deleteRevisions.md](../testcases/content/deleteRevisions.md) が定める
  - 確定しなかった整理: 確定済みの削除は残り、残りの版とごみ箱のコンテンツは保持期限切れのまま残る。二つの定期ジョブの次の起動が整理する。残る状態は [../testcases/content/cleanUpExpiredRevisions.md](../testcases/content/cleanUpExpiredRevisions.md) と [../testcases/content/purgeExpiredTrash.md](../testcases/content/purgeExpiredTrash.md) が定める
  - 版とともに消えるメディアの使用箇所: 遷移は F-20、反映の確定で削除する版の使用箇所は F-03 が持つ
  - 利用終了の途中のワークスペース: 二つの定期ジョブの対象は、[../domains/index.md](../domains/index.md) の「定期ジョブ」の表が定める。利用終了の途中のワークスペースの版とごみ箱のコンテンツは、保持の状態のまま残り、F-23 の区画の消去が消す
  - 復旧と利用の再開: 版、ごみ箱のコンテンツ、保持方針は、復旧点の内容に戻る。期限を過ぎた対象は、利用の再開の後に、二つの定期ジョブが整理する（F-24）

### F-18 回答の保持

- **対象**: `Submission`。保持期限は `Submission.expiresAt`
- **トリガー**: 「フォームの送信を受け付ける（`acceptSubmission`）」（Form）
- **有効期限**: [../domains/form.md](../domains/form.md) の「RetentionPolicy」が定める

経路。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （なし）→ 保存済み | 「フォームの送信を受け付ける（`acceptSubmission`）」（Form）。受け付けた送信。通知（F-04）とともに作る |
| 保存済み → 保存済み（保持方針の置き換え） | 「保持方針を既存の回答へ適用する（`applyRetentionToSubmissions`）」（Form） |
| 保存済み → 保存済み（保持期限切れ） | 主体は無い。`Submission.expiresAt` と現在時刻から求める |
| 保存済み →（削除） | 「回答を削除する（`deleteSubmissions`）」（Form）。保持期限によらない |
| 保存済み（保持期限切れ）→（削除） | 「保持期限を過ぎた回答を自動整理する（`purgeExpiredSubmissions`）」（Form）（定期ジョブ「回答の自動整理」） |
| 保存済み（保持期限切れ）→（削除） | 「利用を再開する（`resumeInstallation`）」（Installation）。Form の「利用再開前の処理」（F-24） |

- **参照**: 「回答を一覧・検索する（`searchSubmissions`）」（Form）、「回答を取得する（`getSubmission`）」（Form）、「回答の対象のフォームを一覧する（`listSubmissionForms`）」（Form）、「保持方針の既存の回答への適用を確認する（`previewRetentionApplication`）」（Form）、通知の送信（F-04）。「利用を再開する（`resumeInstallation`）」（Installation）は、確認に、Form が [../domains/form.md](../domains/form.md) の「利用再開前の処理」に公開する項目を、「ワークスペースの終了前の確認を示す（`previewWorkspaceClosure`）」（Installation）は、同じ文書の「利用終了の消去」に公開する項目を示す
- **終端**: 削除。回答と、その通知は残らない（F-04、F-25）
- **失敗時の扱い**
  - 受け付けなかった送信と、確定しなかった送信は、回答も通知もイベントも残さない。確定しなかった送信は、アプリが送り直す。アプリへ返す結果は、[../usecases/app-api.md](../usecases/app-api.md) の「フォームの受付結果」が定める
  - 途中で止まった保持方針の適用は、確定した分の回答を置き換えたまま、残りの回答を元の保持方針のまま残す。止まった原因ごとの回復の主体は、[../testcases/form/applyRetentionToSubmissions.md](../testcases/form/applyRetentionToSubmissions.md) が定める
  - 確定しなかった「回答を削除する（`deleteSubmissions`）」（Form）は、どの回答も変えず、メンバーがやり直す
  - 途中で中断した自動整理は、確定した削除を残す。残りの保持期限切れの回答は、次の起動が削除する
  - 利用終了の途中のワークスペースの回答から出る遷移は、定期ジョブ「回答の自動整理」による削除、区画の消去（F-23）、復旧の後の利用再開前の処理（F-24）
  - 復旧の後: 「利用を再開する（`resumeInstallation`）」（Installation）が呼ぶ Form の「利用再開前の処理」で、削除が確定しなかった回答は、保存済み（保持期限切れ）のまま、Form の残件数に残る。残件数が残る間、設置は `awaiting_resume` のまま残り、実行し直した再開が残りを削除する（F-24）

## 集約のライフサイクル

### F-19 コンテンツ

- **対象**: `Content`（公開状態、下書きの有無、置き場所）と、公開内容を変える操作ごとの公開の変更の記録（`PublicationRecord`）。記録の型と通し番号は、[../domains/content.md](../domains/content.md) の「PublicationLedger」が定める
- **トリガー**: 「コンテンツを作成する（`createContent`）」（Content）

経路。状態と振る舞いの対応は [../domains/content.md](../domains/content.md) の「Content」の「ライフサイクル」、操作ごとに作る内容と版にする内容は同じ文書の「操作の種類」が定める。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （なし）→ 未公開 | 「コンテンツを作成する（`createContent`）」（Content） |
| 未公開 → 未公開（下書きの置き換え） | 「下書きを保存する（`saveDraft`）」（Content）、「版を下書きとして復元する（`restoreRevisionAsDraft`）」（Content）、「値を書き換える（`rewriteValues`）」（Content） |
| 未公開 → 公開中（下書きなし） | 「コンテンツを公開する（`publishContent`）」（Content） |
| 公開中（下書きなし）→ 公開中（下書きあり） | 「下書きを保存する（`saveDraft`）」（Content）、「版を下書きとして復元する（`restoreRevisionAsDraft`）」（Content） |
| 公開中（下書きあり）→ 公開中（下書きあり）（下書きの置き換え） | 「下書きを保存する（`saveDraft`）」（Content）、「版を下書きとして復元する（`restoreRevisionAsDraft`）」（Content）、「値を書き換える（`rewriteValues`）」（Content） |
| 公開中（下書きあり）→ 公開中（下書きなし） | 「コンテンツを公開する（`publishContent`）」（Content）、「下書きを破棄する（`discardDraft`）」（Content） |
| 公開中 → 公開中（公開内容の値の置き換え） | 「値を書き換える（`rewriteValues`）」（Content） |
| 公開中（下書きなし / 下書きあり）→ 未公開 | 「公開を取り消す（`unpublishContent`）」（Content） |
| 通常（未公開 / 公開中）→ ごみ箱（未公開） | 「コンテンツをごみ箱へ移動する（`trashContent`）」（Content）。ごみ箱の保持の期限は F-17 が持つ |
| ごみ箱（未公開）→ ごみ箱（未公開）（下書きの置き換え） | 「値を書き換える（`rewriteValues`）」（Content） |
| ごみ箱（未公開）→ 通常（未公開） | 「ごみ箱のコンテンツを復元する（`restoreTrashedContent`）」（Content） |
| 通常 / ごみ箱 →（完全削除） | 「コンテンツを完全削除する（`purgeContent`）」（Content）。履歴の版の遷移は F-17、メディアの使用箇所の遷移は F-20 が持つ |
| ごみ箱（未公開）→（完全削除） | F-17 の、保持期限切れのごみ箱のコンテンツの完全削除の主体 |
| 任意の状態 → 同じ状態（内容の変換） | 「反映を確定する（`commitSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。準備領域の切り替え（F-03） |
| 任意の状態 →（削除） | 「反映を確定する（`commitSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。モデルを削除した反映の、準備領域の切り替え（F-03）。履歴の版の遷移は F-17 が持つ |

経路（公開の変更の記録）。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （なし）→ 追記済み | 「コンテンツを公開する（`publishContent`）」（Content）、「公開を取り消す（`unpublishContent`）」（Content）、「コンテンツをごみ箱へ移動する（`trashContent`）」（Content）、「コンテンツを完全削除する（`purgeContent`）」（Content）、「値を書き換える（`rewriteValues`）」（Content）。公開内容が変わった操作 |
| （なし）→ 追記済み | 「反映を確定する（`commitSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。公開内容が変わる反映の、準備領域の切り替え（F-03） |

- **参照**
  - 「編集するコンテンツを取得する（`getContentForEditing`）」（Content）、「参照と一意な値の状態を確認する（`checkReferenceAndUniqueStatus`）」（Content）
  - 公開の変更の記録は、「公開内容の変更の通知を作る（`createChangeNotification`）」（Webhook）（`content.publication_changed` の消費者）が読む（F-05）
- **終端**
  - 正常: 完全削除、モデルの削除
  - 通常の管理対象のコンテンツは、公開状態と下書きの有無のどの状態にも、期限なく留まる。ごみ箱のコンテンツの期限は F-17 が持つ
  - 公開の変更の記録は、追記の後に変わらず、F-23 の区画の消去まで残る
- **公開内容を変える遷移に伴うもの**
  - 公開内容を変える遷移と、同じ UnitOfWork で確定するものは、[../domains/index.md](../domains/index.md) の「公開内容の整合性と通し番号」と、[../usecases/content.md](../usecases/content.md) の「公開内容の変更の確定」が定める
  - 公開の変更の記録の追記は、上の経路が持つ
  - 新たに公開するメディア（[../domains/content.md](../domains/content.md) の「新たに公開するメディア」の `NewlyPublishedMedia.of`）の公開範囲の遷移は F-20、置き換えた、またはなくした内容の履歴の版の遷移は F-17 が持つ
  - `content.publication_changed` から始まる通知の作成と送信は F-05 が持つ
- **失敗時の扱い**
  - 確定しなかった書き込み: コンテンツ、履歴の版、公開の変更の記録、メディアの公開範囲と使用箇所は変わらず、イベントも残らない。要求した側が送り直す。確認を伴う操作は、確認を示し直してから送り直す（[../domains/index.md](../domains/index.md) の「公開内容の整合性と通し番号」）
  - スキーマの反映の適用中（F-01）: コンテンツと履歴の版の書き込みは確定せず、コンテンツは変わらない。反映が終わった後に、要求した側が送り直す。規則は [../domains/schema.md](../domains/schema.md) の「WriteScope とスキーマ」が定める
  - 反映の確定が確定しなかった場合: コンテンツと履歴の版は反映前の内容のまま残り、公開の変更の記録は追記されない。出る遷移は F-01 が持つ
  - 完全削除を妨げる参照: 対象は、削除されずに、通常の管理対象またはごみ箱に残る。ごみ箱のコンテンツは、保持の期限を過ぎても残る（F-17）。妨げる参照の条件は、[../domains/content.md](../domains/content.md) の「公開取り消し・ごみ箱・完全削除の参照の条件」が定める。出る遷移は次の四つ
    - 参照元のコンテンツを対象に加えた「コンテンツを完全削除する（`purgeContent`）」（Content）
    - 妨げている履歴の版を対象に加えた「コンテンツを完全削除する（`purgeContent`）」（Content）
    - 参照が解消された後の「コンテンツを完全削除する（`purgeContent`）」（Content）と、保持期限切れのごみ箱のコンテンツでは定期ジョブ「ごみ箱の完全削除」の次の起動（F-17）。参照を解消する主体は、この表の、参照元の内容を置き換える遷移と、F-17 の、参照元の版を削除する遷移の主体
    - ごみ箱のコンテンツでは「ごみ箱のコンテンツを復元する（`restoreTrashedContent`）」（Content）
  - 参照切れ: 参照先が完全削除で無くなった下書きと履歴の版は、参照切れの参照を持ったまま残る。履歴の版から復元した下書きも同じ。出る遷移は、参照を直した「下書きを保存する（`saveDraft`）」（Content）。参照切れの規則は [../domains/content.md](../domains/content.md) の「Content」の「不変条件」と「参照の解決」が定める
  - 利用終了の途中のワークスペース: 区画が `closing` になった後、コンテンツはその時点の状態で残る。コンテンツ、履歴の版、公開の変更の記録は、F-23 の区画の消去が消す
  - 復旧と利用の再開: コンテンツと公開の変更の記録は、復旧点の内容に戻る。復旧の後の通し番号と通知の扱いは F-05 が持つ（F-24）

### F-20 メディアの公開範囲

- **対象**: `Media` の `visibility`（`private` / `public`）と、メディアの使用箇所の記録（`MediaUsageRepository`）。公開と非公開化の規則は [../domains/media.md](../domains/media.md) の「公開と非公開化」、削除の規則は同じ文書の「削除」が定める
- **トリガー**: 「メディアを登録する（`registerMedia`）」（Media）

経路。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （なし）→ `private` | 「メディアを登録する（`registerMedia`）」（Media）。実体の保管の後。保管の予約を取り除く確定と同じ UnitOfWork（F-07）。画像は、同じ UnitOfWork で派生形式の記録を作る（F-06） |
| `private` → `public` | 「コンテンツを公開する（`publishContent`）」（Content）、「値を書き換える（`rewriteValues`）」（Content）。新たに公開するメディア（[../domains/content.md](../domains/content.md) の「新たに公開するメディア」）。公開内容の変更（F-19）と同じ UnitOfWork |
| `public` → `private` | 「メディアを非公開化する（`unpublishMedia`）」（Media） |
| `private` / `public` →（削除） | 「メディアを削除する（`deleteMedia`）」（Media）、「履歴の版とともにメディアを削除する（`deleteMediaWithRevisions`）」（Content）。どちらも、[../usecases/media.md](../usecases/media.md) の「メディアの削除の手続き」で削除する。後者は、妨げている履歴の版の削除（F-17）と同じ UnitOfWork。派生形式の記録が同時に消え（F-06）、同じ UnitOfWork で実体を消去待ちにする（F-07） |

経路（使用箇所の記録）。内容の持ち主（公開内容、下書き、履歴の版）ごとの記録。書き方は [../usecases/content.md](../usecases/content.md) の「コンテンツの書き込み」が定める。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （なし）→ 使用箇所あり / 使用箇所の置き換え / 使用箇所あり →（なし） | 「コンテンツを作成する（`createContent`）」（Content）、「下書きを保存する（`saveDraft`）」（Content）、「下書きを破棄する（`discardDraft`）」（Content）、「コンテンツを公開する（`publishContent`）」（Content）、「公開を取り消す（`unpublishContent`）」（Content）、「コンテンツをごみ箱へ移動する（`trashContent`）」（Content）、「値を書き換える（`rewriteValues`）」（Content）、「版を下書きとして復元する（`restoreRevisionAsDraft`）」（Content）。内容の保存（F-17、F-19）と同じ UnitOfWork |
| 使用箇所あり →（なし） | 「コンテンツを完全削除する（`purgeContent`）」（Content）、「履歴の版を削除する（`deleteRevisions`）」（Content）、「履歴の版とともにメディアを削除する（`deleteMediaWithRevisions`）」（Content）、「期限を過ぎた履歴の版を削除する（`cleanUpExpiredRevisions`）」（Content）（定期ジョブ「履歴の版の整理」）、「期限を過ぎたごみ箱のコンテンツを完全削除する（`purgeExpiredTrash`）」（Content）（定期ジョブ「ごみ箱の完全削除」）。持ち主の削除（F-17、F-19）と同じ UnitOfWork |
| 使用箇所あり / （なし）→ 使用箇所の置き換え（切り替え） | 「反映を確定する（`commitSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。準備領域の使用箇所（F-03）への切り替え。反映の確定（F-01）と同じ UnitOfWork |

- **参照**: 「メディアを一覧・検索する（`listMedia`）」（Media）、「メディアを取得する（`getMedia`）」（Media）、「メディアを配信する（`deliverMedia`）」（Media）、「メディアの使用箇所を一覧する（`listMediaUsages`）」（Content）。配信の可否は、[../domains/media.md](../domains/media.md) の「MediaDelivery」が定める。「利用を再開する（`resumeInstallation`）」（Installation）は、確認に、Media が「利用再開前の処理」に公開する項目を示す（F-24）
- **終端**
  - 正常: 削除
  - 削除しないメディアは、`private` と `public` のどちらにも、期限なく留まる。公開使用がなくなったメディアは `public` のまま残り、出る遷移は、非公開化と削除
  - 使用箇所の記録: 持ち主の削除で無くなる
- **失敗時の扱い**
  - 確定が拒否された操作（公開範囲と使用箇所の整合性を破る確定）は、公開範囲と使用箇所を変えない。拒否する確定は、[../domains/media.md](../domains/media.md) の「MediaRepository」と「MediaUsageRepository」の契約が定める。Content の操作の続きは、[../domains/index.md](../domains/index.md) の「公開内容の整合性と通し番号」が定める（F-19）
  - 原因の操作が確定しなければ、メディアの公開範囲も使用箇所も変わらない
  - 使用箇所が残り、削除できないメディア: 出る遷移は、経路（使用箇所の記録）の主体が使用箇所を無くした後の削除
  - 準備領域の使用箇所は、切り替えまで F-03 が持つ。切り替えの確定が拒否された反映は F-01
  - スキーマの反映の適用中に確定しない操作（F-01）は、公開範囲と使用箇所を変えない
  - 削除の後に残る消去待ちは F-07 が持つ。実体の消去の失敗の扱いは [../domains/media.md](../domains/media.md) の「実体と記録の整合」に従う
  - 利用終了の途中のワークスペース: 区画が `closing` になった後、公開範囲と使用箇所はその時点の状態で残る。出る遷移は、区画の消去（F-23）。入口ごとの結果は、[../domains/index.md](../domains/index.md) の「利用できるワークスペース」が定める
  - 復旧と利用の再開: 公開範囲と使用箇所は、復旧点の内容に戻る（F-07、F-24）

### F-21 フォームの運用設定

- **対象**: 紐付け待ちの設定 `PendingFormSettings`（フォームのキーで識別する）と、`FormSettings`（`active` / `removed`。`FormId` で識別する）。経路は、紐付け待ちの設定を「紐付け待ち」と書く
- **トリガー**: 「設置を起動する（`bootInstallation`）」（Installation）（初期構成でフォームのキーを指定した宣言）、「反映を確定する（`commitSchemaApply`）」（Schema）（フォームを追加した反映）

スキーマの反映の確定に伴う作成・紐付け・`removed` への遷移を分ける規則は、[../domains/common.md](../domains/common.md) の「キーで宣言した設定の紐付け」と [../domains/form.md](../domains/form.md) の「スキーマの反映に伴う変更」が定める。

経路。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （なし）→ 紐付け待ち | 「設置を起動する（`bootInstallation`）」（Installation）。初期構成の読み込みの、ワークスペースごとの段（F-24） |
| 紐付け待ち →（なし）と、（なし）→ `active` | 「反映を確定する（`commitSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。追加したフォームと同じキーの紐付け待ちがある反映 |
| （なし）→ `active` | 「反映を確定する（`commitSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。追加したフォームと同じキーの紐付け待ちが無い反映 |
| `active` → `active`（受付状態の変更） | 「フォームの受付を有効にする（`openReception`）」（Form）、「フォームの受付を停止する（`closeReception`）」（Form） |
| `active` → `active`（通知先、通知定義、保持方針の変更） | 「通知先を追加する（`addRecipient`）」（Form）、「通知先のメールアドレスを修正する（`changeRecipientAddress`）」（Form）、「通知先を有効・無効にする（`setRecipientEnabled`）」（Form）、「通知先を削除する（`removeRecipient`）」（Form）、「通知定義を変更する（`changeNotificationTemplate`）」（Form）、「回答の保持方針を変更する（`changeRetentionPolicy`）」（Form） |
| `active` → `removed` | 「反映を確定する（`commitSchemaApply`）」（Schema）（`schema.apply_requested` の消費者）。フォームを削除した反映 |
| `removed` → `removed`（通知先の変更） | 「通知先を追加する（`addRecipient`）」（Form）、「通知先のメールアドレスを修正する（`changeRecipientAddress`）」（Form）、「通知先を有効・無効にする（`setRecipientEnabled`）」（Form）、「通知先を削除する（`removeRecipient`）」（Form） |
| `removed` →（削除） | 「回答が残らない削除済みフォームの運用設定を削除する（`disposeRemovedFormSettings`）」（Form）（定期ジョブ「回答の自動整理」） |

- **参照**: 「フォームの運用設定を一覧する（`listFormSettings`）」（Form）、「フォームの運用設定を取得する（`getFormSettings`）」（Form）、「参照の不整合が残る通知定義を一覧する（`listBrokenNotificationTemplates`）」（Form）、「回答の対象のフォームを一覧する（`listSubmissionForms`）」（Form）、「送信接続設定の更新が影響するフォームを一覧する（`listFormsAffectedByMailConnection`）」（Form）、「紐付けを待つ運用設定を一覧する（`listPendingBindings`）」（Installation）、「フォームの送信を受け付ける（`acceptSubmission`）」（Form）、「反映・事前確認を準備する（`prepareSchemaApply`）」（Schema）、通知の送信と手動再試行（F-04）、保持方針の適用（F-18）。「ワークスペースの終了前の確認を示す（`previewWorkspaceClosure`）」（Installation）は、Form が [../domains/form.md](../domains/form.md) の「利用終了の消去」に公開する項目を示す
- **終端**
  - 紐付け待ち: そのキーのフォームを追加する反映の確定まで、変わらずに残る。利用終了の区画の消去で消える（F-23）
  - `active`: フォームを削除する反映の確定まで残る
  - `removed`: 削除。そのフォームの回答が残る間（F-18）は、`removed` のまま残る
- **失敗時の扱い**
  - 初期構成の読み込みが途中で止まった設置では、ワークスペースごとの段を確定したワークスペースだけが紐付け待ちの設定を持つ。同じ宣言での起動のやり直しが、段を確定していないワークスペースの紐付け待ちの設定を作る（F-24）
  - 確定しなかった反映と、失敗で終えた反映は、設定を変えない。確定しなかった反映のその後は、F-01 が持つ
  - 確定しなかった設定の変更は、設定を変えず、メンバーがやり直す
  - 削除が確定しなかった `removed` の設定は、`removed` のまま残り、定期ジョブ「回答の自動整理」の次の起動が削除する
  - 利用終了の途中のワークスペースの設定から出る遷移は、定期ジョブ「回答の自動整理」による `removed` の削除と、区画の消去（F-23）

### F-22 アカウントと所属の索引

- **対象**: `Account`（多要素認証の `disabled` / `enrolling` / `enabled`）と、所属の索引（`MembershipIndex`）の項目 `Membership`（`joining` / `joined`。世代 `generation` を持つ）、アカウントごとの、最後に利用したワークスペースの値。どれも設置の区画にある。所属の正は、ワークスペースの区画の `Member` が持つ。型と不変条件は [../domains/identity.md](../domains/identity.md) の「Account」と、[../domains/workspace.md](../domains/workspace.md) の「Membership」「MembershipIndex」が定める
- **トリガー**: 「招待を受諾する（`acceptInvitation`）」（Workspace）、「設置を起動する（`bootInstallation`）」（Installation）（初期構成の所有者）
- **有効期限**: `joining` の項目の参加の期限。期限と、期限が決めることは、[../domains/workspace.md](../domains/workspace.md) の「Membership」が定める

経路（アカウントと多要素認証）。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （なし）→ `disabled` | 「招待を受諾する（`acceptInvitation`）」（Workspace）。アカウントを持たない人の参加の前段。所属の索引の `joining` の項目と同じ UnitOfWork（F-11） |
| （なし）→ `disabled` | 「設置を起動する（`bootInstallation`）」（Installation）。初期構成の読み込みの最初の段。宣言した所有者のアカウント（F-24） |
| `disabled` → `enrolling` | 「多要素認証の設定を始める（`startMfaEnrollment`）」（Identity） |
| `enrolling` → `enrolling`（秘密の置き換え） | 「多要素認証の設定を始める（`startMfaEnrollment`）」（Identity） |
| `enrolling` → `enabled` | 「多要素認証の設定を確定する（`confirmMfaEnrollment`）」（Identity） |
| `enabled` → `disabled` | 「多要素認証を解除する（`disableMfa`）」（Identity） |
| `enabled` → `disabled` | 「新しいパスワードを設定してアクセスを回復する（`completeAccessRecovery`）」（Identity）。解除を選んだ回復。回復依頼の `used` と同じ UnitOfWork（F-10） |

経路（所属の索引の項目）。項目を書く UnitOfWork は、どれも設置の区画で、`Member` を書くワークスペースの区画の UnitOfWork と別に確定する。段の順は [../domains/index.md](../domains/index.md) の「区画をまたぐ手順」（参加、メンバーの利用の解除、所属の索引の修復、初期構成の読み込み、ワークスペースの利用終了、利用終了の完了）、修復の決定（`MembershipRepair.decide`）と取り下げは [../domains/workspace.md](../domains/workspace.md) の「MembershipRepair」、世代の規則と項目の遷移は同じ文書の「Membership」、利用終了の最後の段は「利用終了の消去」が定める。「修復の二つの主体」は、「所属の索引を修復する（`repairMembershipIndex`）」（Workspace）（定期ジョブ「所属の索引の修復」）と、「メンバーの利用を解除する（`removeMember`）」（Workspace）の後段を指す。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （なし）→ `joining` | 「招待を受諾する（`acceptInvitation`）」（Workspace）。参加の前段 |
| `joining`・`joined` → `joining`（新しい世代） | 「招待を受諾する（`acceptInvitation`）」（Workspace）。項目があるアカウントの、受諾の前段 |
| `joining` → `joined` | 「招待を受諾する（`acceptInvitation`）」（Workspace）。確かめ直しの段。後段で参加が成立した後（F-11）。アカウントを作った参加では、セッションの作成と同じ UnitOfWork（F-09） |
| （なし）→ `joining` | 「設置を起動する（`bootInstallation`）」（Installation）。初期構成の読み込みの最初の段。宣言した所有者とワークスペースの組（F-24） |
| `joining` → `joined` | 「設置を起動する（`bootInstallation`）」（Installation）。ワークスペースごとの設置の区画の段。名簿を開設済みにする変更と同じ UnitOfWork（F-23、F-24） |
| `joining` → `joined` | 修復の二つの主体。`MembershipRepair.decide` が `confirm` を返す項目 |
| 名簿で開設済みのワークスペースの `joining` / `joined` →（除外） | 修復の二つの主体。`MembershipRepair.decide` が `withdraw` を返す項目。手順と、止まったときに残る状態は、[../domains/workspace.md](../domains/workspace.md) の「MembershipRepair」の「取り下げ」が定める |
| （なし）→ `joined` | 「招待を受諾する（`acceptInvitation`）」（Workspace）。確かめ直しの段。項目が外れた後に確定した確かめ直し。名簿に無いワークスペースでは利用終了の最後の段の後、名簿で開設済みのワークスペースでは後段の後に利用の解除と取り下げで項目が外れた後 |
| 名簿に無いワークスペースの `joining` / `joined` →（除外） | 修復の二つの主体。`MembershipRepair.decide` が `remove` を返す項目。この状態から出ることを担う主体は、[../domains/workspace.md](../domains/workspace.md) の「MembershipRepair」が定める |
| 利用終了の途中のワークスペースの `joining` / `joined` →（除外） | 「ワークスペースの利用終了を完了させる（`completeWorkspaceClosures`）」（Installation）（定期ジョブ「利用終了の完了」）。最後の段。名簿からの除外と同じ UnitOfWork（F-23） |
| 復旧の後の（なし）/ `joining` → `joined` | 「利用を再開する（`resumeInstallation`）」（Installation）。Workspace の「利用再開前の処理」。名簿で開設済みのワークスペースの、`Member` がある組（F-24） |
| 復旧の後の `joining` / `joined` →（除外） | 「利用を再開する（`resumeInstallation`）」（Installation）。Workspace の「利用再開前の処理」。名簿で開設済みのワークスペースの、`Member` が無い項目（F-24） |

経路（最後に利用したワークスペース）。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| 無し / 別のワークスペース → そのワークスペース | 「最後に利用したワークスペースを記録する（`recordLastWorkspace`）」（Workspace）。値を変える主体は、これだけ（[../domains/workspace.md](../domains/workspace.md) の「Membership」） |

- **参照**: 「所属するワークスペースを一覧する（`listMyWorkspaces`）」（Workspace）と「スクリプトの認証の内容を確かめる（`getDeviceAuthorization`）」（Token）が、[../usecases/workspace.md](../usecases/workspace.md) の「所属ワークスペースを求める手続き（`resolveAccountWorkspaces`）」で、項目と、最後に利用したワークスペースを読む。「利用を再開する（`resumeInstallation`）」（Installation）は、確認に、確定する対象と外す対象の件数を示す（F-24）
- **終端**
  - アカウントは、多要素認証のどの状態でも残り、正常の終端を持たない。`enrolling` から出る遷移は、設定の確定と、設定の始め直し
  - どのワークスペースの項目も持たないアカウントは、残る。メンバーの利用の解除、取り下げて外れた参加、利用終了、利用再開前の処理が、この状態を残す。出る遷移は、参加の前段
  - 索引の項目の終端は、`Member` がある `joined` と、除外
- **失敗時の扱い**
  - `Member` が無い項目から出る遷移と主体は、経路（所属の索引の項目）の表が持つ。止まった場所ごとに残る状態は、[../testcases/workspace/acceptInvitation.md](../testcases/workspace/acceptInvitation.md)、[../testcases/workspace/removeMember.md](../testcases/workspace/removeMember.md)、[../testcases/workspace/repairMembershipIndex.md](../testcases/workspace/repairMembershipIndex.md) が定める
  - 後段の後に止まった参加と、確かめ直しが確定しなかった参加は、成立していて、`Member` がある `joining` の項目を残す。定期ジョブ「所属の索引の修復」が `joined` にする
  - 修復の UnitOfWork が確定しなかった項目は、変わらずに残り、定期ジョブ「所属の索引の修復」の次の起動が直す。途中で止まった起動の残りも同じ
  - 最初の段の後に止まった初期構成の読み込みは、名簿で準備中のワークスペースの、所有者の `joining` の項目を残す。同じ宣言での次の起動が、`joined` にする（F-24）
  - 最後の段が確定しなかった場合、名簿の項目は終了中のまま、そのワークスペースの項目も残り、定期ジョブ「利用終了の完了」の次の起動が完了させる。最後の段の後に置かれた、そのワークスペースの項目は、経路の、名簿に無いワークスペースの項目の遷移をたどる
  - 復旧の後: 復旧点より後に作ったアカウントは、復旧で無くなる。Workspace の「利用再開前の処理」が、アカウントが無いメンバーと、索引と `Member` の食い違いを直す。削除するメンバーが所有者のワークスペースに残る状態は F-23 が持つ。削除された人は、招待を受けて、新しいアカウントで参加し直す（F-11）。直す対象が残る間、設置は利用再開前のまま残り、実行し直した再開が、残りを直す（F-24）。利用終了の途中のワークスペースの項目は、利用の再開の後の利用終了の続きが、名簿に無いワークスペースの項目は、定期ジョブ「所属の索引の修復」が扱う

### F-23 ワークスペース

- **対象**: 設置の区画の名簿の項目（`WorkspaceDirectoryEntry`。`provisioning` / `open` / `retiring`）と、ワークスペースの区画の `Workspace`（`active` / `closing` / `closed`）の組。`active` の区画では、アクセスの方針 `AccessPolicy` の所有者の指定（`designated` / `vacant`）を括弧で添える。経路は、状態を「名簿の項目 / 区画の `Workspace`」で書く。「なし」は、項目または `Workspace` が無いことを指す。名簿と区画は、別の UnitOfWork で確定する。型は [../domains/workspace.md](../domains/workspace.md) の「Workspace」「AccessPolicy」「WorkspaceDirectoryEntry」が定める。利用終了が消去する区画の集約と記録の遷移は各フローが、区画の外の実体は F-07 が、設置の区画に残る記録は F-12、F-22 が持つ
- **トリガー**: 「ワークスペースを作成し、初期所有者の招待を発行する（`createWorkspace`）」（Installation）、「設置を起動する（`bootInstallation`）」（Installation）、「ワークスペースの利用を終了する（`closeWorkspace`）」（Installation）

経路。段の順は、[../domains/index.md](../domains/index.md) の「区画をまたぐ手順」（ワークスペースの作成、初期構成の読み込み、ワークスペースの利用終了、利用終了の完了、バックアップと復旧）が定める。各段の内容と、途中で止まったときの状態は、[../domains/workspace.md](../domains/workspace.md) の「初期構成の宣言」「利用終了の消去」「利用再開前の処理」が、作成と利用終了が並行したときの結果は、同じ文書の「作成と利用終了の並行」が定める。名簿と `Workspace` の組ごとの出る遷移と主体は、下の表が持つ。「利用終了の二つの主体」は、「ワークスペースの利用を終了する（`closeWorkspace`）」（Installation）と、「ワークスペースの利用終了を完了させる（`completeWorkspaceClosures`）」（Installation）（定期ジョブ「利用終了の完了」）を指す。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| なし / なし → `provisioning` / なし | 「ワークスペースを作成し、初期所有者の招待を発行する（`createWorkspace`）」（Installation）。名簿への登録の段 |
| `provisioning` / なし → `provisioning` / `active`（`vacant`） | 「ワークスペースを作成し、初期所有者の招待を発行する（`createWorkspace`）」（Installation）。区画の段 |
| `provisioning` / `active`（`vacant`）→ `open`（作成が終わっていない）/ `active`（`vacant`） | 「ワークスペースを作成し、初期所有者の招待を発行する（`createWorkspace`）」（Installation）。名簿を開設済みにする段。この確定の後に、作成は、初期所有者の招待を発行する（F-11） |
| `open`（作成が終わっていない）/ `active`（`vacant`）→ `open`（作成を終えた）/ `active`（`vacant`） | 「ワークスペースを作成し、初期所有者の招待を発行する（`createWorkspace`）」（Installation）と「初期所有者の招待を発行し直す（`reissueInitialOwnerInvitation`）」（Installation）。作成を終える段。招待を届けた後（F-11）。作成が終わったかどうかは、名簿の項目の `creation`（[../domains/workspace.md](../domains/workspace.md) の「WorkspaceDirectoryEntry」）が持つ |
| なし / なし → `provisioning` / なし | 「設置を起動する（`bootInstallation`）」（Installation）。初期構成の読み込みの最初の段（F-24） |
| `provisioning` / なし → `provisioning` / `active`（`designated`） | 「設置を起動する（`bootInstallation`）」（Installation）。ワークスペースごとの区画の段（F-12、F-13、F-21） |
| `provisioning` / `active`（`designated`）→ `open` / `active`（`designated`） | 「設置を起動する（`bootInstallation`）」（Installation）。ワークスペースごとの設置の区画の段（F-22） |
| `open` / `active`（`vacant`）→ `open` / `active`（`designated`） | 「招待を受諾する（`acceptInvitation`）」（Workspace）。初期所有者の招待の受諾の後段（F-11、F-22） |
| `provisioning`・`open` / なし・`active` → `retiring` / 同じ `Workspace` | 「ワークスペースの利用を終了する（`closeWorkspace`）」（Installation）。名簿の状態の変更。この確定から、ワークスペースは利用終了の途中になる |
| `retiring` / なし → `retiring` / `closing` | 利用終了の二つの主体。開始。区画の段の前に止まった作成の利用終了が当たる |
| `retiring` / `active` → `retiring` / `closing` | 利用終了の二つの主体。開始 |
| `retiring` / `closing` → `retiring` / `closed` | 利用終了の二つの主体。開始の後の、区画の消去 |
| `retiring` / `closed` → なし / `closed` | 「ワークスペースの利用終了を完了させる（`completeWorkspaceClosures`）」（Installation）（定期ジョブ「利用終了の完了」）。Media の処理が残っていないことを確かめ、区画の外の実体を消去した後（F-07）の、最後の段。同じ UnitOfWork で、設置の区画に残るそのワークスペースの記録が消える（F-12、F-22） |
| 復旧の後の、名簿の状態が `WorkspaceDirectoryEntry.statusFor` の結果と違う組 → `statusFor` の結果 / 同じ `Workspace` | 「利用を再開する（`resumeInstallation`）」（Installation）。Workspace の「利用再開前の処理」の名簿の状態（F-24） |
| 復旧の後の `open` / `active`（`designated`。所有者のメンバーのアカウントが無い）→ `open` / `active`（`vacant`） | 「利用を再開する（`resumeInstallation`）」（Installation）。Workspace の「利用再開前の処理」のアカウントが無いメンバー（F-22、F-24） |

- **参照**
  - 利用できるかどうかは `WorkspaceAvailability.of` が求め、入口ごとの結果は [../domains/index.md](../domains/index.md) の「利用できるワークスペース」が定める
  - 定期ジョブ、「状態を確認する（`checkInstallationStatus`）」（Installation）、「所属するワークスペースを一覧する（`listMyWorkspaces`）」（Workspace）は、対象のワークスペースを名簿の状態から求める。状態確認が示す Workspace の項目は、[../domains/workspace.md](../domains/workspace.md) の「状態確認の項目」が定める
  - 「ワークスペースの終了前の確認を示す（`previewWorkspaceClosure`）」（Installation）、「設置全体の終了前の確認を示す（`previewInstallationClosure`）」（Installation）。項目は、各ドメインの「利用終了の消去」が定める
- **終端**
  - 正常: なし / `closed`。最後の段の確定で、利用終了は完了する。完了の時点で区画と保管先に残るものは、[../domains/workspace.md](../domains/workspace.md) の「利用終了の消去」の最後の段が定める。`closed` の `Workspace` は、設置全体の利用終了での保存先の消去で消える（F-24）。完了の後のトークンの値は F-12、取得済みのバックアップに残るデータは F-08 が持つ
  - 正常: `open` / `active`（`designated`）。利用終了まで留まる
  - 異常: 所有者がいない（`vacant`）ワークスペースは、`open` / `active` のまま残る。状態確認が示す（[../domains/workspace.md](../domains/workspace.md) の「状態確認の項目」）。出る遷移は、届いている招待の受諾、運営者が「初期所有者の招待を発行し直す（`reissueInitialOwnerInvitation`）」（Installation）で発行する招待の受諾（作成の実行し直しも同じ手順で発行する。F-11）、利用終了
  - 異常: 名簿を開設済みにする前に止まった作成は、準備中の項目（`provisioning` / なし、`provisioning` / `active`）を残す。状態確認が示す。出る遷移は、運営者による作成の続き（同じワークスペースの ID での実行）、利用終了の名簿の状態の変更、区画に `Workspace` がある項目の、復旧の後の利用再開前の処理。初期構成の読み込みが作った準備中の項目は、同じ宣言での次の起動が続きを進める（F-24）
  - 終了中の項目（`retiring` / なし、`retiring` / `active`、`retiring` / `closing`、`retiring` / `closed`）は、終端ではない。定期ジョブ「利用終了の完了」が、開始から、名簿から外すまで進める
- **失敗時の扱い**
  - 作成が途中で止まった場合: 準備中の項目、または `open`（作成が終わっていない）/ `active`（`vacant`）が残る。止まった場面ごとの残る状態と続け方は、[../domains/workspace.md](../domains/workspace.md) の「初期構成の宣言」が定める。回復の主体は運営者で、同じワークスペースの ID で作成を実行し直す（F-11）
  - 作成と利用終了が並行した場合: 残る組と、先に確定した側ごとの結果は、[../domains/workspace.md](../domains/workspace.md) の「作成と利用終了の並行」が定める。名簿に項目が残る組の回復の主体は、定期ジョブ「利用終了の完了」で、経路の開始から、名簿から外すまで進める
  - 利用終了の途中で、区画が `active` の間（`retiring` / `active`）: 名簿の状態の変更の後、開始の前に止まった利用終了、開始が競合で確定しなかった利用終了、復旧の後の食い違い、区画の段が開始より先に確定した作成が残す。この間に区画に確定する書き込みは、[../domains/index.md](../domains/index.md) の「ワークスペースの作成と利用終了」が定める。定期ジョブ「利用終了の完了」の次の起動、または運営者の実行し直しが、開始から進める
  - 利用終了の途中のワークスペースの、進行中の反映と事前確認、準備領域、未送信の通知は、どれも区画の消去で消える。進まなくなる時点、残る状態、区画の消去のほかに出る遷移は、対象ごとに違い、F-01、F-02、F-03、F-04、F-05 が持つ
  - 開始、区画の消去の途中で失敗・中断した利用終了: 項目は終了中のまま残る。区画に残る状態と続け方は、[../domains/workspace.md](../domains/workspace.md) の「利用終了の消去」が定める。回復の主体は、定期ジョブ「利用終了の完了」の次の起動、または運営者の実行し直し
  - `retiring` / `closed` のワークスペースは、Media の保管の予約と消去待ちの記録が区画に残る間、`retiring` / `closed` のまま残る。二つの記録の遷移と、実体の消去を始められる条件は F-07 が持つ。回復の主体は、定期ジョブ「利用終了の完了」の次の起動
  - 実体の消去または最後の段が失敗・中断した場合、項目は終了中のまま残る。回復の主体は、定期ジョブ「利用終了の完了」の次の起動
  - そのワークスペースを指すスクリプトの認証の依頼に残る状態は F-15 が持つ
  - `operating` でない設置では、定期ジョブ「利用終了の完了」は動かず、終了中の項目は残る（F-24）
  - 復旧の後: 名簿の状態を直す対象は、[../domains/workspace.md](../domains/workspace.md) の「WorkspaceDirectoryEntry」の `statusFor` の表が定める。`provisioning` / なしの項目は、設置が `operating` になった後も準備中のまま残り、運営者が、作成の続きを行うか、利用終了を実行する。終了中の項目は、設置が `operating` になった後に、定期ジョブ「利用終了の完了」が、開始から進めて完了させる（F-24）。利用再開前の処理が所有者の指定を `vacant` にしたワークスペースと、開設済みにしたワークスペースは、所有者がいないワークスペースとして残る。出る遷移は、終端の異常の、所有者がいないワークスペースのとおり（F-11、F-22）

### F-24 設置

- **対象**: 設置の状態 `InstallationState`（`restoring` / `restore_failed` / `awaiting_resume` / `uninitialized` / `loading` / `update_required` / `updating` / `operating`）。`Installation`（設置の区画）、復旧の記録 `Recovery`（設置のデータの外）、実行中の CMS が扱うデータの版から求める。経路は、設置の状態で書き、状態の中を分ける値（`updating` の `cms`、`awaiting_resume` のデータの版が合うかどうか、`uninitialized` の設置に残った完了した復旧の記録 `leftover` の有無）を括弧で添える。状態の求め方（`InstallationState.of`）と、状態ごとに受け付けるもの（`InstallationState.admit`）は、[../domains/installation.md](../domains/installation.md) の「設置の状態」が定める
- **トリガー**: 「設置を起動する（`bootInstallation`）」（Installation）、「バックアップから設置全体を復旧する（`restoreInstallation`）」（Installation）、「設置を更新する（`updateInstallation`）」（Installation）

経路。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| `uninitialized`（`leftover` がある）→ `uninitialized`（`leftover` が無い） | 「設置を起動する（`bootInstallation`）」（Installation）。設置を初期化する起動 |
| `uninitialized`（`leftover` が無い）→ `loading` | 「設置を起動する（`bootInstallation`）」（Installation）。宣言がある起動の最初の段（F-12、F-22、F-23） |
| `loading` → `loading`（名簿の項目が開設済みになる） | 「設置を起動する（`bootInstallation`）」（Installation）。宣言のワークスペースごとの、区画の段と、設置の区画の段。ワークスペース、所属の索引、トークン、運用設定の遷移は、各フローが持つ（F-12、F-13、F-21、F-22、F-23） |
| `loading` → `operating` | 「設置を起動する（`bootInstallation`）」（Installation）。すべてのワークスペースの段の後の、最後の段。途中で止まった読み込みでは、同じ宣言での起動が、続きからこの段まで行う |
| `uninitialized`（`leftover` が無い）→ `operating` | 「設置を起動する（`bootInstallation`）」（Installation）。宣言が無い起動 |
| `operating` → `update_required` | ホスティング運営者（運用手順）。更新手順に沿って、CMS を、扱うデータの版が保存済みの版と違う CMS に入れ替える |
| `update_required` → `operating` | ホスティング運営者（運用手順）。CMS を、保存済みのデータの版を扱う CMS に入れ替える |
| `update_required` → `updating` | 「設置を更新する（`updateInstallation`）」（Installation）。`UpdateReadiness.assess` が `ready` と判定した設置の、更新の開始の保存（F-08） |
| `updating`（`target`）→ `operating` | 「設置を更新する（`updateInstallation`）」（Installation）。更新先の版を扱う CMS での実行の、更新の完了の保存 |
| `updating`（`target`）↔ `updating`（`other`） | ホスティング運営者（運用手順）。実行中の CMS の版を入れ替える |
| `uninitialized` / `loading` / `update_required` / `updating` / `operating` / `awaiting_resume` → `restoring` | 「バックアップから設置全体を復旧する（`restoreInstallation`）」（Installation）。開始の記録の保存 |
| `restoring` → 開始の前の状態 | 「バックアップから設置全体を復旧する（`restoreInstallation`）」（Installation）。開始の取り下げ（`Recovery.withdraw`）。`BackupStore.restore` が置き換えを始めずに拒否した開始。復旧の記録は開始の前の内容に戻り、記録が無かった設置では記録が消える |
| `restoring` → `awaiting_resume` | 「バックアップから設置全体を復旧する（`restoreInstallation`）」（Installation）。置き換えの成功の後の、完了の記録の保存。`Installation` を含む設置のデータと、メディアの保管先の実体は、バックアップの内容になり（F-07、F-08）、更新の途中だった設置の更新の進行は消える |
| `restoring` → `restore_failed` | 「バックアップから設置全体を復旧する（`restoreInstallation`）」（Installation）。`BackupStore.restore` の、置き換えを始めなかった三つの結果のほかの失敗（打ち切りを含む）を受け取った後の、失敗の記録の保存。結果ごとの記録の遷移は [../domains/installation.md](../domains/installation.md) の「Recovery」が定める |
| `restoring` → `restore_failed` | 主体は無い。結果が分からないまま止まった復旧で、`Recovery.isOccupied` が現在時刻から求める（占有の期限の経過） |
| `restore_failed` → `restoring` | 「バックアップから設置全体を復旧する（`restoreInstallation`）」（Installation）。復旧の始め直し |
| `awaiting_resume`（データの版が違う）↔ `awaiting_resume`（データの版が合う） | ホスティング運営者（運用手順）。実行中の CMS の版を入れ替える |
| `awaiting_resume`（データの版が合う。残件数が 1 以上）→ 同じ状態（残件数が減る） | 「利用を再開する（`resumeInstallation`）」（Installation）。利用再開前の確認の内容を示して確認を求め、確認を受けた実行が、各ドメインの「利用再開前の処理」を呼ぶ。処理が変える状態の遷移は、各フローが持つ（F-01、F-02、F-03、F-04、F-18、F-22、F-23、F-25） |
| `awaiting_resume`（データの版が合う。残件数が 0）→ `operating` | 「利用を再開する（`resumeInstallation`）」（Installation）。同じ実行が、処理の後に、復旧の記録を消す。再開を受け付ける設置は [../domains/installation.md](../domains/installation.md) の「設置の状態」の `admit`、再開の可否は同じ文書の「Recovery」の `assertResumable` が定める |
| 任意の状態 → `uninitialized`（`leftover` が無い） | ホスティング運営者（運用手順）。終了手順に沿って、保存先を空にし、復旧の記録を消去する。始め直す設置は、ここから起動する |
| `awaiting_resume` → `uninitialized`（`leftover` がある） | ホスティング運営者（運用手順）。完了した復旧の記録を消さずに、保存先を空にする |

初期構成の読み込みの段の順と、利用再開前の処理で直す食い違いは [../domains/index.md](../domains/index.md) の「区画をまたぐ手順」、各ドメインの手順を呼ぶ順は同じ文書の「手順を呼ぶ順」の表、各段の内容と残件数の求め方は各ドメインの「初期構成の宣言」「利用再開前の処理」が定める。復旧の記録を消す主体は、[../domains/installation.md](../domains/installation.md) の「Recovery」が定める。

- **参照**
  - 「設置の状態を確かめる（`checkInstallationAvailability`）」（Installation）。呼ぶ入口は、[../domains/index.md](../domains/index.md) の「設置の状態」が定める
  - 「設置を起動する（`bootInstallation`）」（Installation）。`restoring` と `restore_failed` の設置の起動の結果が、設置の状態と占有の期限を示す。[../domains/installation.md](../domains/installation.md) の「起動と初期構成」が定める
  - 「バックアップを一覧する（`listBackups`）」（Installation）（F-08）
  - 「利用を再開する（`resumeInstallation`）」（Installation）。確認に、残件数と、各ドメインが「利用再開前の処理」に公開する項目を示す
  - 「状態を確認する（`checkInstallationStatus`）」（Installation）。各ドメインが「状態確認の項目」に公開する項目を示す（F-08、F-23）
  - 「設置全体の終了前の確認を示す（`previewInstallationClosure`）」（Installation）。各ドメインが「利用終了の消去」に公開する項目を示す（F-08、F-23）
- **終端**
  - 正常: `operating`。初期構成の読み込みの完了、宣言が無い起動、設置の更新の完了、`update_required` の設置の CMS を保存済みのデータの版を扱う CMS に入れ替えること、利用の再開が至る
  - 正常: 設置全体の利用終了。ホスティング運営者（運用手順）が、[../domains/installation.md](../domains/installation.md) の「終了手順」に沿って、CMS の外で消去する（F-08）。利用終了が完了したワークスペースの区画に残る `closed` の `Workspace` も、保存先の消去で消える（F-23）
  - 異常: 最初の段の確定の後に止まった読み込みは、`loading` のまま残る。準備中の名簿の項目が残る（F-23）。出る遷移は、読み込みを始めた宣言と版での起動による読み込みの続き、復旧の開始、保存先を空にした始め直し。続きを進める起動の条件は、[../domains/installation.md](../domains/installation.md) の「起動と初期構成」が定める
  - 異常: 更新を始めていない設置（`update_required`）は、そのまま残る。出る遷移は、設置の更新の開始、CMS を保存済みのデータの版を扱う CMS に入れ替えること、復旧の開始。実行中の CMS が保存済みのデータの版からの更新を持たない設置（`UpdateReadiness.assess` の `unsupported`）では、設置の更新は始まらず、出る遷移は、残りの二つ
  - 異常: 更新の途中の設置（`updating`）は、`updating` のまま残る。回復の主体はホスティング運営者で、更新先の版を扱う CMS で更新を実行し直すか、復旧を始める。操作は [../domains/installation.md](../domains/installation.md) の「UpdateReadiness」が定める
  - 異常: `BackupStore.restore` の、置き換えを始めなかった三つの結果のほかの失敗を受け取った復旧は、設置を `restore_failed` にする。回復の主体はホスティング運営者で、復旧を始め直すか、保存先を空にして始め直す（経路の表）
  - 異常: 結果を受け取る前に止まった復旧と、完了・失敗・取り下げの保存が確定しなかった復旧は、占有の期限の内は `restoring`、過ぎると `restore_failed` で残る。回復の主体は、上の `restore_failed` と同じ
  - 異常: 利用を再開していない設置、済ませる処理が残る設置、データの版が合わない設置は、`awaiting_resume` のまま残る。出る遷移は、利用の再開の実行し直し、CMS を復旧したデータの版を扱う CMS に合わせた後の利用の再開、新しい復旧の開始。再開を受け付ける設置は、[../domains/installation.md](../domains/installation.md) の「設置の状態」が定める
- **失敗時の扱い**
  - 受け付けない運用コマンドは、設置の状態を変えない。設置の設定の問題で受け付けない設置から出る遷移は、ホスティング運営者が設置の設定を直すこと。当たる入口と運用コマンドは、[../domains/installation.md](../domains/installation.md) の「設置の状態」の `InstallationState.blockingIssues` の表が定める
  - 保存先の障害で `Installation` を読めない設置: 回復の主体はホスティング運営者で、障害を解消するか、空の保存先を用意する。そのときの状態の求め方は [../domains/installation.md](../domains/installation.md) の「設置の状態」が定める
  - 起動しない起動と、設置の状態を変えない起動は、[../domains/installation.md](../domains/installation.md) の「起動と初期構成」と「設置の状態」が定める
  - 最初の段の確定の前に止まった起動は、`uninitialized` の設置を残す。回復の主体は、起動し直した起動で、初期化する。段ごとに残る状態は、[../testcases/installation/bootInstallation.md](../testcases/installation/bootInstallation.md) が定める
  - 始まらなかった更新と、途中で止まった更新は、設置を `update_required` または `updating` のまま残す。回復の主体は、ホスティング運営者が実行し直す更新。止まった場面ごとに残る状態は、[../testcases/installation/updateInstallation.md](../testcases/installation/updateInstallation.md) が定める
  - 開始の保存の前に終わった復旧は、復旧の記録と設置のデータを変えない。開始の保存の後に止まった復旧が残す状態は、終端の異常のとおりで、回復の主体は、ホスティング運営者が始め直す復旧。やり直しの単位は復旧の全体。止まった場面ごとに残る状態は、[../testcases/installation/restoreInstallation.md](../testcases/installation/restoreInstallation.md)、置き換えの契約と占有は、[../domains/installation.md](../domains/installation.md) の「BackupStore」と「Recovery」が定める
  - 復旧の後に残る、設置の区画とワークスペースの区画の食い違いの種類と、直す主体は、[../domains/index.md](../domains/index.md) の「手順ごとの規則」が定める（F-15、F-22、F-23）
  - 復旧の前から続く手順の後の段が、置き換えの完了の後に確定して残す食い違いと、直す主体は、[../domains/installation.md](../domains/installation.md) の「BackupStore」が定める（F-15、F-22）
  - 利用の再開の実行で確定しなかった対象と、その実行が読まなかった対象は、残件数に残り、設置は `awaiting_resume` のまま残る。確定した処理の結果は残る。回復の主体は、ホスティング運営者が実行し直す再開で、残りだけを行う。実行し直しても残件数が減らない設置から出る遷移は、保存先の障害の解消と、復旧のやり直し。一回の実行が読む対象の範囲は、各ドメインの「利用再開前の処理」が定める（F-04）
  - 復旧の記録の競合で確定しなかった利用の再開は、復旧の記録を変えない。設置は、先に確定した操作が残した状態（並行した復旧の開始が先なら `restoring`）になり、出る遷移は、その状態の経路のとおり
  - 再開の後に残る名簿の項目（終了中の項目、準備中の項目）と、所有者がいないワークスペースから出る遷移は、F-23 が持つ
  - バックアップから戻ったドメインイベントは、利用の再開の後に配送が続く。戻る配送の状態は [../domains/installation.md](../domains/installation.md) の「BackupStore」が定める。設置が稼働中でない間の配送で隔離されたイベントは、ホスティング運営者（運用手順）が配送し直す（[../domains/common.md](../domains/common.md) の「QuarantinedEventQuery」）。復旧点の状態に戻った集約と記録の、再開の後の遷移は、各フローの失敗時の扱いが持つ

### F-25 未対処の失敗

- **対象**: 送信失敗の `FailureHandling`（`unhandled` / `handled`）。フォームの通知は `failed` の `SubmissionNotification`、表示アプリへの通知は `failed` の `WebhookDelivery`（通知と宛先の組ごとの送信）が持つ。規則は [../domains/common.md](../domains/common.md) の「FailureHandling」が定める
- **トリガー**: 送信失敗への遷移（F-04、F-05）

経路。

| 遷移元 → 遷移先 | 主体 |
| --- | --- |
| （対処の状態なし）→ `unhandled` | F-04 と F-05 の、`failed` への遷移の主体 |
| `unhandled` → `handled` | 「通知の送信失敗を対処済みとして記録する（`markFailuresHandled`）」（Form） |
| `unhandled` → `handled` | 「送信失敗を対処済みとして記録する（`markDeliveryFailureHandled`）」（Webhook） |
| `unhandled` / `handled` →（対処の状態なし） | F-04 と F-05 の、`failed` から出る手動再試行の遷移の主体 |
| `unhandled` / `handled` →（削除） | 回答を削除する遷移の主体（F-18）。フォームの通知が持つ遷移 |

- **参照**
  - Form: 「未対処の送信失敗の件数を取得する（`countUnhandledFailures`）」（Form）、「回答を一覧・検索する（`searchSubmissions`）」（Form）、「回答を取得する（`getSubmission`）」（Form）。単位は、回答ごとの通知
  - Webhook: 「未対処の送信失敗の件数を求める（`countUnhandledDeliveryFailures`）」（Webhook）、「通知と宛先の組を一覧する（`listDeliveries`）」（Webhook）、「一つの通知の送信結果を確かめる（`getNotificationDeliveries`）」（Webhook）。単位は、通知と宛先の組
  - Installation: 「状態を確認する（`checkInstallationStatus`）」（Installation）、「ワークスペースの終了前の確認を示す（`previewWorkspaceClosure`）」（Installation）、「設置全体の終了前の確認を示す（`previewInstallationClosure`）」（Installation）は、Form と Webhook が公開する未対処の送信失敗の項目を示す。「利用を再開する（`resumeInstallation`）」（Installation）は、確認に、Form が公開する項目を示す。項目の定義は、[../domains/form.md](../domains/form.md) の「状態確認の項目」「利用終了の消去」「利用再開前の処理」と、[../domains/webhook.md](../domains/webhook.md) の「状態確認の項目」「利用終了の消去」が持つ
- **終端**
  - 手動再試行の開始。再試行の後の対処の状態は、[../domains/common.md](../domains/common.md) の「FailureHandling」が定める
  - 回答の削除（フォームの通知）
  - 再試行しない送信失敗は、`unhandled` または `handled` のまま残る。フォームの通知は回答の削除まで（F-18）、表示アプリへの通知はワークスペースの利用終了の区画の消去まで残る（F-23）
- **失敗時の扱い**
  - 受け付けなかった対処の記録と、確定しなかった対処の記録は、どの対象も変えず、メンバーがやり直す。受け付ける条件と結果は、[../domains/form.md](../domains/form.md) の「対処の記録」と [../domains/webhook.md](../domains/webhook.md) の「DeliveryBatch」が定める
  - 通知を作る前のイベントの配送の失敗（隔離された `content.publication_changed`）は、通知と送信が無いので、対処の状態を持たない。ホスティング運営者の状態確認が担う（F-05）
  - 利用終了の途中のワークスペース: 区画が `closing` になった後、対処の状態はその時点の状態で残る。出る遷移は、フォームの通知では、区画の消去（F-23）と、回答の削除（定期ジョブ「回答の自動整理」と、復旧の後の利用再開前の処理による、保持期限切れの回答の削除。F-18）。表示アプリへの通知では、区画の消去（F-23）だけ
  - 復旧の後: 「利用を再開する（`resumeInstallation`）」（Installation）が `failed` にしたフォームの通知は、経路の（対処の状態なし）→ `unhandled` をたどる（F-04）。表示アプリへの通知の対処の状態は、復旧点のまま残る（F-05）
