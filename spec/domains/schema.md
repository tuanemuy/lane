# Schema

スキーマと移行定義の意味を定め、ワークスペースの現行スキーマ、版の記録、反映の進行を保つ。境界、依存方向、ドメインをまたぐ規則は [index.md](index.md) が定める。集約の ID、`Actor` と `TokenActor`、`Pagination`、`TransactionalRepository`、`SingletonRepository`、`WriteScope`、`UnitOfWorkProvider`、`IdGenerator`、各ドメインが公開する項目の型（`DomainKind`、`StatusItem`、`ResumeItemDefinition`、`ResumeItem`）、共通のエラーの理由は [common.md](common.md) が定め、このファイルはそれを使う。

Schema の集約（`WorkspaceSchema`、`SchemaVersionRecord`、`SchemaApplyJob`）とポートは、すべてワークスペースの区画にある（[index.md](index.md) の「区画」）。

コードは `packages/core/src/domain/schema/` に置く。スキーマ、移行定義、識別子、検証、差分からの生成、変換は、保存を伴わない純粋な値と関数で、CMS と、CMS に接続しない開発環境が同じものを使う。関数は入出力を持たず、同じ引数から同じ結果を返す。

## ユビキタス言語

| 英語名 | 日本語名 | 定義 |
| --- | --- | --- |
| Schema | スキーマ | ワークスペースのモデル、フォーム、画像の派生形式の定義の全体 |
| Model | モデル | コンテンツの定義。単一（`single`）または複数件（`collection`） |
| Form | フォーム | 受付の定義 |
| Field | 項目 | モデル、項目のまとまり、繰り返し、ブロックの種類、フォームが持つ値の定義 |
| Field kind | 値の種類 | 項目が持つ値の種類。`kind` で判別する |
| Level | 階層 | 項目が並ぶ場所。モデルの直下、項目のまとまりの中、繰り返しの要素の中、ブロックの種類の中 |
| Option | 選択肢 | 単一選択・複数選択の項目が取り得る値 |
| Block type | ブロックの種類 | ブロックの項目が許可する、項目の組 |
| Image derivative | 画像の派生形式 | 大きさ、画像形式、圧縮の品質の組。スキーマに一組だけ定義し、ワークスペースのすべての画像に適用する |
| Content structure | コンテンツの構造 | 移行定義が変える範囲。モデル・項目・選択肢・ブロックの種類のキー、単一と複数件の別、値の種類、必須、参照を許可するモデル、入れ子の構成 |
| Type shape | 型の形 | 識別子を求める範囲。コンテンツの構造、フォームの型の形、派生形式のキー |
| Fingerprint | 識別子 | 型の形から求める値。スキーマ全体、モデルごと、フォームごとに持つ |
| Unset | 未設定 | 項目に値が無いこと。値の組にその項目のキーが無いことで表す |
| Migration | 移行定義 | 名前と、操作の並び |
| Operation | 操作 | コンテンツの構造と保存済みの値に対する、一つの変更 |
| Unit | 適用の単位 | 補完値をまとめて入れる、操作の並びの一区切り。移行定義一つ、または直接反映の操作の全体 |
| Fill | 補完値 | 項目を加える、または必須にする操作が、未設定の値に入れる値 |
| Copy | 値の複写 | 同じ階層の別の項目の値を、未設定の項目へ写す操作 |
| Lossy operation | データを失い得る操作 | 既存の値・項目・コンテンツを取り除くか未設定にし得る操作 |
| Digest | 操作の並びの要約 | 移行定義の操作の並びから求める値。適用済みの移行定義との一致の判定に使う |
| Migrated structure | 移行定義による構造 | 適用済みの移行定義を、空の構造へ順に適用した結果 |
| Drift | 直接反映による食い違い | 現行スキーマのコンテンツの構造が、移行定義による構造と一致しないこと |
| Schema version | スキーマの版 | 現行スキーマと適用済みの移行定義の列の版を表す番号（`WorkspaceSchema.schemaVersion`）。進み方は「WorkspaceSchema」の不変条件が定める |
| Apply | 反映 | 要求したスキーマで現行スキーマを置き換えること。確定の有無が `commit` の要求が行う。操作の求め方（`migrations` / `direct`）を問わない |
| Migration apply | 移行定義による反映 | 操作の求め方が `migrations` の反映。未適用の移行定義の適用と、移行定義を必要としない定義の置き換え |
| Apply source | 操作の求め方 | 反映する操作を、移行定義から求める（`migrations`）か、現行スキーマとの差分から求める（`direct`）か |
| Apply mode | 確定の有無 | 反映を確定する（`commit`）か、データを変えずに結果だけを求める（`dryRun`）か。`commit` は反映が取る |
| Direct apply | 直接反映 | 操作の求め方が `direct` の反映。移行定義を作らずに、差分から求めた操作でスキーマを置き換える |
| Dry run | 適用の事前確認 | データを変えずに、反映の結果を求めること。確定の有無が `dryRun` の要求 |
| In-progress apply | 適用中の反映 | 受け付けて、確定も失敗もしていない反映。`mode` が `commit` の進行中のジョブ。`WorkspaceSchema` が、そのジョブの ID を一つだけ持つ |
| Apply result | 反映の結果 | 終えた反映の、成功または失敗の結果。`mode` が `commit` の終端のジョブ |
| Failed apply | 失敗した反映 | 失敗で終えた反映。`mode` が `commit` の `failed` のジョブ |
| Unresolved failed apply | 未解消の失敗した反映 | 解消していない、失敗した反映。`WorkspaceSchema` が、そのジョブの ID を `failedApply` に持つ |
| Preparation | 準備の一回 | 受け付けた反映・事前確認の、変換と集計の進行。計画、実行者と名称、要約、進行を持つ。ジョブが持つ |
| Job | ジョブ | 受け付けた反映・事前確認の一回を表す集約（`SchemaApplyJob`）。種類、準備の進行、終端の結果を持つ |
| Job ID | ジョブの ID | 受け付けた反映・事前確認の一回を指す ID。要求した側が作る |
| Record as applied | 適用済みとしての記録 | データを変えずに、未適用の移行定義を適用済みにすること |
| Guidance | 案内 | 問い、不整合、反映の結果とともに開発者に示す、次の操作の手がかり |

## エンティティ

### WorkspaceSchema

ワークスペースの区画に一つ。現行スキーマ、版、適用済みの移行定義の列、モデル・フォームの ID と、適用中の反映のジョブの ID、未解消の失敗した反映のジョブの ID を持つ。区画に一つなので、ID のフィールドを持たない。

```ts
type WorkspaceSchemaBase = Readonly<{
  schemaVersion: number;
  schema: Schema;
  modelIds: Readonly<Record<string, ModelId>>;
  formIds: Readonly<Record<string, FormId>>;
  appliedMigrations: readonly AppliedMigration[];
  migratedStructure: ContentStructure;
  failedApply: SchemaApplyJobId | null;
  version: number;
  createdAt: Date;
  updatedAt: Date;
}>;

type IdleWorkspaceSchema = WorkspaceSchemaBase & Readonly<{ state: Readonly<{ kind: "idle" }> }>;
type ApplyingWorkspaceSchema = WorkspaceSchemaBase &
  Readonly<{ state: Readonly<{ kind: "applying"; jobId: SchemaApplyJobId }> }>;
type WorkspaceSchema = IdleWorkspaceSchema | ApplyingWorkspaceSchema;

type AppliedMigration = Readonly<{
  name: MigrationName;
  digest: MigrationDigest;
  via: "applied" | "recorded";
  schemaVersion: number;
}>;
```

| フィールド | 制約 |
| --- | --- |
| `schemaVersion` | スキーマの版。0 以上の整数。作成時は 0。0 は、スキーマを一度も反映していないことを表す |
| `schema` | 必須。作成時はモデル・フォーム・派生形式が空 |
| `modelIds` / `formIds` | キーから ID への対応。`schema` のモデル・フォームのキーと過不足なく一致する |
| `appliedMigrations` | 適用した順。作成時は空。`schemaVersion` は、その移行定義を列に加えた版 |
| `migratedStructure` | `appliedMigrations` の移行定義を、空の構造へ順に適用した結果。作成時は空 |
| `state` | 作成時は `idle`。`applying` は、受け付けた反映が一つ進行中であることを表し、そのジョブの ID（`jobId`）だけを持つ。計画、進行、影響は、そのジョブ（「SchemaApplyJob」）が持つ |
| `failedApply` | 未解消の失敗した反映のジョブの ID。作成時は `null`。未解消の失敗した反映がある間だけ、そのジョブの ID を持つ |
| `version` | 楽観ロックの番号（[common.md](common.md) の「TransactionalRepository」）。作成時は 0。スキーマの版と別の値 |

#### 振る舞い

```ts
const WorkspaceSchema: {
  create(now: Date): IdleWorkspaceSchema;

  fingerprints(ws: WorkspaceSchema): SchemaFingerprints;
  matchFingerprints(ws: WorkspaceSchema, claimed: ClaimedFingerprints): readonly FingerprintMatch[];
  status(ws: WorkspaceSchema, local: readonly Migration[]): MigrationStatus;
  inProgressApply(ws: WorkspaceSchema, job: SchemaApplyJob | null): InProgressJob | null;

  planApply<S extends ApplySource, M extends ApplyMode>(
    ws: WorkspaceSchema,
    request: ApplyRequest<S, M>,
    now: Date,
    ...references: S extends "direct" ? [NotificationReferenceProbe] : []
  ): ApplyDecision<S, M>;
  planRecord(ws: WorkspaceSchema, request: RecordRequest): RecordDecision;

  beginApply(ws: IdleWorkspaceSchema, job: PreparingJob<"commit">, now: Date): ApplyingWorkspaceSchema;
  requiredIds(job: PreparedJob): RequiredIds;
  commit(
    ws: ApplyingWorkspaceSchema,
    job: PreparedJob,
    ids: NewIds,
    now: Date,
  ): WithEventDrafts<CommittedApply, SchemaAppliedEvent>;
  endApply(ws: ApplyingWorkspaceSchema, job: FailedJob<"commit">, now: Date): IdleWorkspaceSchema;
  clearFailedApply(
    ws: IdleWorkspaceSchema & Readonly<{ failedApply: SchemaApplyJobId }>,
    now: Date,
  ): IdleWorkspaceSchema;

  recordApplied(ws: IdleWorkspaceSchema, plan: RecordPlan, now: Date): IdleWorkspaceSchema;
};

type CommittedApply = Readonly<{
  workspaceSchema: IdleWorkspaceSchema;
  success: ApplySuccess;
  summary: CommitSummary;
}>;
```

| 振る舞い | 処理 |
| --- | --- |
| `create` | 空のスキーマ（`Schema.empty()`）、`schemaVersion` 0、`idle`、`failedApply` が `null` で作る |
| `fingerprints` | `SchemaFingerprint.of(ws.schema)` を返す |
| `matchFingerprints` | 添えられた識別子ごとに、現行の識別子と一致するかを返す。現行スキーマに無いモデル・フォームの識別子は不一致とする |
| `status` | 適用状況を返す。「適用状況」が定める |
| `inProgressApply` | 適用中の反映を返す。`job` は、`ws.state.jobId` のジョブを読んだ結果（無ければ `null`）。`ws.state` が `applying` で、`job` が進行中なら `job`、ほか（`idle`、`job` が終端、`job` が `null`）は `null`。`null` は適用中でないことを表す。`ws` とジョブを別に読む間にジョブが終端になる場合も、この判定で決まる |
| `planApply` | 反映・適用の事前確認の要求を検証し、進める場合は受け付けられる要求（`AcceptableRequest`）、差分が無い場合は `unchanged` または `resolvesFailure`、拒否する場合は理由を返す。「反映の要求の検証」が定める |
| `planRecord` | 適用済みとしての記録の要求を検証し、受け付ける場合は `RecordPlan` を返す |
| `beginApply` | `state` を `applying` にし、`jobId` を、渡したジョブの ID にする。`mode` が `commit` の、受け付けたばかりのジョブ（`SchemaApplyJob.accept` の結果）だけを受ける |
| `requiredIds` | `commit` が新しく割り当てる `ModelId` と `FormId` の個数を返す。「モデルとフォームの ID の割り当て」が定める |
| `commit` | 準備済みのジョブを確定する。`schema` を `plan.nextSchema` に、`schemaVersion` を `plan.nextSchemaVersion` に置き換え、`state` を `idle` に、`failedApply` を `null` にする。`plan` は `job.preparation.plan`。`plan.source` が `migrations` の場合は、`plan.migrations` を `via: "applied"` で `appliedMigrations` に加え、`migratedStructure` を進める。`direct` の場合は `appliedMigrations` と `migratedStructure` を変えない。`modelIds` / `formIds` を、渡された `ids` で、「モデルとフォームの ID の割り当て」に従って更新する。成功の内容（`success`）を返す。ジョブは変えない。`schema.applied` を発行する。ID を作らないので、同じ引数から同じ結果を返す |
| `endApply` | 失敗で終えたジョブの適用中を解く。`state` を `idle` に、`failedApply` を、渡したジョブの ID にする。`state`、`failedApply`、`version`、`updatedAt` のほかを変えない |
| `clearFailedApply` | 未解消の失敗した反映を解消する。`failedApply` を `null` にする。`failedApply`、`version`、`updatedAt` のほかを変えない。`planApply` が `resolvesFailure` の `workspaceSchema` を作るのに使う |
| `recordApplied` | `plan.migrations` を `via: "recorded"` で `appliedMigrations` に加え、`migratedStructure` を `plan.migratedStructure` にし、`schemaVersion` を 1 進める。`schema`、`modelIds`、`formIds` は変えない |

- `RecordPlan` は `planRecord` だけが作る
- `commit` と `endApply` に渡すジョブの ID が `ws.state.jobId` と違う呼び出しは `BusinessRuleError`（`SCHEMA_JOB_MISMATCH`）。何も変えない
- `commit` は `PreparedJob` だけを受ける。準備を終えていない反映は確定できない
- 適用中を解いて `idle` に戻す振る舞いは、`commit` と `endApply` だけ。`endApply` は、`mode` が `commit` の `failed` のジョブ（`SchemaApplyJob` の `fail`、または `blocked` を返す `complete` の結果）だけを受ける
- `recordApplied` は、同じ `ws` から `planRecord` が返した計画を受ける
- `failedApply` を変える振る舞いは、`commit`、`endApply`、`clearFailedApply` だけ。`beginApply` と `recordApplied` は保つ
- `beginApply`、`commit`、`endApply`、`clearFailedApply`、`recordApplied` は `version` を 1 進め、`updatedAt` を `now` にする

#### 不変条件

- `modelIds` のキーの集合は `schema.models` のキーの集合と一致する。`formIds` も `schema.forms` と一致する
- 一度割り当てた `ModelId` / `FormId` を、別のモデル・フォームに割り当てない
- `schema` は `SchemaValidation.validate` が不整合を返さない
- `migratedStructure` は `appliedMigrations` の列だけから決まる
- `schemaVersion` は減らない。`schema`、`appliedMigrations` のどちらかが変わると 1 進み、どちらも変わらなければ進まない
- 適用中の反映は、一つのワークスペースに高々一つ。`state` が持てるジョブの ID は一つだけ
- `applying` の `jobId` は、`mode` が `commit` で、進行中（`preparing` または `prepared`）のジョブを指す。`mode` が `commit` の進行中のジョブは、`applying` の `jobId` が指す一件だけ。`state` が `idle` であることと、`mode` が `commit` の進行中のジョブが一件も無いことは、同時に成り立つ
- `failedApply` が `null` でなければ、その ID は、`mode` が `commit` の `failed` のジョブを指す
- `applying` の間、`schema`、`schemaVersion`、`appliedMigrations`、`modelIds`、`formIds` は変わらない。そのジョブの `preparation.plan.baseSchemaVersion` は `schemaVersion` と等しい
- `state.jobId` と `failedApply` が指すジョブについての不変条件は、区画が `closing` になるまで成り立つ。区画が `closing` の間は、「利用終了の消去」がジョブを先に消去するので、`WorkspaceSchema` が、消去済みのジョブの ID を持ち得る。区画が `closing` のワークスペースで、`state.jobId` と `failedApply` からジョブを引くユースケースは無い

#### ライフサイクル

```text
create → idle ─ beginApply → applying ─ commit → idle（schemaVersion + 1、failedApply は null）
                                      └ endApply → idle（failedApply は失敗した反映のジョブの ID）
         idle ─ recordApplied → idle（schemaVersion + 1）
         idle（failedApply あり）─ clearFailedApply → idle（failedApply は null）
```

`create` は、ワークスペースの作成で一度だけ行う（「初期構成の宣言」）。ワークスペースの利用終了で消去する。適用中の反映の進行は、ジョブのライフサイクル（「SchemaApplyJob」）が定める。

#### 適用状況

```ts
type MigrationStatus =
  | Readonly<{ kind: "invalidMigrations"; issues: readonly MigrationIssue[] }>
  | Readonly<{
      kind: "compared";
      comparison:
        | (Extract<SequenceComparison, { kind: "consistent" }> & Readonly<{ recordable: readonly MigrationName[] }>)
        | Extract<SequenceComparison, { kind: "diverged" }>;
      drift: readonly StructureDifference[];
    }>;
```

- `status` は、手元の列（`local`）の不整合を先に確かめる。`MigrationSequence.issues(local)` が空でなければ、その結果を `issues` に持つ `invalidMigrations` を返す。この場合、列の比較、未適用の移行定義、記録できる移行定義を求めない
- 不整合が無ければ `compared` を返す
- `comparison` は `MigrationSequence.compare(ws.appliedMigrations, local)` の結果。`consistent` には `recordable` を添える
- `drift` は `ContentStructure.diff(ws.migratedStructure, Schema.structureOf(ws.schema))` の結果。空でなければ、直接反映による食い違いがある
- `recordable` は、適用済みとして記録できる範囲の終端になる移行定義の名前。未適用の移行定義の先頭から k 個（k は 1 以上）を `ws.migratedStructure` へ順に適用した結果が、現行スキーマのコンテンツの構造と一致する k について、k 番目の移行定義の名前を並べる

#### 反映の要求の検証

要求は、操作の求め方（`source`）と確定の有無（`mode`）の二つの軸を、別のフィールドで持つ。

```ts
type ApplySource = "migrations" | "direct";
type ApplyMode = "commit" | "dryRun";

type RequestBase = Readonly<{ baseSchemaVersion: number; schema: Schema }>;
type AnyApplyRequest =
  | (RequestBase & Readonly<{ source: "migrations"; mode: "commit"; migrations: readonly Migration[] }>)
  | (RequestBase & Readonly<{ source: "migrations"; mode: "dryRun"; migrations: readonly Migration[] }>)
  | (RequestBase & Readonly<{ source: "direct"; mode: "commit"; lossAcknowledged: boolean }>);
type ApplyRequest<S extends ApplySource = ApplySource, M extends ApplyMode = ApplyMode> = Extract<
  AnyApplyRequest,
  Readonly<{ source: S; mode: M }>
>;

const ApplyRequest: {
  digestOf(request: ApplyRequest): RequestDigest;
};

type ApplyDecision<S extends ApplySource, M extends ApplyMode> =
  | Readonly<{ outcome: "planned"; request: AcceptableRequest<M> }>
  | Readonly<{ outcome: "rejected"; rejection: ApplyRejection<S, M> }>
  | (M extends "commit"
      ?
          | Readonly<{ outcome: "unchanged"; schemaVersion: number }>
          | Readonly<{ outcome: "resolvesFailure"; schemaVersion: number; workspaceSchema: IdleWorkspaceSchema }>
      : never);

type AcceptableRequest<M extends ApplyMode = ApplyMode> = M extends ApplyMode
  ? Readonly<{ mode: M; requestDigest: RequestDigest; plan: PlanFor<M> }> & { readonly [acceptableRequestBrand]: true }
  : never;

type CommonRejection =
  | Readonly<{ reason: "staleVersion"; currentSchemaVersion: number }>
  | Readonly<{ reason: "schemaInvalid"; issues: readonly SchemaIssue[] }>;
type RejectionByMode = {
  commit: Readonly<{ reason: "applying"; jobId: SchemaApplyJobId }>;
  dryRun: never;
};
type RejectionBySource = {
  migrations:
    | Readonly<{ reason: "migrationsInvalid"; issues: readonly MigrationIssue[] }>
    | Readonly<{ reason: "sequenceDiverged"; divergences: readonly SequenceDivergence[] }>
    | Readonly<{ reason: "drift"; differences: readonly StructureDifference[]; recordable: readonly MigrationName[] }>
    | Readonly<{ reason: "structureMismatch"; differences: readonly StructureDifference[] }>;
  direct: Readonly<{
    reason: "lossNotAcknowledged";
    lossyOperations: readonly LossyOperation<"direct">[];
    forms: FormChanges;
    brokenNotificationReferences: readonly BrokenNotificationReference[];
  }>;
};
type ApplyRejection<S extends ApplySource = ApplySource, M extends ApplyMode = ApplyMode> =
  | CommonRejection
  | RejectionByMode[M]
  | RejectionBySource[S];

type PlanBase = Readonly<{
  baseSchemaVersion: number;
  nextSchemaVersion: number;
  baseStructure: ContentStructure;
  nextSchema: Schema;
  units: readonly Unit[];
  modelFates: readonly ModelFate[];
  forms: FormChanges;
}>;

type MigrationsPlan = PlanBase &
  Readonly<{ source: "migrations"; migrations: readonly Migration[]; lossyOperations: readonly LossyOperation<"migrations">[] }>;
type DirectPlan = PlanBase &
  Readonly<{ source: "direct"; operations: readonly MigrationOperation[]; lossyOperations: readonly LossyOperation<"direct">[] }>;
type ApplyPlan = MigrationsPlan | DirectPlan;
type PlanFor<M extends ApplyMode> = M extends "dryRun" ? MigrationsPlan : ApplyPlan;

type NotificationReferenceProbe = (next: Schema, forms: FormChanges) => readonly BrokenNotificationReference[];
```

