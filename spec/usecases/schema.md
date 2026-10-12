# Schema のユースケース

Schema ドメインのユースケースを定める。型、振る舞い、ポートは [../domains/schema.md](../domains/schema.md)、ドメインをまたぐ規則は [../domains/index.md](../domains/index.md) が定める。規則は domains が定め、ユースケースは、振る舞いとポートを組み合わせる。結果の種類ごとの条件、エラー、途中で止まったときに残る状態は、`spec/testcases/schema/` のユースケーステストが定める。コードは `packages/core/src/application/schema/` に置く。

ほかのドメインに属するユースケースは、[../domains/index.md](../domains/index.md) の「ユースケースの所属」に従う。スキーマから型を導く規則は、[../domains/schema.md](../domains/schema.md) の「型の対応」が定め、`packages/client` が使う。

## 共通

各ユースケースは、次の手続きを名前で指す。

### 実行者と認可

- 入口と認可の分担、メンバーとトークンの認可の手続き、認可を行う時点、拒否の順は [../domains/index.md](../domains/index.md) の「認可」「利用できるワークスペース」に、確定の時点の `WORKSPACE_NOT_ACTIVE` の扱いは同「競合の扱い」に従う
- 各ユースケースの「認可」は、受け付ける実行者の種類と、メンバーに必要な `Permission`、トークンに必要な `TokenGrant` を書く。「トークン」は `appToken` と `memberToken` を指す
- トークンの認可は、ユースケースがトークンの認可の手続き（`authorizeTokenOperation`、[token.md](token.md)）で行う。`AccessRequirement` の `grant` は `schema_apply`、`target` は `{ kind: "workspace" }`、`permissions` は `schema:change`
- ワークスペースの ID は、入口が渡す。トークンを示す要求では、入口がトークンの値から取り出す（[../domains/index.md](../domains/index.md) の「ワークスペースを知らない要求」）
- イベントの消費者と定期ジョブの対象のワークスペースは、イベントでは、記録したイベントが持つ `workspaceId`、定期ジョブでは入力の `WorkspaceId`

### 対象のワークスペース

実行者が `member` またはトークンのユースケースは、出力に、対象のワークスペースの `WorkspaceId` を含める。値は、ユースケースが区画を求めるのに使った、入力の `WorkspaceId`。コマンドが、対象のワークスペースを結果に示すために使う。各ユースケースの「出力」は、これを書き直さない。

### ワークスペースのデータの書き込み

- Schema のユースケースが `UnitOfWorkProvider.run` に渡す `WriteScope`（[../domains/common.md](../domains/common.md)）は、ワークスペースの区画の `workspace` で、`workspaceId` はユースケースが入力に受けた ID。条件は次の二つのどちらか。各ユースケースのトランザクション境界が、どちらかを書く
  - 「`active` の `WriteScope`」: 条件 `active`。前提の版の `WriteScope` を使わない UnitOfWork のすべて。反映の要求と確定は、`WorkspaceSchema` を読んだ `expectedVersion` で保存して、判定を確定の時点まで保つ
  - 「前提の版の `WriteScope`」: 条件 `schemaVersion`。渡す版は [../domains/common.md](../domains/common.md) の「WriteScope」に従う。`WorkspaceSchema` を保存しない、事前確認の受け付けと準備の進行の UnitOfWork が使う
- Schema のユースケースは、条件 `stableSchema` を使わない
- UnitOfWork の中と外の読み書き、書き込みの順、競合をやり直さないことは、[../domains/common.md](../domains/common.md) の「TransactionalRepository」と「UnitOfWorkProvider」に従う

### 書き込みの量

書き込みの回数は、[../domains/common.md](../domains/common.md) の「UnitOfWorkProvider」の「書き込みの量」に従う。各ユースケースは、トランザクション境界に、UnitOfWork の回数の上限を書く。

### 拒否した要求と差分の無い要求

`WorkspaceSchema.planApply` と `planRecord` の `outcome` ごとの書き込み。どの要求が当たるかは [../domains/schema.md](../domains/schema.md) の「反映の要求の検証」「適用済みとしての記録の検証」が、拒否の理由からエラーへの対応は同「拒否の理由とエラー」が定める。

| `outcome` | 書き込み |
| --- | --- |
| `rejected`、`unchanged` | 何も書かない |
| `resolvesFailure` | `active` の `WriteScope` の一つの UnitOfWork で、`workspaceSchema` の保存だけを確定する。書き込みは 1 回 |

### ジョブを引く

ジョブの ID（`SchemaApplyJobId`）で、受け付けた要求の一回（`SchemaApplyJob`）を引く手続き。対象のワークスペースの区画の `SchemaApplyJobRepository.findById` で読む。結果は、ジョブ、または `null`。

- 反映と適用の事前確認の要求は、検証の前に、入力のジョブの ID でこの手続きを行い、結果を [../domains/schema.md](../domains/schema.md) の「受け付けた要求と再送」の判定（`SchemaApplyJob.isReplayOf`）に渡す
- イベントの消費者と「受け付けた反映・事前確認の進行と結果を取得する」も、この手続きで対象を引く

### 要求の入力

反映と適用の事前確認の入力は、移行定義による反映と事前確認は `{ jobId, baseSchemaVersion, schema, migrations }`、直接反映は `{ jobId, baseSchemaVersion, schema, lossAcknowledged }`。入口は、[app-api.md](app-api.md) の要求の本文から作る。

- `jobId` は `SchemaApplyJobId`。境界での扱いは [../domains/common.md](../domains/common.md) の「Clock と IdGenerator」に従う
- `baseSchemaVersion`、`schema`、`migrations`、`lossAcknowledged` は、`ApplyRequest` の同じ名前のフィールドになる。`lossAcknowledged` は真偽値で、省けない
- `baseSchemaVersion` は、要求した側が開発者に示した現行スキーマの版。どの版を添えるかは、[app-api.md](app-api.md) の「要求とユースケースの対応」の、スキーマの反映が定める。ユースケースは、渡された版を `planApply` と `planRecord` に渡すだけ
- `source` と `mode` は、入力に持たない。ユースケースごとに決まる

### 実行者の名称

