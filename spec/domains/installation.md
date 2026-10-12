# Installation

設置の初期化とデータの版、初期構成、バックアップ、復旧の進行を保つ。境界、依存方向、区画、ドメインをまたぐ規則、定期ジョブは [index.md](index.md) が、集約の ID、`Actor`、`Pagination`、`TransactionalRepository`、`WriteScope`、`UnitOfWorkProvider`、`PartitionReader`、`SecretCipher`、共通のエラーの理由は [common.md](common.md) が定める。

各ドメインが公開する手順と項目を Installation のユースケースが呼ぶ分担は [index.md](index.md) の「依存方向」に、呼ぶ順は同じ文書の「手順を呼ぶ順」と「区画をまたぐ手順」に従う。各ドメインが公開する項目は、[common.md](common.md) の「各ドメインが公開する項目」の型で受ける。設置の全体でホスティング運営者に示す語彙（運用コマンドの種類 `OperationsCommand`、復旧で元に戻る操作の種類 `RevertibleOperationKind`）は、Installation が持つ。ワークスペースは `WorkspaceId` で参照する。

コードは `packages/core/src/domain/installation/` に置く。Installation はドメインイベントを発行しない。集約とポートの区画は、[index.md](index.md) の「集約」と「集約とポートの置き場所」の表に従う。

## ユビキタス言語

| 英語名 | 日本語名 | 定義 |
| --- | --- | --- |
| Installation | 設置 | 一つの保存先を共有する CMS の全体。複数のワークスペースを持つ |
| Installation data | 設置のデータ | 設置の区画、名簿にあるワークスペースの区画、メディアの保管先にあるデータ。バックアップが写し、復旧が置き換える範囲 |
| Installation state | 設置の状態 | 設置が取る八つの状態の一つ。求め方は「設置の状態」の `of` が定める |
| Recovery | 復旧の記録 | 復旧の開始からの進行の記録。設置のデータの外に、設置に一つ持つ。消す主体は「Recovery」の不変条件が定める |
| Recovery lease | 復旧の占有 | `restoring` の復旧の記録が持つ、開始の時刻と期限（`leaseUntil`）。期限の内か外かで決まる設置の状態は、「設置の状態」の `of` が定める |
| Withdrawal | 開始の取り下げ | 置き換えを始めずに拒否された復旧の開始を、復旧の記録を開始の前の内容に戻して取り消すこと |
| Uninitialized | 初期化の前 | 保存先が空（`Installation` が無い）で、復旧の記録が `restoring` でも `failed` でもない状態。完了した復旧の記録（`restored`）が残る設置を含む |
| Loading | 読み込みの途中 | 初期構成の読み込みを終えていない状態 |
| Update required | 更新が必要 | 保存済みのデータの版が、実行中の CMS が扱うデータの版と違い、保存済みのデータの更新を始めていない状態 |
| Updating | 更新の途中 | 保存済みのデータの更新を始めて完了していない状態。この状態の設置を「更新中の設置」と呼ぶ |
| Update in progress | 更新の進行 | `Installation` の `update` の `updating`（「Installation」） |
| Operating | 稼働中 | 取得、フォーム受付、通知、管理画面の利用を行う状態 |
| Restoring | 復旧の進行中 | 設置の状態の `restoring`（「設置の状態」の `of`） |
| Restore failed | 復旧の失敗 | 設置の状態の `restore_failed`（「設置の状態」の `of`） |
| Recovery underway | 復旧中 | 復旧の進行中と復旧の失敗の総称。復旧を始めてから、完了するまで |
| Awaiting resume | 利用再開前 | 復旧を終えてから利用を再開するまでの状態。`Installation` があり、復旧の記録が `restored` |
| Data version | データの版 | 設置の保存先のデータの形を表す版。CMS の版ごとに、扱うデータの版が一つ決まる |
| Update | 設置の更新 | 保存済みのデータを、実行中の CMS が扱うデータの版へ進めること |
| Installation procedure | 導入手順 | 設置の導入の前にホスティング運営者が確認する文書。運用条件を示す |
| Operating conditions | 運用条件 | 対応する実行環境、必要な資源、利用上限、バックアップ間隔、復旧手順 |
| Closure procedure | 終了手順 | ワークスペースと設置全体の利用終了で、ホスティング運営者が行う作業を示す文書 |
| Initial configuration | 初期構成 | ホスティング運営者が宣言する、ワークスペース、所有者、トークン、運用設定 |
| Declaration | 宣言 | 初期構成の中の一つの指定 |
| Declaration digest | 宣言の照合値 | 初期構成の宣言から、各ドメインが照合値から除くと公開した値を除いたものから求めた照合値。読み込みの途中の設置が、続きの起動の宣言が同じであることを確かめるために持つ |
| Pending binding | 紐付け待ち | キーを指定して宣言した運用設定が、そのキーの対象の作成を待つこと。持ち方（設定の集約の中の状態か、別の集約か）は、運用設定を持つドメインが定める（[common.md](common.md) の「キーで宣言した設定の紐付け」）。紐付け待ちは、そのキーの対象が作成されて紐付くまで残る |
| Operations command | 運用コマンド | 設置の実行環境で実行する、ホスティング運営者のコマンド |
| Boot | 起動 | 運用コマンド `boot`。設置の設定と資源を確かめ、初期化の前の設置の初期化と、読み込みの途中の設置の読み込みの続きを行う。「起動しない」は、このコマンドが何も行わずに問題を示す結果を指す。利用者の要求とワーカーの処理を受け付けるかどうかは、起動の結果によらず、「設置の状態」の `blockingIssues` と `admit` が決める |
| Setup check | 動作確認 | 保存先、メディアの保管先、設置全体のメール送信が動くことを確かめる操作 |
| Status check | 状態確認 | 容量、各ドメインが公開する項目、隔離されたイベント、最後のバックアップを示す操作 |
| Backup | バックアップ | 設置のデータの、区画ごとの一つの時点の写しと、その記録 |
| Restore point | 復旧点 | 復旧に使うバックアップの取得日時。設置の区画の写しの時点 |
| Loss window | 失われる可能性のある期間 | 復旧点から復旧の時点までの期間 |
| Revertible operation | 元に戻る操作 | 復旧点以降に行い、復旧で取り消される操作 |
| Resume | 利用の再開 | 利用再開前の設置で、利用再開前の確認を示して運営者の確認を受け、利用再開前の処理を済ませ、復旧の記録を消すこと。再開の後の設置は、稼働中になる |
| Workspace closure | ワークスペースの利用終了 | 一つのワークスペースの配信・受付・通知を終え、そのデータを消去すること |
| Public URL | 設置の公開の URL | 利用者が管理画面を開く、設置の URL。設置の設定が宣言する |
| Secret key | 別に管理する秘密情報 | `SecretCipher` が秘密情報の保護に使う鍵。設置のデータの外に置く。`SecretCipher.currentKeyId` の値で識別する |

## 設置の状態

設置は、八つの状態のいずれか一つにある。状態は、`Installation`（設置の区画）、復旧の記録（設置のデータの外）、実行中の CMS が扱うデータの版（`InstallationEnvironment.dataVersion`）、現在時刻から求める一つの直和で、設置の停止では変わらない。設置の設定から受け付けないものを決める関数（`blockingIssues`）、設置の状態を求める関数（`of`）、状態ごとに受け付けるものを決める関数（`admit`）は、この節の三つだけ。

```ts
type AwaitingResume = Readonly<{
  installation: SettledInstallation;
  running: DataVersion;
  recovery: RestoredRecovery;
  stage: "version_mismatch" | "version_matched";
}>;

type InstallationState =
  | Readonly<{ kind: "restoring"; recovery: RestoringRecovery }>
  | Readonly<{ kind: "restore_failed"; recovery: FailedRecovery | RestoringRecovery }>
  | (Readonly<{ kind: "awaiting_resume" }> & AwaitingResume)
  | Readonly<{ kind: "uninitialized"; leftover: RestoredRecovery | null }>
  | Readonly<{ kind: "loading"; installation: LoadingInstallation }>
  | Readonly<{ kind: "update_required"; installation: SettledInstallation; running: DataVersion }>
  | Readonly<{
      kind: "updating";
      cms: "target" | "other";
      installation: UpdatingInstallation;
      running: DataVersion;
    }>
  | Readonly<{ kind: "operating"; installation: SettledInstallation }>;

type InstallationStateKey =
  | Exclude<InstallationState["kind"], "awaiting_resume" | "updating">
  | `awaiting_resume.${AwaitingResume["stage"]}`
  | "updating.target"
  | "updating.other";

type OperationsCommand =
  | "boot"
  | "setup_check"
  | "list_pending_bindings"
  | "create_workspace"
  | "reissue_initial_owner_invitation"
  | "update"
  | "status_check"
  | "take_backup"
  | "list_backups"
  | "restore"
  | "resume"
  | "preview_closure"
  | "close_workspace"
  | "operator_recovery";

type InstallationAccess =
  | Readonly<{ kind: "request" }>
  | Readonly<{ kind: "worker" }>
  | Readonly<{ kind: "operations"; command: OperationsCommand }>;

type StateCheckedAccess =
  | Readonly<{ kind: "request" }>
  | Readonly<{ kind: "worker" }>
  | Readonly<{ kind: "operations"; command: Exclude<OperationsCommand, "list_backups"> }>;

type SettingsRejection = Readonly<{
  admitted: false;
  state: null;
  cause: Readonly<{ kind: "settings"; issues: readonly SettingIssue[] }>;
}>;

type StateRejection =
  | Readonly<{
      admitted: false;
      state: "restoring";
      cause: Readonly<{ kind: "restore_occupied"; restorableAt: Date }>;
    }>
  | Readonly<{
      admitted: false;
      state: "awaiting_resume.version_mismatch";
      cause: Readonly<{ kind: "data_version_mismatch"; saved: DataVersion; running: DataVersion }>;
    }>
  | Readonly<{
      admitted: false;
      state: "updating.target" | "updating.other";
      cause: Readonly<{ kind: "update_in_progress"; to: DataVersion; running: DataVersion }>;
    }>
  | Readonly<{
      admitted: false;
      state: Exclude<
        InstallationStateKey,
        "restoring" | "awaiting_resume.version_mismatch" | "updating.target" | "updating.other"
      >;
      cause: Readonly<{ kind: "state" }>;
    }>;

type RejectionCause = SettingsRejection["cause"] | StateRejection["cause"];

type Admission = Readonly<{ admitted: true }> | SettingsRejection | StateRejection;

const InstallationState: {
  requiredSettings(access: InstallationAccess): readonly InstallationSetting[];
  blockingIssues(
    access: InstallationAccess,
    settingIssues: readonly SettingIssue[],
  ): readonly SettingIssue[];
  of(
    installation: Installation | null,
    recovery: Recovery | null,
    running: DataVersion,
    now: Date,
  ): InstallationState;
  keyOf(state: InstallationState): InstallationStateKey;
  admit(state: InstallationState, access: StateCheckedAccess): Readonly<{ admitted: true }> | StateRejection;
};
```

`OperationsCommand` は、設置のすべての運用コマンドを持つ。`operator_recovery`（管理者復旧）は Identity の運用コマンドで、Installation は名前だけを持つ。`InstallationStateKey` は、`admit` の表の行を指す名前で、`awaiting_resume` を `stage` ごとに、`updating` を `cms` ごとに分ける。

`requiredSettings` は、`access` ごとに、受け付けるのに要る設置の設定を返す。値は次の表だけが持ち、どの `access` も、どれかの行に当たる。

| `access` | 要る設定 |
| --- | --- |
| `request`、`worker`、`boot` | `data_store`、`media_store`、`secret_key`、`public_url`、`system_mail` |
| `take_backup`、`restore` | `data_store`、`media_store`、`secret_key` |
| `create_workspace`、`reissue_initial_owner_invitation`、`operator_recovery` | `data_store`、`public_url` |
| `setup_check`、`list_pending_bindings`、`update`、`status_check`、`list_backups`、`resume`、`preview_closure`、`close_workspace` | `data_store` |

- `blockingIssues` は、`settingIssues` のうち、`setting` が `requiredSettings(access)` に入るものを返す
- 返った列が空でない設置は、その `access` を受け付けない。結果は `SettingsRejection` で、設置の状態を持たず（`state` は `null`）、該当する設定と理由（`cause` の `issues`）だけを持つ
- 要る設定の理由: `data_store` は、保存先を読めないと状態を求められない。`media_store` は、取得と復旧が、バックアップが持つメディアの実体を写し、置き換える（「BackupStore」）。`secret_key` は、取得と復旧が `SecretCipher.currentKeyId` を読む（`BackupRecord.secretKeyId`、`RestoreEnvironment`）。`public_url` は、結果または本文に、`InstallationEnvironment.publicUrl` の値から作る URL を持つ（招待、回復用の URL）
- `list_backups` は、設置の状態によらず受け付ける
- 起動の設定の問題は、「BootReadiness」の `assess` が、資源の不足とともに求める

`of` は、上の行から順に当てはめ、最初に当たる状態を返す。