- 要求の変種は、移行定義による反映、事前確認、直接反映の三つ
- 要求の `migrations` は、開発者の手元の移行定義の列の全体
- `ApplyRequest.digestOf` は、要求の内容の要約を求める。対象は、要求の `source`、`mode`、`baseSchemaVersion`、`schema`、`migrations` または `lossAcknowledged`。求め方は「正規化した JSON と要約」に従い、どの配列も並べ替えない
- `AcceptableRequest` は、受け付けられる要求。`planApply` が `planned` で返す値だけがこの型を持つ。`mode` は要求の `mode`、`requestDigest` は `ApplyRequest.digestOf(request)`、`plan` は検証した計画。`Preparation.start` はこの値だけを受けるので、要求と計画と要約は食い違わない
- 計画は `source` で形が決まり、`mode` を持たない。確定の有無は、`AcceptableRequest` の `mode` が表す。どちらの `mode` の要求も、受け付けるとジョブになる
- 直接反映の確定の前に確認を求める対象は、要求から求める計画（`planned` で返す `plan`）の `lossyOperations`、`forms.removed`（削除して受付を終了するフォーム）、`forms.changed` の `removedFields`、`references(plan.nextSchema, plan.forms)` が返す不整合となる通知定義の参照。キーを変えたフォームと項目は、`FormChanges` の削除と追加に現れる（「FormChanges」）。`lossAcknowledged` は、これらを確認したことを表す。`lossNotAcknowledged` の `forms` は計画の `forms`、`brokenNotificationReferences` は `references` の結果
- `NotificationReferenceProbe` は、反映後のスキーマと `FormChanges` から、不整合となる通知定義の参照（「反映の影響と失敗の理由」の `brokenNotificationReferences` と同じもの）を返す関数。同じ引数から同じ結果を返す。呼び出し側が、通知定義を持つ Form の判定から作って、`source` が `direct` の要求の検証に渡す（[../usecases/schema.md](../usecases/schema.md) の「ほかのドメインとの対応」）

`planApply` は次の順に確かめ、最初に当てはまった理由で拒否する。各段は、二つの軸のどちらか一方だけで、掛かる要求が決まる。掛かる要求は、`ApplyRejection<S, M>` の型と一致する。`schemaInvalid` と `migrationsInvalid` はスキーマ・移行定義の誤りで、既存データが条件を満たさないこと（`ApplyBlocker`）と別に示す。

| 順 | 条件 | 拒否の理由 | 掛かる要求 |
| --- | --- | --- | --- |
| 1 | `ws.state` が `applying` | `applying` | `mode` が `commit` |
| 2 | `request.baseSchemaVersion` が `ws.schemaVersion` と異なる | `staleVersion` | すべて |
| 3 | `SchemaValidation.validate(request.schema)` が不整合を返す | `schemaInvalid` | すべて |
| 4 | `MigrationSequence.issues(request.migrations)` が不整合を返す | `migrationsInvalid` | `source` が `migrations` |
| 5 | `MigrationSequence.compare(ws.appliedMigrations, request.migrations)` が `diverged` | `sequenceDiverged` | `source` が `migrations` |
| 6 | 未適用の移行定義が 1 個以上あり、「適用状況」の `drift` が空でない | `drift` | `source` が `migrations` |
| 7 | 現行スキーマのコンテンツの構造へ未適用の移行定義を順に適用した結果が、`Schema.structureOf(request.schema)` と一致しない | `structureMismatch` | `source` が `migrations` |
| 8 | `Schema.equals(ws.schema, request.schema)` が偽で、確認を求める対象のどれかが空でなく、`lossAcknowledged` が `false` | `lossNotAcknowledged` | `source` が `direct` |

どの理由にも当たらない `mode` が `commit` の要求は、差分が無ければ、`ws.failedApply` が `null` なら `unchanged`、`null` でなければ `resolvesFailure` を返す。差分が無いとは、`source` が `migrations` の要求では、`Schema.equals(ws.schema, request.schema)` が真で、未適用の移行定義が 0 個であること。`source` が `direct` の要求では、`Schema.equals(ws.schema, request.schema)` が真であること。どちらも成功で、`schemaVersion` は現在の版。現行スキーマ、スキーマの版、`state` を変えない。`unchanged` は、[index.md](index.md) の「変更の無い変更」に当たる。`resolvesFailure` は、未解消の失敗した反映を解消する変更で、`workspaceSchema` は `clearFailedApply(ws, now)` の結果。`mode` が `dryRun` の要求は、差分が無くても `planned` を返す。

それ以外は `planned` を返す。`request.plan` の各フィールドは次のとおり。

- `source` は要求の `source`
- `baseSchemaVersion` は要求の `baseSchemaVersion`。段 2 を通ったので `ws.schemaVersion` と等しい
- `nextSchemaVersion` は `baseSchemaVersion` に 1 を足した値で、反映を確定した後のスキーマの版。この値を求めるのは `planApply` だけ
- `baseStructure` は現行スキーマのコンテンツの構造。値への適用の出発点になる
- `migrations` は未適用の移行定義。`operations` は `MigrationGeneration.directOperations(baseStructure, Schema.structureOf(request.schema))` の結果
- `units` は、`migrations` では移行定義ごとの操作の並び、`direct` では操作の全体を一つにした並び
- `lossyOperations` は `StructureMigration.lossyOperations` の結果。`migrations` では、操作ごとに移行定義の名前を持つ
- `modelFates` は `StructureMigration.modelFates(baseStructure, units)` の結果
- `forms` は `Schema.diffForms(ws.schema, request.schema)` の結果

移行定義による反映は、未適用の移行定義のすべてを適用する。計画は、未適用の移行定義の途中までを持てない。

#### 適用済みとしての記録の検証

```ts
type RecordRequest = Readonly<{
  baseSchemaVersion: number;
  migrations: readonly Migration[];
  through: MigrationName;
}>;

type RecordDecision =
  | Readonly<{ outcome: "planned"; plan: RecordPlan }>
  | Readonly<{ outcome: "rejected"; rejection: RecordRejection }>;

type RecordRejection =
  | Extract<ApplyRejection<"migrations", "commit">, { reason: "applying" | "staleVersion" | "migrationsInvalid" | "sequenceDiverged" }>
  | Readonly<{ reason: "notPending"; through: MigrationName }>
  | Readonly<{ reason: "recordStructureMismatch"; differences: readonly StructureDifference[]; recordable: readonly MigrationName[] }>;

type RecordPlan = Readonly<{
  migrations: readonly Migration[];
  migratedStructure: ContentStructure;
}>;
```

記録する範囲は、未適用の移行定義の先頭から `through` まで。要求が持つのは範囲の終端だけで、先頭から始まらない範囲は指定できない。`planRecord` は、`applying`、`staleVersion`、`migrationsInvalid`、`sequenceDiverged` を `planApply` と同じ条件で確かめ、続けて次を確かめる。

- `through` が未適用の移行定義の名前でない場合は `notPending`
- `ws.migratedStructure` へ記録する範囲を順に適用した結果が、現行スキーマのコンテンツの構造と一致しない場合は `recordStructureMismatch`。`recordable` は `status(ws, request.migrations)` の `comparison` の `recordable`

`RecordPlan.migrations` は記録する範囲の移行定義、`migratedStructure` はそれを適用した結果。記録した移行定義の補完値、置き換え先、変換、値の複写は、既存データに適用しない。記録する移行定義は必ず 1 個以上あるので、適用済みとしての記録は「変更の無い変更」に当たらない。

#### 拒否の理由とエラー

ユースケースは、`planApply` と `planRecord` の拒否の理由を、次のエラーとして返す。理由からエラーへの対応は、この表だけが定める。理由コードは「エラーの理由コード」が定める。拒否の順は [index.md](index.md) の「利用できるワークスペース」に従う。どの拒否の理由も、`TokenAuthorization.authorize` の拒否より後に返す。

| 拒否の理由 | エラー | 添える内容 |
| --- | --- | --- |
| `applying` | `UnavailableError`（`SCHEMA_APPLYING`） | 適用中の反映のジョブの ID |
| `staleVersion` | `ConflictError`（`SCHEMA_VERSION_CHANGED`） | 現行の版 |
| `schemaInvalid` | `BusinessRuleError`（`SCHEMA_INVALID`） | `issues` |
| `migrationsInvalid` | `BusinessRuleError`（`MIGRATIONS_INVALID`） | `issues` |
| `sequenceDiverged` | `BusinessRuleError`（`MIGRATION_SEQUENCE_DIVERGED`） | `divergences` |
| `drift` | `BusinessRuleError`（`SCHEMA_DRIFT`） | `differences`、`recordable` |
| `structureMismatch` | `BusinessRuleError`（`SCHEMA_STRUCTURE_MISMATCH`） | `differences` |
| `lossNotAcknowledged` | `BusinessRuleError`（`SCHEMA_LOSS_NOT_ACKNOWLEDGED`） | `lossyOperations`、`forms`、`brokenNotificationReferences` |
| `notPending` | `BusinessRuleError`（`MIGRATION_NOT_PENDING`） | `through` |
| `recordStructureMismatch` | `BusinessRuleError`（`SCHEMA_RECORD_MISMATCH`） | `differences`、`recordable` |

#### モデルとフォームの ID の割り当て

```ts
type RequiredIds = Readonly<{ models: number; forms: number }>;
type NewIds = Readonly<{ modelIds: readonly ModelId[]; formIds: readonly FormId[]; operationId: OperationId }>;

type CommitSummary = Readonly<{
  schemaVersion: number;
  createdModels: readonly Readonly<{ key: ModelKey; id: ModelId }>[];
  removedModels: readonly Readonly<{ key: ModelKey; id: ModelId }>[];
  createdForms: readonly Readonly<{ key: FormKey; id: FormId }>[];
  removedForms: readonly Readonly<{ key: FormKey; id: FormId }>[];
}>;
```

- 反映前のモデルのうち、`plan.modelFates` が `kept` とするものは、`ModelId` を保ち、反映後のキーに対応させる
- `removed` とするモデルの `ModelId` は対応から外す
- 反映後のモデルのうち、保った `ModelId` を持たないものに、新しい ID を割り当てる。同じキーで削除して追加したモデルも新しい ID になる
- フォームはキーで比べる。`plan.forms.removed` の `FormId` を対応から外し、`plan.forms.added` に新しい ID を割り当てる
- `requiredIds(job)` は、新しい ID の要るモデルの数（`models`）とフォームの数（`forms`）を返す。`job.preparation.plan` の `modelFates`、`nextSchema`、`forms.added` だけから決まる。準備済みのジョブの計画は変わらないので、確定をやり直しても同じ個数になる
- `NewIds` は、呼び出し側が、`requiredIds` が返した個数の `ModelId` と `FormId` と、確定の操作の ID（`operationId`）を一つ作って `commit` に渡す。振る舞いは ID を作らない
- 確定の操作の ID は、反映の確定の一回を指す `OperationId`。作る主体は [index.md](index.md) の「公開内容の整合性と通し番号」に従う。ジョブの ID と別の値になる。公開内容が変わった反映では、公開の変更の記録と `content.publication_changed` がこの ID を持ち、`ApplySuccess.operationId` が示す。確定しなかった UnitOfWork の ID は残らず、確定をやり直すと新しい ID を作る
- `commit` は、新しい ID の要るモデルに `ids.modelIds` を、フォームに `ids.formIds` を、キーの昇順に、列の順で一つずつ割り当てる
- `ids.modelIds` または `ids.formIds` の個数が `requiredIds(job)` と違う呼び出し、列の中に同じ ID がある呼び出し、`ws.modelIds` / `ws.formIds` が既に持つ ID を含む呼び出しは、`BusinessRuleError`（`SCHEMA_NEW_IDS_MISMATCH`）。何も割り当てない

`CommitSummary` は、反映の確定が作成・削除したモデルとフォームの要約。フォームの運用設定とトークンの範囲を変える振る舞いへ渡す。受け口は [../usecases/schema.md](../usecases/schema.md) の「ほかのドメインとの対応」が定める。

#### 反映の影響と失敗の理由

```ts
type Sampled<T> = Readonly<{ total: number; items: readonly T[] }>;

type ApplyBlocker<D = unknown> = Readonly<{ content: ContentId; detail: D }>;

type CommitRejection = "stagingMismatch" | "publicationIntegrity" | "mediaMissing";

type PreparingFailure = Readonly<{ reason: "capacityExceeded" | "interrupted" }>;
type PreparedFailure =
  | Readonly<{ reason: "blocked"; blockers: Sampled<ApplyBlocker>; effects: ApplyEffects }>
  | Readonly<{ reason: "capacityExceeded" | "interrupted"; effects: ApplyEffects }>
  | Readonly<{ reason: "commitRejected"; cause: CommitRejection; effects: ApplyEffects }>;
type ApplyFailure = PreparingFailure | PreparedFailure;

type ApplyEffects = Readonly<{
  losses: readonly OperationLoss[];
  changedPublished: Sampled<Readonly<{ content: ContentId; change: "updated" | "unpublished" }>>;
  constraintViolations: Sampled<ConstraintViolation>;
  draftsWithErrors: Sampled<ContentId>;
  forms: FormChanges;
  brokenNotificationReferences: readonly BrokenNotificationReference[];
}>;

type OperationLoss = Readonly<{ operation: OperationRef; loss: LossKind; contents: Sampled<ContentId>; values: number }>;
type ConstraintViolation = Readonly<{ content: ContentId; paths: readonly ValuePath[] }>;
type BrokenNotificationReference<R = unknown> = Readonly<{ form: FormKey; field: FieldKey; reason: R }>;

type PreparationOutcome = Readonly<{ blockers: Sampled<ApplyBlocker>; effects: ApplyEffects }>;

type ApplySuccess = Readonly<{
  schemaVersion: number;
  operationId: OperationId;
  fingerprints: SchemaFingerprints;
  appliedMigrations: readonly MigrationName[];
  effects: ApplyEffects;
}>;
```

- `Sampled` は、該当する全件の件数（`total`）と、先頭 `SchemaLimits.sampleLimit` 件までの内容（`items`）を持つ。先頭は、コンテンツID の昇順で決める
- `ApplyBlocker` は、反映を失敗させる公開内容を運ぶ型。`content` はそのコンテンツ、`detail` は理由。Schema は `detail` を読まず、型引数の既定（`unknown`）で持つ。理由の変種の直和、どの公開内容を加えるか、一件のコンテンツの中の並べ方は、条件を決める Content が定める。`blockers` の件数と先頭は、ほかの `Sampled` と同じ
- `BrokenNotificationReference` は、不整合となる通知定義の参照を運ぶ型。`form` と `field` は反映後のスキーマのフォームと項目のキー、`reason` は理由。Schema は `reason` を読まず、型引数の既定（`unknown`）で持つ。理由の変種と判定は、通知定義を持つ Form が定める
- 結果を読む側（ユースケースと入口）は、`detail` と `reason` を、作った側の値のまま返す
- `ApplyFailure` は、反映の失敗の理由。`blocked` は、反映後の公開内容が条件を満たさないこと。`capacityExceeded` は、容量や処理能力の上限（準備領域の容量を含む）で進められないこと。`interrupted` は、進行が記録されないまま止まったこと。`commitRejected` は、切り替えが拒否されて確定できないこと。拒否の原因（`cause`）は「確定できない場合の扱い」が定める
- `PreparingFailure` は、準備の途中で失敗した反映の理由。`PreparedFailure` は、準備を終えた後に失敗した反映の理由で、準備が求めた影響（`effects`）を持つ。どちらの理由を持つかは、ジョブの `preparation` の段階と組にして、`FailedJob<"commit">` の型が決める
- `ApplyEffects.losses` は、データを失い得る操作ごとの件数。`contents` は、その操作で完全削除されるコンテンツ（件数と先頭の対象）。`values` は、その操作で公開内容、下書き、履歴の版が失う値の件数で、全件を数える。`removeModel` の `values` は、ほかのコンテンツから取り除く、そのモデルのコンテンツへのコンテンツ参照の値と本文のリンクの数。失うものの対象は、`operation` が指す操作の、モデル・項目・選択肢・ブロックの種類で示す
- `changedPublished` は、公開内容が変わるコンテンツ。`change` は [content.md](content.md) の「ContentMigration」の `accumulate` に従う
- `constraintViolations` は、反映後の制約を満たさない公開内容。一件が、一つのコンテンツと、そのコンテンツで制約を満たさない値の位置（`paths`）を持つ。`total` は、そのようなコンテンツの件数。`draftsWithErrors` は、変換後に入力エラーを持つ下書き。数える下書きは [content.md](content.md) の「ContentMigration」の `accumulate` に従う
- `forms` は計画の `forms`。`brokenNotificationReferences` は、不整合となる通知定義の参照
- `PreparationOutcome` は、準備の結果（「Preparation」の `outcome`）
- `ApplySuccess` は、成功した反映の内容。`commit` が、確定後の版と識別子、確定の操作の ID（`ids.operationId`）、`plan.source` が `migrations` の場合の移行定義の名前（`direct` では空）、準備済みのジョブの `effects` から作る
- 結果は、保存方式に依存する情報とトークンの値を含まない

#### 確定できない場合の扱い

準備済みの反映の確定の UnitOfWork が確定しなかった理由（`CommitObstacle`）から失敗の理由への対応は、この表だけが定める。確定が `ConflictError`（`WORKSPACE_NOT_ACTIVE`）で拒否された場合は分類せず、[index.md](index.md) の「イベントの消費者と定期ジョブ」に従う。ポートのエラーから `CommitObstacle` への対応は、[../usecases/schema.md](../usecases/schema.md) の「確定しなかった理由の分類」が定める。

```ts
type CommitObstacle = "conflict" | CommitRejection | "capacityExceeded" | "unclassified";

const CommitObstacle: {
  failureOf(
    obstacle: CommitObstacle,
  ):
    | Readonly<{ reason: "commitRejected"; cause: CommitRejection }>
    | Readonly<{ reason: "capacityExceeded" }>
    | null;
};
```

| `CommitObstacle` | 意味 | `failureOf` | 扱い |
| --- | --- | --- | --- |
| `conflict` | 並行して確定した書き込みとの競合。確定し直せば確定し得る | `null` | 反映を失敗させない |
| `stagingMismatch` | 準備領域が、区画のすべてのコンテンツと履歴の版の結果を過不足なく持たない | `commitRejected`（`cause` は `stagingMismatch`） | `SchemaApplyJob.fail` で失敗させる |
| `publicationIntegrity` | 切り替えた後の公開内容が、公開内容の整合性（[index.md](index.md) の「公開内容の整合性と通し番号」）を満たさない | `commitRejected`（`cause` は `publicationIntegrity`） | 同上 |
| `mediaMissing` | 切り替えた後の使用箇所が、存在しないメディアを指す（[index.md](index.md) の「参照の記録と保護」） | `commitRejected`（`cause` は `mediaMissing`） | 同上 |
| `capacityExceeded` | 切り替えが、量の上限を超える | `capacityExceeded` | 同上 |
| `unclassified` | 上のどれにも分類しない理由 | `null` | 反映を失敗させない |

