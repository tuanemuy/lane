# Content

コンテンツの公開内容・下書き・ごみ箱・履歴と、公開・参照・一意性の規則を保つ。境界、依存方向、ドメインをまたぐ規則は [index.md](index.md) が、集約の ID、`Actor`、`TokenDesignation`、`PublishedChangeNotice`、`TransactionalRepository`、`UnitOfWorkProvider`、共通のエラーの理由は [common.md](common.md) が定める。Content と Media の分担は、[index.md](index.md) の「参照の記録と保護」に従う。Content の集約とポートの区画は、[index.md](index.md) の「集約」と「集約とポートの置き場所」の表に従う。

コードは `packages/core/src/domain/content/` に置く。

## ユビキタス言語

| 英語名 | 日本語名 | 定義 |
| --- | --- | --- |
| Content | コンテンツ | モデルに属する一件。変わらないコンテンツID と、公開内容・下書きを持つ |
| Body | 内容 | 一件のコンテンツの、すべての項目の値の組。公開内容、下書き、履歴の版が一つずつ持つ |
| Published body | 公開内容 | 公開取得で配信する内容 |
| Draft | 下書き | 公開していない編集中の内容。コンテンツごとに一つ |
| Latest body | 最新の内容 | 下書きがあれば下書き、なければ公開内容 |
| Publication status | 公開状態 | 未公開（公開内容なし）または公開中（公開内容あり） |
| Location | 置き場所 | 通常の管理対象、またはごみ箱 |
| Structure violation | 構造の違反 | 現行スキーマの値の種類・構造に従わない値。保存できない。当たるものは「ContentIssue」の `StructureViolation` が定める |
| Type error | 型を満たさない入力エラー | 値が項目の型を満たさないこと。当たるものは「ContentIssue」が定める |
| Constraint violation | 制約違反 | 値が項目の制約を満たさないこと。当たる制約は「ContentValidation」が定める |
| Issue | 入力エラー | 型を満たさない入力エラーと制約違反の総称 |
| Reference | 参照 | 内容が、他のコンテンツまたはメディアを ID で指すこと |
| Required reference | 必須の参照 | `required` が真の項目による、コンテンツへの参照 |
| Optional reference | 任意の参照 | `required` が偽の項目による、コンテンツへの参照 |
| Body link | 本文のリンク | リッチテキストの中の、コンテンツへのリンク |
| Protecting reference | 保護する参照 | 参照先の公開内容と存在を保護する、コンテンツへの参照。当たる参照は「ContentReference」が定める |
| Media reference | メディアへの参照 | 内容がメディアを使う一箇所。使用箇所ごとの説明を持つ |
| Body holder | 内容の持ち主 | 内容を持つもの。一件のコンテンツの公開内容、下書き、一つの履歴の版のいずれか |
| Resolved reference | 解決できる参照 | 参照先を返せる状態にある参照 |
| Broken reference | 参照切れ | 解決できない参照 |
| Unique value | 一意な値 | 一意にする制約を持つ項目の値 |
| Revision | 履歴の版 | 操作が置き換えた、またはなくした過去の内容 |
| Provenance | 内容の由来 | 内容を作った操作、実行者、日時 |
| Rewrite | 値の書き換え | 指定した項目の値だけを変える操作 |
| Publication change | 公開内容の変更 | 一件のコンテンツの公開・更新・公開取り消し |
| Publication sequence | 通し番号 | 公開の変更の記録に割り当てる、ワークスペースごとの番号。規則は `PublicationLedgerRepository` の契約が定める |
| Publication record | 公開の変更の記録 | 公開内容を変える一つの操作の、通し番号、操作の ID、操作の種類、実行者、変更の一覧と参照元の一覧、スキーマの版 |
| Optimistic lock version | `version` | 一件のコンテンツの、楽観ロックの番号。この番号は `version` と書く。単独の「版」は履歴の版を指し、スキーマの版は「スキーマの版」と書く |
| Operation ID | 操作の ID | 公開内容を変える操作の一回を指す、入口が作る ID |
| Public fetch | 公開取得 | 公開内容を返す取得 |
| Preview fetch | プレビュー取得 | 最新の内容を返す取得 |
| Expansion | 参照先の展開 | 参照先の内容を、参照元と同じ取得で返すこと |
| Reference summary | 参照先の要約 | 解決できる参照の参照先ごとに返す、参照先のモデル、コンテンツID、一意な値 |
| Purge | 完全削除 | コンテンツを、公開内容・下書き・履歴とともに削除すること |
| Retention policy | 保持方針 | 履歴の版とごみ箱のコンテンツを保持する日数 |
| Auto cleanup | 自動整理 | 保持方針を外れた履歴の版を削除する「履歴の版の整理」と、保持方針を外れたごみ箱のコンテンツを完全削除する「ごみ箱の完全削除」。どちらも定期ジョブ（[index.md](index.md) の「定期ジョブ」） |
| Staging area | 準備領域 | スキーマの反映の変換結果を、取得から見えない形で置く場所。反映と事前確認の一回（`jobId`）ごとに分かれる |
| Editing presence | 編集中の記録 | あるコンテンツをあるメンバーが編集していることの、短命な記録 |

## エンティティ

### Content

コンテンツ一件。コンテンツID はワークスペースの中で一意で、集約は `ContentId` で識別する。属するワークスペースは、区画が決める（[common.md](common.md) の「TransactionalRepository」の「区画」）。

```ts
type ContentBase = Readonly<{
  id: ContentId;
  modelId: ModelId;
  cardinality: "single" | "collection";
  createdAt: Date;
  lastChange: ContentLastChange;
  version: number;
}>;

type ContentLastChange = Readonly<{
  operation: LastChangeOperation;
  by: Performer;
  at: Date;
}>;

type BodyBase = Readonly<{
  values: ContentValues;
  references: readonly ContentReference[];
  uniqueValues: readonly UniqueValue[];
  updatedAt: Date;
  migrated: MigrationMark | null;
}>;
type MigrationMark = Readonly<{ at: Date; schemaVersion: number }>;
type DraftBody = BodyBase &
  Readonly<{ produced: Provenance<DraftProducingOperation>; issues: readonly ContentIssue[] }>;
type PublishedBody = BodyBase &
  Readonly<{ produced: Provenance<PublishedProducingOperation>; violations: readonly ConstraintIssue[] }>;

type Unpublished = Readonly<{ status: "unpublished"; draft: DraftBody }>;
type Published = Readonly<{ status: "published"; published: PublishedBody; draft: DraftBody | null }>;

type InActive = Readonly<{ kind: "active" }>;
type InTrash = Readonly<{ kind: "trashed"; trashedAt: Date; trashedBy: Performer }>;

type Content = ContentBase &
  (
    | Readonly<{ location: InActive; state: Unpublished | Published }>
    | Readonly<{ location: InTrash; state: Unpublished }>
  );

type ActiveContent = Content & Readonly<{ location: InActive }>;
type TrashedContent = Content & Readonly<{ location: InTrash }>;
type UnpublishedContent = ActiveContent & Readonly<{ state: Unpublished }>;
type PublishedContent = ActiveContent & Readonly<{ state: Published }>;
type DraftedContent = ActiveContent &
  Readonly<{ state: Unpublished | (Published & Readonly<{ draft: DraftBody }>) }>;
```

| フィールド | 制約 |
| --- | --- |
| `id` | 必須。生成後に変わらない。ワークスペースの中で一意。形式は [common.md](common.md) の「Clock と IdGenerator」が定める |
| `modelId` | 必須。生成後に変わらない。現行スキーマのモデルを指す |
| `cardinality` | モデルの `cardinality` と同じ。生成後に変わらない。単一と複数件の間のモデルの変更は削除と追加で、新しい `ModelId` になる |
| `createdAt` | 作成日時。生成後に変わらない |
| `lastChange` | 最後の変更の操作・実行者・日時。`version` を進める振る舞いが更新する。操作の種類は「操作の種類」が定める。保存の競合で、入力と最新の内容を比べるときに、変えた操作を示すのに使う |
| `state` | 作成時は `unpublished` |
| `location` | 作成時は `active` |
| `values` | 現行スキーマの、そのモデルの構造に従う。構造の違反を持たない |
| `references` | `ContentReferences.of` が `values` から求めた参照 |
| `uniqueValues` | `UniqueValues.of` が `values` から求めた一意な値 |
| `updatedAt` | その内容の更新日時。値は、その内容を作った、または置き換えた振る舞いが決める（「振る舞い」） |
| `produced` | その内容を作った操作、実行者、日時。操作の種類は、下書きと公開内容で別の型で、「操作の種類」が定める |
| `migrated` | スキーマの反映が、その内容の値を最後に変換した日時（`at`）と、変換後のスキーマの版（`schemaVersion`）。変換されていない内容は `null`。始まりの値は、その内容を作った、または置き換えた振る舞いが決める（「振る舞い」） |
| `issues` | 下書きの入力エラー。現行スキーマに対する `ContentValidation.inspect` の結果 |
| `violations` | 公開内容の制約違反。現行スキーマに対する `ContentValidation.inspect` の結果 |
| `version` | 楽観ロックの番号。作成時は 0。進める振る舞いは「振る舞い」が定める |

CMS が管理する日時は、`createdAt` と、公開内容・下書きそれぞれの `updatedAt`。

#### 振る舞い

```ts
type ContentChange<T extends Content, K extends PublicationChangeKind = never> = Readonly<{
  content: T;
  retired: readonly RetiredBody[];
  publication: PublicationChange<K> | null;
}>;

const Content: {
  create(
    params: Readonly<{
      id: ContentId;
      modelId: ModelId;
      model: ModelDefinition;
      draft: CheckedDraft;
      occupant: ActiveContent | null;
    }>,
    by: Performer,
    now: Date,
  ): UnpublishedContent;
  isReplayOf(existing: Content, params: Readonly<{ modelId: ModelId; draft: CheckedDraft }>): boolean;

  saveDraft(
    content: ActiveContent,
    model: ModelDefinition,
    draft: CheckedDraft,
    by: Performer,
    now: Date,
  ): ContentChange<DraftedContent>;
  discardDraft(content: PublishedContent & DraftedContent, by: Performer, now: Date): ContentChange<PublishedContent>;
  publish(
    content: PublishableContent,
    model: ModelDefinition,
    by: Performer,
    now: Date,
  ): ContentChange<PublishedContent, "published" | "updated">;
  unpublish(content: PublishedContent, by: Performer, now: Date): ContentChange<UnpublishedContent, "unpublished">;
  trash(content: UnpublishedContent, by: Performer, now: Date): TrashedContent;
  restore(
    content: TrashedContent,
    model: ModelDefinition,
    occupant: ActiveContent | null,
    by: Performer,
    now: Date,
  ): UnpublishedContent;
  rewrite(content: Content, rewrite: CheckedRewrite, by: Performer, now: Date): ContentChange<Content, "updated">;
  restoreRevision(
    content: ActiveContent,
    model: ModelDefinition,
    draft: CheckedDraft,
    by: Performer,
    now: Date,
  ): ContentChange<DraftedContent>;
  applyMigration(content: Content, migrated: MigratedBodies, by: Performer, mark: MigrationMark): ContentChange<Content, "updated">;

  latest(content: Content): Readonly<{ source: "published"; body: PublishedBody }> | Readonly<{ source: "draft"; body: DraftBody }>;
  hasTypeErrors(draft: DraftBody): boolean;

  asActive(content: Content): ActiveContent;
  asTrashed(content: Content): TrashedContent;
  asPublished(content: Content): PublishedContent;
  asDrafted(content: Content): DraftedContent;

  duplicateValues(model: ModelDefinition, source: ContentValues): Readonly<{ values: ContentValues; cleared: readonly FieldKey[] }>;
};
```

| 振る舞い | 処理 |
| --- | --- |
| `create` | `draft` を下書きとして持つ、未公開で `active` のコンテンツを作る。`cardinality` は `model.cardinality`。`version` は 0、`createdAt` と下書きの `updatedAt` は `now`、下書きの `migrated` は `null`。`occupant` は、単一モデルの通常の管理対象に既にあるコンテンツ。複数件のモデルでは `null`。`occupant` が `null` でなければ `BusinessRuleError`（`SingleContentExists`）を投げる |
| `isReplayOf` | `existing` が、同じ `modelId` に属し、最新の内容の `values` が `params.draft.values` と等しいかを返す。作成が、同じ ID のコンテンツが既にある場合に、同じ内容の送り直しかどうかを見分けるのに使う |
| `saveDraft` | 下書きを `draft` で置き換える。置き換えた下書きを `retired` に返す。下書きが無い公開中のコンテンツでは、下書きを作り、`retired` は空。作った、または置き換えた下書きの `updatedAt` は `now`、`migrated` は `null`。公開内容は変えない |
| `discardDraft` | 下書きをなくし、`retired` に返す。公開内容は変えない |
| `publish` | 下書きを公開内容にし、下書きをなくす。置き換えた公開内容を `retired` に返す。公開内容の `updatedAt` は `now`、`migrated` は `null`、`violations` は空。`publication` は、未公開だった場合 `published`、公開中だった場合 `updated` |
| `unpublish` | 公開内容をなくす。下書きがある場合は、下書きを残し、公開内容を `retired` に返す。下書きが無い場合は、公開内容を下書きにし、`retired` は空。下書きにした内容は、`values`・`references`・`uniqueValues`・`updatedAt`・`migrated` を引き継ぎ、`issues` は `violations`。`publication` は `PublicationChange.ofRemoval(content)` |
| `trash` | 未公開のコンテンツの `location` を `trashed` にする。`trashedAt` は `now`、`trashedBy` は `by`。内容を変えない |
| `restore` | `location` を `active` にする。状態は未公開のまま。内容を変えない。`model` は、コンテンツのモデルの現行スキーマの定義。`occupant` は `create` と同じで、`null` でなければ `BusinessRuleError`（`SingleContentExists`）を投げる |
| `rewrite` | `rewrite` が持つ書き換え先の内容を置き換える。置き換えた内容を `retired` に返す。置き換えた内容の `updatedAt` は `now`、`migrated` は `null`。公開内容を置き換えた場合の `publication` は `updated`。値が現在の値と等しい書き換え先は変えず、`retired` にも `publication` にも加えない |
| `restoreRevision` | `saveDraft` と同じ処理を行う。違いは、操作の種類（`revisionRestored`）だけ。`draft` は、`ContentValidation.checkRestore` の結果 |
| `applyMigration` | 公開内容と下書きの `values`、`references`、`uniqueValues`、`issues`、`violations` を `migrated` の内容で置き換える。`values` が変わった内容の `migrated` を `mark` にする。`lastChange` の日時は `mark.at`。`updatedAt` と `produced` は変えない。`retired` は空。公開内容の `values` が変わった場合の `publication` は `updated`。どちらかの `values` が変わった場合に限り、`version` を 1 進め、`lastChange` を更新する |
| `latest` | 下書きがあれば下書き、なければ公開内容を返す |
| `hasTypeErrors` | `issues` が `typeError` を一つでも含むかを返す |
| `asActive` ほか | 状態を絞り込む。合わない場合は `BusinessRuleError` を投げる。`asActive` は `ContentInTrash`、`asTrashed` は `ContentNotInTrash`。`asPublished` と `asDrafted` は、置き場所を先に確かめ、ごみ箱のコンテンツでは `ContentInTrash`、通常の管理対象では、それぞれ `ContentNotPublished`、`ContentHasNoDraft` |
| `duplicateValues` | 複製の入力にする値を返す。一意な項目を未設定にし、そのキーを `cleared` に返す。非推奨の部分は、`DeprecatedUse.carry` に `base` を `null` で渡して取り除く |

- `model` を受ける振る舞いは、「操作の種類」の表で、非推奨のモデルに行えない操作の振る舞い。はじめに `DeprecatedModel.ensure` を、その操作と `model` で呼ぶ。`model` は、コンテンツのモデルの現行スキーマの定義
- 値・状態・置き場所を変えた振る舞いは、`version` を 1 進め、`lastChange` を、その操作・`by`・`now` にする。振る舞いごとの操作の種類と、作る内容の `produced`、`retired` の `Retirement` は「操作の種類」が定める
- 公開中のコンテンツのごみ箱への移動は、`unpublish` の結果の `content` を `trash` に渡す。二つの振る舞いなので、`version` を 2 進める。`lastChange` は `trashed`、なくした公開内容の `Retirement` と、下書きにした内容の `produced` は `unpublished`
- 完全削除とモデルの削除でなくなる公開中のコンテンツの `PublicationChange` は、`PublicationChange.ofRemoval` が作る。振る舞いを呼ばない
- 検証結果（`issues`、`violations`、`references`、`uniqueValues`）は、`values` を現行スキーマと参照先に照らして求める値。求める時点は「不変条件」が定める。スキーマの反映がこれらを求め直しても、`version` と `lastChange` は変わらない
- 現在と同じ値への変更（`saveDraft` と `restoreRevision` で `draft.values` が現在の下書きの `values` と等しいもの、`rewrite` でどの書き換え先の値も現在の値と等しいもの）は、`content` をそのまま返し、`retired` は空、`publication` は `null`。扱いは [index.md](index.md) の「変更の無い変更」が定める
- `CheckedDraft`、`CheckedRewrite`、`PublishableContent`、`MigratedBodies` は、ドメインサービスだけが作るブランド型。検証を経ない値は、振る舞いに渡せない
- 検証済みの値は、検証した相手を持つ。`CheckedDraft` は検証に使ったモデルの `modelId`、`CheckedRewrite` と `MigratedBodies` は対象の `contentId` を持つ。振る舞いは、受けた値の `modelId` がコンテンツの `modelId`（`create` では `params.modelId`）と、`contentId` が `content.id` と一致し、`MigratedBodies` の変種が `content` の持つ内容（公開内容だけ、下書きだけ、両方）と同じことを確かめる。違えば `BusinessRuleError`（`CheckedValueMismatch`）を投げる
- 値の等価性は、キーの集合が同じで、対応する値が入れ子を含めてすべて等しいこと。並びの順序を区別する
- `retired` の内容は、`Revision.record` が履歴の版にする

#### 競合の検出

読み取った時点を示す情報は `version`。読み取った内容を前提にする要求は、読み取ったときの `version` を添える。添える要求は、各ユースケースの入力が示す（[../usecases/content.md](../usecases/content.md)）。保存済みの `version` と違う `version` を添えた要求は、競合（`STALE_VERSION`）。管理画面の操作と、トークンによる操作は、同じ規則に従う。

- 添えた `version` が古くても、振る舞いが `content` をそのまま返す要求（「振る舞い」の、現在と同じ値への変更）は、競合にせず、変更の無い変更として成功を返す。届いていた要求の送り直しを、二重に書かない

#### 不変条件

- 公開内容と下書きの少なくとも一方を持つ
- ごみ箱のコンテンツは未公開
- 公開内容は、型を満たさない入力エラーを持たない
- 公開内容の保護する参照は、同じワークスペースの、通常の管理対象にある公開中のコンテンツを指す。公開内容のほかの参照と、下書き・履歴の版の参照は、参照切れになり得る
- 内容が使うメディアの条件は、[media.md](media.md) の「MediaUsageRepository」の契約が定める
- 一意な値は重複しない。重複は、同じ `modelId` の二つの公開内容が、同じ項目に同じ値を持つこと。下書き、ごみ箱のコンテンツ、履歴の版の値は予約しない
- 保存済みの `values` は構造の違反を持たない
- `issues`、`violations`、`references`、`uniqueValues` は、内容を書く操作とスキーマの反映の確定が、その時点の現行スキーマと参照先に照らして求め、保存する。現行スキーマに照らす結果は、現行スキーマに対して求めた結果と一致する。参照先のモデルに照らす `referenceMismatch` は、内容を書いた時点の参照先から決まり、参照先の完全削除と、同じ ID での作り直しでは変わらない
- 単一モデルは、通常の管理対象にコンテンツを一つまで持つ。ごみ箱のコンテンツは数えない

公開内容の保護する参照、一意な値、単一モデルの条件は、`ContentRepository` が確定の時点で担保する。メディアの条件の担保は、[index.md](index.md) の「参照の記録と保護」に従う。

#### ライフサイクル

```text
create → 未公開 ─ publish → 公開中（下書きなし） ─ saveDraft / restoreRevision → 公開中（下書きあり）
           │                    │                         │ publish → 公開中（下書きなし）
           │                    │                         └ discardDraft → 公開中（下書きなし）
           │                    └ unpublish → 未公開 ←─────┘ unpublish
           └ saveDraft / restoreRevision → 未公開

通常（未公開） ─ trash → ごみ箱（未公開） ─ restore → 通常（未公開）
通常（公開中） ─ unpublish、trash（一つの操作 `trashed`） → ごみ箱（未公開）

通常（未公開、公開中）、ごみ箱（未公開） ─ 完全削除（`ContentRepository.deleteMany`） → 終端
```

- `rewrite` と `applyMigration` は、公開状態と置き場所を変えない
- 完全削除は後続のエンティティを持たない
- ごみ箱のコンテンツは、復元（`restore`）で通常の管理対象へ戻り、利用者が対象を明示した完全削除、ごみ箱の完全削除（「ContentRetentionPolicy」）、モデルの削除でなくなる。`ContentRepository.deleteMany` が `REFERENCED` で拒否したごみ箱のコンテンツは、ごみ箱に残る。参照が無くなれば、次の起動が削除する。互いを保護する参照で指す、期限を過ぎたごみ箱のコンテンツ二件は、一件ずつの完全削除がどちらも拒否されるので、自動整理では消えない。この二件の終端は、利用者が二件を同じ操作の対象に加えて行う完全削除（対象間の参照は妨げない）、または復元

#### 操作の種類

内容と状態を変える操作の種類は、次の表が定める。`lastChange`、`Provenance`、`Retirement`、公開の変更の記録の `operation` と記録が持てる変更の種類、非推奨のモデルで行えるかは、この表から導く。

