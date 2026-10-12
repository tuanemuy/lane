# Content のユースケース

コンテンツの作成・編集・公開・削除、履歴、メディアの使用箇所の一覧、取得、編集中の記録、保持方針と自動整理を実現する。型・振る舞い・ポート・規則は [../domains/content.md](../domains/content.md) が、ドメインをまたぐ規則は [../domains/index.md](../domains/index.md) が定める。ユースケースは、振る舞いとポートを組み合わせる。このファイルが定めるのは、認可、入出力、使う振る舞いとポート、トランザクション境界。結果の種類ごとの条件、エラー、途中で止まったときに残る状態は、`spec/testcases/content/` のユースケースごとのテストが定める。

メディアの登録・非公開化と、使用箇所の無いメディアの削除は Media、プレビューを開く操作と引き換えは Token、表示アプリへの通知の作成・送信・再試行は Webhook、スキーマの反映の準備・確定・失敗は Schema のユースケースが行う。

見出しの識別子は、`spec/testcases/content/` のファイル名と同じ。ユースケースを通らないドメインの純粋な関数の代表ケースは、同じディレクトリの、関数の名前のファイル（`ContentDefaults.initial.md` の形）に置く。

## 共通

一度に渡せる件数に上限があるポートのメソッドは、どのユースケースも、そのポートの契約が定める上限の範囲に分けて呼ぶ。利用者の入力の検証は、ポートを呼ぶ前に済ませる。

各ユースケースは、次の手続きを名前で指す。

### 実行者と認可

- 認可の手続きと、認可を行う位置は [../domains/index.md](../domains/index.md) の「認可」に、ワークスペースが利用できるかどうかは同「利用できるワークスペース」に従う。区画を書かず、区画に結び付かないポート（[../domains/index.md](../domains/index.md) の「集約とポートの置き場所」）だけに書くユースケースは、UnitOfWork を開かず、そのポートを呼ぶ前に認可を行う
- Content のユースケースは、入力の `WorkspaceId` のワークスペースの区画だけを書く
- 各ユースケースの「認可」は、受け付ける実行者の種類（`member` / `appToken` / `memberToken` / `system`）と、必要な `Permission`、トークンに必要な `TokenGrant` を書く。「トークン」は `appToken` と `memberToken` の両方を指す。「メンバー」とだけ書いた認可は、権限を必要としない。「トークン: 不可」「メンバー: 不可」は、その種類の実行者を受け付けないことを表す
- トークンの許可・範囲・権限は、ユースケースがトークンの認可の手続き（`authorizeTokenOperation`、[token.md](token.md)）で判定する。`AccessRequirement` の `target` は対象のコンテンツが属するモデルの `ModelId`。複数の対象を扱う操作は、対象のモデルごとに判定する。モデルをまたぐ一覧（`listTrashedContents`）は、絞り込むモデルを指定した場合は、そのモデルを `target` にする。省略した場合は、[../domains/token.md](../domains/token.md) の「TokenScope」の複数のモデルにまたがる読み取りに当たる
- 対象の状態、書き換え先、新たに公開するメディアによって加わる権限は、`ContentPermissions.additional` が返す。メディアの削除に必要な権限は、Media の `inspectMediaRemoval`（[media.md](media.md)）が返す。ユースケースは、対象を読んでから、結果の `permission` を実行者の権限の組に `PermissionSet.missing`（Workspace）で照らす。不足の扱いは [../domains/content.md](../domains/content.md) の「ContentPermissions」が定める。各ユースケースの「認可」は、これを「加わる権限」と書く
- 値・状態・置き場所を変える振る舞いには、実行者を `Performer`（`Actor` と記録の時点の名称）で渡す。メンバーの `memberName` は、認可で読んだアカウントの名前。トークンの `token`（`GrantedTokenAccess` の `designation`）と `memberName` は、[token.md](token.md) の `authorizeTokenOperation` の「入出力」の出力の読み方に従う
- 入力を保存するユースケースは、入力の帰属（`InputSource`）を受け、書き込みの前に `InputSource.isOwnedBy` で確かめる（[../domains/content.md](../domains/content.md) の「InputSource」）。管理画面の入口は `attributed` と、入力を始めたときのメンバーを渡す
- 各ユースケースの「使うリポジトリ」は、認可で読むリポジトリを省く

### コンテンツの書き込み

コンテンツと履歴の版を書き込むユースケースは、次を行う。

- UnitOfWork を開く前に、`WorkspaceSchemaRepository.find` で現行スキーマを読む。適用中の扱いと、読んだスキーマで入力を検証することは、[../domains/schema.md](../domains/schema.md) の「WriteScope とスキーマ」に従う
- 一つの UnitOfWork を、読んだスキーマの版の `stableSchema` の条件で開く。条件の定義は [../domains/common.md](../domains/common.md) の「WriteScope」が定める
- モデルは、キーを `WorkspaceSchema.modelIds` で `ModelId` に解決し、`ModelContext.of` で検証の文脈にする
- 対象は `ContentRepository.findById` または `findByIds` で読む。要求が添えた `version` を、読んだ `Content.version` と比べる（[../domains/content.md](../domains/content.md) の「競合の検出」）。`version` を添える要求は、入力に `version` を持つユースケースの要求
- `Content` を保存する書き込みは、`version` を添えない要求でも、UnitOfWork の中で読んだ `expectedVersion` を渡す
- 振る舞いに渡す `now` は、`Clock.now` で得た時刻
- 振る舞いが返した `retired` は、`IdGenerator.generate` で `RevisionId` を作り、`Revision.record` で版にして、一度の `RevisionRepository.insertMany` で同じ UnitOfWork に保存する
- 内容（公開内容、下書き、履歴の版）を足す、置き換える、なくす UnitOfWork は、その UnitOfWork の書き込みを `MediaUsages.ofWrite` に渡し、結果を、同じ UnitOfWork で `MediaUsageRepository.replaceMany`（Media）に一度だけ渡す。コンテンツとともに削除される版は、書き込みの前に `RevisionRepository.findTargetsByContents` で、続きが無くなるまで読む
- 複数の対象を扱う操作は、対象の列を [../domains/content.md](../domains/content.md) の「一度に扱う対象」の表のとおりに作り、すべての対象の検証を終えてから、変えた対象の `Content` を、一度の `ContentRepository.saveMany` で保存する

各ユースケースの「トランザクション境界」は、`WriteScope` を「条件: `stableSchema`」のように、`condition` の種類で書く。区画は、どれも入力の `WorkspaceId` のワークスペースの区画。書き込みの回数の数え方は、[../domains/common.md](../domains/common.md) の「UnitOfWorkProvider」に従う。

### 検証に渡す事実

- 参照先の状態: 内容の `references`（保存前の入力は `ContentReferences.of` で求める）を `ContentReferences.contentIds` で ID にし、`ContentRepository.findByIds` で読んで、`ReferenceTarget.of` で `ReferenceTargets` にする
- メディアの事実: `ContentReferences.mediaIds` の ID を `MediaRepository.findByIds`（Media）で読み、`Media.facts` の結果（`MediaFactSet`）を渡す
- 一意な値の保持者: 対象のモデルごとに、`UniqueValues.of` の結果を `ContentRepository.findUniqueHolders` に渡して `UniqueHolder` を得る
- 参照元: `ContentReferenceQuery.findInbound` に、ID の集合を、メソッドが受ける上限の件数までずつ渡し、`Pagination` で続きを読む

### 公開内容の変更の確定

公開内容を変えるユースケースは、入力に操作の ID（`GeneratedId`）を受け、`OperationId.create` で `OperationId` にして、出力に返す。操作の ID を作る入口と、応答に含める経路は、[../domains/index.md](../domains/index.md) の「公開内容の整合性と通し番号」が定める。

振る舞いが `PublicationChange` を一つ以上返したユースケースは、「コンテンツの書き込み」と同じ UnitOfWork で次を行う。