- どの場合も、確定の UnitOfWork は何も反映しない。現行スキーマ、データ、運用設定は変わらない
- 適用中はコンテンツの書き込みを止めるので（「WriteScope とスキーマ」）、準備が読み終えたコンテンツと履歴の版は、確定までに変わらない。変換結果が使うメディアは、現行の内容も使っているので（「値」の `LiteralValue`）、確定までに削除も非公開化もされない（[index.md](index.md) の「参照の記録と保護」）。準備の後に、切り替えの結果を変える並行した変更は無い
- 準備が確かめ終える公開内容の整合性は、[content.md](content.md) の「ContentMigration」が定める。`stagingMismatch`、`publicationIntegrity`、`mediaMissing` が生じるのは、準備の確認に漏れがある場合だけ
- 切り替えを拒否された反映は、成功として示さず、失敗として終える
- 切り替えを拒否された反映の結果（`commitRejected`）は、開発者に、切り替えを確定できなかったことと `cause`、準備が求めた影響（`effects`）を示す。現行スキーマとデータは変わっていない
- 区画の `Workspace` が `active` でないワークスペースの扱いは、[index.md](index.md) の「イベントの消費者と定期ジョブ」が定める

#### ドメインイベント

```ts
type SchemaApplyRequestedEvent = DomainEventBase<"schema.apply_requested", Readonly<{ jobId: SchemaApplyJobId }>>;

type SchemaAppliedEvent = DomainEventBase<"schema.applied", Readonly<{ schemaVersion: number }>>;
```

- ドラフトとペイロードの形は、[common.md](common.md) の「ドメインイベント」に従う
- `schema.apply_requested` は、受け付けたジョブの、準備の開始と続きを伝える。`jobId` は、そのジョブの ID。発行するのは `Preparation` の `start`、`recordConverted`、`recordTallied` だけ。ドラフトの `aggregateId` は `jobId`
- `schema.applied` の `schemaVersion` は、確定後の版。発行するのは `WorkspaceSchema.commit` だけ。`schema.applied` は、このドメインイベントだけを指す。操作履歴の種類は [audit.md](audit.md) が定める
- 公開内容の変更の一覧と通し番号は、Content が持つ。公開内容が変わる反映の確定が残す公開の変更の記録は、[content.md](content.md) の「PublicationLedger」が定める

### SchemaVersionRecord

スキーマの版の記録の一件。追記だけの集約。`WorkspaceSchema` の `schemaVersion` が進む変更と同じ UnitOfWork で追記する。追記した後は変更しない。

```ts
type SchemaVersionRecord = Readonly<{
  schemaVersion: number;
  fingerprint: SchemaFingerprint;
  actor: TokenActor;
  actorNames: ActorNames;
  occurredAt: Date;
  change:
    | Readonly<{ kind: "applied"; migrations: RecordedMigrations; lossyOperations: readonly LossyOperation<"migrations">[] }>
    | Readonly<{ kind: "reflected" }>
    | Readonly<{ kind: "direct"; lossyOperations: readonly LossyOperation<"direct">[] }>
    | Readonly<{ kind: "recorded"; migrations: RecordedMigrations }>;
}>;

type RecordedMigration = Readonly<{ name: MigrationName; digest: MigrationDigest }>;
type RecordedMigrations = readonly [RecordedMigration, ...RecordedMigration[]];
```

| フィールド | 制約 |
| --- | --- |
| `schemaVersion` | 区画の中で一件を識別する。1 以上 |
| `fingerprint` | その版のスキーマ全体の識別子 |
| `actor` | 反映または記録を要求した実行者。`TokenActor` に限る |
| `actorNames` | 記録の時点の名称。`Preparation` の `actorNames` と同じ規則。メンバーの名前は、利用の解除の後もアカウントから読める。メンバーの解除とトークンの削除の後も、この値で示す |
| `change.kind` | 版の変更の種類。`applied` は移行定義の適用による版、`reflected` は移行定義を必要としない定義だけの反映による版、`direct` は直接反映による版、`recorded` は適用済みとしての記録による版 |
| `migrations` | `applied` と `recorded` が持つ。1 個以上 |
| `lossyOperations` | その版に含むデータを失い得る操作。`direct` では、失った値・項目・コンテンツの種類を `LossyOperation.loss` が表す |

#### 振る舞い

```ts
const SchemaVersionRecord: {
  ofCommit(committed: IdleWorkspaceSchema, job: PreparedJob, now: Date): SchemaVersionRecord;
  ofRecord(recorded: IdleWorkspaceSchema, plan: RecordPlan, by: RecordedActor, now: Date): SchemaVersionRecord;
};
```

- `ofCommit` は、`commit` に渡した準備済みのジョブ（`job`）と、`commit` が返した `WorkspaceSchema`（`committed`）から記録を作る。`plan` は `job.preparation.plan`。`plan.source` が `direct` なら `direct`、`migrations` で `plan.migrations` が 1 個以上なら `applied`、0 個なら `reflected`。`lossyOperations` は `plan.lossyOperations`、`fingerprint` は `committed` のスキーマ全体の識別子、`actor` と `actorNames` は準備のもの。`PreparedJob` だけを受けるので、事前確認から記録は作れない
- `ofRecord` は `recorded` の記録を作る。`by.names` は、ユースケースが記録の時点に読んだ名称
- 名称は、記録の後に変えない
- 操作履歴の種類は、ユースケースが `change.kind` から一対一に決める

#### 不変条件

- `schemaVersion` は、同じ UnitOfWork で保存する `WorkspaceSchema` の `schemaVersion` と一致する
- `applied` と `recorded` の `migrations` は、その版で `appliedMigrations` に加えた移行定義と、順序を含めて一致する。`reflected` と `direct` の版は、`appliedMigrations` を変えない

### SchemaApplyJob

受け付けた反映・事前確認の一回。`planApply` が `planned` を返した要求だけがジョブになる。種類（`preparation.mode`）、準備の進行（`preparation`）、終端の結果を持つ。ID は `preparation.jobId`。「ジョブ」は、この集約だけを指す。

```ts
type PreparingJob<M extends ApplyMode = ApplyMode> = M extends ApplyMode
  ? Readonly<{ status: "preparing"; preparation: OngoingPreparation<M>; version: number }>
  : never;
type PreparedJob = Readonly<{
  status: "prepared";
  preparation: FinishedPreparation<"commit">;
  effects: ApplyEffects;
  version: number;
}>;
type SucceededJob = Readonly<{
  status: "succeeded";
  preparation: FinishedPreparation<"commit">;
  success: ApplySuccess;
  finishedAt: Date;
  version: number;
}>;
type CheckedJob = Readonly<{
  status: "checked";
  preparation: FinishedPreparation<"dryRun">;
  blockers: Sampled<ApplyBlocker>;
  effects: ApplyEffects;
  finishedAt: Date;
  version: number;
}>;
type FailedJobOf<P, F> = Readonly<{ status: "failed"; preparation: P; failure: F; finishedAt: Date; version: number }>;
type FailedJob<M extends ApplyMode = ApplyMode> = M extends "commit"
  ? FailedJobOf<OngoingPreparation<"commit">, PreparingFailure> | FailedJobOf<FinishedPreparation<"commit">, PreparedFailure>
  : FailedJobOf<OngoingPreparation<"dryRun">, DryRunFailure>;

type TallyingJob<M extends ApplyMode = ApplyMode> = PreparingJob<M> & Readonly<{ preparation: TallyingPreparation<M> }>;

type InProgressJob = PreparingJob | PreparedJob;
type FinishedJob<M extends ApplyMode = ApplyMode> = M extends "commit"
  ? SucceededJob | FailedJob<"commit">
  : CheckedJob | FailedJob<"dryRun">;
type SchemaApplyJob = InProgressJob | FinishedJob;

type DryRunFailure =
  | Readonly<{ reason: "capacityExceeded" }>
  | Readonly<{ reason: "schemaChanged" }>
  | Readonly<{ reason: "interrupted" }>;
```

| `status` | 取る `mode` | 内容 |
| --- | --- | --- |
| `preparing` | `commit`、`dryRun` | 準備の途中 |
| `prepared` | `commit` | 準備済み。準備領域に変換結果がそろい、確定を待つ。準備が求めた影響（`effects`）を持つ |
| `succeeded` | `commit` | 確定した反映。成功の内容（`ApplySuccess`）を持つ |
| `checked` | `dryRun` | 終えた事前確認。`blockers` と `effects` を持つ。`blockers.total` が 0 でなければ、反映は失敗する見込みを表す |
| `failed` | `commit`、`dryRun` | 失敗して終えたジョブ。理由（反映は `ApplyFailure`、事前確認は `DryRunFailure`）を持つ |

| フィールド | 制約 |
| --- | --- |
| `preparation` | そのジョブの準備の一回（「Preparation」）。ジョブの ID、種類、実行者と名称、要求の要約、計画、進行を持つ。どの状態でも持つ。`failed` では、失敗した時点の値のまま。操作の求め方は `preparation.plan.source` |
| `finishedAt` | ジョブを終えた日時。終端（`succeeded`、`checked`、`failed`）だけが持つ |
| `version` | 楽観ロックの番号。作成時は 0 |

- 状態と `mode` の、表に無い組（事前確認の `prepared`・`succeeded`、反映の `checked`）は、型が表現できなくしている
- 進行中（`InProgressJob`）は `preparing` と `prepared`。終端（`FinishedJob`）は `succeeded`、`checked`、`failed`
- 反映の結果は、`mode` が `commit` の終端のジョブ。反映ごとの結果の一覧は、これを読む。成功の内容は `success`、失敗の理由は `failure` が持つ
- 事前確認は、適用中の反映とも、ほかの事前確認とも並行して受け付け、進める。ジョブごとに準備領域が分かれるので、互いの結果を変えない
- 失敗した反映のジョブは、後の反映が成功しても残る。未解消かどうかは、`WorkspaceSchema.failedApply` が表す

#### 振る舞い

```ts
const SchemaApplyJob: {
  accept<M extends ApplyMode>(started: OngoingPreparation<M>): PreparingJob<M>;
  advance<M extends ApplyMode>(job: PreparingJob<M>, advanced: OngoingPreparation<M>): PreparingJob<M>;
  complete(job: TallyingJob<"commit">, outcome: PreparationOutcome, now: Date): PreparedJob | FailedJob<"commit">;
  complete(job: TallyingJob<"dryRun">, outcome: PreparationOutcome, now: Date): CheckedJob;
  failIfSchemaChanged(job: PreparingJob<"dryRun">, ws: WorkspaceSchema, now: Date): FailedJob<"dryRun"> | null;
  fail(job: PreparingJob<"commit"> | PreparedJob, failure: PreparingFailure, now: Date): FailedJob<"commit">;
  fail(job: PreparingJob<"dryRun">, failure: PreparingFailure, now: Date): FailedJob<"dryRun">;
  fail(job: PreparedJob, failure: Readonly<{ reason: "commitRejected"; cause: CommitRejection }>, now: Date): FailedJob<"commit">;
  succeed(job: PreparedJob, success: ApplySuccess, now: Date): SucceededJob;
  isReplayOf(job: SchemaApplyJob, request: ApplyRequest): boolean;
};
```

| 振る舞い | 処理 |
| --- | --- |
| `accept` | `Preparation.start` の結果を持つ、`preparing` のジョブを作る。反映・事前確認のどれも、この振る舞いで作る |
| `advance` | `preparation` を、`advanced`（`Preparation.recordConverted` または `recordTallied` の結果）に置き換える。`advanced.jobId` が違う呼び出しは `BusinessRuleError`（`SCHEMA_JOB_MISMATCH`） |
| `complete` | 準備を終える。集計の途中（`preparation.phase` が `tallying`）のジョブだけを受ける。`preparation` は `Preparation.finish` の結果。反映は、`outcome.blockers.total` が 0 なら `prepared`（`effects` は `outcome.effects`）、1 以上なら `failed`（理由は `blocked` で、`outcome.blockers` と `outcome.effects` を持つ）。事前確認は `checked`（`blockers` と `effects` は `outcome` のもの） |
| `failIfSchemaChanged` | 事前確認が前提とした版を照らす。`ws.schemaVersion` が `job.preparation.plan.baseSchemaVersion` と異なれば、`schemaChanged` で `failed` にしたジョブを返す。同じなら `null` を返す。`schemaChanged` の失敗は、この振る舞いだけが作る |
| `fail` | 進行中のジョブを `failed` にする。`preparation` は変えない。`prepared` のジョブの失敗は、そのジョブの `effects` を引き継いだ `PreparedFailure` になる。`commitRejected` は、`prepared` のジョブだけが受ける。`blocked` と `schemaChanged` は渡せない |
| `succeed` | 準備済みのジョブを `succeeded` にし、`success` を持たせる。`success` は、同じジョブを渡した `WorkspaceSchema.commit` の結果。`succeeded` のジョブを作るのは、この振る舞いだけ |
| `isReplayOf` | `job.preparation.requestDigest` が `ApplyRequest.digestOf(request)` と一致すれば真。要約が `source` と `mode` を含むので、求め方または確定の有無が違う要求は偽になる |

- `accept` のほかの振る舞いは、`version` を 1 進める。終端にする振る舞いは、`finishedAt` を `now` にする
- `accept` と `advance` は、準備（「Preparation」）を包んで持つだけ
- 反映と事前確認は、同じ振る舞いで進む。違いは、`complete` の結果と、事前確認だけが受ける `failIfSchemaChanged`、反映だけが `WorkspaceSchema` を適用中にすること

#### 不変条件

- 終端のジョブは、以後変わらない
- `checked` の `blockers` と `effects` は、同じデータと同じ計画の反映の準備が求める `PreparationOutcome` と等しい
- 終端のジョブは、準備領域を持たない。`succeeded` では切り替えで、`checked` と `failed` では破棄で無くなる

#### ライフサイクル

```text
accept → preparing ─ advance → preparing（続きを始める）

反映      preparing ─ complete → prepared ─ succeed → succeeded
                    │                     └ fail → failed（commitRejected、capacityExceeded、interrupted）
                    ├ complete → failed（blocked）
                    └ fail → failed（capacityExceeded、interrupted）

事前確認  preparing ─ complete → checked
                    ├ failIfSchemaChanged → failed（schemaChanged）
                    └ fail → failed（capacityExceeded、interrupted）
```

ジョブは、ワークスペースの利用終了で削除する。

#### 受け付けた要求と再送

要求した側は、応答を受け取れなかった要求を、同じ `SchemaApplyJobId` で送り直す。要求は、検証の前に、その ID のジョブの有無で扱いが決まる。

- ジョブが無い場合は、要求を `planApply` で検証する。`planned` なら、返った `AcceptableRequest` で `Preparation.start` を行い、その結果を `SchemaApplyJob.accept` に渡す。反映では、そのジョブを `WorkspaceSchema.beginApply` に渡す。`planned` でなければ、結果を返す（ジョブになる要求は「SchemaApplyJob」が定める）。拒否された要求と差分の無い要求の送り直しは、その時点の `WorkspaceSchema` で検証し直す
- ジョブがあり、`isReplayOf` が真の場合は、何も書かず、イベントも発行せず、そのジョブを現在の状態のまま返す
- ジョブがあり、`isReplayOf` が偽の場合は、`ConflictError`（`SCHEMA_JOB_ID_CONFLICT`）
- ジョブの ID の一意性は、`SchemaApplyJobRepository.insert` の契約が担保する。同じ ID の反映と事前確認は、同時に存在しない
- 並行した二つの受け付けが、どちらも先に読めなかった場合、同じ ID では一方だけが確定し、遅い方は `ConflictError` になる。理由は、ジョブの `insert` の `ALREADY_EXISTS`、または、反映どうしでは `WorkspaceSchema` の保存の `STALE_VERSION`（「WorkspaceSchemaRepository」）。次の再送が、受け付け済みのジョブを引き、同じ内容・同じ `mode` なら再送として返し、違えば `SCHEMA_JOB_ID_CONFLICT` で拒否する

#### 中断

進行中のジョブを `interrupted` で失敗させること。止まった反映の検出と、利用再開前の処理が行い、対象の絞り方だけが違う。手続き（読み方と UnitOfWork の分け方）は [../usecases/schema.md](../usecases/schema.md) の「進行中のジョブの中断」が定める。

| 行うもの | 対象 |
| --- | --- |
| 止まった反映の検出 | 進行中のジョブのうち、`ApplyStall.isStalled` が真のもの |
| 利用再開前の処理 | 進行中のジョブのすべて。閾値によらない |

- 対象は、反映と事前確認、`preparing` と `prepared` を分けない
- 対象のジョブを `SchemaApplyJob.fail`（`interrupted`）で失敗させる
- 読み直した時点で終端のジョブと、止まった反映の検出では `ApplyStall.isStalled` が偽になったジョブは、失敗させない

## 値オブジェクト

値オブジェクトは、生成時に制約を確かめる。スキーマと移行定義は利用者の入力なので、満たさない場合は、位置ごとの理由を持つ `ValidationError` になる。理由コードは「エラーの理由コード」が定める。等価性は、断りが無い限り、すべてのフィールドの値の一致で決める。

### Preparation

受け付けた反映・事前確認の、準備の一回。計画、実行者と名称、要約、進行を持つ。ジョブ（`SchemaApplyJob`）が、どの状態でも持つ。反映と事前確認は、同じ準備を行う。準備の進行の規則と、止まった準備の判定は、この値だけが定める。

```ts
type Preparation<M extends ApplyMode = ApplyMode, Ph extends PreparationPhase = PreparationPhase> = Readonly<{
  jobId: SchemaApplyJobId;
  mode: M;
  actor: TokenActor;
  actorNames: ActorNames;
  requestDigest: RequestDigest;
  requestedAt: Date;
  plan: PlanFor<M>;
  phase: Ph;
  progressedAt: Date;
}>;

type UntargetedConverting = Readonly<{ kind: "converting"; target: null }>;
type TargetedConverting = Readonly<{ kind: "converting"; target: ConversionTarget; processed: number }>;
type PreparationPhase =
  | UntargetedConverting
  | TargetedConverting
  | Readonly<{ kind: "tallying"; total: number; tally: ConversionTally; cursor: TallyCursor }>
  | Readonly<{ kind: "finished"; total: number }>;
type OngoingPreparation<M extends ApplyMode = ApplyMode> = Preparation<M, Exclude<PreparationPhase, { kind: "finished" }>>;
type TallyingPreparation<M extends ApplyMode = ApplyMode> = Preparation<M, Extract<PreparationPhase, { kind: "tallying" }>>;
type FinishedPreparation<M extends ApplyMode = ApplyMode> = Preparation<M, Extract<PreparationPhase, { kind: "finished" }>>;

type JobProgress = Readonly<{ processed: number; total: number | null }>;
type RequestDigest = string & { readonly [requestDigestBrand]: true };
type ActorNames = Readonly<{ token: TokenDesignation; member: string }>;
type RecordedActor = Readonly<{ actor: TokenActor; names: ActorNames }>;

type ConversionTarget = Readonly<{
  lastContentId: ContentId | null;
  lastRevisionId: RevisionId | null;
  total: number;
}>;
type ConversionPart = "contents" | "revisions";
type ConversionStep = Readonly<{ part: ConversionPart; staged: number; chunk: number; exhausted: boolean }>;

type TallyCursor =
  | Readonly<{ part: "contents"; after: ContentId | null }>
  | Readonly<{ part: "revisions"; after: RevisionId | null }>;

type ConversionTally = Readonly<{
  blockers: Sampled<ApplyBlocker>;
  losses: readonly OperationLoss[];
  changedPublished: Sampled<Readonly<{ content: ContentId; change: "updated" | "unpublished" }>>;
  constraintViolations: Sampled<ConstraintViolation>;
  draftsWithErrors: Sampled<ContentId>;
}>;

const Preparation: {
  start<M extends ApplyMode>(
    jobId: SchemaApplyJobId,
    request: AcceptableRequest<M>,
    by: RecordedActor,
    now: Date,
  ): WithEventDrafts<OngoingPreparation<M>, SchemaApplyRequestedEvent>;
  progress(preparation: Preparation): JobProgress;
  fixTarget<M extends ApplyMode>(
    preparation: Preparation<M, UntargetedConverting>,
    target: ConversionTarget,
  ): Preparation<M, TargetedConverting>;
  recordConverted<M extends ApplyMode>(
    preparation: Preparation<M, TargetedConverting>,
    step: ConversionStep,
    now: Date,
  ): WithEventDrafts<OngoingPreparation<M>, SchemaApplyRequestedEvent>;
  nextTallyCursor(
    cursor: TallyCursor,
    read: Readonly<{ count: number; lastId: ContentId | RevisionId | null }>,
  ): TallyCursor | null;
  recordTallied<M extends ApplyMode>(
    preparation: TallyingPreparation<M>,
    tally: ConversionTally,
    next: TallyCursor,
    now: Date,
  ): WithEventDrafts<TallyingPreparation<M>, SchemaApplyRequestedEvent>;
  finish<M extends ApplyMode>(preparation: TallyingPreparation<M>, now: Date): FinishedPreparation<M>;
  outcome(tally: ConversionTally, forms: FormChanges, broken: readonly BrokenNotificationReference[]): PreparationOutcome;
};

const ConversionTarget: {
  of(
    read: Readonly<{
      lastContentId: ContentId | null;
      contents: number;
      lastRevisionId: RevisionId | null;
      revisions: number;
    }>,
  ): ConversionTarget;
  clip<T extends Readonly<{ id: ContentId | RevisionId }>>(
    target: ConversionTarget,
    part: ConversionPart,
    scanned: readonly T[],
  ): Readonly<{ items: readonly T[]; exhausted: boolean }>;
};

const ConversionTally: { empty(): ConversionTally };

const ApplyStall: {
  readonly thresholdMinutes: 30;
  before(now: Date): Date;
  isStalled(preparation: Preparation, now: Date): boolean;
};
```