```ts
type ContentOperation =
  | "draftSaved"
  | "draftDiscarded"
  | "published"
  | "unpublished"
  | "trashed"
  | "restoredFromTrash"
  | "revisionRestored"
  | "rewritten"
  | "migrated"
  | "purged";

type LastChangeOperation = Exclude<ContentOperation, "purged">;
type DraftProducingOperation = Extract<ContentOperation, "draftSaved" | "unpublished" | "revisionRestored" | "rewritten">;
type PublishedProducingOperation = Extract<ContentOperation, "published" | "rewritten">;
type DraftRetiringOperation = Extract<ContentOperation, "draftSaved" | "draftDiscarded" | "revisionRestored" | "rewritten">;
type PublishedRetiringOperation = Extract<ContentOperation, "published" | "unpublished" | "rewritten">;
type PublicationOperation = Extract<
  ContentOperation,
  "published" | "unpublished" | "trashed" | "purged" | "rewritten" | "migrated"
>;
```

| 操作 | 行う振る舞い | `lastChange` | 作る内容と、その `produced` | 履歴の版にする内容（`retired`）と、その `Retirement` | 公開の変更の記録の `operation` と、記録が持てる変更の種類 | 非推奨のモデル |
| --- | --- | --- | --- | --- | --- | --- |
| `draftSaved` | `create`、`saveDraft` | 持つ | 下書き | 置き換えた下書き | 持たない | 行えない |
| `draftDiscarded` | `discardDraft` | 持つ | なし | なくした下書き | 持たない | 行える |
| `published` | `publish` | 持つ | 公開内容 | 置き換えた公開内容 | 持つ。`published`、`updated` | 行えない |
| `unpublished` | `unpublish` | 持つ | 下書きが無い場合の、下書きにした内容 | 下書きがある場合の、なくした公開内容 | 持つ。`unpublished` | 行える |
| `trashed` | `trash` | 持つ | なし | なし | 公開中だった対象を、同じ操作で `unpublish` した場合に持つ。`unpublished` | 行える |
| `restoredFromTrash` | `restore` | 持つ | なし | なし | 持たない | 行えない |
| `revisionRestored` | `restoreRevision` | 持つ | 下書き | 置き換えた下書き | 持たない | 行えない |
| `rewritten` | `rewrite` | 持つ | 書き換えた内容 | 置き換えた公開内容、置き換えた下書き | 公開内容を書き換えた場合に持つ。`updated` | 行える |
| `migrated` | `applyMigration` | 値が変わった場合に持つ | なし。値が変わった内容の `migrated` を進める | なし | 公開内容が変わった場合に持つ。`updated`、`unpublished` | 行える |
| `purged` | なし。`ContentRepository.deleteMany` | 持たない | なし | なし | 公開中だった対象がある場合に持つ。`unpublished` | 行える |

- `LastChangeOperation` と `PublicationOperation` は、表の列が「持つ」（条件つきを含む）の行から成る。内容を作る操作と、内容を履歴の版にする操作の型は、内容の種類（下書き、公開内容）ごとに分かれ、表の列がその種類の内容を挙げる行から成る。下書きを作らない操作を `produced` に持つ下書きと、公開内容を版にしない操作を `retired` に持つ公開内容だった版は、型で表せない。`ChangeKindOf`（「PublicationLedger」）は、`PublicationOperation` の行ごとに、記録が持てる変更の種類の列の値から成る。ほかの組の記録は、型で表せない
- 公開の変更の記録の `operation` は、利用者の一つの操作の種類
- 現在の公開内容と下書きは、履歴の版ではない。公開で公開内容になった下書きは、版にしない。移行で値が変わった内容は、公開内容・下書き・履歴の版のどれも、`migrated` で、移行で変換されたことと、変換した日時と、変換後のスキーマの版を示す

非推奨のモデルのコンテンツに行える操作は、表の「非推奨のモデル」の列だけが定める。

```ts
const DeprecatedModel: {
  allows(operation: ContentOperation, model: ModelDefinition): boolean;
  ensure(operation: ContentOperation, model: ModelDefinition): void;
};
```

- `allows` は、`model.deprecated` が偽なら真。真なら、表の列が「行える」の操作で真、「行えない」の操作で偽を返す
- `ensure` は、`allows` が偽なら `BusinessRuleError`（`ModelDeprecated`）を投げる。非推奨のモデルで行えない操作のエラーは、この一つ
- 確認を求める読み取りは、`allows` で、行えない対象を出力に示す
- 非推奨の項目・選択肢・種類のブロックの規則は、モデルの規則と別で、`DeprecatedUse` が定める

### Revision

履歴の版。操作が置き換えた、またはなくした内容を一つ持つ。

```ts
type RevisionBase = Readonly<{
  id: RevisionId;
  contentId: ContentId;
  values: ContentValues;
  references: readonly ContentReference[];
  updatedAt: Date;
  migrated: MigrationMark | null;
}>;

type Revision = RevisionBase &
  (
    | Readonly<{
        origin: "published";
        produced: Provenance<PublishedProducingOperation>;
        retired: Retirement<PublishedRetiringOperation>;
      }>
    | Readonly<{
        origin: "draft";
        produced: Provenance<DraftProducingOperation>;
        retired: Retirement<DraftRetiringOperation>;
      }>
  );

type RetiredBody = Readonly<{ contentId: ContentId }> &
  (
    | Readonly<{ origin: "published"; body: PublishedBody; retired: Retirement<PublishedRetiringOperation> }>
    | Readonly<{ origin: "draft"; body: DraftBody; retired: Retirement<DraftRetiringOperation> }>
  );

type RevisionTarget = Readonly<{ contentId: ContentId; revisionId: RevisionId }>;
```

`RevisionTarget` は、保存済みの版の ID と、その版の `contentId` の組。版の持ち主（「メディアの使用箇所」の `BodyHolder`）を、版の全体を読まずに求めるのに使う。`RevisionRepository` の読み取りが返すか、読んだ `Revision` の `id` と `contentId` から作る。利用者の入力からは作らない。

| フィールド | 制約 |
| --- | --- |
| `id` | 必須。形式は [common.md](common.md) の「Clock と IdGenerator」が定める |
| `contentId` | その内容を持っていたコンテンツ。生成後に変わらない |
| `origin` | `published` は公開内容だった版、`draft` は下書きだった版。`produced` と `retired` の操作の種類は、`origin` ごとの型で決まる（「操作の種類」） |
| `values`、`references` | 現行スキーマの構造に従う。スキーマの反映の確定で変換する |
| `updatedAt`、`produced` | 版にした内容から引き継ぐ。その内容を作った操作・実行者・日時を表す |
| `migrated` | 版にした内容から引き継ぐ。スキーマの反映が、版の値を変換した場合は、その日時と変換後のスキーマの版にする。`null` は、変換されていない版 |
| `retired` | 版にした操作・実行者・日時 |

#### 振る舞い

```ts
const Revision: {
  record(id: RevisionId, retired: RetiredBody): Revision;
  applyMigration(revision: Revision, migrated: MigratedRevisionBody, mark: MigrationMark): Revision;
  belongsTo(revision: Revision, contentId: ContentId): boolean;
};
```

| 振る舞い | 処理 |
| --- | --- |
| `record` | `retired` の内容から版を作る。`origin` と `retired` をそのまま持ち、`body` の `values`、`references`、`updatedAt`、`produced`、`migrated` を引き継ぐ |
| `applyMigration` | `values` と `references` を、変換結果で置き換える。`values` が変わった場合は、`migrated` を `mark` にする。`origin`、`updatedAt`、`produced`、`retired` は変えない |
| `belongsTo` | `revision.contentId` が `contentId` と一致するかを返す。コンテンツを指定して版を読む操作は、偽の版を、無くなった版と同じく、存在しない版として扱う |

#### 不変条件

- 版は、スキーマの反映の確定による変換を除いて変わらない
- 版は、型を満たさない入力エラーと制約違反を持ってよい。違反は、復元した後の下書きの `issues` が示す
- 版の内容は、移行の前の保存内容を再現しない
- 操作の監査記録は、操作履歴（Audit）が持つ

#### ライフサイクル

`record` で作る。削除は次のいずれか。

- 明示的な削除。対象の版を指定し、すべての対象が存在する場合に限り、一度に削除する（`RevisionRepository.deleteMany`）。履歴の版の削除と、完全削除またはメディアの削除の対象に加えた版の削除が当たる
- 履歴の版の整理。保持方針の期限を過ぎた版を削除する（「ContentRetentionPolicy」）
- コンテンツの完全削除と、モデルの削除。削除する版は、`ContentRepository.deleteMany` と `ContentStagingArea.promote` の契約が定める

版の削除は、公開内容、下書き、ほかの版を変えない。版の削除は、参照先の保護に妨げられない。

### PublicationLedger

ワークスペースに一つ。公開の変更の記録に割り当てる通し番号の台帳で、最後に割り当てた通し番号を持つ。公開の変更の記録（`PublicationRecord`）は、台帳の値に含まれない、追記だけの記録で、`PublicationLedgerRepository` が保存する。記録の型と作り方はこの節が、追記・通し番号・参照元の規則は `PublicationLedgerRepository` の契約が定める。操作の ID と、ドメインをまたぐ分担は [index.md](index.md) の「公開内容の整合性と通し番号」が定める。

```ts
type PublicationLedger = Readonly<{ sequence: number }> & { readonly [publicationLedgerBrand]: true };

type ChangeKindOf = Readonly<{
  published: "published" | "updated";
  unpublished: "unpublished";
  trashed: "unpublished";
  purged: "unpublished";
  rewritten: "updated";
  migrated: "updated" | "unpublished";
}>;

type PublicationRecordDraft = {
  [O in PublicationOperation]: Readonly<{
    operationId: OperationId;
    operation: O;
    actor: Actor;
    changes: readonly NoticeOf<ChangeKindOf[O]>[];
    schemaVersion: number;
  }>;
}[PublicationOperation];

type PublicationRecord = {
  [O in PublicationOperation]: Readonly<{
    sequence: number;
    operationId: OperationId;
    operation: O;
    actor: Actor;
    changeList: PublishedChangeList & Readonly<{ changes: readonly NoticeOf<ChangeKindOf[O]>[] }>;
    schemaVersion: number;
  }>;
}[PublicationOperation];

const PublicationLedger: {
  create(): PublicationLedger;
  record<O extends PublicationOperation>(
    params: Readonly<{
      operationId: OperationId;
      operation: O;
      actor: Actor;
      changes: readonly NoticeOf<ChangeKindOf[O]>[];
      schemaVersion: number;
    }>,
    occurredAt: Date,
  ): WithEventDrafts<PublicationRecordDraft, ContentPublicationChangedEvent>;
};
```

| フィールド | 制約 |
| --- | --- |
| `PublicationLedger.sequence` | 最後に割り当てた通し番号。`create` が作る値は 0。進め方は `PublicationLedgerRepository` の「通し番号」が定める。割り当てた番号は、記録の `sequence` で読む |
| `PublicationRecord.sequence` | その操作の通し番号。1 以上の整数。ドラフトは持たない |
| `operationId` | 操作の ID。Content のユースケースでは、入力に受けた ID。スキーマの反映の確定では、確定のユースケースが作った ID（[schema.md](schema.md) の `NewIds`） |
| `operation` | 操作の種類。「操作の種類」の `PublicationOperation` |
| `actor` | 操作の実行者 |
| `changes` | その操作の変更の一覧。コンテンツID の昇順に並ぶ。ほかの条件は [common.md](common.md) の「PublishedChangeNotice」が定める |
| `changeList` | その操作の変更の一覧と参照元の一覧（[common.md](common.md) の「PublishedChangeNotice」）。`changes` は、ドラフトの `changes`。`referrers` は、ポートが求める |
| `schemaVersion` | 確定の時点の現行スキーマの、スキーマの版。スキーマの反映の確定では、反映後のスキーマの版 |

- `PublicationLedger` の値は、`create` だけが作る。通し番号は 0
- `record` は、一つの操作の記録のドラフトと、`content.publication_changed` のドラフトを一つ（`ContentEvents.publicationChanged` の結果）返す。`changes` は、その操作が返した `PublicationChange` ごとの `PublicationChange.toNotice` の値
- `PublicationChange` を一つも返さなかった操作は、公開内容が変わらないので、記録もイベントも作らない
- 記録は、追記の後に変わらない。ワークスペースの利用終了まで残り、「利用終了の消去」が消す
- 記録のドラフトを作るのは `record` だけ。スキーマの反映の確定の記録の追記は、`PublicationLedgerRepository.appendStaged` の契約が定める
- 復旧の後の通し番号は、[webhook.md](webhook.md) の「通し番号と復旧」が定める

### ContentRetentionPolicy

ワークスペースに一つ。履歴の版とごみ箱のコンテンツの保持方針。

```ts
type ContentRetentionPolicy = Readonly<{
  revisionDays: RetentionDays;
  trashDays: RetentionDays;
  version: number;
  updatedAt: Date;
}>;

type ContentRetentionPolicyInput = Readonly<{ revisionDays: number; trashDays: number }>;
type RetentionIssue = Readonly<{ field: "revisionDays" | "trashDays"; min: number; max: number }>;

const ContentRetentionPolicy: {
  standard: Readonly<{ revisionDays: number; trashDays: number }>;
  validate(input: ContentRetentionPolicyInput): readonly RetentionIssue[];
  create(input: ContentRetentionPolicyInput | null, now: Date): ContentRetentionPolicy;
  change(policy: ContentRetentionPolicy, next: ContentRetentionPolicyInput, now: Date): ContentRetentionPolicy;
  revisionCutoff(policy: ContentRetentionPolicy, now: Date): Date;
  trashCutoff(policy: ContentRetentionPolicy, now: Date): Date;
  purgeDueAt(policy: ContentRetentionPolicy, trashedAt: Date): Date;
};
```

| フィールド | 制約 |
| --- | --- |
| `revisionDays` | 履歴の版を保持する日数。標準の値は `standard.revisionDays` |
| `trashDays` | ごみ箱のコンテンツを保持する日数。標準の値は `standard.trashDays` |
| `version` | 楽観ロックの番号。作成時は 0 |
| `updatedAt` | 日数を最後に決めた日時。`create` と、日数を変えた `change` が、`now` にする |

- `validate` は、保存を伴わない純粋な検証。`RetentionDays.create` が `ValidationError` を投げる日数ごとに、項目と指定できる範囲（`RetentionDays.min` と `RetentionDays.max`）を返す。日数の有効な値は `RetentionDays` だけが定める。空は不整合が無いことを表す。管理画面での変更と、初期構成の宣言の検証が、同じ関数を使う
- `standard` の値は、[../scenario/app-integration.md](../scenario/app-integration.md) の APP-12 の「履歴の保持方針」「ごみ箱の保持方針」の標準値
- `create` は、`input` が `null` なら `standard` の日数、値があればその日数で作る。`version` は 0
- `create` と `change` は、日数を `RetentionDays.create` で作る。`change` は日数を置き換え、`version` を 1 進める。二つの日数がどちらも現在と同じ場合は、`policy` をそのまま返す
- `revisionCutoff` は `now` から `revisionDays` 日前の時点、`trashCutoff` は `now` から `trashDays` 日前の時点を返す。一日は 24 時間
- `purgeDueAt` は、`trashedAt` から `trashDays` 日後の時点を返す。ごみ箱のコンテンツが、自動整理で完全削除される時期を表す
- 履歴の版は `retired.at` が `revisionCutoff` 以前になると、ごみ箱のコンテンツは `trashedAt` が `trashCutoff` 以前になると、保持方針の期限を過ぎ、自動整理の対象になる。期限を過ぎた対象の規則は、この一文だけが定める
- 変更した保持方針は、既存の履歴の版とごみ箱のコンテンツにも適用する

自動整理の対象は次のとおり。

- 履歴の版の整理は、履歴の版だけを削除する。公開内容と下書きを残す
- ごみ箱の完全削除は、対象のコンテンツを一件ずつ完全削除する
- 削除するのは、期限を過ぎた対象だけ。ほかのコンテンツと履歴の版は残す

### EditingPresence

あるコンテンツを、あるメンバーが編集していることの記録。ワークスペース、コンテンツ、メンバーの組で一つ。`EditingPresenceStore` は区画に結び付かないので、記録がワークスペースの ID を持つ。

```ts
type EditingPresence = Readonly<{
  workspaceId: WorkspaceId;
  contentId: ContentId;
  memberId: MemberId;
  memberName: string;
  since: Date;
  expiresAt: Date;
}>;

const EditingPresence: {
  ttlMs: 60_000;
  announce(
    current: EditingPresence | null,
    params: Readonly<{ workspaceId: WorkspaceId; contentId: ContentId; memberId: MemberId; memberName: string }>,
    now: Date,
  ): EditingPresence;
  isActive(presence: EditingPresence, now: Date): boolean;
  others(presences: readonly EditingPresence[], memberId: MemberId, now: Date): readonly EditingPresence[];
};
```

| 振る舞い | 処理 |
| --- | --- |
| `announce` | `expiresAt` を `now` から `ttlMs` 後にし、`memberName` を `params` の値にする。`current` が有効な場合は `since` を引き継ぎ、そうでない場合は `since` を `now` にする |
| `isActive` | `now` が `expiresAt` より前かを返す |
| `others` | 有効な記録のうち、`memberId` 以外のメンバーのものを、`since` の早い順、同じ値では `memberId` の昇順で返す |

- 編集しているメンバーの記録は、`ttlMs` より短い間隔の `announce` で保つ。繰り返しが止まった記録は、`expiresAt` で失効する
- 読み取った記録が有効かどうかの判定と、返す順序は、`isActive` と `others` だけが定める。失効した記録は、`others` が除くので、編集中のメンバーの確認に現れない
- 編集を終えたメンバーの記録は、削除する
- `memberName` は、知らせた時点のメンバーの名前。編集中のメンバーの確認は、この名前を示す
- 記録は、保存の可否と競合の検出に影響しない

## 値オブジェクト

値オブジェクトは、生成時に制約を確かめる。利用者の入力から作る値オブジェクト（`RichText`、`MediaReference`、`RetentionDays`、一覧の条件）の違反は、項目ごとの理由を持つ `ValidationError`。等価性は、断りが無い限り、すべてのフィールドの値の一致で決める。

### ContentValues

```ts
type ContentLeaf = Readonly<{ richText: RichText; media: MediaReference }>;
type ContentValue = FieldValue<ContentLeaf>;
type ContentValues = FieldValues<ContentLeaf>;
```

Schema の `FieldValue` に、リッチテキストとメディアの値の形を与える。未設定の表し方は Schema の「値」に従う。本文を持たないリッチテキストは未設定で表す。

### MediaReference

メディアへの参照の一箇所。メディアの項目の値と、本文中の画像が持つ。Media の `MediaUsageRepository` に渡す使用の型（`MediaUse`。[media.md](media.md)）と別の型。`MediaUse` は `MediaUsages` が求める。

```ts
type MediaReference = Readonly<{
  mediaId: MediaId;
  alt: string | null;
  caption: string | null;
}>;

const MediaReference: {
  maxLength: 2000;
  create(input: Readonly<{ mediaId: MediaId; alt: string | null; caption: string | null }>): MediaReference;
};
```

- `alt` は代替テキスト、`caption` はキャプション。使用箇所ごとに持ち、メディアの側の説明と別に保つ
- `alt` と `caption` は、空文字列を `null` に正規化する。長さは `maxLength` 文字以下。この上限はスキーマの制約でない
- メディアの側の説明（共有する説明）の変更は、`MediaReference` を書き換えない。共有する説明を使用箇所へ写すことと取り込むことは、編集の入力を組み立てる側が、Media の読み取りで得た共有する説明で `alt` と `caption` を置き換え、下書きの保存がその値を保存する

### RichText

リッチテキストの本文。表示アプリが解釈して描画する、ブロックの並び。

```ts
type RichText = Readonly<{ blocks: readonly RichTextBlock[] }>;

type RichTextNestedBlock =
  | Readonly<{ type: "paragraph"; content: readonly RichTextInline[] }>
  | Readonly<{ type: "bulletList" | "orderedList"; items: readonly RichTextListItem[] }>;

type RichTextBlock =
  | RichTextNestedBlock
  | Readonly<{ type: "heading"; level: 1 | 2 | 3 | 4 | 5 | 6; content: readonly RichTextInline[] }>
  | Readonly<{ type: "blockquote"; blocks: readonly RichTextNestedBlock[] }>
  | Readonly<{ type: "image"; media: MediaReference }>;

type RichTextListItem = Readonly<{ blocks: readonly RichTextNestedBlock[] }>;

type RichTextInline = Readonly<{
  text: string;
  bold: boolean;
  italic: boolean;
  link: RichTextLink | null;
}>;

type RichTextLink =
  | Readonly<{ type: "external"; url: string }>
  | Readonly<{ type: "content"; contentId: ContentId }>;

const RichText: {
  limits: Readonly<{ externalUrlLength: 2048; listDepth: 6 }>;
  create(input: RichText): RichText | null;
  sanitize(source: RichTextSource): Readonly<{ value: RichText | null; changes: readonly RichTextChange[] }>;
  links(value: RichText): readonly ContentId[];
  images(value: RichText): readonly MediaReference[];
  withoutLinksTo(value: RichText, isRemoved: (id: ContentId) => boolean): RichText;
};

type RichTextSource = readonly RichTextSourceNode[];
type RichTextSourceNode = Readonly<{
  type: string;
  attrs: Readonly<Record<string, string | number | boolean | null>>;
  text: string | null;
  children: readonly RichTextSourceNode[];
}>;

type RichTextChange = Readonly<{ nodeType: string }> &
  (
    | Readonly<{ change: "removed"; reason: "unsupportedElement" | "disallowedPosition" }>
    | Readonly<{ change: "convertedToParagraph"; reason: "disallowedPosition" }>
    | Readonly<{ change: "linkRemoved"; reason: "unsafeUrl" }>
  );
```

扱う要素は、段落、見出し、太字、斜体、箇条書き、番号付きリスト、引用、リンク、画像。型に無い要素は表現できない。