| 順 | 条件 | 状態 | `kind` |
| --- | --- | --- | --- |
| 1 | 復旧の記録が `restoring` で、`Recovery.isOccupied(recovery, now)` が真 | 復旧の進行中 | `restoring` |
| 2 | 復旧の記録が `failed`、または `restoring` で `Recovery.isOccupied(recovery, now)` が偽 | 復旧の失敗 | `restore_failed` |
| 3 | 復旧の記録が `restored` で、`Installation` が `SettledInstallation` | 利用再開前 | `awaiting_resume` |
| 4 | `Installation` が無い | 初期化の前 | `uninitialized` |
| 5 | `initialConfiguration` が `loading` | 読み込みの途中 | `loading` |
| 6 | `update` が `updating` | 更新の途中 | `updating` |
| 7 | `dataVersion` が `running` と違う | 更新が必要 | `update_required` |
| 8 | 上のどれにも当たらない | 稼働中 | `operating` |

- 利用再開前の `stage` は、`installation.dataVersion` が `running` と違えば `version_mismatch`、同じなら `version_matched`
- 更新の途中の `cms` は、`installation.update.to` が `running` と同じなら `target`、違えば `other`
- 初期化の前の `leftover` は、`Installation` が無い設置に残っている、完了した復旧の記録（`restored`）。無ければ `null`。運営者が、復旧の記録を消さずに保存先を空にした設置で残る。消す主体は「Recovery」の不変条件が定める
- 利用再開前の `installation` は `SettledInstallation`（初期構成の読み込みを終え、`update` が `idle` の `Installation`）。バックアップの中の `Installation` は、どれも `SettledInstallation` になる。`take_backup` を受け付ける状態（`admit` の表）と、`BackupStore.capture` の契約（更新中の設置の写しを取らない）による
- `loading` の `Installation` と完了した復旧の記録の組、更新中の `Installation` と完了した復旧の記録の組は生じない。`leftover` を消す時点は、`spec/usecases/installation.md` の「設置を起動する」のトランザクション境界が定める
- 行の順と求まる状態は、`spec/scenario/hosting.md` の「設置の状態」の表と同じ
- 設定は正しく、保存先の障害で `Installation` を読めない設置では、`of` に渡す値が得られず、状態を求められない。読み取りの `SystemError` が、そのまま返る。運営者は、障害を解消するか、空の保存先を用意する。空の保存先の設置は、初期化の前（`restoring` または `failed` の記録が残っていれば復旧中）として、復旧を受け付ける

`admit` は、`InstallationState.keyOf(state)` の行の `access` だけを受け付ける。セルの条件と、理由の優先順は無い。

| 状態（`InstallationStateKey`） | 受け付ける `access` |
| --- | --- |
| `restoring`（復旧の進行中） | `boot` |
| `restore_failed`（復旧の失敗） | `boot`、`restore` |
| `awaiting_resume.version_mismatch`（利用再開前。データの版が違う） | `boot`、`restore` |
| `awaiting_resume.version_matched`（利用再開前。データの版が合う） | `boot`、`restore`、`resume` |
| `uninitialized`（初期化の前） | `boot`、`restore` |
| `loading`（読み込みの途中） | `boot`、`restore` |
| `update_required`（更新が必要） | `restore`、`update`、`take_backup` |
| `updating.target`（更新の途中。実行中の CMS が更新先の版） | `restore`、`update` |
| `updating.other`（更新の途中。実行中の CMS がほかの版） | `restore` |
| `operating`（稼働中） | `request`、`worker`、`boot`、`setup_check`、`list_pending_bindings`、`create_workspace`、`reissue_initial_owner_invitation`、`update`、`status_check`、`take_backup`、`restore`、`preview_closure`、`close_workspace`、`operator_recovery` |

- 更新の途中の設置のバックアップの取得を受け付けないのは、写しを復旧に使えないため（データの形は「Installation」の不変条件）
- 更新が必要の設置の `update` を受け付けても、実行中の CMS が、保存済みのデータの版からの更新を持たない場合は、`UpdateReadiness.assess` が、更新を始める前に拒否する（「UpdateReadiness」の `unsupported`）

状態が受け付けない結果（`StateRejection`）は、状態の行の名前（`state`）と、理由（`cause`）を持つ。`cause` は、型のとおり、状態だけから決まる。

- `restore_occupied` は、復旧の進行中であることと、始め直せる時期（`restorableAt`。占有の期限 `recovery.leaseUntil`）を示す
- `data_version_mismatch` は、保存済みの版と実行中の版を示し、CMS を、復旧したデータの版を扱う CMS に合わせることを示す
- `update_in_progress` は、更新を始めて完了していないことと、更新先の版（`to`。`installation.update.to`）を示す。運営者の次の操作は「UpdateReadiness」の「更新中の設置」
- `state` は、設置の状態を示す

設置の設定が受け付けない結果（`SettingsRejection`）の `cause` は `settings` で、該当する設定と理由を示し、設置の状態を示さない。

入口とユースケースは、受け付けない結果を、次のとおりに返す。

| `access` | 受け付けない場合 |
| --- | --- |
| `request` | 処理を始めず、`UnavailableError`（理由 `INSTALLATION_NOT_OPERATING`） |
| `worker` | 処理を始めない。イベントは配送の対象に残る |
| `operations` | コマンドを実行せず、`BusinessRuleError`（`CommandNotAccepted`）。`StateRejection` では設置の状態と `cause` を、`SettingsRejection` では `cause`（該当する設定と理由）だけを添える。起動は、起動しない結果として返す |

- 設定は正しく、保存先の障害で状態を求められない設置（`of` の表の下の最後の箇条）では、`request` と `operations` は、状態を求める読み取りの `SystemError` を返す。`worker` は、イベントを配送の対象に残す
- 可否の規則は、`requiredSettings` の表と、`list_backups` の箇条と、`admit` の表だけ。関数を呼ぶ手順は、`spec/usecases/installation.md` の「設置の状態の確認」と「設置の状態を確かめる（`checkInstallationAvailability`）」が定める
- 起動は、初期化の前の設置の初期化と、読み込みの途中の設置の読み込みの続きを除いて、設置の状態を変えない
- 復旧は設置のデータの全体を置き換えるので、復旧の進行を設置のデータの中に持たない。完了した復旧の後の `Installation` は、バックアップの中の値になる

## エンティティ

### Installation

設置に一つ。初期化の前の設置の起動で作る。

フィールド。

```ts
type InitialConfiguration =
  | Readonly<{ status: "loading"; declarationDigest: DeclarationDigest }>
  | Readonly<{ status: "settled" }>;

type UpdateProgress =
  | Readonly<{ status: "idle" }>
  | Readonly<{ status: "updating"; to: DataVersion }>;

type InstallationBase = Readonly<{
  dataVersion: DataVersion;
  version: number;
  createdAt: Date;
  updatedAt: Date;
}>;

type LoadingInstallation = InstallationBase &
  Readonly<{
    initialConfiguration: Extract<InitialConfiguration, { status: "loading" }>;
    update: Extract<UpdateProgress, { status: "idle" }>;
  }>;
type SettledInstallation = InstallationBase &
  Readonly<{
    initialConfiguration: Exclude<InitialConfiguration, { status: "loading" }>;
    update: Extract<UpdateProgress, { status: "idle" }>;
  }>;
type UpdatingInstallation = InstallationBase &
  Readonly<{
    initialConfiguration: Exclude<InitialConfiguration, { status: "loading" }>;
    update: Extract<UpdateProgress, { status: "updating" }>;
  }>;

type InitializedInstallation = SettledInstallation | UpdatingInstallation;
type Installation = LoadingInstallation | InitializedInstallation;
```

| フィールド | 制約 |
| --- | --- |
| `initialConfiguration` | 必須。`loading` は初期構成の読み込みの途中、`settled` は、読み込みを終えたこと、または初期構成の宣言なしに始めたことを表す。`loading` だけが、読み込みを始めた宣言の照合値（`declarationDigest`）を持つ |
| `dataVersion` | 必須。保存済みのデータの版。作成時は、実行中の CMS が扱うデータの版。設置の更新の完了だけが進める |
| `update` | 必須。更新の進行。`idle` は、更新を始めていないか、完了したこと。`updating` は、保存済みのデータの更新を始めて完了していないことを表し、更新先の版（`to`）を持つ。`to` は `dataVersion` と違う。作成時は `idle`。読み込みの途中の `Installation` は `idle` だけを取る |
| `version` | 楽観ロックの番号。生成時は 0。状態を変える振る舞いごとに 1 進む |

振る舞い。

```ts
type BootDecision =
  | Readonly<{ action: "load_initial_configuration"; declarationDigest: DeclarationDigest }>
  | Readonly<{ action: "resume_initial_configuration"; installation: LoadingInstallation }>
  | Readonly<{ action: "start_without_configuration" }>
  | Readonly<{ action: "skip"; declarationIgnored: boolean }>
  | Readonly<{ action: "refuse"; reason: "initial_configuration_incomplete" }>;

type BootableState = Exclude<InstallationState, { kind: "update_required" | "updating" }>;

const Installation: {
  decideBoot(state: BootableState, declared: DeclarationDigest | null, running: DataVersion): BootDecision;
  initialize(
    initialConfiguration: InitialConfiguration,
    running: DataVersion,
    now: Date,
  ): LoadingInstallation | SettledInstallation;
  completeInitialConfiguration(installation: LoadingInstallation, now: Date): SettledInstallation;
  beginUpdate(installation: SettledInstallation, to: DataVersion, now: Date): UpdatingInstallation;
  completeUpdate(installation: UpdatingInstallation, now: Date): SettledInstallation;
};
```

| 振る舞い | 処理 |
| --- | --- |
| `decideBoot` | 「起動と初期構成」が定める |
| `initialize` | `Installation` を作る。`initialConfiguration` は渡された値（`loading` または `settled`）、`dataVersion` は `running`、`update` は `idle` |
| `completeInitialConfiguration` | `initialConfiguration` を `settled` にする |
| `beginUpdate` | `update` を、更新先の版が `to` の `updating` にする。`dataVersion` は変えない。`to` は、`UpdateReadiness.assess` が返した計画の `to`（`dataVersion` と違う版） |
| `completeUpdate` | `dataVersion` を `update.to` に、`update` を `idle` にする |

#### 起動と初期構成

起動できるかどうかは、設置の設定と資源から「BootReadiness」が決める。起動できない設置では、設置の状態を求めない。

起動を受け付けるかどうかは、ほかの運用コマンドと同じく `InstallationState.admit` が決める。受け付けない設置では、CMS は起動せず、`StateRejection` の設置の状態と `cause` を、保存済みの版（`state.installation.dataVersion`）・実行中の版（`state.running`）とともに示す。運用コマンド「設置を更新する」が、データを更新して、保存済みのデータの版を進める。

`decideBoot` は、起動を受け付ける状態（`BootableState`）の設置で、設置の状態、初期構成の宣言の照合値（宣言が無ければ `null`）、実行中の CMS が扱うデータの版から、行うことを決める。

| 設置の状態 | 結果 |
| --- | --- |
| 復旧の進行中、復旧の失敗、利用再開前 | `skip`。`declarationIgnored` は、`declared` が `null` でないこと。`Installation` が無い復旧中の設置でも、初期化しない |
| 初期化の前 | `declared` が `null` でなければ `load_initial_configuration`（`declarationDigest` は `declared`）、`null` なら `start_without_configuration` |
| 読み込みの途中で、`declared` が `declarationDigest` と違う（`null` を含む）、または `dataVersion` が `running` と違う | `refuse`（`initial_configuration_incomplete`） |
| 読み込みの途中 | `resume_initial_configuration` |
| 稼働中 | `skip`。`declarationIgnored` は、`declared` が `null` でないこと |

- 復旧した設置は、バックアップの中の `Installation`（`SettledInstallation`）を持つので、起動は `skip` になる
- `resume_initial_configuration` は、読み込みの途中の設置の続きを進める。確定済みの段を通過し、作成済みのワークスペース、所有者、トークン、運用設定を作り直さない
- 照合値から除く値だけを変えた宣言での起動は、同じ宣言として続きを進める。確定済みの段は変えない。確定した値は、読み込みを始めた宣言の値のまま残る。まだ確定していないワークスペースの区画の生成は、起動し直した宣言の値を使う
- `refuse`（`initial_configuration_incomplete`）では、CMS は起動せず、読み込みの途中の設置であることと、読み込みを始めた宣言と、読み込みを始めたデータの版を扱う CMS で起動し直すか、保存先を空にして始め直すことを示す。設置のデータは変わらない
- `start_without_configuration` では、`initialConfiguration` を `settled` にした `Installation` だけを作る。ワークスペースは無い
- 並行した起動は、`InstallationRepository.insert` の一意性で、一方だけが `Installation` を作る。負けた側を起動し直すと、`resume_initial_configuration` または `skip` になる。ワークスペース、所有者、トークンは重複しない
- `declarationIgnored` が真の場合、起動の結果に、初期構成を読み込まなかったことを示す。起動の結果は、起動の後の設置の状態（`InstallationState` の `kind`）も示す
- 復旧の進行中の設置の起動の結果は、加えて、始め直せる時期（占有の期限 `state.recovery.leaseUntil`）を示す。運営者は、復旧を実行せずに、起動の結果の設置の状態で、復旧が進行中か、始め直せるかを確かめる
- 読み込んだ後の運用設定、メンバー、トークンの正本は、各集約が持つ

不変条件。

- `Installation` は設置に一つ
- 初期構成の読み込みは、一つの宣言について一度だけ終える。`initialConfiguration` が `settled` の設置は、初期構成を読み込まない
- `update` が `updating` の間、保存済みのデータの形は、`dataVersion` の版の形とも `update.to` の版の形とも限らない。CMS の操作で `updating` から出る遷移は、`completeUpdate` と、バックアップからの復旧だけ

