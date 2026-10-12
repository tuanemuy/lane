# Media のユースケース

メディアの登録、検索、説明の編集、非公開化、削除、派生形式の生成、配信を行う。型、振る舞い、ポートは [../domains/media.md](../domains/media.md) が、区画とドメインをまたぐ規則は [../domains/index.md](../domains/index.md) が、`WriteScope`、共通のポート、エラーの理由は [../domains/common.md](../domains/common.md) が定める。各ユースケースの振る舞いの結果（結果の種類ごとの条件、エラー、途中で止まったときに残る状態）は、`spec/testcases/media/` の、見出しの識別子と同じ名前のファイルが定める。

コードは `packages/core/src/application/media/` に置く。

メディアを扱うユースケースのうち、どれが Media に属するかは、[../domains/index.md](../domains/index.md) の「ユースケースの所属」が定める。メディアを削除するユースケースは、所属によらず「メディアの削除の手続き」を使う。ワークスペースの利用終了でのメディアの消去は、Installation のユースケースが、[../domains/media.md](../domains/media.md) の「利用終了の消去」の手順で行う。

Media のユースケースがどの UnitOfWork をどの条件で開くかは、このファイルだけが定める。実体の保管・消去と UnitOfWork の順序の規則は、[../domains/media.md](../domains/media.md) の「実体と記録の整合」が定める。

## 共通

区画の外のポート（`MediaStorage`、`MediaDeliveryLinks`）へ渡す実体の位置（`MediaObjectRef`）は、ユースケースが、区画の中の位置（`MediaObjectKey`）と、処理する区画のワークスペースの ID（入力の `WorkspaceId`、またはイベントの `workspaceId`）から、`MediaObjectRef.of` で作る。

### 認可

認可の手続きは、[../domains/index.md](../domains/index.md) の「認可」が定める。各ユースケースの「認可」は、受け付ける実行者の種類と、必要な権限と許可だけを書く。「トークン」は `appToken` と `memberToken` の両方を指す。「`member`」とだけ書いたユースケースは、メンバーであれば権限を必要としない。

### 書き込みの UnitOfWork

- Media の UnitOfWork は、対象のワークスペースの区画を書く。`UnitOfWorkProvider.run` に `{ kind: "workspace", workspaceId, condition }` を渡す。各ユースケースの「トランザクション境界」と、「実体の保管」「消去待ちの消去」の表は、`condition` を「条件: `active`」のように書く
- 条件 `none` の UnitOfWork は、保管の予約を取り除く UnitOfWork（「実体の保管」の後始末と、回収）と、消去を終えた記録を取り除く UnitOfWork（「消去待ちの消去」）だけ。含むものは「実体の保管」の表と「消去待ちの消去」が定める。どちらも、ドメインイベントを記録しない
- 条件 `schemaVersion` の UnitOfWork は、派生形式の定義を適用する UnitOfWork（「派生形式の定義を登録済みの画像に適用する」）だけ。`schemaVersion` には、UnitOfWork の前に `PartitionReader` で読んだ `WorkspaceSchema` のスキーマの版を渡す
- ほかの UnitOfWork の条件は `active`
- どのユースケースの UnitOfWork も、リポジトリの書き込みのメソッドの呼び出しは 4 回まで。一つの UnitOfWork が記録するイベントの件数の上限は、`MediaDerivatives.requestGeneration` が定める。どちらも、書き込みの回数（[../domains/common.md](../domains/common.md) の「UnitOfWorkProvider」）に収まる
- 区画の外のポートと、区画に結び付かないポートの扱いは、[../domains/common.md](../domains/common.md) の「UnitOfWorkProvider」に従う

### 消費者と定期ジョブ

処理を行えない対象に当たったときの扱いは、[../domains/index.md](../domains/index.md) の「イベントの消費者と定期ジョブ」に従う。区画の外に保管した実体の後始末は、「実体の保管」の後始末の UnitOfWork で行う。