| 対象 | 制約 |
| --- | --- |
| `RichTextInline.text` | 空でない。段落の中の改行は、改行文字で表す |
| `RichTextLink` の `external` | 絶対 URL。スキームは `http`、`https`、`mailto`、`tel` のいずれか。長さは `limits.externalUrlLength` 文字以下 |
| `RichTextLink` の `content` | 同じワークスペースのコンテンツを指す。コンテンツ参照として扱う |
| リストの項目の `blocks`、`blockquote` の `blocks` | 1 個以上 |
| リストの入れ子 | `limits.listDepth` 段まで |

リストの項目と引用の中に置けるブロックは、段落とリスト（`RichTextNestedBlock`）。見出し、引用、画像は、最上位のブロックにだけ置く。置けない位置のブロックは、型で表せない。

- `create` は制約を確かめる。テキストも画像も持たない本文では `null`（未設定）を返す
- `links` は、本文のリンクが指すコンテンツID を、重複を除いて返す。`images` は、本文中の画像を並びの順に返す
- `withoutLinksTo` は、`isRemoved` が真を返すコンテンツへのリンクを外し、テキストを残す

許可していない要素の扱いは `sanitize` が定める。編集の入力を組み立てる側は、貼り付けた内容を `RichTextSource` にして `sanitize` に渡し、`changes` を保存の前に示す。保存は `RichText` の形だけを受け付け、その形でない入力は、境界のデコードが `ValidationError` として拒否する。

| 入力 | 扱い | `change` と `reason` |
| --- | --- | --- |
| 扱う要素に無い種類のノード（任意の HTML、スクリプト、表を含む） | 中のノードとともに取り除く | `removed`、`unsupportedElement` |
| 置けない位置にある段落・見出し | 段落に変える | `convertedToParagraph`、`disallowedPosition` |
| 置けない位置にある、そのほかのブロック | 取り除く | `removed`、`disallowedPosition` |
| 制約を満たさない URL のリンク | リンクを外し、テキストを残す | `linkRemoved`、`unsafeUrl` |
| 空のテキスト | 取り除く。`changes` に加えない | なし |

- `changes` は、取り除いた、または変えたノードごとに一件。取り除いたノードの中のノードは、`changes` に加えない

### ContentReference

内容が持つ参照の一つ。

```ts
type ContentReference =
  | Readonly<{ kind: "content"; target: ContentId; strength: "required" | "optional" | "link"; path: ValuePath }>
  | Readonly<{ kind: "media"; target: MediaId; path: ValuePath }>;

const ContentReferences: {
  of(model: ModelDefinition, values: ContentValues): readonly ContentReference[];
  contentIds(references: readonly ContentReference[]): readonly ContentId[];
  mediaIds(references: readonly ContentReference[]): readonly MediaId[];
};
```

`of` は、入れ子を含むすべての項目の値から、次の参照を求める。同じ内容を持つ参照は一つにする。

| 値 | 参照 | `path` |
| --- | --- | --- |
| コンテンツ参照（一件） | `content`。`strength` は、項目の `required` が真なら `required`、偽なら `optional` | その項目 |
| コンテンツ参照（複数件） | 参照先ごとに `content`。`strength` は一件と同じ | その項目と、組の中の位置 |
| メディア | `media` | その項目 |
| リッチテキストのコンテンツへのリンク | `content`。`strength` は `link` | その項目 |
| リッチテキストの画像 | `media` | その項目 |

`strength` は、内容を書いた時点の現行スキーマの `required` から決まる。スキーマの反映の確定が、すべての内容の参照を求め直す。

`strength` が `required` または `link` の参照を、保護する参照と呼ぶ。どの `strength` が保護する参照に当たるかは、この一文だけが定める。ほかの規則とポートの契約は、「保護する参照」の語で指す。

`kind` が `content` の参照は、リポジトリが参照の記録にする。参照の記録とメディアの使用箇所の分担は、[index.md](index.md) の「参照の記録と保護」に従う。

### ContentIssue

```ts
type ContentIssue = Readonly<{
  path: ValuePath;
  issue:
    | Readonly<{ type: "typeError"; error: Extract<TypeErrorKind, "missingRequired" | "referenceMismatch"> }>
    | Readonly<{ type: "constraintViolation"; constraint: ConstraintKind }>;
}>;
type ConstraintIssue = ContentIssue & Readonly<{ issue: Readonly<{ type: "constraintViolation" }> }>;

type StructureViolation = Readonly<{
  path: ValuePath;
  reason: "undefinedField" | "kindMismatch" | "optionMismatch" | "unknownBlockType";
}>;
type ImageViolation = Readonly<{ path: ValuePath; reason: "imageRequired" }>;
type DeprecationViolation = Readonly<{ path: ValuePath; reason: "deprecated" }>;
type EditViolation = StructureViolation | ImageViolation | DeprecationViolation;
```

- `ContentIssue` は、保存した内容が持てる入力エラー。`typeError` は型を満たさない入力エラー、`constraintViolation` は制約違反
- 型を満たさない入力エラーは、必須項目の欠落（`missingRequired`）と、値の種類（`kindMismatch`）・選択肢（`optionMismatch`）・参照先（`referenceMismatch`）の不適合。このうち値の種類と選択肢の不適合は、保存の時点で構造の違反として拒否し、反映も変換できない値と取り除いた選択肢の値を未設定にするので、保存した内容には残らない。`ContentIssue` の `typeError` は、保存した内容に残り得る `missingRequired` と `referenceMismatch` だけを持つ
- `StructureViolation` は、構造の違反。値そのものの性質で、保存済みの `values` は持たない
- `EditViolation` は、編集の入力の保存を拒否する理由。構造の違反に、編集の入力にだけ生じる二つを加える。`imageRequired` は「ContentValidation」の `checkEdit` の本文の画像の違反、`deprecated` は `DeprecatedUse.newUses` の違反

### Provenance と Retirement

```ts
type Performer =
  | Readonly<{ actor: Extract<Actor, { kind: "member" }>; memberName: string }>
  | Readonly<{
      actor: Extract<Actor, { kind: "appToken" }>;
      memberName: string;
      token: Extract<TokenDesignation, { binding: "app" }>;
    }>
  | Readonly<{
      actor: Extract<Actor, { kind: "memberToken" }>;
      memberName: string;
      token: Extract<TokenDesignation, { binding: "member" }>;
    }>;

type Provenance<O extends DraftProducingOperation | PublishedProducingOperation> = Readonly<{
  operation: O;
  by: Performer;
  at: Date;
}>;

type Retirement<O extends DraftRetiringOperation | PublishedRetiringOperation> = Readonly<{
  operation: O;
  by: Performer;
  at: Date;
}>;
```

`Provenance` は内容を作った操作、`Retirement` は内容を履歴の版にした操作を表す。操作の種類は「操作の種類」が定め、型の引数が、内容の種類（下書き、公開内容）ごとの操作に絞る。

- `Performer` は、操作の実行者と、記録の時点の実行者の名称。`memberName` は、メンバーの名前（`member` ではそのメンバー、`appToken` では発行者、`memberToken` では紐付くメンバー）。`token` は、トークンを指す表示の元で、[common.md](common.md) の「TokenDesignation」の値をそのまま持つ。表示の文字列は presentation が決める。名称はユースケースが渡し、空でない
- 記録した名称は、後から変えない。メンバーの解除とトークンの失効の後も、履歴の一覧はこの名称を示す
- トークンによる操作の `actor` は、`appToken`（`TokenId` と発行者の `MemberId`）または `memberToken`（`TokenId` と紐付くメンバーの `MemberId`）

### InputSource

保存する入力の帰属。入力したメンバーと、保存する実行者が同じことを確かめるのに使う。

```ts
type InputSource =
  | Readonly<{ kind: "attributed"; enteredBy: MemberId }>
  | Readonly<{ kind: "direct" }>;

const InputSource: {
  isOwnedBy(source: InputSource, actor: Actor): boolean;
};
```

- `attributed` は、入力したメンバーが決まっている入力。`enteredBy` は、入力を始めたときのメンバー。入口が、保存の要求に添えて渡す
- `direct` は、要求そのものが入力である入力。入力したのは、要求の実行者
- `isOwnedBy` は、`direct` で真を返す。`attributed` では、`actor` が `member` で、その `memberId` が `enteredBy` と一致する場合に真を返す。セッションの期限切れの後に、入力したメンバーと別のメンバーがログインして保存し直した入力が、偽になる

### PublicationChange

```ts
type PublicationChange<K extends PublicationChangeKind = PublicationChangeKind> = Readonly<{
  contentId: ContentId;
  modelId: ModelId;
  change: K;
  uniqueValues: readonly UniqueValueChange[];
}>;

type NoticeOf<K extends PublicationChangeKind> = PublishedChangeNotice & Readonly<{ change: K }>;

const PublicationChange: {
  ofRemoval(content: PublishedContent): PublicationChange<"unpublished">;
  toNotice<K extends PublicationChangeKind>(change: PublicationChange<K>, model: ModelKey): NoticeOf<K>;
};
```

- `change` の値と意味と、`uniqueValues` の要素の型（`UniqueValueChange`）は、[common.md](common.md) の「PublishedChangeNotice」が定める
- `uniqueValues` は、前後の公開内容の、モデル直下のテキストの項目のうち、`unique` が真のものから求める
- `ofRemoval` は、公開中のコンテンツの公開内容がなくなる変更を返す。`uniqueValues` は、公開内容の一意な値を `before` に、`null` を `after` に持つ。`Content.unpublish`、完全削除、モデルの削除が使う
- `toNotice` は、公開の変更の記録に載せる値（[common.md](common.md) の `PublishedChangeNotice`）を、変更の種類を保って返す。`model` は、`change.modelId` のモデルのキーで、ユースケースが現行スキーマから求めて渡す

### PublishedContentFacts

公開中のコンテンツの、公開内容から求める事実。フォームの問い合わせ対象の、送信時の情報の元になる。

```ts
type PublishedContentFacts = Readonly<{
  contentId: ContentId;
  model: ModelKey;
  label: string | null;
  uniqueValues: readonly UniqueValue[];
}>;

const PublishedContentFacts: {
  of(ws: WorkspaceSchema, content: Content | null): PublishedContentFacts | null;
};
```

- `of` は、`content` が `null`（存在しない）、ごみ箱にある、公開中でない、またはモデルが現行スキーマに無い場合に `null` を返す
- `label` は、モデルの `inquiryLabelField` の項目の、公開内容の値。`inquiryLabelField` が `null`、または値が未設定の場合は `null`
- `uniqueValues` は、公開内容に対する `UniqueValues.of` の結果。スキーマの項目の順に並ぶ
- 下書きの値を使わない

### UniqueValue

```ts
type UniqueValue = Readonly<{ field: FieldKey; value: string }>;
type UniqueHolder = Readonly<{ modelId: ModelId; field: FieldKey; value: string; contentId: ContentId }>;

const UniqueValues: {
  of(model: ModelDefinition, values: ContentValues): readonly UniqueValue[];
};
```

`of` は、モデル直下の `unique` が真の項目のうち、値が設定されているものを返す。一意な値の一致は、「取得と検索の規則」の「文字列の比較」に従う。

`UniqueHolder` は、一意な値を公開内容に持つコンテンツと、そのモデル。重複は「Content」の「不変条件」が定める。

### ReferenceTarget

参照先の状態。検証と取得が、参照を解決できるかを決めるのに使う。`PublicationRules.resolve` が読む。

```ts
type ReferenceTarget = Readonly<{
  id: ContentId;
  modelId: ModelId;
  status: "published" | "unpublished" | "trashed";
  latestHasTypeErrors: boolean;
}>;
type ReferenceTargets = ReadonlyMap<ContentId, ReferenceTarget>;

const ReferenceTarget: {
  of(content: Content): ReferenceTarget;
};
```

- `status` は、通常の管理対象の公開中のコンテンツが `published`、通常の管理対象の未公開のコンテンツが `unpublished`、ごみ箱のコンテンツが `trashed`
- `latestHasTypeErrors` は、最新の内容が下書きで、`Content.hasTypeErrors` が真であること。プレビュー取得の参照の解決が使う
- `ReferenceTargets` に無いコンテンツID は、同じワークスペースに存在しないコンテンツを表す

### ModelContext

検証に使う、現行スキーマの一部。

```ts
type ModelContext = Readonly<{
  model: ModelDefinition;
  modelId: ModelId;
  modelIds: Readonly<Record<string, ModelId>>;
}>;

const ModelContext: {
  of(ws: WorkspaceSchema, modelId: ModelId): ModelContext | null;
};
```

`of` は、`modelId` が現行スキーマに無い場合 `null` を返す。

### RetentionDays

```ts
type RetentionDays = number & { readonly [retentionDaysBrand]: true };

const RetentionDays: {
  min: number;
  max: number;
  create(days: number): RetentionDays;
};
```

`min` 以上 `max` 以下の整数。範囲の外は `ValidationError`。`min` と `max` の値は、[../scenario/app-integration.md](../scenario/app-integration.md) の APP-12 の「履歴の保持方針」「ごみ箱の保持方針」の範囲。

### 一覧の条件

```ts
type FilterInput =
  | Readonly<{ type: "equals"; field: FieldKey; value: string }>
  | Readonly<{ type: "range"; field: FieldKey; min: string | null; max: string | null }>
  | Readonly<{ type: "includes"; field: FieldKey; option: OptionKey }>
  | Readonly<{ type: "references"; field: FieldKey; contentId: ContentId }>;

type ContentFilter =
  | Readonly<{
      type: "equals";
      field: FieldKey;
      value: Extract<LiteralValue, { kind: "text" | "number" | "boolean" | "date" | "datetime" | "select" }>;
    }>
  | Readonly<{ type: "range"; field: FieldKey; range: FilterRange }>
  | Readonly<{ type: "includes"; field: FieldKey; option: OptionKey }>
  | Readonly<{ type: "references"; field: FieldKey; contentId: ContentId }>;

type Bounds<T> = Readonly<{ min: T; max: T | null }> | Readonly<{ min: null; max: T }>;

type FilterRange =
  | Readonly<{ kind: "number"; bounds: Bounds<number> }>
  | Readonly<{ kind: "date"; bounds: Bounds<CalendarDate> }>
  | Readonly<{ kind: "datetime"; bounds: Bounds<Instant> }>;

type ContentSort = Readonly<{
  by: Readonly<{ type: "field"; field: FieldKey }> | Readonly<{ type: "createdAt" }> | Readonly<{ type: "updatedAt" }>;
  direction: "asc" | "desc";
}>;

type StatusFilter = Readonly<{
  publication: "published" | "unpublished" | null;
  hasDraft: boolean | null;
  draftHasIssues: boolean | null;
  publishedHasViolations: boolean | null;
}>;

type ContentSearch = Readonly<{ term: string; textFields: readonly FieldKey[] }>;

type ListScope =
  | Readonly<{ type: "public" }>
  | Readonly<{ type: "preview" }>
  | Readonly<{ type: "management"; status: StatusFilter; search: ContentSearch | null }>;

type ContentListCriteria = Readonly<{
  modelId: ModelId;
  scope: ListScope;
  filters: readonly FilterInput[];
  sort: ContentSort;
  pagination: Pagination;
}>;
type ContentListQuery = Readonly<{
  modelId: ModelId;
  scope: ListScope;
  filters: readonly ContentFilter[];
  sort: ContentSort;
  pagination: Pagination;
}> & { readonly [contentListQueryBrand]: true };

type CandidateCriteria = Readonly<{
  models: readonly Readonly<{ modelId: ModelId; textFields: readonly FieldKey[] }>[];
  term: string | null;
  location: "active" | "trashed";
  publication: "published" | "unpublished" | null;
  hasDraft: boolean | null;
  pagination: Pagination;
}>;
type CandidateQuery = CandidateCriteria & { readonly [candidateQueryBrand]: true };
```

条件の意味は「取得と検索の規則」が定める。

- `FilterInput` は、一覧を呼ぶ側が渡す条件。`equals` の `value` と、`range` の `min`・`max` は、値の表記のままの文字列で、値の種類を持たない。呼ぶ側は、現行スキーマを読まず、表記を値にしない
- `ContentFilter` は、`ContentDelivery.validateQuery` が、読んだスキーマの項目の値の種類で、表記を値にした条件。表記の読み方は「絞り込み」が定める
- `Bounds` は、少なくとも一方の境界を持つ範囲。境界を一つも持たない範囲は、型で表せない
- `ContentListQuery` と `CandidateQuery` は、`ContentDelivery.validateQuery` と `validateCandidates` だけが作るブランド型。ブランドは、条件が「取得と検索の規則」に合うことを表す
- `ContentSearch.textFields` と `CandidateCriteria` の `textFields` は、`ContentDelivery.searchFields` の結果
- Content のページを受ける一覧（メディアの使用箇所の一覧を含む）のページは、[common.md](common.md) の「Pagination と PaginationResult」に従って作る

### 一度に扱う対象

一つの操作が対象にするコンテンツと履歴の版の件数の上限と、一つの操作が新たに公開するメディアの数の上限。対象の列は、共通の `BatchTargets`（[common.md](common.md)）で作る。

```ts
const ContentBatch: {
  maxContents: number;
  maxRevisions: number;
  maxPublishedMedia: typeof MediaLimits.maxPublishedAtOnce;
};

type AddedRevisions =
  | Readonly<{ kind: "none" }>
  | Readonly<{ kind: "some"; revisionIds: BatchTargets<RevisionId> }>;

const AddedRevisions: {
  of(revisionIds: readonly RevisionId[] | null): AddedRevisions;
  ids(added: AddedRevisions): readonly RevisionId[];
};
```

| 対象 | 作り方 | 使う操作 |
| --- | --- | --- |
| コンテンツID の列 | `BatchTargets.create(ids, ContentBatch.maxContents)` | 確認を求める読み取り（公開、公開取り消し、ごみ箱への移動、完全削除） |
| コンテンツID と `version` の組の列 | `BatchTargets.createBy(items, ContentBatch.maxContents, key)`。`key` はコンテンツID | 公開、公開取り消し、ごみ箱への移動、完全削除、値の書き換え |
| 版の ID の列 | `BatchTargets.create(ids, ContentBatch.maxRevisions)` | 履歴の版の削除とその確認、メディアの削除の対象に加える履歴の版 |
| 完全削除の対象に加える履歴の版 | `AddedRevisions.of(ids)` | 完全削除と、その確認 |

- `maxContents` と `maxRevisions` の値は、[../scenario/app-integration.md](../scenario/app-integration.md) の APP-12 の「一つの操作が対象にできるコンテンツの件数」「一つの操作が対象にできる履歴の版の件数」
- 重複と件数の範囲の扱いは `BatchTargets` が定める
- `AddedRevisions` は、完全削除の対象に加える履歴の版。「指定なし」と、1 個以上の版の ID の列の直和。`of` は、`null`（指定なし）で `none` を、列で `BatchTargets.create(ids, ContentBatch.maxRevisions)` の結果を持つ `some` を返す。`ids` は、`none` で空を、`some` でその列を返す
- 版は、版の ID だけで指す。版が属するコンテンツは、読んだ `Revision` の `contentId` が示す
- コンテンツの上限と、対象に加える履歴の版の上限は、別々に数える
- `maxPublishedMedia` は、一つの操作（公開、公開内容の値の書き換え）が新たに公開するメディアの数の上限。値は、Media の `MediaLimits.maxPublishedAtOnce`（[media.md](media.md) の「MediaLimits」）と同じ。対象のコンテンツの件数と別に数える
- コンテンツと履歴の版の上限の値は、`maxContents` と `maxRevisions` だけが持つ。ドメインサービスとポートは、ID の集合（`readonly ContentId[]` など）を受け取る。ポートのメソッドが一度に受ける件数の上限は、そのメソッドの契約が、この定数を名前で指す

### Expansion

```ts
type ExpansionField = Extract<StructureRef, { target: "field" }>;
type Expansion = Readonly<{ fields: readonly ExpansionField[]; depth: number }>;
```

- `fields` は、展開するコンテンツ参照の項目。入れ子の中の項目も指せる。本文のリンクは展開しない
- `depth` は 0 以上 `ContentDelivery.limits.expansionDepth` 以下の整数。0 は展開しない

## ドメインサービス

どのサービスもポートに依存しない。保存データを要する判定は、ユースケースがリポジトリから読んだ結果を引数で受け取る。

### ContentValidation

内容を現行スキーマに照らして検証する。Schema の `ValueValidation` に、入れ子・件数・参照・メディア・リッチテキストの検証を加える。

```ts
const ContentValidation: {
  inspect(ctx: ModelContext, values: ContentValues, targets: ReferenceTargets): Inspection;
  checkEdit(
    ctx: ModelContext,
    before: ContentValues | null,
    after: ContentValues,
    targets: ReferenceTargets,
    media: MediaFactSet,
  ): EditCheck;
  checkRestore(ctx: ModelContext, revision: Revision, latest: ContentValues, targets: ReferenceTargets): CheckedDraft;
};

type Inspection = Readonly<{
  issues: readonly ContentIssue[];
  structure: readonly StructureViolation[];
  references: readonly ContentReference[];
}>;

type CheckedDraft = Readonly<{
  modelId: ModelId;
  values: ContentValues;
  issues: readonly ContentIssue[];
  references: readonly ContentReference[];
  uniqueValues: readonly UniqueValue[];
}> & { readonly [checkedDraftBrand]: true };

type EditCheck =
  | Readonly<{ ok: true; draft: CheckedDraft }>
  | Readonly<{ ok: false; violations: readonly EditViolation[] }>;
```

`inspect` は、モデルの直下から階層をたどり、階層ごとに `ValueValidation.validateLevel(ValueValidation.contentLevel(fields), values)` を呼ぶ。結果の位置は、たどった階層の `ValuePath` を前に付けて表す。加える検証は次のとおり。