| フィールド | 制約 |
| --- | --- |
| `jobId` | ジョブの ID。要求した側が作り、再送でも同じ ID を使う（[../usecases/schema.md](../usecases/schema.md) の「要求の入力」）。準備領域は、この ID の値で指す（[../usecases/schema.md](../usecases/schema.md) の「準備領域の識別子」） |
| `mode` | `commit` は反映、`dryRun` は適用の事前確認 |
| `actor` / `actorNames` | 要求した実行者と、受け付けた時点の名称。要求できる実行者はトークンだけなので、`TokenActor`（`appToken` または `memberToken`）に限る。`token` はトークンを指す表示の元（[common.md](common.md) の「TokenDesignation」）。`member` は、関係するメンバー（[token.md](token.md) の `Token.relatedAccountId`）の名前。トークンの値を含まない |
| `requestDigest` | 要求の内容の要約。`AcceptableRequest.requestDigest` の値 |
| `plan` | `AcceptableRequest.plan` の値。操作の求め方は `plan.source`、要求が前提とした現行スキーマの版は `plan.baseSchemaVersion` が持つ |
| `phase` | `converting` は、変換結果を準備領域へ書いている途中で、変換の対象（`target`）を持つ。`target` は、最初の区切りが決めるまで `null`。決めた後は、変換を終えたコンテンツと履歴の版の件数（`processed`）を持つ。`tallying` は、変換を終え、準備領域の報告を集計している途中で、変換の対象の件数（`total`）、途中までの集計（`tally`）、次に読む位置（`cursor`）を持つ。`finished` は、準備を終えたことで、変換の対象の件数（`total`）を持つ |
| `progressedAt` | 進行を最後に記録した日時 |

| 振る舞い | 処理 |
| --- | --- |
| `start` | 受け付けられる要求から、準備を始める。`mode`、`plan`、`requestDigest` は `request` のもの、実行者と名称は `by` のもの、`phase` は `converting`（`target` は `null`）、`requestedAt` と `progressedAt` は `now`。`schema.apply_requested` を発行する |
| `progress` | 進行（`JobProgress`）を、`phase` から求める。`target` が `null` の `converting` は `{ processed: 0, total: null }`、対象を決めた `converting` は `{ processed: phase.processed, total: phase.target.total }`、`tallying` と `finished` は `processed` と `total` がどちらも `phase.total` |
| `fixTarget` | 変換の対象を決める。`phase.target` が `null` の準備だけを受け、`phase` を、`target` が `target`、`processed` が 0 の `converting` にした準備を返す。`progressedAt` を変えず、イベントを発行しない。結果は、同じ消費の `recordConverted` に渡す |
| `recordConverted` | 変換の一区切りを記録する。対象を決めた準備だけを受ける。`step.part` は、この区切りで走査した部分、`staged` は、この区切りの前に準備領域へ書き終えていた件数、`chunk` は、この区切りで書いた件数、`exhausted` は、その部分の対象を走査し終えたかどうか（`ConversionTarget.clip` の結果）。変換の終端は、`part` が `revisions` で `exhausted` が真の区切り。終端でない区切りは、`phase.processed` を、`staged + chunk` と `phase.target.total` の小さいほうにする。終端の区切りは、`phase` を `tallying`（`total` は `phase.target.total`、`tally` は `ConversionTally.empty()`、`cursor` は `{ part: "contents", after: null }`）にする。どちらも `progressedAt` を `now` にし、続きを始める `schema.apply_requested` を発行する |
| `nextTallyCursor` | 報告の一区切りを読んだ後の、次に読む位置を返す。`read.count` が `SchemaLimits.preparationChunk` に等しければ、同じ `part` の `read.lastId` の後ろ。少なければ、`contents` では `{ part: "revisions", after: null }`、`revisions` では `null`（読み終えた） |
| `recordTallied` | 集計の一区切りを記録する。`phase` が `tallying` の準備だけを受ける。`tally` を渡された値に、`cursor` を `next` に置き換え、`progressedAt` を `now` にし、続きを始める `schema.apply_requested` を発行する |
| `finish` | `phase` が `tallying` の準備だけを受け、`phase` を `finished`（`total` は `phase.total`）にし、`progressedAt` を `now` にする。`SchemaApplyJob.complete` が呼ぶ |
| `outcome` | 集計を終えた `tally` に、計画の `forms` と、不整合となる通知定義の参照を加えて、`PreparationOutcome` を作る |

- `jobId`、`mode`、`plan`、`requestDigest`、`actor`、`actorNames`、`requestedAt` は、受け付けた後に変わらない
- `converting` の `processed` は減らず、`target.total` を超えない
- 変換の対象（`ConversionTarget`）は、準備の最初の区切りの時点の、区画のコンテンツと履歴の版。走査の終端（`lastContentId`、`lastRevisionId`。その時点の最大の ID。無ければ `null`）と、件数（`total`。コンテンツの件数と履歴の版の件数の和）で表す。決めた後に変わらない
- `ConversionTarget.of` は、読んだ最大の ID と件数から対象を作る。`total` は `contents + revisions`
- 走査の一区切りは、区画のコンテンツ、または履歴の版を、ID の昇順（コードポイントの順）に、指定した ID の後ろから `SchemaLimits.preparationChunk` 件まで読んだ列。走査の位置は、準備領域が持つ変換結果の最大の ID（部分ごと。無ければ `null`）
- `ConversionTarget.clip` は、走査の一区切りを、対象の範囲に切る。`items` は、`scanned` のうち、ID がその部分の終端以下のもの。終端が `null` の部分では空。`exhausted` は、`scanned` が `SchemaLimits.preparationChunk` 件より少ない、または `items` が `scanned` より少ない場合に真
- 変換の一区切りは、コンテンツを先に、履歴の版を後に走査する。走査の位置の後ろのコンテンツの一区切りを `clip`（`part` は `contents`）に渡し、`items` が 1 件以上なら、その `items` を変換して準備領域へ書き、`contents` の区切りとして `recordConverted` に渡す。`items` が空なら、コンテンツを終えているので、走査の位置の後ろの履歴の版の一区切りを `clip`（`part` は `revisions`）に渡し、`items` を変換して準備領域へ書き、`revisions` の区切りとして渡す。一つの区切りが `recordConverted` に渡すのは、`contents` と `revisions` のどちらか一方。`items` が空の `revisions` の区切りは、準備領域へ書かずに渡す
- `phase.target` が `null` の準備の区切りは、走査の前に、区画の最大の ID と件数を読み、`ConversionTarget.of` の結果を `fixTarget` に渡す。`null` でない準備は、読まない
- 変換は、走査の位置が終端に達したことで終わる。対象を決めた後に、終端より後ろの ID で作られたコンテンツと履歴の版は、変換と集計の対象に入らない。対象を決めた後に削除されたコンテンツと履歴の版は、走査に現れず、飛ばす。終端までの、走査がまだ通っていない位置に作られたものは、走査に現れ、変換する。どの場合も、終端の区切りで `tallying` になる
- 走査の位置は、区切りごとに終端へ向かって進み、戻らない。`converting` の準備は、終端の区切りで `tallying` になるか、失敗で終わる
- 反映の対象は、区画のすべてのコンテンツと履歴の版に等しい（「確定できない場合の扱い」）。事前確認は書き込みを止めないので、対象を決めた後の変化は、結果に現れるとは限らない。事前確認の結果は、適用時の成功を保証しない。反映の確定は、準備領域が区画のすべての結果を過不足なく持つことを確かめる（「確定できない場合の扱い」の `stagingMismatch`）
- `ConversionTally` は、変換の報告を畳み込んだ集計。`PreparationOutcome` のうち、コンテンツの変換から決まる部分（`blockers` と、`ApplyEffects` の `losses`、`changedPublished`、`constraintViolations`、`draftsWithErrors`）を持つ。`Sampled` の内容は `SchemaLimits.sampleLimit` 件までなので、集計の大きさは、コンテンツの件数に比例しない。報告の畳み込みと、一意性の違反の追加は、コンテンツの値の形を持つ側が定め、この型を入出力に使う
- `ApplyStall.thresholdMinutes` は、止まった準備の閾値の長さ（分）。値は、[../scenario/app-integration.md](../scenario/app-integration.md) の APP-12 の「止まった反映を中断として失敗で終えるまでの時間」。技術設計の層で値を持つのは、この定数だけ。ユースケース、動的フロー、テストケースは、定数の名前で指す
- `ApplyStall.before` は、閾値の日時を返す。`now` の `thresholdMinutes` 分前
- `ApplyStall.isStalled` は、`preparation.progressedAt` が `before(now)` 以前（同じ日時を含む）なら真。対象は、進行中のジョブ（`preparing` と `prepared`。反映と事前確認）の `preparation`
- 止まったジョブは、止まった反映の検出が失敗させる（「SchemaApplyJob」の「中断」）

### キーと名前

```ts
type ModelKey = string & { readonly [modelKeyBrand]: true };
type FormKey = string & { readonly [formKeyBrand]: true };
type FieldKey = string & { readonly [fieldKeyBrand]: true };
type OptionKey = string & { readonly [optionKeyBrand]: true };
type BlockTypeKey = string & { readonly [blockTypeKeyBrand]: true };
type DerivativeKey = string & { readonly [derivativeKeyBrand]: true };
type MigrationName = string & { readonly [migrationNameBrand]: true };
```

- キーは、先頭が英小文字、2 文字目以降が英数字または `_`、長さ 1 から 64 文字。大文字と小文字を区別する
- `MigrationName` は、空白だけでない、長さ 1 から 128 文字の文字列
- 等価性は文字列の一致

キーが一意である範囲は次のとおり。

| キー | 一意である範囲 |
| --- | --- |
| `ModelKey` | スキーマのモデルの間 |
| `FormKey` | スキーマのフォームの間 |
| `FieldKey` | 同じ階層の項目の間。フォームでは同じフォームの項目の間 |
| `OptionKey` | 同じ項目の選択肢の間 |
| `BlockTypeKey` | 同じ項目のブロックの種類の間 |
| `DerivativeKey` | スキーマの派生形式の間 |

モデルとフォームは、同じキーを持てる。

### 日付と日時

```ts
type CalendarDate = string & { readonly [calendarDateBrand]: true };
type Instant = string & { readonly [instantBrand]: true };
```

- `CalendarDate` は `YYYY-MM-DD` の形式の、実在する暦日。タイムゾーンを持たない
- `Instant` は UTC の時点。`YYYY-MM-DDTHH:mm:ss.sssZ` の形式に正規化して持つ

### Schema

```ts
type Schema = Readonly<{
  models: readonly ModelDefinition[];
  forms: readonly FormDefinition[];
  derivatives: readonly ImageDerivative[];
}>;

const Schema: {
  empty(): Schema;
  equals(a: Schema, b: Schema): boolean;
  structureOf(schema: Schema): ContentStructure;
  typeShapeOf(schema: Schema): TypeShape;
  form(schema: Schema, key: FormKey): FormDefinition | null;
  displayName(definition: Readonly<{ key: string; label: string | null }>): string;
  diffForms(before: Schema, after: Schema): FormChanges;
};
```

配列の順序は表示の順序を表す。`equals` は、配列の順序を含めて、すべての定義が一致する場合に真を返す。`Schema` は生成時にキーの形式だけを確かめる。定義の間の整合は `SchemaValidation` が確かめる。

`form` は、そのキーのフォームの定義を返す。無ければ `null`。

`displayName` は、モデル、フォーム、項目、選択肢、ブロックの種類の表示名称を返す。`label` が `null` の定義は、キーを表示名称として使う。

### ModelDefinition

```ts
type ModelDefinition = Readonly<{
  key: ModelKey;
  cardinality: "single" | "collection";
  fields: readonly ContentField[];
  label: string | null;
  deprecated: boolean;
  listFields: readonly FieldKey[];
  inquiryLabelField: FieldKey | null;
  previewPath: PreviewPath | null;
}>;
```

| フィールド | 内容 |
| --- | --- |
| `cardinality` | `single` は単一の情報、`collection` は複数件を持つモデル |
| `label` | 表示名称。`null` の場合の表示は `Schema.displayName` が定める。項目、選択肢、ブロックの種類、フォームの `label` も同じ |
| `listFields` | 一覧表示と検索に使う項目。モデル直下の項目のキー |
| `inquiryLabelField` | 問い合わせ対象の表示に使う項目。モデル直下のテキストの項目のキー。`null` は表示名称を持たないことを表す |
| `previewPath` | プレビュー先の経路。`null` はプレビュー先を定義しないことを表す |

### ContentField

```ts
type ContentField = Readonly<{
  key: FieldKey;
  label: string | null;
  description: string | null;
  example: string | null;
  deprecated: boolean;
  type: ContentFieldType;
}>;

type ContentFieldType =
  | Readonly<{ kind: "text"; required: boolean; input: "singleLine" | "multiline"; constraints: TextConstraints; unique: boolean; defaultValue: string | null }>
  | Readonly<{ kind: "number"; required: boolean; constraints: RangeConstraints<number>; defaultValue: number | null }>
  | Readonly<{ kind: "boolean"; required: boolean; defaultValue: boolean | null }>
  | Readonly<{ kind: "date"; required: boolean; constraints: RangeConstraints<CalendarDate>; defaultValue: TemporalDefault<CalendarDate> | null }>
  | Readonly<{ kind: "datetime"; required: boolean; constraints: RangeConstraints<Instant>; defaultValue: TemporalDefault<Instant> | null }>
  | Readonly<{ kind: "select"; required: boolean; options: readonly ContentOption[]; defaultValue: OptionKey | null }>
  | Readonly<{ kind: "multiSelect"; required: boolean; options: readonly ContentOption[]; constraints: CountConstraints; defaultValue: readonly OptionKey[] }>
  | Readonly<{ kind: "richText"; required: boolean }>
  | Readonly<{ kind: "media"; required: boolean }>
  | Readonly<{ kind: "reference"; required: boolean; models: readonly ModelKey[] }>
  | Readonly<{ kind: "referenceList"; required: boolean; models: readonly ModelKey[]; constraints: CountConstraints }>
  | Readonly<{ kind: "group"; fields: readonly ContentField[] }>
  | Readonly<{ kind: "repeat"; required: boolean; fields: readonly ContentField[]; constraints: CountConstraints }>
  | Readonly<{ kind: "blocks"; required: boolean; blockTypes: readonly BlockTypeDefinition[]; constraints: CountConstraints }>;

type ContentOption = Readonly<{ key: OptionKey; label: string | null; deprecated: boolean }>;
type BlockTypeDefinition = Readonly<{ key: BlockTypeKey; label: string | null; deprecated: boolean; fields: readonly ContentField[] }>;

type TextConstraints = Readonly<{ minLength: number | null; maxLength: number | null; format: TextFormat | null }>;
type TextFormat = Readonly<{ type: "email" }> | Readonly<{ type: "url" }> | Readonly<{ type: "pattern"; pattern: string }>;
type RangeConstraints<T> = Readonly<{ min: T | null; max: T | null }>;
type CountConstraints = Readonly<{ min: number | null; max: number | null }>;
type TemporalDefault<T> = Readonly<{ type: "fixed"; value: T }> | Readonly<{ type: "createdAt" }>;
```

値の種類と、持てる制約・初期値は次のとおり。型に無い組み合わせは表現できない。型に無い制約を持つ入力は、境界のデコードが `ValidationError` として拒否する。

| `kind` | 値の種類 | 制約 | 初期値 |
| --- | --- | --- | --- |
| `text` | テキスト。`input` が一行と複数行の入力を分ける | 文字数、書式、一意性 | 固定の値 |
| `number` | 数値 | 範囲 | 固定の値 |
| `boolean` | 真偽値 | なし | 固定の値 |
| `date` | 日付 | 範囲 | 固定の値、または作成時点 |
| `datetime` | 日時 | 範囲 | 固定の値、または作成時点 |
| `select` | 単一選択 | なし | 選択肢のキー |
| `multiSelect` | 複数選択 | 選択数 | 選択肢のキーの組。空は初期値なし |
| `richText` | リッチテキスト | なし | なし |
| `media` | メディア | なし | なし |
| `reference` | コンテンツ参照（一件） | なし | なし |
| `referenceList` | コンテンツ参照（複数件） | 参照先の件数 | なし |
| `group` | 項目のまとまり | なし | なし |
| `repeat` | 繰り返し | 件数 | なし |
| `blocks` | ブロック | 件数 | なし |

- `group` は必須を持たない。必須は中の項目が持つ
- `input` は編集画面の入力欄だけを変える。値の検証を変えない
- 文字数は Unicode のコードポイントで数える。`minLength`、`maxLength`、`CountConstraints` の `min`、`max` は 0 以上の整数
- `pattern` は ECMAScript の正規表現（`u` フラグ）のソースで、長さ 512 文字以下。値の全体への一致で判定する
- `unique` は、その項目の値が一意であることを求める。一意である範囲は [content.md](content.md) の「Content」の不変条件に従う
- 選択肢が空の単一選択・複数選択と、`models` が空のコンテンツ参照は、値を持てない項目として扱う

### FormDefinition

```ts
type FormDefinition = Readonly<{
  key: FormKey;
  label: string | null;
  fields: readonly FormField[];
  listFields: readonly FieldKey[];
}>;

type FormField = Readonly<{
  key: FieldKey;
  label: string | null;
  description: string | null;
  example: string | null;
  required: boolean;
  mustMatch: FieldKey | null;
  type: FormFieldType;
}>;

type FormFieldType =
  | Readonly<{ kind: "text"; input: "singleLine" | "multiline"; constraints: TextConstraints; defaultValue: string | null }>
  | Readonly<{ kind: "number"; constraints: RangeConstraints<number>; defaultValue: number | null }>
  | Readonly<{ kind: "boolean"; mustBeTrue: boolean; defaultValue: boolean | null }>
  | Readonly<{ kind: "date"; constraints: RangeConstraints<CalendarDate>; defaultValue: CalendarDate | null }>
  | Readonly<{ kind: "datetime"; constraints: RangeConstraints<Instant>; defaultValue: Instant | null }>
  | Readonly<{ kind: "select"; options: readonly FormOption[]; defaultValue: OptionKey | null }>
  | Readonly<{ kind: "multiSelect"; options: readonly FormOption[]; constraints: CountConstraints; defaultValue: readonly OptionKey[] }>
  | Readonly<{ kind: "contentReference"; models: readonly ModelKey[] }>;

type FormOption = Readonly<{ key: OptionKey; label: string | null }>;
```