イベントの消費者は、受けたイベントの `workspaceId` の区画で処理する。ワークスペースの状態は、その区画の `WorkspaceRepository.find()` で読む。

定期ジョブのユースケースは、対象のワークスペース（`WorkspaceId`）を入力に受け、その区画で処理する。名簿と、区画のワークスペースの状態を読まない。対象にするワークスペースと、呼ぶ順は、[../domains/index.md](../domains/index.md) の「定期ジョブ」が定める。

### 実体の保管

実体を保管するユースケース（「メディアを登録する」「派生形式一つを生成する」）は、[../domains/media.md](../domains/media.md) の「実体と記録の整合」の規則に従い、UnitOfWork を次のように分ける。各ユースケースは、予約の `target` と、結果の UnitOfWork が含む結果の保存だけを書く。

| UnitOfWork | 条件 | 含むもの |
| --- | --- | --- |
| 予約 | `active` | `StorageReservation.reserve`（`id` は `IdGenerator.generate`）の記録の `insert` だけ |
| 結果 | `active` | 処理の結果の保存、`Media` と `MediaDerivatives` の読み取りと `MediaRecord.of`、予約の `settle`、`StorageReservation.release` が返した消去待ちの `enqueue`、`release` の判定を保つ保存 |
| 後始末 | `none` | `Media` と `MediaDerivatives` の読み取りと `MediaRecord.of`、予約の `settle`、`StorageReservation.release` が返した消去待ちの `enqueue`、`release` の判定を保つ保存 |

- `MediaStorage.put`、予約の確認、`MediaStorage.commit`、`MediaStorage.discard` は、UnitOfWork の外で行う。予約の確認は、`PartitionReader.workspace` の `StorageReservationRepository.findById` で読む。順序は [../domains/media.md](../domains/media.md) の「実体と記録の整合」に従う
- `enqueue` と `release` の判定を保つ保存の使い分けは、[../domains/media.md](../domains/media.md) の「実体と記録の整合」が定める
- 後始末の UnitOfWork は、結果の UnitOfWork を確定できなかった処理が、終える前に開く

### 消去待ちの消去

消去待ちの記録（`QueuedRemoval`）ごとに、`target` の `kind` に応じたポートを UnitOfWork の外で呼び、`ObjectRemovalRepository.complete(queued)` を一つの UnitOfWork（条件: `none`）で確定する。順序は [../domains/media.md](../domains/media.md) の「実体と記録の整合」に従う。

| `kind` | 呼ぶポート |
| --- | --- |
| `media` | `MediaStorage.removeAll(workspaceId, mediaId)` |
| `object` | `MediaStorage.remove`（`MediaObjectRef.of` で作った実体の位置） |

消去待ちを記録したユースケースは、その UnitOfWork の確定の後に、`enqueue` が返した記録を消去する。保管の予約の回収だけは、消去を行わず、続けて呼ばれる「消去待ちの実体を消去する」に任せる。

### メディアの削除の手続き

```ts
type MediaRemovalInspection = Readonly<{
  permission: Permission;
  mediaId: MediaId;
  expectedVersion: ExpectedVersion<Media>;
  removal: ObjectRemoval;
}>;

inspectMediaRemoval(
  context: WorkspaceContext,
  target: Readonly<{ mediaId: MediaId; expectedVersion: ExpectedVersion<Media> }>,
  now: Date,
): Promise<MediaRemovalInspection>;
deleteMediaInUnitOfWork(context: WorkspaceContext, inspection: MediaRemovalInspection): Promise<readonly QueuedRemoval[]>;
```

メディア一件の削除を、呼び出し側の UnitOfWork の中で、二つの手続きで行う。メディアを削除するユースケースは、同じ UnitOfWork で、メンバーの認可の手続き、`inspectMediaRemoval`、返した `permission` の確認、`deleteMediaInUnitOfWork` の順に行う。`Media.removal`、`MediaRepository.delete`、`ObjectRemovalRepository.enqueue` を自分で呼ばない。どちらの手続きも、UnitOfWork を開かず、認可を行わない。

