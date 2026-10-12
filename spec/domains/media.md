# Media

メディアの実体・公開範囲・派生形式を保つ。境界、依存方向、区画、ドメインをまたぐ規則、定期ジョブは [index.md](index.md) が、集約の ID、`Actor`、`Pagination`、`TransactionalRepository`、`WriteScope`、`UnitOfWorkProvider`、共通のエラーの理由は [common.md](common.md) が定める。

Media の集約、記録、ポートの区画は、[index.md](index.md) の「集約とポートの置き場所」に従う。集約と記録は、ワークスペースの ID を持たない。区画の記録と区画の外の実体が食い違わないための規則は、「実体と記録の整合」だけが定める。

コードは `packages/core/src/domain/media/` に置く。

集約は三つ。`Media` は、登録時の記録と公開範囲を持つ。`MediaProfile` は、管理用の名称、検索情報、共有する説明を持つ。`MediaDerivatives` は、画像一件の、派生形式ごとの状態を持つ。三つは `MediaId` で対応する。作成は「トランザクション境界」に、削除は `MediaRepository.delete` に従う。互いの保存が競合しないことは、`MediaProfileRepository.save` と `MediaDerivativesRepository.save` の契約が定める。

## ユビキタス言語

| 英語名 | 日本語名 | 定義 |
| --- | --- | --- |
| Media | メディア | ワークスペースに登録した一つのファイルと、その記録 |
| Original file | 元のファイル | 登録したファイル。内容は登録後に変わらない |
| File record | 登録時の記録 | 登録時に求めた形式、容量、画像の幅・高さ |
| Visibility | 公開範囲 | 非公開（`private`）または公開（`public`） |
| Media profile | 管理情報 | 管理用の名称、検索情報、共有する説明の組（`MediaProfile`） |
| Name | 管理用の名称 | 管理画面でメディアを見分ける名称。公開表示に使わない |
| Search info | 検索情報 | 管理画面の検索に使う語。公開表示に使わない |
| Shared description | 共有する説明 | メディアが持つ代替テキストとキャプション。使用箇所の説明の元になる。使用箇所の説明は、使う側が持つ |
| Holder | 使う側 | メディアを使うもの一つ。`holderId` で表す（`MediaUsageRepository`） |
| Usage | 使用箇所 | 使う側一つが、メディア一件を使っているという記録。使う側のユースケースが `MediaUsageRepository` に記録する |
| Staged usage | 準備領域の使用箇所 | スキーマの反映の準備が書く、切り替える前の使用箇所（`MediaUsageRepository` の「準備領域」） |
| Public use | 公開使用 | 公開の配信を前提にする使用箇所（`public` が真） |
| Derivative | 派生形式 | スキーマの `ImageDerivative` の定義から、元の画像を変換して作る画像 |
| Derivative record | 派生形式の記録 | 画像一件の、派生形式ごとの状態（`DerivativeState`）を持つ集約（`MediaDerivatives`） |
| Target | 生成の対象 | 派生形式の定義と、その世代の組 |
| Generation | 世代 | 生成の対象ごとに割り当てる値。定義が変わると新しい値になる |
| Derivative status | 生成状況 | 現行スキーマが定義する派生形式一つの、生成中（`generating`）、生成済み（`ready`）、失敗（`failed`）の別（`DerivativeStatus`）。`MediaDerivatives.statuses` が求める |
| Previous derivative | 変更前の派生形式 | 定義の変更の前に生成済みだった派生形式。新しい派生形式が生成済みになるまで配信を続ける |
| Stalled generation | 止まった生成 | 生成中のまま、`DerivativePolicy.isStalled` が真になった派生形式 |
| Rendition | 配信できる画像 | 画像の取得結果に載せる、元の画像と生成済みの派生形式。文書の取得結果は、元のファイルを載せる |
| Object | 実体 | 元のファイル、または派生形式一つの、メディアの保管先で確定した内容 |
| Object commit | 実体の確定 | `MediaStorage.commit` が、保管の試みの内容を実体にすること。UnitOfWork に属さない。「確定した実体」「確定していない試み」の確定はこれを指し、ほかの「確定」は UnitOfWork の確定を指す |
| Object location | 実体の位置 | 実体一つを指す値。区画の中では `MediaObjectKey`、区画の外のポートへは `MediaObjectRef` で表す。実体を保管する前から決まる |
| Object scope | 実体の範囲 | 保管の予約と消去待ちが指す実体のまとまり（`StorageTarget`）。メディア一件の全部、または派生形式一つの一つの世代 |
| Media storage | メディアの保管先 | 実体を置く場所。区画の外にある |
| Attempt | 保管の試み | メディアの保管先への一回の書き込み。確定するまで、その内容は実体でない。確定していない試みの内容は、消去が取り除くか、アダプターが期限で捨てる |
| Storage reservation | 保管の予約 | 実体の保管を始める前に確定する記録。実体の範囲を指し、期限を持つ。保管を行う処理が、結果の確定または後始末で取り除く |
| Object removal | 消去待ち | メディアの保管先からの消去を待つ、実体の範囲の記録 |
| Delivery | 配信 | CMS が実体を、要求に応じて返すこと（`MediaDelivery`） |
| Downstream retention | 配信の先の保持 | 公開のメディアの内容を、配信の入口の先（表示アプリ、その CDN、ブラウザ）が、長く保持すること |
| Preview link | プレビュー用の配信先 | 期限付きの許可を含む配信先。非公開のメディアを取得できる |

## エンティティ

### Media

フィールド。

```ts
type Visibility = "private" | "public";

type MediaBase = Readonly<{
  id: MediaId;
  visibility: Visibility;
  version: number;
  createdAt: Date;
  updatedAt: Date;
}>;

type ImageMedia = MediaBase & Readonly<{ file: ImageFile }>;
type DocumentMedia = MediaBase & Readonly<{ file: DocumentFile }>;
type Media = ImageMedia | DocumentMedia;

type PrivateMedia = Media & Readonly<{ visibility: "private" }>;
type PublicMedia = Media & Readonly<{ visibility: "public" }>;

type MediaRemoval = Readonly<{ permission: Permission; removal: ObjectRemoval }>;

type MediaPublication =
  | Readonly<{ needed: false }>
  | Readonly<{ needed: true; permission: Permission }>;
```

| フィールド | 制約 |
| --- | --- |
| `id` | 必須。生成後に変わらない |
| `file` | 必須。登録時の記録。生成後に変わらない。`file.kind` が画像（`image`）と文書（`document`）を判別する |
| `visibility` | 必須。登録時は `private` |
| `version` | 楽観ロックの番号。登録時は 0。状態を変える振る舞いごとに 1 進む |
| `createdAt` | 登録日時 |

振る舞い。

```ts
const Media: {
  register(params: { id: string; file: MediaFile }, now: Date): PrivateMedia;

  publication(media: Media): MediaPublication;
  publish(media: PrivateMedia, now: Date): PublicMedia;
  unpublish(media: Media, now: Date): PrivateMedia;
  removal(media: Media, now: Date): MediaRemoval;

  assertImage(media: Media): ImageMedia;
  facts(media: readonly Media[]): MediaFactSet;
  isImage(media: Media): media is ImageMedia;
  isPublic(media: Media): media is PublicMedia;
};
```

| 振る舞い | 処理 |
| --- | --- |
| `register` | `MediaId` を生成し、`private` のメディアを返す。`file` は `MediaAdmission.examine` が返した記録。イベントを発行しない |
| `publication` | 「公開と非公開化」が定める |
| `publish` | `visibility` を `public` にする。適用するのは `MediaRepository.publishMany` |
| `unpublish` | `public` のメディアは、`visibility` を `private` にする。`private` のメディアは、渡されたメディアをそのまま返す（[index.md](index.md) の「変更の無い変更」） |
| `removal` | 「削除」が定める |
| `assertImage` | 画像でないメディアは `BusinessRuleError`（`MediaNotImage`）。画像なら、渡されたメディアを返す |
| `facts` | 「使う側へ渡す事実」が定める |
| `isImage` | `file.kind` が `image` なら真。Media の中で、画像のメディアに型を絞るために使う |
| `isPublic` | `visibility` が `public` なら真 |

`publish` と、値を変えた `unpublish` は、`version` を 1 進め、`updatedAt` を `now` にする。

#### 公開と非公開化

- `publication` は、メディアを公開使用する操作が、そのメディアを新たに公開するかどうかと、公開に必要な権限を返す。`private` のメディアは `needed` が真で、`permission` は `media:publish`。`public` のメディアは `needed` が偽で、権限を求めない。メディアを変えない
- 公開に必要な権限は、この振る舞いだけが定める
- メディアを公開するのは、メディアを公開使用する側のユースケースだけで、手段は `MediaRepository.publishMany`。使う側の求め方は [content.md](content.md) の「新たに公開するメディア」が、同じ UnitOfWork で確定するものは [index.md](index.md) の「同じ UnitOfWork で確定するもの」が定める
- メディアの登録と、公開使用でない使用箇所の記録は、メディアを公開しない
- 公開使用と公開範囲の整合は、`MediaRepository.save` と `MediaUsageRepository` の契約が定める
- 非公開化は実体を変えない。非公開化の後の配信は、`MediaDelivery.decide` が決める

#### 削除

`removal` は、メディアの削除に必要な権限と、削除と同じ UnitOfWork で記録する消去待ちを返す。メディアを変えない。

- `permission` は、`private` のメディアでは `content:edit`、`public` のメディアでは `media:publish`
- `removal` は、`ObjectRemoval.ofMedia(media.id, now)` の値。そのメディアの元のファイルとすべての派生形式の実体を指す
- 使用箇所による拒否は、`MediaRepository.delete` の契約が定める
- メディアの削除に必要な権限と消去待ちは、この節だけが定める。`removal` を呼ぶのは、[../usecases/media.md](../usecases/media.md) の「メディアの削除の手続き」だけ

#### 使う側へ渡す事実

```ts
type MediaFactSet = ReadonlyMap<MediaId, Readonly<{ kind: MediaFile["kind"]; visibility: Visibility }>>;
```

- `facts` は、渡したメディアごとに、メディアの種類（`file.kind`）と公開範囲を返す。無いメディアは、結果に含まれないことで表す
- `facts` は、場所ごとに使える種類を判定しない。判定は使う側が行う

不変条件。

- `file` は登録後に変わらない
- 登録したメディアは `private` である。`public` になるのは `publish` だけ、`private` に戻るのは `unpublish` だけ
- 公開使用のあるメディアは `private` にならない

ライフサイクル。

```text
register → private ─ publish → public ─ unpublish（公開使用が無い） → private
```

- 生成: メディアの登録（「トランザクション境界」）
- `private` → `public`: `MediaRepository.publishMany`
- 削除: 完全削除（`MediaRepository.delete`）。ワークスペースの利用終了は、そのワークスペースのメディアをすべて削除する（「利用終了の消去」）

### MediaProfile

メディア一件の管理情報。`MediaId` で識別する。

フィールド。

```ts
type MediaProfile = Readonly<{
  mediaId: MediaId;
  name: MediaName;
  searchInfo: MediaSearchInfo;
  sharedDescription: SharedDescription;
  version: number;
  createdAt: Date;
  updatedAt: Date;
}>;
```

| フィールド | 制約 |
| --- | --- |
| `mediaId` | 必須。その `Media` の `id`。生成後に変わらない |
| `name` | 必須。登録時は元のファイル名から作る |
| `searchInfo` | 必須。登録時は空 |
| `sharedDescription` | 必須。登録時は代替テキストとキャプションが未設定 |
| `version` | 楽観ロックの番号。作成時は 0。状態を変える振る舞いごとに 1 進む |

振る舞い。

```ts
const MediaProfile: {
  create(media: Media, fileName: string, now: Date): MediaProfile;
  describe(
    profile: MediaProfile,
    input: { name: string; searchInfo: string; alt: string | null; caption: string | null },
    now: Date,
  ): MediaProfile;
};
```

| 振る舞い | 処理 |
| --- | --- |
| `create` | `MediaName.fromFileName(fileName)` を名称にし、検索情報を空、共有する説明を未設定にした管理情報を返す |
| `describe` | `MediaName`、`MediaSearchInfo`、`SharedDescription` を生成して置き換える。生成した値がすべて現在の値と等しければ、渡された管理情報をそのまま返す（[index.md](index.md) の「変更の無い変更」）。値を変えた結果は、`version` を 1 進め、`updatedAt` を `now` にする |

不変条件。

- `MediaProfile` は、`Media` 一件に一つ
- 管理情報の変更は、公開範囲、使用箇所の説明、公開表示を変えない。使用箇所の説明は、使う側が持つ

### MediaRecord

メディア一件の `Media` と、画像ではその `MediaDerivatives` の組。メディアの種類と派生形式の記録の有無の組み合わせを、型で一つにする。

```ts
type MediaRecord =
  | Readonly<{ kind: "image"; media: ImageMedia; derivatives: MediaDerivatives }>
  | Readonly<{ kind: "document"; media: DocumentMedia }>;

const MediaRecord: {
  of(media: Media | null, derivatives: MediaDerivatives | null): MediaRecord | null;
  renditions(record: MediaRecord): MediaRenditions;
};
```

- `of` は、画像の `Media` と、同じ `MediaId` の `MediaDerivatives` から `image` を、文書の `Media` と `null` から `document` を返す。それ以外の組（`media` が `null`、画像に `null`、文書に派生形式の記録、`MediaId` の違う組）は `null`。別々に読んだ二つの間にメディアの削除が確定した場合が当たる
- 一件のメディアについて別々に読んだ `Media`、`MediaProfile`、派生形式の記録が揃わない場合（`MediaRecord.of` が `null` を返す、または `Media` があり `MediaProfile` が `null`）は、メディアが無い場合と同じに扱う。派生形式の記録だけを読む操作でも、画像の `Media` を読んだ後に派生形式の記録が `null` なら同じ
- `MediaRecord` を受ける振る舞い（`renditions`、`MediaListFilter.matches`、`StorageReservation.release`、`MediaDelivery.decide`）は、組み合わせごとの分岐を持たない

#### 配信できる画像

```ts
type ImageRendition = Readonly<{
  object: MediaObjectKey;
  format: ImageFormat | DerivativeFormat;
  width: number;
  height: number;
}>;

type MediaRenditions =
  | Readonly<{
      kind: "image";
      original: ImageRendition;
      derivatives: ReadonlyMap<DerivativeKey, ImageRendition>;
    }>
  | Readonly<{ kind: "document"; original: Readonly<{ object: MediaObjectKey; format: DocumentFormat }> }>;
```

`renditions` は、取得結果に載せる元のファイルと派生形式を返す。画像では配信できる画像（`kind` が `image`）、文書では元のファイル（`kind` が `document`）。