ライフサイクル。

```text
（記録なし）─ initialize（settled）→ Installation（初期化済み）
（記録なし）─ initialize（loading）→ Installation（読み込みの途中）─ completeInitialConfiguration → Installation（初期化済み）
Installation（初期化済み）─ beginUpdate → Installation（更新中）─ completeUpdate → Installation（初期化済み。データの版が進む）
```

`Installation` は、CMS の操作では削除しない。終端は設置全体の利用終了で、ホスティング運営者が「終了手順」に沿って行う消去。

### Recovery

復旧の記録。設置に一つ。復旧の開始（`begin` の保存）で作る。消す主体は、不変条件が挙げる四つ。設置のデータの外にあり、`RecoveryJournal` が保存する。バックアップとの関係は「BackupStore」が定める。

フィールド。

```ts
type RecoveryBase = Readonly<{
  backupId: BackupRecordId;
  restorePoint: Date;
  startedAt: Date;
  version: number;
}>;

type RestoringRecovery = RecoveryBase & Readonly<{ phase: "restoring"; leaseUntil: Date }>;
type FailedRecovery = RecoveryBase & Readonly<{ phase: "failed"; failedAt: Date }>;
type RestoredRecovery = RecoveryBase & Readonly<{ phase: "restored"; restoredAt: Date }>;
type Recovery = RestoringRecovery | FailedRecovery | RestoredRecovery;
```

| フィールド | 制約 |
| --- | --- |
| `backupId` | 必須。復旧に使うバックアップ |
| `restorePoint` | 必須。そのバックアップの取得日時（`BackupRecord.takenAt`） |
| `startedAt` | 必須。復旧を始めた日時。始め直した復旧では、始め直した日時 |
| `leaseUntil` | `restoring` だけが持つ。占有の期限。`startedAt` に `Recovery.leaseMs` を加えた日時 |
| `phase` | `restoring` は復旧の開始から結果まで、`failed` は、失敗を受け取った復旧、`restored` は置き換えの完了。`restoring` と `failed` の設置のデータは、途中まで置き換わっていることがある |
| `version` | 楽観ロックの番号。作成時は 0。状態を変える振る舞いごとに 1 進む |

振る舞い。

```ts
const Recovery: {
  leaseMs: 3_600_000;
  restoreTimeoutMs: 2_700_000;
  isOccupied(recovery: RestoringRecovery, now: Date): boolean;
  begin(
    current: Recovery | null,
    backup: BackupRecord,
    environment: RestoreEnvironment,
    now: Date,
  ): RestoringRecovery;
  complete(recovery: RestoringRecovery, now: Date): RestoredRecovery;
  fail(recovery: RestoringRecovery, now: Date): FailedRecovery;
  withdraw(begun: RestoringRecovery, previous: Recovery | null): Recovery | null;
  assertResumable(recovery: RestoredRecovery, remaining: ResumeCleanup): void;
};
```

| 振る舞い | 処理 |
| --- | --- |
| `isOccupied` | `now` が `leaseUntil` より前なら真。`leaseUntil` と同じ日時と、それ以降は偽 |
| `begin` | `BackupRecord.restorability(backup, environment)` が復旧できないと返せば、`BusinessRuleError`（`BackupNotRestorable`）を投げ、理由を添える。復旧できれば、`backupId` が `backup.id`、`restorePoint` が `backup.takenAt`、`startedAt` が `now`、`leaseUntil` が `now` に `leaseMs` を加えた日時の `restoring` の記録を返す。`version` は、`current` が `null` なら 0、それ以外は `current.version` を 1 進めた値。`InstallationState.admit` が `restore` を受け付けた設置で呼ぶ |
| `complete` | `phase` を `restored` にし、`restoredAt` を `now` にする。占有を持たなくなる。呼ぶ場面は、下の `BackupStore.restore` の結果ごとの表 |
| `fail` | `phase` を `failed` にし、`failedAt` を `now` にする。占有を解く。呼ぶ場面は、下の `BackupStore.restore` の結果ごとの表 |
| `withdraw` | 開始の取り下げ。`begun` は、保存した `begin` の結果。`previous` は、その `begin` に渡した `current`。`previous` が `null` なら `null`（記録なし）を返す。それ以外は、`previous` のフィールドを持ち、`version` が `begun.version` を 1 進めた値の記録を返す。呼ぶ場面は、下の `BackupStore.restore` の結果ごとの表 |
| `assertResumable` | `remaining`（`ResumeReview.remaining` の結果）のいずれかの項目の件数（`count`）が 1 以上なら `BusinessRuleError`（`ResumeCleanupIncomplete`）を投げ、`remaining` を添える。すべて 0 なら何も返さない。`restored` の記録だけを受ける |

`ResumeCleanup` は、利用の再開の前に済ませる処理の、処理の種類ごとの残っている件数の列。「ResumeReview」が定める。

占有。

- `leaseMs` の値は `spec/scenario/app-integration.md` の APP-12 の「復旧を進行中として扱う時間」、`restoreTimeoutMs` の値は同じ表の「復旧のデータの置き換えを打ち切る時間」。`restoreTimeoutMs` は、`BackupStore.restore` の打ち切りの時間で、`leaseMs` より短い
- `BackupStore.restore` の結果ごとの記録の遷移は、次の表のとおり。保存する手順と `expectedVersion` は、`spec/usecases/installation.md` の「バックアップから設置全体を復旧する」が定める

| `BackupStore.restore` の結果 | 記録 |
| --- | --- |
| 成功 | `complete` の結果。`restored` |
| 失敗（`SystemError`。「BackupStore」の `restore` の、置き換えを始めなかった結果のほかの失敗） | `fail` の結果。`failed`。占有を解く |
| 置き換えを始めなかった拒否（「BackupStore」の `restore` の、置き換えを始めなかった結果） | `withdraw` の結果。`null` なら記録なし |
| 結果を受け取れずに中断した（結果を受ける前、または結果の記録の保存が確定する前に止まった） | `restoring` のまま |

- 同じ設置の大きさと資源で始め直した復旧は、また打ち切られ得る。`InstallationEnvironment.restorableSizeLimit` が返す大きさまでのバックアップを、運用条件が示す資源の実行環境で、`restoreTimeoutMs` の内に置き換えられることは、ポートが担保せず、アダプターごとの検証で確かめる。運営者は、復旧前の確認が示すバックアップの大きさと、この上限を比べる（`spec/scenario/hosting.md` の HOST-08「時間内に終わらない復旧」）
- 始め直した復旧は、前の記録を引き継ぎ、新しい占有を持つ。同じバックアップでも、別のバックアップでも始め直せる
- `restored` の設置への復旧の開始は、新しい復旧として始まる

不変条件。

- 復旧の記録は設置に一つ
- 記録を消す主体は四つ。CMS では、`restored` の記録に、残件数がすべて 0 の `assertResumable` を通した利用の再開、`Installation` が無い設置に残った完了の記録（`restored`）を消す起動、記録が無い設置での開始の取り下げ（`withdraw` が `null` を返した場合）の三つ。CMS の外では、ホスティング運営者が、終了手順に沿って行う消去
- CMS の操作で `restoring` の記録から出る遷移は、`complete`、`fail`、`withdraw`、占有の期限を過ぎた後の `begin` だけ。`failed` の記録から出る遷移は `begin` だけ。ほかに、どの `phase` の記録も、ホスティング運営者が終了手順に沿って消去できる
- 設置のデータの置き換え（`BackupStore.restore`）は、保存が確定した `restoring` の記録を渡して行う。`restore` は、`RestoringRecovery` だけを受ける

ライフサイクル。

```text
（記録なし）─ begin → restoring ─ BackupStore.restore 成功 → complete → restored ─ 利用の再開（assertResumable）→（記録なし）
restoring ─ BackupStore.restore 失敗・打ち切り → fail → failed ─ begin → restoring
restoring ─ BackupStore.restore 拒否 → withdraw → 開始の前の記録（無ければ記録なし）
restored（Installation が無い）─ 起動 →（記録なし）
restoring（占有の期限を過ぎた）─ begin → restoring
restored ─ begin → restoring
```

終端は、不変条件が挙げる、記録を消す四つの主体による消去。設置全体の利用終了と、検証に使う設置の終了では、ホスティング運営者が、終了手順に沿って、データの保存先とともに復旧の記録を消去する。保存先を空にして始め直す設置でも、運営者は、復旧の記録を同じ手順で消去する。

### BackupRecord

バックアップの一件の記録。取得と整合性の確認を終えたバックアップについて作る。作成後は変更しない。写しとともに、設置のデータの外（バックアップの保管先）に `BackupStore` が保存し、復旧で変わらない。

フィールド。

```ts
type WorkspaceRef = Readonly<{ id: WorkspaceId; name: string }>;

type BackupIntegrity =
  | Readonly<{ status: "verified"; checkedAt: Date }>
  | Readonly<{ status: "failed"; checkedAt: Date; reason: string }>;

type BackupRecord = Readonly<{
  id: BackupRecordId;
  takenAt: Date;
  integrity: BackupIntegrity;
  retainUntil: Date;
  dataVersion: DataVersion;
  secretKeyId: string;
  workspaces: readonly WorkspaceRef[];
  sizeBytes: number;
}>;
```

| フィールド | 制約 |
| --- | --- |
| `id` | 必須。写しを取得した `BackupStore.capture` の `id` |
| `takenAt` | 必須。`BackupCapture.takenAt` の値（「BackupStore」）。復旧点になる |
| `integrity` | 必須。取得の直後に行った整合性の確認の結果。`checkedAt` は確認した日時。`reason` は運営者に示せる理由で、秘密情報を含まない |
| `retainUntil` | 必須。保持・削除期限。`takenAt` に `BackupPolicy.retentionMs` を加えた日時 |
| `dataVersion` | 必須。`BackupCapture.dataVersion` の値 |
| `secretKeyId` | 必須。空でない。取得の時点の `SecretCipher.currentKeyId` の値。バックアップの中の保護した秘密情報を戻すのに必要な鍵を指す。鍵そのものを含まない |
| `workspaces` | 必須。バックアップが含むワークスペース。設置の区画の写しの、ワークスペースの名簿にある項目の ID と名称。名簿の状態によらない。空の列を取れる |
| `sizeBytes` | 必須。0 以上の整数。バックアップの写しの大きさ（バイト）。`BackupCapture.sizeBytes` の値。復旧前の確認が、復旧できる大きさの上限とともに示す |

`WorkspaceRef` は、ワークスペースの ID と名称の組。Installation の型は、ワークスペースをこの形で示す。

振る舞い。

```ts
type RestoreEnvironment = Readonly<{ dataVersion: DataVersion; secretKeyId: string }>;

type Restorability =
  | Readonly<{ restorable: true }>
  | Readonly<{
      restorable: false;
      reason: "integrity_failed" | "incompatible_version" | "secret_unavailable";
    }>;

const BackupRecord: {
  record(params: {
    id: BackupRecordId;
    capture: BackupCapture;
    secretKeyId: string;
    integrity: BackupIntegrity;
  }): BackupRecord;
  restorability(record: BackupRecord, environment: RestoreEnvironment): Restorability;
  isPastRetention(record: BackupRecord, now: Date): boolean;
  previewRestore(
    record: BackupRecord,
    current: readonly WorkspaceRef[],
    restorableSizeLimit: number,
    now: Date,
  ): RestorePreview;
};
```

| 振る舞い | 処理 |
| --- | --- |
| `record` | 渡された `id`、`capture` の取得日時、データの版、ワークスペース、大きさと、`secretKeyId`、`integrity` を持つ記録を返す。`retainUntil` を求める |
| `restorability` | 次の順に確かめ、最初に当てはまった理由を返す。`integrity.status` が `failed` なら `integrity_failed`。`dataVersion` が `environment.dataVersion` と違えば `incompatible_version`。`secretKeyId` が `environment.secretKeyId` と違えば `secret_unavailable`。`environment` は、`InstallationEnvironment.dataVersion`（実行中の CMS が扱うデータの版）と `SecretCipher.currentKeyId` の値。判定に使うのは、実行中の CMS が扱う版とバックアップの版で、`Installation` の保存済みの版（`dataVersion`）は使わない。どれにも当たらなければ復旧できる |
| `isPastRetention` | `now` が `retainUntil` 以降なら真 |
| `previewRestore` | 「復旧前の確認」が定める |

不変条件。

- 記録は、取得済みのバックアップ（「BackupStore」）だけを表す
- 整合性の確認に失敗したバックアップは、復旧に使えない
- 保持・削除期限は、通常データの保持方針によらず、`BackupPolicy` だけから決まる
- バックアップは、取得時点の保存回答と、保護した秘密情報を含む。通常データで削除した回答も、バックアップを削除するまで残る
- 記録は、秘密情報の値を含まない

ライフサイクル。