- `PublicationChange` ごとに `PublicationChange.toNotice` を呼ぶ。モデルのキーは `WorkspaceSchema.modelIds` から求める
- `PublicationLedger.record` に、操作の ID、操作の種類、実行者、`toNotice` の値、`stableSchema` の条件のスキーマの版を渡す。結果の記録のドラフトを、読んだ `WorkspaceSchema.modelIds` とともに `PublicationLedgerRepository.append` に渡し、結果のイベントのドラフトを `collectEvents` に渡す
- メディアの公開（公開と、公開内容の値の書き換え）: 新しい公開内容が指すメディアを `MediaRepository.findByIds`（Media）で読み、`NewlyPublishedMedia.of` に渡す。その結果を、検証（`PublicationRules.checkPublish` と `RewriteRules.check` の `newlyPublishedMedia`）と、加わる権限（`ContentPermissions.additional`）に渡し、結果の `ids` を一度の `MediaRepository.publishMany`（Media）に渡す。確認を求める読み取りも、同じ結果を検証と加わる権限に使う

### 公開内容の整合性と、確認の示し直し

- 公開内容の整合性の担保と、確認した `version` を添える確定の規則は、[../domains/index.md](../domains/index.md) の「公開内容の整合性と通し番号」が定める
- 確認のユースケースは、対象ごとの `version` を返す。確定のユースケースは、確認した `version` を受け取り、確認と同じ振る舞い（`PublicationRules`、`RewriteRules`、`ContentPermissions`、`NewlyPublishedMedia.of`）で確かめ直す
- 確定の時点のポートの拒否は、`ContentRepository`、Media の `MediaUsageRepository`、`MediaRepository` の契約が定める

### 履歴の版の明示的な削除

利用者が指定した履歴の版を削除するユースケースは、「コンテンツの書き込み」と同じ UnitOfWork で、次を行う。その削除の確認を求める読み取りは、同じ読み方で版を読む。

- 版の ID の列を `RevisionRepository.findByIds` で読む。コンテンツを指定するユースケースは、`Revision.belongsTo` で、読んだ版がそのコンテンツの版かを確かめる
- 読んだ版の全体を、一度の `RevisionRepository.deleteMany` に渡す
- 読んだ版の `id` と `contentId` から作った `RevisionTarget` を、`MediaUsages.ofWrite` の `removedRevisions` に渡し、削除した版のメディアの使用箇所を消す

### 読み取り

- UnitOfWork を開かない。リポジトリの読み取りを、`PartitionReader.workspace(workspaceId)` から呼ぶ
- 読み始めと読み終わりに `WorkspaceSchemaRepository.find` で現行スキーマを読み、[../domains/content.md](../domains/content.md) の「スキーマの反映との関係」のとおりに読み直す
- ページを受ける一覧（メディアの使用箇所の一覧を含む）のページは、[../domains/content.md](../domains/content.md) の「一覧の条件」に従って作る

### 自動整理の共通

自動整理のユースケースは、次を行う。

- 対象のワークスペースを入力に受け、そのワークスペースの区画だけを読み書きする。対象のワークスペースと、ワークスペースごとの進め方は、[../domains/index.md](../domains/index.md) の「定期ジョブ」の表と、「区画をまたぐ手順」の定期ジョブに従う
- `ContentRetentionPolicyRepository.find` で保持方針を読む
- 削除の UnitOfWork ごとに、開く前に `WorkspaceSchemaRepository.find` で現行スキーマを読む。削除の UnitOfWork は、条件 `stableSchema` で開く
- 削除する版のメディアの使用箇所は、「コンテンツの書き込み」に従って消す
- 一つのワークスペースの中の対象（`findExpired` の一回の結果、ごみ箱のコンテンツ一件）ごとに、一つの UnitOfWork で削除する。一つの UnitOfWork が確定しなかったときの扱いは、各ユースケースのテストが定める

## 管理の読み取り

### コンテンツを一覧・検索する（`listContents`）

一つのモデルの通常の管理対象のコンテンツを、最新の内容で一覧する。

- **シナリオ**: EDIT-01、EDIT-15、EDIT-16、APP-09、APP-10、SMIG-11
- **認可**: メンバー。トークン: `content_manage`
- **入力**: モデルのキー、検索語、状態の絞り込み（公開状態、下書きの有無、下書きの入力エラーの有無、現行の制約を満たさない公開内容の有無）、項目の値の絞り込み（`FilterInput`。値は表記のまま）、並び替え、ページ
- **出力**: コンテンツの一覧と総件数。一覧の一件は、最新の内容で示し、公開状態と下書きの有無、続く操作に添える `version` を持つ。モデルが非推奨かどうか
- **使用するドメインの振る舞い・ポート**: 「読み取り」。モデルのキーの解決（`WorkspaceSchema.modelIds`）、`Pagination.create`、`ContentDelivery.searchFields`、`ContentDelivery.validateQuery`（`ListScope` は `management`）、`ContentRepository.findPage`、`Content.latest`
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

### ごみ箱のコンテンツを一覧する（`listTrashedContents`）

ごみ箱のコンテンツを、モデルをまたいで一覧する。

- **シナリオ**: EDIT-01、PUB-06、APP-09、APP-10、SMIG-11
- **認可**: メンバー。トークン: `content_manage`
- **入力**: 絞り込むモデルのキー（省略時はすべてのモデル）、ページ
- **出力**: ごみ箱のコンテンツの一覧と総件数。一覧の一件は、最新の内容で示し、移動した日時と実行者の名称、自動整理で完全削除される時期、モデルが非推奨かどうか、続く操作に添える `version` を持つ
- **使用するドメインの振る舞い・ポート**: 「読み取り」。モデルのキーの解決（`WorkspaceSchema.modelIds`）、`Pagination.create`、`ContentDelivery.validateCandidates`、`ContentRepository.findCandidates`（`location` は `trashed`、`term` は `null`）、`Content.latest`、`ContentRetentionPolicyRepository.find`、`ContentRetentionPolicy.purgeDueAt`
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

### モデルごとの入力エラーと制約違反の件数を確認する（`countContentIssues`）

現行スキーマのモデルごとに、通常の管理対象のコンテンツのうち、入力エラーを持つ下書きの現在の件数と、現行の制約を満たさない公開内容の現在の件数を返す。

- **シナリオ**: EDIT-15、SMIG-14
- **認可**: メンバー。トークン: 不可
- **入力**: なし
- **出力**: モデルごとの、モデルのキー、入力エラーを持つ下書きの件数、制約を満たさない公開内容の件数
- **使用するドメインの振る舞い・ポート**: 「読み取り」。`Pagination.create`、`ContentDelivery.validateQuery`（`ListScope` は `management`。`status` は、`draftHasIssues` だけを真にしたものと、`publishedHasViolations` だけを真にしたもの。`search` は `null`、`filters` は空）、`ContentRepository.findPage`（結果の `count` だけを使う）
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

### 編集するコンテンツを取得する（`getContentForEditing`）

一件のコンテンツの公開内容と下書きを返す。

- **シナリオ**: EDIT-01、EDIT-03、EDIT-04、EDIT-13、EDIT-15、EDIT-16、PUB-06、APP-09、APP-10
- **認可**: メンバー。トークン: `content_manage`
- **入力**: コンテンツID、または単一モデルのキー
- **出力**: コンテンツ（`Content`）と、最新の内容がどちらか。または、単一モデルの通常の管理対象にコンテンツが無いこと
- **使用するドメインの振る舞い・ポート**: 「読み取り」。`ContentRepository.findById`、`ContentRepository.findSingle`、`Content.latest`
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

### 参照と一意な値の状態を確認する（`checkReferenceAndUniqueStatus`）

一件のコンテンツの最新の内容について、参照切れ、参照先の表示に使う情報、一意な値の重複を返す。下書きを持つ公開中のコンテンツでは、公開内容についても、参照切れと参照先の表示に使う情報を返す。保存した下書きの、選ぶ間に完全削除された参照先（`missing`）と、ほかのコンテンツの公開内容との重複は、このユースケースが示す。