- 要求と記録のユースケースは、記録の時点の名称を読み、`RecordedActor`（`TokenActor` と `ActorNames`）を作る。`names.token` と `names.member` は、[token.md](token.md) の `authorizeTokenOperation` の「入出力」の出力の読み方に従う。`AccountRepository.findById` は、設置の区画（`PartitionReader.installation()`）のリポジトリで読む
- ジョブと `SchemaVersionRecord` は、この名称を持つ。準備、確定、一覧、取得は、それらが持つ名称を使い、トークンとメンバーを読み直さない
- 準備が Content の振る舞いへ渡す `Performer`（[../domains/content.md](../domains/content.md)）は、ジョブの `actor` と `actorNames` から作る。`memberName` は `names.member`、トークンの変種の `token` は `names.token`
- 確定が操作履歴へ渡す `AuditActor.from` の入力（`AuditActorSource`）は、ジョブの `actor` と `actorNames` から作る。`appToken` と `memberToken` の `token` は `names.token`、`appToken` の `issuerName` と `memberToken` の `memberName` は `names.member`。適用済みとしての記録は、読んだ `RecordedActor` から同じく作る

### 操作履歴の種類

反映の確定と適用済みとしての記録は、追記する `SchemaVersionRecord` の `change.kind` から、操作履歴の種類を一対一に決める。

| `change.kind` | 操作履歴の種類 |
| --- | --- |
| `applied` | `schema.migrations_applied` |
| `reflected` | `schema.reflected` |
| `direct` | `schema.pushed` |
| `recorded` | `schema.migrations_marked_applied` |

### 終了の書き込み

確定のほかで、ジョブを終端にする書き込み。ジョブを失敗（`failed`）または事前確認の完了（`checked`）で終える。終端にしたジョブの保存、そのジョブの準備領域の破棄、`mode` が `commit` のジョブでは `WorkspaceSchema.endApply` の結果の保存を、一つの UnitOfWork で確定する。失敗の理由によらない。ユースケースは、振る舞いが返した値を確定するだけで、何を書くかを自分で決めない。反映と事前確認で、同じ書き込みを行う。

| 書き込み | 対象 | 回数 |
| --- | --- | --- |
| 終端にしたジョブの保存（`SchemaApplyJobRepository.save`） | すべて | 1 |
| `WorkspaceSchema.endApply` の結果の保存（`WorkspaceSchemaRepository.save`） | `mode` が `commit` のジョブ | 0 または 1 |
| 準備領域の破棄（`ContentStagingArea.discard(jobId)`、`MediaUsageRepository.discardStaged(stagingId)`） | すべて | 2 |

振る舞いに渡すジョブと `WorkspaceSchema` は、同じ UnitOfWork で読む。`stagingId` は「ほかのドメインとの対応」の「準備領域の識別子」が定める。

### 進行中のジョブの中断

進行中のジョブを `interrupted` で失敗させる手続き。対象は [../domains/schema.md](../domains/schema.md) の「SchemaApplyJob」の「中断」が定める。「止まった反映・事前確認を検出して失敗させる」と、Installation の「利用を再開する」が行う。

| 行うもの | `findInProgress` の `progressedBefore` | `WriteScope` の `condition` |
| --- | --- | --- |
| 止まった反映・事前確認を検出して失敗させる | `ApplyStall.before(now)` | `active` |
| 利用を再開する（Installation） | `null` | `none` |

- 対象は、`SchemaApplyJobRepository.findInProgress` を、`limit` を `SchemaLimits.inProgressLimit` にして、`PartitionReader.workspace(workspaceId)` のリポジトリで読む。`after` に、読んだ列の最後のジョブの位置を渡して次を読み、空の列が返れば終える
- 読んだジョブ一件ごとに、そのワークスペースの区画の一つの UnitOfWork を開き、ジョブを読み直して、`SchemaApplyJob.fail`（`interrupted`）の結果で「終了の書き込み」を行う。書き込みは 4 回以下

### 何もせずに終える場合

`schema.apply_requested` の消費（「反映・事前確認を準備する」「反映を確定する」）、「反映・事前確認を失敗させる」、「止まった反映・事前確認を検出して失敗させる」は、[../domains/index.md](../domains/index.md) の「イベントの消費者と定期ジョブ」の規則に従う。対象の集約が無いことは、「ジョブを引く」が `null` を返すことで確かめる。区画の `Workspace` の状態は `WorkspaceRepository.find` で読む。

### 適用状況の確認のコマンド

適用状況の確認のコマンド（SMIG-04）は、「適用状況を確認する」「スキーマの版の記録を一覧する」「反映の結果を一覧する」の三つのユースケースを呼び、結果を合わせて出力にする。適用中の反映は「適用状況を確認する」が、終えた反映の結果は「反映の結果を一覧する」が返す。手元のスキーマの識別子は、CMS に接続しない「スキーマの識別子を求める」で求める。コマンドが示す内容は、SMIG-04 に従う。

### イベントの消費者と定期ジョブ

イベント `schema.apply_requested` の消費者は、「ジョブを引く」で読んだジョブの状態で、呼ぶユースケースを決める。`preparing` は「反映・事前確認を準備する」、`prepared` は「反映を確定する」。「反映・事前確認を準備する」が `prepared` のジョブを返した消費は、同じ消費の中で、続けて「反映を確定する」を呼ぶ。一回の消費が確定する UnitOfWork は、準備の一つと、続けて呼ぶ確定の一つまで。準備または確定を進められないジョブは「反映・事前確認を失敗させる」で失敗させる。終端のジョブには何もしない。定期ジョブ「止まった反映の検出」は「止まった反映・事前確認を検出して失敗させる」を呼ぶ。

## ほかのドメインとの対応

直接反映の要求、準備と確定は、Content の `ContentMigration`・`MediaUsages`・`ContentStagingArea`（[../domains/content.md](../domains/content.md)）、Media の `MediaUsageRepository`（[../domains/media.md](../domains/media.md)）、Form の `NotificationReferenceCheck`・`FormSettings.planSchemaChange`（[../domains/form.md](../domains/form.md)）、Token の `Token.applyScopeChange`（[../domains/token.md](../domains/token.md)）を使う。Schema のドメイン層は、これらのドメインを知らない。受け渡しは、このファイルのユースケースが行い、対応は、この節だけが定める。