- `inspectMediaRemoval` は、`context` の `mediaRepository.findById` でメディアを読み、読んだ `version` を `target.expectedVersion` と照らし、`Media.removal(media, now)` を求める。書き込まない
- `target.expectedVersion` は、利用者が削除を確認したときのメディアの `version`（[../domains/index.md](../domains/index.md) の「公開内容の整合性と通し番号」の、確認した `version` を添える確定）。呼び出し側は、確定の要求が添えた値を渡す
- `permission` は、`Media.removal` が返した、そのメディアの削除に必要な権限。呼び出し側は、メンバーの認可の手続き（[../domains/index.md](../domains/index.md) の「認可」）で得た権限の組と、この権限を `PermissionSet.missing` で照らす
- `deleteMediaInUnitOfWork` は、`inspection` の `expectedVersion` での `MediaRepository.delete` と、`removal` の `enqueue` を行い、`enqueue` が返した記録を返す。呼び出し側は、UnitOfWork の確定の後に、その記録を「消去待ちの消去」に従って消去する
- 同じ UnitOfWork で使用箇所を取り除く呼び出し側は、使用箇所の取り除きを、`permission` の確認の後に行う
- 呼び出し側の UnitOfWork の条件は、`active`、または `active` を含む条件。メディアの書き込みは、`deleteMediaInUnitOfWork` の二つだけ

## メディアを登録する（`registerMedia`）

- **シナリオ**: MEDIA-01、MEDIA-02、MEDIA-09、APP-09、SDEV-06
- **認可**: `member` は `content:edit`。トークンは `content_manage` の許可（対象は `null`）と `content:edit`
- **イベント**: `media.registered`（画像だけ）

### 概要

ファイルを確かめ、実体をメディアの保管先に保管し、メディアとして登録する。管理情報（`MediaProfile`）を同時に作る。画像では、派生形式の記録（`MediaDerivatives`）も同時に作る。派生形式の定義は、`media.registered` の消費者（「派生形式の定義を登録済みの画像に適用する」）が当てる。管理画面からの登録、編集中のドロップ・貼り付けによる登録、スクリプトからの登録が、このユースケースを通る。項目と本文への挿入は、メディアを使う側の下書きの保存が行う。受け付ける種類は呼び出し側が決める。画像だけを受け付ける場所への登録は `image`、ほかの登録は `any`。

### 入出力

- **入力**: `Actor`、`WorkspaceId`、ファイルの内容、元のファイル名、受け付ける種類（`MediaAcceptance`）。境界での容量の拒否は、[app-api.md](app-api.md) の「要求とユースケースの対応」に従う
- **出力**: 登録したメディア

### 使用するドメインの振る舞い・ポート

- `MediaAdmission.measurable`、`ImageProcessor.measure`（`measurable` が形式を返したファイルだけ）、`MediaAdmission.examine`
- `IdGenerator.generate`（`MediaId`、予約の `id`）、`Clock.now`
- `StorageReservation.reserve`、`StorageReservationRepository.insert`、`findById`、`settle`、`StorageReservation.release`
- `MediaObjectKey.original`、`MediaObjectRef.of`、`MediaStorage.put`、`commit`、`discard`（元のファイルの実体の位置）
- `Media.register`、`MediaRepository.insert`、`MediaProfile.create`、`MediaProfileRepository.insert`
- `Media.isImage`、`MediaDerivatives.create`、`MediaDerivativesRepository.insert`（画像）
- `MediaRecord.of`
- `WorkspaceContext.collectEvents`
- `ObjectRemovalRepository.enqueue`、`MediaStorage.removeAll`、`ObjectRemovalRepository.complete`（後始末と、その消去待ちの消去）