- **シナリオ**: EDIT-03、EDIT-05、EDIT-07、EDIT-15、PUB-06、APP-09
- **認可**: メンバー。トークン: `content_manage`
- **入力**: コンテンツID
- **出力**: 最新の内容の、コンテンツ参照と本文のリンクごとの解決結果（解決できない理由、必須・任意・本文のリンクの別）、参照先ごとの状態、一意な項目ごとの、同じ値を公開内容に持つほかのコンテンツ。公開内容の、参照ごとの解決結果と参照先ごとの状態（下書きを持つ公開中のコンテンツだけ）
- **使用するドメインの振る舞い・ポート**: 「読み取り」「検証に渡す事実」。`ContentRepository.findById`、`Content.latest`、`PublicationRules.resolve`（`mode` は `public`。最新の内容と、下書きを持つ公開中のコンテンツの公開内容に、それぞれ呼ぶ）、`ContentRepository.findUniqueHolders`
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

### コンテンツの候補を検索する（`searchContentCandidates`）

参照先の候補と、一括操作の対象に加える候補を、複数のモデルをまたいで探す。

- **シナリオ**: EDIT-05、EDIT-07、PUB-02、PUB-04、PUB-05、PUB-07
- **認可**: メンバー。トークン: 不可
- **入力**: 対象のモデルのキー（参照先の候補では、項目が参照を許可するモデル）、検索語、置き場所（通常の管理対象とごみ箱のどちらか一方）、公開状態、下書きの有無、ページ
- **出力**: 候補の一覧と総件数。候補は、最新の内容で示し、公開状態、下書きの有無、置き場所と、続く操作に添える `version` を持つ
- **使用するドメインの振る舞い・ポート**: 「読み取り」。モデルのキーの解決（`WorkspaceSchema.modelIds`）、`Pagination.create`、`ContentDelivery.searchFields`、`ContentDelivery.validateCandidates`、`ContentRepository.findCandidates`
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

### コンテンツを複製する入力を求める（`prepareDuplicateInput`）

複製元の最新の内容から、新しいコンテンツの入力にする値を求める。コンテンツを作らない。

- **シナリオ**: EDIT-08
- **認可**: メンバー。編集権限（`content:edit`）。トークン: 不可
- **入力**: 複製元のコンテンツID
- **出力**: 入力にする値と、未設定にした一意な項目のキー
- **使用するドメインの振る舞い・ポート**: 「読み取り」。`ContentRepository.findById`、`Content.asActive`、`Content.latest`、`Content.duplicateValues`
- **トランザクション境界**: UnitOfWork は不要。書き込みがない。複製先は「コンテンツを作成する」が作る

### 入力を最新の内容と比較する（`compareInputWithLatest`）

保存が競合した入力と、保存済みの最新の内容の差分を項目ごとに求める。

- **シナリオ**: EDIT-03、EDIT-04、EDIT-10、EDIT-11
- **認可**: メンバー。編集権限（`content:edit`）。トークン: 不可
- **入力**: コンテンツID、入力の値
- **出力**: 最新の内容の値と `version`、項目ごとの差分、最後の変更の操作の種類・実行者の名称・日時（`lastChange`）
- **使用するドメインの振る舞い・ポート**: 「読み取り」。`ContentRepository.findById`、`Content.asActive`、`Content.latest`、`ContentDiff.compare`
- **トランザクション境界**: UnitOfWork は不要。書き込みがない。取り込んだ結果の保存は「下書きを保存する」が行う

## 作成と下書き

### コンテンツを作成する（`createContent`）

入力を下書きとして持つ、未公開のコンテンツを作る。新規作成、複製、単一モデルの最初の保存、トークンによる ID を指定した作成が使う。

- **シナリオ**: EDIT-02、EDIT-04、EDIT-05、EDIT-06、EDIT-07、EDIT-08、EDIT-10、EDIT-11、EDIT-16、MEDIA-02、MEDIA-03、MEDIA-05、APP-09
- **認可**: メンバー、トークン。編集権限（`content:edit`）。トークン: `content_manage`
- **入力**: モデルのキー、コンテンツID（`GeneratedId`）、値、入力の帰属（`InputSource`）。メディアの項目の値と本文の画像は、使用箇所ごとの説明（`MediaReference` の `alt` と `caption`）を含む。送り直しでは、同じ ID を渡す
- **出力**: 「作成した」または「同じ内容で既にある」と、コンテンツ（コンテンツID、`version`、下書きの入力エラー）
- **使用するドメインの振る舞い・ポート**: 「コンテンツの書き込み」「検証に渡す事実」。`InputSource.isOwnedBy`、`ContentValidation.checkEdit`（`before` は `null`）、`ContentRepository.findById`、`Content.isReplayOf`、`ContentRepository.findSingle`（単一モデル。結果を `Content.create` の `occupant` に渡す）、`Content.create`、`ContentRepository.insert`、`MediaUsages.ofWrite`、`MediaUsageRepository.replaceMany`（Media）
- **トランザクション境界**:
  - UnitOfWork は必要。条件: `stableSchema`。一つの UnitOfWork で行うのは、認可、`Content` の `insert`、下書きのメディアの使用箇所の書き込み
  - 使うリポジトリ: `contentRepository`、`mediaUsageRepository`、`mediaRepository`（読み取り）
  - 書き込みの回数: 2

### 下書きを保存する（`saveDraft`）

通常の管理対象のコンテンツの入力を、下書きとして保存する。単一モデルの二度目以降の保存も、このユースケースを使う。

- **シナリオ**: EDIT-03、EDIT-04、EDIT-05、EDIT-06、EDIT-07、EDIT-10、EDIT-11、EDIT-15、EDIT-16、PUB-01（公開を始めるときの未保存の入力の保存）、EDIT-12（プレビューを開く前の未保存の入力の保存）、MEDIA-02、MEDIA-03、MEDIA-05、MEDIA-09、APP-09
- **認可**: メンバー、トークン。編集権限（`content:edit`）。トークン: `content_manage`
- **入力**: コンテンツID、読み取ったときの `version`、値、入力の帰属（`InputSource`）。メディアの項目の値と本文の画像は、使用箇所ごとの説明（`MediaReference` の `alt` と `caption`）を含む
- **出力**: 保存したコンテンツ（`version`、下書きの入力エラー）
- **使用するドメインの振る舞い・ポート**: 「コンテンツの書き込み」「検証に渡す事実」。`InputSource.isOwnedBy`、`Content.asActive`、`Content.latest`、`ContentValidation.checkEdit`（`before` は最新の内容の値）、`Content.saveDraft`、`ContentRepository.save`、`MediaUsages.ofWrite`、`MediaUsageRepository.replaceMany`（Media）
- **トランザクション境界**:
  - UnitOfWork は必要。条件: `stableSchema`。一つの UnitOfWork で行うのは、認可、`Content` の `save`、置き換えた下書きの `Revision` の追加、メディアの使用箇所の書き込み
  - 使うリポジトリ: `contentRepository`、`revisionRepository`、`mediaUsageRepository`、`mediaRepository`（読み取り）
  - 書き込みの回数: 3

### 下書きを破棄する（`discardDraft`）

公開中のコンテンツの下書きをなくす。

- **シナリオ**: EDIT-09、EDIT-16、APP-09
- **認可**: メンバー、トークン。編集権限（`content:edit`）。トークン: `content_manage`
- **入力**: コンテンツID、読み取ったときの `version`
- **出力**: 破棄した後のコンテンツ（`version`）
- **使用するドメインの振る舞い・ポート**: 「コンテンツの書き込み」。`Content.asPublished`、`Content.asDrafted`、`Content.discardDraft`、`ContentRepository.save`、`MediaUsages.ofWrite`、`MediaUsageRepository.replaceMany`（Media）
- **トランザクション境界**:
  - UnitOfWork は必要。条件: `stableSchema`。一つの UnitOfWork で行うのは、認可、`Content` の `save`、破棄した下書きの `Revision` の追加、メディアの使用箇所の書き込み
  - 使うリポジトリ: `contentRepository`、`revisionRepository`、`mediaUsageRepository`
  - 書き込みの回数: 3

## 公開と削除

### 公開の確認を求める（`reviewPublish`）

公開する対象の全体を検証し、確認に示す内容を返す。何も変えない。