- `mustMatch` は、同じフォームの同じ `kind` の別の項目と値が一致することを求める
- `mustBeTrue` は、値が真であることを求める。`required` は値が存在することを求め、`mustBeTrue` と独立に指定する
- `contentReference` は問い合わせ対象の項目。`models` は許可するモデル
- `listFields` は、回答の一覧表示と検索に使う項目のキー
- `defaultValue` は表示アプリが入力欄の初期値に使う

### ImageDerivative

```ts
type ImageDerivative = Readonly<{
  key: DerivativeKey;
  size:
    | Readonly<{ fit: "width"; width: number }>
    | Readonly<{ fit: "height"; height: number }>
    | Readonly<{ fit: "contain" | "cover"; width: number; height: number }>;
  output:
    | Readonly<{ format: "jpeg" | "webp" | "avif"; quality: number }>
    | Readonly<{ format: "png" }>;
}>;
```

- `width`、`height` は 1 以上の整数。`quality` は 1 から 100 の整数
- `size` の意味は [media.md](media.md) の「DerivativeGeometry」が定める
- 派生形式はスキーマに一組で、モデルや項目ごとに分けない

### PreviewPath

```ts
type PreviewPath = readonly PreviewPathPart[];
type PreviewPathPart =
  | Readonly<{ type: "literal"; value: string }>
  | Readonly<{ type: "contentId" }>
  | Readonly<{ type: "uniqueValue"; field: FieldKey }>;

const PreviewPath: {
  resolve(
    path: PreviewPath,
    input: Readonly<{ contentId: ContentId; uniqueValue: (field: FieldKey) => string | null }>,
  ): Extract<PreviewPathResolution, { status: "resolved" | "values_missing" }>;
  resolveFor(
    model: ModelDefinition,
    input: Readonly<{ contentId: ContentId; uniqueValue: (field: FieldKey) => string | null }>,
  ): PreviewPathResolution;
};

type PreviewPathResolution =
  | Readonly<{ status: "not_defined" }>
  | Readonly<{ status: "values_missing"; fields: readonly FieldKey[] }>
  | Readonly<{ status: "resolved"; path: string }>;
```

- 経路は部分を順に連結した文字列。先頭の部分が `/` で始まる `literal` であることと、`literal` が `?` と `#` を含まないことは、`SchemaValidation` の `previewPathShape` が確かめる。`WorkspaceSchema` のスキーマは不整合を持たないので、`resolveFor` に渡すモデルの `resolved` の `path` は `/` で始まり、クエリとフラグメントを持たない
- `contentId` と `uniqueValue` の値は、URL のパスの一区切りとしてパーセントエンコードして連結する
- `resolve` は、経路の文字列を `resolved` で返す。`uniqueValue` が指す項目の値が未設定の場合は、未設定の項目を `fields` に並べた `values_missing` を返す
- `resolveFor` は、モデルのプレビュー先の経路を解決する。`model.previewPath` が `null` なら `not_defined`、そうでなければ `resolve` の結果を返す
- `PreviewPathResolution` は、プレビュー先の経路の解決の結果。定義は、この型だけ。プレビュー先を求める側が、この型で受ける

### スキーマの 4 区分

| 区分 | 当たる型とフィールド |
| --- | --- |
| コンテンツの構造 | `ContentStructure`。`ModelDefinition` の `key`・`cardinality`、`ContentField` の `key`、`ContentFieldType` の `kind`・`required`・`options` のキー・`models`・`fields`・`blockTypes` のキー |
| コンテンツの検証 | `ContentFieldType` の `constraints`・`unique` |
| フォームの構造 | `FormDefinition` の `key`、`FormField` の `key`・`required`・`mustMatch`、`FormFieldType` の `kind`・`options` のキー・`models`・`constraints`・`mustBeTrue` |
| 表示・運用の定義 | `label`、配列の順序、`description`、`example`、`defaultValue`、`input`、`deprecated`、`listFields`、`inquiryLabelField`、`previewPath`、`ImageDerivative` |

### ContentStructure と TypeShape

```ts
type ContentStructure = Readonly<{ models: readonly ModelStructure[] }>;
type ModelStructure = Readonly<{ key: ModelKey; cardinality: "single" | "collection"; fields: readonly FieldStructure[] }>;
type FieldStructure = Readonly<{ key: FieldKey; type: FieldTypeStructure }>;

type FieldTypeStructure =
  | Readonly<{ kind: "text" | "number" | "boolean" | "date" | "datetime" | "richText" | "media"; required: boolean }>
  | Readonly<{ kind: "select" | "multiSelect"; required: boolean; options: readonly OptionKey[] }>
  | Readonly<{ kind: "reference" | "referenceList"; required: boolean; models: readonly ModelKey[] }>
  | Readonly<{ kind: "group"; fields: readonly FieldStructure[] }>
  | Readonly<{ kind: "repeat"; required: boolean; fields: readonly FieldStructure[] }>
  | Readonly<{ kind: "blocks"; required: boolean; blockTypes: readonly BlockTypeStructure[] }>;

type BlockTypeStructure = Readonly<{ key: BlockTypeKey; fields: readonly FieldStructure[] }>;
type FieldKind = FieldTypeStructure["kind"];

type TypeShape = Readonly<{
  models: readonly ModelStructure[];
  forms: readonly FormShape[];
  derivatives: readonly DerivativeKey[];
}>;
type FormShape = Readonly<{ key: FormKey; fields: readonly FormFieldShape[] }>;
type FormFieldShape = Readonly<{
  key: FieldKey;
  required: boolean;
  type:
    | Readonly<{ kind: "text" | "number" | "boolean" | "date" | "datetime" }>
    | Readonly<{ kind: "select" | "multiSelect"; options: readonly OptionKey[] }>
    | Readonly<{ kind: "contentReference"; models: readonly ModelKey[] }>;
}>;

const ContentStructure: {
  empty(): ContentStructure;
  equals(a: ContentStructure, b: ContentStructure): boolean;
  diff(expected: ContentStructure, actual: ContentStructure): readonly StructureDifference[];
};

type StructureDifference = Readonly<{
  ref: StructureRef;
  difference: "missing" | "unexpected" | "kind" | "required" | "cardinality" | "models";
}>;
```

- 等価性は順序によらない。モデル、項目、選択肢、ブロックの種類、許可するモデルを、キーの集合として比べる
- `diff` は、`expected` にあって `actual` に無い定義を `missing`、その逆を `unexpected`、同じ位置で異なる属性をその属性の名前で返す。空は一致を表す
- `TypeShape` の `models` は `ContentStructure` の `models` と同じ。コンテンツの構造は、型の形のうちコンテンツに関わるものから派生形式のキーを除いたものになる

### 位置

```ts
type PathStep =
  | Readonly<{ type: "field"; key: FieldKey }>
  | Readonly<{ type: "blockType"; key: BlockTypeKey }>;
type LevelPath = Readonly<{ model: ModelKey; steps: readonly PathStep[] }>;

type StructureRef =
  | Readonly<{ target: "model"; model: ModelKey }>
  | Readonly<{ target: "field"; level: LevelPath; field: FieldKey }>
  | Readonly<{ target: "option"; level: LevelPath; field: FieldKey; option: OptionKey }>
  | Readonly<{ target: "blockType"; level: LevelPath; field: FieldKey; blockType: BlockTypeKey }>;

type DefinitionRef =
  | StructureRef
  | Readonly<{ target: "form"; form: FormKey }>
  | Readonly<{ target: "formField"; form: FormKey; field: FieldKey }>
  | Readonly<{ target: "formOption"; form: FormKey; field: FieldKey; option: OptionKey }>
  | Readonly<{ target: "derivative"; derivative: DerivativeKey }>;

type ValuePath = readonly (Readonly<{ field: FieldKey }> | Readonly<{ index: number }>)[];
```

- `LevelPath` は階層を指す。`steps` が空ならモデルの直下。`field` の step は、項目のまとまり・繰り返し・ブロックの項目へ入る。`blockType` の step は、ブロックの項目の step の直後に置き、そのブロックの種類へ入る
- `StructureRef` はコンテンツの構造の定義を、`DefinitionRef` はスキーマの定義を指す
- `ValuePath` は、一件の内容の中の値を指す。`index` は、繰り返しの要素とブロックの位置（0 始まり）

### SchemaFingerprint

```ts
type SchemaFingerprint = string & { readonly [schemaFingerprintBrand]: true };

type SchemaFingerprints = Readonly<{
  schema: SchemaFingerprint;
  models: Readonly<Record<string, SchemaFingerprint>>;
  forms: Readonly<Record<string, SchemaFingerprint>>;
}>;

type ClaimedFingerprints = Readonly<{
  schema: SchemaFingerprint | null;
  models: Readonly<Record<string, SchemaFingerprint>>;
  forms: Readonly<Record<string, SchemaFingerprint>>;
}>;

type FingerprintMatch = Readonly<{
  target: Readonly<{ type: "schema" }> | Readonly<{ type: "model"; key: string }> | Readonly<{ type: "form"; key: string }>;
  matches: boolean;
}>;

const SchemaFingerprint: {
  of(schema: Schema): SchemaFingerprints;
};
```

`SchemaFingerprint` は、次の対象の要約（「正規化した JSON と要約」）。並べ替える配列は、モデル・フォーム・項目・選択肢・ブロックの種類・許可するモデル・派生形式で、どれもキーの昇順に並べる。

| 識別子 | 対象 |
| --- | --- |
| スキーマ全体 | `Schema.typeShapeOf(schema)` |
| モデルごと | そのモデルの `ModelStructure` と、派生形式のキー。派生形式のキーは、モデルがメディアまたはリッチテキストの項目を、入れ子を含めて一つでも持つ場合に含め、持たない場合は空にする |
| フォームごと | そのフォームの `FormShape` |

- 含むものは、モデル・フォーム・項目・選択肢・ブロックの種類のキー、単一と複数件の別、値の種類、必須、入れ子の構成、許可するブロックの種類、参照・問い合わせ対象に許可するモデルのキー、派生形式のキー
- 含まないものは、制約と検証（`constraints`、`mustMatch`、`mustBeTrue`）、一意性、表示・運用の定義、派生形式の `size` と `output`
- モデル・フォームごとの識別子は、参照・問い合わせ対象に許可するモデルをキーとして含み、そのモデルの型の形を含まない

### 正規化した JSON と要約

`SchemaFingerprint`、`ApplyRequest.digestOf`、`MigrationSequence.digest` が使う、値の要約の求め方。

- 正規化した JSON は、オブジェクトのプロパティを名前の昇順に並べ、空白を含めない。配列は、要約を使う側が並べ替えると定めた配列だけを並べ替え、ほかの配列の順序を保つ
- 要約は、正規化した JSON の UTF-8 のバイト列の SHA-256 を、64 文字の小文字の 16 進で表した文字列
- SHA-256 は domain の純粋な関数として持ち、実行環境の API を使わない

### 値

```ts
type LeafValues = Readonly<{ richText: unknown; media: unknown }>;

type FieldValue<L extends LeafValues> =
  | Readonly<{ kind: "text"; value: string }>
  | Readonly<{ kind: "number"; value: number }>
  | Readonly<{ kind: "boolean"; value: boolean }>
  | Readonly<{ kind: "date"; value: CalendarDate }>
  | Readonly<{ kind: "datetime"; value: Instant }>
  | Readonly<{ kind: "select"; value: OptionKey }>
  | Readonly<{ kind: "multiSelect"; value: readonly OptionKey[] }>
  | Readonly<{ kind: "richText"; value: L["richText"] }>
  | Readonly<{ kind: "media"; value: L["media"] }>
  | Readonly<{ kind: "reference"; value: ContentId }>
  | Readonly<{ kind: "referenceList"; value: readonly ContentId[] }>
  | Readonly<{ kind: "group"; fields: FieldValues<L> }>
  | Readonly<{ kind: "repeat"; items: readonly FieldValues<L>[] }>
  | Readonly<{ kind: "blocks"; blocks: readonly Readonly<{ type: BlockTypeKey; fields: FieldValues<L> }>[] }>;

type FieldValues<L extends LeafValues> = Readonly<Record<string, FieldValue<L>>>;

type LiteralValue = Extract<
  FieldValue<LeafValues>,
  { kind: "text" | "number" | "boolean" | "date" | "datetime" | "select" | "multiSelect" }
>;
```

- `FieldValues` は、一つの階層の項目のキーから値への対応。未設定の項目はキーを持たない
- `text` の空文字列、`multiSelect`・`referenceList` の空の組、`repeat`・`blocks` の空の並びは持たない。未設定で表す
- `number` は有限の数
- `group` のキーが無いことは、中の項目がすべて未設定であることと同じ
- リッチテキストとメディアの値の形は、`LeafValues` で、使う側のドメインが与える。Schema は中身を解釈しない
- `LiteralValue` は、補完値に使える値。テキスト、数値、真偽値、日付、日時、単一選択、複数選択に限る。メディア、コンテンツ参照、リッチテキスト、項目のまとまり、繰り返し、ブロックは、型が補完値に指定できなくしている。移行の操作は、値を、その内容が既に使うメディアのほかを参照する値にしないので、反映の確定はメディアを公開しない

### Migration

```ts
type Migration = Readonly<{ name: MigrationName; operations: readonly MigrationOperation[] }>;
type Unit = readonly MigrationOperation[];
type OperationRef = Readonly<{ unit: number; index: number }>;
type MigrationDigest = string & { readonly [migrationDigestBrand]: true };

type NewFieldType =
  | Readonly<{ kind: "text" | "number" | "boolean" | "date" | "datetime" | "richText" | "media" | "repeat" | "blocks"; required: boolean }>
  | Readonly<{ kind: "select" | "multiSelect"; required: boolean; options: readonly OptionKey[] }>
  | Readonly<{ kind: "reference" | "referenceList"; required: boolean; models: readonly ModelKey[] }>
  | Readonly<{ kind: "group" }>;

type ConvertedFieldType =
  | Readonly<{ kind: "text" | "number" | "boolean" | "date" | "datetime" }>
  | Readonly<{ kind: "select" | "multiSelect"; options: readonly OptionKey[] }>
  | Readonly<{ kind: "reference" | "referenceList"; models: readonly ModelKey[] }>;

type MigrationOperation =
  | Readonly<{ op: "addModel"; model: ModelKey; cardinality: "single" | "collection" }>
  | Readonly<{ op: "renameModel"; from: ModelKey; to: ModelKey }>
  | Readonly<{ op: "removeModel"; model: ModelKey; lossConfirmed: true }>
  | Readonly<{ op: "addField"; level: LevelPath; field: FieldKey; type: NewFieldType; fill: LiteralValue | null }>
  | Readonly<{ op: "renameField"; level: LevelPath; from: FieldKey; to: FieldKey }>
  | Readonly<{
      op: "changeRequired";
      level: LevelPath;
      field: FieldKey;
      change: Readonly<{ to: "required"; fill: LiteralValue | null }> | Readonly<{ to: "optional" }>;
    }>
  | Readonly<{ op: "changeReferenceModels"; level: LevelPath; field: FieldKey; models: readonly ModelKey[] }>
  | Readonly<{ op: "changeFieldKind"; level: LevelPath; field: FieldKey; to: ConvertedFieldType; lossConfirmed: true }>
  | Readonly<{ op: "removeField"; level: LevelPath; field: FieldKey; lossConfirmed: true }>
  | Readonly<{ op: "addOption"; level: LevelPath; field: FieldKey; option: OptionKey }>
  | Readonly<{ op: "renameOption"; level: LevelPath; field: FieldKey; from: OptionKey; to: OptionKey }>
  | Readonly<{
      op: "removeOption";
      level: LevelPath;
      field: FieldKey;
      option: OptionKey;
      replacement: Readonly<{ to: "option"; option: OptionKey }> | Readonly<{ to: "unset"; lossConfirmed: true }>;
    }>
  | Readonly<{ op: "addBlockType"; level: LevelPath; field: FieldKey; blockType: BlockTypeKey }>
  | Readonly<{ op: "renameBlockType"; level: LevelPath; field: FieldKey; from: BlockTypeKey; to: BlockTypeKey }>
  | Readonly<{ op: "removeBlockType"; level: LevelPath; field: FieldKey; blockType: BlockTypeKey; lossConfirmed: true }>
  | Readonly<{ op: "copyValue"; level: LevelPath; from: FieldKey; to: FieldKey }>;
```

移行定義は、名前と操作の並びだけを持つデータ。操作は、キーで対象を指す値だけからなる。この直和型に無い操作は表現できず、境界のデコードが、該当する操作を示す `ValidationError` として拒否する。

追加の操作は、中の項目を持たない定義を加える。`addModel` は項目を持たないモデルを、`addField` の `group`・`repeat` は項目を持たない入れ子を、`blocks` はブロックの種類を持たない項目を、`addBlockType` は項目を持たないブロックの種類を加える。中の項目は、後に続く `addField` が加える。単一選択・複数選択の `addField` は、選択肢を `options` に持つ。

スキーマの変更と操作の対応は次のとおり。

| スキーマの変更 | 操作 | 記録する判断 |
| --- | --- | --- |
| モデル・項目・選択肢・ブロックの種類の追加 | `addModel`、`addField`、`addOption`、`addBlockType` | `addField` の `fill`（補完値） |
| 同じ階層での削除と追加 | `renameModel`、`renameField`、`renameOption`、`renameBlockType`、または削除の操作と追加の操作 | どちらの操作にしたか |
| 任意項目の必須化、必須項目の任意化 | `changeRequired` | `change.fill`（補完値） |
| 参照を許可するモデルの変更 | `changeReferenceModels` | なし |
| 変換規則のある組み合わせでの値の種類の変更 | `changeFieldKind` | `lossConfirmed` |
| 変換規則のない組み合わせでの値の種類の変更 | 同じキーの `removeField` と `addField` | `removeField` の `lossConfirmed` |
| 選択肢の削除 | `removeOption` | `replacement`（置き換え先）。未設定では `lossConfirmed` |
| ブロックの種類の削除 | `removeBlockType` | `lossConfirmed` |
| 項目・モデルの削除 | `removeField`、`removeModel` | `lossConfirmed` |
| 繰り返しの内外をまたぐ構造の組み替え | 元の階層の `removeField` と、先の階層の `addField` | `removeField` の `lossConfirmed` |
| 単一モデルと複数件モデルの間の変更 | 同じキーの `removeModel` と `addModel`、項目ごとの `addField` | `removeModel` の `lossConfirmed` |
| 値の複写（生成の確認で加える、または開発者が書き加える） | `copyValue` | 生成では `copy` の答え |

- 生成時の判断は、操作の種類と引数として記録する。データを失い得る操作の確認は、その操作の `lossConfirmed: true` として記録する。確認の記録が無い、データを失い得る操作は表現できない
- `copyValue` は、複写元と複写先を一つの `level` で指す。別の階層の項目は指せない
- データを失い得る操作は、`removeModel`、`removeField`、`removeBlockType`、`changeFieldKind`、`replacement.to` が `unset` の `removeOption`

```ts
type LossKind = "contents" | "field" | "blocks" | "optionValues" | "unconvertibleValues";
type LossyOperationBase = Readonly<{ ref: OperationRef; operation: MigrationOperation; loss: LossKind }>;
type LossyOperationOf = Readonly<{
  migrations: LossyOperationBase & Readonly<{ migration: MigrationName }>;
  direct: LossyOperationBase;
}>;
type LossyOperation<S extends ApplySource = ApplySource> = LossyOperationOf[S];
```

| 操作 | `loss` | 失うもの |
| --- | --- | --- |
| `removeModel` | `contents` | そのモデルのコンテンツと、ほかのコンテンツが持つ、そのコンテンツへの参照 |
| `removeField` | `field` | その項目と値 |
| `removeBlockType` | `blocks` | その種類のブロック |
| `removeOption`（未設定へ） | `optionValues` | その選択肢を選んだ値 |
| `changeFieldKind` | `unconvertibleValues` | 変換できない値 |

`LossyOperation` は、操作の求め方（`ApplySource`）で形が決まる。移行定義の操作は、その移行定義の名前（`migration`）を持つ。直接反映の操作は、移行定義の名前を持たない。

`OperationRef` は、その値を求めた関数（`StructureMigration.fold`、`StructureMigration.lossyOperations`、`ValueMigration.apply`）に渡した単位の列の中の操作を指す。`unit` は列の中の単位の位置、`index` は単位の中の操作の位置で、どちらも 0 始まり。移行定義の列を渡す場合は、移行定義一つが一つの単位になる。計画から求める値は計画の `units` を渡して求めるので、`source` が `migrations` の計画では、`unit` は `plan.migrations` の同じ位置の移行定義を指す。`duplicateName` の `positions` は、`MigrationSequence.issues` に渡した列の中の位置。