- 生成: バックアップの取得と整合性の確認の後の、`BackupStore.complete`。取得は、運用コマンド「バックアップを取得する」だけが起こす（定期的な取得は [index.md](index.md) の「定期ジョブ」）
- 削除: ホスティング運営者が、運用手順に沿って、保管先でバックアップを削除する。対象は、保持・削除期限（`retainUntil`）を過ぎたバックアップと、不要になったバックアップ。バックアップを削除するのは、運営者のこの操作だけ。CMS は、期限を過ぎたバックアップを保ち続ける。`isPastRetention` が真のバックアップは、一覧で削除の対象として示す。保管先で削除したバックアップは、`BackupStore` の読み取りに現れない。設置全体の利用終了で残す・削除するバックアップも、ホスティング運営者が決めて保管先で削除する
- 完了しなかった取得の写しの扱いは、「BackupStore」の `capture` が定める。以後に取得を行わない設置では、写しは保管先に残り続ける。これが終端で、設置のデータと、取得済みのバックアップには影響しない。ホスティング運営者は、設置全体の利用終了の手順で、保管先ごと消去する

#### 復旧前の確認

```ts
type RevertibleOperationKind =
  | "individual_deletion"
  | "member_removal"
  | "permission_reduction"
  | "token_revocation";

type RestoredWorkspaces = Readonly<{
  restored: readonly (WorkspaceRef & Readonly<{ existsNow: boolean }>)[];
  lost: readonly WorkspaceRef[];
}>;

type RestorePreview = Readonly<{
  backupId: BackupRecordId;
  restorePoint: Date;
  lossWindow: Readonly<{ from: Date; to: Date }>;
  workspaces: RestoredWorkspaces;
  revertibleOperations: readonly RevertibleOperationKind[];
  requiredSecretKeyId: string;
  backupSizeBytes: number;
  restorableSizeLimitBytes: number;
}>;
```

- `RevertibleOperationKind` は、復旧点以降に行うと復旧で元に戻り、利用の再開が再適用しない操作の種類。値は順に、個別の削除、メンバーの解除、権限の縮小、トークンの失効。元に戻る操作の種類は、この型だけが持つ
- `restorePoint` は `takenAt`
- `lossWindow` は `takenAt` から `now` まで。すべてのワークスペースに同じ期間を適用する。この期間の受付、回答、編集、公開、スキーマの反映が失われ得る。利用再開前の確認の `lossWindow` の期間も、同じものが失われ得る。どの書き込みが戻るかは、「BackupStore」の区画ごとの写しが定める
- `previewRestore` は、データを変えない
- `current` は、現在のワークスペースの名簿のすべての項目の ID と名称。初期化の前の設置では空の列
- `restored` は、バックアップが含むワークスペース。すべて復旧点へ戻る。`existsNow` は、`current` に同じ ID があるかどうか
- `lost` は、`current` にあり、バックアップに無いワークスペース。復旧で無くなる
- `revertibleOperations` は、`RevertibleOperationKind` のすべての値
- `requiredSecretKeyId` は、秘密情報の復旧条件。`SecretCipher.currentKeyId` がこの値を返す鍵を、設置に用意していることを指す
- `backupSizeBytes` は `record.sizeBytes`。`restorableSizeLimitBytes` は、渡された `restorableSizeLimit`（`InstallationEnvironment.restorableSizeLimit` の値。運用条件が示す、復旧できる大きさの上限）。`previewRestore` は、二つを比べず、そのまま示す。上限を超えるバックアップの復旧を、CMS は拒否しない

## 値オブジェクト

ブランド付きの型は、`create` だけが生成する。利用者の入力に由来しない値の違反は `BusinessRuleError` を投げる。等価性は、断りが無い限り、すべてのフィールドの値の一致で決める。

| 名前 | 型 | バリデーションルール |
| --- | --- | --- |
| `DataVersion` | `string`（ブランド） | 空でない。設置のデータの版。`InstallationEnvironment.dataVersion` が返す。等価性は値の一致で、大小は比べない。ある版からの更新を、実行中の CMS が持つかどうかは、`InstallationEnvironment.supportsUpdateFrom` が返す |
| `DeclarationDigest` | `string`（ブランド） | 空でない。下の定義 |
| `BackupPolicy` | 定数 | 下の定義 |
| `BackupCapture` | オブジェクト | `BackupStore.capture` が返す。「BackupStore」の定義 |
| `ResourceReading` | 直和型 | 下の定義 |

### DeclarationDigest

初期構成の宣言から、照合値から除く値を除いたものから求めた照合値。読み込みの途中の `Installation` が持ち、続きの起動の宣言が、読み込みを始めた宣言と同じであることを `decideBoot` が確かめる。

```ts
type DeclarationValue =
  | string
  | number
  | boolean
  | null
  | readonly DeclarationValue[]
  | { readonly [key: string]: DeclarationValue };

const DeclarationDigest: {
  source(declaration: DeclarationValue): string;
  create(value: string): DeclarationDigest;
};
```

- `declaration` は、境界が「初期構成の読み込み」の表の形にデコードした宣言から、照合値から除く値を取り除いたもの。宣言しない位置は、表が定める値（`null` または空の列）にそろえてある
- 照合値から除く値は、宣言を受けるドメインが、自分のファイルの「初期構成の宣言」に「宣言の照合値から除く値」として公開するプロパティ。公開していないドメインの宣言は、すべての項目を照合値に含める。Installation は、除く値を列挙しない。ユースケースが、公開されたプロパティを取り除いてから `source` に渡す
- 除いた値だけが違う宣言は、同じ照合値になり、同じ宣言として扱う。除かない値を変えた宣言は、違う照合値になり、別の宣言として扱う
- `source` は、宣言を JSON の文字列にする。オブジェクトのプロパティを名前のコードポイントの昇順に並べ、列の順を保ち、空白を入れない。同じ宣言からは、プロパティを書いた順によらず、同じ文字列を返す。列の順が違う宣言と、値が一つでも違う宣言からは、違う文字列を返す
- ユースケースは、`source` の文字列を `SecretTokenIssuer.digest` に渡し、結果を `create` で包む。`create` は、空の文字列を `BusinessRuleError` にする
- `source` の文字列は、除かない秘密の値（トークンの値）を含み、宣言と同じく扱う（「初期構成の読み込み」）
- 等価性は値の一致

### BackupPolicy

```ts
const BackupPolicy: {
  intervalMs: 86_400_000;
  retentionMs: 2_592_000_000;
};
```

`intervalMs` の値は `spec/scenario/app-integration.md` の APP-12 の「バックアップの間隔」、`retentionMs` の値は同じ表の「バックアップの保持期間」。間隔は、ホスティング運営者が運用コマンドを起動する間隔の上限として運用条件に示し、状態確認の `overdue` の判定に使う。

### ResourceReading

```ts
type ResourceKind = "data_storage" | "media_storage" | "notification_queue";

type ResourceReading =
  | Readonly<{ kind: ResourceKind; status: "measured"; used: number; limit: number }>
  | Readonly<{ kind: ResourceKind; status: "unavailable"; reason: string }>;
```

- `data_storage` はデータの保存容量、`media_storage` はメディアの保管先の容量、`notification_queue` は通知用の空き（配送を待つ通知を保てる量）
- `used` と `limit` は 0 以上の整数。保存容量はバイト、通知用の空きは件数で表す。`limit` は運用条件の利用上限
- `unavailable` は、状態を取得できなかったことを表す。`reason` は秘密情報を含まない

### DomainKind、DomainCount、StatusItem

各ドメインが公開する項目の型（`DomainKind`、`DomainCount`、`StatusItem`、`ResumeItemDefinition`、`ResumeItem`）は、[common.md](common.md) の「各ドメインが公開する項目」が定める。

- Installation が自分で定める識別子（`InstallationState` の `kind`、`RejectionCause` の `kind`、`RequiredAction`、`RevertibleOperationKind`、`Restorability` の `reason`）も、[index.md](index.md) の「手順を呼ぶ順」の `kind` の表記で書く

### エラーコード

```ts
type InstallationErrorCode =
  // BusinessRuleError
  | "CommandNotAccepted"
  | "BackupNotRestorable"
  | "ResumeCleanupIncomplete"
  | "UpdateNotReady"
  | "UpdateNotSupported"
  // ConflictError の理由
  | "INSTALLATION_EXISTS"
  | "BACKUP_IN_PROGRESS"
  | "UPDATE_IN_PROGRESS"
  // SystemError の理由
  | "RESTORE_FAILED"
  | "RESTORE_TIMED_OUT"
  | "UPDATE_FAILED"
  | "BACKUP_CAPACITY_EXCEEDED"
  | "BACKUP_DESTINATION_UNREACHABLE";
```

| コード | 種類 | 意味 |
| --- | --- | --- |
| `CommandNotAccepted` | `BusinessRuleError` | 設置の状態、または設置の設定が受け付けない運用コマンド。添える値は「設置の状態」の、受け付けない結果の返し方の表が定める |
| `BackupNotRestorable` | `BusinessRuleError` | 復旧に使えないバックアップでの復旧の開始。`Restorability` の理由を添える |
| `ResumeCleanupIncomplete` | `BusinessRuleError` | 済ませる処理が残る設置の、利用の再開。処理の種類ごとの残っている件数（`ResumeCleanup`）を添える |
| `UpdateNotReady` | `BusinessRuleError` | 更新先の版が必要とする資源が不足する設置の更新。不足する資源を添える |
| `UpdateNotSupported` | `BusinessRuleError` | 実行中の CMS が、保存済みのデータの版からの更新を持たない設置の更新。保存済みの版と実行中の版を添える |
| `INSTALLATION_EXISTS` | `ConflictError` | 並行した起動で、`Installation` が既にある |
| `BACKUP_IN_PROGRESS` | `ConflictError` | 写しの取得と重なった `BackupStore.restore` の拒否。当たる重なりは「取得・置き換え・更新の排他」が定める |
| `UPDATE_IN_PROGRESS` | `ConflictError` | 進行中の更新と重なった操作の拒否。当たるのは、「InstallationRepository」の更新の開始、`InstallationEnvironment.updateData` が `in_progress` を返した更新、「取得・置き換え・更新の排他」の `capture` |
| `RESTORE_FAILED` | `SystemError` | `BackupStore.restore` の、ほかの失敗の理由（「BackupStore」の「失敗」） |
| `RESTORE_TIMED_OUT` | `SystemError` | 置き換えを打ち切った復旧（「BackupStore」の「打ち切り」） |
| `UPDATE_FAILED` | `SystemError` | データの更新の失敗。保存済みの版は進まず、設置は更新中のまま残る |
| `BACKUP_CAPACITY_EXCEEDED`、`BACKUP_DESTINATION_UNREACHABLE` | `SystemError` | バックアップの保管先の容量不足、保管先へ接続できない |

初期構成の宣言の問題の理由（`DeclarationIssue.code`）は、宣言を受けるドメインが定める。利用終了の対象の確認の名称の不一致、初期所有者の招待の発行の拒否は、手順を公開するドメインが定める。共通の理由（`STALE_VERSION`、`ALREADY_EXISTS`、`INVALID_QUERY`、`InvalidPagination`、`ACTOR_NOT_ALLOWED`、`INSTALLATION_NOT_OPERATING`、`RESTORE_IN_PROGRESS`、`SECRET_UNAVAILABLE`）は [common.md](common.md) が定める。`RESTORE_IN_PROGRESS` に当たる重なりは「取得・置き換え・更新の排他」が定める。

## ドメインサービス

### BootReadiness

責務: 設置の設定と資源から、起動できるかどうかと、利用者の要求を受け付けられる設定かどうかを判定する。

```ts
type InstallationSetting = "data_store" | "media_store" | "secret_key" | "public_url" | "system_mail";

type SettingInspection =
  | Readonly<{ setting: InstallationSetting; status: "valid" }>
  | Readonly<{ setting: InstallationSetting; status: "not_declared" }>
  | Readonly<{ setting: InstallationSetting; status: "invalid"; reason: string }>;

type SettingIssue =
  | Readonly<{ kind: "setting_missing"; setting: InstallationSetting }>
  | Readonly<{ kind: "setting_invalid"; setting: InstallationSetting; reason: string }>;

type BootIssue = SettingIssue | Readonly<{ kind: "resource_short"; reading: ResourceReading }>;

type BootReadinessResult =
  | Readonly<{ ready: true }>
  | Readonly<{ ready: false; issues: readonly BootIssue[] }>;

const BootReadiness: {
  settingIssues(settings: readonly SettingInspection[]): readonly SettingIssue[];
  assess(input: {
    settings: readonly SettingInspection[];
    readings: readonly ResourceReading[];
  }): BootReadinessResult;
};
```

- `settings` は、`InstallationEnvironment.inspectSettings` の結果
- `invalid` の設定は、どれも `setting_invalid`。`reason` は `inspectSettings` が返した理由
- `not_declared` の `data_store`、`media_store`、`secret_key`、`public_url` は `setting_missing`。`not_declared` の `system_mail` は、外部メールを使えない設置として受け付ける
- `readings` のうち、`StatusAssessment.assessResource` が `short` と判定した資源は `resource_short`。`unknown` は起動を妨げない
- `settingIssues` は、上の規則のうち、設定の問題（`setting_missing` と `setting_invalid`）だけを、すべて返す
- `assess` は、`blockingIssues`（`access` は `boot`）の結果に `resource_short` を加える。起動に要る設定は、`InstallationState.requiredSettings` の表が定める
- 見つけた問題をすべて返す。一つでもあれば `ready: false`。起動しない結果は、`issues` の設定・資源と理由を示す。何も保存しない

依存するポート: 無し。

### UpdateReadiness

責務: 設置の更新の前に、更新先の版が必要とする資源がそろっているかを判定し、更新の計画を返す。