| 対象 | 検証 | 結果 |
| --- | --- | --- |
| すべての階層 | `undefinedKeys` のキー | 構造の違反（`undefinedField`） |
| すべての階層 | `validateLevel` の `kindMismatch`、`optionMismatch` | 構造の違反（`kindMismatch`、`optionMismatch`） |
| すべての階層 | `validateLevel` の `missingRequired` と制約違反 | 入力エラー |
| 項目のまとまり | 中の階層を検証する。値が無い場合は、空の値の組として検証する | 中の項目の結果 |
| 繰り返し | 要素の数を `constraints` と比べる。要素ごとに中の階層を検証する | 制約違反（`minCount`、`maxCount`）と、中の項目の結果 |
| ブロック | ブロックの `type` が `blockTypes` に無い | 構造の違反（`unknownBlockType`） |
| ブロック | ブロックの数を `constraints` と比べる。ブロックごとに、その種類の階層を検証する | 制約違反（`minCount`、`maxCount`）と、中の項目の結果 |
| コンテンツ参照（複数件） | 参照先の数を `constraints` と比べる | 制約違反（`minCount`、`maxCount`） |
| コンテンツ参照 | 参照先が `targets` にあり、その `modelId` が、項目の `models` のどのモデルの ID でもない | 型を満たさない入力エラー（`referenceMismatch`） |
| メディア、リッチテキスト | 値の形は `MediaReference` と `RichText` の生成が確かめる。加える検証は無い | なし |

- `issues` に入れる型を満たさない入力エラーは、`ContentIssue` の `typeError` が持つもの。`validateLevel` が返す `kindMismatch` と `optionMismatch` は、`issues` に入れず `structure` に入れる。語の定義と、保存した内容に残らない理由は「ContentIssue」が定める
- 制約違反は、文字数・範囲・書式・選択数と、繰り返し・ブロック・参照先の件数
- `targets` に無い参照先は、入力エラーにしない。参照切れの扱いは「参照の解決」が、公開の可否は「公開内容の検証」が定める
- リッチテキストと、メディアの説明（`alt`、`caption`）は、スキーマの制約を持たない。説明の長さの上限は `MediaReference.maxLength`
- 一意性と、参照先の公開状態は、`PublicationRules` が確かめる
- 非推奨の指定は、`inspect` の結果を変えない

`checkEdit` と `checkRestore` が返す `CheckedDraft` の `modelId` は `ctx.modelId`。

`checkEdit` は、作成と下書きの保存の入力を検証する。`inspect` の `structure` が空で、本文の画像と非推奨の規則を満たす場合に限り、`CheckedDraft` を返す。満たさない場合は、`inspect` の `structure` と、本文の画像の違反（`imageRequired`）と、非推奨の違反（`deprecated`）を、`EditViolation` としてすべて返す。`before` は編集前の最新の内容の値で、作成では `null`。

- `media` は、`after` が使うメディアの事実（Media の `MediaFactSet`。[media.md](media.md)）。本文の画像は、`kind` が `image` のメディアだけを使える。`kind` が `image` でないメディアを使う本文の画像は `imageRequired`。`media` に無いメディアは、ここで確かめない。画像だけを使える場所の判定は、この規則だけが定める
- 非推奨の違反（`deprecated`）は、`DeprecatedUse.newUses(ctx.model.fields, before, after)` の結果
- 必須項目の欠落、参照先の不適合、制約違反は、`CheckedDraft.issues` に入れて保存する

`checkRestore` は、履歴の版を下書きとして復元する入力を検証する。`latest` は、復元するコンテンツの最新の内容の値。

- 非推奨の部分は、`DeprecatedUse.carry(ctx.model.fields, latest, revision.values)` で、`latest` のものを保つ
- 参照は、必須の参照・任意の参照・本文のリンクのどれも、版の値のまま復元する。`targets` に無いコンテンツを指す参照も、未設定にしない。復元した下書きの参照切れになる（「参照の解決」）
- 以上を行った値に `inspect` を行い、入力エラーを `CheckedDraft.issues` に入れて返す

履歴の版の値は、現行スキーマの構造に従う（スキーマの反映の確定が変換する）ので、`checkRestore` は、常に `CheckedDraft` を返す。

### DeprecatedUse

内容の値のうち、非推奨の部分の扱いを定める。非推奨の対象を新たに使わない規則（C6.4-04）を、この節だけが定める。`ContentValidation.checkEdit`、`ContentValidation.checkRestore`、`Content.duplicateValues`、`ContentDefaults.initial` が使う。

```ts
const DeprecatedUse: {
  newUses(fields: readonly ContentField[], base: ContentValues | null, proposed: ContentValues): readonly DeprecationViolation[];
  carry(fields: readonly ContentField[], base: ContentValues | null, proposed: ContentValues): ContentValues;
};
```

- 非推奨の部分は、非推奨の項目（項目のまとまりを含む）の値、非推奨でない選択の項目が選ぶ非推奨の選択肢、非推奨の種類のブロック。入れ子の中も含み、非推奨の部分の中の値は、その部分に含める。`fields` は、`base` と `proposed` の階層の項目
- 位置は `ValuePath` で表す。`newUses` は、繰り返しの要素とブロックの並びの位置を除いた `ValuePath` で位置を比べる。`carry` は、並びの位置を含む `ValuePath` で比べる
- `newUses` は、`proposed` の非推奨の部分のうち、`base` の同じ位置に同じ値で存在する数を超える分を返す。取り除くことは違反にしない。`base` が `null` なら、`proposed` の非推奨の部分のすべてを返す
- `carry` は、`proposed` の非推奨の部分を、`base` の同じ位置のものに置き換えた値を返す。非推奨の項目は、`base` の同じ位置の値（無ければ未設定）。非推奨でない選択の項目は、`proposed` の非推奨でない選択肢に、`base` の同じ位置の非推奨の選択肢を加える。単一選択では、`base` の値が非推奨の選択肢ならその値、そうでなければ `proposed` の値が非推奨でない場合にその値、ほかは未設定。ブロックの並びは、位置ごとに、`base` のその位置のブロックが非推奨の種類ならそのブロック、そうでなければ `proposed` のその位置のブロックのうち非推奨の種類でないもの。`base` が `null` なら、非推奨の部分を取り除く
- `carry` の結果を `proposed` にした `newUses` は、空を返す
- 保存済みの値は、非推奨の部分を持てる

### ContentDefaults

新規作成と、繰り返しの要素・ブロックの追加で入れる初期値を求める。編集の入力を組み立てる側が呼ぶ。

```ts
const ContentDefaults: {
  initial(fields: readonly ContentField[], at: Readonly<{ date: CalendarDate; instant: Instant }>): ContentValues;
};
```

- 項目の `defaultValue` を値にする。日付・日時の `createdAt` は、`at.date` と `at.instant` を入れる。`at.date` は、編集するメンバーの手元の暦日
- 項目のまとまりは、中の項目の初期値を入れる。繰り返しとブロックは、要素を持たない
- 非推奨の部分は、`DeprecatedUse.carry` に `base` を `null` で渡して取り除く
- `fields` は、初期値を求める階層の項目。新規作成ではモデル直下の項目、繰り返しの要素の追加ではその要素の項目、ブロックの追加では選んだ種類の項目
- 保存は、欠落した値を初期値で補わない。トークンによる作成は、指定した値をそのまま検証する

### PublicationRules

公開内容の検証と、公開、公開取り消し、完全削除の可否を、対象の全体について、事前に確かめる。利用者に違反の内容と解消方法を示すために使う。公開内容の整合性そのものは、確定の時点でポートが担保する（[index.md](index.md) の「公開内容の整合性と通し番号」）。

```ts
const PublicationRules: {
  checkBodies(input: Readonly<{
    schema: WorkspaceSchema;
    candidates: readonly PublishedCandidate[];
    targets: ReferenceTargets;
    holders: readonly UniqueHolder[];
    newlyPublishedMedia: NewlyPublishedMedia;
  }>): PublishedCheck;
  checkPublish(input: Readonly<{
    schema: WorkspaceSchema;
    candidates: readonly ActiveContent[];
    targets: ReferenceTargets;
    holders: readonly UniqueHolder[];
    newlyPublishedMedia: NewlyPublishedMedia;
  }>): PublishDecision;
  checkWithdraw(targets: readonly ContentId[], inbound: readonly InboundReference[]): RemovalDecision;
  checkPurge(
    targets: readonly ContentId[],
    inbound: readonly InboundReference[],
    removedRevisions: readonly RevisionId[],
  ): RemovalDecision;
  resolve(mode: "public" | "preview", target: ReferenceTarget | null): ReferenceResolution;
};

type PublishedCandidate = Readonly<{
  content: ContentId;
  modelId: ModelId;
  values: ContentValues;
  fields: readonly FieldKey[];
}>;

type PublishedBodyState = Readonly<{
  values: ContentValues;
  violations: readonly ConstraintIssue[];
  references: readonly ContentReference[];
  uniqueValues: readonly UniqueValue[];
}>;
type CheckedPublishedBody = PublishedBodyState & { readonly [checkedPublishedBrand]: true };

type PublishedViolation =
  | Readonly<{ type: "inputError"; content: ContentId; issues: readonly ContentIssue[] }>
  | Readonly<{
      type: "referenceNotPublished";
      content: ContentId;
      path: ValuePath;
      target: ContentId;
      cause: "missing" | "trashed" | "unpublished";
    }>
  | Readonly<{ type: "uniqueConflict"; content: ContentId; field: FieldKey; value: string; others: readonly ContentId[] }>
  | Readonly<{ type: "mediaLimitExceeded"; count: number; limit: number }>;

type PublishedCheck =
  | Readonly<{ ok: true; bodies: ReadonlyMap<ContentId, CheckedPublishedBody> }>
  | Readonly<{ ok: false; violations: readonly PublishedViolation[] }>;

type PublishableContent = DraftedContent & { readonly [publishableBrand]: true };

type PublishDecision =
  | Readonly<{ allowed: true; publishable: readonly PublishableContent[] }>
  | Readonly<{ allowed: false; violations: readonly PublishViolation[] }>;

type PublishViolation =
  | Readonly<{ type: "noDraft"; content: ContentId }>
  | PublishedViolation;

type RemovalDecision =
  | Readonly<{ allowed: true }>
  | Readonly<{ allowed: false; blocking: readonly InboundReference[] }>;

type ReferenceResolution =
  | Readonly<{ resolved: true }>
  | Readonly<{ resolved: false; cause: "missing" | "trashed" | "unpublished" | "typeErrors" }>;
```

#### 参照の解決

`resolve` は、コンテンツへの参照を解決できるかを、参照先の状態と `mode` から返す。

| `target` | `public` | `preview` |
| --- | --- | --- |
| `null` | `missing` | `missing` |
| `status` が `trashed` | `trashed` | `trashed` |
| `status` が `unpublished` | `unpublished` | `latestHasTypeErrors` が真なら `typeErrors`。偽なら解決できる |
| `status` が `published` | 解決できる | `latestHasTypeErrors` が真なら `typeErrors`。偽なら解決できる |

表は上の行から当てる。`public` は、`latestHasTypeErrors` を見ない。公開内容は、型を満たさない入力エラーを持たない。

公開内容の検証、取得、参照の状態の確認は、この規則を使う。参照先のモデルが項目の許可するモデルであることは、`ContentValidation.inspect` だけが確かめる（`referenceMismatch`）。

#### 公開内容の検証

`checkBodies` は、同じ操作で公開内容にする値の全体を検証する。公開内容にする値を検証する規則は、ここだけが定める。公開と、公開内容の値の書き換えが、検証する項目の集合を変えて使う。

- `candidates` は、公開内容にする値と、検証する項目の集合（`fields`。モデル直下の項目のキー）。渡す項目は、呼ぶ側（「公開時の検証」、「RewriteRules」）が定める
- 値を `ContentValidation.inspect` で検証する。`path` の先頭が `fields` の項目である入力エラーが一つでもあれば `inputError`。型を満たさない入力エラーと制約違反のどちらも妨げる
- `fields` の項目のコンテンツ参照と本文のリンクは、すべて `public` で解決できること。`candidates` に含むコンテンツは、公開中として扱う。自己参照と、対象間の参照を許容する。対象の外の参照先は、`targets` の現在の状態で確かめる。解決できない参照は、必須・任意・本文のリンクのどれも `referenceNotPublished`
- `fields` の項目の一意な値が、`holders` のうち `candidates` に含まないコンテンツの値、または `candidates` のほかの対象の値と重複する場合は `uniqueConflict`。重複は「Content」の「不変条件」が定める。`holders` は、対象の一意な値を公開内容に持つコンテンツ
- `fields` に含まない項目に残る制約違反は、妨げない。結果の `violations` に残す
- `newlyPublishedMedia` は、その操作が新たに公開するメディア（「新たに公開するメディア」）。`ids` の数が `ContentBatch.maxPublishedMedia` を超える場合は `mediaLimitExceeded`（`count` はその数、`limit` は上限）。対象ごとの違反でなく、操作の全体の違反
- 違反が一つも無い場合に限り、すべての対象の `CheckedPublishedBody` を返す。違反はすべて返す

`PublishedBodyState` は、検証結果を伴う公開内容の値の形。`CheckedRewrite` と `MigratedBodies` の公開内容が、同じ形を使う。

#### 公開時の検証

`checkPublish` は、同じ操作で公開する `candidates` の全体を検証する。違反が一つも無い場合に限り、すべての対象を `PublishableContent` として返す。違反はすべて返す。

- 対象が下書きを持たない場合は `noDraft`
- 下書きの値を、モデルのすべての項目を `fields` にして、`newlyPublishedMedia` とともに `checkBodies` に渡す。その違反を、そのまま返す

`checkPublish` は、非推奨のモデルを違反に数えない。`Content.publish` は、`checkPublish` が返した `PublishableContent` だけを受ける。

#### 新たに公開するメディア

公開内容を作る操作（公開、公開内容の値の書き換え）が新たに公開するメディアと、その公開に必要な権限を求める。求め方は、この関数だけが定める。

```ts
type NewlyPublishedMedia = Readonly<{
  ids: readonly MediaId[];
  demands: readonly Readonly<{ permission: Permission; ids: readonly MediaId[] }>[];
}> & { readonly [newlyPublishedMediaBrand]: true };

const NewlyPublishedMedia: {
  of(media: readonly Media[]): NewlyPublishedMedia;
};
```

- `media` は、新しい公開内容（公開では対象の下書き、公開内容の値の書き換えでは書き換えた後の公開内容）の `ContentReferences.mediaIds` が指すメディア
- `ids` は、`media` のうち、`Media.publication`（[media.md](media.md)）が `needed` を真で返すメディアの ID。重複を持たない
- `demands` は、`Media.publication` が返す `permission` ごとに、その権限を必要とするメディアの ID。`ids` が空なら空。権限の値と条件は `Media.publication` が決め、この関数は持たない
- `of` の結果の使い道は、[../usecases/content.md](../usecases/content.md) の「公開内容の変更の確定」が定める
- スキーマの反映の確定は、[schema.md](schema.md) の「値」の `LiteralValue` に従い、メディアを公開しない

#### 公開取り消し・ごみ箱・完全削除の参照の条件

```ts
type ReferenceHolder =
  | Readonly<{ type: "published" }>
  | Readonly<{ type: "draft" }>
  | Readonly<{ type: "revision"; revisionId: RevisionId }>;

type InboundSource = Readonly<{ contentId: ContentId; modelId: ModelId }> &
  (
    | Readonly<{ location: "active"; holder: ReferenceHolder }>
    | Readonly<{ location: "trashed"; holder: Exclude<ReferenceHolder, Readonly<{ type: "published" }>> }>
  );

type InboundReference = Readonly<{
  target: ContentId;
  source: InboundSource;
  strength: "required" | "optional" | "link";
  path: ValuePath;
}>;
```

- `InboundSource` は、参照元の内容の持ち主と、そのコンテンツの置き場所。ごみ箱のコンテンツは未公開なので、置き場所が `trashed` の持ち主は、下書きか履歴の版

- `checkWithdraw` は、公開内容をなくす操作（公開取り消し、公開中のコンテンツのごみ箱への移動・完全削除）の対象を検証する。`inbound` のうち、`targets` の公開内容を同じ UnitOfWork でなくした場合に、`ContentRepository` の「確定の時点の整合性」が `PUBLISHED_REFERENCE_REQUIRED` で拒否する参照を、`blocking` に返す
- `checkPurge` は、完全削除の対象を検証する。`inbound` のうち、`targets` と `removedRevisions` を同じ UnitOfWork で削除した場合に、`ContentRepository.deleteMany` が `REFERENCED` で拒否する参照を、`blocking` に返す。残る公開内容からの参照は、対象が公開中の場合にだけ生じ、`checkWithdraw` が返す
- `removedRevisions` は、同じ操作で削除する、対象の外のコンテンツの履歴の版。利用者が、削除を妨げている版を一つずつ明示して加える。CMS は、版を自動で加えない。利用者が加えた版のうち、`contentId` が対象のコンテンツである版は、要求を拒否せず、`removedRevisions` に含めない。その版は、`ContentRepository.deleteMany` がコンテンツとともに削除する
- 未公開のコンテンツのごみ箱への移動は、参照の条件を持たない
- 参照を自動で解消しない。`blocking` は、参照元と、参照している内容の種類を示す

### RewriteRules

値の書き換えを、対象の全体について検証する。

```ts
type FieldPatch = Readonly<Record<string, ContentValue | null>>;

type RewriteTarget =
  | Readonly<{ type: "published"; published: FieldPatch }>
  | Readonly<{ type: "draft"; draft: FieldPatch }>
  | Readonly<{ type: "both"; published: FieldPatch; draft: FieldPatch }>;

type RewriteItem = Readonly<{ content: Content; target: RewriteTarget }>;

type RewrittenBodies =
  | Readonly<{ type: "published"; published: CheckedPublishedBody }>
  | Readonly<{ type: "draft"; draft: CheckedDraft }>
  | Readonly<{ type: "both"; published: CheckedPublishedBody; draft: CheckedDraft }>;

type CheckedRewrite = Readonly<{
  contentId: ContentId;
  bodies: RewrittenBodies;
}> & { readonly [checkedRewriteBrand]: true };

const RewriteRules: {
  check(input: Readonly<{
    schema: WorkspaceSchema;
    items: readonly RewriteItem[];
    targets: ReferenceTargets;
    media: MediaFactSet;
    holders: readonly UniqueHolder[];
    newlyPublishedMedia: NewlyPublishedMedia;
  }>): RewriteDecision;
};

type RewriteDecision =
  | Readonly<{ allowed: true; rewrites: readonly CheckedRewrite[] }>
  | Readonly<{ allowed: false; violations: readonly RewriteViolation[] }>;

type RewriteViolation =
  | Readonly<{ type: "noPublished" | "noDraft"; content: ContentId }>
  | Readonly<{
      type: "structure";
      content: ContentId;
      body: "published" | "draft";
      violations: readonly (StructureViolation | ImageViolation)[];
    }>
  | PublishedViolation;
```

- 一つの操作で書き換えられるコンテンツの件数は、ユースケースが `BatchTargets`（「一度に扱う対象」）で確かめる
- 書き換える項目は、モデル直下の項目のキーで指定する。`FieldPatch` のキーは `FieldKey`、値は、その項目の入れ子を含む値の全体。`null` は未設定にする。指定しなかった項目の値は変えない
- 書き換え先は、公開内容、下書き、またはその両方。`both` は、`published` と `draft` に、別の値を指定できる。`CheckedRewrite` の `bodies` は、`RewriteTarget` と同じ変種を持ち、書き換え先を一つ以上持つ
- 公開内容を持たないコンテンツの公開内容の書き換えは `noPublished`、下書きを持たないコンテンツの下書きの書き換えは `noDraft`。書き換えは、下書きを公開せず、新たな下書きを作らない
- ごみ箱のコンテンツは、下書きを書き換えられる
- 書き換えた後の値を `ContentValidation.inspect` で検証する。構造の違反は、どちらの書き換え先でも `structure`。現行スキーマに無い項目のキーは `undefinedField`。書き換える項目の本文の画像は、`checkEdit` と同じ規則で `media` に照らし、`ImageViolation`（`imageRequired`）として同じ `structure` に入れる
- `DeprecatedUse` を使わない。`structure` の違反は、`StructureViolation` と `ImageViolation` だけ。非推奨の項目の値も書き換えられる。非推奨のモデルのコンテンツを書き換えられることは、「操作の種類」の表が定める
- 下書きは、入力エラーを残して書き換えられる
- 公開内容は、書き換えた後の値を、書き換える項目を `fields` にして、`newlyPublishedMedia` とともに `PublicationRules.checkBodies` に渡す。その違反を、そのまま返す
- 違反が一つも無い場合に限り、すべての対象を `CheckedRewrite` として返す。一件でも違反があれば、どの対象も書き換えない
- 指定した値がすべて現在の値と等しい対象は、違反にしない

競合の検出は「競合の検出」に従い、対象ごとに、読み取ったときの `version` を確かめる。

### ContentPermissions

対象の状態、書き換え先、新たに公開するメディアによって加わる権限を求める。

```ts
type PermissionDemand = Readonly<{
  permission: Permission;
  subject:
    | Readonly<{ kind: "contents"; ids: readonly ContentId[] }>
    | Readonly<{ kind: "media"; ids: readonly MediaId[] }>;
}>;

type PermissionSubject =
  | Readonly<{ operation: "published"; media: NewlyPublishedMedia }>
  | Readonly<{ operation: "trashed" | "purged"; targets: readonly Content[] }>
  | Readonly<{ operation: "rewritten"; items: readonly RewriteItem[]; media: NewlyPublishedMedia }>;

const ContentPermissions: {
  additional(subject: PermissionSubject): readonly PermissionDemand[];
};
```

| 操作 | 条件 | 加わる権限 | `subject` |
| --- | --- | --- | --- |
| `trashed`、`purged` | 公開中の対象がある | `content:publish` | `contents`。公開中の対象 |
| `rewritten` | 書き換え先が `published` または `both` の対象がある | `content:publish` | `contents`。その対象 |
| `rewritten` | 書き換え先が `draft` または `both` の対象がある | `content:edit` | `contents`。その対象 |
| `published`、`rewritten` | `media.demands` が空でない | `media.demands` の `permission` | `media`。その `ids` |