#### 順序と一致の判定

```ts
const MigrationSequence: {
  digest(operations: readonly MigrationOperation[]): MigrationDigest;
  issues(local: readonly Migration[]): readonly MigrationIssue[];
  compare(applied: readonly AppliedMigration[], local: readonly Migration[]): SequenceComparison;
};

type SequenceComparison =
  | Readonly<{ kind: "consistent"; pending: readonly Migration[] }>
  | Readonly<{ kind: "diverged"; divergences: readonly SequenceDivergence[] }>;

type SequenceDivergence =
  | Readonly<{ type: "changed"; position: number; applied: MigrationName }>
  | Readonly<{ type: "missingLocally"; position: number; applied: MigrationName }>
  | Readonly<{ type: "insertedBefore"; position: number; applied: MigrationName; inserted: readonly MigrationName[] }>;
```

- 移行定義の順序は、列（配列）の中の位置で決まる。名前は列の中で一意で、表示と範囲の指定に使う
- `issues` は、手元の列の移行定義の不整合を返す。同じ名前の移行定義が二つ以上ある名前ごとの `duplicateName` と、`local` の全体を空の構造へ適用した `StructureMigration.fold` の `issues` を、この順に並べる。空は不整合が無いことを表す。手元の列の不整合は、この関数だけが求める
- `digest` は、操作の並びの要約（「正規化した JSON と要約」）。並べ替える配列は `models` と `options` で、キーの昇順に並べる。操作の順序を保つ。名前は含めない
- 一致の判定は `digest` の一致で行う。操作の並びを変えない記述の変更と、名前の変更は、移行定義の変更にならない
- `compare` は、適用済みの i 番目の `digest` が、手元の i 番目の `digest` とすべて一致する場合に `consistent` を返す。`pending` は、手元の列のうち適用済みの個数より後ろの移行定義
- 一致しない位置があれば `diverged` を返す。一致しない最初の位置から、適用済みの移行定義ごとに、手元に同じ名前が無ければ `missingLocally`、同じ名前が同じ位置にあれば `changed`、同じ名前が後ろの位置にあれば `insertedBefore`（`inserted` は、その前に入った手元の移行定義）とする。同じ名前が前の位置にある場合は `changed` とする

## ドメインサービス

どのサービスもポートに依存しない。

### SchemaValidation

スキーマ自体の不整合を検出する。

```ts
const SchemaValidation: {
  validate(schema: Schema): readonly SchemaIssue[];
};

type SchemaIssue = Readonly<{ ref: DefinitionRef; reason: SchemaIssueReason }>;

type SchemaIssueCategory =
  | "duplicateKey"
  | "unknownReference"
  | "inapplicableConstraint"
  | "defaultMismatch"
  | "displayFieldMismatch"
  | "deprecatedRequired";

const SchemaIssue: {
  categoryOf(reason: SchemaIssueReason): SchemaIssueCategory;
};
```

`validate` はスキーマ全体を調べ、見つけた不整合をすべて返す。空は不整合が無いことを表す。`SchemaIssueReason` は、次の表の `reason` の直和。不整合は `reason` だけを持つ。分類（`SchemaIssueCategory`）は、`SchemaIssue.categoryOf` が、次の表のとおりに `reason` から求める。

| 分類（`categoryOf` の結果） | `reason` | 条件 |
| --- | --- | --- |
| `duplicateKey` | `duplicateKey` | 「キーと名前」の一意である範囲の中で、同じキーが二つ以上ある |
| `unknownReference` | `unknownModel` | コンテンツ参照・問い合わせ対象の `models` が、スキーマに無いモデルのキーを含む |
| `unknownReference` | `unknownField` | `listFields`、`inquiryLabelField`、`previewPath` の `uniqueValue`、`mustMatch` が、その階層・フォームに無い項目のキーを指す |
| `inapplicableConstraint` | `invalidRange` | 範囲・文字数・件数・選択数の `min` が `max` より大きい |
| `inapplicableConstraint` | `invalidPattern` | `pattern` が正規表現として解釈できない、または長さの上限を超える |
| `inapplicableConstraint` | `uniqueNotAtModelLevel` | `unique` が、モデル直下でない項目に付いている |
| `inapplicableConstraint` | `countExceedsOptions` | 複数選択の選択数の `min` が、選択肢の数より大きい |
| `inapplicableConstraint` | `mustMatchKindDiffers` | `mustMatch` が、自分自身または `kind` の異なる項目を指す |
| `defaultMismatch` | `defaultNotInOptions` | 単一選択・複数選択の `defaultValue` が、その項目の選択肢に無いキーを含む |
| `displayFieldMismatch` | `listFieldKind` | `listFields` が、テキスト・数値・真偽値・日付・日時・単一選択のいずれでもない項目を指す |
| `displayFieldMismatch` | `inquiryLabelFieldKind` | `inquiryLabelField` が、テキストでない項目を指す |
| `displayFieldMismatch` | `previewFieldNotUnique` | `previewPath` の `uniqueValue` が、`unique` を持たない項目を指す |
| `displayFieldMismatch` | `previewPathShape` | `previewPath` の先頭が `/` で始まる `literal` でない、または `literal` の値が `?` か `#` を含む |
| `deprecatedRequired` | `deprecatedRequired` | `required` が `true` の項目に `deprecated` が付いている |

値の種類に合わない初期値と、値の種類に無い制約は、型が表現できなくしている。

### StructureMigration

移行定義をコンテンツの構造へ適用し、移行定義の不整合を検出する。

```ts
const StructureMigration: {
  fold(base: ContentStructure, units: readonly Unit[]): FoldResult;
  lossyOperations(units: readonly Unit[], names: readonly MigrationName[]): readonly LossyOperation<"migrations">[];
  lossyOperations(units: readonly Unit[]): readonly LossyOperation<"direct">[];
  modelFates(base: ContentStructure, units: readonly Unit[]): readonly ModelFate[];
};

type FoldResult = Readonly<{ structure: ContentStructure; issues: readonly MigrationIssue[] }>;
type MigrationIssue =
  | Readonly<{ ref: OperationRef; reason: "unknownKey"; missing: StructureRef }>
  | Readonly<{ ref: OperationRef; reason: "noConversionRule"; from: FieldKind; to: FieldKind }>
  | Readonly<{ ref: OperationRef; reason: "keyExists" | "notApplicable" | "fillMismatch" }>
  | Readonly<{ reason: "duplicateName"; name: MigrationName; positions: readonly number[] }>;
type ModelFate =
  | Readonly<{ from: ModelKey; fate: "kept"; to: ModelKey }>
  | Readonly<{ from: ModelKey; fate: "removed"; by: OperationRef }>;
```

`fold` は、操作を順に `base` へ適用する。不整合のある操作は構造を変えずに飛ばし、`issues` に加える。`issues` が空の場合に限り、`structure` は適用の結果として使える。開発環境は、空の構造から移行定義の列の全体を適用する。

操作ごとの、構造への適用と不整合は次のとおり。どの操作も、指す階層・項目・選択肢・ブロックの種類・モデルがその時点の構造に無ければ `unknownKey` とする。加える、または変更後のキーがその範囲に既にあれば `keyExists` とする。

| 操作 | 構造への適用 | そのほかの不整合 |
| --- | --- | --- |
| `addModel` | 項目を持たないモデルを加える | なし |
| `renameModel` | モデルのキーを変える。コンテンツ参照の `models` にある同じキーも変える | なし |
| `removeModel` | モデルを取り除く。コンテンツ参照の `models` から、そのキーを取り除く | なし |
| `addField` | 階層に項目を加える | `fill` が `null` でなく、加える項目に合う補完値でなければ `fillMismatch` |
| `renameField` | 項目のキーを変える | なし |
| `changeRequired` | 項目の必須を変える | 項目が `group` なら `notApplicable`。`fill` は `addField` と同じ |
| `changeReferenceModels` | `models` を置き換える | 項目がコンテンツ参照でなければ `notApplicable`。構造に無いモデルを含めば `unknownKey` |
| `changeFieldKind` | 値の種類と、`options` または `models` を置き換える。必須は保つ | 変換元と変換先の値の種類が同じなら `notApplicable`。変換規則が無い組み合わせなら `noConversionRule` |
| `removeField` | 項目を、入れ子の中の定義とともに取り除く | なし |
| `addOption` / `renameOption` / `removeOption` | 選択肢を加える、キーを変える、取り除く | 項目が単一選択・複数選択でなければ `notApplicable`。置き換え先が、取り除く選択肢自身または項目に無い選択肢なら `unknownKey` |
| `addBlockType` / `renameBlockType` / `removeBlockType` | ブロックの種類を加える、キーを変える、取り除く | 項目が `blocks` でなければ `notApplicable` |
| `copyValue` | 構造を変えない | `from` と `to` が同じなら `notApplicable`。変換規則が無い組み合わせなら `noConversionRule` |

- 項目に合う補完値は、項目の値の種類が補完値を持てる値の種類（`LiteralValue` の `kind`）で、補完値がその値の種類の値であり、単一選択・複数選択では、補完値のキーがすべて項目の選択肢にあるもの。この条件は、ここだけが定める
- `fillMismatch` は、単位の適用を終えた構造でも確かめる。補完値を入れる時点の項目に合わない補完値は `fillMismatch` とする
- 操作の不整合の理由は `unknownKey`、`keyExists`、`notApplicable`、`noConversionRule`、`fillMismatch`。どれも、該当する操作を `ref` で示す
- `unknownKey` の `missing` は、操作が指す定義のうち、その時点の構造に無いもの。階層（`level`）の途中が無い場合は、無い最初の定義を指す。値の複写の複写元が同じ階層に無い場合は、その階層の `from` の項目を指す
- `noConversionRule` の `from` と `to` は、その時点の構造での値の種類。値の複写では複写元と複写先、値の種類の変更では変換元と変換先
- `MigrationOperation` に無い操作は、型と境界のデコードが拒否する
- スキーマからキーを削除しても、列の途中の構造は変わらないので、そのキーを扱う既存の移行定義は不整合にならない
- `lossyOperations` は、データを失い得る操作を、列の順に返す。`names` は単位ごとの移行定義の名前で、`migration` に入れる。直接反映の操作では `names` を渡さず、結果は `migration` を持たない
- `modelFates` は、`base` のモデルごとに、`units` を適用した後もキーの変更をたどって残るか（`kept`）、取り除かれるか（`removed`）を返す。`removed` の `by` は、そのモデルを取り除いた `removeModel`。取り除いた後に同じキーで加えたモデルは、別のモデルとして `removed` にする

スキーマと移行定義の食い違いは、`ContentStructure.diff(fold(ContentStructure.empty(), units).structure, Schema.structureOf(schema))` で検出する。

### MigrationGeneration

コンテンツの構造の差分から、操作の並びを生成する。

```ts
const MigrationGeneration: {
  generate(from: ContentStructure, to: Schema, decisions: readonly Decision[]): GenerationResult;
  directOperations(from: ContentStructure, to: ContentStructure): readonly MigrationOperation[];
};

type GenerationResult =
  | Readonly<{ kind: "noChange" }>
  | Readonly<{ kind: "invalidDecisions"; issues: readonly DecisionIssue[] }>
  | Readonly<{ kind: "needsDecisions"; questions: readonly Question[] }>
  | Readonly<{ kind: "generated"; operations: readonly MigrationOperation[] }>;

type DecisionIssue = Readonly<{ decision: Decision; reason: "unmatched" | "outOfRange" | "duplicate" }>;

type Question =
  | Readonly<{ type: "rename"; subject: StructureRef; candidates: readonly string[] }>
  | Readonly<{ type: "copy"; subject: StructureRef; candidates: readonly FieldKey[] }>
  | Readonly<{ type: "fill"; subject: StructureRef; kind: FieldKind }>
  | Readonly<{ type: "optionReplacement"; subject: StructureRef; candidates: readonly OptionKey[] }>
  | Readonly<{ type: "confirmLoss"; subject: StructureRef; cause: LossCause }>;

type LossCause =
  | Readonly<{ type: "removed" }>
  | Readonly<{ type: "optionUnset" }>
  | Readonly<{ type: "kindChangeWithRule"; rule: ConversionRule }>
  | Readonly<{ type: "kindChangeWithoutRule"; from: FieldKind; to: FieldKind }>
  | Readonly<{ type: "cardinalityChange" }>
  | Readonly<{ type: "movedAcrossLevels"; to: StructureRef }>;

type Decision =
  | Readonly<{ type: "rename"; subject: StructureRef; to: string | null }>
  | Readonly<{ type: "copy"; subject: StructureRef; from: FieldKey | null }>
  | Readonly<{ type: "fill"; subject: StructureRef; value: LiteralValue | null }>
  | Readonly<{ type: "optionReplacement"; subject: StructureRef; replacement: OptionKey | null }>
  | Readonly<{ type: "confirmLoss"; subject: StructureRef }>;
```

`from` は、既存の移行定義を順に適用した構造。`to` は、変更後のスキーマ。差分は、`from` と `Schema.structureOf(to)` の差分。`generate` は、次の順に結果を決める。

1. 差分が無ければ `noChange`
2. 問いに合わない判断があれば `invalidDecisions`。操作を生成しない
3. 判断が要る箇所のうち `decisions` に答えの無いものがあれば、その問いを `needsDecisions` で返す
4. すべてに答えがあれば `generated`

問いに合わない判断は次のとおり。

| `reason` | 条件 |
| --- | --- |
| `duplicate` | 同じ `type` と `subject` の判断が二つ以上ある。二つ目以降が当たる |
| `outOfRange` | 生じた問いへの答えが、取り得る範囲に無い。`rename` の `to` と `copy` の `from` が候補に無いキー、`optionReplacement` の `replacement` が候補に無い選択肢、`fill` の `value` が、`to` の項目に合う補完値（「StructureMigration」）でない |
| `unmatched` | 答えの無い問いが一つも残っていない時点で、どの問いにも使わなかった判断。同じ `type` と `subject` の問いが生じない |

- 答えの無い問いが残っている間は、使わなかった判断を `unmatched` にしない。残りの問いの答えによって、その判断の問いが生じ得る
- `generated` の操作を `StructureMigration.fold(from, [operations])` で適用した構造は、`Schema.structureOf(to)` と一致する

対話する環境は、`needsDecisions` の問いに答えて `generate` を呼び直す。答えによって新しい問いが生じるので、`generated` になるまで繰り返す。対話できない環境は、`Decision` の組を指定して一度呼ぶ。`needsDecisions` が返れば、指定が必要な判断として問いを示し、生成しない。対話と指定は、同じ `Decision` を使う。

判断が要る箇所は次のとおり。`subject` は、取り除く定義では `from` の位置、加える定義では `to` の位置で指す。

| 問い | 生じる条件 | 答え |
| --- | --- | --- |
| `rename` | 同じ範囲で、`from` にだけあるキーと、`to` にだけある候補がある | `to` に候補のキーを指定するとキーの変更、`null` は削除と追加 |
| `copy` | 項目を加える（`rename` の答えに使った項目を除く）。同じ階層に、取り除く項目、または非推奨の項目があり、そのうちに加える項目へ変換規則のある項目がある | `from` に候補の項目のキーを指定すると、その項目を複写元とし、加える項目を複写先とする値の複写を加える。`null` は加えない |
| `fill` | 必須の項目を加える、または任意の項目を必須にする。項目の値の種類が補完値を持てる | 補完値、または `null`（補完しない） |
| `optionReplacement` | 選択肢を取り除く（キーの変更にしなかったものを含む） | 置き換え先の選択肢、または `null`（未設定にする） |
| `confirmLoss` | 項目・モデル・ブロックの種類を取り除く、値の種類を変える、単一と複数件の別を変える、選択肢の置き換え先を未設定にする | 確認した |

- `rename` の候補は、モデルでは `cardinality` が同じモデル、項目では値の種類が同じ項目、選択肢とブロックの種類では同じ項目に加わるキー。候補は `to` にだけあるキーに限る
- `copy` の候補は、同じ階層の、取り除く項目（`from` にだけあり、`rename` の答えに使わなかったもの）と、`to` で非推奨の項目（`from` にもあるもの）のうち、その値の種類から加える項目の値の種類への変換規則（`ValueConversion.rule`）があるもの。値の種類が同じ項目と異なる項目の両方を含む。`rename` の問いが生じた項目は、その答えの後に候補になる。非推奨の項目は、非推奨にした時期によらず候補になる
- 取り除く定義（モデル、項目、ブロックの種類）の中にある項目・選択肢・ブロックの種類には、問いも操作も生じない。中の定義は、取り除く定義の操作（`removeModel`、`removeField`、`removeBlockType`）がともに取り除く
- キーの変更にした対は、同じ定義として、必須・値の種類・選択肢・入れ子の差分を続けて求める
- 補完値を持てる値の種類は、`LiteralValue` の `kind`
- `confirmLoss` の `cause` は、値の種類の変更に変換規則があれば `kindChangeWithRule`、無ければ `kindChangeWithoutRule`、単一と複数件の別の変更は `cardinalityChange`、階層をまたぐ組み替えは `movedAcrossLevels`、選択肢の置き換え先を未設定にする答えは `optionUnset`、そのほかは `removed`
- `optionReplacement` に `null` を答えると、同じ `subject`（取り除く選択肢）に、`cause` が `optionUnset` の `confirmLoss` の問いが生じる。別の選択肢への置き換えは値を保つので、確認の問いは生じない。データを失い得る操作の確認は、どれも `confirmLoss` の答えで行う
- `kindChangeWithRule` は、変換元と変換先の値の種類と、適用する変換規則（`ConversionRule`）を持つ。開発者に、適用する規則と、変換できない値を失うことを示す
- `movedAcrossLevels` は、同じモデルの中で、取り除く項目と同じキーの項目が、別の階層に加わる場合に当たる。繰り返しの内外をまたぐ組み替えがこれになる。`to` は、加わる項目の位置
- 問いとともに開発者に示す案内は「Guidance」が定める
- `confirmLoss` に答えの無い、データを失い得る操作は生成しない

生成する操作の順序は次のとおり。同じ引数からは、同じ操作の並びを返す。同じ段の中は、階層の浅い順、キーの昇順に並べる。

1. 同じキーの `removeModel`・`addModel` の対（単一と複数件の別の変更）
2. `addModel`
3. `renameModel`
4. モデルごとに、階層の浅い順に、`renameField`、`renameOption`、`renameBlockType`
5. `changeFieldKind` と、同じキーの `removeField`・`addField` の対（変換規則のない値の種類の変更）
6. `addField`、`addOption`、`addBlockType`
7. `copyValue`、`changeReferenceModels`、`changeRequired`
8. `removeOption`、`removeBlockType`、`removeField`
9. `removeModel`

`removeModel` は、ほかのモデルのコンテンツ参照の `models` から、そのキーを取り除く（`StructureMigration.fold`）。段 1 の対で取り除いて加えたモデルを、`to` のコンテンツ参照が `models` に持つ場合、その項目ごとに、`to` の `models` を指定する `changeReferenceModels` を段 7 に並べる。

同じキーの削除と追加の対は、同じ階層の中で、同じキーが削除と追加の両方に現れる組を指す（段 1 のモデルと、段 5 の項目）。この対は、どの階層でも削除を先に並べる。階層をまたぐ組み替え（`movedAcrossLevels`）は、削除する項目と追加する項目の階層が違うので、対に当たらない。追加は段 6、削除は段 8 に並び、追加が先になる。対に当たらない追加は、どれも削除より前に並ぶ。`copy` の答えの `copyValue` は段 7 に並び、追加の後、削除の前になる。開発者は、追加と削除の間に `copyValue` を書き加えることもできる。

`directOperations` は、直接反映の操作を返す。差分が無ければ空の列を返す。構造が `to` のスキーマを渡した `generate` に、キーの変更としない、値の複写を加えない、補完しない、置き換え先を未設定にする、損失を確認した、という答えを与えた `generated` の操作と同じになる。変換規則のある値の種類の変更は `changeFieldKind` になる。

### Guidance

問い、スキーマの不整合、反映の影響とともに、開発者に示す案内を定める。案内は種類だけを持ち、文面は示す側が決める。

```ts
type Guidance = "irreversibleLoss" | "stagedChange" | "optionalizeWithDeprecation";

const Guidance: {
  forQuestion(question: Question): readonly Guidance[];
  forSchemaIssue(issue: SchemaIssue): readonly Guidance[];
  forEffects(effects: ApplyEffects): readonly Guidance[];
};
```