- **シナリオ**: PUB-01、PUB-02
- **認可**: メンバー。トークン: 不可
- **入力**: 対象のコンテンツID の列
- **出力**: 対象ごとの `version`、公開内容と下書きの差分（未公開では下書きの内容）、変わる一意な値の変更前後、対象を参照している公開中の参照元。新たに公開するメディア。加わる権限（`ContentPermissions.additional`）と、実行者に不足する権限。検証の違反のすべて（`PublishViolation`）と、違反が指す参照先の置き場所・公開状態・下書きの有無。公開できない対象（存在しない、ごみ箱にある、非推奨のモデル）
- **使用するドメインの振る舞い・ポート**: 「読み取り」「検証に渡す事実」。`BatchTargets.create`、`ContentRepository.findByIds`、`DeprecatedModel.allows`（`published`）、`ContentPermissions.additional`（`published`）、`PublicationRules.checkPublish`（存在しない対象、ごみ箱の対象、`DeprecatedModel.allows` が偽の対象を除いて渡す）、`ContentDiff.compare`、`MediaRepository.findByIds`（Media）、`NewlyPublishedMedia.of`、`ContentReferenceQuery.findInbound`（`holders` は `published`）
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

### コンテンツを公開する（`publishContent`）

一件以上のコンテンツの下書きを、一つの操作で公開内容にする。

- **シナリオ**: PUB-01、PUB-02、EDIT-16、APP-09
- **認可**: メンバー、トークン。公開権限（`content:publish`）と、加わる権限（`ContentPermissions.additional` の `published`）。トークン: `content_manage`
- **入力**: 操作の ID（`GeneratedId`）、対象ごとのコンテンツID と、確認したときの `version` の列
- **出力**: 操作の ID、公開した対象ごとの `version` と、公開か更新かの別。公開したメディア
- **イベント**: `content.publication_changed`（記録の `operation` は `published`）
- **使用するドメインの振る舞い・ポート**: 「コンテンツの書き込み」「検証に渡す事実」「公開内容の変更の確定」。`BatchTargets.createBy`、`Content.asActive`、`ContentPermissions.additional`、`PublicationRules.checkPublish`、`Content.publish`、`ContentRepository.saveMany`
- **トランザクション境界**:
  - UnitOfWork は必要。条件: `stableSchema`。一つの UnitOfWork で、認可、すべての対象の検証、すべての対象の `Content` の `saveMany`、置き換えた公開内容の `Revision` の追加、メディアの使用箇所の書き込み、メディアの公開の保存、公開の変更の記録の追記、イベントの記録を行う
  - 使うリポジトリ: `contentRepository`、`revisionRepository`、`mediaUsageRepository`、`mediaRepository`、`publicationLedgerRepository`
  - 書き込みの回数: 6（`ContentRepository.saveMany`、`insertMany`、`MediaUsageRepository.replaceMany`、`MediaRepository.publishMany`、`append`、イベント）。対象とメディアの件数によらない

### 公開取り消しの確認を求める（`reviewUnpublish`）

公開を取り消す対象の参照の条件を確かめ、確認に示す内容を返す。何も変えない。

- **シナリオ**: PUB-03、PUB-04
- **認可**: メンバー。トークン: 不可
- **入力**: 対象のコンテンツID の列
- **出力**: 対象ごとの `version` と下書きの有無、取り消しを妨げる参照（参照元、項目、参照の `strength`）、保護する参照でない参照だけで参照している公開中の参照元、下書きから参照しているコンテンツ、公開のまま残るメディア
- **使用するドメインの振る舞い・ポート**: 「読み取り」「検証に渡す事実」。`BatchTargets.create`、`ContentRepository.findByIds`、`ContentReferenceQuery.findInbound`（`holders` は `published` と `draft`）、`PublicationRules.checkWithdraw`、`ContentReferences.mediaIds`
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

### 公開を取り消す（`unpublishContent`）

一件以上のコンテンツの公開内容を、一つの操作でなくす。

- **シナリオ**: PUB-03、PUB-04、EDIT-16、APP-09
- **認可**: メンバー、トークン。公開権限（`content:publish`）。トークン: `content_manage`
- **入力**: 操作の ID（`GeneratedId`）、対象ごとのコンテンツID と、確認したときの `version` の列
- **出力**: 操作の ID、取り消した対象ごとの `version`
- **イベント**: `content.publication_changed`（記録の `operation` は `unpublished`）
- **使用するドメインの振る舞い・ポート**: 「コンテンツの書き込み」「検証に渡す事実」「公開内容の変更の確定」。`BatchTargets.createBy`、`Content.asPublished`、`ContentReferenceQuery.findInbound`（`holders` は `published`）、`PublicationRules.checkWithdraw`、`Content.unpublish`、`ContentRepository.saveMany`
- **トランザクション境界**:
  - UnitOfWork は必要。条件: `stableSchema`。一つの UnitOfWork で、認可、参照の条件の確認、すべての対象の `Content` の `saveMany`、なくした公開内容の `Revision` の追加、メディアの使用箇所の書き込み、公開の変更の記録の追記、イベントの記録を行う
  - 使うリポジトリ: `contentRepository`、`revisionRepository`、`mediaUsageRepository`、`publicationLedgerRepository`
  - 書き込みの回数: 5（`saveMany`、`insertMany`、`MediaUsageRepository.replaceMany`、`append`、イベント）。対象の件数によらない

### ごみ箱への移動の確認を求める（`reviewTrash`）

ごみ箱へ移動する対象の参照の条件を確かめ、確認に示す内容を返す。何も変えない。

- **シナリオ**: PUB-05、EDIT-04
- **認可**: メンバー。トークン: 不可
- **入力**: 対象のコンテンツID の列
- **出力**: 対象ごとの `version` と公開状態、移動の後にごみ箱から復元できるか（`DeprecatedModel.allows`）、下書きから参照しているコンテンツ、確認の時点に移動した場合に自動整理で完全削除される時期の見込み。公開中の対象については「公開取り消しの確認を求める」と同じ内容。加わる権限（`ContentPermissions.additional`）と、実行者に不足する権限
- **使用するドメインの振る舞い・ポート**: 「読み取り」「検証に渡す事実」。`BatchTargets.create`、`ContentRepository.findByIds`、`DeprecatedModel.allows`（`restoredFromTrash`）、`ContentPermissions.additional`（`trashed`）、`ContentReferenceQuery.findInbound`（`holders` は `published` と `draft`）、`PublicationRules.checkWithdraw`、`ContentRetentionPolicyRepository.find`、`Clock.now`、`ContentRetentionPolicy.purgeDueAt`（`trashedAt` に確認の時点を渡す）
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

### コンテンツをごみ箱へ移動する（`trashContent`）

一件以上のコンテンツを、一つの操作でごみ箱へ移動する。

- **シナリオ**: PUB-05、EDIT-04、EDIT-16、APP-09
- **認可**: メンバー、トークン。削除権限（`content:delete`）と、加わる権限（`ContentPermissions.additional` の `trashed`）。トークン: `content_manage`
- **入力**: 操作の ID（`GeneratedId`）、対象ごとのコンテンツID と、確認したときの `version` の列
- **出力**: 操作の ID、移動した対象ごとの `version`。公開中だった対象
- **イベント**: `content.publication_changed`（記録の `operation` は `trashed`）
- **使用するドメインの振る舞い・ポート**: 「コンテンツの書き込み」「検証に渡す事実」「公開内容の変更の確定」。`BatchTargets.createBy`、`Content.asActive`、`ContentPermissions.additional`、`ContentReferenceQuery.findInbound`（`holders` は `published`）、`PublicationRules.checkWithdraw`、`Content.unpublish`（公開中の対象）、`Content.trash`、`ContentRepository.saveMany`
- **トランザクション境界**:
  - UnitOfWork は必要。条件: `stableSchema`。一つの UnitOfWork で、認可、参照の条件の確認、すべての対象の `Content` の `saveMany`、なくした公開内容の `Revision` の追加、メディアの使用箇所の書き込み、「公開内容の変更の確定」の書き込みを行う
  - 使うリポジトリ: `contentRepository`、`revisionRepository`、`mediaUsageRepository`、`publicationLedgerRepository`
  - 書き込みの回数: 5（`saveMany`、`insertMany`、`MediaUsageRepository.replaceMany`、`append`、イベント）。対象の件数によらない