```ts
type UpdatePlan = Readonly<{ from: DataVersion; to: DataVersion }>;

type UpdatePrerequisite = Readonly<{ kind: "resource_short"; reading: ResourceReading }>;

type UpdateDecision =
  | Readonly<{ outcome: "ready"; plan: UpdatePlan }>
  | Readonly<{ outcome: "up_to_date"; dataVersion: DataVersion }>
  | Readonly<{ outcome: "unsupported"; saved: DataVersion; running: DataVersion }>
  | Readonly<{ outcome: "not_ready"; missing: readonly UpdatePrerequisite[] }>;

const UpdateReadiness: {
  assess(input: {
    installation: InitializedInstallation;
    running: DataVersion;
    updateSupported: boolean;
    readings: readonly ResourceReading[];
  }): UpdateDecision;
};
```

`assess` は、`InstallationState.admit` が `update` を受け付けた設置で呼ぶ。上の行から順に当てはめ、最初に当たる結果を返す。

| `installation` | 結果 |
| --- | --- |
| `update` が `idle` で、`dataVersion` が `running` と同じ | `up_to_date`。データを変えない |
| `update` が `idle` で、`dataVersion` が `running` と違い、`updateSupported` が偽 | `unsupported`。`saved` は `installation.dataVersion`。実行中の CMS は、保存済みのデータの版からの更新を持たない。更新の完了の後に、CMS を更新前の版の CMS に戻した設置が当たる |
| `update` が `idle` で、`dataVersion` が `running` と違う | 不足する資源をすべて集める。一つでもあれば `not_ready`。無ければ `ready`。`plan.from` は `installation.dataVersion`、`plan.to` は `running` |
| `update` が `updating`（更新中の設置） | 不足する資源をすべて集める。一つでもあれば `not_ready`。無ければ `ready`。`plan.from` は `installation.dataVersion`、`plan.to` は `installation.update.to` |

- 資源: `readings` のうち、`StatusAssessment.assessResource` が `short` と判定した資源ごとに、`resource_short`。`unknown` は更新を妨げない
- `updateSupported` は、`InstallationEnvironment.supportsUpdateFrom(installation.dataVersion)` の結果。更新中の設置では使わない。更新中の設置で `update` を受け付ける CMS（`admit` の表）は、更新を始めた版からの更新を持つ
- `readings` は、実行中の CMS（更新先の版）の `InstallationEnvironment.readResource` の、`ResourceKind` の三つの結果
- 更新の前のバックアップの取得は、更新手順が示す運営者の作業で、`assess` は確かめない
- `unsupported` と `not_ready` の設置は、更新を始めない

更新の進み方。

- 呼ぶ順と UnitOfWork は、`spec/usecases/installation.md` の「設置を更新する」のトランザクション境界が定める
- 更新は、設置を停止して行う。復旧の記録を変えない
- 並行した更新は、後の実行を始めない。`update` が `idle` の設置で重なった二つの実行は、`beginUpdate` の保存で、一方だけが進む。`updateData` どうしの重なりは、「BackupStore」の「取得・置き換え・更新の排他」が定める

更新中の設置。`beginUpdate` の保存の後、`completeUpdate` の保存の前に、失敗または中断した更新が残す。

- 運営者の次の操作は二つ。CMS は、どちらも自動で行わない

| 操作 | 内容 |
| --- | --- |
| 更新をやり直す | 原因を解消し、更新先の版（`update.to`）を扱う CMS で「設置を更新する」を実行し直す。`updateData` は、前の更新がどこで止まっていても、保存済みのデータを更新先の版の形にする |
| 更新をやめる | CMS を更新前の版の CMS に戻し、更新の前に取得したバックアップ（`dataVersion` の版）から復旧する |

- 更新を完了した設置を更新前の版へ戻す場合も、「更新をやめる」と同じ操作で戻す
- 更新を始める前（`unsupported` と `not_ready` で始まらなかった更新を含む）の設置は、CMS を、保存済みのデータの版を扱う CMS に入れ替えて起動すると、稼働中になる

更新のライフサイクル。

```text
更新が必要 ─ 設置を更新する ─ assess（ready）─ beginUpdate → 更新の途中 ─ updateData 成功 ─ completeUpdate → 稼働中
                           │                                    └ 失敗・中断 → 更新の途中 ─ 更新先の版で設置を更新する → …
                           │                                                                      └ 更新前の版の CMS に戻して復旧 → 復旧中
                           ├ assess（not_ready）→ バックアップの取得、資源の用意 → 設置を更新する
                           └ assess（unsupported）→ CMS を、保存済みのデータの版を扱う CMS に入れ替える → 稼働中
```

依存するポート: 無し。

### WorkspaceClosure

責務: ワークスペースの利用終了の前に示す内容を組み立てる。

```ts
type ClosurePreview = Readonly<{
  workspace: WorkspaceRef & Readonly<{ status: string }>;
  items: readonly DomainCount[];
  lostData: readonly DomainKind[];
  erasedCredentials: readonly DomainKind[];
  remainsOutside: readonly DomainKind[];
  backupsRetainedUntil: Date | null;
}>;

const WorkspaceClosure: {
  preview(input: {
    workspace: ClosurePreview["workspace"];
    items: readonly DomainCount[];
    lostData: readonly DomainKind[];
    erasedCredentials: readonly DomainKind[];
    remainsOutside: readonly DomainKind[];
    backups: readonly BackupRecord[];
  }): ClosurePreview;
};
```

- `preview` は、終了前に示す内容を返す。データを変えない。渡された四つの列を、そのまま持つ
- `workspace` は、ユースケースが読んで渡す、ワークスペースの ID、名称、名簿の状態（`status`。値は Workspace が定める）。終了前の確認は、名簿に項目があるワークスペースのすべて（作成の途中、利用終了の途中を含む）に行え、その時点で残っている件数を示す
- `items` は、各ドメインが「利用終了の消去」に、終了前の確認に示す件数として公開した項目。違う種類の件数を足し合わせない。失われるデータの種類のうち、件数も示すと公開されたもの（メディアの件数など）は、件数を `items` に、種類の名前を `lostData` に持つ
- `lostData` は利用終了で失われるデータの種類、`erasedCredentials` は消去する資格情報、`remainsOutside` は CMS の通常データの外に残る情報。各ドメインが「利用終了の消去」に公開した `kind` の列を、ユースケースが、利用終了の区画の消去の順（[index.md](index.md) の「手順を呼ぶ順」）に、その順に入らないドメインの分を最後に続けて、つないで渡す
- `backupsRetainedUntil` は、`backups` のうち、そのワークスペースを含むバックアップの `retainUntil` の最も遅い日時。該当が無ければ `null`。この日時は、そのバックアップの削除の期限（「BackupRecord」のライフサイクル）
- 運用コマンドの入口は、確認の列を、そのまま示す。入口は、項目を足さない

利用終了の手順と順。

- 区画の順は [index.md](index.md) の「区画をまたぐ手順」が、各ドメインの消去を呼ぶ順は同じ文書の「手順を呼ぶ順」が、各段の中身は各ドメインの「利用終了の消去」が定める。Installation は、手順の中身と順を定めない
- 設置全体の終了前の確認は、名簿のすべてのワークスペースの `preview` を並べる。設置全体の利用終了では、CMS は終了前の確認を示すだけで、消去を行わない。保存データ、メディア、復旧の記録、資格情報の消去と、バックアップの整理（残すバックアップと削除するバックアップを決め、保管先で削除する）は、ホスティング運営者が終了手順で、CMS の外で行う

依存するポート: 無し。

### ResumeReview

責務: 各ドメインが公開した利用再開前の項目を、確認の列と残件数に振り分け、利用再開前の確認に示す内容を組み立てる。

```ts
type CountedResumeItem =
  | (ResumeItem & Readonly<{ unit: "installation"; workspaces: readonly WorkspaceRef[] }>)
  | (ResumeItem & Readonly<{ unit: "workspace"; workspace: WorkspaceRef }>);

type ResumeCleanup = readonly DomainCount[];

type ResumeReviewReport = Readonly<{
  restorePoint: Date;
  lossWindow: Readonly<{ from: Date; to: Date }>;
  revertibleOperations: readonly RevertibleOperationKind[];
  remaining: ResumeCleanup;
  installationItems: readonly (DomainCount & Readonly<{ workspaces: readonly WorkspaceRef[] }>)[];
  workspaces: readonly (WorkspaceRef & Readonly<{ items: readonly DomainCount[] }>)[];
}>;

const ResumeReview: {
  remaining(items: readonly CountedResumeItem[]): ResumeCleanup;
  compose(recovery: RestoredRecovery, items: readonly CountedResumeItem[]): ResumeReviewReport;
};
```

#### 利用再開前の処理

処理を持つドメインが自分のファイルの「利用再開前の処理」の節に公開するものは、[index.md](index.md) の「依存方向」が定める。Installation は、処理の中身を定めない。ドメインを呼ぶ順は、[index.md](index.md) の「手順を呼ぶ順」が定める。

- 処理は二種類ある。設置の区画の記録を、ワークスペースの区画の内容に合わせて直す処理と、ワークスペースの区画の中で、復旧点で終えていなかった処理を片付ける処理。区画の間の食い違いが生じる理由は、[index.md](index.md) の「手順ごとの規則」の、バックアップと復旧が定める
- ワークスペースごとの処理の対象は、各節が、名簿の状態で定める。呼び方は、`spec/usecases/installation.md` の「利用を再開する」が定める
- 失敗した対象の扱いと、残件数の意味は、[index.md](index.md) の「手順ごとの規則」の、バックアップと復旧が定める。一回の実行が読む対象の範囲は、各節が定める

`CountedResumeItem` は、各ドメインが件数を求める関数で返した `ResumeItem` に、ユースケースが区画を添えた一件。`unit` が `workspace` の項目は、件数を求めたワークスペース（`workspace`）を、`installation` の項目は、公開した節が定める、該当するワークスペースの列（`workspaces`）を持つ。

`remaining` の内容。

- `countsAsRemaining` が真の項目を、`domain` と `kind` の組ごとに一件にまとめた列。`count` は、その組の項目の件数の和で、その処理が済ませていない対象の数。違う `kind` の件数を足し合わせない。列の順は、`items` に組が初めて現れた順

`compose` の内容。

- `restorePoint` は `recovery.restorePoint`
- `lossWindow` は、`restorePoint` から `recovery.restoredAt` まで
- `revertibleOperations` は、`RevertibleOperationKind` のすべての値
- `remaining` は、`ResumeReview.remaining(items)` の結果
- `installationItems` は、`shownInReview` が真で、`unit` が `installation` の項目。設置の全体で一つの件数と、該当するワークスペースの ID と名称の列を持つ
- `workspaces` は、`shownInReview` が真で、`unit` が `workspace` の項目を、`workspace` ごとにまとめた列。ワークスペースの順は、`items` に初めて現れた順
- 確認に示す項目は、`unit` で、二つの列の一方だけに入る。`shownInReview` が偽の項目は、残件数にだけ入る。処理を持たないドメインも、`countsAsRemaining` が偽の項目だけを公開できる

依存するポート: 無し。

### StatusAssessment

責務: 設置の資源と運用の状況から、不足と必要な対処を判定する。状態確認と、導入時の動作確認が使う。

```ts
type ResourceAssessment = Readonly<{
  reading: ResourceReading;
  level: "sufficient" | "short" | "unknown";
}>;

type RequiredAction =
  | "expand_capacity"
  | "rerun_check"
  | "take_backup"
  | "redeliver_quarantined_events";

type WorkspaceStatus = WorkspaceRef & Readonly<{ items: readonly StatusItem[] }>;
type StatusItemWorkspace = WorkspaceRef & Readonly<{ detail: Readonly<Record<string, string>> | null }>;
type InstallationStatusItem = StatusItem & Readonly<{ workspaces: readonly StatusItemWorkspace[] }>;

type StatusReport = Readonly<{
  resources: readonly (ResourceAssessment & Readonly<{ actions: readonly RequiredAction[] }>)[];
  installationItems: readonly InstallationStatusItem[];
  workspaces: readonly WorkspaceStatus[];
  quarantinedEvents: Readonly<{ count: number; actions: readonly RequiredAction[] }>;
  lastBackup: LastBackupStatus;
}>;

type LastBackupStatus =
  | Readonly<{ status: "none"; actions: readonly RequiredAction[] }>
  | Readonly<{
      status: "taken";
      takenAt: Date;
      integrity: BackupIntegrity["status"];
      overdue: boolean;
      actions: readonly RequiredAction[];
    }>
  | Readonly<{ status: "unavailable"; reason: string; actions: readonly RequiredAction[] }>;

type ProbeResult =
  | Readonly<{ status: "passed" }>
  | Readonly<{ status: "failed"; reason: string }>;

type MailProbeResult = ProbeResult | Readonly<{ status: "not_configured" }>;

type SetupCheckReport = Readonly<{
  dataStore: ProbeResult;
  mediaStore: ProbeResult;
  systemMail: MailProbeResult;
  resources: readonly ResourceAssessment[];
}>;

const StatusAssessment: {
  assessResource(reading: ResourceReading): ResourceAssessment;
  status(
    input: {
      readings: readonly ResourceReading[];
      installationItems: readonly InstallationStatusItem[];
      workspaces: readonly WorkspaceStatus[];
      quarantinedEvents: number;
      latestBackup: LatestBackupReading;
    },
    now: Date,
  ): StatusReport;
  setupCheck(input: {
    dataStore: ProbeResult;
    mediaStore: ProbeResult;
    systemMail: MailProbeResult;
    readings: readonly ResourceReading[];
  }): SetupCheckReport;
};
```