| 案内 | 示す内容 | 返す対象 |
| --- | --- | --- |
| `irreversibleLoss` | 失う値・項目・コンテンツを移行では戻せず、戻す手段が設置全体のバックアップからの復旧に限られること | `forQuestion`: `confirmLoss` の問い。`forEffects`: `losses` に、`contents.total` または `values` が 1 以上の要素がある影響 |
| `stagedChange` | 値を保つ場合の、段階的変更の手順 | `forQuestion`: `cause` が `movedAcrossLevels` または `cardinalityChange` の `confirmLoss` の問い |
| `optionalizeWithDeprecation` | 任意化と非推奨の指定を、同じ反映で行う手順 | `forSchemaIssue`: `deprecatedRequired` の不整合 |

- 当たる案内が無い対象には、空の列を返す。一つの対象が複数の案内に当たる場合は、表の順に並べる
- 問いの案内は、問いだけから決まる

### ValueConversion

値の種類の変換規則を定める。値の種類の変更と、値の複写が使う。

```ts
const ValueConversion: {
  convert<L extends LeafValues>(
    value: FieldValue<L>,
    to: FieldTypeStructure,
  ): Readonly<{ converted: true; value: FieldValue<L> }> | Readonly<{ converted: false }>;
  rule(from: FieldKind, to: FieldKind): ConversionRule | null;
  rules(): readonly ConversionRule[];
};

type ConversionRule = Readonly<{ from: FieldKind; to: FieldKind; lossless: boolean }>;
```

- `rules` は、次の表の組み合わせのすべてを返す。CMS に接続しない「変換規則を一覧する」と、生成時の確認が使う
- `rules` は、変換元と変換先の値の種類の組ごとに、一件を返す。表の最後の行は、同じ値の種類の組ごとの規則になる
- `lossless` は、変換元の値の種類のどの値も変換できる規則で真。表の「変換できない値」が「なし」の行の規則が当たる。同じ値の種類の規則は、単一選択と複数選択で偽、ほかの値の種類で真

変換できる組み合わせと規則は、次の表がすべて。未設定は、どの変換でも未設定のままにする。

| 変換元 | 変換先 | 規則 | 変換できない値 |
| --- | --- | --- | --- |
| 数値 | テキスト | ECMAScript の `Number` の文字列表現にする | なし |
| 真偽値 | テキスト | `true` または `false` にする | なし |
| 日付 | テキスト | `YYYY-MM-DD` にする | なし |
| 日時 | テキスト | `YYYY-MM-DDTHH:mm:ss.sssZ` にする | なし |
| 単一選択 | テキスト | 選択肢のキーにする | なし |
| テキスト | 数値 | 前後の空白を除き、`-?(0\|[1-9][0-9]*)(\.[0-9]+)?` に一致する文字列を数にする | 一致しない文字列、有限の数にならない文字列 |
| テキスト | 真偽値 | 前後の空白を除き、`true` を真、`false` を偽にする | そのほかの文字列 |
| テキスト | 日付 | 前後の空白を除き、`YYYY-MM-DD` の形式の実在する暦日にする | そのほかの文字列 |
| テキスト | 日時 | 前後の空白を除き、`YYYY-MM-DDTHH:mm:ss` に、任意の小数部と、`Z` または `±HH:mm` が続く文字列を時点にする | そのほかの文字列 |
| テキスト | 単一選択 | 変換先の選択肢のキーと一致する文字列を、その選択肢にする | 一致する選択肢が無い文字列 |
| 日付 | 日時 | その暦日の UTC の 0 時にする | なし |
| 日時 | 日付 | UTC での暦日にする | なし |
| 単一選択 | 複数選択 | その選択肢だけを選んだ値にする | 変換先に無い選択肢 |
| 複数選択 | 単一選択 | 選んだ選択肢が一つなら、その選択肢にする | 二つ以上を選んだ値、変換先に無い選択肢 |
| コンテンツ参照（一件） | コンテンツ参照（複数件） | その一件だけの組にする | なし |
| コンテンツ参照（複数件） | コンテンツ参照（一件） | 一件なら、その参照にする | 二件以上の組 |
| 同じ値の種類（項目のまとまり、繰り返し、ブロックを除く） | 同じ値の種類 | そのまま写す | 単一選択・複数選択で、変換先に無い選択肢 |

- 変換規則を持つ値の種類は、表に現れるものだけ。リッチテキストとメディアの規則は、同じ値の種類どうしに限る
- 同じ値の種類の規則は、値の複写だけが使う。同じ値の種類への値の種類の変更は、移行定義の不整合（「StructureMigration」の `changeFieldKind`）
- 複数選択から複数選択への複写は、変換先に無い選択肢を一つでも含む値を変換できない値とする
- 変換は、変換先の制約と、参照を許可するモデルを確かめない

### ValueValidation

項目の値を定義に照らして検証する。コンテンツの項目とフォームの項目に共通する、葉の値の規則を一つだけ定める。保存データを読まない。

```ts
type LeafRule =
  | Readonly<{ kind: "text"; constraints: TextConstraints }>
  | Readonly<{ kind: "number"; constraints: RangeConstraints<number> }>
  | Readonly<{ kind: "boolean" }>
  | Readonly<{ kind: "date"; constraints: RangeConstraints<CalendarDate> }>
  | Readonly<{ kind: "datetime"; constraints: RangeConstraints<Instant> }>
  | Readonly<{ kind: "select"; options: readonly OptionKey[] }>
  | Readonly<{ kind: "multiSelect"; options: readonly OptionKey[]; constraints: CountConstraints }>;

type LevelField = Readonly<{ key: FieldKey; kind: FieldKind; required: boolean; rule: LeafRule | null }>;

type TypeErrorKind = "missingRequired" | "kindMismatch" | "optionMismatch" | "referenceMismatch";

type ConstraintKind = "minLength" | "maxLength" | "format" | "min" | "max" | "minCount" | "maxCount";

type ValueIssue = Readonly<{
  field: FieldKey;
  issue:
    | Readonly<{ type: "typeError"; error: TypeErrorKind }>
    | Readonly<{ type: "constraintViolation"; constraint: ConstraintKind }>;
}>;

type LevelValidation = Readonly<{ issues: readonly ValueIssue[]; undefinedKeys: readonly string[] }>;

const ValueValidation: {
  contentLevel(fields: readonly ContentField[]): readonly LevelField[];
  formLevel(fields: readonly FormField[]): readonly LevelField[];
  validateLevel<L extends LeafValues>(fields: readonly LevelField[], values: FieldValues<L>): LevelValidation;
};
```

- `contentLevel` と `formLevel` は、一つの階層の項目の定義を `LevelField` へ写す。テキスト、数値、真偽値、日付、日時、単一選択、複数選択の項目は `rule` を持つ。そのほかの値の種類の項目は `rule` が `null` になる。項目のまとまりの `required` は `false`、フォームの `contentReference` の `kind` は `reference` とする
- `validateLevel` は、値の一つの階層を検証し、見つけた問題をすべて返す。項目の位置は `field` で示す。入れ子の中の位置（`ValuePath`）は、階層をたどる呼び出し側が組み立てる
- `undefinedKeys` は、`values` にあって `fields` に無いキー

`validateLevel` の規則は次のとおり。項目ごとに、上から順に確かめ、型を満たさない入力エラーが見つかった項目は、制約を確かめない。

| 順 | 対象 | 条件 | 結果 |
| --- | --- | --- | --- |
| 1 | すべての項目 | `required` が `true` で、値が未設定 | `typeError`（`missingRequired`） |
| 2 | すべての項目 | 値の `kind` が、項目の `kind` と異なる | `typeError`（`kindMismatch`） |
| 3 | 単一選択、複数選択 | 値が、`options` に無いキーを含む | `typeError`（`optionMismatch`） |
| 4 | テキスト | コードポイントの数が `minLength` より少ない、`maxLength` より多い | `constraintViolation`（`minLength`、`maxLength`） |
| 4 | テキスト | 値が `format` に合わない | `constraintViolation`（`format`） |
| 4 | 数値、日付、日時 | 値が `min` より小さい、`max` より大きい | `constraintViolation`（`min`、`max`） |
| 4 | 複数選択 | 選んだ数が `min` より少ない、`max` より多い | `constraintViolation`（`minCount`、`maxCount`） |

- 未設定で必須でない項目は、問題を持たない
- 範囲と数の境界の値は、制約を満たす。日付は暦日の前後、日時は時点の前後で比べる
- `format` の `email` は、`EmailAddress.isWellFormed`（[common.md](common.md) の「EmailAddress」）が真であること
- `format` の `url` は、スキームが `http` または `https` の絶対 URL として解釈できること
- `format` の `pattern` は、値の全体が正規表現に一致すること
- 非推奨の指定は、検証の結果を変えない

`rule` が `null` の項目は、順 1 と 2 だけを確かめる。そのほかの検証は、使う側のドメインが加える。参照先・メディアの存在と公開状態、一意性のように、保存データを要する検証は、このサービスに置かない。

### ValueMigration

操作の並びを、一件の内容の値へ適用する。純粋な関数で、公開内容、下書き、ごみ箱のコンテンツの内容、履歴の版のそれぞれに、同じ規則で適用する。

```ts
const ValueMigration: {
  apply<L extends LeafValues>(
    input: Readonly<{ model: ModelKey; values: FieldValues<L> }>,
    base: ContentStructure,
    units: readonly Unit[],
    context: Readonly<{ removedBy: (id: ContentId) => OperationRef | null }>,
  ): ValueMigrationResult<L>;
};

type ValueMigrationResult<L extends LeafValues> =
  | Readonly<{ outcome: "removed"; by: OperationRef }>
  | Readonly<{
      outcome: "kept";
      model: ModelKey;
      values: FieldValues<L>;
      losses: readonly ValueLoss[];
      blocking: readonly BlockingValue[];
    }>;

type ValueLoss = Readonly<{ operation: OperationRef; loss: LossKind; path: ValuePath }>;
type BlockingValue = Readonly<{ type: "unconvertibleValue"; operation: OperationRef; path: ValuePath }>;
```

- `input.model` は、`base` でのモデルのキー。`base` と `units` は `ApplyPlan` の `baseStructure` と `units`
- `context.removedBy` は、参照先のコンテンツが、`ApplyPlan.modelFates` で `removed` とするモデルに属する場合はその `by`、ほかは `null` を返す。呼び出し側が与える
- `outcome: "removed"` は、内容のモデルが取り除かれたことを表す。そのコンテンツは、ごみ箱と履歴とともに完全削除される
- `outcome: "kept"` の `model` は適用後のモデルのキー、`values` は適用後の値
- `losses` は、データを失い得る操作が、この内容から取り除いた、または未設定にした値。`path` は適用前の値の位置。モデルの削除が取り除く参照の値では、すべての単位を終えた後の値の位置
- `blocking` は、変換できなかった値。`values` は、その値を未設定にした結果になる。`blocking` の使い方は [content.md](content.md) の「ContentMigration」が定める。`path` は、すべての単位を終えた後の値の位置で、適用後の項目のキーで表す。変換できなかった値の項目が、後続の操作でキーを変えた場合は、変えた後のキーで表す。後続の操作で取り除かれた場合（項目、その項目を持つ項目のまとまり・繰り返しの要素・ブロックの種類、モデルの削除）は、`blocking` に加えない。`losses` には加える
- 適用は制約を確かめない。変換・補完・複写した値が制約を満たさなくても、結果は変わらない
- 同じ引数からは同じ結果を返す。同じデータに同じ列を適用した結果は、ワークスペースによらない

適用は、単位ごとに次の順で行う。

1. 単位の操作を順に適用する。補完値は入れない
2. 単位の操作のうち補完値を持つものを順に見て、項目が単位の適用を終えた構造に残っていれば、未設定の値に補完値を入れる。項目の位置は、同じ単位の後続のキーの変更（モデル、項目、ブロックの種類）をたどって決める

すべての単位を終えた後、`context.removedBy` が `null` でないコンテンツを指すコンテンツ参照の値を取り除く。一件の参照は未設定にし、複数件の参照は、その参照だけを組から取り除いて、空になれば未設定にする。公開内容、下書き、ごみ箱、履歴の版のどれにも同じく適用する。取り除いた参照ごとに、`removedBy` が返した操作と `loss` の `contents` で `losses` に加える。`blocking` には加えない。取り除いた後の公開内容の検証は、[content.md](content.md) の「ContentMigration」が行う。

補完値を単位の最後に入れるので、同じ項目に値の複写と補完値を指定すると、値の複写を先に行い、残る未設定の値に補完値が入る。

`level` が繰り返しを通る操作は、繰り返しのすべての要素に適用する。ブロックの種類を通る操作は、その種類のすべてのブロックに適用する。操作ごとの値への適用は次のとおり。

| 操作 | 値への適用 |
| --- | --- |
| `addModel`、`addField`、`addOption`、`addBlockType`、`changeReferenceModels` | 値を変えない。加えた項目は未設定になる |
| `renameModel` | 内容のモデルのキーを変える。値を変えない |
| `removeModel` | そのモデルの内容を `removed` にする。ほかのモデルの内容が持つ、そのモデルのコンテンツへの参照は、上の「すべての単位を終えた後」の規則で取り除く |
| `renameField` | 値を、新しいキーへ移す |
| `changeRequired` | 値を変えない。補完値は手順 2 で入れる |
| `changeFieldKind` | `ValueConversion.convert` で変換する。変換できない値は未設定にし、`losses` と `blocking`（`unconvertibleValue`）に加える |
| `removeField` | 値を取り除く。値があった場合は `losses` に加える |
| `renameOption` | 単一選択・複数選択の値のうち、そのキーを新しいキーに置き換える |
| `removeOption`（選択肢へ） | そのキーを置き換え先のキーに置き換える。複数選択で置き換え先が既に選ばれていれば、重複を一つにする |
| `removeOption`（未設定へ） | 単一選択では値を未設定にする。複数選択ではそのキーを組から取り除き、空になれば未設定にする。`losses` に加える |
| `renameBlockType` | その種類のブロックの `type` を新しいキーに置き換える |
| `removeBlockType` | その種類のブロックを並びから取り除く。並びが空になれば未設定にする。`losses` に加える |
| `copyValue` | `to` が未設定で `from` に値がある場合に、`from` の値を `ValueConversion.convert` で `to` の値の種類へ変換して写す。変換できない値は写さない。`from` の値は変えない |

削除するモデルのコンテンツを指す本文のリンクは、リッチテキストの値の形を持つ側（[content.md](content.md) の「ContentMigration」）が扱う。

### FormChanges

```ts
type FormChanges = Readonly<{
  added: readonly FormKey[];
  removed: readonly FormKey[];
  changed: readonly Readonly<{
    form: FormKey;
    addedFields: readonly FieldKey[];
    removedFields: readonly FieldKey[];
  }>[];
}>;
```

`Schema.diffForms` は、フォームと項目をキーで比べる。キーを変えたフォームは `removed` と `added` に、キーを変えた項目は `removedFields` と `addedFields` に現れ、削除と追加として扱う。`removed` は、削除して受付を終了するフォームを表す。

## 型の対応

取得結果、コンテンツ管理で指定する値、フォームの送信内容、入力エラーの型のうち、スキーマの定義から決まる部分を定める。`packages/client` は、この規則でスキーマの定義から型を導き、ほかの部分を持つドメインの型と組み合わせる（[../usecases/app-api.md](../usecases/app-api.md) の「型の導出」）。型を導くために CMS へ接続しない。CMS の入力検証は、同じ定義を `ValueValidation` で使う。

### 値の種類ごとの型

| `kind` | 取得結果 | コンテンツ管理で指定する値 | フォームの送信内容 |
| --- | --- | --- | --- |
| `text` | 文字列 | 文字列 | 文字列 |
| `number` | 数 | 数 | 数 |
| `boolean` | 真偽値 | 真偽値 | 真偽値 |
| `date` | `YYYY-MM-DD` の文字列 | 同じ | 同じ |
| `datetime` | `YYYY-MM-DDTHH:mm:ss.sssZ` の文字列 | 時点を表す文字列 | 同じ |
| `select` | 選択肢のキーの直和 | 同じ | 同じ |
| `multiSelect` | 選択肢のキーの直和の配列 | 同じ | 同じ |
| `richText` | 本文の文書 | 同じ | ─ |
| `media` | メディアの取得結果 | メディアの ID と、使用箇所ごとの説明 | ─ |
| `reference` | 参照の取得結果 | コンテンツID | ─ |
| `referenceList` | 参照の取得結果の配列 | コンテンツID の配列 | ─ |
| `group` | 中の項目のキーから値への対応 | 同じ | ─ |
| `repeat` | 中の項目のキーから値への対応の配列 | 同じ | ─ |
| `blocks` | ブロックの種類のキー（`type`）と、その種類の項目の対応（`fields`）の組の、種類ごとの直和の配列 | 同じ | ─ |
| `contentReference` | ─ | ─ | コンテンツID |

- 型は、モデルとフォームのキー、項目のキー、選択肢のキー、ブロックの種類のキーを、文字列のリテラルとして持つ
- 単一モデルと複数件モデルは、同じ規則で一件の型を導く。取得の入口（指定なしの一件、ID または一意な値による一件、一覧）が異なる

### 必須と任意

| 対象 | 必須（`required` が真） | 任意 |
| --- | --- | --- |
| 取得結果の、テキスト・数値・真偽値・日付・日時・単一選択・リッチテキスト・メディア・参照（一件） | 値 | 値または `null` |
| 取得結果の、複数選択・参照（複数件）・繰り返し・ブロック | 配列 | 配列。未設定は空の配列 |
| 取得結果の、項目のまとまり | 対応。中の項目ごとに、この表を適用する | 同じ |
| コンテンツ管理で指定する値 | 省略できる。下書きは必須項目の欠落を入力エラーとして持てる | 省略できる |
| フォームの送信内容 | 省略できない | 省略できる |

- 取得結果は、定義したすべての項目のキーを持つ。未設定は `null` または空の配列で表す
- 取得が返す内容は、型を満たさない入力エラーを持たないので、必須の項目の型は `null` を含まない
- 任意の参照（一件）の型は `null` を含む。値が `null` になる場合と、複数件の参照から除く場合は、[content.md](content.md) の「参照先」に従う

### 参照の展開

- 展開しない参照の取得結果は、参照先のモデルのキー、コンテンツID、一意な値の組。モデルのキーの型は、`models` のキーの直和で、モデルのキーで判別する。一意な値は、項目のキーから値への対応。型は、そのモデルの直下のテキストの項目のキーを、省略できるキーとして持ち、値の型は文字列または `null`。一意性は型の形に含まれないので、型は `unique` の指定で変わらない。取得結果は、`unique` を持つ項目のキーだけを持つ
- 展開する参照の取得結果は、`models` の各モデルの一件の取得結果の、モデルのキーで判別する直和
- 参照の型は、参照の位置ごとに、展開する形か展開しない形のどちらか。どちらになるかは、取得の要求の展開の指定で決まる（[../usecases/app-api.md](../usecases/app-api.md) の「型の導出」の `Content<S, M, E>`）
- モデルごとの識別子は、参照先のモデルの型の形を含まない。展開する取得は、展開する参照の項目が許可するモデル（`models`）ごとの識別子も添えられる

### 画像の派生形式

メディアの取得結果のうち、スキーマから決まるのは、派生形式のキーごとの結果の型。

- 型は、スキーマが定義する `DerivativeKey` のすべてを、キーのリテラルとして持つ
- 型は、どのキーの結果も `null` を含む。`null` になるキーは、[media.md](media.md) の「MediaRecord」の `renditions` が返さないキー
- 画像形式の型は、派生形式が取り得る画像形式（`jpeg`、`webp`、`avif`、`png`）の直和。派生形式の `size` と `output` は型の形に含まれないので、型は、定義の `output.format` で変わらない

### 非推奨

- 非推奨のモデル、項目、選択肢、ブロックの種類は、型から除かない。値と取得結果は、非推奨の指定で変わらない。編集で非推奨の対象を新たに使う場合は、[content.md](content.md) の「DeprecatedUse」に従う
- 型は、非推奨の対象の宣言に、非推奨の印を付ける。モデルと項目は、その型とプロパティの宣言に付ける。選択肢とブロックの種類は、キーごとの宣言を持ち、その宣言に付ける
- 非推奨の指定だけの変更は、識別子を変えない