### ごみ箱のコンテンツを復元する（`restoreTrashedContent`）

ごみ箱のコンテンツを、通常の管理対象へ戻す。

- **シナリオ**: PUB-06、EDIT-16、APP-09
- **認可**: メンバー、トークン。削除権限（`content:delete`）。トークン: `content_manage`
- **入力**: コンテンツID
- **出力**: 復元したコンテンツ（`version`、公開状態、下書きの入力エラー）
- **使用するドメインの振る舞い・ポート**: 「コンテンツの書き込み」。`Content.asTrashed`、`ContentRepository.findSingle`（単一モデル。結果を `Content.restore` の `occupant` に渡す）、`Content.restore`、`ContentRepository.save`
- **トランザクション境界**:
  - UnitOfWork は必要。条件: `stableSchema`。一つの UnitOfWork で行うのは、認可、`Content` の `save`
  - 使うリポジトリ: `contentRepository`
  - 書き込みの回数: 1

### 完全削除の確認を求める（`reviewPurge`）

完全削除する対象の削除条件を確かめ、確認に示す内容を返す。何も変えない。

- **シナリオ**: PUB-07、EDIT-04
- **認可**: メンバー。トークン: 不可
- **入力**: 対象のコンテンツID の列。通常の管理対象とごみ箱の両方を含められる。対象に加える履歴の版（`AddedRevisions`）
- **出力**: 対象ごとの `version`・置き場所・公開状態、削除を妨げる参照（対象、参照元、参照している内容の種類（公開内容・下書き・履歴の版。履歴の版ではその ID）、参照元の置き場所（通常・ごみ箱））、保護する参照でない参照だけで参照している参照元。対象に加えた版については、加えた版の全体を渡した「履歴の版の削除の確認を求める」と同じ内容。ただし、残る使用・参照の持ち主から、完全削除の対象のコンテンツの公開内容・下書き・版を除く。公開中の対象については「公開取り消しの確認を求める」と同じ内容。加わる権限（`ContentPermissions.additional`）と、実行者に不足する権限
- **使用するドメインの振る舞い・ポート**: 「読み取り」「検証に渡す事実」。`ContentPermissions.additional`（`purged`）、`BatchTargets.create`（コンテンツ）、`AddedRevisions.of`、`ContentRepository.findByIds`、「履歴の版の明示的な削除」の版の読み方、`ContentReferenceQuery.findInbound`（`holders` は `published`、`draft`、`revision`）、`PublicationRules.checkPurge`（`removedRevisions` は[../domains/content.md](../domains/content.md) の「公開取り消し・ごみ箱・完全削除の参照の条件」の `removedRevisions`）、`PublicationRules.checkWithdraw`
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

### コンテンツを完全削除する（`purgeContent`）

一件以上のコンテンツを、公開内容・下書き・履歴とともに、一つの操作で削除する。

- **シナリオ**: PUB-07、EDIT-04、EDIT-16、APP-09
- **認可**: メンバー、トークン。削除権限（`content:delete`）と、加わる権限（`ContentPermissions.additional` の `purged`）。履歴の版を対象に加える権限も削除権限。トークン: `content_manage`。トークンの範囲は、対象のコンテンツのモデルと、加える版のコンテンツ（読んだ `Revision` の `contentId`）のモデルについて確かめる
- **入力**: 操作の ID（`GeneratedId`）、対象ごとのコンテンツID と、確認したときの `version` の列。対象に加える履歴の版（`AddedRevisions`）
- **出力**: 操作の ID、削除した対象。削除した版。公開中だった対象
- **イベント**: `content.publication_changed`（記録の `operation` は `purged`）
- **使用するドメインの振る舞い・ポート**: 「コンテンツの書き込み」「検証に渡す事実」「公開内容の変更の確定」「履歴の版の明示的な削除」の版の読み方（加える版）。`BatchTargets.createBy`（コンテンツ）、`ContentPermissions.additional`、`ContentReferenceQuery.findInbound`（`holders` は `published`、`draft`、`revision`）、`AddedRevisions.of`、`PublicationRules.checkWithdraw`（公開中の対象）、`PublicationRules.checkPurge`（`removedRevisions` は[../domains/content.md](../domains/content.md) の「公開取り消し・ごみ箱・完全削除の参照の条件」の `removedRevisions`）、`PublicationChange.ofRemoval`（公開中の対象）、`RevisionRepository.findTargetsByContents`、`ContentRepository.deleteMany`、`MediaUsages.ofWrite`、`MediaUsageRepository.replaceMany`（Media）
- **トランザクション境界**:
  - UnitOfWork は必要。条件: `stableSchema`。一つの UnitOfWork で、認可、削除条件の確認、`checkPurge` に渡した `removedRevisions` の全体を渡した一度の `RevisionRepository.deleteMany`、対象の全体を渡した一度の `ContentRepository.deleteMany`、削除した内容と版のメディアの使用箇所の消去、「公開内容の変更の確定」の書き込みを行う
  - 使うリポジトリ: `contentRepository`、`revisionRepository`、`mediaUsageRepository`、`publicationLedgerRepository`
  - 書き込みの回数: 5（版の `deleteMany`、コンテンツの `deleteMany`、`MediaUsageRepository.replaceMany`、`append`、イベント）。対象の件数と、対象が持つ履歴の版の数によらない

### 値を書き換える（`rewriteValues`）

一件以上のコンテンツの、指定した項目の値だけを、一つの操作で変える。

- **シナリオ**: APP-10、APP-09、APP-12、SMIG-11、EDIT-15、EDIT-16
- **認可**: トークン。`content_manage`。加わる権限（`ContentPermissions.additional` の `rewritten`）。メンバー: 不可
- **入力**: 操作の ID（`GeneratedId`）、対象ごとのコンテンツID、読み取ったときの `version`、書き換え先（公開内容・下書き・両方の `RewriteTarget`）と、書き換える項目と値の列
- **出力**: 操作の ID、書き換えた対象ごとの `version` と書き換え先。変わらなかった対象。公開したメディア
- **イベント**: `content.publication_changed`（記録の `operation` は `rewritten`）
- **使用するドメインの振る舞い・ポート**: 「コンテンツの書き込み」「検証に渡す事実」「公開内容の変更の確定」。`BatchTargets.createBy`、`ContentPermissions.additional`、`RewriteRules.check`、`Content.rewrite`、`ContentRepository.saveMany`
- **トランザクション境界**:
  - UnitOfWork は必要。条件: `stableSchema`。一つの UnitOfWork で、認可、すべての対象の検証、変わった対象の `Content` の `saveMany`、置き換えた公開内容と下書きの `Revision` の追加（一度の `insertMany`）、メディアの使用箇所の書き込み、「公開内容の変更の確定」の書き込みを行う
  - 使うリポジトリ: `contentRepository`、`revisionRepository`、`mediaUsageRepository`、`mediaRepository`、`publicationLedgerRepository`
  - 書き込みの回数: 6（`ContentRepository.saveMany`、`insertMany`、`MediaUsageRepository.replaceMany`、`MediaRepository.publishMany`、`append`、イベント）。対象、版、メディアの件数によらない

## 履歴

### 履歴を一覧する（`listRevisions`）

一件のコンテンツの履歴の版を一覧する。

- **シナリオ**: EDIT-13、EDIT-17、PUB-06
- **認可**: メンバー。トークン: 不可
- **入力**: コンテンツID、ページ
- **出力**: 版の一覧と総件数、履歴を保持する日数。一覧の一件は、値を除く版（`Revision`）。実行者の名称は、版の `retired.by` の `Performer` が持つ
- **使用するドメインの振る舞い・ポート**: 「読み取り」。`ContentRepository.findById`、`RevisionRepository.findPage`、`ContentRetentionPolicyRepository.find`
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

### 履歴の版を取得する（`getRevision`）

一つの版の内容を、現行スキーマの形で返す。