| 向き | Schema の値 | 受け口 |
| --- | --- | --- |
| Schema → Content | 計画（`ApplyPlan`） | `MigrationInput.plan`、`ContentMigration.uniqueFields`、`withDuplicates` |
| Schema → Content | 反映前の `WorkspaceSchema.modelIds` | `MigrationInput.modelIds` |
| Schema → Content | ジョブの `actor` と `actorNames` | `MigrationInput.by`（`Performer`。作り方は「実行者の名称」） |
| Schema → Content | 変換した日時（`Clock.now`）と、反映後のスキーマの版（`plan.nextSchemaVersion`） | `MigrationInput.mark`（`MigrationMark`） |
| Schema → Content | `ValueMigration.apply`、`ValueValidation.validateLevel` | `ContentMigration.migrateContent` と `migrateRevision`、`ContentValidation.inspect` が呼ぶ |
| Content → Schema | `ContentMigration.accumulate` と `withDuplicates` の結果（`ConversionTally`） | 準備の `phase.tally` と、`Preparation.outcome` の入力 |
| Content → Media | `MediaUsages.ofStagedContents`、`ofStagedRevisions` の結果（`HolderUses` の列） | `MediaUsageRepository.stage` の `entries` |
| Schema → Content | ジョブの ID | `ContentStagingArea.promote` の `jobId` |
| Schema → Content | ジョブの ID、確定の操作の ID（`NewIds.operationId`）、ジョブの `actor`、反映後の版、反映後の `WorkspaceSchema.modelIds` | `PublicationLedgerRepository.appendStaged` の `jobId`、`MigrationRecordHeader`（操作の ID、実行者、スキーマの版）、`modelIds`。記録の中身は Content が決める |
| Schema → Form | 計画の `forms`（`FormChanges`）と `nextSchema`（直接反映の要求の検証では `NotificationReferenceProbe` の引数）、反映前の `WorkspaceSchema.formIds`、`FormSettingsRepository.findAll` と `PendingFormSettingsRepository.findAll` の結果 | `NotificationReferenceCheck.afterApply` の `next`、`forms`、`formIds`、`settings`、`pending` |
| Form → Schema | `NotificationReferenceCheck.afterApply` の結果 | `ApplyEffects.brokenNotificationReferences`、直接反映の要求の検証では `NotificationReferenceProbe` の戻り値 |
| Schema → Form | `CommitSummary` の `createdForms`・`removedForms` と、そのフォームの定義、`FormSettingsRepository.findAll` と `PendingFormSettingsRepository.findAll` の結果 | `FormSettings.planSchemaChange` の `added`・`removed`・`settings`・`pending` |
| Schema → Token | `CommitSummary` の `createdModels`・`removedModels`・`createdForms`・`removedForms` | `Token.applyScopeChange` の `ScopeChange`（`{ models: { created: createdModels, removed: removedModels }, forms: { created: createdForms, removed: removedForms } }`） |

- Content の報告（`ContentMigrationReport`）、変換結果（`StagedContent`、`StagedRevision`）、公開の変更の記録に載せる値は、[../domains/content.md](../domains/content.md) の「ContentMigration」が定める。Schema のユースケースは、変換結果を、中身を読まずに `ContentStagingArea.putContents` と `putRevisions` へ渡す
- どの設定を作り、紐付け、削除済みにするかと、通知定義を検証する設定の選び方は Form が、範囲の紐付けと除外は Token が定める。ユースケースは、読んだ値をそのまま渡し、結果を一括のメソッドで書く

### 準備領域の識別子

| 準備領域 | 識別子 |
| --- | --- |
| Content の `ContentStagingArea` | ジョブの ID（`jobId`） |
| Media の `MediaUsageRepository` の準備領域 | `stagingId`。ジョブの ID の値の文字列 |

反映と事前確認の一回ごとに、準備領域が分かれる。`stagingId` の作り方は、この表だけが定める。

### 走査と走査の位置

[../domains/schema.md](../domains/schema.md) の「Preparation」の走査は、次のポートで行う。

| Schema の語 | ポート |
| --- | --- |
| コンテンツの走査の一区切り | `ContentRepository.scan`（`after` は走査の位置、`limit` は `SchemaLimits.preparationChunk`） |
| 履歴の版の走査の一区切り | `RevisionRepository.scan`（同上） |
| 走査の位置 | `ContentStagingArea.cursor` が返す `lastContentId` と `lastRevisionId` |
| 区画の最大の ID と件数 | `ContentRepository.lastId`・`count`、`RevisionRepository.lastId`・`count` |
| 区切りの前に準備領域へ書き終えていた件数（`ConversionStep.staged`） | `ContentStagingArea.cursor` が返す `contents` と `revisions` の和 |
| 準備領域への書き込み | `ContentStagingArea.putContents`・`putRevisions`、`MediaUsageRepository.stage` |

### 確定しなかった理由の分類

確定の UnitOfWork が確定しなかった理由を、Schema の `CommitObstacle` へ写す。写した後の扱いは、[../domains/schema.md](../domains/schema.md) の「確定できない場合の扱い」が定める。

| 確定しなかった理由 | `CommitObstacle` |
| --- | --- |
| `ConflictError`（`STALE_VERSION`）。`WorkspaceSchema`、ジョブ、`FormSettings`、紐付け待ちの設定、トークンのどれの競合も同じ。`PublicationLedgerRepository.appendStaged` が返すものを除く | `conflict` |
| `ContentStagingArea.promote` の `ConflictError`（`STAGING_INCOMPLETE`） | `stagingMismatch` |
| `ContentStagingArea.promote` の `ConflictError`（`UNIQUE_VALUE_TAKEN`、`REFERENCE_NOT_PUBLISHED`） | `publicationIntegrity` |
| `MediaUsageRepository.promoteStaged` の `ConflictError`（`MEDIA_NOT_PUBLIC`） | `publicationIntegrity` |
| `MediaUsageRepository.promoteStaged` の `ConflictError`（`MEDIA_NOT_FOUND`） | `mediaMissing` |
| `ContentStagingArea.promote` の `SystemError`（`STAGING_CAPACITY_EXCEEDED`） | `capacityExceeded` |
| `ConflictError`（`WORKSPACE_NOT_ACTIVE`） | 分類しない。「何もせずに終える場合」に従う |
| 上のどれにも当たらない理由 | `unclassified` |