- `original` は、`variant` が `original` の実体の位置と、`file` の形式、幅、高さ
- `derivatives` は、`MediaDerivatives.generated` が返す派生形式を載せる。`format` は生成に使った定義の `output.format`、`width` と `height` は生成済みの派生形式が持つ値
- 配信先の種類（`publicUrl` と `previewUrl` のどちらを使うか）は、`MediaDelivery.linkFor` が決める

### MediaDerivatives

派生形式の記録。画像一件の、派生形式ごとの状態を持つ。`MediaId` で識別する。

フィールド。

```ts
type MediaDerivatives = Readonly<{
  mediaId: MediaId;
  source: ImageDimensions;
  states: ReadonlyMap<DerivativeKey, DerivativeState>;
  version: number;
  createdAt: Date;
  updatedAt: Date;
}>;
```

| フィールド | 制約 |
| --- | --- |
| `mediaId` | 必須。その画像の `Media` の `id`。生成後に変わらない |
| `source` | 必須。元の画像の幅と高さ。`Media` の `file` の値。生成後に変わらない |
| `states` | 派生形式のキーごとの状態。作成時は `create` が定める |
| `version` | 楽観ロックの番号。作成時は 0。状態を変える振る舞いごとに 1 進む |

振る舞い。

```ts
type Unchanged = Readonly<{ changed: false }>;

type DerivativesChange<E = never> =
  | Unchanged
  | (WithEventDrafts<MediaDerivatives, E> & Readonly<{ changed: true }>);

type DefinitionsChange =
  | Unchanged
  | (WithEventDrafts<MediaDerivatives, never> &
      Readonly<{ changed: true; started: readonly DerivativeKey[]; discarded: readonly DerivativeObjectKey[] }>);

type DerivativeCompletion =
  | Readonly<{ outcome: "completed"; derivatives: MediaDerivatives; superseded: readonly DerivativeObjectKey[] }>
  | Readonly<{ outcome: "already_completed" }>
  | Readonly<{ outcome: "stale" }>;

const MediaDerivatives: {
  create(media: ImageMedia, now: Date): WithEventDrafts<MediaDerivatives, MediaRegisteredEvent>;

  applyDefinitions(
    derivatives: MediaDerivatives,
    candidates: ReadonlyMap<DerivativeKey, DerivativeTarget>,
    now: Date,
  ): DefinitionsChange;
  requestGeneration(
    derivatives: MediaDerivatives,
    keys: readonly DerivativeKey[],
  ): readonly EventDraft<MediaDerivativeRegenerationRequestedEvent>[];
  claim(derivatives: MediaDerivatives | null, object: DerivativeObjectKey): DerivativeTarget | null;
  holds(derivatives: MediaDerivatives | null, object: DerivativeObjectKey): boolean;
  complete(
    derivatives: MediaDerivatives | null,
    key: DerivativeKey,
    target: DerivativeTarget,
    now: Date,
  ): DerivativeCompletion;
  fail(
    derivatives: MediaDerivatives | null,
    key: DerivativeKey,
    target: DerivativeTarget,
    failure: DerivativeFailure,
    now: Date,
  ): DerivativesChange;
  retry(
    derivatives: MediaDerivatives,
    key: DerivativeKey,
    now: Date,
  ): WithEventDrafts<MediaDerivatives, MediaDerivativeRegenerationRequestedEvent>;
  restartStalled(
    derivatives: MediaDerivatives,
    now: Date,
  ): DerivativesChange<MediaDerivativeRegenerationRequestedEvent>;

  generated(derivatives: MediaDerivatives): ReadonlyMap<DerivativeKey, GeneratedDerivative>;
  statuses(
    derivatives: MediaDerivatives,
    definitions: readonly ImageDerivative[],
  ): ReadonlyMap<DerivativeKey, DerivativeStatus>;
};
```

| 振る舞い | 処理 |
| --- | --- |
| `create` | `states` が空の派生形式の記録を返す。`version` は 0。`media.registered` を発行する |
| `applyDefinitions` | 「定義の適用」が定める |
| `requestGeneration` | 「生成を起こすイベント」が定める |
| `claim` | 「生成の引き受け」が定める |
| `holds` | 「生成の引き受け」が定める |
| `complete` | 「生成の完了」が定める |
| `fail` | `key` の状態が `generating` で、その生成の対象の世代が `target.generation` と一致する場合にだけ、状態を `failed` にする。`target` と `previous` を引き継ぎ、`failure` と `failedAt`（`now`）を持つ。それ以外（`derivatives` が `null`、状態が無い、`ready` または `failed`、世代が一致しない）は `Unchanged`。その世代を派生形式の記録が指しているかどうかは、`holds` が返す |
| `retry` | `key` の派生形式が `failed` でなければ `BusinessRuleError`（`DerivativeNotFailed`）。`key` の状態が無い場合も同じ。`failed` なら、失敗の理由（`stalled` を含む）によらず、同じ生成の対象と `previous` を持ち、`requestedAt` が `now`、`restarts` が 0 の `generating` にし、`media.derivative_regeneration_requested` を発行する |
| `restartStalled` | 「止まった生成のやり直し」が定める |
| `generated` | キーごとに、配信できる生成済みの派生形式を返す。`ready` なら `current`、`generating` と `failed` なら `previous`。`previous` が `null` のキーは含めない |
| `statuses` | 「生成状況」が定める |

`retry` の結果、`changed` が真の結果、`completed` の `derivatives` は、`version` を 1 進め、`updatedAt` を `now` にする。`Unchanged` は、値もイベントも持たない。呼び出し側は、渡した値を保存しない。

#### 定義の適用

`applyDefinitions` は、`states` を、渡された候補（`candidates`）の定義の組に合わせる。候補は、現行スキーマの `derivatives` の定義ごとに一つで、その定義の `key` をキーにした、定義とユースケースが `IdGenerator.generate` で作った世代の組（`DerivativeTarget.propose`）。候補ごとに、候補のキーの状態に、次を行う。

| 現在の状態 | 候補の定義との関係 | 結果 |
| --- | --- | --- |
| そのキーの状態が無い | 定義が追加された | 候補を生成の対象にした `generating`（`previous` は `null`） |
| `generating` または `failed` | 生成の対象の定義と等しい | 変えない |
| `ready` | `current` の定義と等しい | 変えない |
| `ready` | `current` の定義と異なる | 候補を生成の対象にした `generating`。`previous` は `current` |
| `generating` または `failed` | 生成の対象の定義と異なる | 候補を生成の対象にした `generating`。`previous` は引き継ぐ |

- 新しく `generating` にした状態は、`requestedAt` が `now`、`restarts` が 0
- 候補に無いキーの状態は取り除く。取り除いた状態が持つ生成済みの派生形式（`current`、`previous`）と、生成の対象（`generating`、`failed` の `target`）の `DerivativeObjectKey` を `discarded` に入れる
- 生成の対象を候補に置き換えた `generating` と `failed` の状態は、置き換える前の生成の対象の `DerivativeObjectKey` を `discarded` に入れる。前の処理が、その世代の実体を保管済みであり得る。実体が無い位置の消去は成功する
- `discarded` は、この変更の後に、どの状態も指さない実体（「ObjectRemoval」）
- 定義の等しさは、`definition` の値の一致で決める
- 新しい生成の対象の世代は、候補の世代。変えなかったキーの候補の世代は、使わない。振る舞いは、世代の値を作らない
- 状態が一つも変わらなければ、`Unchanged` を返す
- `started` は、新しく `generating` にしたキー。キーの昇順。変えなかったキーと、取り除いたキーを含まない。`applyDefinitions` はイベントを発行しない。生成を起こすイベントは、ユースケースが `requestGeneration` で作る
- 候補は、現行スキーマの `derivatives` の定義の全体について渡す

#### 生成状況

`statuses` は、`definitions`（現行スキーマの `derivatives` の定義の全体）の定義ごとに、その `key` の生成状況を返す。

| そのキーの状態 | 生成状況 |
| --- | --- |
| 無い | `generating` |
| 状態の定義（`generating` と `failed` は `target.definition`、`ready` は `current.definition`）が、その定義から `key` を除いた値と異なる | `generating` |
| それ以外 | 状態の `status` |

- 上の二つの `generating` は、「定義の適用」が `generating` にする状態。適用の前から生成中として示す
- `definitions` に無いキーの状態は、結果に含めない。`definitions` が空なら、結果は空

#### 生成を起こすイベント

`requestGeneration` は、`keys` のうち、状態が `generating` のキーごとに、そのキーと生成の対象の世代を指す `media.derivative_regeneration_requested` のドラフトを返す。状態を変えない。

- 返すドラフトは、キーの昇順で先頭から `DerivativePolicy.maxRequestsPerChange` 件まで。それを超えるキーは、イベントなしで `generating` のまま残り、「止まった生成のやり直し」が生成を始める。一つの派生形式の記録の変更が発行するイベントの上限は、ここだけが適用する
- このイベントのドラフトを作るのは、この振る舞いだけ。`retry` は、`generating` にしたキーをこの振る舞いに渡して作ったドラフトを返す。`restartStalled` は、始め直す候補のキーをこの振る舞いに渡し、ドラフトが返ったキーだけを始め直す。上限の値を、ほかの振る舞いとユースケースは使わない
- 定義の適用の後は、`started` を渡す

#### 生成の引き受け

`claim` は、生成を起こすイベントが指す派生形式（キーと世代）を、いま生成するかどうかを返す。`object` は、イベントのペイロード（メディア、キー、世代）から作る、その派生形式の実体の位置（`DerivativeObjectKey`）。`derivatives` は、そのメディアの派生形式の記録。メディアが無い場合は `null` を渡す。

- そのキーの状態が `generating` で、生成の対象の世代が `object` の世代と一致する場合にだけ、その生成の対象を返す
- それ以外（`derivatives` が `null`、そのキーの状態が無い、`ready` または `failed`、世代が一致しない）は `null`。配送の重複と順序の逆転で届く

`holds` は、派生形式の記録が `object` の位置を指しているかどうかを返す。

- そのキーの状態の生成の対象（`generating` と `failed` の `target`）の世代、または `generated` が返すそのキーの生成済みの派生形式の世代が、`object` の世代と一致すれば真
- `derivatives` が `null` の場合、そのキーの状態が無い場合、世代がどれとも一致しない場合は偽。偽になった世代は、以後も真に戻らない（不変条件）
- 生成の対象が指す位置には、実体があることも、無いこともある。`StorageReservation.release` が、この結果を使う

#### 生成の完了

`complete` は、`key` と `target.generation` の派生形式の実体を保管した後に呼ぶ。`MediaStorage.commit` が `OBJECT_EXISTS` で拒否した生成も、実体を保管した生成として呼ぶ。同じ位置の実体は、同じ元の画像と同じ定義から作る（「MediaObjectKey と MediaObjectRef」）。その実体の扱いを、三つの結果のどれか一つで返す。上から順に確かめる。

| `key` の状態 | 結果 | 保管した実体 |
| --- | --- | --- |
| `generating` または `failed` で、その生成の対象の世代が `target.generation` と一致する | `completed`。状態を `ready` にした `derivatives` を返す。`current` は、`target`、`DerivativeGeometry.outputSize(derivatives.source, target.definition.size)` の幅・高さ、`generatedAt`（`now`）を持つ。`superseded` は、`previous` があればその `DerivativeObjectKey`、無ければ空 | `current` が指す |
| `generated` が返すそのキーの生成済みの派生形式の世代が、`target.generation` と一致する（`ready` の `current`、または `generating` と `failed` の `previous`） | `already_completed`。何も変えない | 生成済みの派生形式が既に指している。消去しない |
| それ以外（`derivatives` が `null`、状態が無い、世代がどれとも一致しない） | `stale`。何も変えない | どの状態も指さず、以後も指さない |

- 同じ世代の生成を複数の処理が並行して行っても、結果は一つに決まる。先に保存した処理は `completed`、後の処理は `already_completed` を得る。保管済みの実体は、どちらの結果でも残る
- `failed` の派生形式も、同じ生成の対象の結果が後から届けば `ready` になる。止まった生成として失敗にした後に届いた結果を含む
- `superseded` は、この変更の後に、どの状態も指さない実体（「ObjectRemoval」）
- 三つの結果は、`holds` と対応する。`completed` と `already_completed` の後の派生形式の記録は、その世代の位置を指す。`stale` の世代は指さない。保管した実体を残すか消去待ちにするかは、その処理の保管の予約を取り除く確定が、`StorageReservation.release` で決める

#### 止まった生成のやり直し

```ts
const DerivativePolicy: {
  stalledAfterMs: 600_000;
  maxRestarts: 3;
  maxRequestsPerChange: 100;
  isStalled(state: DerivativeState, now: Date): boolean;
  stalledBefore(now: Date): Date;
};
```

- `isStalled` は、状態が `generating` で、`requestedAt` から `now` までが `stalledAfterMs` 以上なら真。ちょうど `stalledAfterMs` を含む
- `stalledBefore` は、`now` の `stalledAfterMs` 前の日時を返す。`MediaDerivativesRepository.findStalled` に渡す。ユースケースは期間を計算しない
- `restartStalled` は、`isStalled` が真の派生形式ごとに、次のどちらかを行う。該当が無ければ `Unchanged`
  - `restarts` が `maxRestarts` 未満: 始め直す候補にする。候補のキーを `requestGeneration` に渡し、ドラフトが返ったキーだけを始め直して、そのドラフトを返す。始め直しは、`requestedAt` を `now` に置き換え、`restarts` を 1 進める。生成の対象と `previous` は変えない。ドラフトが返らなかった候補は変えず、次の呼び出しが対象にする
  - `restarts` が `maxRestarts`: 状態を `failed` にする。`failure.reason` は `stalled`、`failedAt` は `now`。生成の対象と `previous` を引き継ぐ。イベントを発行しない
- 一つの派生形式は、最初の生成と `maxRestarts` 回の始め直しまで生成を試みる。イベントなしで `generating` になった派生形式は、最初の始め直しが最初の生成になる。どの試みも `stalledAfterMs` のあいだに結果を記録しなければ、`failed` になる。生成中のまま残らない
- 始め直しは `requestedAt` を進めるので、始め直した派生形式は、次の `stalledAfterMs` のあいだ `isStalled` が偽になる
- 始め直しは世代を変えない。前の試みの結果が後から届いても、`complete` と `fail` は、同じ生成の対象の結果として扱う

不変条件。

- `MediaDerivatives` は、画像の `Media` 一件に一つ。文書には無い
- 派生形式は公開範囲を持たない。元のメディアの `visibility` に従う
- 世代は使い回さない。新しい生成の対象の世代は、ユースケースがその都度 `IdGenerator.generate` で作った値で、それまでに作ったどの値とも異なる。状態から取り除いたキーを再び加えるときも、バックアップから復旧した後も同じ（「復旧の置き換え」）
- `ready` の `current` と、`previous` は、実体がある派生形式だけを指す。`discarded` と `superseded` は、変更の後の状態が指す実体を含まない
- 生成中のまま進まない派生形式は、`restartStalled` が `failed` にする