`RequiredAction` は、Installation が自分で判定する項目（資源、隔離されたイベント、バックアップ）の必要な対処。各ドメインの項目の必要な対処は、`StatusItem.actionRequired` で示す。報告が持つのは、対処が要るかどうか（`actionRequired`）と、ワークスペースごとに添える値（`detail`）まで。対処の内容は、運用コマンドの入口が、項目の `domain`、`kind`、`detail` から、公開したドメインの「状態確認の項目」の表で引いて示す。

`assessResource` の規則。

- `unavailable` は `unknown`。不足なしと判定しない
- `measured` で、`used` が `limit` 以上（空きが無い）なら `short`
- それ以外は `sufficient`

`status` の規則。

| 項目 | 必要な対処 |
| --- | --- |
| 資源が `short` | `expand_capacity` |
| 資源が `unknown`、または `latestBackup` が `unavailable` | `rerun_check` |
| `quarantinedEvents` が 1 以上 | `redeliver_quarantined_events`。ホスティング運営者が、実行基盤の運用手順で配送し直す |
| `latestBackup` の `record` が `null`、`now` が `takenAt` に `BackupPolicy.intervalMs` を加えた日時以降（`overdue`）、または整合性の確認が `failed` | `take_backup` |

- `lastBackup` は、`latestBackup` が `unavailable` なら `unavailable` で、`reason` はその理由。`record` が `null` なら `none`。それ以外は `taken` で、`takenAt` と `integrity` は `record` の値、`overdue` は上の表の判定
- `latestBackup` は、`BackupStore.findLatest` の結果
- `readings` は、`ResourceKind` の三つすべてを含む
- `workspaces` は、ワークスペースごとに、各ドメインが「状態確認の項目」に公開する項目を、ユースケースがその区画から読んだ値を、各ドメインが公開する関数に渡して作った `StatusItem` の列。報告は、渡された項目をそのまま持つ。区画を読むワークスペースは、[index.md](index.md) の「区画をまたぐ手順」の状態確認が定める
- `installationItems` は、各ドメインが「状態確認の項目」に公開する項目のうち、設置の全体について求めるもの。項目ごとに、該当するワークスペースの ID と名称を持つ。どの項目を、名簿と区画のどちらから求めるかは、公開したドメインの節が定める。利用終了が完了したワークスペースは、名簿に無いので現れない
- `StatusItemWorkspace.detail` は、項目を公開したドメインが、該当するワークスペースごとに添えると公開した値の組。キーと値は、公開したドメインの節が定める。添える値を公開しない項目では `null`。値の意味と、値ごとの必要な対処は、公開したドメインの節が定める
- `quarantinedEvents` は、隔離されたイベントの件数。ユースケースが、[common.md](common.md) の「QuarantinedEventQuery」のとおりに求めて渡す。報告は、ワークスペースで分けない
- 報告は、回答の入力内容、連絡先、秘密情報を含まない。件数と日時、ワークスペースの ID と名称、各ドメインがワークスペースごとに添えると公開した値だけを持つ

`setupCheck` の規則。

- `dataStore` は保存先への書き込みと読み出し、`mediaStore` はメディアの保存と取り出し、`systemMail` は指定した宛先への設置全体のメール送信の結果
- `systemMail` の `passed` は、メール送信基盤が受理したことを表す。受信箱への到着は表さない。`not_configured` は、設置全体のメール送信を宣言していない設置を表し、失敗でない
- `resources` は、`readings` の資源ごとの `assessResource` の結果
- ユースケースは、設置全体のメール送信の結果を `MailProbeResult` に写す。受理は `passed`、未設定は `not_configured`、失敗は `failed`。確認メールは秘密情報を含まない

依存するポート: 無し。

## 初期構成の読み込み

宣言を受けるドメインが自分のファイルの「初期構成の宣言」の節に公開するものは、[index.md](index.md) の「依存方向」が定める。Installation は、宣言の中身と検証の規則を定めない。区画をまたぐ段の順は [index.md](index.md) の「区画をまたぐ手順」が、ドメインを呼ぶ順は同じ文書の「手順を呼ぶ順」が定める。

宣言の全体の形。境界が、初期構成の記述をこの形にデコードする。この表は、宣言の位置と、その位置の宣言を受けるドメインだけを持つ。i はワークスペースの位置、j は列の中の位置（どちらも 0 始まり）。各位置の宣言の型は、受けるドメインの節が定める。

| 位置（`path`） | 宣言 | 受けるドメイン |
| --- | --- | --- |
| `workspaces.i` | ワークスペース | Workspace |
| `workspaces.i.owner` | 所有者。ワークスペースごとに一人 | Identity |
| `workspaces.i.tokens.j` | トークン | Token |
| `workspaces.i.settings.mailConnection` | 送信接続設定。宣言しなければ `null` | Form |
| `workspaces.i.settings.webhookEndpoints` | 表示アプリへの通知の宛先の列。宣言しなければ空 | Webhook |
| `workspaces.i.settings.previewUrl` | プレビュー先の表示アプリの URL。宣言しなければ `null` | Token |
| `workspaces.i.settings.contentRetention` | コンテンツの保持方針。宣言しなければ `null` | Content |
| `workspaces.i.forms.j` | フォームのキーと、フォームの運用設定 | Form |

- 形に合わない記述は、境界が `ValidationError` として拒否する。所有者を宣言していないワークスペースと、所有者を複数並べたワークスペースは、形に合わない記述に当たる。CMS は起動せず、該当する宣言と理由を示す
- ユースケースは、各ドメインの検証に、宣言と、この表の位置を `path` として渡す。`DeclarationIssue`（[common.md](common.md)）の `path` は、この位置に、各ドメインが項目の位置を続けたものになる。値そのものを含まない
- 宣言は、秘密の値を、保護していない値で含む。宣言は保存せず、通常のログ、操作履歴、起動の結果に含めない

検証。

- [index.md](index.md) の「手順を呼ぶ順」の順に、宣言を受けるドメインの検証を呼び、返った `DeclarationIssue` の列を、その順につなぐ。一件でも問題があれば、CMS は起動せず、列を返す。何も作らない

生成。

- 問題が無い宣言の生成の段と UnitOfWork は、`spec/usecases/installation.md` の「設置を起動する」のトランザクション境界が、各段が呼ぶ振る舞いとポートは、各ドメインの「初期構成の宣言」が定める
- 起動の結果は、読み込んだワークスペース、所有者、トークンの名称、宣言した運用設定の種類を示す。秘密情報の値を含まない。続きを進めた起動も、読み込みを終えると、同じ内容を示す
- 各ドメインの生成のやり直し（確定済みの段の通過）と、途中で止まったときに各ドメインの集約と記録に残るものは、各ドメインの「初期構成の宣言」が定める

## 導入手順

導入手順は、ホスティング運営者が設置の導入の前に確認する文書で、運用条件を示す。CMS の外で提供する。運用条件は、次の五つ。

| 示すこと | 内容 | 設計が定める部分 |
| --- | --- | --- |
| 対応する実行環境 | 設置を動かせる実行環境と、設置の設定に宣言する項目（`InstallationSetting`） | 対応する実行環境は、文書が示す。設定の確かめ方は `InstallationEnvironment.inspectSettings` と `probe`、示し方は `BootReadiness` と `StatusAssessment.setupCheck` |
| 必要な資源 | データの保存容量、メディアの保管先の容量、通知用の空き | 資源の種類は `ResourceKind`。不足の判定は `StatusAssessment.assessResource`。不足する設置は、`BootReadiness` が起動させず、動作確認と状態確認が不足を示す |
| 利用上限 | 資源ごとの上限と、復旧できるバックアップの大きさの上限 | 資源の上限は `ResourceReading.limit`、復旧できる大きさの上限は `InstallationEnvironment.restorableSizeLimit`。状態確認が、上限に対する現在の利用量を示す。値は、文書が示す |
| バックアップ間隔 | バックアップを取得する間隔の上限と、バックアップの保持期間 | `BackupPolicy`。取得の起こし方は [index.md](index.md) の「定期ジョブ」。間隔を過ぎた設置は、状態確認が `overdue` と `take_backup` で示す |
| 復旧手順 | バックアップの一覧、復旧（復旧前の確認を含む）、利用の再開（利用再開前の確認を含む）、起動の順と、復旧の動作を定期的に確かめる方法。復旧の前提（「BackupStore」の、復旧の前から続く手順） | 「Recovery」「BackupRecord」の「復旧前の確認」、「BackupStore」、「ResumeReview」。復旧の動作の確認は、検証に使う設置への復旧で行う |

## 更新手順

更新手順は、CMS の版ごとに、CMS の外で提供する文書で、ホスティング運営者が設置の更新の前に確認する。文書は、版ごとに、次の五つを示す。

| 示すこと | 内容 | 設計が定める部分 |
| --- | --- | --- |
| 更新前に必要な作業 | 利用者への停止の知らせ、設置の停止、バックアップの取得 | 作業は、文書が示す。取得は「バックアップを取得する」。更新の前に CMS が確かめるものは「UpdateReadiness」が定める |
| 影響 | 更新で形が変わる保存済みのデータと、更新の間に止まる機能 | 形が変わるデータは、版ごとに文書が示す。止まる機能は、利用者の要求のすべてと、定期ジョブ、イベントの消費。受け付けるものは「設置の状態」の `admit` の表が定める |
| 必要な停止時間 | 設置の停止から、更新の後の起動までの時間の目安 | 目安は、版ごとに文書が示す。停止の規則は「UpdateReadiness」が定める |
| 復旧方法 | 更新に失敗した場合に、更新前の版で利用を再開する手順 | 「UpdateReadiness」の「更新中の設置」が定める |
| 更新先の版が必要とする実行環境と資源 | 対応する実行環境と、必要な資源 | 資源の不足は、`UpdateReadiness` が、更新先の版の `InstallationEnvironment.readResource` の結果から判定する |

## 終了手順

終了手順は、ワークスペースの利用終了と、設置全体の利用終了で、ホスティング運営者が行う作業を示す文書。CMS の外で提供する。文書は、次を示す。

| 示すこと | 内容 | 設計が定める部分 |
| --- | --- | --- |
| ワークスペースの利用終了 | 終了前の確認、名称を指定した終了の一回の実行、状態確認での完了の確かめ | 確認の内容は「WorkspaceClosure」。消去の手順と順は [index.md](index.md) の「区画をまたぐ手順」と、各ドメインの「利用終了の消去」。残りの消去は、定期ジョブ「利用終了の完了」が進める |
| 設置全体の終了前の確認 | ワークスペースごとの、未完了の通知と失われるデータ | 「WorkspaceClosure」の、設置全体の終了前の確認 |
| 設置の停止 | すべてのワークスペースの配信・受付・通知の終了 | 停止は CMS の外の操作で、設置の状態を変えない |
| 保存先と記録の消去 | データの保存先、メディアの保管先、復旧の記録、設置の設定と初期構成に含む資格情報の消去 | 「WorkspaceClosure」の、設置全体の利用終了。消去の後の設置の状態は、「設置の状態」の `of` が定める |
| バックアップの整理 | 残すバックアップと削除するバックアップの決定と、保管先での削除 | 保持・削除期限は `BackupRecord.retainUntil`。示し方と削除は「BackupRecord」のライフサイクル |
| CMS の外に残る情報 | 外部のメール送信環境、受信箱、表示アプリに渡した内容 | 各ドメインが「利用終了の消去」に公開する、CMS の通常データの外に残る情報（`ClosurePreview.remainsOutside`） |

- 検証に使う設置の終了は、設置の停止と、保存先と記録の消去だけを行う

## ポート

ポートは `packages/core/src/domain/installation/ports/` に置く。どのポートもドメイン型を受け渡す。`InstallationRepository`（設置の区画のリポジトリ）と、`BackupStore` の読み取りは、[common.md](common.md) の「TransactionalRepository」の、すべてのリポジトリに共通の規則に従う。`BackupStore` の記録の順序は、共通の規則と違い、`findPage` の契約が定める。下の表は、共通の規則に加える契約だけを書く。`BackupStore`、`RecoveryJournal`、`InstallationEnvironment` は、区画の外または区画に結び付かないポート（[index.md](index.md) の「集約とポートの置き場所」）で、UnitOfWork との関係は [common.md](common.md) の「UnitOfWorkProvider」に従う。この三つは、外部の相手を持つポート（[common.md](common.md) の「TransactionalRepository」）で、保存先と保管先の障害は `SystemError` を返す。メソッドの契約が、障害を結果で返すと定めるメソッドは、その契約に従う。

### InstallationRepository

目的: 設置に一つの `Installation` を保存し、取り出す。`InstallationContext` と `PartitionReader.installation()` が `installationRepository` として公開する。

```ts
interface InstallationRepository {
  find(): Promise<Versioned<Installation> | null>;
  insert(installation: Installation): Promise<void>;
  save(installation: Installation, expectedVersion: ExpectedVersion<Installation>): Promise<void>;
}
```

メソッドの分類: `find` は読み取り。`insert` と `save` は書き込み。