- 準備の `putContents` と `putRevisions` の `SystemError`（`STAGING_CAPACITY_EXCEEDED`）は、失敗の理由 `capacityExceeded` にする

## 反映の要求

### 反映を要求する（`requestSchemaApply`）

#### 概要

手元のスキーマと移行定義の列を受け取り、検証して、反映を受け付ける。変換と確定は、`schema.apply_requested` の消費者が行う。

- **シナリオ**: SMIG-06、SMIG-07、SMIG-09、SMIG-10、SMIG-12、SMIG-13、SDEV-04、SDEV-06
- **認可**: 実行者はトークンだけ。許可 `schema_apply`、権限 `schema:change`
- **イベント**: `schema.apply_requested`

#### 入出力

- **入力**: `jobId`（要求した側が作ったジョブの ID）、`baseSchemaVersion`（前提とする現行スキーマの版）、`schema`（スキーマの全体）、`migrations`（移行定義の列の全体）。「要求の入力」に従う。ユースケースは、`source` が `migrations`、`mode` が `commit` の `ApplyRequest` にする
- **出力**: 受け付けたジョブ、再送で引いたジョブ、差分が無いことと現在の版のいずれか

#### 使用するドメインの振る舞い・ポート

- 「ジョブを引く」、`SchemaApplyJob.isReplayOf`
- `WorkspaceSchemaRepository.find`、`save`
- `WorkspaceSchema.planApply`、`Preparation.start`、`SchemaApplyJob.accept`、`WorkspaceSchema.beginApply`
- `SchemaApplyJobRepository.insert`
- `authorizeTokenOperation` の出力、`AccountRepository.findById`（「実行者の名称」。`PartitionReader.installation()`）
- `WorkspaceContext.collectEvents`
- `Clock.now`

#### トランザクション境界

- **UnitOfWork の要否**: 必要。`active` の `WriteScope`
- `planned` の場合: 一つの UnitOfWork に、ジョブの `insert`、`beginApply` の結果の `WorkspaceSchema` の保存、`schema.apply_requested` の記録を含める。書き込みは 3 回
- `unchanged`、`resolvesFailure`、`rejected` の場合: 「拒否した要求と差分の無い要求」
- 再送の場合: 何も書かない

### 直接反映を要求する（`requestDirectSchemaApply`）

#### 概要

移行定義を使わず、現行スキーマとの差分から求めた操作で反映する要求を受け付ける。

- **シナリオ**: SDEV-05、SDEV-04、SDEV-06
- **認可**: 実行者はトークンだけ。許可 `schema_apply`、権限 `schema:change`
- **イベント**: `schema.apply_requested`

#### 入出力

- **入力**: `jobId`、`baseSchemaVersion`、`schema`、`lossAcknowledged`（確定の前に確認を求める対象を確認したかどうか）。「要求の入力」に従う。ユースケースは、`source` が `direct`、`mode` が `commit` の `ApplyRequest` にする
- **出力**: 「反映を要求する」と同じ

#### 使用するドメインの振る舞い・ポート

- 「反映を要求する」と同じもの
- `WorkspaceSchema.planApply` に渡す `NotificationReferenceProbe`: `FormSettingsRepository.findAll`、`PendingFormSettingsRepository.findAll`、`NotificationReferenceCheck.afterApply`（「ほかのドメインとの対応」）

#### トランザクション境界

- **UnitOfWork の要否**: 必要。`active` の `WriteScope`
- 「反映を要求する」と同じ。運用設定は、`WorkspaceSchema` と同じ UnitOfWork で読む

### 適用の事前確認を要求する（`requestSchemaDryRun`）

#### 概要

反映と同じ検証を行い、データを変えずに結果を求める事前確認を受け付ける。変換と集計は、`schema.apply_requested` の消費者が行う。

- **シナリオ**: SMIG-05、SMIG-07、SMIG-09、SMIG-13
- **認可**: 実行者はトークンだけ。許可 `schema_apply`、権限 `schema:change`
- **イベント**: `schema.apply_requested`

#### 入出力

- **入力**: `jobId`、`baseSchemaVersion`、`schema`、`migrations`。「要求の入力」に従う。ユースケースは、`source` が `migrations`、`mode` が `dryRun` の `ApplyRequest` にする
- **出力**: 受け付けたジョブ、または再送で引いたジョブ

#### 使用するドメインの振る舞い・ポート

- 「ジョブを引く」、`SchemaApplyJob.isReplayOf`
- `Preparation.start`、`SchemaApplyJob.accept`、`SchemaApplyJobRepository.insert`
- `WorkspaceSchemaRepository.find`
- `WorkspaceSchema.planApply`
- `authorizeTokenOperation` の出力、`AccountRepository.findById`（「実行者の名称」。`PartitionReader.installation()`）
- `WorkspaceContext.collectEvents`
- `Clock.now`

#### トランザクション境界

- **UnitOfWork の要否**: 必要。前提の版の `WriteScope`
- `planned` の場合: 一つの UnitOfWork に、ジョブの `insert` と、`schema.apply_requested` の記録を含める。書き込みは 2 回。`WorkspaceSchema` を保存しない
- `rejected` の場合: 「拒否した要求と差分の無い要求」
- 再送の場合: 何も書かない

### 移行定義を適用済みとして記録する（`recordMigrationsAsApplied`）

#### 概要

未適用の移行定義の先頭から指定した移行定義までを、データを変えずに、適用済みとして記録する。

- **シナリオ**: SMIG-08
- **認可**: 実行者はトークンだけ。許可 `schema_apply`、権限 `schema:change`
- **操作履歴**: `schema.migrations_marked_applied`（「操作履歴の種類」）

#### 入出力

- **入力**: 前提とする現行スキーマの版、移行定義の列の全体、記録する範囲の終端の移行定義の名前（`RecordRequest`）
- **出力**: 記録した移行定義の名前と、記録後の版

#### 使用するドメインの振る舞い・ポート