### トランザクション境界

- **UnitOfWork の要否**: 必要
- 認可と、ファイルの確認（`measure` と `examine`）は、UnitOfWork の外で先に行う
- UnitOfWork の分け方は、「実体の保管」が定める。予約の `target` は、登録するメディアの ID の `media`
- 結果の UnitOfWork（登録の確定）は、認可の手続き、`Media` と `MediaProfile` の `insert`、画像では `MediaDerivatives` の `insert` と、`MediaDerivatives.create` が返したイベントの記録、予約の `settle` を含む。条件は `active`。使うリポジトリは `mediaRepository`、`mediaProfileRepository`、`mediaDerivativesRepository`、`storageReservationRepository` と、認可のリポジトリ

## メディアを一覧・検索する（`listMedia`）

- **シナリオ**: MEDIA-01、MEDIA-03、MEDIA-10
- **認可**: `member`

### 概要

ワークスペースのメディアを、管理用の名称と検索情報の部分一致、派生形式の生成状況で絞り込んで返す。メディアの管理と、コンテンツの編集でのメディアの選択が使う。選んだメディアの共有する説明は、メディアを使う側が、挿入した使用箇所の説明に写す。

### 入出力

- **入力**: `Actor`、`WorkspaceId`、検索の語、生成状況の絞り込み、`Pagination`
- **出力**: メディアの一覧のページ

### 使用するドメインの振る舞い・ポート

- `WorkspaceSchemaRepository.find`（`PartitionReader` で読む現行スキーマの `derivatives` の全体）
- `MediaListFilter.create`、`Pagination.create`
- `MediaRepository.findPage`
- `MediaProfileRepository.findByIds`（返すメディアの管理情報）、`MediaDerivativesRepository.findByIds`（返すメディアのうち画像の派生形式の記録）
- `MediaRecord.of`、`MediaRecord.renditions`、`MediaDerivatives.statuses`、`Media.isPublic`
- `MediaDelivery.linkFor`（`audience` は `member`）、`MediaDeliveryLinks.publicUrl`

### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みが無い

## メディアを取得する（`getMedia`）

- **シナリオ**: MEDIA-03、MEDIA-04、MEDIA-05、MEDIA-06、MEDIA-07、MEDIA-08、MEDIA-10
- **認可**: `member`

### 概要

メディア一件の記録、説明、公開範囲と、画像の派生形式の生成状況を返す。使用箇所の説明へ取り込む、現在の共有する説明も、この出力から得る。

### 入出力

- **入力**: `Actor`、`WorkspaceId`、`MediaId`
- **出力**: メディア一件の詳細

### 使用するドメインの振る舞い・ポート

- `MediaRepository.findById`、`MediaProfileRepository.findById`
- `MediaDerivativesRepository.findById`（画像）
- `WorkspaceSchemaRepository.find`（`PartitionReader` で読む現行スキーマの `derivatives` の全体。画像）
- `MediaRecord.of`、`MediaRecord.renditions`、`MediaDerivatives.statuses`、`Media.isImage`、`Media.isPublic`
- `MediaDelivery.linkFor`（`audience` は `member`）、`MediaDeliveryLinks.publicUrl`

### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みが無い

## 管理用の名称・検索情報・共有する説明を編集する（`describeMedia`）

- **シナリオ**: MEDIA-04
- **認可**: `member` は `content:edit`

### 概要

メディアの管理用の名称、検索情報、共有する説明（代替テキスト、キャプション）を置き換える。

### 入出力

- **入力**: `Actor`、`WorkspaceId`、`MediaId`、管理用の名称、検索情報、代替テキスト、キャプション
- **出力**: 保存後の管理情報

### 使用するドメインの振る舞い・ポート

- `MediaProfileRepository.findById`、`MediaProfileRepository.save`
- `MediaProfile.describe`

### トランザクション境界