### 入力エラー

- コンテンツの入力エラーは、値の位置（`ValuePath`）ごとの、型を満たさない入力エラー（`TypeErrorKind`）と制約違反（`ConstraintKind`）。型は、位置を、モデルの項目のキーから導く
- フォームの入力エラーは、項目のキーごとの理由の列。キーの型は、そのフォームの項目のキーの直和

## ポート

リポジトリのポートは `packages/core/src/domain/schema/ports/` に置く。どのポートも、ドメイン型を受け渡す。

三つのポートは、ワークスペースの区画のリポジトリ。[common.md](common.md) の「TransactionalRepository」の、すべてのリポジトリに共通する規則に従う。ここには、上限の値と、このドメインのクエリだけを書く。

| 分類 | メソッド |
| --- | --- |
| 読み取り | `find`、`findById`、`findPage`、`findInProgress`、`countInProgress`、`findApplyResultsPage` |
| 書き込み | `insert`、`save`、`append`、`deleteAll` |

```ts
const SchemaLimits: {
  readonly inProgressLimit: 100;
  readonly preparationChunk: 100;
  readonly sampleLimit: 100;
};
```

- 一覧の 1 ページの件数の上限は、[common.md](common.md) の `Pagination.maxLimit`
- `inProgressLimit` は、`findInProgress` の `limit` の上限
- `preparationChunk` は、準備の一区切りで変換するコンテンツまたは履歴の版の件数の上限。準備領域への一回の書き込みが受ける件数の上限以下
- `sampleLimit` は、`Sampled` が内容（`items`）として持つ件数の上限。値は、APP-12 の「結果に示す先頭の対象の件数」。`Sampled` を返すほかのドメインのポートと振る舞いも、この定数を名前で指す
- 三つの値を持つのは、この定数だけ。ユースケースとテストケースは、定数の名前で指す

### WorkspaceSchemaRepository

`WorkspaceSchema` を保存し、取り出す。

```ts
interface WorkspaceSchemaRepository extends SingletonRepository<WorkspaceSchema> {
  deleteAll(limit: number): Promise<number>;
}
```

| メソッド | 契約 |
| --- | --- |
| `insert` / `find` / `save` | [common.md](common.md) の `SingletonRepository` の契約に従う。`applying` の `jobId` を、保存した値のまま返す。`jobId` が指すジョブの有無を確かめない |
| `deleteAll` | 結び付いた区画の `WorkspaceSchema` を消去する。返す件数は 0 または 1。楽観ロックを使わない |

- 適用中の反映が高々一つであることは、集約の型と、`save` の楽観ロックが保つ。並行した二つの `beginApply` の保存は、一方が `ConflictError`（`STALE_VERSION`）になる

### SchemaVersionRecordRepository

スキーマの版の記録を追記し、取り出す。

```ts
interface SchemaVersionRecordRepository {
  append(record: SchemaVersionRecord): Promise<void>;
  findPage(pagination: Pagination): Promise<PaginationResult<SchemaVersionRecord>>;
  deleteAll(limit: number): Promise<number>;
}
```

| メソッド | 契約 |
| --- | --- |
| `append` | 区画の中で `schemaVersion` が一意であることをポートが担保する。同じ版の記録が既にあれば `ConflictError`（`ALREADY_EXISTS`） |
| `findPage` | 記録を、`schemaVersion` の降順で返す。`count` は、区画の記録の数。範囲の先のページは、空の一覧と `count` を返す |
| `deleteAll` | 記録を `limit` 件まで消去する。消去する順は定めない |

記録が受ける書き込みは、追記と、利用終了の消去だけ。楽観ロックを使わない。

### SchemaApplyJobRepository

ジョブ（`SchemaApplyJob`）を保存し、取り出す。進行中のジョブ、反映の結果、事前確認の結果は、どれもこのポートで読む。事前確認の結果は、ジョブの ID で読む（`findById`）。

```ts
type InProgressCursor = Readonly<{ progressedAt: Date; id: SchemaApplyJobId }>;
type InProgressCounts = Readonly<Record<ApplyMode, number>>;

interface SchemaApplyJobRepository extends Omit<TransactionalRepository<SchemaApplyJob, SchemaApplyJobId>, "delete"> {
  findInProgress(
    filter: Readonly<{ progressedBefore: Date | null; after: InProgressCursor | null }>,
    limit: number,
  ): Promise<readonly Versioned<InProgressJob>[]>;
  countInProgress(): Promise<InProgressCounts>;
  findApplyResultsPage(pagination: Pagination): Promise<PaginationResult<FinishedJob<"commit">>>;
  deleteAll(limit: number): Promise<number>;
}
```

| メソッド | 契約 |
| --- | --- |
| `insert` / `findById` / `save` | [common.md](common.md) の `TransactionalRepository` の契約に従う。`insert` は、同じ ID のジョブが、種類と状態によらず既にあれば `ConflictError`（`ALREADY_EXISTS`）。計画、進行、影響、結果を、保存した値のまま返す |
| `findInProgress` | 進行中（`preparing` と `prepared`）のジョブを、種類によらず、`preparation.progressedAt` の昇順、同じ値ではジョブの ID の昇順で、`limit` 件まで返す。`progressedBefore` が `null` なら日時で絞り込まず、`null` でなければ、`preparation.progressedAt` がその日時以前（同じ日時を含む）のジョブに絞る。`after` が `null` でなければ、この順で、その位置より後ろのジョブに絞る。ワークスペースの状態を見ない。利用終了の途中のワークスペースのジョブも返す |
| `countInProgress` | 進行中のジョブの件数を、種類（`mode`）ごとに返す。全件を数える。ワークスペースの状態を見ない |
| `findApplyResultsPage` | 反映の結果（`mode` が `commit` の終端のジョブ）を、`finishedAt` の降順、同じ値ではジョブの ID の昇順で返す。成功も失敗も返す。事前確認のジョブを返さない。`count` は、区画の反映の結果の数。範囲の先のページは、空の一覧と `count` を返す |
| `deleteAll` | ジョブを、種類と状態によらず `limit` 件まで消去する。消去する順は定めない。楽観ロックを使わない |

## WriteScope とスキーマ

`WriteScope` の `WorkspaceCondition` の `stableSchema` と `schemaVersion`（[common.md](common.md) の「WriteScope」）が確かめる「スキーマの版」と「適用中」は、同じ区画の `WorkspaceSchema` の次の値を指す。

| 語 | 意味 |
| --- | --- |
| スキーマの版が同じ | `WorkspaceSchema.schemaVersion` が、条件の `schemaVersion` と等しい。楽観ロックの番号（`version`）は見ない |
| 適用中でない | `WorkspaceSchema.state.kind` が `idle` |

条件を変える確定に当たる操作は、次のとおり。

| 確定 | 当たる保存 |
| --- | --- |
| 適用中にする確定 | `beginApply` の結果の保存 |
| スキーマの版を進める確定 | `commit` の結果の保存、`recordApplied` の結果の保存 |

- 準備の進行（`SchemaApplyJob` の `advance` と、`prepared` を返す `complete` の結果の保存）は、`WorkspaceSchema` を保存しない。どちらの条件とも両立する
- `endApply` の結果の保存は、スキーマの版を変えずに `idle` に戻す。その確定の後は、同じ版を渡したどちらの条件も満たす
- `clearFailedApply` の結果の保存は、スキーマの版も `state` も変えない。どちらの条件とも両立する
- `stableSchema` を渡すユースケースは、読んだ `state` が `applying` なら、UnitOfWork を開かない。実行者が `system` でないユースケースは `UnavailableError`（`SCHEMA_APPLYING`）を返す。返す順は [index.md](index.md) の「利用できるワークスペース」に従う。`system` のユースケースの扱いは、[index.md](index.md) の「イベントの消費者と定期ジョブ」に従う

## 初期構成の宣言

初期構成の読み込みとワークスペースの作成で、Schema が行うのは、ワークスペースの区画に、ワークスペースに一つの集約を作ることだけ。モデル・フォームは、スキーマの反映で作る。手順を呼ぶ順は [index.md](index.md) の表が定める。

- 生成: `WorkspaceSchema.create(now)` の結果を `WorkspaceSchemaRepository.insert` で保存する。作った時点のスキーマは空で、モデルとフォームを持たない
- 前提（先に終えているべきドメイン）は Workspace。`Workspace` の保存と同じ UnitOfWork で保存する（[index.md](index.md) の「ワークスペースの作成と利用終了」）
- 初期構成の読み込みでも、運用コマンドによるワークスペースの作成でも同じ。設置の区画には、何も書かない
- 区画の段のやり直しは、区画に `Workspace` が既にあれば段ごと通過するので、`insert` を二度呼ばない

## 利用再開前の処理

バックアップから復旧した設置で、利用の再開の前に Schema が済ませる処理。Installation のユースケースが、名簿のワークスペースごとに、その区画について呼ぶ。手順を呼ぶ順は [index.md](index.md) の表が定める。

- 前提（先に終えているべきドメイン）は Workspace。対象のワークスペースを、直した名簿から求めるため
- 処理は、「SchemaApplyJob」の「中断」の、利用再開前の処理の行。手続きと、UnitOfWork に渡す `WriteScope` は、[../usecases/schema.md](../usecases/schema.md) の「進行中のジョブの中断」が定める
- 利用終了の途中のワークスペースのジョブも失敗させる。区画の消去の途中で取得したバックアップから復旧した区画でも、進行中のジョブが残っていれば、`WorkspaceSchema` がある（「利用終了の消去」の順）
- 区画にジョブが無いワークスペース（区画が無い場合を含む）では、何も書かない
- 失敗させたジョブは `findInProgress` に現れず、反映のジョブを失敗させたワークスペースの `WorkspaceSchema` は `idle` になり、そのジョブを未解消の失敗した反映（`failedApply`）として持つ

利用再開前の確認に、Schema が公開する項目。[common.md](common.md)「各ドメインが公開する項目」の `ResumeItemDefinition` の列と、件数を求める関数で公開する。

```ts
type SchemaResumeKind = "apply_to_interrupt" | "dry_run_to_interrupt";

const SchemaResumeItems: {
  definitions: readonly (ResumeItemDefinition & Readonly<{ kind: SchemaResumeKind }>)[];
  counts(inProgress: InProgressCounts): readonly ResumeItem[];
};
```

| `kind` | 内容 | 件数 | 単位 | `countsAsRemaining` | 確認に示す（`shownInReview`） |
| --- | --- | --- | --- | --- | --- |
| `apply_to_interrupt` | 失敗させる、復旧点で適用中だった反映 | `inProgress.commit` | ワークスペース | 真 | 真 |
| `dry_run_to_interrupt` | 失敗させる、復旧点で進行中だった事前確認 | `inProgress.dryRun` | ワークスペース | 真 | 真 |

- `definitions` は、表の行を表の順に持つ。`domain` は `schema`
- `counts` は、二つの項目を、表の順に、件数を添えて返す。`inProgress` は、`SchemaApplyJobRepository.countInProgress()` の結果。`PartitionReader.workspace(workspaceId)` のリポジトリで読む
- 処理を終えたワークスペースでは、どちらの件数も 0 になる。項目は、成功として示さない

## 利用終了の消去

ワークスペースの利用終了で、そのワークスペースの区画の Schema の集約を消去する。Installation のユースケースが、区画の消去で呼ぶ。手順を呼ぶ順は [index.md](index.md) の表が定める。

| 消去する集約 | 呼ぶメソッド |
| --- | --- |
| ジョブ（進行中のジョブ、反映の結果、事前確認の結果） | `SchemaApplyJobRepository.deleteAll` |
| スキーマの版の記録 | `SchemaVersionRecordRepository.deleteAll` |
| `WorkspaceSchema` | `WorkspaceSchemaRepository.deleteAll` |

- Schema の消去は、`ModelId` / `FormId` とジョブの ID を指す側（Token のトークンの範囲、Form の運用設定、Content の準備領域）の消去の後に行う。前提（先に終えているべきドメイン）は Token、Form、Content。順は [index.md](index.md) の表が持つ
- 消去は、表の順（ジョブ、スキーマの版の記録、`WorkspaceSchema`）に行う。一つの集約の消去を終えてから、次の集約へ進む。`WorkspaceSchema` を最後に消去するので、進行中のジョブが残る区画には、`WorkspaceSchema` がある
- 件数を区切る削除の進め方は、[common.md](common.md) の「TransactionalRepository」の「件数を区切る削除」に従う。`WriteScope` は、そのワークスペースの `workspace`、`condition` は `none`
- 進行中のジョブは、失敗させずに消去する。準備領域は、先に消去されている
- 区画が `closing` のワークスペースでは、イベントの消費者と定期ジョブが進める準備、確定、失敗は、確定しない（[index.md](index.md) の「イベントの消費者と定期ジョブ」）。消去の間、現行スキーマとコンテンツは反映で変わらない
- Schema が消すものは、ワークスペースの区画の三つの集約だけ
- 消去済みの区画への呼び出しは 0 を返す

利用終了の前の確認に、Schema が公開するもの。[common.md](common.md)「各ドメインが公開する項目」の `DomainKind`（`domain` は `schema`）の列で公開する。

```ts
type SchemaClosureLossKind = "schema" | "schema_version_record" | "schema_apply_result";

const SchemaClosure: { readonly lossKinds: readonly (DomainKind & Readonly<{ kind: SchemaClosureLossKind }>)[] };
```

| 利用終了で失われるデータの種類（`kind`） | 内容 |
| --- | --- |
| `schema` | 現行スキーマと、適用済みの移行定義の列 |
| `schema_version_record` | スキーマの版の記録 |
| `schema_apply_result` | 反映の結果と、事前確認の結果（終端のジョブ） |

- スキーマと移行定義の原本は、開発者のリポジトリにあり、CMS は消さない。Schema が CMS の通常データの外に持つ情報と、資格情報は、どちらも 0 件

## 状態確認の項目

設置の状態確認（Installation の `checkInstallationStatus`）に、Schema が示す項目。[common.md](common.md)「各ドメインが公開する項目」の `StatusItem`（`domain` は `schema`）で公開する。Installation のユースケースが、ワークスペースごとに、その区画から読んで求める。

```ts
type SchemaStatusKind = "apply_in_progress" | "unresolved_failed_apply";
type SchemaStatusItem = StatusItem & Readonly<{ kind: SchemaStatusKind }>;

const SchemaStatus: {
  of(ws: WorkspaceSchema | null): readonly SchemaStatusItem[];
};
```

| 項目の `kind` | 内容 | 件数（`count`） | 必要な対処が要るか（`actionRequired`） |
| --- | --- | --- | --- |
| `apply_in_progress` | 適用中の反映 | `ws.state` が `applying` なら 1、ほかは 0 | 常に偽。適用中であることを示すだけ |
| `unresolved_failed_apply` | 未解消の失敗した反映 | `ws.failedApply` が `null` でなければ 1、ほかは 0 | `count` が 1 なら真。開発者が、反映の結果を確かめて、原因を解消し、反映し直す（SMIG-14、SMIG-06）。変更を取りやめる場合は、現行スキーマと差分の無い反映を要求する |

- `of` は、二つの項目を、表の順に返す。件数と判定は、この関数だけが求める。`ws` が `null` の区画では、どちらの件数も 0
- `ws` は `WorkspaceSchemaRepository.find` の結果。`PartitionReader.workspace(workspaceId)` のリポジトリで読む
- 失敗した反映のジョブの件数を数えない。事前確認のジョブは、項目にならない

## エラーの理由コード

`SchemaErrorCode` は、Schema ドメインの理由コードのすべてを列挙する。共通の理由コード（`STALE_VERSION`、`ALREADY_EXISTS`、`SCHEMA_CHANGED`、`SCHEMA_APPLYING`、`WORKSPACE_NOT_ACTIVE`、`INVALID_QUERY`）は [common.md](common.md) が定める。拒否の理由からエラーへの対応は「拒否の理由とエラー」が定める。トランザクション境界は [../usecases/schema.md](../usecases/schema.md) が定める。

| 種類 | 理由 | 意味 |
| --- | --- | --- |
| `BusinessRuleError` | `SCHEMA_INVALID` | スキーマ自体の不整合。`SchemaIssue` の一覧を添える |
| `BusinessRuleError` | `MIGRATIONS_INVALID` | 移行定義の不整合。`MigrationIssue` の一覧を添える |
| `BusinessRuleError` | `MIGRATION_SEQUENCE_DIVERGED` | 手元の移行定義の列が、適用済みの列と食い違う。`SequenceDivergence` の一覧を添える |
| `BusinessRuleError` | `SCHEMA_DRIFT` | 直接反映による食い違いがある状態での、移行定義の適用。食い違う定義と、適用済みとして記録できる移行定義を添える |
| `BusinessRuleError` | `SCHEMA_STRUCTURE_MISMATCH` | 反映するスキーマのコンテンツの構造が、現行の構造へ未適用の移行定義を順に適用した結果と一致しない。食い違う定義を添える |
| `BusinessRuleError` | `SCHEMA_RECORD_MISMATCH` | 適用済みとして記録する範囲を適用した結果が、現行スキーマのコンテンツの構造と一致しない。食い違う定義と、記録できる移行定義を添える |
| `BusinessRuleError` | `SCHEMA_LOSS_NOT_ACKNOWLEDGED` | 確定の前に確認を求める対象（「反映の要求の検証」）を持つ直接反映を、確認なしに要求した。その対象を添える |
| `BusinessRuleError` | `MIGRATION_NOT_PENDING` | 適用済みとして記録する範囲の終端が、未適用の移行定義でない |
| `BusinessRuleError` | `SCHEMA_NEW_IDS_MISMATCH` | 確定に渡した新しい ID が、割り当ての条件に合わない。条件は「モデルとフォームの ID の割り当て」が定める。利用者の入力に由来しない |
| `BusinessRuleError` | `SCHEMA_JOB_MISMATCH` | 振る舞いに渡したジョブの ID が食い違う。条件は「WorkspaceSchema」の振る舞いと、`SchemaApplyJob.advance` が定める。利用者の入力に由来しない |
| `ConflictError` | `SCHEMA_VERSION_CHANGED` | 要求が前提とした現行スキーマの版が、現行の版と違う |
| `ConflictError` | `SCHEMA_JOB_ID_CONFLICT` | 同じワークスペースに、同じ `SchemaApplyJobId` で、内容の違うジョブが既にある |
| `ValidationError` | `INVALID_KEY` | キーの形式に合わない |
| `ValidationError` | `INVALID_MIGRATION_NAME` | 移行定義の名前の形式に合わない |
| `ValidationError` | `INVALID_DATE` | 暦日または時点として解釈できない |
| `ValidationError` | `INVALID_NUMBER` | 整数、範囲、有限の数の条件に合わない（文字数、件数、幅、高さ、品質、数値の値） |
| `ValidationError` | `INVALID_DEFINITION` | 型に無い値の種類、制約、操作を含む |

`ValidationError` は、スキーマと移行定義の中の位置ごとに理由を持つ。

## ユースケース（概要）

- 反映を要求する（`requestSchemaApply`）
- 直接反映を要求する（`requestDirectSchemaApply`）
- 適用の事前確認を要求する（`requestSchemaDryRun`）
- 反映・事前確認を準備する（`prepareSchemaApply`）
- 反映を確定する（`commitSchemaApply`）
- 反映・事前確認を失敗させる（`failSchemaApply`）
- 止まった反映・事前確認を検出して失敗させる（`failStalledSchemaApplies`）
- 移行定義を適用済みとして記録する（`recordMigrationsAsApplied`）
- 適用状況を確認する（`getMigrationStatus`）
- 現行スキーマを取得する（`getCurrentSchema`）
- 反映の適用中かを確かめる（`getSchemaApplyState`）
- モデルを一覧する（`listModels`）
- スキーマの版の記録を一覧する（`listSchemaVersionRecords`）
- 反映の結果を一覧する（`listSchemaApplyResults`）
- 受け付けた反映・事前確認の進行と結果を取得する（`getSchemaApplyJob`）
- スキーマを検証する（`validateSchema`）
- スキーマの識別子を求める（`computeSchemaFingerprints`）
- 移行定義を生成する（`generateMigration`）
- スキーマと移行定義の食い違いを検出する（`checkMigrations`）
- 変換規則を一覧する（`listConversionRules`）

詳細は [../usecases/schema.md](../usecases/schema.md) が定める。