- `WorkspaceSchemaRepository.find`、`save`
- `WorkspaceSchema.planRecord`、`recordApplied`
- `SchemaVersionRecord.ofRecord`、`SchemaVersionRecordRepository.append`
- `AuditActor.from`、`AuditLogEntry.record`、`AuditLogRepository.append`
- `authorizeTokenOperation` の出力、`AccountRepository.findById`（「実行者の名称」。`PartitionReader.installation()`）
- `IdGenerator.generate`、`Clock.now`

#### トランザクション境界

- **UnitOfWork の要否**: 必要。`active` の `WriteScope`
- `planned` の場合: 一つの UnitOfWork に、`recordApplied` の結果の `WorkspaceSchema` の保存、`SchemaVersionRecord` の追記、`AuditLogEntry` の追記を含める。書き込みは 3 回
- `rejected` の場合: 「拒否した要求と差分の無い要求」

## 反映の進行

### 反映・事前確認を準備する（`prepareSchemaApply`）

#### 概要

`schema.apply_requested` の消費者が、`preparing` のジョブに呼ぶ。一回の呼び出しは、ジョブの準備（`Preparation`）を一区切り進める。準備の段階（`phase`）で、行うことが決まる。反映と事前確認は、同じ準備を行い、同じジョブのポートに保存する。

- **シナリオ**: SMIG-05、SMIG-06、SMIG-07、SMIG-10、SDEV-05、イベント `schema.apply_requested` の消費
- **認可**: 実行者は `system` だけ
- **イベント**: `schema.apply_requested`

#### 入出力

- **入力**: `WorkspaceId`、`SchemaApplyJobId`（イベントの内容）
- **出力**: 呼び出しを終えた後のジョブ、または引いたジョブ

#### 使用するドメインの振る舞い・ポート

- 「ジョブを引く」、`WorkspaceRepository.find`
- `WorkspaceSchemaRepository.find`（反映前の `modelIds` と `formIds`。事前確認では `failIfSchemaChanged` にも渡す）
- 進行: `Preparation.fixTarget`、`recordConverted`、`nextTallyCursor`、`recordTallied`、`outcome`、`ConversionTarget.of`、`clip`
- 進行の保存: `SchemaApplyJob.advance`、`complete`、`failIfSchemaChanged`（事前確認）、`SchemaApplyJobRepository.save`
- `blocked` で失敗した反映: `WorkspaceSchemaRepository.save`、`WorkspaceSchema.endApply`
- 変換: 「走査と走査の位置」のポート、`ContentRepository.findByIds`、`ReferenceTarget.of`、`ContentMigration.migrateContent`、`migrateRevision`、`MediaUsages.ofStagedContents`、`ofStagedRevisions`
- 集計: `ContentStagingArea.listContentReports`、`listRevisionReports`、`findDuplicates`、`ContentMigration.accumulate`、`uniqueFields`、`withDuplicates`、`FormSettingsRepository.findAll`、`PendingFormSettingsRepository.findAll`、`NotificationReferenceCheck.afterApply`
- 破棄: `ContentStagingArea.discard`、`MediaUsageRepository.discardStaged`
- `WorkspaceContext.collectEvents`、`Clock.now`

段階ごとに行うこと。進行の値、段階の移り方、変換の一区切りの範囲と走査の順は、[../domains/schema.md](../domains/schema.md) の「Preparation」が決める。

| `phase` | 一回の呼び出しが行うこと |
| --- | --- |
| `converting` | 変換の一区切りを行い、結果を `Preparation.recordConverted` に渡す。変換に渡す `MigrationInput` は「ほかのドメインとの対応」のとおりに作る。`targets` は、対象の内容と版が参照するコンテンツを `findByIds` で読んだ結果から作る。`migrateRevision` の `modelId` は、版のコンテンツを `findByIds` で読んで得る |
| `tallying` | `phase.cursor` の位置から、`SchemaLimits.preparationChunk` 件以下の報告を読み、`phase.tally` に `ContentMigration.accumulate` で畳み込む。`Preparation.nextTallyCursor` が位置を返せば、`recordTallied` で記録する。`null` を返せば、準備を終える |

準備を終える消費は、`uniqueFields` の項目ごとの `findDuplicates` の結果を `withDuplicates` に渡し、`NotificationReferenceCheck.afterApply` に「ほかのドメインとの対応」の値を渡して、`Preparation.outcome` で作った `PreparationOutcome` を `SchemaApplyJob.complete` に渡す。

#### トランザクション境界

- **UnitOfWork の要否**: 必要。一回の呼び出しは、一つの UnitOfWork だけを確定する。反映のジョブは `active` の `WriteScope`。事前確認のジョブは、変換、集計、`checked` にする完了の UnitOfWork を前提の版の `WriteScope` で、`failIfSchemaChanged` が返した失敗の「終了の書き込み」を `active` の `WriteScope` で開く
- 変換の UnitOfWork: 区切りの `putContents` または `putRevisions`、`MediaUsageRepository.stage`、`advance` の結果のジョブの保存、続きを始める `schema.apply_requested` の記録。区切りの変換結果と使用箇所は、それぞれ一回の呼び出しに渡す。書き込みは 4 回以下で、区切りの件数に比例しない。変換の対象を決める読み取りは、UnitOfWork の前に行う
- 集計の UnitOfWork: ジョブの保存と、続きを始める `schema.apply_requested` の記録。書き込みは 2 回。報告の読み取りは、回数に数えない
- 完了の UnitOfWork: `complete` が `prepared` を返した場合は、そのジョブの保存だけ。`checked`、または `blocked` の `failed` を返した場合は、「終了の書き込み」。書き込みは 4 回以下。最後の区切りの報告の読み取りと重複の確認は読み取りで、回数に数えない
- 事前確認の消費は、どの UnitOfWork でも、その前に読んだ `WorkspaceSchema` を `failIfSchemaChanged` に渡す。失敗したジョブが返れば、「終了の書き込み」だけを行う
- 準備領域への書き込みが容量の上限を超えた場合は、「反映・事前確認を失敗させる」を、別の UnitOfWork で呼ぶ（「確定しなかった理由の分類」）

### 反映を確定する（`commitSchemaApply`）

#### 概要