| メソッド | 契約 |
| --- | --- |
| `find` | 設置の `Installation` を返す。無ければ `null`。`ExpectedVersion` を返す。`initialConfiguration` は、保存した変種と、`loading` の照合値を、`update` は、保存した変種と、`updating` の更新先の版を、そのまま返す |
| `insert` | 設置に一つであることをポートが担保する。既にあれば `ConflictError`（理由 `INSTALLATION_EXISTS`）。並行した `insert` は一方だけが成功し、失敗した側の UnitOfWork の書き込みはすべて反映されない |
| `save` | 楽観ロックで保存する。`expectedVersion` が保存済みの `version` と違う場合と、`Installation` が無い場合は `ConflictError`（理由 `STALE_VERSION`）。初期構成の読み込みの完了と、設置の更新の開始と完了が使う |

- 書き込み（`insert`、`save`）は、設置の区画の UnitOfWork（`{ kind: "installation" }`）の中で呼び、その UnitOfWork のほかの書き込みと、全部成功か全部反映されないかを共にする。`find` は、`PartitionReader.installation()` のリポジトリでも呼べる
- `Installation` は、設置全体の利用終了まで残る
- 成功した `BackupStore.restore` の後の `find` は、バックアップの中の `Installation` を、取得の時点の値のまま返す。`version` も、取得の時点の値に戻る。空の保存先（`Installation` が無い設置）への復旧の後も同じ
- `Installation` を保存する UnitOfWork の読み方は、`spec/usecases/installation.md` の「区画と UnitOfWork」が定める
- 保存する UnitOfWork の中で読んだ `Installation` の `version` が、その実行が前提にした `Installation`（設置の状態の確認で読んだもの。同じ実行が先に保存した場合は、保存した結果）の `version` と違う場合と、`Installation` が無い場合は、何も書かずに `ConflictError`（理由 `STALE_VERSION`）。並行した実行が先に確定した場合が当たる。設置の更新の開始で、中で読んだ `Installation` が更新中（並行した実行が開始を先に確定した）なら、理由は `UPDATE_IN_PROGRESS`。`version` が同じ `Installation` は、前提にしたものと同じ変種なので、振る舞いに渡せる
- `Installation` が無いことは、`find` が `null` を返すことで判定する。ほかの集約の有無を調べない

### BackupStore

目的: バックアップの保管先に、設置のデータの写しとその記録を一つのバックアップとして保ち、設置のデータをバックアップの内容へ戻す。復旧の記録は持たない。

```ts
type BackupCapture = Readonly<{
  takenAt: Date;
  dataVersion: DataVersion;
  workspaces: readonly WorkspaceRef[];
  sizeBytes: number;
}>;

type BackupVerification =
  | Readonly<{ intact: true }>
  | Readonly<{ intact: false; reason: string }>;

type LatestBackupReading =
  | Readonly<{ status: "read"; record: BackupRecord | null }>
  | Readonly<{ status: "unavailable"; reason: string }>;

interface BackupStore {
  capture(id: BackupRecordId): Promise<BackupCapture>;
  verify(id: BackupRecordId): Promise<BackupVerification>;
  complete(record: BackupRecord): Promise<void>;

  findById(id: BackupRecordId): Promise<BackupRecord | null>;
  findLatest(): Promise<LatestBackupReading>;
  findPage(pagination: Pagination): Promise<PaginationResult<BackupRecord>>;

  restore(recovery: RestoringRecovery): Promise<void>;
}
```

バックアップは、取得中（`capture` を始めてから `complete` まで）と、取得済み（`complete` の後）のどちらか。取得済みのバックアップだけが、記録を持ち、読み取りと `restore` の対象になる。写しだけ、または記録だけがある状態は、読み取りと `restore` から見えない。

バックアップは、区画ごとの写しからなる。

- 設置の区画の写しと、その写しのワークスペースの名簿にある各ワークスペースの区画の写しを持つ。名簿の状態（準備中、開設済み、終了中）によらない。名簿に無いワークスペースの区画は、バックアップに入らない
- 各区画の写しは、その区画の一つの時点の内容で、その区画の中で整合している。一つの UnitOfWork は一つの区画だけを書くので、確定した UnitOfWork の書き込みは、すべて含むか、まったく含まないかのどちらかになる
- 設置の区画の写しを先に取り、ワークスペースの区画の写しを後に取る。`takenAt` は、設置の区画の写しの時点の `Clock.now` の値で、`capture` の呼び出しの開始から完了までの間の時刻になる。ワークスペースの区画の写しの時点は、`takenAt` 以降で、区画ごとに違ってよい
- 区画の間の時点は揃わない。ワークスペースの区画の写しは、設置の区画の写しより後の書き込みを含み得る
- `takenAt` 以前に確定した書き込みは、どの区画のものも含む
- 区画の写しの後、メディアの実体の写しの前に保管先から取り除かれた実体（並行したメディアの削除、または派生形式の置き換えの後の、消去待ちの消去）は、写しに入らないことがある。その取得は、`verify` が `intact: false` を返し、復旧に使えないバックアップになる

`BackupCapture.workspaces` は、設置の区画の写しの名簿にある項目の ID と名称を、ID の昇順で持つ。

`BackupCapture.dataVersion` は、バックアップが記録するデータの版で、取得の時点で設置に保存済みのデータの版（`Installation` が持つ `dataVersion`）。実行中の CMS が扱う版（`InstallationEnvironment.dataVersion`）ではない。更新が必要な設置（保存済みの版が実行中の版と違う）で取得したバックアップは、保存済みの版を記録する。`capture` は、`Installation` がある設置で呼ぶ（取得を受け付ける状態は「設置の状態」の `admit` の表）。

`BackupCapture.sizeBytes` は、取得した写しの全体（すべての区画の写しと、メディアの実体）の大きさ（バイト）。0 以上の整数。

バックアップの対象は、設置のデータの全体とする。

- 設置の区画の全体。`UnitOfWorkProvider` の `InstallationRepositories`（[common.md](common.md)）のリポジトリが保存する、すべての集約と記録
- 名簿のワークスペースの区画の全体。`WorkspaceRepositories` のリポジトリが保存する、すべての集約と記録（スキーマの反映の準備領域を含む）と、区画が記録したドメインイベント。ドメインイベントは、配送の状態（未配送、再配送を待つもの、隔離されたもの）によらず含む。復旧の後のイベントは、取得の時点の配送の状態を持つ
- Media の範囲（バックアップが持つ実体と、写しに含めない区画の記録）は、[media.md](media.md) の「復旧の置き換え」が定める
- 秘密情報は、保護した形（`ProtectedSecret`）のまま含む。別に管理する秘密情報（`SecretCipher` の鍵）は含まない
- 区画に結び付かないポートが保つ記録（利用量の制限の記録、コンテンツの編集中の記録）は含まない。`capture` は、これらの記録を写さない。`restore` は、これらの記録を読まず、変えない。復旧の後も、復旧の前と同じ内容を読める
- バックアップの記録（`BackupRecord`）と復旧の記録（`Recovery`）は含まない。設置のデータの外にあるので、`restore` は変えず、復旧の後も、復旧の前と同じ内容を読める

バックアップの取得。

| メソッド | 契約 |
| --- | --- |
| `capture` | 初めに、取得中のまま残っているほかのバックアップの写しを消す。続けて、設置のデータの全体を、上の区画ごとの写しとして、`id` の取得中のバックアップに取得する。失敗または中止した取得は、取得済みのバックアップを残さない。同じ `id` のバックアップ（取得中、取得済み）が既にあれば `ConflictError`（理由 `ALREADY_EXISTS`）。置き換えと更新との重なりの結果は「取得・置き換え・更新の排他」が定める。取得済みのバックアップの `Installation` は、どれも `update` が `idle` |
| `verify` | `id` のバックアップ（取得中、取得済み）が、取得した内容のまま欠けず、区画の写しについてバックアップが持つメディアの実体（Media の「復旧の置き換え」）をすべて含み、復旧に使えることを確かめる。設置のデータを変えない。欠けや改変は `intact: false` と理由で返す。バックアップが無ければ `NotFoundError`。`intact: false`（保管先を直接壊した場合と、取得と並行して実体が取り除かれた場合）は、試験用の相手で、アダプターの写しの進め方によらずに再現できないので、アダプターごとの検証の対象で、ポート適合テストの対象にしない |
| `complete` | `record.id` の取得中のバックアップに記録を付け、取得済みにする。記録の確定と、取得済みになることは同時に起こる。`record.id` の取得中のバックアップが無ければ `NotFoundError`。既に取得済みなら `ConflictError`（理由 `ALREADY_EXISTS`） |

- 並行した取得は、後から `capture` を始めた側が、先に始めた側の取得中の写しを消す。先に始めた側は、`verify` または `complete` が `NotFoundError` になり、取得済みにならない。取得済みのバックアップは、`capture` で消えない

バックアップの読み取り。

| メソッド | 契約 |
| --- | --- |
| `findById` | 取得済みのバックアップの記録を一件返す。無ければ `null` |
| `findLatest` | `read` と、`findPage` と同じ順の先頭の記録（`record`）を返す。`takenAt` が最も新しい記録で、`takenAt` が同じ記録の間は ID の降順で先頭のもの。整合性の確認の結果によらない。一件も無ければ `record` は `null`。保管先から読めなければ `unavailable` と理由を返し、例外を投げない。理由は秘密情報を含まない。読めない保管先は、試験用の相手で再現できないので、`unavailable` はアダプターごとの検証の対象で、ポート適合テストの対象にしない |
| `findPage` | 記録を、`takenAt` の降順、同じ日時は ID の降順で返す |

- 保管先で削除されたバックアップは、どの読み取り（`findPage` の `count` を含む）にも現れない
- 取得済みのバックアップの記録は、変わらない
- 設置の保存先を失っても、バックアップは読める。保存先を失った後に読めることは、試験用の相手で再現できないので、アダプターごとの検証の対象で、ポート適合テストの対象にしない

復旧。

| メソッド | 契約 |
| --- | --- |
| `restore` | 設置のデータの全体を、`recovery.backupId` の取得済みのバックアップの内容に置き換える。成功した場合、設置の区画と、バックアップが含む各ワークスペースの区画は、バックアップのその区画の写しの内容になり、メディアの保管先は、バックアップが持つメディアの実体（Media の「復旧の置き換え」）を、取得の時点の内容で持つ。バックアップに無い現在のデータ、バックアップに無いワークスペースの区画、バックアップに無いメディアの実体は残らない。打ち切りは下の「打ち切り」のとおり。バックアップが無ければ、置き換えを始めずに `NotFoundError`。取得と更新との重なりの結果は「取得・置き換え・更新の排他」が定める |

- 引数は `RestoringRecovery`（「Recovery」の不変条件）
- 途中で止まった置き換え: 契約は、区画をまたいで、全部成功か全部反映なしになることを約束しない。失敗、打ち切り、中断した呼び出しは、一部の区画と実体だけを置き換えた設置を残し得る。契約が約束するのは、区画ごとの結果で、一つの区画の中は、置き換えの前の内容か、バックアップの写しの内容のどちらかで、その間の内容にならない。どの区画がどちらになるかは、約束しない
- 繰り返し: `restore` は、呼び出しの前の設置のデータによらず、成功すれば、同じバックアップから同じ結果を作る。前の呼び出し（同じバックアップ、別のバックアップ）がどこで止まっていても、空の保存先でも、設置のデータを、指定したバックアップの内容に置き換える。成功した `restore` を、同じバックアップでもう一度呼んでも、結果は変わらない
- メディアの実体: バックアップに無い実体を取り除くことは、`restore` の成功に含む。取り除きを終えていない呼び出しは、成功を返さない。取り除きと、バックアップの実体の保管は、Media の「復旧の置き換え」（[media.md](media.md)）の手順と前提で行う。Installation は、実体の位置の規則を持たない
- 一意性・整合性: 復旧後の各区画の中の整合は、ポートが担保する。復旧後の読み取りは、区画ごとに、その区画の写しの時点で確定していた書き込みだけを反映する。区画の間の整合は、ポートが担保しない
- 置き換えと書き込み: 置き換えの完了より前に確定した書き込みは、置き換えで無くなる。成功した `restore` の後の設置のデータは、バックアップの内容だけを持つ。置き換えと重なった UnitOfWork と `updateData` の結果は「取得・置き換え・更新の排他」が定める
- 復旧の前から続く手順（運用コマンド）: 復旧は、ほかの運用コマンドの実行が終わっている設置で行う（HOST-08）。運営者が、自分の運用手順で確かめる。CMS は、確かめる出力を持たない。運用コマンドの段の間に復旧が完了することは、この前提の外にある
- 復旧の前から続く手順（利用者の要求とワーカー）: [index.md](index.md) の「区画をまたぐ手順」の表の、参加、メンバーの利用の解除、所属の索引の修復、利用終了の完了、スクリプトの認証の受け取りが当たる。設置の停止の後は、新しく始まらない。停止の前に始まった手順の後の段が、置き換えの完了の後に確定した場合、その書き込みは復旧の後のデータに加わる。生じ得る食い違いと直す主体は、[index.md](index.md) の「手順ごとの規則」の、バックアップと復旧が定める。利用の再開が残件数を読んだ後に確定した所属の索引の食い違いは、残件数に現れず、再開の後に、定期ジョブ「所属の索引の修復」が直す。スクリプトの認証の受け取りの後の段の扱いは、Token の `redeemDeviceAuthorization` が定める
- 可視性: 成功した `restore` の後、すべての読み取りは復旧後の内容を返す
- 復旧の前提: 復旧できる条件は、`BackupRecord.restorability` が判定する。`restore` は、呼び出し側が条件を確かめた後に呼ぶ
- 打ち切り: 置き換えが進む時間は、`Recovery.restoreTimeoutMs` を超えない。その時間の内に置き換えを終えられない呼び出しは、打ち切って、`SystemError`（理由 `RESTORE_TIMED_OUT`）で失敗し、置き換えは進行中で残らない。設置のデータは、途中まで置き換わっていることがある
- 置き換えを始めなかった結果は三つ。バックアップが無い `NotFoundError`、`ConflictError` の `RESTORE_IN_PROGRESS` と `BACKUP_IN_PROGRESS`。どれも、設置のデータを変えていない。結果ごとの復旧の記録の遷移は「Recovery」の、`BackupStore.restore` の結果ごとの表が定める
- 失敗: 三つのほかの失敗は、置き換えを始めたかどうかによらず `SystemError` で、理由は `restore` が付ける。打ち切りの理由は上の「打ち切り」のとおり。保管先の容量不足は `BACKUP_CAPACITY_EXCEEDED`、保管先へ接続できない場合は `BACKUP_DESTINATION_UNREACHABLE`、ほかは `RESTORE_FAILED`。保管先の容量不足と接続できない場合は、試験用の相手で再現できないので、アダプターごとの検証の対象で、ポート適合テストの対象にしない