- `media` は、`NewlyPublishedMedia.of` の結果（「新たに公開するメディア」）。この関数は、その `demands` を `PermissionDemand` にして返し、メディアの権限の値と条件を自分で持たない
- 結果は、上の三行では、行ごとに、条件に当たるものがある場合に一件。最後の行では、`demands` の要素ごとに一件。`subject` の `ids` は 1 個以上
- どの対象にも必要な権限は、各ユースケースの認可が定める。この関数は、加わる権限だけを返す
- 結果の `permission` のうち実行者が持たないものは、不足する権限と、その `subject` を添えた `ForbiddenError`（`PERMISSION_MISSING`）。確認を求める読み取りは、エラーにせず、加わる権限と不足を出力で返す。結果を実行者の権限の組に照らす手続きは、[../usecases/content.md](../usecases/content.md) の「実行者と認可」が定める

### MediaUsages

内容の変更から、内容の持ち主ごとのメディアの使用箇所を求める。結果は、Media の `MediaUsageRepository` に渡す型（`HolderUses`。`holderId` と `MediaUse` の列。[media.md](media.md)）。渡し方は [../usecases/content.md](../usecases/content.md) の「コンテンツの書き込み」に従う。

```ts
type BodyWrite = Readonly<{
  saved: readonly Content[];
  recorded: readonly Revision[];
  removedContents: readonly ContentId[];
  removedRevisions: readonly RevisionTarget[];
}>;

const MediaUsages: {
  ofWrite(write: BodyWrite): readonly HolderUses[];
  ofStagedContents(items: readonly StagedContent[]): readonly HolderUses[];
  ofStagedRevisions(items: readonly StagedRevision[]): readonly HolderUses[];
};
```

| 入力 | 求める持ち主と `uses` |
| --- | --- |
| `saved` のコンテンツ | 公開内容と下書きの二つの持ち主。持つ内容は、その `references` の `ContentReferences.mediaIds` のメディア。持たない内容は、空の `uses` |
| `recorded` の版 | その版の持ち主。`references` の `ContentReferences.mediaIds` のメディア |
| `removedContents` のコンテンツ | 公開内容と下書きの二つの持ち主。空の `uses` |
| `removedRevisions` の版 | その版の持ち主。空の `uses` |
| `StagedContent` | `result` が `Content` の変種は `saved` と同じ、`null` の変種は `removedContents` と同じ |
| `StagedRevision` | `result` が `null` でなければ `recorded` と同じ、`null` なら `removedRevisions` と同じ |

- `holderId` は、`BodyHolder.toHolderId` で作る（「メディアの使用箇所」）
- `MediaUse` の `public` は、持ち主が公開内容の場合に真、下書きと履歴の版の場合に偽
- `saved` は、その UnitOfWork が書くコンテンツ。`recorded` は、その UnitOfWork が追加する版。`removedContents` は、その UnitOfWork が削除するコンテンツ。`removedRevisions` は、その UnitOfWork が削除する版のすべてで、コンテンツとともに削除される版を含む
- 結果は、同じ持ち主を一度だけ含む

### ContentDiff

二つの内容の差分を、項目ごとに求める。

```ts
type FieldDiff = Readonly<{ path: ValuePath; change: "added" | "removed" | "changed" }>;

const ContentDiff: {
  compare(before: ContentValues, after: ContentValues): readonly FieldDiff[];
};
```

- 項目のまとまりは、中の項目ごとに比べる
- 繰り返しとブロックは、並びの同じ位置どうしを比べる。位置の片方にだけある要素は `added` または `removed`。同じ位置で種類が異なるブロックは、その位置の `changed`
- そのほかの値は、値の全体を比べる。リッチテキストとメディアは、一つの値として比べる
- 履歴の版の比較、版と最新の内容・公開内容の比較、下書きと公開内容の比較、保存の競合で示す差分に使う

### ContentDelivery

取得と検索の規則を定め、取得結果を組み立てる。

```ts
const ContentDelivery: {
  limits: Readonly<{
    filters: number;
    expansionDepth: number;
    referencedContents: number;
    uniqueMatches: number;
    searchTermLength: 200;
  }>;
  searchFields(model: ModelDefinition): readonly FieldKey[];
  validateQuery(model: ModelDefinition, criteria: ContentListCriteria): QueryCheck<ContentListQuery>;
  validateLookup(model: ModelDefinition, field: FieldKey): readonly QueryIssue[];
  validateExpansion(schema: Schema, expansion: Expansion): readonly QueryIssue[];
  validateCandidates(criteria: CandidateCriteria): QueryCheck<CandidateQuery>;
  selectUnique(unique: UniqueValue, matches: readonly Content[]): Readonly<{ ok: true; content: Content | null }> | Readonly<{ ok: false; failure: DeliveryFailure }>;
  referencedIds(input: Readonly<{
    mode: "public" | "preview";
    expansion: Expansion;
    level: number;
    contents: readonly Content[];
  }>): Readonly<{ load: readonly ContentId[]; expand: readonly ContentId[] }>;
  expands(expansion: Expansion, field: ExpansionField, level: number): boolean;
  arrange<M extends "public" | "preview">(
    result: Extract<DeliveryResult<M>, { outcome: "delivered" }>,
    expansion: Expansion,
  ): ArrangedDelivery<M>;
  assemble<M extends "public" | "preview">(input: Readonly<{
    mode: M;
    schema: WorkspaceSchema;
    roots: readonly Content[];
    loaded: ReadonlyMap<ContentId, Content>;
    expansion: Expansion;
  }>): DeliveryResult<M>;
};

type QueryCheck<Q> =
  | Readonly<{ ok: true; query: Q }>
  | Readonly<{ ok: false; issues: readonly QueryIssue[] }>;

type QueryIssue =
  | Readonly<{
      subject: "filter";
      index: number;
      field: FieldKey;
      reason: "unknownField" | "unsupportedField" | "valueKindMismatch" | "unknownOption" | "invalidRange";
    }>
  | Readonly<{ subject: "filters"; reason: "tooManyFilters"; limit: number }>
  | Readonly<{ subject: "sort"; field: FieldKey; reason: "unknownField" | "unsupportedField" }>
  | Readonly<{ subject: "search"; reason: "invalidTerm" }>
  | Readonly<{ subject: "lookup"; field: FieldKey; reason: "unknownField" | "unsupportedField" }>
  | Readonly<{ subject: "expansion"; index: number; reason: "unknownField" | "unsupportedField" }>
  | Readonly<{ subject: "depth"; reason: "depthExceeded"; limit: number }>;

type PublishedDelivery = Readonly<{
  id: ContentId;
  modelId: ModelId;
  createdAt: Date;
  updatedAt: Date;
  values: ContentValues;
}>;
type PreviewDelivery = PublishedDelivery &
  Readonly<{ source: "published" | "draft"; hasConstraintViolations: boolean }>;
type DeliveredContent<M extends "public" | "preview"> = M extends "preview" ? PreviewDelivery : PublishedDelivery;

type ReferenceSummary = Readonly<{ id: ContentId; modelId: ModelId; uniqueValues: readonly UniqueValue[] }>;

type DeliveryResult<M extends "public" | "preview"> =
  | Readonly<{
      outcome: "delivered";
      items: readonly DeliveredContent<M>[];
      references: ReadonlyMap<ContentId, ReferenceSummary>;
      expanded: ReadonlyMap<ContentId, DeliveredContent<M>>;
    }>
  | Readonly<{ outcome: "rejected"; failure: DeliveryFailure }>
  | (M extends "public" ? Readonly<{ outcome: "stale" }> : never);

type ArrangedReference<M extends "public" | "preview"> =
  | Readonly<{ form: "expanded"; content: ArrangedContent<M> }>
  | Readonly<{ form: "summary"; summary: ReferenceSummary }>;
type ArrangedContent<M extends "public" | "preview"> = DeliveredContent<M> &
  Readonly<{ placements: readonly Readonly<{ path: ValuePath; reference: ArrangedReference<M> }>[] }>;
type ArrangedDelivery<M extends "public" | "preview"> = Readonly<{
  items: readonly ArrangedContent<M>[];
  references: ReadonlyMap<ContentId, ReferenceSummary>;
}>;

type DeliveryFailure =
  | Readonly<{ reason: "typeErrors"; content: ContentId; issues: readonly ContentIssue[] }>
  | Readonly<{
      reason: "referenceUnavailable";
      source: ContentId;
      path: ValuePath;
      target: ContentId;
      cause: "typeErrors" | "trashed" | "missing";
    }>
  | Readonly<{ reason: "ambiguousUniqueValue"; unique: UniqueValue; matches: readonly ContentId[] }>
  | Readonly<{ reason: "referenceLimitExceeded"; limit: number }>;
```

規則は「取得と検索の規則」が定める。

- `limits` の `filters`、`expansionDepth`、`referencedContents`、`uniqueMatches` の値は、[../scenario/app-integration.md](../scenario/app-integration.md) の APP-12 の「絞り込みの条件の数」「展開する参照の深さ」「1 回の取得で読む参照先の件数」「一意な値による取得で複数のコンテンツが該当したときに示すコンテンツの件数」
- `searchFields` は、モデルの `listFields` のうちテキストの項目を返す。管理の検索と候補の検索が、検索語を照らす項目
- `validateQuery` と `validateCandidates` は、不整合が無い条件を、リポジトリに渡せる型（`ContentListQuery`、`CandidateQuery`）にして返す。不整合がある条件は、不整合をすべて返す。`validateQuery` は、`filters` の表記を、項目の値の種類で値にする（「絞り込み」）
- `QueryIssue` は、不整合の対象（`subject`）ごとに、その対象で生じる理由だけを持つ。`filter` と `expansion` の `index` は、指定の位置（0 始まり）。`filters` は条件の数、`depth` は展開の深さで、項目を持たず、上限の値（`limit`）を持つ。`limit` は、`filters` では `limits.filters`、`depth` では `limits.expansionDepth`
- `DeliveryResult` の `stale` は、公開取得が、複数回の読み取りの間に変わった公開内容を読んだことを表す（「参照先」）。プレビュー取得の結果は、この変種を持たない
- 返す内容の種類（`source`）と制約違反の有無（`hasConstraintViolations`）は、プレビュー取得の結果だけが持つ
- `arrange` は、取得結果の各内容のコンテンツ参照の項目の、参照先ごとの位置（`placements`）に、その位置の形（展開した内容か要約か。「参照先の展開」）を置いた結果を返す。展開した内容も、同じ形で位置を持つ。本文のリンクの参照先の要約は、`references` が持つ

### ContentMigration

スキーマの反映の `ApplyPlan` を、コンテンツと履歴の版へ適用する。

```ts
type MigrationInput = Readonly<{
  plan: ApplyPlan;
  modelIds: Readonly<Record<string, ModelId>>;
  targets: ReferenceTargets;
  by: Performer;
  mark: MigrationMark;
}>;

type MigratedBodies = Readonly<{ contentId: ContentId }> &
  (
    | Readonly<{ type: "published"; published: PublishedBodyState }>
    | Readonly<{ type: "draft"; draft: CheckedDraft }>
    | Readonly<{ type: "both"; published: PublishedBodyState; draft: CheckedDraft }>
  ) & { readonly [migratedBodiesBrand]: true };
type MigratedRevisionBody = Readonly<{ values: ContentValues; references: readonly ContentReference[] }> & {
  readonly [migratedRevisionBrand]: true;
};

type PublicationDetail<K extends PublicationChangeKind> = Readonly<{
  change: K;
  uniqueValues: readonly UniqueValueChange[];
}>;

type KeptContentOutcome = Readonly<{
  outcome: "kept";
  model: ModelKey;
  publication: PublicationDetail<"updated"> | null;
  blockers: readonly ApplyBlocker<ApplyBlockerDetail>[];
  losses: readonly ValueLoss[];
  constraintViolations: readonly ValuePath[];
  draftHasIssues: boolean;
}>;
type RemovedContentOutcome = Readonly<{
  outcome: "removed";
  model: ModelKey;
  publication: PublicationDetail<"unpublished"> | null;
  removedBy: OperationRef;
}>;
type ContentMigrationReport = Readonly<{ contentId: ContentId }> & (KeptContentOutcome | RemovedContentOutcome);

type StagedContent =
  | Readonly<{ result: Content; report: KeptContentOutcome }>
  | Readonly<{ result: null; contentId: ContentId; report: RemovedContentOutcome }>;
type StagedRevision =
  | Readonly<{ result: Revision; losses: readonly ValueLoss[] }>
  | Readonly<{ result: null; target: RevisionTarget; losses: readonly ValueLoss[] }>;

type RevisionMigrationReport = RevisionTarget & Readonly<{ losses: readonly ValueLoss[] }>;

type UniqueDuplicate = Readonly<{ modelId: ModelId; field: FieldKey; value: string; contents: readonly ContentId[] }>;
type UniqueDuplicates = Sampled<UniqueDuplicate> & Readonly<{ contentCount: number }>;

const ContentMigration: {
  migrateContent(content: Content, input: MigrationInput): StagedContent;
  migrateRevision(revision: Revision, modelId: ModelId, input: MigrationInput): StagedRevision;
  uniqueFields(plan: ApplyPlan, modelIds: Readonly<Record<string, ModelId>>): readonly Readonly<{ modelId: ModelId; field: FieldKey }>[];
  noticeOf(report: ContentMigrationReport): NoticeOf<"updated" | "unpublished"> | null;
  accumulate(tally: ConversionTally, report: ContentMigrationReport | RevisionMigrationReport): ConversionTally;
  withDuplicates(tally: ConversionTally, plan: ApplyPlan, modelIds: Readonly<Record<string, ModelId>>, duplicates: UniqueDuplicates): ConversionTally;
};
```

`input.plan` は、反映または事前確認の計画（Schema の `ApplyPlan`。`source` で判別する `MigrationsPlan` と `DirectPlan` の直和）。`ContentMigration` は、どちらの変種にも共通の `nextSchema`、`units`、`modelFates` だけを使い、`source` で分岐しない。反映と事前確認を区別しない。`input.modelIds` は反映前のモデルのキーから `ModelId` への対応、`input.by` は変換の実行者、`input.mark` は変換した内容に付ける印。`input.targets` は、対象の内容と版が参照するコンテンツの状態。Schema のどの値を渡すかは、[../usecases/schema.md](../usecases/schema.md) の「ほかのドメインとの対応」に従う。

`migrateContent` は、一件のコンテンツの公開内容と下書きを変換する。ごみ箱のコンテンツの下書きも同じ規則で変換する。変換結果（`StagedContent`、`StagedRevision`）は、対象の ID を一度だけ持つ。`result` がある変種は `result` の ID、`null` の変種は `contentId` と `target` が対象を指す。`migrateContent` の手順は次のとおり。

1. `content.modelId` から、反映前のモデルのキーと、`plan.modelFates` でたどった反映後のモデルのキーを求める。報告の `model` は反映後のキー。`blockers` の表し方は「ApplyBlocker」が定める。削除するモデルのコンテンツは、反映後のキーが無いので、報告の `model` は反映前のキー
2. 公開内容と下書きのそれぞれに `ValueMigration.apply` を行う。`context.removedBy` は、`targets` と `plan.modelFates` から作る
3. `outcome` が `removed` の場合、`result` は `null` で、報告は `removed` の変種。`removedBy` にその操作を入れる。削除するコンテンツの報告は、`blockers`、`losses`、制約違反、下書きの入力エラーを持たない。公開中だったコンテンツの `publication` は、`PublicationChange.ofRemoval` の結果の `change` と `uniqueValues`。未公開だったコンテンツでは `null`
4. `kept` の場合、報告は `kept` の変種で、以下の手順が埋める。リッチテキストの本文のリンクのうち、`removedBy` が `null` でないコンテンツを指すものを扱う。公開内容にある場合は、`blockers` に `referenceToRemovedContent` を加える。下書きにある場合は、`RichText.withoutLinksTo`（`isRemoved` は `removedBy` が `null` でないこと）でリンクを外し、外したリンクごとに、`operation` が `removedBy` の返した操作、`loss` が `contents`、`path` がすべての単位を終えた後のそのリッチテキストの値の位置の `ValueLoss` を `losses` に加える（[schema.md](schema.md) の「ValueMigration」の、取り除く参照の値と同じ形）
5. 変換後の値を、`plan.nextSchema` のモデルに対して `ContentValidation.inspect` で検証し、`references` を求め直す。参照先のモデルは、`plan.modelFates` でキーの変更をたどって照らす
6. 公開内容の `ValueMigration` の `blocking` を `unconvertibleValue`、型を満たさない入力エラー（`ContentIssue` の `typeError`）を `typeError` として `blockers` に加える。制約違反は `violations` と `constraintViolations` に入れ、`blockers` に加えない
7. 変換後の公開内容の、保護する参照のうち、`targets` の現在の状態で `PublicationRules.resolve`（`public`）が解決できないものを、`ApplyBlockerDetail` の `referenceNotPublished` として `blockers` に加える。型を満たさない入力エラーでなく、公開内容の整合性の違反なので、`typeError` と別の変種にする。任意の参照を必須にする反映で、参照先が公開中でない公開内容が当たる。保護する参照でない参照は加えない。同じ反映で削除するモデルのコンテンツへの参照は、手順 2 と 4 が扱う
8. 下書きの入力エラーは `issues` に入れる。`blocking` の値は未設定のまま使う。下書きは `blockers` を作らない。存在しないコンテンツを指す参照は、必須にした項目のものも、参照切れとして残す。`draftHasIssues` は、通常の管理対象のコンテンツで、変換後の下書きが入力エラーを持つかを表す。ごみ箱のコンテンツでは偽
9. `Content.applyMigration` を `input.mark` で行い、`result` にする。`MigratedBodies` は、コンテンツが持つ内容と同じ変種で作る（「Content」の「振る舞い」）。報告の `publication` は、結果の `publication` の `change` と `uniqueValues`。公開内容が変わらなかったコンテンツでは `null`

- 移行は、下書きを公開しない。公開内容と下書きの対応を入れ替えない
- 公開内容の検証は、反映後のスキーマに対してだけ行う。途中の単位を適用した時点では行わない
- 制約違反は、変換・補完・複写した値を含めて、反映を失敗させない
- `losses` は、公開内容と下書きの `ValueMigration` の `losses` と、手順 4 で加えた `ValueLoss` を合わせたもの

`noticeOf` は、報告から、公開の変更の記録に載せる値を作る。`contentId` は報告の `contentId`、`model` は報告の `model`、`change` と `uniqueValues` は報告の `publication`。`publication` が `null` の報告では `null`。

`migrateRevision` は、履歴の版を下書きと同じ規則で変換する。`modelId` は、版のコンテンツのモデル。`outcome` が `removed` の場合、`result` は `null` で、`target` に版の ID とコンテンツID を持つ。`kept` の場合、手順 4 の下書きと同じくリンクを外して `losses` に加え、`references` を求め直して、`Revision.applyMigration` を行う。版は検証せず、`blockers` を作らない。存在しないコンテンツを指す参照の扱いは、手順 8 と同じ。値の複写は、その版の複写元の値を写す。

一意性は、すべての変換結果を準備領域へ書いた後に確かめる。`uniqueFields` は、`plan.nextSchema` で `unique` が真の項目を、反映後も残るモデルの ID とともに返す。`withDuplicates` は、その項目の一つについての `ContentStagingArea.findDuplicates` の結果を受け、返った重複の組（`UniqueDuplicate`）の各コンテンツについて、`uniqueViolation` を一件ずつ `blockers` に加える。`content` はそのコンテンツ、`duplicates` は組のほかのコンテンツ、`model` は反映後のモデルのキー。`blockers` の `total` には、結果の `contentCount`（重複の組に属するコンテンツの全件数）を足す。移行定義を伴わない一意性の変更にも、同じ規則を適用する。

準備は、公開内容の整合性のうち、反映が変え得るもの（一意な値、保護する参照の参照先）を、手順 7 と `withDuplicates` で `blockers` として確かめ終える。確定の時点の確認は、`ContentStagingArea.promote` の契約が定める。

準備領域へ書く変換結果のメディアの使用箇所は、「メディアの使用箇所」が定める。Media の準備領域の識別子は、Schema のユースケースが定め（[../usecases/schema.md](../usecases/schema.md) の「準備領域の識別子」）、Content は関わらない。

`accumulate` は、報告を一件ずつ、Schema の `ConversionTally`（途中までの集計。[schema.md](schema.md)）に畳み込む。始まりの値は `ConversionTally.empty()`。報告は、コンテンツID の昇順、版の ID の昇順に渡す。

| `ConversionTally` | 内容 |
| --- | --- |
| `blockers` | `kept` の報告の `blockers` と、`withDuplicates` が加える重複 |
| `losses` | 操作ごとに畳み込む。`contents` は、`removedBy` がその操作である `removed` の報告のコンテンツID を集めた `Sampled<ContentId>`。`values` は、`kept` の報告と版の報告の `losses` の、その操作の値の数を足す。意味は、[schema.md](schema.md) の「反映の影響と失敗の理由」の `ApplyEffects.losses` が定める |
| `changedPublished` | `noticeOf` が `null` でないコンテンツ。`change` は `updated` と `unpublished` |
| `constraintViolations` | `kept` の報告の `constraintViolations`（位置の列）を、コンテンツごとに一件の `ConstraintViolation` にまとめて畳み込む。意味は、[schema.md](schema.md) の「反映の影響と失敗の理由」が定める |
| `draftsWithErrors` | `kept` の報告の `draftHasIssues` が真のコンテンツ |

`ConversionTally`、`OperationLoss`、`ValueLoss`、`ModelFate`、`Sampled`、`ApplyBlocker` は Schema の型をそのまま使う。`ApplyBlocker` の `detail` は「ApplyBlocker」の `ApplyBlockerDetail`。Content は、集計の型を自分で定めない。集計から `PreparationOutcome` を作るのは、Schema の `Preparation.outcome`。

#### ApplyBlocker

反映を失敗させる公開内容の理由。Schema の `ApplyBlocker`（[schema.md](schema.md) の「反映の影響と失敗の理由」）は、`content` に違反を持つ公開内容を、`detail` にこの型の値を持つ。理由の変種の直和は、この節だけが定める。