- **シナリオ**: EDIT-13
- **認可**: メンバー。トークン: 不可
- **入力**: コンテンツID、版の ID
- **出力**: 版（`Revision`）。値と、版にした操作（`retired`）、その内容を作った操作（`produced`）を持つ
- **使用するドメインの振る舞い・ポート**: 「読み取り」。`RevisionRepository.findById`、`Revision.belongsTo`
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

### 内容を比較する（`compareBodies`）

一件のコンテンツの二つの内容の差分を、項目ごとに求める。

- **シナリオ**: EDIT-03、EDIT-09、EDIT-13、EDIT-14
- **認可**: メンバー。トークン: 不可
- **入力**: コンテンツID、比べる二つの内容。それぞれ、履歴の版（版の ID）、現在の下書き、現在の公開内容、最新の内容のいずれか
- **出力**: 二つの内容の値と、項目ごとの差分。コンテンツの `version`
- **使用するドメインの振る舞い・ポート**: 「読み取り」。`ContentRepository.findById`、`RevisionRepository.findById`、`Revision.belongsTo`、`Content.latest`、`ContentDiff.compare`
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

### 版の復元の確認を求める（`reviewRevisionRestore`）

版を下書きとして復元した結果を求め、確認に示す内容を返す。何も変えない。

- **シナリオ**: EDIT-14
- **認可**: メンバー。トークン: 不可
- **入力**: コンテンツID、版の ID
- **出力**: コンテンツの `version`、復元する値と最新の内容の差分。復元できない場合は、その理由（ごみ箱にある、非推奨のモデル）
- **使用するドメインの振る舞い・ポート**: 「読み取り」「検証に渡す事実」。`ContentRepository.findById`、`RevisionRepository.findById`、`Revision.belongsTo`、`DeprecatedModel.allows`（`revisionRestored`）、`Content.latest`、`ContentValidation.checkRestore`、`ContentDiff.compare`
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

### 版を下書きとして復元する（`restoreRevisionAsDraft`）

履歴の版の内容を下書きにする。

- **シナリオ**: EDIT-14、EDIT-16
- **認可**: メンバー。編集権限（`content:edit`）。トークン: 不可
- **入力**: コンテンツID、確認したときの `version`、版の ID
- **出力**: 復元した後のコンテンツ（`version`、下書きの入力エラー）
- **使用するドメインの振る舞い・ポート**: 「コンテンツの書き込み」「検証に渡す事実」。`RevisionRepository.findById`、`Revision.belongsTo`、`Content.asActive`、`Content.latest`、`ContentValidation.checkRestore`、`Content.restoreRevision`、`ContentRepository.save`、`MediaUsages.ofWrite`、`MediaUsageRepository.replaceMany`（Media）
- **トランザクション境界**:
  - UnitOfWork は必要。条件: `stableSchema`。一つの UnitOfWork で行うのは、認可、`Content` の `save`、置き換えた下書きの `Revision` の追加、メディアの使用箇所の書き込み
  - 使うリポジトリ: `contentRepository`、`revisionRepository`、`mediaUsageRepository`
  - 書き込みの回数: 3

### 履歴の版の削除の確認を求める（`reviewRevisionDeletion`）

削除する版が持つメディアの使用とコンテンツへの参照が、削除の後に残るかを返す。何も変えない。

- **シナリオ**: EDIT-17、MEDIA-08
- **認可**: メンバー。トークン: 不可
- **入力**: 版の ID の列（複数のコンテンツの版を含められる）
- **出力**: 対象の版ごとに「履歴を一覧する」の一件と同じ内容、対象の版が使用するメディアと、保護する参照で参照するコンテンツ、それぞれが削除の後も使用・参照されるかと、残る使用・参照の持ち主
- **使用するドメインの振る舞い・ポート**: 「読み取り」「検証に渡す事実」。`BatchTargets.create`（版の ID）、「履歴の版の明示的な削除」の版の読み方、`ContentReferences.mediaIds`、`ContentReferences.contentIds`、`MediaUsageRepository.findPageByMedia`（Media）、`BodyHolder.parse`、`ContentReferenceQuery.findInbound`（`holders` はすべての種類、`strengths` は保護する参照の `strength`）。残る使用・参照の持ち主は、読んだ持ち主のうち、対象の版のどれでもない持ち主
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

### 履歴の版を削除する（`deleteRevisions`）

一件のコンテンツの、指定した履歴の版を削除する。

- **シナリオ**: EDIT-17
- **認可**: メンバー。削除権限（`content:delete`）。トークン: 不可
- **入力**: コンテンツID、版の ID の列
- **出力**: 削除した版の ID
- **使用するドメインの振る舞い・ポート**: 「コンテンツの書き込み」（現行スキーマとワークスペースの確認）「履歴の版の明示的な削除」。`BatchTargets.create`（版の ID）、`ContentRepository.findById`、`MediaUsageRepository.replaceMany`（Media）
- **トランザクション境界**:
  - UnitOfWork は必要。条件: `stableSchema`。一つの UnitOfWork で行うのは、認可、`RevisionRepository.deleteMany`、削除した版のメディアの使用箇所の消去
  - 使うリポジトリ: `contentRepository`（読み取り）、`revisionRepository`、`mediaUsageRepository`
  - 書き込みの回数: 2

## メディアの使用箇所

Media の型・振る舞い・ポートを使う。使用箇所の記録の置き場所と、Content と Media の分担は、[../domains/index.md](../domains/index.md) の「参照の記録と保護」に従う。

### メディアの使用箇所を一覧する（`listMediaUsages`）

メディアを使用している内容を返す。非公開化と削除の前の確認と、拒否された非公開化と削除の後の提示に使う。

- **シナリオ**: MEDIA-06、MEDIA-07、MEDIA-08、MEDIA-09
- **認可**: メンバー。トークン: 不可
- **入力**: メディアの ID、ページ
- **出力**: 使用箇所の一覧と総件数、使用の状況（Media の `MediaUsageStatus`）、メディアの公開範囲。使用箇所は、コンテンツ、項目の位置（一つ以上）、内容の種類（公開内容、下書き、履歴の版）、コンテンツの置き場所を示す
- **使用するドメインの振る舞い・ポート**: 「読み取り」。`Pagination.create`、`MediaRepository.findById`（Media）、`Media.isPublic`（Media）、`MediaUsageRepository.findPageByMedia`（Media）、`MediaUsageRepository.findStatusByMedia`（Media）、`BodyHolder.parse`、`ContentRepository.findByIds`、`RevisionRepository.findByIds`。持ち主、項目の位置、総件数の求め方は、[../domains/content.md](../domains/content.md) の「メディアの使用箇所」が定める
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

使用箇所の種類ごとの解消方法と必要な権限は、出力の内容の種類と置き場所から、呼び出し側が示す。

### 履歴の版とともにメディアを削除する（`deleteMediaWithRevisions`）

メディアの削除を妨げている履歴の版を、利用者が明示して対象に加え、版とメディアを同じ確定で削除する。版を加えない削除は、Media の「メディアを削除する」（[media.md](media.md)）が行う。

- **シナリオ**: MEDIA-08、MEDIA-09
- **認可**: メンバー。削除権限（`content:delete`）と、加わる権限（Media の `inspectMediaRemoval` が返す、そのメディアの削除に必要な権限）。トークン: 不可
- **入力**: メディアの ID と、削除を確認したときのメディアの `version`、対象に加える履歴の版（版の ID の列）
- **出力**: 削除した版の ID
- **使用するドメインの振る舞い・ポート**: 「コンテンツの書き込み」「履歴の版の明示的な削除」。`BatchTargets.create`（加える版）、`MediaUsageRepository.replaceMany`（Media）。メディアの削除は、[media.md](media.md) の「メディアの削除の手続き」に従う
- **トランザクション境界**:
  - UnitOfWork は必要。条件: `stableSchema`。一つの UnitOfWork で、[media.md](media.md) の「メディアの削除の手続き」（`inspectMediaRemoval` の `expectedVersion` は確認したメディアの `version`）を行い、その手続きの `permission` の確認の後に、加える版の `RevisionRepository.deleteMany` と、その版のメディアの使用箇所の消去を行う。拒否が重なった場合の順は [../domains/content.md](../domains/content.md) の「拒否の順」が定める
  - 使うリポジトリ: `revisionRepository`、`mediaUsageRepository`、`mediaRepository`、`objectRemovalRepository`
  - 書き込みの回数: 4。加える版の件数によらない