ライフサイクル。

```text
generating ─ complete → ready ─ applyDefinitions（定義の変更） → generating
           ├ fail → failed ─ retry → generating（restarts 0）
           │                └ complete（同じ生成の対象の結果が後から届く） → ready
           ├ restartStalled（restarts < maxRestarts） → generating（restarts + 1）
           └ restartStalled（restarts = maxRestarts） → failed（stalled）
```

- 生成: 画像の登録（「トランザクション境界」）
- 削除: `MediaRepository.delete`

### ドメインイベント

```ts
type MediaDerivativeRegenerationRequestedEvent = DomainEventBase<
  "media.derivative_regeneration_requested",
  Readonly<{ mediaId: MediaId; derivative: DerivativeKey; generation: DerivativeGeneration }>
>;

type MediaRegisteredEvent = DomainEventBase<"media.registered", Readonly<{ mediaId: MediaId }>>;

type MediaEvent = MediaRegisteredEvent | MediaDerivativeRegenerationRequestedEvent;
```

`aggregateId` は `mediaId`。`media.registered` は、画像を登録したことを表すイベントで、`MediaDerivatives.create` がドラフトを作る。消費者が、現行スキーマの派生形式の定義を、その画像の派生形式の記録に当てる。`media.derivative_regeneration_requested` は、一つの派生形式の、一つの世代の生成を起こすイベントで、`MediaDerivatives.requestGeneration` がドラフトを作る。`derivative` と `generation` は、発行の時点の生成の対象のキーと世代。

派生形式の実体を保管するのは、`media.derivative_regeneration_requested` の消費者だけ。同じイベントが再び届いても、`claim`、`complete`、`fail` が、現在の状態から結果を決める。消費者の UnitOfWork の分け方は、[../usecases/media.md](../usecases/media.md) が定める。

## 値オブジェクト

ブランド付きの型は、`create` だけが生成する。`MediaUrl` は `create` を持たず、`MediaDeliveryLinks` だけが生成する。等価性は、断りが無い限り、すべてのフィールドの値の一致で決める。文字数は Unicode のコードポイントで数える。

利用者の入力から値オブジェクトを作るときの違反は `ValidationError`（項目ごとの理由）になる。下の表の「違反」は、その理由のコード。文字数の上限（`MediaName`、`MediaSearchInfo`、`MediaDescriptionText` と、`MediaListFilter` の `text`）は技術設計の層の値で、違反の拒否が利用者に示すのは、項目と理由のコード。

| 名前 | 型 | バリデーションルール |
| --- | --- | --- |
| `MediaName` | `string`（ブランド） | 前後の空白を除いて 1 文字以上 200 文字以下。違反は `InvalidMediaName` |
| `MediaSearchInfo` | `string`（ブランド） | 前後の空白を除いて 0 文字以上 1000 文字以下。違反は `InvalidMediaSearchInfo` |
| `MediaDescriptionText` | `string`（ブランド） | 前後の空白を除いて 1 文字以上 1000 文字以下。違反は `InvalidMediaDescription` |
| `SharedDescription` | オブジェクト | 下の定義 |
| `MediaFormat` | 文字列リテラルの直和型 | 下の定義 |
| `MediaFile` | 直和型 | 下の定義 |
| `DerivativeFormat` | 文字列リテラルの直和型 | 下の定義 |
| `MediaAcceptance` | 文字列リテラルの直和型 | `any` または `image`。定義は「MediaAdmission」 |
| `MediaLimits` | 定数 | 下の定義 |
| `ImageDimensions` | `Readonly<{ width: number; height: number }>` | `width`、`height` は 1 以上の整数 |
| `DerivativeGeneration` | `string`（ブランド） | 空でない。新しい世代の値は `IdGenerator.generate` が作り、生成の対象に割り当てるのは `DerivativeTarget.propose` だけ。保存した記録、イベントのペイロード、配信先から得た世代は、記録した値を `create` で戻す |
| `DerivativeState` | 直和型 | 下の定義 |
| `MediaObjectKey`、`MediaObjectRef` | オブジェクト | 下の定義 |
| `StoragePolicy` | 定数 | 下の定義 |
| `StorageTarget` | 直和型 | 下の定義 |
| `StorageReservationId` | `string`（ブランド） | 空でない。値は `IdGenerator.generate` が作る |
| `StorageReservation` | オブジェクト | 下の定義 |
| `ObjectRemoval` | オブジェクト | 下の定義 |
| `MediaUrl` | `string`（ブランド） | 空でない。`MediaDeliveryLinks` の `publicUrl`、`previewUrl`、`parse` だけが生成する。メディアのワークスペースの ID を含む |
| `MediaAccess` | 直和型 | 下の定義 |
| `MediaListFilter` | オブジェクト | 下の定義 |

### MediaName

```ts
const MediaName: {
  create(value: string): MediaName;
  fromFileName(fileName: string): MediaName;
};
```

`fromFileName` は、ファイル名の前後の空白を除き、先頭から 200 文字までを名称にする。結果が空なら `media` を名称にする。

### SharedDescription

```ts
type SharedDescription = Readonly<{
  alt: MediaDescriptionText | null;
  caption: MediaDescriptionText | null;
}>;

const SharedDescription: {
  create(input: { alt: string | null; caption: string | null }): SharedDescription;
  empty(): SharedDescription;
};
```

`create` は、`null` と、前後の空白を除いて空になる文字列を、未設定（`null`）にする。

### MediaFormat と MediaFile

```ts
type ImageFormat = (typeof MediaLimits.formats.image)[number];
type DocumentFormat = (typeof MediaLimits.formats.document)[number];
type MediaFormat = ImageFormat | DocumentFormat;
type DerivativeFormat = ImageDerivative["output"]["format"];

type ImageFile = Readonly<{
  kind: "image";
  format: ImageFormat;
  byteSize: number;
  width: number;
  height: number;
}>;
type DocumentFile = Readonly<{ kind: "document"; format: DocumentFormat; byteSize: number }>;
type MediaFile = ImageFile | DocumentFile;

const MediaFormat: {
  detect(head: Uint8Array): MediaFormat | null;
  contentType(format: MediaFormat | DerivativeFormat): string;
};
```

- `ImageFormat` は、登録で受け付け、`ImageProcessor` が読み取る画像の形式。`DerivativeFormat` は、派生形式の出力の画像形式で、語彙は Schema の `ImageDerivative` が定める。`DerivativeFormat` は `ImageFormat` に含まれる（出力した画像を、`MediaFormat.detect` と `ImageProcessor.measure` で読める）
- `byteSize` は 1 以上の整数（バイト）。`width`、`height` は 1 以上の整数（ピクセル）で、画像が持つ向きの情報を適用した後の値
- `detect` は、ファイルの先頭 32 バイトまでの内容から形式を判定する。ファイル名と、送信側が申告した種類を使わない。判定は次の表に従い、どれにも当たらなければ `null`

| 形式 | 先頭の内容 |
| --- | --- |
| `jpeg` | `FF D8 FF` |
| `png` | `89 50 4E 47 0D 0A 1A 0A` |
| `webp` | 0 バイト目から `RIFF`、8 バイト目から `WEBP` |
| `avif` | 4 バイト目から `ftyp`、8 バイト目から `avif` または `avis` |
| `pdf` | `%PDF-` |

- `contentType` は、配信で示す種類を返す。`image/jpeg`、`image/png`、`image/webp`、`image/avif`、`application/pdf`

### MediaLimits

```ts
const MediaLimits: {
  maxBytes: 20_971_520;
  formats: Readonly<{ image: readonly ["jpeg", "png", "webp", "avif"]; document: readonly ["pdf"] }>;
  maxPublishedAtOnce: 1000;
};
```

- 対応形式は `formats` の画像と文書の形式（MEDIA-01）。`ImageFormat` と `DocumentFormat` は、この定数から導く。容量の上限 `maxBytes` は形式によらず、値は APP-12 の「メディアの容量」。登録の拒否は、この値を理由とともに示す
- `maxPublishedAtOnce` は、一つの操作が新たに公開するメディアの数の上限で、一度の `MediaRepository.publishMany` が受ける ID の数の上限。値は APP-12 の「一つの操作で新たに公開できるメディアの数」。技術設計の層では、この定数だけが値を持つ
- Media のリポジトリのメソッドが一度に受ける ID の数と `limit` の上限は、各メソッドの契約の行が定める。上限を超える ID を読む使う側は、上限の数ごとに分けて呼ぶ

### DerivativeState

```ts
type DerivativeTarget = Readonly<{
  definition: Omit<ImageDerivative, "key">;
  generation: DerivativeGeneration;
}>;

const DerivativeTarget: {
  propose(definition: ImageDerivative, generation: string): DerivativeTarget;
};

type GeneratedDerivative = DerivativeTarget &
  Readonly<{ width: number; height: number; generatedAt: Date }>;

type DerivativeFailure = Readonly<{
  reason: "source_unreadable" | "output_unsupported" | "processing_failed" | "stalled";
}>;

type DerivativeState =
  | Readonly<{
      status: "generating";
      target: DerivativeTarget;
      requestedAt: Date;
      restarts: number;
      previous: GeneratedDerivative | null;
    }>
  | Readonly<{ status: "ready"; current: GeneratedDerivative }>
  | Readonly<{
      status: "failed";
      target: DerivativeTarget;
      failure: DerivativeFailure;
      failedAt: Date;
      previous: GeneratedDerivative | null;
    }>;

type DerivativeStatus = DerivativeState["status"];
```

- `generation` は、生成の対象を見分ける値。順序を持たず、等しいかどうかだけを比べる
- `propose` は、定義と、ユースケースが `IdGenerator.generate` で作った値から、生成の対象の候補を作る。候補の `definition` は、定義から `key` を除いた値。派生形式のキーは、`MediaDerivatives` の `states` のキーと、候補のキーだけが持つ。世代の値は不変条件（「MediaDerivatives」）に従う
- `DerivativeTarget.definition` は、生成に使う定義。`GeneratedDerivative` の `definition` は、生成に使った定義
- `previous` は、定義の変更の前に生成済みだった派生形式。`generating` と `failed` の間、配信を続ける
- `requestedAt` は、生成を始めた、または最後に始め直した日時。`restarts` は、定期ジョブが始め直した回数で、0 以上 `DerivativePolicy.maxRestarts` 以下の整数。`generating` を離れると、回数は持たない
- `DerivativeFailure.reason` は、失敗の理由。派生形式一つの生成が失敗したときの理由は、この表だけが決める。生成するユースケースは、失敗した段に当たる理由を `MediaDerivatives.fail` に渡す

| 理由 | 当たる失敗 |
| --- | --- |
| `source_unreadable` | 元の画像の実体が無い（`MediaStorage.get` が `null`）。`ImageProcessor.transform` が `source_unreadable` を返した |
| `output_unsupported` | `ImageProcessor.transform` が `output_unsupported` を返した |
| `processing_failed` | `ImageProcessor.transform` が `processing_failed` を返した。`MediaStorage` の `get`、`put`、`commit` が失敗した（打ち切りを含む。`commit` の `OBJECT_EXISTS` は「生成の完了」） |
| `stalled` | `DerivativePolicy.maxRestarts` 回始め直しても生成が終わらない（「止まった生成のやり直し」） |

### StoragePolicy

```ts
const StoragePolicy: {
  callTimeoutMs: 300_000;
  attemptLifetimeMs: 86_400_000;
};
```

- `callTimeoutMs` は、`MediaStorage` のメソッドの一回の呼び出しが、結果を待つ時間の上限で、5 分。始めてからこの時間の内に終わらない呼び出しは、打ち切られて失敗する
- `attemptLifetimeMs` は、確定していない試みの内容を、メディアの保管先が保つ期限で、24 時間
- 打ち切った呼び出しの書き込み・消去と、期限を過ぎた試みの扱いは、`MediaStorage` の契約が定める

### MediaObjectKey と MediaObjectRef

```ts
type MediaObjectKey = Readonly<{
  mediaId: MediaId;
  variant:
    | Readonly<{ kind: "original" }>
    | Readonly<{ kind: "derivative"; key: DerivativeKey; generation: DerivativeGeneration }>;
}>;

type DerivativeObjectKey = MediaObjectKey &
  Readonly<{
    variant: Readonly<{ kind: "derivative"; key: DerivativeKey; generation: DerivativeGeneration }>;
  }>;

type MediaObjectRef = MediaObjectKey & Readonly<{ workspaceId: WorkspaceId }>;

const MediaObjectKey: {
  original(mediaId: MediaId): MediaObjectKey;
  derivative(mediaId: MediaId, key: DerivativeKey, generation: DerivativeGeneration): DerivativeObjectKey;
};

const MediaObjectRef: {
  of(workspaceId: WorkspaceId, object: MediaObjectKey): MediaObjectRef;
};
```

どちらも実体の位置を表す。

- `MediaObjectKey` は、区画の中の実体の位置。集約の振る舞いと、区画の記録（保管の予約、消去待ち）が使う。ワークスペースの ID を持たない
- `MediaObjectRef` は、区画の外のポート（`MediaStorage`、`MediaDeliveryLinks`）に渡す実体の位置。`MediaObjectKey` に、その実体を持つ区画のワークスペースの ID を添えた値
- 派生形式の実体の位置は、ワークスペース、メディア、派生形式のキー、世代から一意に決まる。同じ位置は同じ実体を指し、指す実体の内容は変わらない。実体を保管する前でも、位置は決まる。定義を変えた派生形式は世代が変わるので、別の位置になる。`MediaId` と世代は使い回さない（`IdGenerator` の値）ので、一度あるメディアと生成の対象を指した位置は、ほかのメディアと生成の対象を指さない

### StorageTarget

```ts
type StorageTarget =
  | Readonly<{ kind: "media"; mediaId: MediaId }>
  | Readonly<{ kind: "object"; object: DerivativeObjectKey }>;

const StorageTarget: {
  key(target: StorageTarget): string;
};
```

実体の範囲。保管の予約と消去待ちが指す。

| `kind` | 範囲の実体 |
| --- | --- |
| `media` | そのメディアの元のファイルと、すべての派生形式 |
| `object` | 派生形式一つの、一つの世代 |

- 元のファイルを含む範囲は、`media` だけ。元のファイルは、メディアの登録が保管し、メディアの削除が、派生形式とともに消去する
- `key` は、`target` を一意に表す文字列を返す。`media` は `media:{mediaId}`、`object` は `object:{mediaId}/{key}/{generation}`
- 等価性は `key` の一致