```ts
type ApplyBlockerDetail =
  | Readonly<{ type: "typeError"; model: ModelKey; path: ValuePath; error: Extract<TypeErrorKind, "missingRequired" | "referenceMismatch"> }>
  | Readonly<{ type: "referenceNotPublished"; model: ModelKey; path: ValuePath; target: ContentId }>
  | Readonly<{ type: "uniqueViolation"; model: ModelKey; field: FieldKey; duplicates: readonly ContentId[] }>
  | Readonly<{ type: "unconvertibleValue"; model: ModelKey; path: ValuePath; operation: OperationRef }>
  | Readonly<{ type: "referenceToRemovedContent"; model: ModelKey; path: ValuePath; target: ContentId }>;
```

- 違反を持つ公開内容（`ApplyBlocker` の `content`）は、反映後も残るモデルに属する
- `model` は反映後のスキーマ（`plan.nextSchema`）でのモデルのキー。`path` と `field` も反映後の項目のキーで表す。`path` は理由に当たる値の位置、`target` は参照先またはリンク先のコンテンツ、`duplicates` は同じ一意な値を持つほかの公開内容
- どの公開内容にどの変種を加えるかは、`migrateContent` の手順と `withDuplicates` が定める

## 取得と検索の規則

公開取得、プレビュー取得、管理の一覧は、同じ `ContentListQuery` と同じ規則を使う。

### 返す内容

| 取得 | 対象 | 返す内容 | `source` |
| --- | --- | --- | --- |
| 公開取得 | 通常の管理対象にある、公開中のコンテンツ | 公開内容 | 持たない |
| プレビュー取得 | 通常の管理対象にあるコンテンツ | 最新の内容 | 下書きがあれば `draft`、なければ `published` |

- `updatedAt` は、返す内容の更新日時。`createdAt` はコンテンツの作成日時
- `hasConstraintViolations` は、プレビュー取得が返す内容が制約違反を持つかを表す。プレビュー取得は、制約違反だけを持つ下書きを返す
- 公開取得は、下書きの内容と存在、未公開のコンテンツの存在、ごみ箱のコンテンツの存在を明かさない。公開取得の結果は、下書きの有無によって変わらない
- ごみ箱のコンテンツは、どちらの取得も返さない。存在しない対象と同じ結果になる
- 取得は、取得時点で保存済みの内容を返す

### 単件の取得

- コンテンツID による取得は、`ContentRepository.findById` の結果を使う。モデルが要求と異なるコンテンツは、存在しない対象として扱う
- 一意な値による取得は、`unique` が真の項目のキーと値で指定する。`ContentDelivery.validateLookup` が確かめる。モデル直下に無い項目は `QueryIssue`（`subject` は `lookup`、`unknownField`）、`unique` が偽の項目は `unsupportedField`。値の一致は「文字列の比較」に従う
- 公開取得の一意な値による取得は、公開内容の値で探す。該当は高々一件
- プレビュー取得の一意な値による取得は、最新の内容の値で探す。`ContentDelivery.selectUnique` は、`findByUniqueValue` の結果が 0 件なら `null`、1 件ならそのコンテンツ、2 件以上なら `DeliveryFailure`（`ambiguousUniqueValue`）を返す。失敗は該当するコンテンツID を示し、一件を選ばない。示す ID は、`findByUniqueValue` が返したもの
- 単一モデルのコンテンツは、対象を指定せずに取得する
- プレビュー取得で、最新の内容が型を満たさない入力エラーを持つ下書きの場合、内容を返さず、`DeliveryFailure`（`typeErrors`）で入力エラーの位置と理由を示す。公開内容で代替しない

### 文字列の比較

文字列の一致・部分一致・順序は、コードポイントの並びで決める。大文字と小文字を区別し、正規化と前後の空白の除去を行わない。保存先の照合順序に依存しない。一意な値の一致と重複、絞り込み、並び替え、検索語の一致に、この規則を適用する。文字列の比較の規則は、この節だけが定める。

### 絞り込み

条件は、返す内容の、モデル直下の項目の値に適用する。複数の条件は、すべてを満たすコンテンツに絞る。条件は `limits.filters` 個まで。超える指定は `QueryIssue`（`subject` は `filters`、`tooManyFilters`）。

| 条件 | 対象の値の種類 | 一致の規則 |
| --- | --- | --- |
| `equals` | テキスト、数値、真偽値、日付、日時、単一選択 | 値が等しい |
| `range` | 数値、日付、日時 | `min` 以上、`max` 以下。境界の値を含む。`null` の側は限らない |
| `includes` | 複数選択 | 選んだ選択肢に `option` を含む |
| `references` | コンテンツ参照（一件・複数件） | 参照先に `contentId` を含む |

`equals` の値と `range` の境界は、表記のままの文字列（`FilterInput`）で受ける。`ContentDelivery.validateQuery` が、読んだスキーマの、その項目の値の種類で、次の表記として読み、`ContentFilter` の値にする。

| 値の種類 | 表記 |
| --- | --- |
| テキスト | そのままの文字列 |
| 数値 | 10 進の表記 |
| 真偽値 | `true` または `false` |
| 日付 | `YYYY-MM-DD` |
| 日時 | UTC の時点を表す文字列 |
| 単一選択 | 選択肢のキー |

- 項目の値の種類の表記として読めない値は `QueryIssue`（`subject` は `filter`、`valueKindMismatch`）。この理由は、`validateQuery` だけが返す
- 単一選択の `equals` と `includes` の、その項目に無い選択肢のキーは `unknownOption`。モデル直下に無い項目は `unknownField`、その条件の対象の値の種類でない項目は `unsupportedField`
- 未設定の値は、どの条件にも一致しない
- テキストと単一選択（選択肢のキー）は、「文字列の比較」で比べる
- 数値は数として比べる。日付は、タイムゾーンを持たない暦日として比べる。日時は、UTC の時点として比べ、ミリ秒まで区別する
- `references` は、保存した値に適用する。参照先を解決できるかは、一致に影響しない
- `range` は、`min` と `max` の少なくとも一方を持ち、`min` が `max` を超えない。満たさない指定は `invalidRange`
- 入れ子の中の項目、参照先の項目、リッチテキスト、メディアは、条件にできない。任意の論理式と全文検索を扱わない

### 並び替え

- 並び替えの条件は一つ。モデル直下のテキスト、数値、日付、日時、単一選択の項目、作成日時、更新日時を、昇順または降順で指定する。指定しない場合は、作成日時の降順
- テキスト、単一選択、数値、日付、日時は、絞り込みと同じ基準で比べる
- 更新日時は、返す内容の更新日時
- 未設定の値は、昇順でも降順でも、設定された値の後ろに並べる
- 並び替えの値が同じコンテンツと、未設定どうしは、コンテンツID の昇順に並べる。同じ条件の取得は、内容が変わらない限り、同じ順序になる

### ページ分割と総件数

- 共通の `Pagination`（[common.md](common.md)）を使う。作り方は「一覧の条件」が定める
- 一覧は、条件に該当する総件数を、指定した位置の結果とともに常に返す
- 該当が無い場合と、位置が末尾を超えた場合は、空の一覧と総件数を返す
- 取得のたびに、その時点の内容に条件を適用する。続きを取得する間に内容が変わると、同じコンテンツが二度現れるか、現れないことがある

### プレビュー取得で返さない下書き

- プレビュー取得の一覧は、最新の内容が型を満たさない入力エラーを持つ下書きのコンテンツを、結果にも総件数にも含めない
- 一覧の結果は、含めなかった対象を `Sampled<ContentId>` で示す。対象は、そのモデルの通常の管理対象のコンテンツのうち、最新の内容が型を満たさない入力エラーを持つもののすべてで、絞り込みの条件によらない
- 含めなかった対象の入力エラーの位置は、そのコンテンツの単件の取得が示す

### 参照先

返す内容のコンテンツ参照と本文のリンクは、`PublicationRules.resolve` で解決する。公開取得は `public`、プレビュー取得は `preview` で解決する。

| 参照 | 解決できる | 解決できない |
| --- | --- | --- |
| 任意の参照 | 参照先を返す | `cause` が `typeErrors` の場合は、取得を失敗とする。そのほかは、値から取り除き、未設定として返す。複数件の参照は、その参照だけを組から取り除く |
| 保護する参照（公開取得） | 参照先を返す | `assemble` が `stale` を返し、取得は、読んだ内容を捨てて読み直す（「スキーマの反映との関係」の読み直しの規則） |
| 保護する参照（プレビュー取得） | 参照先を返す | 取得を失敗とする |

- 公開取得の保護する参照は、同じ時点の公開内容では、必ず解決できる（`ContentRepository` の「確定の時点の整合性」）。解決できないのは、一回の取得の複数回の読み取りの間に、参照元と参照先の公開内容が変わった場合だけ。読み直しは、変わった後の内容を読む
- プレビュー取得の失敗は、`DeliveryFailure`（`referenceUnavailable`。`cause` は `PublicationRules.resolve` の `typeErrors`、`trashed`、`missing`）で、参照元と参照先を示す
- 参照先の最新の内容が、型を満たさない入力エラーを持つ下書きの場合（`typeErrors`）は、必須・任意・本文のリンクによらず、参照先を展開するかどうかにもよらず、プレビュー取得を失敗とする。返す内容（取得の対象と、展開した参照先）が持つ参照に適用する
- 解決できる参照の参照先は、どれも、`references` に要約（モデル、コンテンツID、一意な値）を一件ずつ持つ。一意な値は、公開取得では参照先の公開内容、プレビュー取得では参照先の最新の内容から求める
- 参照先を再び公開すると、参照元を公開し直さなくても、次の取得から参照を返す
- プレビュー取得は、参照先にも最新の内容を使う

### 参照先の展開

- 展開は、`Expansion` の `fields` と `depth` で指定する。展開する集合は、取得の対象から、`fields` に含む項目の解決できる参照を、`depth` 段までたどって届くコンテンツ
- 展開する集合のコンテンツは、要約に加えて、`expanded` に、取得の対象と同じ形の内容を一件ずつ持つ。展開した内容が持つ参照の参照先も、同じ `references` と `expanded` に入れる
- 参照の位置ごとの形は、`expands(expansion, field, level)` が決める。`field` はその参照の項目、`level` は、取得の対象を 0 として、その参照を持つ内容までにたどった参照の数。`field` が `fields` に含まれ、`level` が `depth` より小さい参照は、展開した形（`expanded` の内容）で返す。ほかの参照は、要約で返す。同じコンテンツが、ある位置では展開した形に、別の位置では要約になる
- 参照が循環していても、たどる段数は `depth` で止まる
- `depth` の上限は `limits.expansionDepth`。超える指定は `QueryIssue`（`subject` は `depth`、`depthExceeded`）
- 一回の取得で読む参照先は、展開の有無によらず、重複を除いて `limits.referencedContents` 件まで。`assemble` は、`loaded` がこの件数を超える取得を、`DeliveryFailure`（`referenceLimitExceeded`）にする
- 一覧の取得は、返す対象のいずれかで `referenceUnavailable` が生じた場合、取得の全体を失敗とする
- `fields` が、コンテンツ参照でない項目、または現行スキーマに無い項目を指す場合は `QueryIssue`（`subject` は `expansion`。`unsupportedField`、`unknownField`）
- `referencedIds` は、段 `level` の内容として渡したコンテンツの、返す内容が参照するコンテンツID を `load` に、そのうち `expands(expansion, field, level)` が真の位置の参照先を `expand` に返す。`referencedIds` を段ごとに呼ぶ取得の進め方は、[../usecases/content.md](../usecases/content.md) の「取得」が定める

### 管理の検索と絞り込み

`ListScope` の `management` は、通常の管理対象のコンテンツを対象に、最新の内容に条件を適用する。型を満たさない入力エラーを持つ下書きも含める。

| 条件 | 一致の規則 |
| --- | --- |
| `status.publication` | `published` は公開内容を持つ、`unpublished` は持たない |
| `status.hasDraft` | 下書きを持つかどうか |
| `status.draftHasIssues` | 真は、下書きを持ち、その `issues` が空でない。偽は、それ以外 |
| `status.publishedHasViolations` | 真は、公開内容を持ち、その `violations` が空でない。偽は、それ以外 |
| `search` | コンテンツID が `term` と等しい、または `textFields` のいずれかの値が `term` を含む |
| `filters`、`sort` | 取得と同じ規則 |

- `null` の条件は絞り込まない。指定した条件は、すべてを満たすコンテンツに絞る
- `term` は、空でない、`limits.searchTermLength` 文字以下の文字列。満たさない `term` は `QueryIssue`（`subject` は `search`、`invalidTerm`）。`validateQuery` と `validateCandidates` が、同じ規則で確かめる
- 更新日時による並び替えは、最新の内容の更新日時を使う
- ごみ箱のコンテンツの一覧、参照先の候補、一括操作の対象の追加は、`CandidateQuery` で、複数のモデルをまたいで探す。`models` のいずれかに属し、`location` の置き場所にあり、`publication` と `hasDraft`（`null` は絞り込まない）を満たすコンテンツが対象。`term` が `null` でなければ、`search` と同じ規則で、そのモデルの `textFields` に照らして絞る。条件は最新の内容に適用し、最新の内容の更新日時の降順、同じ値ではコンテンツID の昇順に並べる
- `CandidateCriteria` の `models` の `ModelId` は、ユースケースがモデルのキーを現行スキーマで解決したもの

### スキーマの反映との関係

- 取得は、スキーマの反映が確定するまで、反映前の内容を反映前の構造で返す。準備領域の内容は、切り替えの後に返す
- 取得（公開取得、プレビュー取得、管理の読み取り）は、読み始めに `WorkspaceSchema` を読み、読み終わりにもう一度読んで、`schemaVersion` が同じことを確かめる。違う場合は、読んだ内容を捨てて、読み直す。一つの取得結果は、反映の前か後の、どちらか一方の構造だけを持つ
- 取得の読み直しは、この規則と、「参照先」の公開取得の `stale` の二つ。どちらの読み直しも、一回の取得で一度だけ行う。読み直した取得が、再び読み直しの条件に当たった場合は、取得を失敗とし、`ConflictError` を返す。理由は、スキーマの版が違う場合は `SCHEMA_CHANGED`、`stale` の場合は `STALE_VERSION`。要求した側が送り直す

## ドメインイベント

公開内容を変える操作が記録するイベント。記録する条件は、[index.md](index.md) の「公開内容の整合性と通し番号」と、`PublicationLedgerRepository` の「呼び方」が定める。

```ts
type PublicationChangedPayload = Readonly<{
  operationId: OperationId;
}>;

type ContentPublicationChangedEvent = DomainEventBase<"content.publication_changed", PublicationChangedPayload>;

type ContentEvent = ContentPublicationChangedEvent;

const ContentEvents: {
  publicationChanged(params: PublicationChangedPayload, occurredAt: Date): EventDraft<ContentPublicationChangedEvent>;
};
```

- ドラフトを作るのは `ContentEvents.publicationChanged` だけ。Content のユースケースでは、`PublicationLedger.record` が、記録のドラフトとともに返す（[common.md](common.md) の「ドメインイベント」の `WithEventDrafts`）
- ペイロードは、操作の ID だけを持つ。操作の種類、実行者、変更の一覧、参照元の一覧、スキーマの版、通し番号は、公開の変更の記録（`PublicationRecord`）が持つ
- イベントのワークスペースの ID と `operationId` の組が、一つのイベントを識別する。消費者は、この組で再配送を見分ける

## 参照の記録と保護

[index.md](index.md) の「参照の記録と保護」を、次のとおり実現する。

- 内容の `references` が、その内容の持つ参照のすべて。リポジトリは、内容の保存と同時に、`kind` が `content` の参照を、持ち主（コンテンツと、公開内容・下書き・履歴の版の別）とともに記録する。内容を置き換えた、または削除した保存は、前の内容の記録をなくす
- 公開内容の `uniqueValues` も、内容の保存と同時に記録する。公開内容の間の一意性は、この記録から決まる
- 完全削除からの参照先の保護と、公開内容の整合性は、この記録から、`ContentRepository` が確定の時点で担保する。拒否する確定と理由は、`ContentRepository` の `deleteMany` の契約と「確定の時点の整合性」だけが定める
- 存在しないコンテンツを指す参照は、参照切れとして扱う。その参照を持てる内容は「Content」の「不変条件」が、公開の可否は「公開内容の検証」が定める
- コンテンツへの参照の参照元は、`ContentReferenceQuery` で問い合わせる
- 準備領域の内容は、記録に数えない。切り替えで、参照の記録と一意な値の記録を、変換結果のものに置き換える

### メディアの使用箇所

`kind` が `media` の参照は、メディアの使用箇所。使用箇所の記録の置き場所と、Content のポートとの分担は、[index.md](index.md) の「参照の記録と保護」に従う。

```ts
type BodyHolder = Readonly<{ contentId: ContentId; holder: ReferenceHolder }>;

const BodyHolder: {
  toHolderId(holder: BodyHolder): string;
  parse(holderId: string): BodyHolder;
};
```

`BodyHolder` は、内容の持ち主。`ReferenceHolder` は「公開取り消し・ごみ箱・完全削除の参照の条件」が定める。`toHolderId` は、持ち主から `MediaUsageRepository` の `holderId` を作る。

| 持ち主 | `holderId` |
| --- | --- |
| コンテンツの公開内容 | `content:` + コンテンツID + `:published` |
| コンテンツの下書き | `content:` + コンテンツID + `:draft` |
| 履歴の版 | `revision:` + コンテンツID + `:` + 版の ID |

- `parse` は、`toHolderId` が作った文字列を、元の持ち主に戻す。ID は `:` を含まない（形式は `IdGenerator` が定める）。区画の使用箇所を書くのは Content のユースケースだけなので、保存済みの `holderId` は、どれもこの形。この形でない文字列は、デコードできない保存値として `SystemError`
- `holderId` は、ワークスペースの中で持ち主を一意に表す
- 準備領域の変換結果の `holderId` は、切り替えた後の持ち主のもの。準備領域の使用箇所の扱いは、[media.md](media.md) の「MediaUsageRepository」の契約が定める

書き込み。

- 内容（公開内容、下書き、履歴の版）を足す、置き換える、なくす書き込みの使用箇所は、`MediaUsages.ofWrite` が求める。呼び方は [../usecases/content.md](../usecases/content.md) の「コンテンツの書き込み」が定める
- 内容を変えない書き込み（ごみ箱からの復元）は、使用箇所を変えない
- スキーマの反映と事前確認の変換結果の使用箇所は、`MediaUsages.ofStagedContents` と `ofStagedRevisions` が求める。準備領域の使用箇所の書き込み・切り替え・破棄の呼び方は、[../usecases/schema.md](../usecases/schema.md) の「反映・事前確認を準備する（`prepareSchemaApply`）」「反映を確定する（`commitSchemaApply`）」「終了の書き込み」が定める
- 記録は、内容を保存した UnitOfWork の確定と、全部成功か全部反映されないかを共にする

読み取り。

- 使用箇所の持ち主は、Media の `MediaUsageRepository` が返す `holderId` を `BodyHolder.parse` に通して求める。一覧の一件は、持ち主一つで、ポートが返す使用箇所の一件と一対一に対応する。総件数は、ポートの総件数と同じ
- 使用箇所の位置（項目）は、その持ち主の内容の `references` のうち、`kind` が `media` で、そのメディアを指すものの `path`
- 使用の状況は、Media の `MediaUsageStatus`（[media.md](media.md)）をそのまま使う

## ポート

リポジトリのポートは `packages/core/src/domain/content/ports/` に置く。どのポートも、ドメイン型を受け渡す。

全リポジトリに共通する規則は [common.md](common.md) の「TransactionalRepository」に従う。以下の表は、これに加わる契約と、違う契約だけを書く。

- ポートが結び付く区画は、[index.md](index.md) の「集約とポートの置き場所」の表が定める。ワークスペースの区画に結び付くポートの規則は、[common.md](common.md) の「TransactionalRepository」の「区画」に従う。そのメソッドは、読み取りか書き込みで、ポートごとの「分類」が定める。書き込みは `UnitOfWorkProvider.run(scope, fn)` の文脈（`WorkspaceContext`）のリポジトリから、UnitOfWork の外の読み取りは `PartitionReader.workspace(workspaceId)` のリポジトリから呼ぶ
- 一回の読み取りの呼び出しは、準備領域の切り替えの前か後の、どちらか一方の内容だけを返す。この約束と、準備領域の容量の超過（`STAGING_CAPACITY_EXCEEDED`）は、アダプターごとの検証の対象で、ポート適合テストの対象にしない
- ページングは、共通の `Pagination` と `PaginationResult` を使う。ページの範囲は、[common.md](common.md) の「Pagination と PaginationResult」に従う
- 一回の呼び出しが受ける件数の上限のうち、`ContentBatch` と `ContentDelivery.limits` から導かないものは、`ContentPortLimits` が持つ。各メソッドの上限は、そのメソッドの契約が名前で指す

```ts
const ContentPortLimits: { chunk: 100; ids: 500; revisionTargets: 1000 };
```

一つの確定が、ポートの拒否の理由の複数に当たる場合、ポートは、当たる理由のどれか一つの `ConflictError` を返す。どの理由を返すかは約束しない。残る公開内容と下書きの両方から参照される対象の `deleteMany`（`PUBLISHED_REFERENCE_REQUIRED` と `REFERENCED`）が当たる。事前の確認（`PublicationRules`）が違反をすべて示す。

### ContentRepository

`Content` を保存し、取り出す。一件ずつ削除する呼び手が無いので、単件の `delete` を持たない。