## 取得

公開取得とプレビュー取得は、次を共通に行う。

- モデルのキーを `WorkspaceSchema.modelIds` で解決する
- トークンの範囲は、対象のモデルと、展開に指定した項目が参照を許可するモデルについて確かめる
- 展開は `ContentDelivery.validateExpansion` で確かめる。取得の対象を段 0 の内容として、`ContentDelivery.referencedIds` の `load` を `ContentRepository.findByIds` で読み、`expand` に当たる読んだコンテンツを次の段の内容にすることを、`expand` が空になるまで繰り返す。読んだコンテンツを `ContentDelivery.assemble` に渡す。読んだ参照先が `ContentDelivery.limits.referencedContents` 件を超えた段で、続きを読まずに `assemble` に渡す
- `assemble` の結果が `delivered` なら、`ContentDelivery.arrange` で参照の位置ごとの形を置いた結果を出力にする。入口は、その結果を直列化する（[app-api.md](app-api.md)）
- メディアは、返す内容が使うメディアを `MediaRepository.findByIds` と `MediaDerivativesRepository.findByIds`（Media）で読み、読んだ `Media` と派生形式の記録を `MediaRecord.of` に通した値の `MediaRecord.renditions`、`MediaDelivery.linkFor`、`MediaDeliveryLinks.publicUrl` または `previewUrl` で、元のファイルと生成済みの派生形式の配信先・画像形式・幅・高さを求める。`linkFor` の `audience` は、公開取得で `published`、プレビュー取得で `preview`
- 添えられた識別子は、読んだ `WorkspaceSchema` の `WorkspaceSchema.matchFingerprints` で照合し、結果を出力に添える。取得の対象のモデルの識別子も、展開する参照先のモデルの識別子も、添えられたものをモデルごとに照合する
- 「読み取り」に従う。UnitOfWork を開かない。利用量の記録と最終利用日時の記録は、入口の `authenticateToken` が行う。トークンの認可は、`authorizeTokenOperation` の手続きを、読み取りとして行う
- 公開取得の `DeliveryResult` の `stale` の扱いは、[../domains/content.md](../domains/content.md) の「スキーマの反映との関係」に従う

### 公開コンテンツを単件で取得する（`getPublishedContent`）

一件のコンテンツの公開内容を返す。

- **シナリオ**: APP-02、APP-04、APP-12
- **認可**: トークン。`public_read`。メンバー: 不可
- **入力**: モデルのキー、対象の指定（コンテンツID、一意な項目のキーと値、単一モデルでは指定なし）、展開する項目と深さ、添える識別子
- **出力**: コンテンツの公開内容（値と、メディアの配信先）、参照先の要約と、展開した内容、識別子ごとの一致
- **使用するドメインの振る舞い・ポート**: 取得の共通。`ContentRepository.findById`、`ContentDelivery.validateLookup`、`ContentRepository.findByUniqueValue`（`view` は `published`）、`ContentRepository.findSingle`、`ContentDelivery.assemble`（`mode` は `public`）
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

### 公開コンテンツの一覧を取得する（`listPublishedContents`）

条件に合う公開内容の一覧と総件数を返す。

- **シナリオ**: APP-03、APP-04、APP-12
- **認可**: トークン。`public_read`。メンバー: 不可
- **入力**: モデルのキー、絞り込みの条件（`FilterInput`。値は表記のまま）、並び替え、ページ、展開する項目と深さ、添える識別子
- **出力**: コンテンツの一覧、総件数、参照先の要約と、展開した内容、識別子ごとの一致
- **使用するドメインの振る舞い・ポート**: 取得の共通。`ContentDelivery.validateQuery`（`ListScope` は `public`）、`ContentRepository.findPage`、`ContentDelivery.assemble`（`mode` は `public`）
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

### プレビュー取得で単件を取得する（`getPreviewContent`）

一件のコンテンツの最新の内容を返す。

- **シナリオ**: APP-05、APP-04、APP-12
- **認可**: トークン。`preview_read`。メンバー: 不可
- **入力**: 「公開コンテンツを単件で取得する」と同じ
- **出力**: 「公開コンテンツを単件で取得する」と同じ形で、最新の内容を返す。返した内容が下書きか公開内容かと、制約違反の有無を加える
- **使用するドメインの振る舞い・ポート**: 取得の共通。`ContentRepository.findById`、`ContentDelivery.validateLookup`、`ContentRepository.findByUniqueValue`（`view` は `latest`）、`ContentDelivery.selectUnique`、`ContentRepository.findSingle`、`ContentDelivery.assemble`（`mode` は `preview`）
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

### プレビュー取得で一覧を取得する（`listPreviewContents`）

条件に合う最新の内容の一覧と総件数を返す。

- **シナリオ**: APP-05、APP-04、APP-12
- **認可**: トークン。`preview_read`。メンバー: 不可
- **入力**: 「公開コンテンツの一覧を取得する」と同じ
- **出力**: コンテンツの一覧、総件数、参照先の要約と、展開した内容、識別子ごとの一致、結果に含めなかった対象（件数とコンテンツID）
- **使用するドメインの振る舞い・ポート**: 取得の共通。`ContentDelivery.validateQuery`（`ListScope` は `preview`）、`ContentRepository.findPage`、`ContentRepository.findPreviewExcluded`、`ContentDelivery.assemble`（`mode` は `preview`）
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

## 編集中の記録

### 編集中であることを知らせる（`announceEditing`）

メンバーがコンテンツを編集していることを記録する。

- **シナリオ**: EDIT-03、EDIT-10
- **認可**: メンバー。編集権限（`content:edit`）。トークン: 不可
- **入力**: コンテンツID
- **出力**: なし
- **使用するドメインの振る舞い・ポート**: `ContentRepository.findById`、`Content.asActive`、`Clock.now`、`EditingPresenceStore.find`、`EditingPresence.announce`（`memberName` は、認可で読んだアカウントの名前）、`EditingPresenceStore.put`
- **トランザクション境界**: UnitOfWork は不要。認可とコンテンツの読み取りの後に、`EditingPresenceStore.put` の一回の呼び出しで反映する

### 編集を終えたことを知らせる（`endEditing`）

メンバーの編集中の記録を削除する。

- **シナリオ**: EDIT-03、EDIT-10
- **認可**: メンバー。トークン: 不可
- **入力**: コンテンツID
- **出力**: なし
- **使用するドメインの振る舞い・ポート**: `EditingPresenceStore.remove`
- **トランザクション境界**: UnitOfWork は不要。認可の後に、`remove` の一回の呼び出しで反映する

### 編集中のメンバーを確認する（`listEditingMembers`）

同じコンテンツを編集している、ほかのメンバーを返す。

- **シナリオ**: EDIT-03、EDIT-10
- **認可**: メンバー。トークン: 不可
- **入力**: コンテンツID
- **出力**: 編集しているほかのメンバーの `MemberId` と名前（`memberName`）、編集を始めた日時
- **使用するドメインの振る舞い・ポート**: `Clock.now`、`EditingPresenceStore.list`、`EditingPresence.others`
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

### 編集中の記録を掃除する（`cleanUpEditingPresence`）

定期ジョブ「編集中の記録の掃除」。失効した編集中の記録を削除する。

- **シナリオ**: EDIT-03、EDIT-10（定期ジョブ「編集中の記録の掃除」）
- **認可**: `system` だけ。メンバー: 不可。トークン: 不可
- **入力**: なし
- **出力**: 削除した記録の件数
- **使用するドメインの振る舞い・ポート**: `Clock.now`、`EditingPresenceStore.deleteExpired`（[../domains/common.md](../domains/common.md) の「TransactionalRepository」の「件数を区切る削除」）
- **トランザクション境界**: UnitOfWork は不要

## 保持方針と自動整理

### コンテンツの保持方針を確認する（`getRetentionPolicy`）

履歴とごみ箱の保持方針を返す。