- **UnitOfWork の要否**: 必要。条件: `active`
- UnitOfWork は、認可の手続き、管理情報の読み取り、`MediaProfile.describe`、`save` を含む。使うリポジトリは `mediaProfileRepository` と、認可のリポジトリ

## メディアを非公開化する（`unpublishMedia`）

- **シナリオ**: MEDIA-07、MEDIA-09
- **認可**: `member` は `media:publish`

### 概要

公開のメディアを非公開にする。確定の後の配信は、[../domains/media.md](../domains/media.md) の「MediaDelivery」が決める。

### 入出力

- **入力**: `Actor`、`WorkspaceId`、`MediaId`、非公開化を確認したときのメディアの `version`
- **出力**: 非公開のメディア

### 使用するドメインの振る舞い・ポート

- `MediaRepository.findById`、`MediaRepository.save`
- `Media.unpublish`

### トランザクション境界

- **UnitOfWork の要否**: 必要。条件: `active`
- UnitOfWork は、認可の手続き、メディアの読み取り、読んだ `version` と確認した `version` の照合、`Media.unpublish`、確認した `version` での `save` を含む。使うリポジトリは `mediaRepository` と、認可のリポジトリ

## メディアを削除する（`deleteMedia`）

- **シナリオ**: MEDIA-08、MEDIA-09
- **認可**: `member`。必要な権限は、`Media.removal` が返す `permission`（[../domains/media.md](../domains/media.md) の「削除」）

### 概要

メディアを完全に削除し、確定の後に、元のファイルとすべての派生形式の実体を消去する。確定の後の配信は、[../domains/media.md](../domains/media.md) の「MediaDelivery」が決める。

### 入出力

- **入力**: `Actor`、`WorkspaceId`、`MediaId`、削除を確認したときのメディアの `version`
- **出力**: 無し

### 使用するドメインの振る舞い・ポート

- 「メディアの削除の手続き」（`inspectMediaRemoval`、`deleteMediaInUnitOfWork`）
- `MediaStorage.removeAll`、`ObjectRemovalRepository.complete`（確定の後の消去待ちの消去）

### トランザクション境界

- **UnitOfWork の要否**: 必要。条件: `active`
- UnitOfWork は、「メディアの削除の手続き」が定める順の、メンバーの認可の手続き、`inspectMediaRemoval`（確認した `version` を渡す）、返した `permission` の確認、`deleteMediaInUnitOfWork` を含む。使うリポジトリは `mediaRepository`、`objectRemovalRepository` と、認可のリポジトリ

## 失敗した派生形式の生成をやり直す（`retryDerivative`）

- **シナリオ**: MEDIA-10
- **認可**: `member` は `content:edit`
- **イベント**: `media.derivative_regeneration_requested`

### 概要

生成に失敗した派生形式一つを生成中に戻し、生成を始める。状態の変わり方は `MediaDerivatives.retry` が定める。生成は「派生形式一つを生成する」が行う。

### 入出力

- **入力**: `Actor`、`WorkspaceId`、`MediaId`、派生形式のキー
- **出力**: 保存後の派生形式の記録

### 使用するドメインの振る舞い・ポート

- `MediaRepository.findById`、`Media.assertImage`
- `MediaDerivativesRepository.findById`、`MediaDerivatives.retry`、`MediaDerivativesRepository.save`
- `WorkspaceContext.collectEvents`

### トランザクション境界

- **UnitOfWork の要否**: 必要。条件: `active`
- UnitOfWork は、認可の手続き、メディアと派生形式の記録の読み取り、`MediaDerivatives.retry`、`save`、イベントの記録を含む。使うリポジトリは `mediaRepository`、`mediaDerivativesRepository` と、認可のリポジトリ

## 派生形式一つを生成する（`generateDerivative`）

- **シナリオ**: MEDIA-10
- **認可**: `system`（`media.derivative_regeneration_requested` の消費者）

### 概要