```ts
type TrashCursor = Readonly<{ trashedAt: Date; id: ContentId }>;

interface ContentRepository extends Omit<TransactionalRepository<Content, ContentId>, "delete"> {
  findByIds(ids: readonly ContentId[]): Promise<readonly Versioned<Content>[]>;
  findSingle(modelId: ModelId): Promise<Versioned<ActiveContent> | null>;
  findPage(query: ContentListQuery): Promise<PaginationResult<Content>>;
  findPreviewExcluded(modelId: ModelId): Promise<Sampled<ContentId>>;
  findCandidates(query: CandidateQuery): Promise<PaginationResult<Content>>;
  findByUniqueValue(modelId: ModelId, unique: UniqueValue, view: "published" | "latest"): Promise<readonly Content[]>;
  findUniqueHolders(modelId: ModelId, values: readonly UniqueValue[]): Promise<readonly UniqueHolder[]>;
  saveMany(items: readonly Readonly<{ content: Content; expectedVersion: ExpectedVersion<Content> }>[]): Promise<void>;
  deleteMany(targets: readonly Readonly<{ id: ContentId; expectedVersion: ExpectedVersion<Content> }>[]): Promise<void>;
  findTrashedBefore(cutoff: Date, after: TrashCursor | null, limit: number): Promise<readonly Versioned<TrashedContent>[]>;
  count(): Promise<number>;
  lastId(): Promise<ContentId | null>;
  countByPublication(): Promise<Readonly<{ published: number; unpublished: number }>>;
  countTrashedBefore(cutoff: Date): Promise<number>;
  scan(after: ContentId | null, limit: number): Promise<readonly Content[]>;
  deleteAll(limit: number): Promise<number>;
}
```

分類: `saveMany`、`deleteMany`、`deleteAll` は書き込み。ほかの加えるメソッドは読み取り。

| メソッド | 契約 |
| --- | --- |
| `insert` | コンテンツID の一意性は、[common.md](common.md) の「TransactionalRepository」の `insert` の規則に従う（`ALREADY_EXISTS`）。完全削除したコンテンツの ID は、再び使える。公開内容と下書きのコンテンツへの参照と、公開内容の `uniqueValues` を記録する。下の「確定の時点の整合性」を適用する |
| `findById` | コンテンツID で一件を返す。無ければ `null`。ごみ箱のコンテンツも返す。`ExpectedVersion` を返す |
| `save` | 楽観ロックで保存する。コンテンツへの参照と `uniqueValues` の記録を、保存する内容のものに置き換える。下の「確定の時点の整合性」を適用する |
| `saveMany` | 複数のコンテンツを、一回の呼び出しで、`save` と同じ契約で保存する。全部を保存するか、何も保存しないかのどちらかになる。`items` は `ContentBatch.maxContents` 件まで。空の `items` は、何も変えずに成功する |
| `findByIds` | 指定した ID のうち、存在するコンテンツを返す。コンテンツID の昇順で返す。`ids` は `ContentPortLimits.ids` 個まで。`ExpectedVersion` を返す |
| `findSingle` | 指定した単一モデルの、通常の管理対象にあるコンテンツを返す。無ければ `null`。ごみ箱のコンテンツを返さない |
| `findPage` | `query` に該当するコンテンツと、該当する総件数（`count`）を返す。対象、条件を適用する内容、一致の規則、順序は、「取得と検索の規則」が定める（`scope` が `public` は「返す内容」の公開取得、`preview` は同じ節のプレビュー取得から「プレビュー取得で返さない下書き」を除いたもの、`management` は「管理の検索と絞り込み」） |
| `findPreviewExcluded` | 指定したモデルの、「プレビュー取得で返さない下書き」の対象の件数と、コンテンツID の昇順で先頭 `SchemaLimits.sampleLimit` 件（[schema.md](schema.md)）を返す |
| `findCandidates` | `query` に該当するコンテンツと、該当する総件数（`count`）を返す。対象、一致の規則、順序は、「取得と検索の規則」の「管理の検索と絞り込み」の `CandidateQuery` の規則が定める |
| `findByUniqueValue` | 指定したモデルの通常の管理対象のコンテンツのうち、`view` の内容の `unique.field` の値が `unique.value` と一致するものを、コンテンツID の昇順で `ContentDelivery.limits.uniqueMatches` 件まで返す。`published` は公開内容、`latest` は最新の内容で探す |
| `findUniqueHolders` | 指定したモデルの公開内容のうち、`values` のいずれかの項目と値が一致するものを、`UniqueHolder`（指定したモデルの `modelId`、項目、値、コンテンツID の組）で返す。`values` は `ContentPortLimits.ids` 個まで。項目と値が同じ要素は一つとして数える |
| `deleteMany` | 対象のコンテンツを、公開内容、下書き、そのコンテンツのすべての履歴の版、参照と一意な値の記録とともに削除する。対象の履歴の版の数によらず、一回の呼び出し。全対象を削除するか、何も削除しないかのどちらかになる。対象の `version` が `expectedVersion` と異なる、または対象が無い場合は `ConflictError`（理由 `STALE_VERSION`）。削除の後に残る下書き・履歴の版（ごみ箱のコンテンツのものを含む）から、保護する参照の記録で参照されている対象があれば `ConflictError`（理由 `REFERENCED`）。判定は、UnitOfWork の確定後の状態に対して行う。対象間の参照、対象自身の履歴の版からの参照、同じ UnitOfWork の `RevisionRepository.deleteMany` で削除した版からの参照は、妨げない。残る公開内容からの参照には、下の「確定の時点の整合性」を適用する。対象は `ContentBatch.maxContents` 件まで。空の `targets` は、何も変えずに成功する |
| `findTrashedBefore` | ごみ箱のコンテンツのうち、`trashedAt` が `cutoff` 以前のものを、`trashedAt` の昇順、同じ値ではコンテンツID の昇順で、`limit` 件まで返す。`after` が `null` でなければ、その順序で `after` より後ろのコンテンツから返す。`limit` は `ContentPortLimits.chunk` 以下。`ExpectedVersion` を返す |
| `count` | 区画のコンテンツの件数を、ごみ箱のものを含めて返す。スキーマの反映の準備が、変換の対象の件数に使う |
| `lastId` | 区画のコンテンツのうち、`scan` の順で最後のコンテンツの ID（最大のコンテンツID）を、ごみ箱のものを含めて返す。コンテンツが無ければ `null`。スキーマの反映の準備が、変換の対象の走査の終端に使う |
| `countByPublication` | 区画のコンテンツの件数を、公開中（公開内容を持つ）と未公開（持たない。ごみ箱のものを含む）に分けて返す |
| `countTrashedBefore` | ごみ箱のコンテンツのうち、`trashedAt` が `cutoff` 以前のものの件数を返す |
| `scan` | 区画のすべてのコンテンツを、ごみ箱のものを含めて、コンテンツID の昇順で返す。`after` より後ろの ID から `limit` 件まで。`limit` は `ContentPortLimits.chunk` 以下 |
| `deleteAll` | 区画のコンテンツを、参照と一意な値の記録とともに削除する。履歴の版は削除しない。確定の時点の整合性と楽観ロックを適用しない |

#### 確定の時点の整合性

`insert`、`save`、`saveMany`、`deleteMany` を呼んだ UnitOfWork の確定の時点で、確定後の状態が次に当たれば、UnitOfWork の全体を `ConflictError` として拒否する。どれもポートが担保し、並行した書き込みでも成り立つ。完全削除からの参照先の保護（`REFERENCED`）は、`deleteMany` の契約が定める。

| 理由 | 拒否する確定後の状態 |
| --- | --- |
| `SINGLE_EXISTS` | `cardinality` が `single` のコンテンツが、同じ `modelId` の通常の管理対象に二つ以上ある。ごみ箱のコンテンツは数えない |
| `UNIQUE_VALUE_TAKEN` | 一意な値が重複する（「Content」の「不変条件」） |
| `REFERENCE_NOT_PUBLISHED` | 公開内容が、保護する参照の記録で、通常の管理対象の公開中のコンテンツでないものを参照する。書いた公開内容の参照先が公開中でない場合に返す |
| `PUBLISHED_REFERENCE_REQUIRED` | 同じ状態を、参照先の側の書き込み（公開内容をなくす `save` と `saveMany`、`deleteMany`）が生じさせた場合に返す |

- 判定は、確定後の状態に対して行う。同じ UnitOfWork で公開する対象間の参照と自己参照、同じ UnitOfWork で公開内容をなくす対象間の参照は、違反にならない
- 公開内容の保護する参照でない参照と、下書きの参照は、上の表の拒否の対象にならない（「Content」の「不変条件」）
- 並行した二つの UnitOfWork が、合わせると違反になる書き込み（同じ一意な値の公開、参照元の公開と参照先の公開取り消し）を行った場合、どちらか一方だけが確定する
- 下書きまたは履歴の版が保護する参照で参照する保存と、その参照先の `deleteMany` が並行した場合、保存は拒否されない。削除は、確定の時点でその記録が確定済みなら `REFERENCED` になり、そうでなければ確定して、参照は参照切れになる
- 対象が別のコンテンツで、合わせても上の状態に当たらない二つの UnitOfWork は、どちらも確定する
- `modelId` が現行スキーマのモデルを指すことは、`stableSchema` の条件で書き込むユースケースが保つ
- 文字列の一致・部分一致・順序は、「取得と検索の規則」の「文字列の比較」に従う

### RevisionRepository

`Revision` を保存し、取り出す。履歴の版は `version` を持たず、楽観ロックを使わない。`TransactionalRepository` を継がず、持つメソッドは次のすべて。

```ts
interface RevisionRepository {
  insertMany(revisions: readonly Revision[]): Promise<void>;
  findById(id: RevisionId): Promise<Revision | null>;
  findByIds(ids: readonly RevisionId[]): Promise<readonly Revision[]>;
  findPage(contentId: ContentId, pagination: Pagination): Promise<PaginationResult<Revision>>;
  findTargetsByContents(
    contentIds: readonly ContentId[],
    after: RevisionId | null,
    limit: number,
  ): Promise<readonly RevisionTarget[]>;
  findExpired(cutoff: Date, limit: number): Promise<readonly RevisionTarget[]>;
  deleteMany(ids: readonly RevisionId[]): Promise<void>;
  countExpired(cutoff: Date): Promise<number>;
  scan(after: RevisionId | null, limit: number): Promise<readonly Revision[]>;
  count(): Promise<number>;
  lastId(): Promise<RevisionId | null>;
  deleteAll(limit: number): Promise<number>;
}
```

分類: `insertMany`、`deleteMany`、`deleteAll` は書き込み。ほかは読み取り。

| メソッド | 契約 |
| --- | --- |
| `insertMany` | 版を追加し、そのコンテンツへの参照を記録する。`id` が区画の中で一意であることを、ポートが担保する。同じ `id` の版が既にある、または `revisions` が同じ `id` の版を二つ以上含む場合は `ConflictError`（理由 `ALREADY_EXISTS`）で、どの版も追加しない。`revisions` は `ContentBatch.maxContents` の 2 倍の件数まで（上限の件数のコンテンツの、公開内容と下書きの版）。空の `revisions` は、何も変えずに成功する。版の参照先を確かめない（「Content」の「不変条件」）。`contentId` のコンテンツが存在することは、同じ UnitOfWork でコンテンツを保存するユースケースが保つ |
| `findById` | 版の ID で一件を返す。無ければ `null` |
| `findByIds` | 指定した ID のうち、存在する版を、`id` の昇順で返す。`ids` は `ContentBatch.maxRevisions` 個まで |
| `findPage` | 指定したコンテンツの版を、`retired.at` の降順、同じ値では `id` の降順で返す。`count` はそのコンテンツの版の総件数 |
| `findTargetsByContents` | 指定したコンテンツの版のすべてを、`RevisionTarget` で、`id` の昇順で返す。`after` より後ろの `id` から `limit` 件まで。`contentIds` は `ContentBatch.maxContents` 個まで、`limit` は `ContentPortLimits.revisionTargets` 以下。完全削除が、コンテンツとともになくなる版を求めるのに使う |
| `findExpired` | 区画の版のうち、`retired.at` が `cutoff` 以前のものを、`RevisionTarget` で、`retired.at` の早い順、同じ値では `id` の昇順に、`limit` 件まで返す。`limit` は `ContentBatch.maxRevisions` 以下。続きを読む位置を持たない。返った件数が `limit` より少なければ、期限を過ぎた版は、返ったもののほかに無い |
| `deleteMany` | 指定した ID の版を、参照の記録とともに削除する。複数のコンテンツの版を一回の呼び出しで渡せる。全部を削除するか、何も削除しないかのどちらかになる。確定の時点で、指定した版のいずれかが無い場合は `ConflictError`（理由 `REVISION_MISSING`）とし、該当する版の ID を示す。削除が公開内容・下書き・ほかの版に与える影響と、参照先の保護との関係は、「Revision」の「ライフサイクル」に従う。`ids` は `ContentBatch.maxRevisions` 個まで。空の `ids` は、何も変えずに成功する |
| `count` | 区画の版の件数を返す。スキーマの反映の準備が、変換の対象の件数に使う |
| `lastId` | 区画の版のうち、`scan` の順で最後の版の `id`（最大の `id`）を返す。版が無ければ `null`。スキーマの反映の準備が、変換の対象の走査の終端に使う |
| `countExpired` | `retired.at` が `cutoff` 以前の版の件数を返す |
| `scan` | 区画のすべての版を、`id` の昇順で返す。`after` より後ろの `id` から `limit` 件まで。`limit` は `ContentPortLimits.chunk` 以下 |
| `deleteAll` | 区画の版を、参照の記録とともに削除する |

- 版の変換は、`ContentStagingArea` の切り替えが行う

### ContentReferenceQuery

コンテンツへの参照の記録を問い合わせる。読み取りだけを行う。`PartitionReader.workspace(workspaceId)` が公開する。メディアの使用箇所は、Media の `MediaUsageRepository` が返す。

```ts
interface ContentReferenceQuery {
  findInbound(
    targets: readonly ContentId[],
    filter: Readonly<{ holders: readonly ReferenceHolder["type"][]; strengths: readonly ("required" | "optional" | "link")[] }>,
    pagination: Pagination,
  ): Promise<PaginationResult<InboundReference>>;
}
```

分類: `findInbound` は読み取り。

| メソッド | 契約 |
| --- | --- |
| `findInbound` | `targets` のいずれかを指す、コンテンツへの参照の記録のうち、持ち主の種類が `holders`、`strength` が `strengths` に含まれるものを返す。記録は、参照先のコンテンツが存在しなくても返す。参照元が `targets` に含まれる記録と、自己参照も返す。ごみ箱のコンテンツの下書きは、`holder` が `draft`、`source.location` が `trashed`。版の記録の `source.location` は、その版のコンテンツの置き場所。順序は、参照元のコンテンツID の昇順、同じコンテンツでは公開内容・下書き・履歴の版の順、版どうしは `revisionId` の昇順、同じ持ち主の中では `path` の昇順、同じ `path` では `target` の昇順。`path` は、要素（項目はキー、位置は 10 進の表記）を順に「文字列の比較」で比べ、先に尽きたほうを前にする。`targets` は `ContentPortLimits.chunk` 個まで。それより多い ID は、呼び出し側が分けて呼ぶ |

記録は、内容を保存した UnitOfWork の確定と同時に変わる。確定後の読み取りは、確定した記録を即座に反映する。

ワークスペースの利用終了では、参照の記録は、持ち主の内容とともに消える。版のものは `RevisionRepository.deleteAll`、公開内容・下書きのものは `ContentRepository.deleteAll` が消す。消す順は「利用終了の消去」が定める。

### PublicationLedgerRepository

`PublicationLedger` を保存し、公開の変更の記録を追記して、取り出す。追記だけの記録で、楽観ロックを使わない。`TransactionalRepository` と `SingletonRepository` を継がず、持つメソッドは次のすべて。追記、通し番号、参照元の規則は、この契約だけが定める。

```ts
type MigrationRecordHeader = Readonly<{
  operationId: OperationId;
  actor: Actor;
  schemaVersion: number;
}>;

interface PublicationLedgerRepository {
  insert(ledger: PublicationLedger): Promise<void>;
  append(record: PublicationRecordDraft, modelIds: Readonly<Record<string, ModelId>>): Promise<void>;
  appendStaged(
    jobId: SchemaApplyJobId,
    header: MigrationRecordHeader,
    modelIds: Readonly<Record<string, ModelId>>,
  ): Promise<Readonly<{ recorded: boolean }>>;
  findByOperationId(operationId: OperationId): Promise<PublicationRecord | null>;
  deleteAll(limit: number): Promise<number>;
}
```

分類: `insert`、`append`、`appendStaged`、`deleteAll` は書き込み。`findByOperationId` は読み取り。

| メソッド | 契約 |
| --- | --- |
| `insert` | 区画に一つであることを、ポートが担保する。その区画の `PublicationLedger` が既にあれば `ConflictError`（理由 `ALREADY_EXISTS`） |
| `append` | 記録を一件追記する。番号を返さない。呼んだ UnitOfWork の確定の時点で、ポートが、下の「通し番号」と「参照元」を決めて、`PublicationRecord` にする。同じ `operationId` の記録が既にあれば、UnitOfWork の全体を `ConflictError`（理由 `ALREADY_EXISTS`）として拒否する。`PublicationLedger` が無い区画では、UnitOfWork の全体を `ConflictError`（理由 `STALE_VERSION`）として拒否する。`changes` と参照元の件数に上限は無い |
| `appendStaged` | スキーマの反映の確定の記録を、`jobId` の準備領域（`ContentStagingArea`）の報告から作って追記する。報告のうち `ContentMigration.noticeOf` が `null` でないものが 1 件以上ある場合に、`PublicationLedger.record` に、`header` の `operationId`・`actor`・`schemaVersion`、`operation` の `migrated`、その `noticeOf` の値の全体を `changes` として渡した結果と同じドラフトを、`append` と同じ契約（通し番号、参照元、拒否する確定）で追記し、`recorded` に真を返す。該当する報告が無い場合は、追記せず、`PublicationLedger` を確かめず、`recorded` に偽を返す。`recorded` は、準備領域の報告だけから決まり、呼び出しの時点で返る |
| `findByOperationId` | その操作の ID の記録を返す。無ければ `null` |
| `deleteAll` | 区画の記録と `PublicationLedger` を削除する。件数の単位は、記録の一件と、`PublicationLedger` の一件。通し番号の昇順に記録を削除し、記録を消し切った呼び出しで、`limit` に余りがあれば `PublicationLedger` も削除する。利用終了の消去が使う |

通し番号。

- 確定済みの通し番号に 1 を足した番号を割り当て、`PublicationLedger` の通し番号をその番号にして、記録の `sequence` に入れる。割り当てた番号は、確定の順に 1 ずつ増える
- 並行して `append` を呼んだ二つの UnitOfWork は、どちらも確定し、確定の順の、違う番号を受ける
- 確定しなかった UnitOfWork は番号を使わず、番号に欠けを作らない
- 操作を行ったユースケースは、通し番号を受け取らない

参照元。

- 記録の `changeList.referrers` は、確定後の参照の記録で、`changes` のいずれかのコンテンツを、公開内容から参照するコンテンツ。`strength` によらない。コンテンツID の昇順に並ぶ。ほかの条件は [common.md](common.md) の「PublishedChangeNotice」が定める
- 参照元の `model` は、確定後の現行スキーマでの、その参照元のモデルのキー。`append` と `appendStaged` は、参照元の `modelId` を、渡された `modelIds`（確定後の現行スキーマの `WorkspaceSchema.modelIds`）で引く
- 参照元を、追記と同じ確定で求めるので、追記の前に公開された参照元は、必ず記録に入る。追記の後に公開された参照元は、その公開の記録が、変更した対象として持つ

呼び方。

- 一つの UnitOfWork は、`append` または `appendStaged` を一度だけ呼び、追記した場合に限り、`content.publication_changed` のドラフトをちょうど一つ `collectEvents` に渡す。対象の件数によらない。呼び出し側が守る
- 可視性: 確定後の `findByOperationId` は、確定した記録と、割り当てた通し番号と、参照元を即座に反映する
- ポート適合テストは、割り当てた番号と参照元を、確定の後の `findByOperationId` で確かめる。`appendStaged` の追記も、同じ読み取りで確かめる

### ContentRetentionPolicyRepository

`ContentRetentionPolicy` を保存し、取り出す。区画に一つの集約で、`SingletonRepository`（[common.md](common.md)）を継ぐ。

```ts
interface ContentRetentionPolicyRepository extends SingletonRepository<ContentRetentionPolicy> {
  deleteAll(limit: number): Promise<number>;
}
```

分類: `deleteAll` は書き込み。

| メソッド | 契約 |
| --- | --- |
| `insert`、`find`、`save` | `SingletonRepository` の契約に従う。加える契約は無い |
| `deleteAll` | 区画の一件を削除し、削除した件数（0 または 1）を返す。楽観ロックを適用しない |

### EditingPresenceStore

編集中の記録を置く。区画に結び付かないポートで、UnitOfWork に属さない（[common.md](common.md) の「UnitOfWorkProvider」）。楽観ロックを使わない。対象のワークスペースを引数で受ける。

```ts
interface EditingPresenceStore {
  put(presence: EditingPresence): Promise<void>;
  find(workspaceId: WorkspaceId, contentId: ContentId, memberId: MemberId): Promise<EditingPresence | null>;
  list(workspaceId: WorkspaceId, contentId: ContentId): Promise<readonly EditingPresence[]>;
  remove(workspaceId: WorkspaceId, contentId: ContentId, memberId: MemberId): Promise<void>;
  deleteExpired(now: Date, limit: number): Promise<number>;
  deleteByWorkspace(workspaceId: WorkspaceId, limit: number): Promise<number>;
}
```

| メソッド | 契約 |
| --- | --- |
| `put` | ワークスペース、コンテンツ、メンバーの組の記録を、渡した記録で置き換える。無ければ作る。組が一意であることを、ポートが担保する。並行した `put` は、後の書き込みが残る。楽観ロックを持たない |
| `find` | 組の記録を返す。無ければ `null`。失効した記録も返す |
| `list` | 指定したワークスペースのコンテンツの記録を、失効した記録を含めて、すべて返す。順序を約束しない |
| `remove` | 組の記録を削除する。無い場合も成功する |
| `deleteExpired` | すべてのワークスペースの、`expiresAt` が `now` 以前の記録を削除する。有効な記録を削除しない。定期ジョブ「編集中の記録の掃除」が使う |
| `deleteByWorkspace` | 指定したワークスペースの記録を削除する。ほかのワークスペースの記録を削除しない |