### StorageReservation

```ts
type StorageReservation = Readonly<{
  id: StorageReservationId;
  target: StorageTarget;
  expiresAt: Date;
}>;

const StorageReservation: {
  lifetimeMs: 3_600_000;
  reserve(params: { id: string; target: StorageTarget }, now: Date): StorageReservation;
  isCollectable(reservation: StorageReservation, now: Date): boolean;
  release(
    reservation: StorageReservation,
    record: MediaRecord | null,
    now: Date,
  ): readonly ObjectRemoval[];
};
```

保管の予約の記録。実体を保管する処理（メディアの登録、派生形式の生成）が、保管を始める前に確定し、処理の結果の確定または後始末で取り除く。使い方は「実体と記録の整合」が定める。

- `reserve` は、`expiresAt` を `now` の 1 時間後（`lifetimeMs`）にした記録を返す
- 等価性は `id` の一致。同じ `target` を指す予約は複数あり得る
- `isCollectable` は、`now` が `expiresAt` 以降（同じ日時を含む）なら真。回収できる条件は、これだけ。期限は、止まった処理の予約を回収し始める時期だけを決める。保管と確定の可否は、期限に依らない
- `release` は、予約を取り除く確定が、同じ UnitOfWork で記録する消去待ちを返す。`record` は、その UnitOfWork が確定する時点の、`target` のメディアの記録。同じ UnitOfWork で読んだ値に、同じ UnitOfWork で書く値（`insert` する `Media` と `MediaDerivatives`、`save` する派生形式の記録）を反映した値を `MediaRecord.of` に通したもの

| `target` | 記録が範囲の実体を指している | `release` の結果 |
| --- | --- | --- |
| `media` | `record` が `null` でない | 空 |
| `media` | `record` が `null` | `target` の消去待ち（`requestedAt` は `now`）一件 |
| `object` | `record` が `image` で、`MediaDerivatives.holds(record.derivatives, object)` が真 | 空 |
| `object` | それ以外 | `target` の消去待ち一件 |

- `release` が空を返すのは、予約を取り除いた後も、`Media` または派生形式の記録が範囲の実体を指す場合だけ
- `release` の判定を確定の時点まで保つ方法は、「実体と記録の整合」が定める

### ObjectRemoval

```ts
type ObjectRemoval = Readonly<{
  target: StorageTarget;
  requestedAt: Date;
}>;

type QueuedRemoval = Readonly<{
  removal: ObjectRemoval;
  ticket: string;
}>;

const ObjectRemoval: {
  ofMedia(mediaId: MediaId, now: Date): ObjectRemoval;
  ofObjects(objects: readonly DerivativeObjectKey[], now: Date): readonly ObjectRemoval[];
};
```

消去待ちの記録。メディアの保管先からの消去を待つ、実体の範囲を指す。

| `target` の `kind` | 消去するもの | 記録する変更 |
| --- | --- | --- |
| `media` | そのメディアの元のファイルとすべての派生形式の実体 | メディアの削除（`Media.removal`）、登録の予約の `StorageReservation.release` |
| `object` | 派生形式一つの、一つの世代の実体 | `applyDefinitions` の `discarded`、`complete` の `superseded`、派生形式の生成の予約の `StorageReservation.release` |

- 等価性は `StorageTarget.key` の一致。同じ `target` の記録は一つ
- 記録は、消去を待つ状態だけを持つ。記録したものは、必ず消去する
- 記録が指すのは、`Media` も `MediaDerivatives` も指さない範囲。記録した範囲は、以後も `Media` と `MediaDerivatives` の指す対象の外にある（「MediaObjectKey と MediaObjectRef」）
- `QueuedRemoval` は、`ObjectRemovalRepository` が返す、記録済みの消去待ち。`ticket` は、記録のたびにポートが新しく割り当てる不透明な値（`ObjectRemovalRepository.complete`）
- `requestedAt` は、消去の順を決めるためだけに使う
- 消去の UnitOfWork の分け方は [../usecases/media.md](../usecases/media.md) の「消去待ちの消去」が定める

### MediaAccess

```ts
type MediaAccess =
  | Readonly<{ kind: "anonymous" }>
  | Readonly<{ kind: "preview"; expiresAt: Date }>
  | Readonly<{ kind: "granted" }>;

const MediaAccess: {
  resolve(input: {
    granted: boolean;
    previewGrant: Readonly<{ expiresAt: Date }> | null;
  }): MediaAccess;
};
```

配信の要求が示した資格。`anonymous` は資格を示さない要求、`preview` はプレビュー用の配信先が含む期限付きの許可、`granted` は、ユースケースが確かめた、期限を持たない資格の要求。

`resolve` は、一つの要求の資格を、次の優先順で一つに決める。

1. `granted` が真なら `granted`
2. `previewGrant` があれば、その `expiresAt` の `preview`。期限は判定しない。`MediaDelivery.decide` が `now` と比べる
3. それ以外は `anonymous`

- `previewGrant` は、`MediaDeliveryLinks.resolve` が返した値
- `granted` は、要求のアカウントが、配信先から取り出したワークスペース（`ResolvedMediaLink` の `ref.workspaceId`）のメンバーとして認可されたかどうか。確かめ方は [index.md](index.md) の「認可」に従う
- `granted` は、プレビュー用の配信先の許可より先に見る。期限を過ぎたプレビュー用の配信先も、`granted` の要求には配信する

### MediaListFilter

```ts
type MediaListFilter = Readonly<{
  text: string | null;
  derivativeStatuses: readonly DerivativeStatus[] | null;
  definitions: readonly ImageDerivative[];
}>;

const MediaListFilter: {
  create(
    input: { text: string | null; derivativeStatuses: readonly string[] | null },
    definitions: readonly ImageDerivative[],
  ): MediaListFilter;
  matches(filter: MediaListFilter, profile: MediaProfile, record: MediaRecord): boolean;
};
```

| 条件 | 意味 |
| --- | --- |
| `text` | 前後の空白を除いた文字列が、`profile` の `name` または `searchInfo` に部分一致する。英字の大文字と小文字を区別しない。`null` と、空白を除いて空になる文字列は絞り込まない |
| `derivativeStatuses` | `record` が `image` で、`MediaDerivatives.statuses(record.derivatives, filter.definitions)` の値のいずれかが `derivativeStatuses` にある。`document` は一致しない。`null` は絞り込まない |

- `definitions` は、ユースケースが渡す現行スキーマの `derivatives` の定義の全体

- `create` は、200 文字を超える `text` と、生成状況の語彙にない値を `ValidationError`（理由 `InvalidMediaFilter`）とする
- `matches` は、メディアが条件をすべて満たせば真

### エラーコード

```ts
type MediaErrorCode =
  // ValidationError の理由
  | "InvalidMediaName"
  | "InvalidMediaSearchInfo"
  | "InvalidMediaDescription"
  | "InvalidMediaFilter"
  | "MediaRejected"
  // BusinessRuleError
  | "MediaNotImage"
  | "DerivativeNotFailed"
  // ConflictError の理由
  | "MEDIA_IN_USE"
  | "MEDIA_IN_PUBLIC_USE"
  | "MEDIA_NOT_FOUND"
  | "MEDIA_NOT_PUBLIC"
  | "OBJECT_EXISTS"
  | "ATTEMPT_NOT_FOUND"
  | "RESERVATION_NOT_FOUND"
  // UnavailableError の理由
  | "STORAGE_CAPACITY_EXCEEDED"
  // SystemError の理由
  | "STORAGE_TIMED_OUT"
  | "IMAGE_PROCESSING_TIMED_OUT";
```

| コード | 種類 | 意味 |
| --- | --- | --- |
| `InvalidMediaName`、`InvalidMediaSearchInfo`、`InvalidMediaDescription` | `ValidationError` | 名称、検索情報、説明が規則を満たさない |
| `InvalidMediaFilter` | `ValidationError` | 一覧の絞り込みの条件が規則を満たさない |
| `MediaRejected` | `ValidationError` | 登録するファイルを `MediaAdmission.examine` が拒否した。`MediaRejection` を添える |
| `MediaNotImage` | `BusinessRuleError` | 画像でないメディアへの、派生形式の操作 |
| `DerivativeNotFailed` | `BusinessRuleError` | 失敗していない派生形式の、生成のやり直し |
| `MEDIA_IN_USE` | `ConflictError` | UnitOfWork の確定の時点の拒否。条件は `MediaRepository.delete` の契約が定める |
| `MEDIA_IN_PUBLIC_USE` | `ConflictError` | UnitOfWork の確定の時点の拒否。条件は `MediaRepository.save` の契約が定める |
| `MEDIA_NOT_FOUND` | `ConflictError` | UnitOfWork の確定の時点の拒否。条件は `MediaUsageRepository` の `replaceMany`・`promoteStaged` と、`MediaRepository.publishMany` の契約が定める |
| `MEDIA_NOT_PUBLIC` | `ConflictError` | UnitOfWork の確定の時点の拒否。条件は `MediaUsageRepository` の `replaceMany`・`promoteStaged` の契約が定める |
| `OBJECT_EXISTS`、`ATTEMPT_NOT_FOUND` | `ConflictError` | 実体の確定（`MediaStorage.commit`）の拒否。条件は `MediaStorage.commit` の契約が定める |
| `RESERVATION_NOT_FOUND` | `ConflictError` | 保管の予約が無い。保管の予約の取り除きの拒否（条件は `StorageReservationRepository.settle` の契約が定める）と、予約の確認で予約が無かった処理（「実体と記録の整合」）の結果 |
| `STORAGE_CAPACITY_EXCEEDED` | `UnavailableError` | メディアの保管先の容量の不足。登録を受け付けられない理由として、利用者に示す。条件は `MediaStorage.put` の契約が定める |
| `STORAGE_TIMED_OUT` | `SystemError` | `MediaStorage` の呼び出しの打ち切り。条件は `MediaStorage` の契約が定める |
| `IMAGE_PROCESSING_TIMED_OUT` | `SystemError` | `ImageProcessor.measure` の打ち切り。条件は `ImageProcessor` の契約が定める |

共通の理由（`STALE_VERSION`、`ALREADY_EXISTS`、`WORKSPACE_NOT_ACTIVE`、`INVALID_QUERY`、`ACTOR_NOT_ALLOWED`、`PERMISSION_MISSING`）は [common.md](common.md) が定める。

## ドメインサービス

### MediaAdmission

責務: 登録するファイルが、対応形式、容量の上限、受け付ける種類を満たすかを確かめ、登録時の記録を求める。

```ts
type MediaAcceptance = "any" | "image";

type MediaRejection = Readonly<{
  reason: "unsupported_format" | "too_large" | "not_image" | "unreadable_image";
  limits: typeof MediaLimits;
}>;

type AdmissionDecision =
  | Readonly<{ admitted: true; file: MediaFile }>
  | Readonly<{ admitted: false; rejection: MediaRejection }>;

const MediaAdmission: {
  measurable(bytes: Uint8Array): ImageFormat | null;
  examine(input: {
    bytes: Uint8Array;
    accepts: MediaAcceptance;
    measurement: MeasureResult | null;
  }): AdmissionDecision;
};
```

どちらも純粋な関数で、ポートを呼ばない。

- `measurable` は、幅・高さを測るファイルの画像形式を返す。`bytes` の長さが `MediaLimits.maxBytes` 以下で、`MediaFormat.detect` が画像の形式を返したファイルは、その形式。それ以外は `null`
- `accepts` は、登録を受け付けるメディアの種類。`any` は対応形式のすべて、`image` は画像の形式だけを受け付ける
- `measurement` は、`measurable` が形式を返したファイルでは `ImageProcessor.measure(bytes, 形式)` の結果、ほかは `null`

`examine` は、次の表の上から順に確かめ、最初に当てはまった行の結果を返す。

| 条件 | 結果 |
| --- | --- |
| `bytes` の長さが `MediaLimits.maxBytes` を超える | 拒否。理由 `too_large` |
| `MediaFormat.detect` が `null` を返す。長さ 0 のファイルを含む | 拒否。理由 `unsupported_format` |
| `accepts` が `image` で、文書の形式 | 拒否。理由 `not_image` |
| 画像の形式で、`measurement` が `null`、または `readable` が偽 | 拒否。理由 `unreadable_image` |
| 画像の形式 | `ImageFile`。`byteSize` は `bytes` の長さ、幅・高さは `measurement` の値 |
| 文書の形式 | `DocumentFile`。`byteSize` は `bytes` の長さ |

依存するポート: 無し。

### DerivativeGeometry

責務: 元の画像の幅・高さと派生形式の定義から、生成する画像の幅・高さを求める。

```ts
const DerivativeGeometry: {
  outputSize(original: ImageDimensions, size: ImageDerivative["size"]): ImageDimensions;
};
```

元の画像の幅を W、高さを H とする。

| `fit` | 幅 | 高さ |
| --- | --- | --- |
| `width` | `size.width` | H × `size.width` ÷ W |
| `height` | W × `size.height` ÷ H | `size.height` |
| `contain` | W × s | H × s。s は `size.width` ÷ W と `size.height` ÷ H の小さい方 |
| `cover` | `size.width` | `size.height` |

- `cover` は、縦横比を保って枠を満たす大きさにした画像を、中央で枠の大きさに切り抜く。ほかの `fit` は切り抜かない
- 計算した値は、小数点以下を四捨五入し、1 未満を 1 にする
- 元の画像より大きい定義では、拡大した大きさを返す
- 同じ引数から同じ結果を返す。`MediaDerivatives.complete` と `ImageProcessor.transform` は、この結果を使う

依存するポート: 無し。

### MediaDelivery

責務: 配信の要求に実体を返すかどうかと、取得結果に載せる配信先の種類を決める。

```ts
type DownstreamRetention =
  | Readonly<{ kind: "long"; maxAgeSeconds: number }>
  | Readonly<{ kind: "none" }>;

type DeliveryDecision =
  | Readonly<{
      outcome: "serve";
      object: MediaObjectKey;
      contentType: string;
      retention: DownstreamRetention;
    }>
  | Readonly<{ outcome: "deny" }>;

type MediaLinkKind =
  | Readonly<{ kind: "public" }>
  | Readonly<{ kind: "preview"; expiresAt: Date }>;

const MediaDelivery: {
  previewLinkLifetimeMs: 3_600_000;
  downstreamMaxAgeSeconds: 31_536_000;
  decide(
    input: {
      record: MediaRecord | null;
      workspaceAvailable: boolean;
      object: MediaObjectKey;
      access: MediaAccess;
    },
    now: Date,
  ): DeliveryDecision;
  linkFor(media: Media, audience: "published" | "preview" | "member", now: Date): MediaLinkKind;
};
```

`decide` は次の順に確かめる。