派生形式一つを、元の画像から変換してメディアの保管先に保管し、結果を派生形式の記録に記録する。

### 入出力

- **入力**: `media.derivative_regeneration_requested` のイベント（`workspaceId` と、ペイロードの `mediaId`、`derivative`、`generation`）
- **出力**: 無し

### 使用するドメインの振る舞い・ポート

- `WorkspaceRepository.find`、`MediaRepository.findById`
- `MediaDerivativesRepository.findById`、`MediaDerivativesRepository.save`
- `MediaObjectKey.derivative`（ペイロードから作る）、`MediaObjectRef.of`、`MediaDerivatives.claim`
- `IdGenerator.generate`（予約の `id`）、`Clock.now`
- `StorageReservation.reserve`、`StorageReservationRepository.insert`、`findById`、`settle`、`StorageReservation.release`
- `MediaStorage.get`（元の画像）、`DerivativeGeometry.outputSize`、`ImageProcessor.transform`
- `MediaStorage.put`、`commit`、`discard`（`variant` が `derivative` で、キーと世代を持つ実体の位置）
- `MediaDerivatives.complete`、`MediaDerivatives.fail`（失敗の理由は [../domains/media.md](../domains/media.md) の「DerivativeState」の `DerivativeFailure`）、`MediaRecord.of`
- `ObjectRemoval.ofObjects`、`ObjectRemovalRepository.enqueue`、`MediaStorage.remove`、`ObjectRemovalRepository.complete`

### トランザクション境界

- **UnitOfWork の要否**: 必要
- UnitOfWork の分け方は、「実体の保管」が定める。予約の `target` は、イベントが指す派生形式の `DerivativeObjectKey` の `object`
- 生成の対象の読み取り（区画のワークスペースの状態、`Media`、派生形式の記録、`MediaDerivatives.claim`）は、予約の UnitOfWork の前に、UnitOfWork の外で行う。予約の UnitOfWork は、`claim` が生成の対象を返したイベントだけが開く
- 元の画像の読み出しと変換は、予約の UnitOfWork の確定の後に、UnitOfWork の外で行う
- 結果の UnitOfWork は、`Media` と派生形式の記録の読み直し、読み直した派生形式の記録（無ければ `null`）を渡した `MediaDerivatives.complete`（実体を確定した生成）または `MediaDerivatives.fail`（失敗した生成）、変わった派生形式の記録の `save`、予約の `settle`、`superseded` と `release` が返した消去待ちの一度の `enqueue`、または `release` の判定を保つ保存を含む。使うリポジトリは `mediaRepository`、`mediaDerivativesRepository`、`storageReservationRepository`、`objectRemovalRepository`

## 派生形式の定義を登録済みの画像に適用する（`applyDerivativeDefinitions`）

- **シナリオ**: MEDIA-01、MEDIA-10（SMIG-07 の反映の後）
- **認可**: `system`（`schema.applied` と `media.registered` の消費者）
- **イベント**: `media.derivative_regeneration_requested`

### 概要

現行スキーマの派生形式の定義を、画像の派生形式の記録に適用する。対象は、`schema.applied` ではワークスペースの登録済みの画像すべて、`media.registered` ではその画像一件。変換は行わない。

### 入出力

- **入力**: `schema.applied` のイベント（`workspaceId` だけを使う。ペイロードの `{ schemaVersion }` は使わない）、または `media.registered` のイベント（`workspaceId` と、ペイロードの `mediaId`）
- **出力**: 無し

### 使用するドメインの振る舞い・ポート