- **シナリオ**: WS-10
- **認可**: メンバー。ワークスペース設定の権限（`workspace:manage`）。トークン: 不可
- **入力**: なし
- **出力**: 履歴を保持する日数、ごみ箱のコンテンツを保持する日数、指定できる範囲
- **使用するドメインの振る舞い・ポート**: `ContentRetentionPolicyRepository.find`
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

### 保持方針の変更の影響を確認する（`reviewRetentionPolicyChange`）

保持方針を変えた場合に、整理対象になる履歴の版とごみ箱のコンテンツの件数を返す。何も変えない。

- **シナリオ**: WS-10
- **認可**: メンバー。ワークスペース設定の権限（`workspace:manage`）。トークン: 不可
- **入力**: 履歴を保持する日数、ごみ箱のコンテンツを保持する日数
- **出力**: 現在の保持方針で整理対象になる版とコンテンツの件数、入力の保持方針で整理対象になる版とコンテンツの件数
- **使用するドメインの振る舞い・ポート**: `ContentRetentionPolicy.validate`、`ContentRetentionPolicyRepository.find`、`Clock.now`、`ContentRetentionPolicy.change`、`ContentRetentionPolicy.revisionCutoff`、`ContentRetentionPolicy.trashCutoff`、`RevisionRepository.countExpired`、`ContentRepository.countTrashedBefore`
- **トランザクション境界**: UnitOfWork は不要。書き込みがない。`change` の結果を保存しない

### コンテンツの保持方針を変更する（`changeRetentionPolicy`）

履歴とごみ箱の保持方針を変える。

- **シナリオ**: WS-10
- **認可**: メンバー。ワークスペース設定の権限（`workspace:manage`）。トークン: 不可
- **操作履歴**: `content.retention_policy_changed`。`changes`（`AuditChange` の列）の項目は、`revisionDays`（履歴の版を保持する日数）と `trashDays`（ごみ箱のコンテンツを保持する日数）。値は変更前と変更後の `ValueChange`（Audit）
- **入力**: 履歴を保持する日数、ごみ箱のコンテンツを保持する日数
- **出力**: 変更した保持方針
- **使用するドメインの振る舞い・ポート**: `ContentRetentionPolicy.validate`、`ContentRetentionPolicyRepository.find`、`Clock.now`、`ContentRetentionPolicy.change`、`ContentRetentionPolicyRepository.save`、`AuditActor.from`、`AuditLogEntry.record`、`AuditLogRepository.append`（Audit）
- **トランザクション境界**:
  - UnitOfWork は必要。条件: `active`。一つの UnitOfWork で行うのは、認可、`ContentRetentionPolicy` の `save`、`AuditLogEntry` の追記
  - 使うリポジトリ: `contentRetentionPolicyRepository`、`auditLogRepository`
  - 書き込みの回数: 2

### 期限を過ぎた履歴の版を削除する（`cleanUpExpiredRevisions`）

定期ジョブ「履歴の版の整理」。保持方針の期限を過ぎた履歴の版を削除する。対象は [../domains/content.md](../domains/content.md) の「ContentRetentionPolicy」が定める。

- **シナリオ**: WS-10、EDIT-13（定期ジョブ「履歴の版の整理」）
- **認可**: `system` だけ。メンバー: 不可。トークン: 不可
- **入力**: 対象のワークスペース（`WorkspaceId`）
- **出力**: 削除した版の件数
- **使用するドメインの振る舞い・ポート**: 自動整理の共通。`Clock.now`、`ContentRetentionPolicy.revisionCutoff`、`RevisionRepository.findExpired`（`limit` は `ContentBatch.maxRevisions`）、`RevisionRepository.deleteMany`（読んだ `RevisionTarget` の版の ID）、`MediaUsages.ofWrite`
- **トランザクション境界**:
  - UnitOfWork は必要。条件: `stableSchema`。`findExpired` の一回の結果ごとに一つの UnitOfWork で、読んだ版の全体を渡した一度の `RevisionRepository.deleteMany` と、その版のメディアの使用箇所の消去を行う。読んだ件数が `limit` より少なくなるか、読んだ版を削除せずにその対象の処理を終えるまで繰り返す
  - 使うリポジトリ: `revisionRepository`、`mediaUsageRepository`
  - 書き込みの回数: UnitOfWork ごとに 2

### 期限を過ぎたごみ箱のコンテンツを完全削除する（`purgeExpiredTrash`）

定期ジョブ「ごみ箱の完全削除」。保持方針の期限を過ぎたごみ箱のコンテンツを、一件ずつ完全削除する。対象は [../domains/content.md](../domains/content.md) の「ContentRetentionPolicy」が定める。

- **シナリオ**: WS-10、PUB-05、PUB-06（定期ジョブ「ごみ箱の完全削除」）
- **認可**: `system` だけ。メンバー: 不可。トークン: 不可
- **入力**: 対象のワークスペース（`WorkspaceId`）
- **出力**: 完全削除したコンテンツの件数、参照されているためごみ箱に残したコンテンツの件数
- **使用するドメインの振る舞い・ポート**: 自動整理の共通。`Clock.now`、`ContentRetentionPolicy.trashCutoff`、`ContentRepository.findTrashedBefore`（`limit` は `ContentPortLimits.chunk`。最後に受け取ったコンテンツの `trashedAt` と `id` を `TrashCursor` にして続きを読む）、`RevisionRepository.findTargetsByContents`、`ContentRepository.deleteMany`（対象は一件）、`MediaUsages.ofWrite`
- **トランザクション境界**:
  - UnitOfWork は必要。条件: `stableSchema`。`findTrashedBefore` が返したコンテンツ一件ごとに一つの UnitOfWork で、`ContentRepository.deleteMany` を対象一件で呼び、そのコンテンツと版のメディアの使用箇所を消す
  - 使うリポジトリ: `contentRepository`、`revisionRepository`（読み取り）、`mediaUsageRepository`
  - 書き込みの回数: UnitOfWork ごとに 2。コンテンツが持つ履歴の版の数によらない

## シナリオとの対応

- MEDIA-02、MEDIA-03、MEDIA-05 のうち Content が担うのは、使用箇所ごとの説明（`MediaReference` の `alt` と `caption`）の保存。`createContent` と `saveDraft` が、下書きの値の一部として保存する。メディアの登録と一覧、共有する説明の取得は、Media の `registerMedia`、`listMedia`、`getMedia`（[media.md](media.md)）が担う。共有する説明を使用箇所へ写すことと取り込むことは、[../domains/content.md](../domains/content.md) の「MediaReference」に従う
- EDIT-04 の同時の最初の保存は、どちらも `createContent` を呼ぶ。先に作られたコンテンツは `getContentForEditing`（単一モデルのキー）が返し、入力との差分は `compareInputWithLatest` が求める。取り込んだ結果の保存は `saveDraft` が行う
- EDIT-11 の送り直しと、別のメンバーがログインして保存し直す場合は、`createContent` と `saveDraft` が扱う
- EDIT-16 の非推奨のモデルでの操作は、[../domains/content.md](../domains/content.md) の「操作の種類」の表の操作を行うユースケースが扱う
- PUB-06 の復元は、`restoreTrashedContent` の一つの操作で確定する
- PUB-01、PUB-03、PUB-05 の単件の確認で、示した参照先・参照元を対象に加える操作は、加えた対象を含む列で、同じ確認のユースケース（`reviewPublish`、`reviewUnpublish`、`reviewTrash`）を呼び直す。確認と確定のユースケースは、一件でも複数件でも同じ
- PUB-07 の、ごみ箱のコンテンツの内容・履歴を確認している場面と、単一モデルのコンテンツを編集している場面（EDIT-04）から始める完全削除は、そのコンテンツ一件を対象に `reviewPurge` を呼ぶ。通常の管理対象のコンテンツも対象にできる
- APP-12 の `rewriteValues` は、規則と上限をエラーで示すことを担う

ユースケースは、初期値（`ContentDefaults.initial`。EDIT-02、EDIT-06）と貼り付けた本文の取り込み（`RichText.sanitize`。EDIT-05）を済ませた後の値を受け取る。