準備済み（`prepared`）のジョブを、一つの UnitOfWork で確定する。現行スキーマを置き換え、ジョブを成功で終え、版の記録を追記し、コンテンツ、履歴の版、メディアの使用箇所を準備領域の変換結果へ切り替え、運用設定とトークンの範囲を紐付ける。

- **シナリオ**: SMIG-06、SMIG-07、SMIG-10、SMIG-13、SDEV-04、SDEV-05、SDEV-06、イベント `schema.apply_requested` の消費
- **認可**: 実行者は `system` だけ
- **操作履歴**: `schema.migrations_applied`、`schema.reflected`、`schema.pushed` のいずれか一つ（「操作履歴の種類」）。紐付けたフォームとトークンごとに `bootstrap.setting_bound`
- **イベント**: `schema.applied`、`content.publication_changed`

#### 入出力

- **入力**: `WorkspaceId`、`SchemaApplyJobId`
- **出力**: 成功で終えたジョブ（`SucceededJob`）、または引いたジョブ

#### 使用するドメインの振る舞い・ポート

- 「ジョブを引く」
- `WorkspaceSchemaRepository.find`、`save`、`WorkspaceSchema.requiredIds`、`commit`
- `SchemaApplyJob.succeed`（`commit` が返した `success`）、`SchemaApplyJobRepository.save`
- `SchemaVersionRecord.ofCommit`、`SchemaVersionRecordRepository.append`
- `WorkspaceRepository.find`
- `IdGenerator.generate`（新しい `ModelId` と `FormId`、確定の操作の ID、操作履歴の ID）
- `ContentStagingArea.promote`、`PublicationLedgerRepository.appendStaged`、`ContentEvents.publicationChanged`
- `MediaUsageRepository.promoteStaged`
- `FormSettingsRepository.findAll`、`PendingFormSettingsRepository.findAll`、`FormSettings.planSchemaChange`、`FormSettingsRepository.applySchemaChange`、`PendingFormSettingsRepository.deleteMany`
- `TokenRepository.findSelectionScoped`、`saveMany`、`Token.applyScopeChange`、`Token.designation`
- `AuditActor.from`、`AuditLogEntry.record`、`AuditLogRepository.appendMany`
- `CommitObstacle.failureOf`
- `WorkspaceContext.collectEvents`
- `Clock.now`

新しい ID は、`WorkspaceSchema.requiredIds` が返した個数の `ModelId` と `FormId` と、確定の操作の ID 一つを `IdGenerator.generate` で作り、`NewIds` として `commit` に渡す（[../domains/schema.md](../domains/schema.md) の「モデルとフォームの ID の割り当て」）。`SchemaVersionRecord.ofCommit` には、`commit` に渡した準備済みのジョブと、`commit` が返した `WorkspaceSchema` を渡す。

一括のメソッドで行う書き込みは、次のとおり。

| 書き込み | 一括のメソッド | 渡すもの |
| --- | --- | --- |
| `FormSettings` の作成・紐付け・削除済みへの遷移 | `FormSettingsRepository.applySchemaChange`（Form のポート） | `FormSettings.planSchemaChange` の結果。三つの列がすべて空の反映では呼ばない |
| 紐付け待ちの設定の削除 | `PendingFormSettingsRepository.deleteMany`（Form のポート） | `FormSettingsSchemaChange.bound` の要素の `pending` の列。`bound` が空の反映では呼ばない |
| 範囲が変わったトークンの保存 | `TokenRepository.saveMany`（Token のポート） | `applyScopeChange` が範囲を変えたトークンと、`findSelectionScoped` が返した `expectedVersion` の組の列。`applyScopeChange` がそのまま返したトークンを含めない。0 件では呼ばない |
| 操作履歴 | `AuditLogRepository.appendMany`（Audit のポート） | 反映の一件と、紐付けたフォーム（`FormSettingsSchemaChange.bound`）・トークン（`applyScopeChange` が `bound` を返したもの）ごとの `bootstrap.setting_bound` の列 |

`ContentStagingArea.promote` には、ジョブの ID を渡す。`PublicationLedgerRepository.appendStaged` には、ジョブの ID と、`MigrationRecordHeader`（`NewIds.operationId`、ジョブの `actor`、反映後の版）と、`commit` が返した `WorkspaceSchema` の `modelIds` を渡す。`content.publication_changed` は、`ContentEvents.publicationChanged`（同じ操作の ID）で作り、記録するかどうかは [../domains/content.md](../domains/content.md) の「PublicationLedgerRepository」の呼び方に従う。ユースケースは、変更の一覧、参照元の一覧、通し番号を扱わない。

#### トランザクション境界

- **UnitOfWork の要否**: 必要。`active` の `WriteScope`
- 一つの UnitOfWork に、次の書き込みとイベントの記録を含める。回数は、コンテンツ、履歴の版、フォーム、トークンの件数に比例せず、12 回以下

| 書き込み | 回数 |
| --- | --- |
| `commit` の結果の `WorkspaceSchema` の保存（`WorkspaceSchemaRepository.save`） | 1 |
| `succeed` の結果のジョブ（`succeeded`）の保存（`SchemaApplyJobRepository.save`） | 1 |
| `SchemaVersionRecord` の追記（`SchemaVersionRecordRepository.append`） | 1 |
| 準備領域の変換結果への切り替え（`ContentStagingArea.promote`） | 1 |
| 公開の変更の記録の追記（`PublicationLedgerRepository.appendStaged`） | 1 |
| メディアの使用箇所の切り替え（`MediaUsageRepository.promoteStaged`） | 1 |
| `FormSettings` の作成・紐付け・削除済みへの遷移（`FormSettingsRepository.applySchemaChange`） | 0 または 1 |
| 紐付け待ちの設定の削除（`PendingFormSettingsRepository.deleteMany`） | 0 または 1 |
| 範囲が変わったトークンの保存（`TokenRepository.saveMany`） | 0 または 1 |
| `AuditLogEntry` の追記（`AuditLogRepository.appendMany`） | 1 |
| `schema.applied` の記録 | 1 |
| `content.publication_changed` の記録 | 0 または 1 |

- UnitOfWork が確定しなかった場合は、理由を「確定しなかった理由の分類」で `CommitObstacle` に写し、`CommitObstacle.failureOf` が失敗の内容を返せば、「反映・事前確認を失敗させる」を別の UnitOfWork で呼ぶ