- `WorkspaceRepository.find`
- `WorkspaceSchemaRepository.find`（`PartitionReader` で読む現行スキーマの `derivatives` の全体とスキーマの版）
- `MediaDerivativesRepository.scan`（`schema.applied`）、`MediaDerivativesRepository.findById`（`media.registered`）、`MediaDerivativesRepository.save`
- `IdGenerator.generate`（派生形式の記録ごと、定義ごとの世代）、`DerivativeTarget.propose`
- `MediaDerivatives.applyDefinitions`、`MediaDerivatives.requestGeneration`
- `WorkspaceContext.collectEvents`
- `ObjectRemoval.ofObjects`、`ObjectRemovalRepository.enqueue`、`MediaStorage.remove`、`ObjectRemovalRepository.complete`

### トランザクション境界

- **UnitOfWork の要否**: 必要。条件: `schemaVersion`（派生形式の記録を読む前に読んだスキーマの版）
- 派生形式の記録一件ごとに一つの UnitOfWork で、`MediaDerivatives.applyDefinitions`、`changed` が真の場合の `save`（読んだ `expectedVersion` で）、`started` を `requestGeneration` に渡して作ったイベントの記録、`discarded` の消去待ちの `enqueue` を確定する。`changed` が偽なら、UnitOfWork を開かない
- 適用する定義は、派生形式の記録を読む前に読んだ現行スキーマから求める。`schema.applied` では、イベントが示すスキーマの版によらず、`scan` を `mediaId` の昇順に、返る件数が `limit` より少なくなるまで繰り返し、すべての派生形式の記録を走査する。`media.registered` では、その画像の派生形式の記録を `findById` で読む

## 止まった派生形式の生成をやり直す（`restartStalledDerivatives`）

- **シナリオ**: MEDIA-10
- **認可**: `system`（定期ジョブ「止まった派生形式の生成のやり直し」）
- **イベント**: `media.derivative_regeneration_requested`

### 概要

対象のワークスペースの、止まった生成を持つ派生形式の記録に、`MediaDerivatives.restartStalled` を適用する。始め直すか失敗にするかは、`restartStalled` が定める。

### 入出力

- **入力**: `WorkspaceId`
- **出力**: そのワークスペースで、生成を始め直した派生形式の件数と、失敗にした派生形式の件数

### 使用するドメインの振る舞い・ポート

- `Clock.now`、`DerivativePolicy.stalledBefore`
- `MediaDerivativesRepository.findStalled`（`expectedVersion` を添えて返す）、`MediaDerivativesRepository.save`
- `MediaDerivatives.restartStalled`
- `WorkspaceContext.collectEvents`

### トランザクション境界

- **UnitOfWork の要否**: 必要。条件: `active`
- `findStalled` が返す派生形式の記録一件ごとに一つの UnitOfWork で、`MediaDerivatives.restartStalled`、`changed` が真の場合の `save`（`findStalled` が返した `expectedVersion` で）、イベントの記録を確定する。`findStalled` は、UnitOfWork を開く前に、`limit` に `Pagination.maxLimit` を渡して、一度の起動で一度だけ呼ぶ

## 回収できる保管の予約を回収する（`collectStorageReservations`）

- **シナリオ**: MEDIA-01、MEDIA-10
- **認可**: `system`（定期ジョブ「メディアの実体の消去」）

### 概要

対象のワークスペースの、回収できる保管の予約（`StorageReservation.isCollectable`）を取り除き、`StorageReservation.release` が返した消去待ちを記録する。実体の消去は行わない。

### 入出力

- **入力**: `WorkspaceId`
- **出力**: そのワークスペースで、回収した保管の予約の件数

### 使用するドメインの振る舞い・ポート

- `Clock.now`
- `StorageReservationRepository.findCollectable`、`StorageReservationRepository.settle`
- `MediaRepository.findById`、`MediaDerivativesRepository.findById`、`MediaRecord.of`、`StorageReservation.release`
- `ObjectRemovalRepository.enqueue`、`MediaDerivativesRepository.save`（`release` の判定を保つ保存）

### トランザクション境界