- 書き込みは、呼び出しの完了で反映する。以後の読み取りは即座に反映する
- `deleteExpired` と `deleteByWorkspace` は、[common.md](common.md) の「TransactionalRepository」の「件数を区切る削除」に従う
- 参照整合性を持たない。コンテンツとメンバーの存在は確かめない。削除されたコンテンツの記録は、失効で消える
- `deleteByWorkspace` は、呼び出しの後の `put` を妨げない。認可とコンテンツの読み取りを終えていた `announceEditing` が、`deleteByWorkspace` の後に `put` した記録は残り、`expiresAt` で失効して、定期ジョブ「編集中の記録の掃除」（`deleteExpired`）が削除する。ワークスペースを名簿から外した後に残った記録も同じ
- 記録が失われても、保存済みのコンテンツに影響しない
- バックアップの範囲は、[installation.md](installation.md) の「BackupStore」の契約が定める。復旧の後に残った記録は、失効で消える

### ContentStagingArea

スキーマの反映の変換結果を、反映と事前確認の一回ごとの準備領域に置き、確定で切り替える。`jobId` は、反映・事前確認の一回を表す Schema の `SchemaApplyJob` の ID（`SchemaApplyJobId`。[schema.md](schema.md) の「SchemaApplyJob」）。

```ts
type StagingCursor = Readonly<{
  lastContentId: ContentId | null;
  lastRevisionId: RevisionId | null;
  contents: number;
  revisions: number;
}>;

interface ContentStagingArea {
  putContents(jobId: SchemaApplyJobId, items: readonly StagedContent[]): Promise<void>;
  putRevisions(jobId: SchemaApplyJobId, items: readonly StagedRevision[]): Promise<void>;
  cursor(jobId: SchemaApplyJobId): Promise<StagingCursor>;
  listContentReports(jobId: SchemaApplyJobId, after: ContentId | null, limit: number): Promise<readonly ContentMigrationReport[]>;
  listRevisionReports(jobId: SchemaApplyJobId, after: RevisionId | null, limit: number): Promise<readonly RevisionMigrationReport[]>;
  findDuplicates(jobId: SchemaApplyJobId, modelId: ModelId, field: FieldKey): Promise<UniqueDuplicates>;
  promote(jobId: SchemaApplyJobId): Promise<void>;
  discard(jobId: SchemaApplyJobId): Promise<void>;
  deleteAll(limit: number): Promise<number>;
}
```

分類: `putContents`、`putRevisions`、`promote`、`discard`、`deleteAll` は書き込み。ほかは読み取り。

| メソッド | 契約 |
| --- | --- |
| `putContents` | コンテンツの変換結果と報告を、準備領域に書く。結果のコンテンツは、`result` があればその `id`、`null` なら `contentId` が指す。同じ `jobId` の同じコンテンツの結果が既にあれば置き換える。繰り返しても結果は変わらない。`items` は `ContentPortLimits.chunk` 件まで。空の `items` は、何も変えずに成功する |
| `putRevisions` | 履歴の版の変換結果と報告を、準備領域に書く。結果の版は、`result` があればその `id`、`null` なら `target` が指す。置き換えと件数の上限は `putContents` と同じ |
| `cursor` | 準備領域に書いたコンテンツと版の件数と、それぞれの最大の ID を返す。準備領域が無い場合は、件数 0 と `null` を返す |
| `listContentReports` | 準備領域の報告を、コンテンツID の昇順で返す。`after` より後ろの ID から `limit` 件まで。`limit` は `ContentPortLimits.chunk` 以下 |
| `listRevisionReports` | 準備領域の版の報告を、版の ID の昇順で返す。`after` より後ろの ID から `limit` 件まで。`limit` は `ContentPortLimits.chunk` 以下 |
| `findDuplicates` | 準備領域の変換結果のうち、指定したモデルに属し、公開内容を持つコンテンツについて、公開内容の `uniqueValues` の `field` の値が一致するコンテンツの組を返す。二件以上が同じ値を持つ組だけを返す。未設定の値どうしは、一致する組に数えない。組の件数（`total`）と、組に属するコンテンツの全件数（`contentCount`）と、組の中で最小のコンテンツID の昇順で先頭 `SchemaLimits.sampleLimit` 組（[schema.md](schema.md)）を返す。組の `contents` は、コンテンツID の昇順で `SchemaLimits.sampleLimit` 件まで |
| `promote` | 区画のコンテンツ、履歴の版、参照の記録、一意な値の記録を、準備領域の変換結果に切り替える。`result` が `null` のコンテンツと版は削除する。切り替えは、呼び出した UnitOfWork の確定と、全部成功か全部反映されないかを共にする。変換結果の量によらない。確定の後、準備領域は無くなる |
| `discard` | 準備領域を破棄する。無い場合も成功する。破棄は UnitOfWork の確定で反映する。破棄した準備領域は、何も書いていない準備領域と同じに扱う。`cursor` は件数 0 と `null`、`list*` と `findDuplicates` は空を返し、同じ `jobId` への `put*` は新しく書ける |
| `deleteAll` | 区画のすべての準備領域の変換結果と報告を、削除する。件数の単位は、コンテンツの変換結果と版の変換結果の一件ごと（報告は、その結果とともに消える）。利用終了の消去が使う |

- 準備領域は `jobId` ごとに分かれる。別の `jobId` の準備領域は、互いに影響しない
- 準備領域の内容は、`ContentRepository`、`RevisionRepository`、`ContentReferenceQuery` の読み取りに現れない。`promote` を呼んだ UnitOfWork の確定の前は、どの読み取りも反映前の内容を返す。確定の後は、どの読み取りも変換結果を返す
- `promote` は、準備領域が、区画のすべてのコンテンツとすべての版の結果を過不足なく持つことを、ポートが担保する。持たない場合は `ConflictError`（理由 `STAGING_INCOMPLETE`）とし、何も切り替えない
- コンテンツも版も無い区画で、何も書いていない準備領域の `promote` は成功し、何も変えない
- `promote` が確かめる整合性は、切り替えた後の公開内容だけ。`ContentRepository` の「確定の時点の整合性」の `UNIQUE_VALUE_TAKEN` と `REFERENCE_NOT_PUBLISHED` を適用する。切り替えで削除されるコンテンツを、公開内容が保護する参照で参照する場合も `REFERENCE_NOT_PUBLISHED`。`promote` が返す公開内容の整合性の理由は、この二つだけ
- 準備の確認と `promote` の拒否が食い違う場合の扱いは、[schema.md](schema.md) の「確定できない場合の扱い」が定める
- `promote` は、下書きと履歴の版の参照先を確かめず、`REFERENCED` を返さない（「Content」の「不変条件」）
- `putContents`、`putRevisions`、`promote` は、準備領域の容量または切り替えの処理能力の上限を超える場合、`SystemError`（理由 `STAGING_CAPACITY_EXCEEDED`）を返す。その UnitOfWork は何も反映しない。確定済みの `put*` の結果は残る。Schema のユースケースの扱いは、[../usecases/schema.md](../usecases/schema.md) の「確定しなかった理由の分類」が定める
- `put*` と `list*` と `findDuplicates` の読み取りは、同じ `jobId` の確定した `put*` を即座に反映する
- 一意な値の一致は、「取得と検索の規則」の「文字列の比較」に従う

## 初期構成の宣言

初期構成の読み込みとワークスペースの作成で、Content が受ける宣言と、その検証・生成の手順。各ドメインを呼ぶ順は、[index.md](index.md) の「手順を呼ぶ順」が定める。宣言の形の全体と、`path` に渡す位置は、[installation.md](installation.md) の「初期構成の読み込み」が定める。`DeclarationIssue` と検証の規則は [common.md](common.md) の「DeclarationIssue」が定める。生成は、Installation のユースケースが、そのワークスペースの区画の UnitOfWork（条件は `none`）で、`Workspace` の `insert` とともに確定する（[index.md](index.md) の「区画をまたぐ手順」）。

```ts
const ContentRetentionDeclaration: {
  validate(
    entries: readonly Readonly<{ path: string; declaration: ContentRetentionPolicyInput }>[],
  ): readonly DeclarationIssue[];
};
```

宣言は、ワークスペースのコンテンツの保持方針（`ContentRetentionPolicyInput`）。宣言しないワークスペースは、`entries` に含めない。

- 宣言の照合値から除く値: 無い。保持方針の二つの日数を、照合値に含める
- 手順の前提: `Workspace` の生成。Content の二つの集約は、`Workspace` と同じ UnitOfWork で作り、ワークスペースに一つの集約を欠いたワークスペースを作らない

検証。

| 規則 | `path` | `code` |
| --- | --- | --- |
| `ContentRetentionPolicy.validate` | 渡された `path` に、返った `RetentionIssue.field` を続ける | `ContentRetentionOutOfRange` |

- `ContentRetentionOutOfRange` は、この検証だけが返す理由。エラーとして投げない

生成。ワークスペースごとに、次の二つを保存する。初期構成の読み込みでも、運用コマンドによるワークスペースの作成でも同じ。二つの間に順の約束は無い。

- `PublicationLedger.create()` の結果を、`PublicationLedgerRepository.insert` で保存する
- `ContentRetentionPolicy.create(input, now)` の結果を、`ContentRetentionPolicyRepository.insert` で保存する。宣言が無いワークスペースと、運用コマンドによる作成は `input` に `null` を渡す

## 利用再開前の処理

バックアップから復旧した設置で、Content が利用の再開の前に済ませる処理は無い。手順を持たず、UnitOfWork を開かない。

- 手順の前提: 無い。進行中だったスキーマの反映と事前確認の準備領域は、Schema の「利用再開前の処理」が破棄する
- 残件数: 無い（常に 0）

利用再開前の確認に公開する項目。[common.md](common.md)「各ドメインが公開する項目」の `ResumeItemDefinition` の列と、件数を求める関数で公開する。

```ts
type ContentResumeItemKind = "published_content" | "unpublished_content";

const ContentResumeItems: {
  definitions: readonly (ResumeItemDefinition & Readonly<{ kind: ContentResumeItemKind }>)[];
  counts(publication: Readonly<{ published: number; unpublished: number }>): readonly ResumeItem[];
};
```

| `kind` | 件数の求め方 | 単位 | `countsAsRemaining` | 確認に示す（`shownInReview`） | 内容 |
| --- | --- | --- | --- | --- | --- |
| `published_content` | `ContentRepository.countByPublication()` の `published` | ワークスペース | 偽 | 真 | 公開中のコンテンツの件数 |
| `unpublished_content` | `ContentRepository.countByPublication()` の `unpublished` | ワークスペース | 偽 | 真 | 未公開のコンテンツの件数。ごみ箱のコンテンツを含む |

- `counts` は、名簿のワークスペースごとに、その区画の `countByPublication()`（UnitOfWork の外。`PartitionReader.workspace(workspaceId)`）の結果を受けて、二つの項目を返す

- 復旧点で未公開だったコンテンツは、未公開のまま戻る

## 状態確認の項目

無い。Content は、設置の状態確認（Installation の `checkInstallationStatus`）に示す項目を持たない。必要な対処を伴う Content の状態は無い。

## 利用終了の消去

ワークスペースの利用終了で、Content のデータを消去する手順。区画の消去と件数を区切る削除の規則は、[common.md](common.md) の「TransactionalRepository」に従う。ドメインを呼ぶ順は、[index.md](index.md) の「手順を呼ぶ順」が定める。

呼ぶメソッド。

- `EditingPresenceStore.deleteByWorkspace(workspaceId, limit)`。UnitOfWork に属さない。消去の後に書かれた記録の扱いは、`EditingPresenceStore` の契約が定める
- `ContentStagingArea.deleteAll(limit)`、`RevisionRepository.deleteAll(limit)`、`ContentRepository.deleteAll(limit)`、`ContentRetentionPolicyRepository.deleteAll(limit)`、`PublicationLedgerRepository.deleteAll(limit)`。一回の呼び出しごとに、そのワークスペースの区画の一つの UnitOfWork（条件は `none`）

Content の中の順の前提は一つ。

- 履歴の版（`RevisionRepository.deleteAll`）を、コンテンツ（`ContentRepository.deleteAll`）より先に消し切る。版の参照の記録の `source.location` を求められるコンテンツが、途中の状態でも残るため

ほかのメソッドの間に、順の約束は無い。

- 手順の前提: 先に消去を終えているべきドメインは無い。Content の集約を参照する、ほかのドメインの集約は無い。メディアの使用箇所は、Content の消去が消さず、Media の「利用終了の消去」が消す
- 適用中のスキーマの反映は、準備と確定の UnitOfWork が `active` の条件を満たさないので確定できず、準備領域へ新たに書かれない。消去は、適用中の反映と進行中の事前確認を止めずに進める
- 消去済みのメソッドは 0 を返す

利用終了の前の確認に公開するもの。

| 項目 | 内容 |
| --- | --- |
| 利用終了で失われるデータの種類 | `content`（コンテンツ。公開内容、下書き、ごみ箱のコンテンツ）、`revision`（履歴の版）、`retention_policy`（コンテンツの保持方針） |
| CMS の通常データの外に残る情報 | `delivered_content`（表示アプリに渡した公開内容。CMS は回収しない） |
| 消去する資格情報 | 無い |
| 未完了の処理の件数 | 無い |

## エラー

`ContentErrorCode` は、Content の `BusinessRuleError`、`ConflictError`、`ForbiddenError`、`SystemError` の理由と、初期構成の宣言の問題の理由をすべて列挙する。共通の理由（`STALE_VERSION`、`ALREADY_EXISTS`、`SCHEMA_CHANGED`、`WORKSPACE_NOT_ACTIVE`、`INVALID_QUERY`、`PERMISSION_MISSING`）は [common.md](common.md) が、メディアの理由（`MEDIA_NOT_FOUND`、`MEDIA_NOT_PUBLIC`、`MEDIA_IN_USE`、`MEDIA_IN_PUBLIC_USE`）は [media.md](media.md) が定める。

| 種類 | コード | 意味 |
| --- | --- | --- |
| `DeclarationIssue.code` | `ContentRetentionOutOfRange` | 初期構成のコンテンツの保持方針の日数が、`RetentionDays` の有効な値でない |
| `BusinessRuleError` | `ContentInTrash` | ごみ箱のコンテンツに行えない操作 |
| `BusinessRuleError` | `ContentNotInTrash` | 通常の管理対象のコンテンツの、ごみ箱からの復元 |
| `BusinessRuleError` | `ContentNotPublished` | 未公開のコンテンツの、公開取り消しと下書きの破棄 |
| `BusinessRuleError` | `ContentHasNoDraft` | 下書きを持たないコンテンツの、下書きの破棄 |
| `BusinessRuleError` | `ModelDeprecated` | 非推奨のモデルで行えない操作（「操作の種類」の表） |
| `BusinessRuleError` | `SingleContentExists` | 通常の管理対象にコンテンツがある単一モデルへの作成と、ごみ箱からの復元 |
| `BusinessRuleError` | `CheckedValueMismatch` | 振る舞いが受けた検証済みの値が、別のモデル・コンテンツ、または別の内容の組で検証されたもの（「Content」の「振る舞い」） |
| `BusinessRuleError` | `PublishRejected` | `PublishDecision` の拒否 |
| `BusinessRuleError` | `WithdrawRejected` | `checkWithdraw` の拒否 |
| `BusinessRuleError` | `PurgeRejected` | `checkPurge` の拒否 |
| `BusinessRuleError` | `RewriteRejected` | `RewriteDecision` の拒否のうち、構造の違反を除くもの |
| `BusinessRuleError` | `DeliveryRejected` | `DeliveryFailure` |
| `ForbiddenError` | `EDITOR_MISMATCH` | 入力したメンバーと、保存する実行者が違う（`InputSource.isOwnedBy` が偽） |
| `ConflictError` | `SINGLE_EXISTS`、`UNIQUE_VALUE_TAKEN`、`REFERENCE_NOT_PUBLISHED`、`PUBLISHED_REFERENCE_REQUIRED` | `ContentRepository` の「確定の時点の整合性」が定める |
| `ConflictError` | `REFERENCED` | `ContentRepository.deleteMany` の契約が定める |
| `ConflictError` | `REVISION_MISSING` | `RevisionRepository.deleteMany` の契約が定める |
| `ConflictError` | `STAGING_INCOMPLETE` | `ContentStagingArea.promote` の契約が定める |
| `SystemError` | `STAGING_CAPACITY_EXCEEDED` | `ContentStagingArea` の契約が定める |

ドメインサービスの判定結果と、利用者の入力の違反は、ユースケースが次のエラーにして返す。スキーマの反映の適用中の書き込みは [schema.md](schema.md) の「WriteScope とスキーマ」が、確定の時点でスキーマの版が変わった書き込みは [common.md](common.md) の「WriteScope」が定める。

| 判定結果 | エラー |
| --- | --- |
| 存在しないモデル・コンテンツ・版。`Revision.belongsTo` が偽の版を含む | `NotFoundError`。キーまたは ID を示す |
| `EditCheck` の `ok: false`、`RewriteViolation` の `structure` | `ValidationError`。項目ごとの理由を持つ |
| `PublishDecision`、`RemovalDecision`、`RewriteDecision`（`structure` を除く）の `allowed: false`、`DeliveryResult` の `rejected`、`selectUnique` の `ok: false` | 上の表の `BusinessRuleError`。違反の内容を添える |
| `QueryCheck` の `ok: false`、空でない `QueryIssue`、空でない `ContentRetentionPolicy.validate` の結果 | `ValidationError` |
| `RichText`、`MediaReference`、`RetentionDays` の生成時の制約の違反 | `ValidationError` |
| 同じ ID で、内容の違うコンテンツが既にある作成 | `ConflictError`（理由 `ALREADY_EXISTS`） |

### 拒否の順

コンテンツと履歴の版を書き込むユースケースで、一つの要求に複数の拒否が成り立つ場合は、次の表で上にある行の拒否を返す。複数の対象を扱う操作は、対象ごとの拒否（行 5〜12）で打ち切らず、すべての対象を確かめる。対象ごとに違う行が当たる場合は、当たる行のうちいちばん上の行の拒否を返し、そのエラーの `rejections` に、条件を満たさない対象ごとに、対象の ID（コンテンツ、版、メディア）と、その対象に当たるいちばん上の行の拒否（エラーの種類、理由、添える内容）を、入力の順に添える。行 2 を、行 3 から後のスキーマの反映の適用中などの拒否より先に置く順は、[index.md](index.md) の「利用できるワークスペース」に従う。ほかの拒否の重なりの扱いは、この表だけが定める。拒否ごとのエラーは、上の二つの表と、各行が指す定義元が定める。

| 順 | 拒否 |
| --- | --- |
| 1 | 入力だけで決まる違反（対象の件数、`RichText`・`MediaReference` の形） |
| 2 | 実行者の種類、メンバーの認可（必要な権限を含む）、トークンの `TokenAuthorization.resolve` の拒否、利用できないワークスペース（[index.md](index.md) の「認可」と Token の規則） |
| 3 | スキーマの反映の適用中（[schema.md](schema.md) の「WriteScope とスキーマ」） |
| 4 | 入力したメンバーと、保存する実行者が違う |
| 5 | 操作の対象（モデル、コンテンツ、版、メディア）の不在 |
| 6 | トークンの `TokenAuthorization.authorize` の拒否（許可、範囲、権限。その間の順は Token が定める） |
| 7 | 添えた `version` の食い違い。対象の集約によらない |
| 8 | 対象の状態の絞り込みの拒否（`Content.asActive` ほか） |
| 9 | 加わる権限の不足 |
| 10 | 読んだスキーマに照らした値の違反（`EditCheck` の `ok: false`、`RewriteViolation` の `structure`） |
| 11 | 事前の検証の拒否（`PublicationRules`・`RewriteRules` の判定の拒否と、同じ ID で内容の違うコンテンツの作成）。`checkWithdraw` と `checkPurge` の両方が拒否する要求は、`checkWithdraw` の拒否 |
| 12 | 振る舞いの拒否（`ModelDeprecated`、`SingleContentExists`）。その間の順は「Content」の「振る舞い」が定める |
| 13 | 確定の時点のポートの拒否。複数の理由に当たる確定が返す理由は、「ポート」が定める |

- 確定の時点でワークスペースが `active` でない場合と、スキーマの版が変わった場合の理由は、[common.md](common.md) の「WriteScope」が定める

## ユースケース（概要）

- コンテンツを一覧・検索する
- ごみ箱のコンテンツを一覧する
- 編集するコンテンツを取得する
- 参照と一意な値の状態を確認する
- コンテンツの候補を検索する
- コンテンツを複製する入力を求める
- 入力を最新の内容と比較する
- コンテンツを作成する
- 下書きを保存する
- 下書きを破棄する
- 公開の確認を求める
- コンテンツを公開する
- 公開取り消しの確認を求める
- 公開を取り消す
- ごみ箱への移動の確認を求める
- コンテンツをごみ箱へ移動する
- ごみ箱のコンテンツを復元する
- 完全削除の確認を求める
- コンテンツを完全削除する
- 値を書き換える
- モデルごとの入力エラーと制約違反の件数を確認する
- 履歴を一覧する
- 履歴の版を取得する
- 内容を比較する
- 版の復元の確認を求める
- 版を下書きとして復元する
- 履歴の版の削除の確認を求める
- 履歴の版を削除する
- メディアの使用箇所を一覧する
- 履歴の版とともにメディアを削除する
- 公開コンテンツを単件で取得する
- 公開コンテンツの一覧を取得する
- プレビュー取得で単件を取得する
- プレビュー取得で一覧を取得する
- 編集中であることを知らせる
- 編集を終えたことを知らせる
- 編集中のメンバーを確認する
- 編集中の記録を掃除する
- コンテンツの保持方針を確認する
- 保持方針の変更の影響を確認する
- コンテンツの保持方針を変更する
- 期限を過ぎた履歴の版を削除する
- 期限を過ぎたごみ箱のコンテンツを完全削除する

スキーマの反映の準備・確定・失敗は Schema のユースケースで、`ContentMigration`、`MediaUsages`、`ContentStagingArea` を使う。フォームの受付は Form のユースケースで、`PublishedContentFacts.of` を使う。メディアの使用箇所の一覧と、履歴の版の削除を伴うメディアの削除は Content のユースケースで、Media の型・振る舞い・ポートを使う。詳細は `spec/usecases/` が定める。