### 反映・事前確認を失敗させる（`failSchemaApply`）

#### 概要

準備または確定を進められない進行中のジョブを、理由を付けて失敗させる。「反映・事前確認を準備する」と「反映を確定する」が呼ぶ。

- **シナリオ**: SMIG-05、SMIG-06（容量や処理能力の上限で適用できない）、SMIG-14、SDEV-05、イベント `schema.apply_requested` の消費
- **認可**: 実行者は `system` だけ

#### 入出力

- **入力**: `WorkspaceId`、`SchemaApplyJobId`、失敗の内容。`capacityExceeded`、または準備済みのジョブでの `commitRejected` とその `cause`
- **出力**: `failed` のジョブ、または引いたジョブ

#### 使用するドメインの振る舞い・ポート

- 「ジョブを引く」
- `SchemaApplyJob.fail`、`SchemaApplyJobRepository.save`
- 反映: `WorkspaceSchemaRepository.find`、`save`、`WorkspaceSchema.endApply`
- `WorkspaceRepository.find`
- `ContentStagingArea.discard`、`MediaUsageRepository.discardStaged`
- `Clock.now`

#### トランザクション境界

- **UnitOfWork の要否**: 必要。`active` の `WriteScope`
- 一つの UnitOfWork で、「終了の書き込み」を行う。書き込みは 4 回以下

### 止まった反映・事前確認を検出して失敗させる（`failStalledSchemaApplies`）

#### 概要

定期ジョブが、名簿で開設済みのワークスペースごとに呼ぶ（[../domains/index.md](../domains/index.md) の「定期ジョブ」）。入力のワークスペースの区画の止まったジョブを、「進行中のジョブの中断」で失敗させる。止まったことの判定は `ApplyStall` が持つ。

- **シナリオ**: SMIG-14（反映が進行中のまま終わらない）、SMIG-05（事前確認が中断した）、定期ジョブ「止まった反映の検出」
- **認可**: 実行者は `system` だけ

#### 入出力

- **入力**: `WorkspaceId`（対象のワークスペース）
- **出力**: 失敗させたジョブの件数

#### 使用するドメインの振る舞い・ポート

- `ApplyStall.before`、`isStalled`
- `SchemaApplyJobRepository.findInProgress`（`progressedBefore` は `ApplyStall.before(now)`、`limit` は `SchemaLimits.inProgressLimit`）、`findById`、`save`、`SchemaApplyJob.fail`（`interrupted`）
- 反映: `WorkspaceSchemaRepository.find`、`save`、`WorkspaceSchema.endApply`
- `WorkspaceRepository.find`
- `ContentStagingArea.discard`、`MediaUsageRepository.discardStaged`
- `Clock.now`

#### トランザクション境界

- **UnitOfWork の要否**: 必要。入力のワークスペースの、`active` の `WriteScope`
- 「進行中のジョブの中断」のとおり。ジョブ一件ごとに一つの UnitOfWork で、書き込みは 4 回以下

## 確認

### 適用状況を確認する（`getMigrationStatus`）

#### 概要

手元の移行定義の列を、ワークスペースの適用済みの列と照らす。データを変えない。

- **シナリオ**: SMIG-04、SMIG-08、SMIG-09、SDEV-07
- **認可**: 実行者はトークンだけ。許可 `schema_apply`、権限 `schema:change`

#### 入出力

- **入力**: 移行定義の列の全体
- **出力**: 現行スキーマの版と識別子、適用済みの移行定義の列、`MigrationStatus`、適用中かどうか、適用中の反映のジョブ（`InProgressJob`）

#### 使用するドメインの振る舞い・ポート

- `WorkspaceSchemaRepository.find`、`SchemaApplyJobRepository.findById`
- `WorkspaceSchema.status`、`WorkspaceSchema.fingerprints`、`WorkspaceSchema.inProgressApply`

適用中の反映は、読んだ `WorkspaceSchema` の `state.jobId` のジョブを `findById` で読み、`WorkspaceSchema.inProgressApply` に渡して求める。「反映の適用中かを確かめる」も同じ。

#### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みがない

### 現行スキーマを取得する（`getCurrentSchema`）

#### 概要

ワークスペースの現行スキーマと、その版と識別子を返す。

- **シナリオ**: SDEV-07
- **認可**: 実行者は `member` とトークン。権限 `schema:change`、許可 `schema_apply`

#### 入出力

- **入力**: なし
- **出力**: 現行スキーマ、版、識別子（`SchemaFingerprints`）

#### 使用するドメインの振る舞い・ポート

- `WorkspaceSchemaRepository.find`
- `WorkspaceSchema.fingerprints`

#### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みがない

### 反映の適用中かを確かめる（`getSchemaApplyState`）

#### 概要

ワークスペースが適用中かどうかと、適用中の反映の進行を返す。コンテンツを変える操作を止めていることを、権限によらずメンバーに示すために使う。

- **シナリオ**: SMIG-14、SMIG-06
- **認可**: 実行者は `member` だけ。権限を必要としない

#### 入出力

- **入力**: なし
- **出力**: 適用中かどうか、適用中の反映のジョブの ID と進行（ジョブの `preparation` を渡した `Preparation.progress` の結果）

#### 使用するドメインの振る舞い・ポート

- `WorkspaceSchemaRepository.find`、`SchemaApplyJobRepository.findById`
- `WorkspaceSchema.inProgressApply`、`Preparation.progress`

#### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みがない

### モデルを一覧する（`listModels`）

#### 概要

現行スキーマのモデルを返す。管理画面が、ナビゲーションに示すモデルと、ワークスペースを開いたときの最初の画面を求めるために使う。

- **シナリオ**: AUTH-02、EDIT-01、EDIT-04
- **認可**: 実行者は `member` だけ。権限を必要としない

#### 入出力

- **入力**: なし
- **出力**: モデルの一覧

#### 使用するドメインの振る舞い・ポート

- `WorkspaceSchemaRepository.find`
- `Schema.displayName`

#### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みがない

### スキーマの版の記録を一覧する（`listSchemaVersionRecords`）

#### 概要