1. `record` が `null`、または `workspaceAvailable` が偽なら `deny`
2. `object.variant` が `derivative` の場合、`record` が `image` でない、または `MediaDerivatives.generated(record.derivatives)` に、同じキーと世代の生成済みの派生形式が無ければ `deny`
3. `record.media` が `public` なら、`retention` を `long`（`maxAgeSeconds` は `downstreamMaxAgeSeconds`）にした `serve`
4. `record.media` が `private` の場合、`access` が `granted`、または `expiresAt` が `now` より後の `preview` なら、`retention` を `none` にした `serve`。それ以外は `deny`

- `contentType` は、元のファイルでは `file.format`、派生形式では生成に使った定義の `output.format` の `MediaFormat.contentType`。配信で示す実体の種類は、この値だけで決まる
- `object` は、配信先から取り出した実体の位置（`ResolvedMediaLink` の `ref`）の、区画の中の部分。`record` は、ユースケースが、その位置のワークスペースの区画から `object.mediaId` のメディアを読み、`MediaRecord.of` に通した値
- `workspaceAvailable` は、メディアのワークスペースが利用できるかどうか。求め方は [index.md](index.md) の「利用できるワークスペース」に従う
- `deny` は、どの理由でも同じ値。メディアが無い場合、非公開のメディアに資格が足りない場合、ワークスペースが利用できない場合は、同じ結果になる
- 配信の入口は、要求のたびに `decide` を通す。入口と、入口までの CMS の経路は、どの応答も、その要求の `decide` の結果から作る

`retention` は、配信の入口の先（表示アプリ、その CDN、ブラウザ）が、返した内容を保持してよいかどうか。Media は、保持の方針までを決める。応答での表し方は、配信の入口（[../usecases/app-api.md](../usecases/app-api.md) の「メディアの配信」）が定める。

| `retention` | 当たる配信 | 意味 |
| --- | --- | --- |
| `long` | 公開のメディア | 配信の入口の先が、内容を `maxAgeSeconds` の間、保持してよい。値は `downstreamMaxAgeSeconds` で、APP-12 の「公開のメディアの応答が示す、内容を保持してよい期間」。その間、内容が変わらないものとして扱ってよい |
| `none` | 非公開のメディア | 配信の入口の先は、内容を保持しない |

- 実体の位置が指す内容は変わらない（「MediaObjectKey と MediaObjectRef」）ので、先が保持した内容は、同じ配信先の内容として古くならない。バックアップからの復旧の後も成り立つ（「復旧の置き換え」）
- 先が保持した公開のメディアの内容は、非公開化と削除の後も、保持の期限まで先に残り得る。回収は CMS の範囲の外にある

`linkFor` は、取得結果に載せる配信先の種類を返す。`audience` は、取得結果を受け取る相手で、`published` は公開内容の取得、`preview` は下書きを含むプレビューの取得、`member` はそのワークスペースのメンバーの取得（メディアの一覧と取得）。

- `audience` が `preview` で、メディアが `private` の場合にだけ、`expiresAt` を `now` の `previewLinkLifetimeMs` 後にした `preview` を返す。`previewLinkLifetimeMs` の値は、APP-12 の「プレビュー用の配信先の有効期間」。それ以外は `public` を返す
- `public` は `MediaDeliveryLinks.publicUrl`、`preview` は `MediaDeliveryLinks.previewUrl` の配信先を使う
- 公開内容の取得は、公開内容が使用するメディアが `public` であることを前提にする

依存するポート: 無し。

### MediaClosure

責務: ワークスペースの利用終了で、Media の記録と実体の消去を終えられるかどうかを決める。条件と、条件に要る記録は、この節だけが定める。

```ts
type MediaClosureRemaining = Readonly<{ storageReservations: number; objectRemovals: number }>;

const MediaClosure: {
  isSettled(remaining: MediaClosureRemaining): boolean;
};
```

- `isSettled` は、`storageReservations` と `objectRemovals` がともに 0 なら真。それ以外は偽。条件は、二つの記録の件数だけ。時間の経過を待たない
- `remaining` は、そのワークスペースの区画の `StorageReservationRepository.count` と `ObjectRemovalRepository.count` の結果。区画の消去（`Workspace` を `closed` にする保存まで）を終えた後に読む。保管の予約の件数を先に読み、消去待ちの件数をその後に読む
- 区画の消去を終えた区画で確定する UnitOfWork は [common.md](common.md) の「WriteScope」に、そのうち Media の UnitOfWork は [../usecases/media.md](../usecases/media.md) の「書き込みの UnitOfWork」に従う。そのため、残る予約は減るだけになり、予約が無くなった後は、消去待ちも減るだけになる。予約の件数を先に読むのは、このため
- 真は、回収・消去を待つ記録が残っていないことと、以後に保管を始める処理が、予約の確認を通らないことを表す。予約の確認を既に通った処理の試みは、その時点でメディアの保管先にあり、続く `MediaStorage.removeByWorkspace` が、その処理が確定した実体とともに取り除く（「実体と記録の整合」）。「利用終了の消去」が、実体の最後の消去の前に確かめる

依存するポート: 無し。

## 実体と記録の整合

メディアの保管先は区画の外にあり、区画の書き込みと、全部成功か全部反映されないかを共にしない。実体（`MediaStorage.commit` が確定した内容）と区画の記録が食い違ったまま残らないための規則は、この節だけが定める。UnitOfWork の分け方は、[../usecases/media.md](../usecases/media.md) の「実体の保管」と「消去待ちの消去」が定める。実体を保管する処理（メディアの登録、派生形式の生成）は、どれも同じ規則に従う。

メディアの保管先の実体は、そのワークスペースの区画の、次のいずれかの記録が指す。

| 記録 | 指す実体 |
| --- | --- |
| `Media` | そのメディアの元のファイル |
| `MediaDerivatives` | `holds` が真を返す派生形式（生成の対象と、生成済みの派生形式） |
| 保管の予約（`StorageReservation`） | `target` の範囲の実体 |
| 消去待ち（`ObjectRemoval`） | `target` の範囲の実体 |

実体を保管する処理と、記録を変える確定は、次の規則に従う。

- 保管は予約の後。実体を保管する処理は、保管する実体の範囲を指す保管の予約を、一つの UnitOfWork で確定してから、`MediaStorage.put` を始める。予約は処理ごとに作る
- 確定は予約の確認の後。処理は、`put` が試みを返した後、`MediaStorage.commit` の前に、自分の予約を `StorageReservationRepository.findById` で読む。`commit` を行うのは、予約が残っていた処理だけ。予約が無かった処理は、試みを `MediaStorage.discard` で取り除き、`commit` しない
- 予約を取り除く確定は、`release` を伴う。予約を取り除く UnitOfWork は、`StorageReservation.release` が返した消去待ちを、同じ UnitOfWork で記録する。予約を取り除くのは、保管を行った処理の結果の確定、その処理の後始末、回収のいずれか一つ（`StorageReservationRepository.settle` の契約）
- `release` が空を返した判定は、確定の時点まで保つ。UnitOfWork の中の読み取りは、確定の時点まで有効だと約束されない（[common.md](common.md) の「TransactionalRepository」の「読み取りと確定」）。`target` が `object` の予約で `release` が空を返した UnitOfWork は、判定の元の派生形式の記録を、読んだ `expectedVersion` で `MediaDerivativesRepository.save` に渡す。派生形式の記録を変えた UnitOfWork は変えた値を、変えない UnitOfWork は読んだ値をそのまま渡す。読んだ後に派生形式の記録が変わった、または削除された UnitOfWork は、`STALE_VERSION` で確定せず、予約は残る。結果の確定、後始末、回収のどれも同じ
- 記録が実体を指さなくなる変更は、同じ UnitOfWork で消去待ちを記録する。記録する変更は「ObjectRemoval」が定める
- 消去待ちは、消去の後に取り除く。`MediaStorage` の `remove` または `removeAll` が成功した後にだけ、`ObjectRemovalRepository.complete` を呼ぶ
- 消去の失敗は、消去待ちを記録した処理の結果を変えない。消去待ちを記録した確定の後に行う消去が失敗しても、消去待ちの記録は残る

この規則のもとで、途中で止まった処理、打ち切った呼び出し、並行した処理は、記録の指さない実体を残さない。利用終了の区画の消去が記録を削除した実体と、復旧が置き換える実体だけは、記録によらずに消去する（この節の末尾）。

- 予約が残る間に確定した実体は、その予約が指す。処理が止まれば、予約が残り、回収が `release` で扱いを決める
- 予約を取り除いた確定の時点で、その範囲の実体は、`Media` か派生形式の記録が指す（`release` が空）か、消去待ちが指す。`release` の判定は、確定の時点で成り立つ
  - `object` の予約で空を返した判定は、読んだ `expectedVersion` での保存が保つ。読んだ後に、その世代を生成の対象から外す変更（定義の変更、メディアの削除）が確定していれば、予約を取り除く確定は成り立たない
  - `media` の予約で `release` が空を返すのは、`Media` がある場合。`Media` を `insert` するのは、その予約の登録の確定だけで、同じ UnitOfWork で予約を取り除く。`Media` がある予約の `settle` は、ほかのどの UnitOfWork でも拒否される。`Media` が無いと読んだ後にその登録の確定が並行した場合は、`settle` が一方だけを確定させる
  - 消去待ちを返した判定（`holds` が偽）は、読んだ後も変わらない（「生成の引き受け」）
- 予約を取り除いた後に、`Media` と派生形式の記録が実体を指さなくなる変更は、消去待ちを伴う。その消去は、変更の確定の後に行うので、予約の確認を先に通った処理の試みと、その処理が確定した実体を取り除く
- 記録が実体を指さなくなる変更と、その消去を終えた後に、同じ位置へ `put` した処理は、予約の確認を通れば実体を確定し得る。その予約を取り除く確定は、変更の後の記録を読み、`release` が消去待ちを返す。読みが変更の前だった確定は、`STALE_VERSION` で成り立たない。処理が止まれば、予約が残り、回収が同じ判定を行う
- 予約の確認を通った後に、回収がその予約を取り除いた処理は、実体を確定し得る。その処理の試みは、予約の確認の時点で、内容の全体がメディアの保管先にある（`put` は、書き終えてから試みを返す）。回収が消去待ちを記録した場合、その消去は、先に確定した実体も、確定していない試みの内容も取り除き、取り除いた試みは確定できない（`MediaStorage` の契約）。回収が消去待ちを記録しなかった場合（`release` が空）、回収の確定の時点で派生形式の記録がその位置を指しており、確定した実体は、派生形式の記録が指す。予約の確認が回収の後になった処理は、`commit` を行わない
- 打ち切った `commit` が後から成り立った実体は、予約、派生形式の記録、消去待ちのいずれかが指す。その試みを取り除いた消去の後に成り立つ `commit` は、実体を作らない（`MediaStorage` の契約）
- 打ち切った消去が後から成り立っても、呼び出しの後に確定した実体は、記録が指すまま残る（`MediaStorage` の契約の「打ち切り」）
- 実体の整合は、時間に依らない（予約の期限の意味は「StorageReservation」）。`Clock.now` の進み方は、実体の整合に影響しない

確定していない試みの内容は、実体でなく、記録の指す対象にならない。消去が取り除かなかった試みの内容の終端は、`MediaStorage` の契約の「期限」が定める。

利用終了と復旧は、記録によらず、ワークスペースの実体をまとめて消去する。復旧の置き換えが行う実体の保管も、保管の予約を経ない。

- 利用終了の区画の消去は、消去待ちを記録せずに、`Media` と `MediaDerivatives` を削除する。その実体は、利用終了の完了が、`MediaClosure.isSettled` が真になった後に、`MediaStorage.removeByWorkspace` でまとめて消去する（「利用終了の消去」）。`closing` と `closed` の区画でも、保管の予約と消去待ちは、上の規則のとおりに扱う
- 復旧は、置き換えた区画のワークスペースの実体を、バックアップの内容に合わせる（「復旧の置き換え」）

## ポート

ポートは `packages/core/src/domain/media/ports/` に置く。どのポートもドメイン型を受け渡す。`MediaRepository`、`MediaProfileRepository`、`MediaDerivativesRepository`、`MediaUsageRepository`、`StorageReservationRepository`、`ObjectRemovalRepository` は、ワークスペースの区画のリポジトリで、[common.md](common.md) の「TransactionalRepository」の節が定める、すべてのリポジトリに共通の規則に従う。下の表は、それに加える契約と、上限の値、加えるメソッドの分類（読み取り、書き込み）だけを書く。

- どのリポジトリも、`Workspace` の状態を見ない。区画に `Workspace` が無くても、読み取りと書き込みの結果は変わらない
- `MediaRepository`、`MediaProfileRepository`、`MediaDerivativesRepository` は `TransactionalRepository` を継ぐ。`MediaUsageRepository`、`StorageReservationRepository`、`ObjectRemovalRepository` は、集約に数えない記録を保存し、`TransactionalRepository` を継がない
- `MediaStorage` は、対象のワークスペースを `ref` と引数で受ける

### MediaRepository

目的: メディアを保存し、ID と一覧の条件で引く。

```ts
interface MediaRepository extends TransactionalRepository<Media, MediaId> {
  // 読み取り
  findByIds(ids: readonly MediaId[]): Promise<readonly Media[]>;
  findPage(filter: MediaListFilter, pagination: Pagination): Promise<PaginationResult<Media>>;
  countByVisibility(): Promise<Readonly<{ public: number; private: number }>>;
  // 書き込み
  publishMany(ids: readonly MediaId[], now: Date): Promise<void>;
  deleteAll(limit: number): Promise<number>;
}
```