#### 取得・置き換え・更新の排他

写しの取得（`capture`）、置き換え（`restore`）、データの更新（`InstallationEnvironment.updateData`）、設置のデータへの書き込み（UnitOfWork）が重なったときの結果は、この表だけが定める。拒否した呼び出しは、設置のデータ、取得中の写し、取得済みのバックアップ、進行中の呼び出しを変えない。

| 重なり | 結果 |
| --- | --- |
| 置き換えが進む間の `restore` | `ConflictError`（理由 `RESTORE_IN_PROGRESS`） |
| 置き換えが進む間の `capture` | 写しを取らず、取得中のまま残っているほかの写しも消さずに、`ConflictError`（理由 `RESTORE_IN_PROGRESS`） |
| 置き換えの完了より前に始まり、完了の時点で確定していない UnitOfWork | 確定しない。`run` は `ConflictError`（理由 `RESTORE_IN_PROGRESS`）を投げ、書き込みは一つも反映されない |
| 置き換えの完了より前に始まった `updateData` | 完了の後にデータを変えず、`ok: false`（`failed`） |
| 置き換えの完了の後に始まった `restore`、`capture`、UnitOfWork、`updateData` | 妨げない |
| `capture` の呼び出しが進む間の `restore` | `ConflictError`（理由 `BACKUP_IN_PROGRESS`） |
| `capture` の呼び出しが進む間の `updateData` | データを変えずに `ok: false`（`failed`） |
| `capture` の呼び出しが進む間の UnitOfWork | 妨げない。確定する |
| `updateData` の呼び出しが進む間の `updateData` | データを変えずに `ok: false`（`in_progress`） |
| 設置の区画の写しの `Installation` が更新中（`update` が `updating`）の `capture` | 写しを取らず、ほかの写しも消さずに、`ConflictError`（理由 `UPDATE_IN_PROGRESS`） |

- 置き換えが進む間は、置き換えの開始から、成功、失敗、打ち切りまで
- `capture` と `updateData` の呼び出しが進む間は、呼び出しの開始から、結果を返すまで。結果を返さずに止まった呼び出しは、進行中として残らず、次の呼び出しを妨げない。呼び出した処理の停止は、試験用の相手で再現できないので、止まった呼び出しが次の呼び出しを妨げないことは、アダプターごとの検証の対象で、ポート適合テストの対象にしない
- `capture` を終えた取得中のバックアップ（`verify`、`complete` の前）は、どの呼び出しも妨げない
- 写しは、置き換えの途中と更新の途中の内容を含まない

### RecoveryJournal

目的: 復旧の記録を、設置のデータの外に、設置に一つ保つ。入口と運用コマンドが、処理のたびに読む。バックアップの保管先に依らない。

```ts
interface RecoveryJournal {
  find(): Promise<Versioned<Recovery> | null>;
  save(recovery: Recovery, expectedVersion: ExpectedVersion<Recovery> | null): Promise<void>;
  clear(expectedVersion: ExpectedVersion<Recovery>): Promise<void>;
}
```

| メソッド | 契約 |
| --- | --- |
| `find` | 設置の復旧の記録を、`ExpectedVersion` とともに返す。無ければ `null`。`phase` の変種と、その変種のフィールド（`restoring` の `leaseUntil` を含む）を、保存したまま返す |
| `save` | 復旧の記録を保存する。設置に一つであることをポートが担保する。`expectedVersion` が `null` の保存は、記録が無い場合にだけ成功する。`null` でない保存は、保存済みの `version` と一致する場合にだけ成功する。どちらも、一致しなければ `ConflictError`（理由 `STALE_VERSION`） |
| `clear` | 復旧の記録を消す。`expectedVersion` が保存済みの `version` と違う場合と、記録が無い場合は `ConflictError`（理由 `STALE_VERSION`） |

- 一意性・整合性: 記録が設置に一つであることを、ポートが担保する。`backupId` のバックアップの存在を、ポートは確かめない。呼び出し側が、保存の前に `BackupStore.findById` で読む
- 占有の期限を、ポートは判定しない。保存した `leaseUntil` を返すだけで、期限の内かどうかは `Recovery.isOccupied` が決める
- 並行性: 並行した復旧の開始と利用の再開は、同じ `expectedVersion` で保存または消去する側の一方だけが成功する
- 可視性: 保存と消去は、完了の直後の `find` に現れる
- 記録は、`BackupStore.restore` が置き換えない場所に保つ。`restore` の前後で、同じ内容と同じ `expectedVersion` を返す
- バックアップの保管先へ接続できない間も、読み書きできる。バックアップの保管先の理由（`BACKUP_CAPACITY_EXCEEDED`、`BACKUP_DESTINATION_UNREACHABLE`）を返さない。保管先へ接続できない状態は、試験用の相手で再現できないので、アダプターごとの検証の対象で、ポート適合テストの対象にしない
- 記録を消す手段は、`clear` と、ホスティング運営者が終了手順で行う、記録の保存先の消去
- エラーケース: 記録の保存先の障害は `SystemError`

### InstallationEnvironment

目的: 設置の実行環境から、容量、データの版、更新を持つ版、設置の公開の URL、復旧できる大きさの上限を取得し、保存先と保管先の動作を確かめ、保存済みのデータを更新する。別に管理する秘密情報の識別子は、`SecretCipher.currentKeyId` が返す。

```ts
interface InstallationEnvironment {
  readResource(kind: ResourceKind): Promise<ResourceReading>;
  inspectSettings(): Promise<readonly SettingInspection[]>;
  probe(target: "data_store" | "media_store"): Promise<ProbeResult>;
  dataVersion(): Promise<DataVersion>;
  supportsUpdateFrom(from: DataVersion): Promise<boolean>;
  publicUrl(): Promise<string>;
  restorableSizeLimit(): Promise<number>;
  updateData(plan: UpdatePlan): Promise<UpdateOutcome>;
}

type UpdateOutcome =
  | Readonly<{ ok: true }>
  | Readonly<{ ok: false; cause: "failed"; reason: string }>
  | Readonly<{ ok: false; cause: "in_progress" }>;
```

| メソッド | 契約 |
| --- | --- |
| `readResource` | その資源の使用量と上限を返す。取得できなければ `unavailable` と理由を返す。例外を投げない。状態を変えない |
| `inspectSettings` | 設置の設定の五つの項目ごとに、宣言の有無と、指定の形の妥当性を返す。宣言が無い項目は `not_declared`、必要な指定が欠けている項目と形式が誤っている項目は `invalid` と理由を返す。`public_url` は、`http` または `https` の絶対 URL で、クエリとフラグメントを持たない指定だけが `valid`。検証するのは指定の形までで、相手へ到達できることは確かめない。到達は `probe` が確かめる。例外を投げない。保存先に書き込まず、相手を呼ばない。理由は、設定の秘密の値を含まない |
| `probe` | `data_store` では、保存先に確認用の値を書き、読み出して一致を確かめ、取り除く。`media_store` では、メディアの保管先に確認用の実体を保存し、取り出して一致を確かめ、取り除く。結果は `passed` または `failed` と理由で返し、例外を投げない。確認用の値と実体は、ワークスペースのデータに現れず、完了後に残らない。理由は、設定の秘密の値を含まない |
| `dataVersion` | 実行中の CMS が扱うデータの版を返す。保存済みのデータを読まない。呼ぶたびに同じ値を返す |
| `supportsUpdateFrom` | 実行中の CMS が、`from` の版の形のデータを、`dataVersion()` の版の形へ更新する手順を持てば真。`from` が `dataVersion()` の値と同じ場合は真。保存済みのデータを読まない。呼ぶたびに同じ値を返す。真を返した `from` について、`updateData({ from, to: dataVersion() })` は、更新を持たないことを理由に失敗しない。設置の更新の前の確認が使う |
| `publicUrl` | 設置の設定が宣言する、設置の公開の URL を返す。`inspectSettings` が `public_url` を `valid` と返す設置で、宣言した値をそのまま返す。保存済みのデータを読まない。呼ぶたびに同じ値を返す。`public_url` が `valid` でない設置では `SystemError`。この値を使う運用コマンドは、`InstallationState.blockingIssues` が `public_url` の問題を返さない設置でだけ呼ぶ。起動した設置の入口も同じ |
| `restorableSizeLimit` | 運用条件が示す、復旧できるバックアップの大きさの上限（バイト）を返す。0 以上の整数。保存済みのデータを読まない。呼ぶたびに同じ値を返す。`BackupRecord.sizeBytes` と同じ数え方の値 |
| `updateData` | 保存済みのデータを、`plan.to` の版の形へ更新する。`Installation` の記録は変えない。呼び出しの前のデータが、`plan.from` の版の形、同じ `plan` の更新が途中で止まった形、`plan.to` の版の形のどれでも、成功すれば `plan.to` の版の形になる。既に `plan.to` の版の形の場合（成功した更新の呼び直し）は、データを変えずに `ok: true`。失敗は `ok: false`（`failed`）と理由で返し、例外を投げない。理由は、設定の秘密の値を含まない。失敗または中断した更新の後のデータの形を、契約は約束しない。同じ `plan` で呼び直せる。`plan.to` が `dataVersion()` の値と違う場合と、`supportsUpdateFrom(plan.from)` が偽の場合は、データを変えずに `ok: false`（`failed`）。ほかの `updateData`、写しの取得、置き換えとの重なりの結果（`in_progress` を含む）は、「BackupStore」の「取得・置き換え・更新の排他」が定める |

- 並行性: 呼び出しが重なったときの結果は、「取得・置き換え・更新の排他」の表だけが定める。表に無い組の呼び出しは、互いに影響しない
- `updateData` が途中で失敗したデータの状態と、結果を返さずに止まった呼び出しの後の状態は、アダプターの更新の進め方に依り、試験用の相手で再現できないので、アダプターごとの検証の対象で、ポート適合テストの対象にしない。ポート適合テストは、失敗を返した呼び出しの後に、同じ `plan` で呼び直せることまでを確かめる
- 可視性: `readResource` は、呼び出しの時点の値を返す。直前の書き込みの反映は遅れてよい
- 容量の不足（`readResource` が上限に近い値を返す実際の状態、`probe` の容量による失敗）は、試験用の相手で再現できないので、アダプターごとの検証の対象で、ポート適合テストの対象にしない。ポート適合テストは、試験用の相手に指定した値と応答が、結果にそのまま現れることを確かめる
- `publicUrl` の値の使い方は、[index.md](index.md) の「URL の組み立て」が定める

## トランザクション境界

共通の `UnitOfWorkProvider.run(scope, fn)` を使う（[common.md](common.md) の「WriteScope」「UnitOfWorkProvider」）。`InstallationContext` が `installationRepository` を公開する。各ユースケースが開く UnitOfWork、その `WriteScope`、UnitOfWork に属さない呼び出しとの順は、`spec/usecases/installation.md` の「区画と UnitOfWork」と各ユースケースのトランザクション境界が定める。

## 利用終了の消去

Installation は、ワークスペースの利用終了で消去する集約を持たない。`Installation` と復旧の記録は設置に一つで、ワークスペースに属さない。バックアップは、設置のデータの外にあり、ワークスペースの利用終了で変わらない。

## ユースケース（概要）

- 設置を起動する
- 設置の状態を確かめる
- 設置を更新する
- 導入時の動作確認を行う
- ワークスペースを作成し、初期所有者の招待を発行する
- 初期所有者の招待を発行し直す
- 紐付けを待つ運用設定を一覧する
- 状態を確認する
- バックアップを取得する
- バックアップを一覧する
- 復旧前の確認を示す
- バックアップから設置全体を復旧する
- 利用再開前の確認を示す
- 利用を再開する
- ワークスペースの終了前の確認を示す
- ワークスペースの利用を終了する
- ワークスペースの利用終了を完了させる（定期ジョブ「利用終了の完了」）
- 設置全体の終了前の確認を示す

各ユースケースが使う手順と項目は、`spec/usecases/installation.md` が定める。

どのユースケースの出力も、パスワード、トークンの値、運用設定の秘密情報、回答の入力内容を含まない（`spec/scenario/hosting.md` の「共通の規則」）。初期所有者の招待の値を出力に含める設置は、[index.md](index.md) の「URL の組み立て」が定める。