スキーマの版の記録を、新しい版から返す。

- **シナリオ**: SMIG-04、SMIG-14
- **認可**: 実行者は `member` とトークン。権限 `schema:change`、許可 `schema_apply`

#### 入出力

- **入力**: 件数の区切り（`Pagination`）
- **出力**: 版の記録（`SchemaVersionRecord`）の一覧と総件数

#### 使用するドメインの振る舞い・ポート

- `Pagination.create`
- `SchemaVersionRecordRepository.findPage`

#### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みがない

### 反映の結果を一覧する（`listSchemaApplyResults`）

#### 概要

終えた反映の結果（`mode` が `commit` の終端のジョブ）を、新しいものから返す。

- **シナリオ**: SDEV-07、SMIG-04、SMIG-14
- **認可**: 実行者は `member` とトークン。権限 `schema:change`、許可 `schema_apply`

#### 入出力

- **入力**: 件数の区切り（`Pagination`）
- **出力**: 反映の結果（`FinishedJob<"commit">`）の一覧と総件数

#### 使用するドメインの振る舞い・ポート

- `Pagination.create`
- `SchemaApplyJobRepository.findApplyResultsPage`

#### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みがない

### 受け付けた反映・事前確認の進行と結果を取得する（`getSchemaApplyJob`）

#### 概要

ジョブの ID で、ジョブ一件の進行と結果を返す。要求した側が、反映の進行状況と結果を追うことと、事前確認の結果を受け取ることに使う。

- **シナリオ**: SMIG-05、SMIG-06、SMIG-14、SDEV-05
- **認可**: 実行者は `member` とトークン。権限 `schema:change`、許可 `schema_apply`

#### 入出力

- **入力**: `SchemaApplyJobId`。境界での扱いは [../domains/common.md](../domains/common.md) の「Clock と IdGenerator」に従う
- **出力**: 引いたジョブ（`SchemaApplyJob`）、その進行（`Preparation.progress` の結果）、`checked` のジョブでは案内（その `effects` を渡した `Guidance.forEffects` の結果）

#### 使用するドメインの振る舞い・ポート

- 「ジョブを引く」
- `Preparation.progress`、`Guidance.forEffects`

#### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みがない

## CMS に接続しない操作

開発環境のコマンドが、HTTP API を通らずに呼ぶ。経路を持たない（[app-api.md](app-api.md) の「要求とユースケースの対応」）。保存を伴わず、ポートを使わない。CMS の反映時の検証と同じ関数を使う。実行者を持たず、認可は無い。UnitOfWork を開かない。入力のスキーマと移行定義の形は、境界のデコードが確かめる（[../domains/schema.md](../domains/schema.md) の「値オブジェクト」）。

### スキーマを検証する（`validateSchema`）

#### 概要

スキーマ自体の不整合を、すべて検出する。

- **シナリオ**: SDEV-02、SDEV-01、SMIG-07

#### 入出力

- **入力**: スキーマの全体
- **出力**: 不整合（`SchemaIssue`）の列。不整合ごとに、分類（`SchemaIssue.categoryOf`）と、`Guidance.forSchemaIssue` の案内を添える

#### 使用するドメインの振る舞い・ポート

- `SchemaValidation.validate`、`SchemaIssue.categoryOf`
- `Guidance.forSchemaIssue`

#### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みがない

### スキーマの識別子を求める（`computeSchemaFingerprints`）

#### 概要

スキーマ全体、モデルごと、フォームごとの識別子を求める。

- **シナリオ**: SDEV-03、APP-07、SMIG-04

#### 入出力

- **入力**: スキーマの全体
- **出力**: `SchemaFingerprints`、または不整合（`SchemaIssue` の列）

#### 使用するドメインの振る舞い・ポート

- `SchemaValidation.validate`
- `SchemaFingerprint.of`

#### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みがない

### 移行定義を生成する（`generateMigration`）

#### 概要

既存の移行定義を順に適用した構造と、スキーマのコンテンツの構造の差分から、操作の並びを生成する。判断が要る箇所は問いとして返し、呼び出し側が答えを加えて呼び直す。

- **シナリオ**: SMIG-01、SMIG-08、SMIG-09、SMIG-10、SMIG-12

#### 入出力

- **入力**: スキーマの全体、既存の移行定義の列の全体、判断（`Decision`）の組
- **出力**: `GenerationResult`（`needsDecisions` の問いごとに `Guidance.forQuestion` の案内を添える）、スキーマの不整合（`SchemaIssue` の列）、移行定義の不整合（`MigrationIssue` の列）のいずれか

#### 使用するドメインの振る舞い・ポート

- `SchemaValidation.validate`
- `MigrationSequence.issues`
- `StructureMigration.fold`（`ContentStructure.empty` から）
- `MigrationGeneration.generate`
- `Guidance.forQuestion`

#### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みがない。移行定義のファイルは、呼び出し側のコマンドが書く

### スキーマと移行定義の食い違いを検出する（`checkMigrations`）

#### 概要

移行定義を先頭から順に適用した構造を、スキーマのコンテンツの構造と比べ、移行定義の不整合と、移行定義ごとのデータを失い得る操作を返す。

- **シナリオ**: SMIG-03、SDEV-02、SMIG-02、SMIG-10

#### 入出力

- **入力**: スキーマの全体、移行定義の列の全体
- **出力**: 食い違う定義（`StructureDifference` の列）、移行定義の不整合（`MigrationIssue` の列）、データを失い得る操作（`LossyOperation` の列）。三つを分けて返す

#### 使用するドメインの振る舞い・ポート

- `MigrationSequence.issues`
- `StructureMigration.fold`（`ContentStructure.empty` から）
- `Schema.structureOf`、`ContentStructure.diff`
- `StructureMigration.lossyOperations`（移行定義の名前を渡す）

#### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みがない

### 変換規則を一覧する（`listConversionRules`）

#### 概要

値の種類の変換規則と、変換できる値の種類の組み合わせを返す。値の種類の変更と値の複写が、同じ規則を使う。

- **シナリオ**: SMIG-01、SMIG-02

#### 入出力

- **入力**: なし
- **出力**: `ConversionRule` の列

#### 使用するドメインの振る舞い・ポート

- `ValueConversion.rules`

#### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みがない