| メソッド | 契約 |
| --- | --- |
| `insert` | 共通の契約に従う |
| `findById` | 共通の契約に従う。ほかのワークスペースの区画のメディアは `null` |
| `save` | 共通の契約に従う。加えて、確定の時点で、公開使用（`MediaUsageRepository` の使用箇所のうち `public` が真のもの）があるメディアを `private` にする保存を、ポートが拒否する（`ConflictError`、理由 `MEDIA_IN_PUBLIC_USE`）。公開使用の記録と、そのメディアの非公開化が並行した場合、一方だけが確定する。記録の側が負けた場合は `MediaUsageRepository` が `MEDIA_NOT_PUBLIC` で、非公開化の側が負けた場合はこのメソッドが `MEDIA_IN_PUBLIC_USE` で拒否する |
| `publishMany` | メディアの公開の保存。渡した ID のメディアのうち、確定の時点で `private` のものを、`Media.publish(media, now)` の結果に置き換える。確定の時点で `public` のメディアは変えず、`version` も進めない。非公開から公開への一方向で、読んだときの `version` に依らない。`ExpectedVersion` を取らず、`STALE_VERSION` を返さない。同じメディアを新たに公開する二つの UnitOfWork が並行しても、どちらも確定し、メディアは `public` になる。この呼び出しが公開したメディアへの、前の `version` を期待する `save` と `delete` は、`STALE_VERSION` になる。`MediaProfile` と `MediaDerivatives` の保存とは競合しない。確定の時点でその区画に無いメディアの ID を含む場合は `ConflictError`（理由 `MEDIA_NOT_FOUND`）で、その UnitOfWork は確定しない。同じ ID が複数ある列は、一つとして扱う。ID の上限は `MediaLimits.maxPublishedAtOnce` 件。空の列は何もせずに成功する。件数によらず、書き込み一回に数える |
| `delete` | 共通の契約に従う。加えて、確定の時点で、使用箇所（`MediaUsageRepository` の使用箇所。`public` の値によらない）が残るメディアの削除を、ポートが拒否する（`ConflictError`、理由 `MEDIA_IN_USE`）。削除の確定と、そのメディアの使用箇所の記録の確定が並行した場合、一方だけが成功する。記録の側が負けた場合は `MediaUsageRepository` が `MEDIA_NOT_FOUND` で拒否する。そのメディアの `MediaProfile` と `MediaDerivatives` を、同時に削除する |
| `findByIds` | 渡した ID のうち、その区画に存在するメディアを返す。順序は保証しない。ID の上限は `Pagination.maxLimit`（[common.md](common.md)） 件 |
| `findPage` | その区画のメディアのうち、そのメディアの `MediaProfile` と、`MediaRecord.of` で組んだ記録について `MediaListFilter.matches` が真になるものを、登録日時の降順、同じ日時は ID の降順で返す |
| `countByVisibility` | その区画の、`public` と `private` のメディアの件数を返す |
| `deleteAll` | その区画のメディアを、その `MediaProfile` と `MediaDerivatives` とともに削除する。件数はメディアの数。使用箇所の記録と楽観ロックによらず削除する。実体は消去しない |

- `findPage` と並行した登録・削除は、ページをまたぐと重複または欠落が起こり得る
- 使用箇所と公開使用の有無の判定は、`MediaUsageRepository` の使用箇所だけから決まり、確定後の状態に対して行う。同じ UnitOfWork で使用箇所を取り除いたメディアは、削除と非公開化ができる。準備領域の使用箇所（`MediaUsageRepository.stage`）は、切り替えるまで判定に含めない

### MediaProfileRepository

目的: メディアの管理情報を保存し、`MediaId` で引く。

```ts
interface MediaProfileRepository extends Omit<TransactionalRepository<MediaProfile, MediaId>, "delete"> {
  // 読み取り
  findByIds(ids: readonly MediaId[]): Promise<readonly MediaProfile[]>;
}
```

| メソッド | 契約 |
| --- | --- |
| `insert` | 共通の契約に従う |
| `findById` | 共通の契約に従う。存在しないメディアは `null` |
| `save` | 共通の契約に従う。同じメディアへの並行した保存は、一方だけが成功する。`Media` と `MediaDerivatives` の保存とは競合しない |
| `findByIds` | 渡した ID のうち、その区画に存在する管理情報を返す。順序は保証しない。ID の上限は `Pagination.maxLimit`（[common.md](common.md)） 件 |

### MediaUsageRepository

目的: メディアの使用箇所を、使う側ごとに記録し、メディアごとに返す。

```ts
type MediaUse = Readonly<{ mediaId: MediaId; public: boolean }>;
type HolderUses = Readonly<{ holderId: string; uses: readonly MediaUse[] }>;
type MediaUsage = Readonly<{ holderId: string; public: boolean }>;
type MediaUsageStatus = "unused" | "in_use" | "in_public_use";

interface MediaUsageRepository {
  // 書き込み
  replaceMany(entries: readonly HolderUses[]): Promise<void>;
  stage(stagingId: string, entries: readonly HolderUses[]): Promise<void>;
  promoteStaged(stagingId: string): Promise<void>;
  discardStaged(stagingId: string): Promise<void>;
  deleteAll(limit: number): Promise<number>;
  // 読み取り
  findPageByMedia(mediaId: MediaId, pagination: Pagination): Promise<PaginationResult<MediaUsage>>;
  findStatusByMedia(mediaIds: readonly MediaId[]): Promise<ReadonlyMap<MediaId, MediaUsageStatus>>;
}
```

| メソッド | 分類 | 契約 |
| --- | --- | --- |
| `replaceMany` | 書き込み | `entries` の持ち主ごとに、その `holderId` の使用箇所を、`uses` の内容に置き換える。前の使用箇所のうち `uses` に無いものは無くなる。`uses` の中の同じ `mediaId` は一つの使用箇所にまとめ、`public` は、どれかが真なら真にする。空の `uses` は、その持ち主の使用箇所を消す。使用箇所を消す呼び出しは、空の `uses` を渡すこの形だけ。準備領域の内容は変えない。`entries` に無い持ち主の使用箇所は変えない。同じ `holderId` の要素が複数ある場合は、列の最後の要素を使う。確定の時点で、`uses` のメディアがその区画に存在しない場合は `ConflictError`（理由 `MEDIA_NOT_FOUND`）。ほかのワークスペースのメディアは、その区画に存在しないメディアに当たる。`public` が真の使用箇所のメディアが、確定の時点で `private` なら `ConflictError`（理由 `MEDIA_NOT_PUBLIC`）。一つでも拒否されれば、その UnitOfWork は確定しない。空の `entries` は何もせずに成功する。`entries` と `uses` の件数に上限は無い。件数によらず、書き込み一回に数える |
| `stage` | 書き込み | `stagingId` の準備領域に、`entries` の持ち主ごとの使用箇所を書く。同じ `stagingId` の同じ `holderId` の分が既にあれば置き換える。繰り返しても結果は変わらない。`uses` のまとめ方と、同じ `holderId` の要素が複数ある場合の扱いは `replaceMany` と同じ。空の `uses` の持ち主も、準備領域に書く。メディアの存在と公開範囲を確かめない。使用箇所を変えない。件数に上限は無く、件数によらず、書き込み一回に数える |
| `promoteStaged` | 書き込み | `stagingId` の準備領域にある持ち主ごとに、その `holderId` の使用箇所を、準備領域の `uses` の内容に置き換える。空の `uses` の持ち主の使用箇所は、すべて取り除く。準備領域に無い持ち主の使用箇所は変えない。切り替えは、呼び出した UnitOfWork の確定と、全部成功か全部反映されないかを共にし、件数によらず、書き込み一回に数える。確定の時点で、切り替えた後の使用箇所に `replaceMany` と同じ整合性を確かめる（`MEDIA_NOT_FOUND`、`MEDIA_NOT_PUBLIC`）。確定の後、その準備領域は無くなる。準備領域が無い `stagingId` は、何もせずに成功する |
| `discardStaged` | 書き込み | `stagingId` の準備領域を破棄する。無い場合も成功する。使用箇所を変えない。件数によらず、書き込み一回に数える。破棄した `stagingId` には、新しく `stage` できる |
| `deleteAll` | 書き込み | その区画の使用箇所と、すべての準備領域の内容を削除する。件数は、使用箇所（`holderId` と `mediaId` の組）と、準備領域の持ち主（`stagingId` と `holderId` の組）の数 |
| `findPageByMedia` | 読み取り | そのメディアの使用箇所を、`holderId` の文字列のコードポイントの昇順で返す。要素は、使う側の `holderId` と、その使用箇所の `public`。`count` は、そのメディアの使用箇所の総数。使用箇所が無いメディアと、存在しないメディアには、空の列と 0 を返す |
| `findStatusByMedia` | 読み取り | 渡した ID ごとに、使用の状況を返す。`public` が真の使用箇所が一つ以上あれば `in_public_use`、使用箇所があってどれも `public` が偽なら `in_use`、使用箇所が無ければ `unused`。結果は、渡した ID のすべてをキーに持つ。存在しないメディアは `unused`。ID の上限は `Pagination.maxLimit`（[common.md](common.md)） 件 |

- `holderId` は、使う側が形を定める不透明な識別子。空でない文字列で、ワークスペースの中で使う側を一意に表す。ポートは中身を解釈せず、渡された文字列をそのまま保存して返す。使う側が、`holderId` から持ち主と位置を求める
- 一意性・整合性: 使用箇所は、`holderId` と `mediaId` の組ごとに一つ。使用箇所が指すメディアが存在することと、公開使用のメディアが `public` であることを、このポートと `MediaRepository` が担保する。判定は確定後の状態に対して行う。同じ UnitOfWork の `MediaRepository.publishMany` が公開したメディアの公開使用の記録は確定する
- 準備領域: `stagingId` ごとに分かれ、別の `stagingId` の準備領域は互いに影響しない。`stagingId` は、使う側が形を定める不透明な識別子。空でない文字列で、ワークスペースの中で、使う側の準備の一回を一意に表す。ポートは中身を解釈しない。準備領域の内容は、切り替えるまで使用箇所でなく、`findPageByMedia` と `findStatusByMedia` に現れない
- 並行性: 楽観ロックを使わない。同じ `holderId` への並行した置き換え（`replaceMany`、`promoteStaged`）は、どちらも確定し、使用箇所は、後に確定した側の内容になる。使う側が、自分の集約の楽観ロックと書き込みの条件で順序を決める
- 可視性: 確定した使用箇所は、以後の読み取りに即座に現れる
- 読み取りと、削除・非公開化の保護（`MediaRepository` の `delete` と `save` の契約）は、同じ記録から決まる
- 書き込みは、使う側のユースケースが、自分の内容の保存と同じ UnitOfWork で呼ぶ。使う側のリポジトリの契約は、使用箇所を読み書きしない

### MediaDerivativesRepository

目的: 派生形式の記録を保存し、`MediaId`、区画の全体、止まった生成で引く。

```ts
interface MediaDerivativesRepository
  extends Omit<TransactionalRepository<MediaDerivatives, MediaId>, "delete"> {
  // 読み取り
  findByIds(ids: readonly MediaId[]): Promise<readonly MediaDerivatives[]>;
  scan(after: MediaId | null, limit: number): Promise<readonly Versioned<MediaDerivatives>[]>;
  findStalled(requestedBefore: Date, limit: number): Promise<readonly Versioned<MediaDerivatives>[]>;
}
```

| メソッド | 契約 |
| --- | --- |
| `insert` | 共通の契約に従う |
| `findById` | 共通の契約に従う。文書と、存在しないメディアは `null` |
| `save` | 共通の契約に従う。同じ画像への並行した保存（生成結果、定義の適用、やり直し）は、一方だけが成功する。`Media` と `MediaProfile` の保存とは競合しない。読んだ値をそのまま、読んだ `expectedVersion` で渡す保存は、保存済みの `version` が等しければ確定し、値と `version` を変えない。違えば `STALE_VERSION`。「実体と記録の整合」の、`release` の判定を保つ保存が使う |
| `findByIds` | 渡した ID のうち、その区画に存在する派生形式の記録を返す。文書の ID は結果に含めない。順序は保証しない。ID の上限は `Pagination.maxLimit`（[common.md](common.md)） 件。取得結果に載せる派生形式を求めるために使う |
| `scan` | その区画の派生形式の記録のうち、`mediaId` が `after` より大きいものを、読んだ時点の `version` を `expectedVersion` に添えて、`mediaId` の昇順で最大 `limit` 件返す。`after` が `null` なら先頭から返す。`limit` の上限は `Pagination.maxLimit`（[common.md](common.md)）。返す件数が `limit` より少なければ、残りは無い。走査の途中で加わった・変わった記録は、その時点の状態で返す |
| `findStalled` | その区画の派生形式の記録のうち、`generating` で `requestedAt` が `requestedBefore` 以前（同じ日時を含む）の派生形式を一つ以上持つものを、その `requestedAt` の最も古い値の昇順、同じ値は `mediaId` の昇順で、読んだ時点の `version` を `expectedVersion` に添えて、最大 `limit` 件返す。`limit` の上限は `Pagination.maxLimit`（[common.md](common.md)） |

### StorageReservationRepository

目的: 保管の予約を記録し、処理の結果の確定、後始末、回収で取り除く。

```ts
interface StorageReservationRepository {
  // 書き込み
  insert(reservation: StorageReservation): Promise<void>;
  settle(id: StorageReservationId): Promise<void>;
  // 読み取り
  findById(id: StorageReservationId): Promise<StorageReservation | null>;
  findCollectable(
    now: Date,
    after: StorageReservation | null,
    limit: number,
  ): Promise<readonly StorageReservation[]>;
  count(): Promise<number>;
}
```

| メソッド | 分類 | 契約 |
| --- | --- | --- |
| `insert` | 書き込み | 記録を保存する。`id` の一意性をポートが担保する。同じ `id` の記録が既にあれば `ConflictError`（理由 `ALREADY_EXISTS`）。同じ `target` の記録が既にあっても保存する |
| `settle` | 書き込み | その `id` の記録を取り除く。確定の時点で記録が無ければ `ConflictError`（理由 `RESERVATION_NOT_FOUND`）で、その UnitOfWork は確定しない。期限では拒否しない |
| `findById` | 読み取り | その `id` の記録を返す。無ければ `null`。期限によらない |
| `findCollectable` | 読み取り | その区画の、`StorageReservation.isCollectable(reservation, now)` が真の記録を、`expiresAt` の昇順で、最大 `limit` 件返す。`after` が `null` なら先頭から、記録なら、この並びでその記録より後のものを返す。`after` の記録が既に無くても、その位置より後を返す。返す件数が `limit` より少なければ、残りは無い。`limit` の上限は `BoundedDeletion.maxLimit`（[common.md](common.md)） |
| `count` | 読み取り | その区画の記録の件数を返す。期限によらない。「利用終了の消去」が使う |

- 並行性: 楽観ロックを使わない。同じ記録への並行した `settle` は、一方だけが確定し、もう一方は `ConflictError`（理由 `RESERVATION_NOT_FOUND`）になる。処理の結果の確定と、その予約の回収がともに成り立つことは無い。この担保はポートが行う
- 可視性: 確定した記録と、確定した `settle` は、以後の `findById`、`findCollectable`、`count` に即座に現れる
- 記録を取り除くのは `settle` だけ。ワークスペースの利用終了でも同じ

### ObjectRemovalRepository

目的: 消去待ちの記録（メディアの保管先からの消去を待つ、実体の範囲の記録）を保存し、消去を終えた記録を取り除く。

```ts
interface ObjectRemovalRepository {
  // 書き込み
  enqueue(removals: readonly ObjectRemoval[]): Promise<readonly QueuedRemoval[]>;
  complete(queued: QueuedRemoval): Promise<void>;
  // 読み取り
  findPending(after: ObjectRemoval | null, limit: number): Promise<readonly QueuedRemoval[]>;
  count(): Promise<number>;
}
```