- **UnitOfWork の要否**: 必要。条件: `none`
- `findCollectable(now, after, limit)` が返す予約ごとに、「実体の保管」の後始末の UnitOfWork と同じ内容を、一つの UnitOfWork で確定する。`after` は、最初は `null`、次からは前の呼び出しが返した最後の予約。`limit` は `BoundedDeletion.maxLimit`。返る件数が `limit` より少なくなるまで、同じ起動の中で繰り返す

## 消去待ちの実体を消去する（`purgeRemovedObjects`）

- **シナリオ**: MEDIA-01、MEDIA-08、MEDIA-10
- **認可**: `system`（定期ジョブ「メディアの実体の消去」）

### 概要

対象のワークスペースの、消去待ちの記録が指す実体を、メディアの保管先から消去し、記録を削除する。

### 入出力

- **入力**: `WorkspaceId`
- **出力**: そのワークスペースで、消去を終えた消去待ちの記録の件数

### 使用するドメインの振る舞い・ポート

- `ObjectRemovalRepository.findPending`
- `MediaStorage.removeAll`、`MediaStorage.remove`（記録の `kind` ごとの対応は「消去待ちの消去」）
- `ObjectRemovalRepository.complete`

### トランザクション境界

- **UnitOfWork の要否**: 必要。条件: `none`
- `findPending(after, limit)` が返す記録ごとに、「消去待ちの消去」に従って消去する。`after` は、最初は `null`、次からは前の呼び出しが返した最後の記録。`limit` は `BoundedDeletion.maxLimit`。返る件数が `limit` より少なくなるまで、同じ起動の中で繰り返す

## メディアを配信する（`deliverMedia`）

- **シナリオ**: MEDIA-07、MEDIA-08、MEDIA-10、APP-02、APP-05、HOST-09
- **認可**: `Actor` を受け取らない。配信の要求が示した資格（`MediaAccess`）と、ワークスペースが利用できるかどうかから、`MediaDelivery.decide` が判定する。ワークスペースが利用できるかどうかは [../domains/index.md](../domains/index.md) の「利用できるワークスペース」、メンバーの資格の確かめ方は同じ文書の「認可」のメディアの配信の規則に従う

### 概要

配信先への要求に、元のファイルまたは生成済みの派生形式の実体を返す。対象のワークスペースは、配信先から求める。返すかどうかと、配信の先の保持の方針は、要求のたびに `MediaDelivery.decide` が決める（[../domains/media.md](../domains/media.md) の「MediaDelivery」）。

### 入出力

- **入力**: 配信先（`MediaUrl`）、`AccountId | null`
- **出力**: 実体の内容と、`MediaDelivery.decide` の `serve` の結果。応答での表し方は、入口（[app-api.md](app-api.md) の「メディアの配信」）が定める

### 使用するドメインの振る舞い・ポート

- `MediaDeliveryLinks.resolve`（配信先から、ワークスペースの ID を含む `ref` と、許可を得る）
- `PartitionReader.workspace`（`ref.workspaceId` の区画）、`PartitionReader.installation`（設置の区画）
- `WorkspaceDirectory.findById`（Workspace。設置の区画。名簿の項目の状態）、`WorkspaceRepository.find`（`ref.workspaceId` の区画）、`WorkspaceAvailability.of`（Workspace）
- `MediaRepository.findById`、`MediaDerivativesRepository.findById`（画像だけ）、`MediaRecord.of`
- `MemberRepository.findByAccount`、`AccessPolicyRepository.find`（Workspace。`ref.workspaceId` の区画）、`AccountRepository.findById`、`Account.isMfaEnabled`（Identity。設置の区画）、`MemberAuthorization.resolve`（Workspace。先に読んだ名簿の状態と `Workspace`、読んだ `AccessPolicy` を `policy` に渡す）
- `MediaAccess.resolve`
- `MediaDelivery.decide`（`object` は、`ref` の区画の中の部分。`record` は `MediaRecord.of` の結果）、`Clock.now`
- `MediaObjectRef.of`
- `MediaStorage.get`

### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みが無い