| メソッド | 分類 | 契約 |
| --- | --- | --- |
| `enqueue` | 書き込み | 消去待ちを記録し、記録したものを、割り当てた `ticket` とともに返す。同じ `target` の記録は一つ。既に記録がある `target` への `enqueue` は、その記録の `requestedAt` を渡した値に置き換え、`ticket` を、それまでのどの値とも異なる新しい値にする。一度の呼び出しに同じ `target` が複数ある場合は、列の最後のものを記録し、その一件を返す。この一意性はポートが担保し、並行した `enqueue` でも記録は一つになる。空の列は何もせずに成功し、空の列を返す。件数によらず、書き込み一回に数える |
| `complete` | 書き込み | `queued.removal.target` の記録を、その `ticket` が `queued.ticket` と等しい場合にだけ削除する。`ticket` が違う記録（消去の間に改めて `enqueue` された記録）は、変えずに残して成功する。無い記録も成功する。実体の消去を終えた記録に使う |
| `findPending` | 読み取り | その区画の記録を、現在の `ticket` とともに、`requestedAt` の昇順、同じ日時は `StorageTarget.key` の文字列のコードポイントの昇順で、最大 `limit` 件返す。`after` が `null` なら先頭から、記録なら、この並びでその記録（`requestedAt` と `target`）より後のものを返す。`after` の記録が既に無くても、その位置より後を返す。返す件数が `limit` より少なければ、残りは無い。記録からの経過によらず、すべての記録が対象。`limit` の上限は `BoundedDeletion.maxLimit`（[common.md](common.md)） |
| `count` | 読み取り | その区画の記録の件数を返す。「利用終了の消去」が使う |

- 並行性: 楽観ロックを使わない。同じ `ticket` の並行した `complete` どうしは、どちらも成功する。同じ記録を二つの処理が消去しても、`MediaStorage.remove` と `removeAll` は無い実体に成功するので、結果は変わらない。`enqueue` と `complete` が並行した場合、`enqueue` が返した `ticket` の記録は、その `ticket` を渡した `complete` が確定するまで残る。`requestedAt` が等しい `enqueue` でも同じ
- 可視性: 確定した記録は、以後の `findPending` と `count` に即座に現れる
- 記録を取り除くのは、実体の消去を終えた `complete` だけ。ワークスペースの利用終了でも同じ

### MediaStorage

目的: 元のファイルと派生形式の実体を、メディアの保管先に保管し、読み出す。区画の外にあり、すべてのワークスペースの実体を、実体の位置（`MediaObjectRef`）の `workspaceId` で分けて持つ。

```ts
type MediaObject = Readonly<{ bytes: Uint8Array }>;
type StoredAttempt = Readonly<{ ref: MediaObjectRef; attemptId: string }>;

interface MediaStorage {
  put(ref: MediaObjectRef, object: MediaObject): Promise<StoredAttempt>;
  commit(attempt: StoredAttempt): Promise<void>;
  discard(attempt: StoredAttempt): Promise<void>;
  get(ref: MediaObjectRef): Promise<MediaObject | null>;
  remove(ref: MediaObjectRef): Promise<void>;
  removeAll(workspaceId: WorkspaceId, mediaId: MediaId): Promise<void>;
  removeByWorkspace(workspaceId: WorkspaceId, limit: number): Promise<number>;
}
```

書き込みは二段。`put` は、一回の保管の試みとして、内容をメディアの保管先に書く。`commit` は、試みの内容を、その `ref` の実体にする。確定していない試みの内容は、実体でない。実体になる経路は `commit` だけ。`attemptId` は、アダプターが試みごとに決める不透明な値。

| メソッド | 契約 |
| --- | --- |
| `put` | 内容（`bytes`）を、`ref` を指す新しい試みとしてメディアの保管先に書き、書き終えてから試みを返す。`ref` の実体と、同じ `ref` のほかの試みを変えない。既に実体がある `ref` への `put` も成功する。メディアの保管先が失敗を返した `put` は、試みを返さずに失敗する。打ち切った `put` は、試みを返さない。打ち切った `put` の書き込みは、後からメディアの保管先に届き得る。メディアの保管先の容量が足りなければ `UnavailableError`（理由 `STORAGE_CAPACITY_EXCEEDED`）で失敗し、試みを返さない |
| `commit` | 試みの内容を、その `ref` の実体にする。完了後の `get` は、`put` に渡した内容を返す。結果は、次の順に確かめて決める。確定済みで、その実体が残る試みの `commit` は、何も変えずに成功する。それ以外で、既に実体がある `ref` への `commit` は、試みの内容がメディアの保管先にあるかどうかと、内容が同じかどうかによらず、`ConflictError`（理由 `OBJECT_EXISTS`）で拒否し、保管済みの実体を変えず、その試みの内容を取り除く。実体が無い `ref` への、内容がメディアの保管先に無い試み（`discard`、`remove`、`removeAll`、`removeByWorkspace` が取り除いた試みと、期限を過ぎた試み）の `commit` は、`ConflictError`（理由 `ATTEMPT_NOT_FOUND`）で拒否し、実体を作らない。打ち切った `commit` は、後からメディアの保管先で成り立ち得る。その結果は、成り立つ時点に呼んだ `commit` と同じ |
| `discard` | 確定していない試みの内容を取り除く。取り除いた試みは `commit` できない。確定済みの試みと、既に無い試みへの呼び出しは、何も変えずに成功する。実体を取り除かない |
| `get` | `ref` が指す実体を返す。無ければ `null`。確定していない試みの内容は返さない |
| `remove` | `ref` が指す実体と、呼び出しの時点でメディアの保管先にある、その `ref` の確定していない試みの内容を取り除く。無くても成功する |
| `removeAll` | そのメディアの、元のファイルとすべての派生形式について、実体と、呼び出しの時点でメディアの保管先にある確定していない試みの内容を取り除く。無くても成功する |
| `removeByWorkspace` | そのワークスペースの実体と、呼び出しの時点でメディアの保管先にある確定していない試みの内容を、合わせて最大 `limit` 件取り除き、取り除いた件数を返す。実体一つと、試み一つの内容を、それぞれ一件に数える。返した件数が `limit` より少なければ、呼び出しの時点でメディアの保管先にあったものは残っていない。`limit` は 1 以上 `BoundedDeletion.maxLimit`（[common.md](common.md)）以下で、範囲外は `SystemError`（理由 `INVALID_QUERY`）。`Media` の記録が無い実体も対象にする。「利用終了の消去」の実体の消去と、「復旧の置き換え」だけが使う |

- 打ち切り: どのメソッドも、始めてから `StoragePolicy.callTimeoutMs` が過ぎても終わらない呼び出しを打ち切り、`SystemError`（理由 `STORAGE_TIMED_OUT`）で失敗する。打ち切った `get` は、実体を返さない。打ち切った `discard`、`remove`、`removeAll`、`removeByWorkspace` は、後からメディアの保管先で成り立ち得る。成り立つまでは、取り除いたものとして扱わない。呼び出し側は、同じ呼び出しをやり直す。どの消去も、何度呼んでも結果が変わらない。打ち切った消去が後から成り立つ場合に取り除くのは、呼び出しの時点でメディアの保管先にあった実体と試みの内容（その試みが後から確定した実体を含む）だけ。呼び出しの時点でメディアの保管先に無かった試みの内容と、その試みが確定した実体を取り除かない
- 確定していない試みの内容: `put` が試みを返した時点で、その内容の全体がメディアの保管先にある。確定していない試みの内容は、`get` に現れない。取得、配信、実体の有無の確認は、`get` だけを経る
- 消去が取り除く試み: `remove`、`removeAll`、`removeByWorkspace` が取り除くのは、対象の位置を指す試みのうち、呼び出しの時点で内容がメディアの保管先にあるもの。`put` が返した試みは、これに当たる。取り除いた試みは、以後 `commit` できない。呼び出しの時点でメディアの保管先に届いていない書き込みは、その消去の対象の外にある。届いた後は、確定していない試みの内容としてメディアの保管先にある
- 届いた内容: 試みを返さなかった `put`（失敗、打ち切り）の内容は、実体にならない。打ち切った `put` の書き込みが後からメディアの保管先に届いても、`commit` に渡す試みが無いので、確定できない。届いた内容は、`get` に現れず、その位置を対象にする次の `remove`、`removeAll`、`removeByWorkspace` が取り除く
- 期限: 確定していない試みの内容は、メディアの保管先に届いてから `StoragePolicy.attemptLifetimeMs` が過ぎた後、アダプターが捨てる。期限を過ぎた試みは、`commit` できず（`ATTEMPT_NOT_FOUND`）、その内容は `removeByWorkspace` の件数に入らない。期限の内の試みは、期限を理由に捨てない。実体は期限を持たない。消去が取り除かなかった試みの内容は、この期限切れで終わる
- 一意性・整合性: `ref` と実体の対応をポートが保つ。`Media` の記録との整合は、「実体と記録の整合」の規則が保つ。ポートは、渡された `ref` と試みだけから結果を決め、区画の記録（保管の予約を含む）を読まない。`commit` を呼ぶ処理の条件（予約の確認）は、「実体と記録の整合」が定める
- 並行性: 楽観ロックを使わない。異なる `ref` への並行した操作は互いに影響しない。同じ `ref` への並行した `put` は、どちらも成功し、別の試みになる。同じ `ref` への並行した `commit` は、一方だけが成功し、もう一方は `ConflictError`（理由 `OBJECT_EXISTS`）になる。同じ `ref` への `commit` と、`remove`・`removeAll`・`removeByWorkspace` のそれぞれが並行した場合の結果は、どちらかが先に終わった場合の結果のいずれかになる。`commit` が先なら実体は取り除かれ、消去が先なら `commit` は `ATTEMPT_NOT_FOUND` になる。どちらでも、消去の完了の後に、その試みの実体は無い。打ち切った `commit` が、その試みを取り除いた消去の後にメディアの保管先で成り立つ場合は、消去が先の場合に当たり、実体を作らない。`remove` した `ref` には、新しい試みを `put` して `commit` できる
- メディアの保管先に求めるもの: 契約は、試みの内容があり、`ref` に実体が無い場合にだけ成り立つ確定と、期限による試みの内容の破棄を、メディアの保管先に求める。打ち切った `put` の書き込みが、消去の後に届いてメディアの保管先に残ることを、契約は許す
- 可視性: 保管した実体を読み出す経路は `get` だけとする。メディアの保管先は、公開範囲によらず、実体を CMS の外から直接取得できる場所に置かない。公開のメディアも非公開のメディアも、配信は `MediaDelivery.decide` を経て `get` で読み出す
- 区画の外にあり、UnitOfWork に属さない。UnitOfWork が反映されなくても、確定した実体は残る
- 打ち切りの時間と、試みの期限は、アダプターを作るときに渡す。既定は `StoragePolicy.callTimeoutMs` と `StoragePolicy.attemptLifetimeMs`
- アダプターごとの検証の対象: 容量の不足、実体を CMS の外から直接取得できないこと、期限を過ぎた内容をメディアの保管先が実際に捨てる時期

### MediaDeliveryLinks

目的: 実体の配信先を求め、配信の要求が指す実体と、要求が含む許可を求める。

```ts
type ResolvedMediaLink = Readonly<{
  ref: MediaObjectRef;
  previewGrant: Readonly<{ expiresAt: Date }> | null;
}>;

interface MediaDeliveryLinks {
  publicUrl(ref: MediaObjectRef): MediaUrl;
  previewUrl(ref: MediaObjectRef, expiresAt: Date): Promise<MediaUrl>;
  parse(requested: string): MediaUrl | null;
  resolve(url: MediaUrl): Promise<ResolvedMediaLink>;
}
```

| メソッド | 契約 |
| --- | --- |
| `publicUrl` | CMS が管理する配信の、`ref` の配信先を返す。配信先は、`ref.workspaceId` を含む。同じ `ref` から常に同じ値を返す。許可を含まない。状態を保存しない |
| `previewUrl` | `ref` と `expiresAt` に結び付いた許可を含む配信先を返す。配信先は、`ref.workspaceId` を含む。返した配信先を `resolve` すると、その `ref` と、`expiresAt` を期限とする `previewGrant` が得られる。同じ `ref` と `expiresAt` から同じ値を返すことは約束しない。状態を保存しない |
| `parse` | 配信の要求の URL の全体（スキームとホストを含む）を、`MediaUrl` にする。`publicUrl` と `previewUrl` が返す配信先は、その配信先への要求の URL の全体と同じ文字列。このポートが発行する形で、`ref` を取り出せる文字列だけを `MediaUrl` として返す。それ以外は `null`。`publicUrl` と `previewUrl` が返した値と同じ文字列には、同じ値の `MediaUrl` を返す。許可の正しさと期限を確かめない。区画を読まず、状態を保存しない。例外を投げない |
| `resolve` | 配信先が指す `ref` を返す。`ref.workspaceId` は、配信先から取り出したワークスペースの ID。区画を読まない。`publicUrl` の結果には `previewGrant` を `null` で返す。`previewUrl` が返した値そのものには、発行時の `expiresAt` を `previewGrant` で返す。`previewUrl` が返した値と一文字でも違う配信先には、許可なし（`previewGrant` が `null`）を返す |

- `MediaUrl` を配信の要求から作る手段は `parse` だけ
- `resolve(publicUrl(ref))` と `resolve(await previewUrl(ref, t))` は、同じ `ref` を返す。後者の `previewGrant.expiresAt` は `t` と等しい
- `previewUrl` が返した値を一文字でも変えた文字列は、`parse` が `null` を返すか、`resolve` が `previewGrant` を `null` で返す。許可を持つ結果にならない
- 期限の判定は行わない。`MediaDelivery.decide` が `now` と比べる
- 配信の資格は、`resolve` の結果から `MediaAccess.resolve` が決める
- プレビュー用の配信先は、`ref` と `expiresAt` と、このポートが計算した許可から成る。トークンの値とセッションの値は、ポートの入力に無い
- 並行性と可視性: 状態を持たない。区画に結び付かず、UnitOfWork に属さない
- エラーケース: 許可の計算の失敗は `SystemError`
- アダプターごとの検証の対象: 許可を偽造できないこと（このポートを通さずに、有効な許可を含む配信先を作れないこと）と、許可の計算の失敗

### ImageProcessor

目的: 画像の幅・高さを求め、元の画像を派生形式へ変換する。

```ts
type MeasureResult =
  | Readonly<{ readable: true; width: number; height: number }>
  | Readonly<{ readable: false }>;

type TransformResult =
  | Readonly<{ ok: true; bytes: Uint8Array }>
  | Readonly<{ ok: false; reason: Exclude<DerivativeFailure["reason"], "stalled"> }>;

interface ImageProcessor {
  measure(bytes: Uint8Array, format: ImageFormat): Promise<MeasureResult>;
  transform(input: {
    source: Uint8Array;
    sourceFormat: ImageFormat;
    definition: DerivativeTarget["definition"];
    outputSize: ImageDimensions;
  }): Promise<TransformResult>;
}
```

| メソッド | 契約 |
| --- | --- |
| `measure` | 形式を内容から判定し、向きの情報を適用した後の幅と高さを返す。内容の形式が、申告した `format` と違う場合は `readable: false`。画像として読み取れない内容も `readable: false`。読み取れない内容では、例外を投げない。始めてから `ImageProcessingPolicy.callTimeoutMs` が過ぎても終わらない呼び出しは、打ち切って `SystemError`（理由 `IMAGE_PROCESSING_TIMED_OUT`）で失敗する |
| `transform` | 元の画像を、`definition.output` の画像形式で、幅と高さが引数の `outputSize` と一致する画像に変換する。出力の大きさは `outputSize` に従い、ポートは `definition.size` から大きさを求め直さない。向きの情報を適用してから変換する。切り抜きは「DerivativeGeometry」に従う。透過を持つ画像を `jpeg` にする変換は成功し、透過の部分を白で埋める。元の画像を変更しない。失敗は `ok: false` と理由で返し、例外を投げない。始めてから `ImageProcessingPolicy.callTimeoutMs` が過ぎても終わらない呼び出しは、打ち切って `processing_failed` を返す |

```ts
const ImageProcessingPolicy: {
  callTimeoutMs: 300_000;
};
```

`callTimeoutMs` は、`measure` と `transform` の一回の呼び出しが、結果を待つ時間の上限で、5 分。

`transform` の失敗の理由は、次の順に決まる。

1. `definition.output.format` が、アダプターが変換に対応する出力の画像形式でなければ `output_unsupported`。対応する出力の画像形式は、アダプターを作るときに渡す集合で、呼び出しによらず変わらない。アダプターは、`DerivativeFormat` のうち自分が変換できる形式の、どの部分集合を渡されても作れる
2. 内容の形式が `sourceFormat` と違う画像と、画像として読み取れない内容は、`measure` と同じく読み取れないものとして扱い、`source_unreadable`
3. それ以外の変換の失敗と、打ち切りは `processing_failed`

- 一意性・整合性と並行性: 状態を持たない。並行した呼び出しは互いに影響しない
- 可視性: UnitOfWork に属さない
- 入力の画像形式は、`ImageFormat` のすべてを読み取る。作るときに絞るのは、出力の画像形式だけ
- `outputSize` は、呼び出し側が `DerivativeGeometry.outputSize` で求めて渡す
- アダプターごとの検証の対象: `definition.output.quality` の反映、`cover` の切り抜きの位置、透過の部分を白で埋めること、出力の画素の向き、打ち切りのほかの理由で `processing_failed` になる変換の失敗
- 打ち切りの時間は、アダプターを作るときに渡す。既定は `ImageProcessingPolicy.callTimeoutMs`

## トランザクション境界

`UnitOfWorkProvider` の契約と、書く区画と確定の条件（`WriteScope`）は [common.md](common.md) が定める。UnitOfWork は `WorkspaceContext` の `mediaRepository`、`mediaProfileRepository`、`mediaDerivativesRepository`、`mediaUsageRepository`、`storageReservationRepository`、`objectRemovalRepository` を使う。

Media の不変条件が、一つの UnitOfWork で確定することを求める書き込みは、次の三つ。

- `Media`、`MediaProfile`、`MediaDerivatives` の保存と、その振る舞いが発行したドメインイベントの記録
- `Media` の `insert` と、その `MediaProfile` の `insert`。画像では、その `MediaDerivatives` の `insert` も
- 「実体と記録の整合」が同じ UnitOfWork に置く確定

Media のユースケースがどの UnitOfWork を、どの条件で開くかと、UnitOfWork の書き込みの回数は、[../usecases/media.md](../usecases/media.md) だけが定める。利用終了の段の UnitOfWork は「利用終了の消去」が定める。UnitOfWork と、UnitOfWork の外の実体の保管・消去との順序は、「実体と記録の整合」が定める。

## 復旧の置き換え

バックアップからの復旧（Installation の `BackupStore.restore`）が、メディアの保管先の実体をバックアップの内容に合わせるために呼ぶ手順。呼ぶのは Installation。この節は、バックアップが写す Media の範囲と、呼ぶメソッドと、その前提を公開する。

バックアップが写す Media の範囲は、次の二つだけをこの節が定める。Installation の `BackupStore` は、これを指す。

- バックアップが持つ実体: 区画の写しの `Media` の元のファイルと、写しの `MediaDerivatives` について `MediaDerivatives.generated` が返す派生形式の実体。生成の対象（`generating` と `failed` の `target`）の実体は含まない。その世代は、復旧の後に生成する（下の「復旧点で生成の対象だった世代」）
- 写しに含めない区画の記録: 保管の予約（`StorageReservation`）。区画の写しは保管の予約を含まず、置き換えた区画は保管の予約を持たない

| 対象のワークスペース | 呼ぶメソッド | 終わりの条件 |
| --- | --- | --- |
| 復旧で無くなるワークスペース（現在あり、バックアップに無い） | `MediaStorage.removeByWorkspace(workspaceId, limit)` | 返る件数が `limit` より少なくなる。`limit` は `BoundedDeletion.maxLimit` |
| バックアップにあるワークスペース | `MediaStorage.removeByWorkspace(workspaceId, limit)`。その後に、バックアップが持つそのワークスペースの実体を、`MediaStorage.put` と `commit` で保管する | 返る件数が `limit` より少なくなり、バックアップの実体をすべて保管し終える |

- 前提: そのワークスペースの区画の置き換え（無くなるワークスペースでは、区画の取り除き）を終えた後に呼ぶ
- 前提: 置き換えた区画の記録は「写しに含めない区画の記録」のとおり。復旧をまたいで動く処理は、次のとおり記録の指さない実体を残さない
  - 置き換えの前に予約の確認を済ませた処理の試みは、その時点でメディアの保管先にあり、後の `removeByWorkspace` が、その処理が確定した実体とともに取り除く
  - 置き換えの後に予約の確認を行う処理は、予約が区画に無いので、`commit` を行わない
  - 置き換えの後に始まる、結果の確定と後始末の UnitOfWork は、予約の `settle` が `RESERVATION_NOT_FOUND` で確定しない。取り除かれた実体を指す `Media` と派生形式の記録を作らない
- 前提: 復旧は、停止した設置で行う（HOST-08 の手順 1）。停止した設置の定期ジョブとイベントの消費者は、新しい処理を始めない。停止の前に始まった消去の呼び出しと、この手順の打ち切った `removeByWorkspace` は、後から成り立っても、バックアップの実体の保管が確定した実体を取り除かない（`MediaStorage` の契約の「打ち切り」）
- `removeByWorkspace` の後、そのワークスペースのメディアの保管先に残る実体は、置き換えた区画の記録が指すものだけになる。復旧点より後に保管した実体と、どの記録も指さない実体は、ここで無くなる。続けて保管した実体は、置き換えた区画の `Media` と `MediaDerivatives` が指す
- 写しの消去待ちが指す範囲は、復旧の後に実体が無いことがある。無い実体の消去は成功する
- 途中で止まった手順は、初めから呼び直す

復旧の後も、同じ配信先は、復旧の前と同じ内容だけを指す。配信先は、ワークスペース、メディア、元のファイルか派生形式かの別、派生形式ではキーと世代から決まる（`MediaDeliveryLinks.publicUrl`）。

- `MediaId` と世代は `IdGenerator.generate` の値。`IdGenerator` は状態を持たず、復旧は、作る値を戻さない。復旧の後に作る値は、復旧の前に作ったどの値とも異なる
- 復旧点より後に登録したメディアと、復旧点より後に割り当てた世代は、置き換えた区画の記録に無い。その配信先への要求は `deny` になり、その配信先は、以後も別の内容を指さない（「MediaObjectKey と MediaObjectRef」）
- 復旧点にあったメディアの元のファイルは、登録後に変わらない実体を、バックアップから戻す
- 復旧点で生成済みだった派生形式（`current`、`previous`）は、その実体を、バックアップから戻す
- 復旧点で生成の対象だった世代は、復旧の後に、同じ元の画像と、同じ定義（生成の対象の `definition`）から生成する。復旧の前にその世代を生成して配信していても、同じ配信先は、同じ元の画像と同じ定義の派生形式を指し、取得結果の画像形式・幅・高さと食い違わない

## 利用再開前の処理

バックアップから復旧した設置で、Media が利用の再開の前に行う書き込みは 0 件。公開するのは、利用再開前の確認の項目だけ。

- 手順の前提: 無い
- 残件数: 無い（常に 0）

利用再開前の確認に公開する項目。[common.md](common.md) の「各ドメインが公開する項目」の `ResumeItemDefinition` の列と、件数を求める関数で公開する。

```ts
type MediaResumeItemKind = "public_media" | "private_media";

const MediaResumeItems: {
  definitions: readonly ResumeItemDefinition[];
  counts(visibility: Readonly<{ public: number; private: number }>): readonly ResumeItem[];
};
```

| `kind` | 件数の求め方 | 単位 | `countsAsRemaining` | 確認に示す（`shownInReview`） |
| --- | --- | --- | --- | --- |
| `public_media` | `MediaRepository.countByVisibility()` の `public` | ワークスペース | 偽 | 真 |
| `private_media` | `MediaRepository.countByVisibility()` の `private` | ワークスペース | 偽 | 真 |

- `counts` は、名簿のワークスペースごとに、その区画の `countByVisibility()`（UnitOfWork の外）の結果を受けて、二つの項目を返す

- 二つの件数は足し合わせない。公開と非公開を分けて示す
- 復旧点で非公開だったメディアは、非公開のまま戻る
- 項目は、件数だけを示す

## 利用終了の消去

ワークスペースの利用終了で、Media の集約と記録、メディアの保管先の実体を消去する手順。呼ぶ順は [index.md](index.md) の「手順を呼ぶ順」と「区画をまたぐ手順」が持つ。この節は、呼ぶメソッドと、その前提を公開する。区画が `closing` であることは、呼び出し側が確かめる。

区画の消去。利用終了の区画の消去の中で呼ぶ。

| 呼ぶメソッド | UnitOfWork |
| --- | --- |
| `MediaUsageRepository.deleteAll(limit)` | 内。条件は `none` |
| `MediaRepository.deleteAll(limit)` | 内。条件は `none` |

- 前提: 利用終了の開始（`Workspace` を `closing` にする確定）の後に呼ぶ。その後に確定する UnitOfWork は [common.md](common.md) の「WriteScope」と [../usecases/media.md](../usecases/media.md) の「書き込みの UnitOfWork」に従うので、消去の後に `Media`、`MediaProfile`、`MediaDerivatives` は残らない
- 二つのメソッドは、どちらを先に呼んでもよい。`MediaRepository.deleteAll` は、使用箇所の記録によらず削除する。消去済みのメソッドは 0 を返す
- 保管の予約と、消去待ちの記録は、区画の消去の後も残り、「実体と記録の整合」の規則のとおりに扱う

実体の消去。利用終了の完了の中で呼ぶ。

| 呼ぶもの | UnitOfWork | 終わりの条件 |
| --- | --- | --- |
| 「MediaClosure」の二つの件数を読み、`MediaClosure.isSettled` に渡す | 外。読み取りだけ | `isSettled` が真を返す |
| `MediaStorage.removeByWorkspace(workspaceId, limit)` | 外 | 返る件数が `limit` より少なくなる。`limit` は `BoundedDeletion.maxLimit` |

- 前提: 区画の消去（すべてのドメインの消去と、`Workspace` を `closed` にする保存）を終えている
- 前提: `removeByWorkspace` は、`isSettled` が真を返した後にだけ呼ぶ。渡す値と、読む順は「MediaClosure」が定める
- `removeByWorkspace` は、区画の消去が `Media` の記録とともに指さなくした実体を、メディアの保管先にある確定していない試みの内容とともに消去する。`removeByWorkspace` を終えた後、そのワークスペースの実体は残らず、新たに確定しない（「MediaClosure」）
- 件数の読み取りは書き込まず、消去済みの `removeByWorkspace` は 0 を返す。呼び直す時期は、呼び出し側（Installation）が定める

終了前の確認に公開する項目。型は [common.md](common.md) の「各ドメインが公開する項目」の `DomainCount`（件数を持つ項目）と `DomainKind`（件数を持たない種類）。

| 種類 | 型 | `kind` | 内容 |
| --- | --- | --- | --- |
| 終了前の確認に示す件数（`items`） | `DomainCount` | `media` | メディアの件数。`MediaRepository.countByVisibility()` の `public` と `private` の和 |
| 利用終了で失われるデータの種類（`lostData`） | `DomainKind` | `media` | メディア（元のファイルと派生形式、管理用の名称、検索情報、共有する説明） |
| CMS の通常データの外に残る情報（`remainsOutside`） | `DomainKind` | `external_media_copy` | 配信の入口の先（表示アプリ、その CDN、ブラウザ）と、CMS の外が保存・複製したメディア。CMS は回収しない |

- 件数を持つ項目は `items` の `media` だけ。`lostData` と `remainsOutside` の項目は、件数を持たない
- 消去する資格情報（`erasedCredentials`）: 無い
- バックアップに含まれるメディアは、Installation のバックアップの保持が定める

## 状態確認の項目

設置の状態確認（Installation の `checkInstallationStatus`）に、Media が公開する項目: 無い。

- 失敗した派生形式は、そのワークスペースで、権限を持つメンバーが、メディアの生成状況で確かめ、生成をやり直す
- 保存容量の使用量と不足は、Installation が設置の資源から求める

## ユースケース（概要）

- メディアを登録する
- メディアを一覧・検索する
- メディアを取得する
- 管理用の名称・検索情報・共有する説明を編集する
- メディアを非公開化する
- メディアを削除する
- 失敗した派生形式の生成をやり直す
- 派生形式一つを生成する（`media.derivative_regeneration_requested` の消費者）
- 派生形式の定義を登録済みの画像に適用する（`schema.applied` と `media.registered` の消費者）
- 止まった派生形式の生成をやり直す（定期ジョブ）
- 回収できる保管の予約を回収する（定期ジョブ）
- 消去待ちの実体を消去する（定期ジョブ）
- メディアを配信する

各ユースケースが受け付ける実行者の種類と、必要な権限は [../usecases/media.md](../usecases/media.md) が定める。メディアを扱うユースケースのうち、どれが Media に属するかは、[index.md](index.md) の「ユースケースの所属」が定める。
