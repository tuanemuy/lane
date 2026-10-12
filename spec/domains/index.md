# ドメイン一覧

ビジネス上の関心事の境界と、境界どうしの依存方向を定める。各ドメインの型・振る舞い・ポートは、ドメインごとのファイルが定める。構成は `AGENTS.md` のヘキサゴナルアーキテクチャに従う。

## ドメイン

| ドメイン | 責務 | ファイル |
| --- | --- | --- |
| Identity | 人が本人であることを確かめ、セッションを保つ | [identity.md](identity.md) |
| Workspace | ワークスペースごとに、誰が何を行えるかを保つ | [workspace.md](workspace.md) |
| Audit | 行った操作の履歴を保つ | [audit.md](audit.md) |
| Token | アプリが示すトークンと、トークンを表示アプリへ渡す経路（プレビュー、スクリプトの認証）を保つ | [token.md](token.md) |
| Schema | スキーマと移行定義の意味を定め、ワークスペースの現行スキーマ、版の記録、反映の進行を保つ | [schema.md](schema.md) |
| Media | メディアの実体・公開範囲・派生形式を保つ | [media.md](media.md) |
| Content | コンテンツの公開内容・下書き・ごみ箱・履歴と、公開・参照・一意性の規則を保つ | [content.md](content.md) |
| Form | フォームの受付、回答の保存、メール通知とその送信結果を保つ | [form.md](form.md) |
| Webhook | 公開内容の変更を表示アプリへ伝え、その送信の状態を保つ | [webhook.md](webhook.md) |
| Installation | 設置の初期化とデータの版、初期構成、バックアップ、復旧の進行を保つ。各ドメインが公開する手順を束ねて、ワークスペースの作成と利用終了を進める | [installation.md](installation.md) |

## 依存方向

矢印は「依存する側 → 依存される側」。依存する側のドメイン層（`packages/core/src/domain/${domain}/`）が、依存される側の型と振る舞いを使うこと、または依存する側のポートの契約が、依存される側の集約の状態とポートの契約に依ることを指す。循環は無い。

```text
Token     → Workspace, Schema
Workspace → Identity
Media     → Schema
Content   → Schema, Media
Form      → Schema
Installation → Workspace, Media（BackupStore の契約）
```

Identity、Audit、Schema、Webhook は他のドメインに依存しない。Installation は、ポート `BackupStore` の契約だけが、Workspace（写す名簿）と Media（メディアの実体の置き換えの手順）に依る。Installation のドメイン層の型と振る舞いは、ほかのドメインに依存しない

- 集約の ID の型は「共通の型とポート」に置く。他のドメインの集約を ID で参照することは、依存に数えない。Content はメディアを `MediaId` で、Form は問い合わせ対象を `ContentId` で、Workspace はアカウントを `AccountId` で、Webhook は変更した対象を `ContentId` で、Token は範囲を `ModelId` / `FormId` で参照する
- 権限の語彙（`Permission`）は「共通の型とポート」に置く。操作に必要な権限を `Permission` の値で示すことは、Workspace への依存に数えない。Media の `Media.removal` と `Media.publication` は、削除と公開に必要な権限を `Permission` で返す。権限の組の判定（`PermissionSet.allows`、`PermissionSet.missing`）を使うドメインは、Workspace に依存する
- Token は、Workspace の `PermissionSet` と `MemberAccess`、Schema の `ModelKey`・`FormKey`・`FieldKey` と `PreviewPathResolution` を使う。Media、Content、Form は、Schema の定義（派生形式、項目、フォーム、移行定義の操作）を使う
- Workspace のポート `MembershipIndex`（所属の索引）の契約は、項目のアカウント（Identity の `Account`）が存在することに依る。メンバーの認可（`MemberAuthorization.resolve`）は `Account` を受けず、多要素認証の設定の有無を真偽値で受ける。Identity のドメイン層は、Workspace の型・状態・ポートを使わない（多要素認証の解除との関係は [identity.md](identity.md) の「Account」）。メンバーとアカウントは別の区画にあり、メンバーのアカウントが存在することは、ポートではなく、「区画をまたぐ手順」の順序が担保する
- Content のドメインサービスは、内容が使うメディアを、Media の型（使用箇所）で求める。Content のユースケースが、Media のポート `MediaUsageRepository` に書く。Media のドメイン層とユースケースは、Content の型・状態・ポートを使わない（「参照の記録と保護」）
- Media のポートの対象のワークスペースは、呼び出し側が渡す（[media.md](media.md) の「ポート」）
- 初期構成、利用再開前の処理、利用終了の消去、状態確認は、各ドメインが公開する手順と項目を Installation のユースケースが呼ぶ。各ドメインは、自分のファイルの次の節に公開する。「初期構成の宣言」は、受ける宣言の形、宣言の検証、生成の手順、宣言の照合値から除く値。「利用再開前の処理」は、手順、対象のワークスペース（名簿の状態）、残件数の求め方、確認に示す内容。「利用終了の消去」は、手順と、確認に示す内容。前提（先に終えているべきドメイン）は、前提を持つドメインだけが、同じ節に書く。「状態確認の項目」は、示す項目（種類、件数、必要な対処）。宣言の全体の形と、宣言の位置ごとの受けるドメインの表は、[installation.md](installation.md) の「初期構成の読み込み」が持つ。項目の型は [common.md](common.md) の「各ドメインが公開する項目」が定める。手順を呼ぶ順は、この文書の「手順を呼ぶ順」だけが持つ
- 依存される側のドメイン層は、依存する側の型・状態・ポートを使わない。各ドメインの文書が、公開する手順の前提（先に終えているべきドメイン）に、ほかのドメインの名前を挙げることは、依存に数えない

### ユースケースの所属

アプリケーション層のユースケースは、複数のドメインの振る舞いとポートを使える。依存方向の制約はドメイン層にだけ掛かる。ユースケースの所属は、次の二つで決まる。

- 複数のドメインの集約と記録を、どれか一つを主にせずに束ねるユースケース（各ドメインが「初期構成の宣言」「利用再開前の処理」「利用終了の消去」「状態確認の項目」に公開する手順と項目を呼ぶものを含む）と、設置の全体を対象にするユースケースは、束ねる側の Installation に属する
- それ以外のユースケースは、その操作が主に状態を変える集約のドメイン、または、主に書き込む集約に数えない記録（「集約」の表の後の箇条）を保存するポートを定義するドメインに属する

ユースケースは、属するドメインの名前と、`spec/usecases/${domain}.md` の見出しの識別子の組で指す。識別子は、ドメインの中で一意。違うドメインに同じ識別子があってよい（Content と Form の `changeRetentionPolicy`）。この文書と `spec/flows/` は、識別子にドメインの名前を添える。

| ユースケースの種類 | 属するドメイン |
| --- | --- |
| 公開、公開取り消し、ごみ箱、完全削除、値の書き換え（メディアの公開、メディアの使用箇所の記録、公開の変更の記録を伴う）、メディアの使用箇所の一覧、履歴の版とともに行うメディアの削除 | Content |
| メディアの登録、非公開化、使用箇所の無いメディアの削除、派生形式の生成 | Media |
| スキーマの反映の要求・準備・確定、適用の事前確認、適用済みとしての記録 | Schema |
| フォームの送信の受付、通知の送信と再試行 | Form |
| 公開の変更の記録からの `ChangeNotification` の作成、表示アプリへの通知の送信と再試行 | Webhook |
| 招待の受諾（アカウントの作成、所属の索引への登録、メンバーの作成）、メンバーの利用の解除（メンバーの削除、所属の索引からの除外）、所属の索引の修復、最後に利用したワークスペースの記録 | Workspace |
| スクリプトの認証の依頼の承認と、結果の受け取り（トークンの発行） | Token |
| 多要素認証の設定と解除、アクセス回復 | Identity |
| 初期構成の読み込み、設置の更新、ワークスペースの作成と利用終了、利用終了の完了、バックアップと復旧、状態確認 | Installation |

- Content と Schema のユースケースは、Webhook のポートを使わない。公開内容の変更が Webhook へ伝わる経路は「公開内容の整合性と通し番号」が定める
- 「区画をまたぐ手順」の表の所属の列は、この二つから導いた結果

## 区画

設置のデータは、区画に分かれる。区画は、一つの UnitOfWork が全部成功か全部反映されないかを共にできる範囲。

| 区画 | 数 | 持つもの |
| --- | --- | --- |
| 設置の区画 | 設置に一つ | ワークスペースをまたぐ集約と記録 |
| ワークスペースの区画 | ワークスペースごとに一つ | そのワークスペースの集約と記録、その区画が記録したドメインイベント |

- 一つの UnitOfWork が書く区画と、`fn` が受ける文脈は、[common.md](common.md) の「WriteScope」「UnitOfWorkProvider」に従う。区画をまたぐ読み取りも、同じ文書の「UnitOfWorkProvider」に従う。ポートが担保する一意性と参照整合性の範囲は、同じ文書の「TransactionalRepository」に従う
- 二つの区画をまたぐ整合は、一つの確定では保たない。区画ごとの UnitOfWork を決まった順に行い、どの段もやり直せる手順で保つ（「区画をまたぐ手順」）
- ワークスペースの区画は、`Workspace` の `insert` で始まる。区画の `Workspace` の状態とライフサイクルは、[workspace.md](workspace.md) の「Workspace」と「利用終了の消去」が定める。作成と利用終了が並行したときの規則は、[workspace.md](workspace.md) の「作成と利用終了の並行」が定める。名簿の項目と区画の `Workspace` の組ごとの出る遷移と主体は、`spec/flows/` の F-23 が持つ

### 集約とポートの置き場所

集約ごとの区画は「集約」の表にある。集約に数えない記録とポートは、次のとおり。

| 置き場所 | ポート |
| --- | --- |
| 設置の区画 | Identity の `AccountRepository`、`SessionRepository`、`RecoveryRequestRepository`。Workspace の `WorkspaceDirectory`（ワークスペースの名簿）と `MembershipIndex`（所属の索引）。Token の `DeviceAuthorizationRepository` と `DeclaredTokenIndex`（初期構成で宣言したトークンの索引）。Installation の `InstallationRepository` |
| ワークスペースの区画 | Workspace の `WorkspaceRepository`、`AccessPolicyRepository`、`MemberRepository`、`InvitationRepository`、`JoiningGuardRepository`。Audit、Schema、Content、Form、Webhook のすべてのリポジトリ。Token の `TokenRepository`、`PreviewExchangeRepository`、`PreviewSettingsRepository`。Media の `MediaRepository`、`MediaProfileRepository`、`MediaUsageRepository`、`MediaDerivativesRepository`、`StorageReservationRepository`、`ObjectRemovalRepository`。Content の `ContentStagingArea` と `ContentReferenceQuery`。`QuarantinedEventQuery` |
| 区画の外 | `BackupStore`、`RecoveryJournal`（復旧の記録）、`MediaStorage` |
| 区画に結び付かない | `EditingPresenceStore`、`UsageLimiter`、`InstallationEnvironment`、`SystemMailer`、`MailSender`、`WebhookSender`、`ImageProcessor`、状態を持たないポート（`Clock`、`IdGenerator`、`SecretCipher`、`SecretTokenIssuer`、`PasswordHasher`、`TotpAuthenticator`、`MediaDeliveryLinks`） |

- 区画の外のポートと、区画に結び付かないポートと UnitOfWork の関係は、[common.md](common.md) の「UnitOfWorkProvider」に従う
- スクリプトの認証の依頼（`DeviceAuthorization`）は、ワークスペースを選ぶ前に始まるので、設置の区画に置く
- ワークスペースの名簿は、Workspace が定義するポート `WorkspaceDirectory` が保存する。項目は [workspace.md](workspace.md) の「WorkspaceDirectoryEntry」が定める
- 名簿の状態は、準備中（`provisioning`）、開設済み（`open`）、終了中（`retiring`）。意味と遷移は [workspace.md](workspace.md) の「WorkspaceDirectoryEntry」が定める。名簿の項目が終了中のワークスペースを、「利用終了の途中」のワークスペースと呼ぶ。この語は、区画の `Workspace` の状態を指さない。区画の `Workspace` の状態は、「区画が `closing`」「区画が `closed`」と書く
- 所属の索引は、Workspace が定義するポート `MembershipIndex` が保存する。項目、所属の正、索引の用途は [workspace.md](workspace.md) の「Membership」「MembershipIndex」が定める。索引の書き込みと `Account` の関係は、[identity.md](identity.md) の「Account」が定める
- 宣言したトークンの索引は、Token が定義するポート `DeclaredTokenIndex` が保存する。持つものは [token.md](token.md) の「DeclaredTokenIndex」が定める

### ワークスペースを知らない要求

ワークスペースの区画の集約を、値だけで指す要求（トークンを示す要求、招待の URL、プレビューの引き換え、メディアの配信）は、値からワークスペースを求める。

- 一度きりの値の発行は、[common.md](common.md) の「期限つきの一度きりの値」の「値と区画」に従う。値を受けたユースケースは、`SecretTokenIssuer.workspaceOf` で値からワークスペースの ID を取り出し、その区画を読む
- 示されたトークンを確かめるユースケース（Token の `authenticateToken`）が確かめるトークンを決める規則は、初期構成で宣言したトークンの値の引き方を含めて、[token.md](token.md) の「TokenAuthentication」の `identify` と「DeclaredTokenIndex」が定める
- メディアの配信先は、`MediaDeliveryLinks` がワークスペースの ID を含めて作り、配信の要求から取り出す
- トークンでない値で、ワークスペースの ID を求められない値と、求めた区画に該当が無い値は、該当する集約が無い値として扱う。トークンの値から求めたワークスペースが名簿で開設済みでない場合の扱いは、「利用できるワークスペース」が定める

### 区画をまたぐ手順

二つの区画を書く操作は、この表だけが持つ。表は、手順の名前、区画の順、所属するドメインを持つ。書き込む段は、段ごとに一つの UnitOfWork。ほかの文書は、段の順をこの表への参照で示す。各段が呼ぶ振る舞いとポート、並行した手順との分岐、途中で止まったときの状態は、「手順ごとの規則」の表が挙げる定義元が定める。

| 手順 | 区画の順 | 所属するドメイン |
| --- | --- | --- |
| 参加（招待の受諾） | 設置（前段）→ ワークスペース（後段）→ 設置（確かめ直し） | Workspace（`acceptInvitation`）。アカウントの振る舞いは Identity |
| メンバーの利用の解除 | ワークスペース（メンバーの削除、操作履歴）→ 設置（後段。その組の所属の索引の修復） | Workspace（`removeMember`） |
| 所属の索引の修復 | 設置とワークスペース（読み取りだけ）→ ワークスペース（参加の取り下げ）→ 設置（項目の確定または除外） | Workspace（定期ジョブ「所属の索引の修復」の `repairMembershipIndex` と、`removeMember` の後段）。修復を行うユースケースは、この二つ |
| ワークスペースの作成 | 設置（名簿への登録）→ ワークスペース（区画の段）→ 設置（名簿を開設済みにする段）→ ワークスペース（招待の発行の段）→ 設置（作成を終える段） | Installation（`createWorkspace`）。招待の発行の段と作成を終える段は、`reissueInitialOwnerInvitation` も行う |
| 初期構成の読み込み | 設置（最初の段）→ ワークスペースごと（区画の段）→ ワークスペースごとに設置（設置の区画の段）→ 設置（最後の段） | Installation（`bootInstallation`） |
| ワークスペースの利用終了 | 設置（名簿の状態の変更）→ ワークスペース（開始）→ ワークスペース（区画の消去） | Installation（`closeWorkspace`） |
| 利用終了の完了 | ワークスペース（開始、区画の消去のうち、済んでいない段の続き）→ ワークスペース（Media の処理が残っていないことの確かめ。読み取りだけ）→ 区画の外（メディアの実体の消去）→ 設置（最後の段） | Installation（`completeWorkspaceClosures`）。定期ジョブ「利用終了の完了」が行う |
| スクリプトの認証の受け取り | 設置（前段）→ ワークスペース（後段） | Token（`redeemDeviceAuthorization`） |
| 定期ジョブ | 設置の区画が対象の定期ジョブは設置の区画、ワークスペースの区画が対象の定期ジョブは名簿のワークスペースごとの区画 | 各定期ジョブのドメイン（「定期ジョブ」） |
| 状態確認 | 名簿のワークスペースごとの区画（読み取りだけ） | Installation（`checkInstallationStatus`） |
| バックアップと復旧 | 取得（区画ごと）→ 復旧（設置のデータと、区画の外のメディアの実体）→ 利用再開前の処理（設置の区画とワークスペースの区画） | Installation（`takeBackup`、`restoreInstallation`、`resumeInstallation`） |

#### 手順を呼ぶ順

各ドメインが公開する手順を、Installation のユースケースが呼ぶ順は、この表だけが持つ。前提（先に終えているべきドメイン）は、前提を持つドメインの節だけが持つ。この表の順は、各ドメインの節の前提をすべて満たす順の一つで、前提に関わらないドメインは、前提を満たす位置の一つに置く。Installation のドメイン層とユースケースは、順を求める関数を持たず、この表を参照する。

| 手順 | 呼ぶ順 |
| --- | --- |
| 初期構成の宣言の検証と生成（各ドメインの「初期構成の宣言」） | Identity → Workspace → Schema → Content → Token → Form → Webhook |
| 利用再開前の処理（各ドメインの「利用再開前の処理」） | Workspace → Form → Schema |
| 利用終了の区画の消去（各ドメインの「利用終了の消去」） | Token → Form → Webhook → Content → Media → Schema → Audit → Workspace |

- 検証は、すべてのドメインを順に呼び、最初の問題で打ち切らない。一つの段の中の生成も、この順に呼ぶ
- 運用コマンドによるワークスペースの作成は、初期構成の順のうち、作成が作る集約を持つドメイン（Workspace、Schema、Content、Token）の生成を、宣言なしで呼ぶ
- 順に入るのは、その手順で行う処理を持つドメインだけ。利用再開前の処理は Workspace、Form、Schema が、利用終了の区画の消去は Identity と Installation を除くドメインが持つ
- 利用終了の完了の最後の段（設置の区画）の順は、Workspace の「利用終了の消去」が定める
- 各ドメインが「状態確認の項目」「利用再開前の処理」「利用終了の消去」に公開する項目の `kind` は、英小文字の語を `_` でつないだ識別子（`owner_vacant` の形）で書く。項目は、`domain` と `kind` の組で識別する。`kind` は、一つのドメインの中で一意で、違うドメインに同じ `kind` があってよい。ドメインは、項目の型（[common.md](common.md) の「各ドメインが公開する項目」）の `domain` が持ち、`kind` は、ドメインを区別する前置きを持たない（Workspace の `owner_vacant`、Form の `notifications_unsent`）。種類の名前だけを示す項目（利用終了で失われるデータの種類、CMS の通常データの外に残る情報、消去する資格情報）の `kind` は、その種類の名前を単数形で使う（Media の `media`、Form の `submission`、Audit の `audit_log`）。集約の名前が複数形の語を含む場合は、集約の名前のとおりに書く（Form の `form_settings`、Webhook の `webhook_endpoints`）

#### 手順ごとの規則

どの手順にも共通の規則。

- 前の段が確定してから、次の段を始める。後の段が確定しなくても、前の段の確定は取り消されない
- どの段も、もう一度行っても結果が変わらない。確定済みの段のやり直しは、変更の無い変更として通過する
- 途中で止まった手順が残す状態は、利用者に、所属や対象が無い場合と同じに見える

各段の中身、並行した手順との分岐、途中で止まったときの状態は、手順を所有する次の定義元だけが定める。この文書と `spec/flows/` は、状態の名前と、定義元への参照で示す。

| 手順 | 定義元 |
| --- | --- |
| 参加（招待の受諾） | 段の UnitOfWork の分け方は [../usecases/workspace.md](../usecases/workspace.md) の `acceptInvitation`、結果と途中で止まったときの状態は [../testcases/workspace/acceptInvitation.md](../testcases/workspace/acceptInvitation.md)。受諾の判定は [workspace.md](workspace.md) の「InvitationProcess」、取り下げは「MembershipRepair」 |
| メンバーの利用の解除 | [../usecases/workspace.md](../usecases/workspace.md) の `removeMember`。途中で止まったときの状態は [../testcases/workspace/removeMember.md](../testcases/workspace/removeMember.md) と [../testcases/workspace/repairMembershipIndex.md](../testcases/workspace/repairMembershipIndex.md) |
| 所属の索引の修復 | [workspace.md](workspace.md) の「MembershipRepair」。項目の世代の規則は「MembershipIndex」。定期ジョブの走査の進め方は [../usecases/workspace.md](../usecases/workspace.md) の `repairMembershipIndex` |
| ワークスペースの作成 | [workspace.md](workspace.md) の「初期構成の宣言」。利用終了と並行した場合は「作成と利用終了の並行」 |
| 初期構成の読み込み | [installation.md](installation.md) の「初期構成の読み込み」と、各ドメインの「初期構成の宣言」 |
| ワークスペースの利用終了、利用終了の完了 | [workspace.md](workspace.md) の「利用終了の消去」と、各ドメインの「利用終了の消去」。メディアの実体の消去の条件は [media.md](media.md) の「MediaClosure」。作成と並行した場合は [workspace.md](workspace.md) の「作成と利用終了の並行」 |
| スクリプトの認証の受け取り | 段の中身は [../usecases/token.md](../usecases/token.md) の `redeemDeviceAuthorization` のトランザクション境界、途中で止まったときの状態は [../testcases/token/redeemDeviceAuthorization.md](../testcases/token/redeemDeviceAuthorization.md)。結果の種類は [token.md](token.md) の「DeviceAuthorizationProcess」 |
| 定期ジョブ、状態確認 | 下の「定期ジョブと状態確認」 |
| バックアップと復旧 | 下の「バックアップと復旧」。置き換えの契約は [installation.md](installation.md) の `BackupStore.restore`、処理の中身は各ドメインの「利用再開前の処理」 |

定期ジョブと状態確認。

- ワークスペースの区画が対象の定期ジョブは、名簿のワークスペースを順に取り、ワークスペースごとにその区画の処理を行う。定期ジョブごとの対象の名簿の状態（開設済みだけ、開設済みと終了中、終了中だけ）は、「定期ジョブ」の表が定める。準備中のワークスペースは、どの定期ジョブの対象にもならない
- 一つのワークスペースの失敗の扱いは、「イベントの消費者と定期ジョブ」が定める
- ワークスペースをまたぐ対象は、名簿から求める。ポートの読み取りは、一つの区画に限る
- 状態確認は、名簿で開設済みのワークスペースの区画ごとの値（各ドメインの項目と、隔離されたイベントの件数）を足す。足した値は、同じ時点の件数を約束しない。準備中と終了中のワークスペースは、区画を読まず、名簿の状態で示す。示す項目と必要な対処は、[workspace.md](workspace.md) の「状態確認の項目」が定める

バックアップと復旧。

- バックアップの写しの時点は、[installation.md](installation.md) の「BackupStore」が定める
- 復旧の後に残り得る区画の間の食い違いは、名簿の状態と区画の `Workspace` の食い違い、アカウントが無いメンバー、所属の索引と `Member` の食い違い、スクリプトの認証の依頼とトークンの食い違い。区画ごとの写しの時点の差と、復旧の前から続く手順の後の段（[installation.md](installation.md) の「BackupStore」）から生じる
- 利用再開前の処理が、利用の再開の前に、ワークスペースの区画を正として、各ドメインの「利用再開前の処理」が定める食い違いを直し、復旧点で終えていなかった処理を片付ける。対象にするワークスペースの名簿の状態は、各ドメインの「利用再開前の処理」が定める。順は「手順を呼ぶ順」が定める。スクリプトの認証の依頼とトークンの食い違いは、復旧点の依頼が [token.md](token.md) の「DeviceAuthorization」のライフサイクルのとおり期限で終わることで解ける。受け取りの後の段が置き換えの後に確定した場合の扱いは、Token の `redeemDeviceAuthorization` が定める
- 残件数は、利用再開前の処理が済ませていない対象の数。意味はこの一つで、ドメインごとに、区画の記録から数える。処理を行う実行と、利用再開前の確認は、同じ数え方をする。数え方は各ドメインの「利用再開前の処理」が、利用の再開の可否は [installation.md](installation.md) の `Recovery.assertResumable` が定める
- 利用再開前の処理は、その実行で失敗した対象を読み飛ばし、同じ実行でやり直さない。失敗した対象は残件数に残り、次の実行が扱う
- 利用終了の途中のワークスペースは、利用の再開の後に、定期ジョブ「利用終了の完了」が完了させる

## 集約

| ドメイン | 集約 | 区画 | 概要 |
| --- | --- | --- | --- |
| Identity | Account | 設置 | 設置に一つの人のアカウント。資格情報と、多要素認証の設定を持つ |
| Identity | Session | 設置 | 人のセッション |
| Identity | RecoveryRequest | 設置 | アクセス回復の依頼。メールで届ける回復と、運用コマンドで発行する管理者復旧 |
| Workspace | Workspace | ワークスペース | ワークスペースの区画の状態 |
| Workspace | AccessPolicy | ワークスペース | アクセスの方針。所有者の指定（`designated` か `vacant`）と多要素認証の必須。定義は [workspace.md](workspace.md) の「AccessPolicy」 |
| Workspace | JoiningGuard | ワークスペース | 参加の囲い。アカウントごとの、進行中の参加の世代。`AccountId` で識別する。定義は [workspace.md](workspace.md) の「JoiningGuard」 |
| Workspace | Member | ワークスペース | アカウントのワークスペースへの所属と権限の組 |
| Workspace | Invitation | ワークスペース | 対象と有効期限を持つ招待 |
| Audit | AuditLogEntry | ワークスペース | 操作履歴の一件。追記だけを行う |
| Token | Token | ワークスペース | アプリのトークン。種類、許可、範囲、紐付け先、有効期限、利用元の限定、利用量の制限を持つ |
| Token | PreviewExchange | ワークスペース | プレビューを開く操作で表示アプリへ渡す、一度だけ使える引き換えの値の記録 |
| Token | DeviceAuthorization | 設置 | スクリプトの実行環境で、メンバーの承認を経てトークンを発行するスクリプトの認証の依頼 |
| Token | PreviewSettings | ワークスペース | プレビュー先の表示アプリの URL |
| Schema | WorkspaceSchema | ワークスペース | ワークスペースの現行スキーマ、版、適用済みの移行定義の列、モデル・フォームの ID、反映の進行 |
| Schema | SchemaVersionRecord | ワークスペース | スキーマの版の記録の一件。追記だけの集約 |
| Schema | SchemaApplyJob | ワークスペース | 受け付けた反映・事前確認の一回。種類（`mode`）、準備の進行（`Preparation`）、終端の結果（成功と失敗）を持つ |
| Media | Media | ワークスペース | メディア一件。登録時の記録、公開範囲 |
| Media | MediaProfile | ワークスペース | メディア一件の管理情報（管理用の名称、検索情報、共有する説明）。`MediaId` で識別する。定義は [media.md](media.md) の「MediaProfile」 |
| Media | MediaDerivatives | ワークスペース | 派生形式の記録。画像一件の、派生形式ごとの状態。`MediaId` で識別する |
| Content | Content | ワークスペース | コンテンツ一件。コンテンツID、公開内容、下書き、置き場所（通常・ごみ箱） |
| Content | Revision | ワークスペース | 履歴の版 |
| Content | PublicationLedger | ワークスペース | ワークスペースに一つの、公開の変更の記録の台帳。確定の順の通し番号を持つ。公開内容を変える操作ごとの記録（`PublicationRecord`）は、台帳のポートが追記して保存する |
| Content | ContentRetentionPolicy | ワークスペース | 履歴とごみ箱の保持方針 |
| Content | EditingPresence | 区画に結び付かない | あるコンテンツをあるメンバーが編集していることの、短命な記録 |
| Form | FormSettings | ワークスペース | フォーム一つの運用設定（受付状態、通知先、通知定義、回答の保持方針） |
| Form | PendingFormSettings | ワークスペース | 紐付け待ちの設定。初期構成でフォームのキーを指定して宣言した運用設定。フォームのキーで識別する。定義は [form.md](form.md) の「PendingFormSettings」 |
| Form | MailConnection | ワークスペース | ワークスペースの送信接続設定 |
| Form | Submission | ワークスペース | 回答一件。受付時に確定した内容の写し（フォームの定義、通知定義、通知対象）と、保持方針 |
| Form | SubmissionNotification | ワークスペース | 回答一件の通知の進行と、宛先ごとの最新の送信の結果。`SubmissionId` で識別する |
| Webhook | WebhookEndpoints | ワークスペース | ワークスペースに一つの、表示アプリへの通知の宛先の列。宛先ごとに URL と秘密情報を持つ |
| Webhook | ChangeNotification | ワークスペース | 一つの操作で確定した公開内容の変更をまとめた通知 |
| Webhook | WebhookDelivery | ワークスペース | 通知と、通知を作る時点の宛先の組ごとの、送信の状態。通知の ID と宛先の ID の組で識別する |
| Installation | Installation | 設置 | 設置に一つ。初期構成の記録、保存済みのデータの版、更新の進行。設置のデータの中にあり、バックアップが含む |
| Installation | Recovery | 区画の外 | 復旧の記録。設置に一つ。復旧の進行と対象のバックアップ。設置のデータの外にあり、`RecoveryJournal` が保存する |
| Installation | BackupRecord | 区画の外 | バックアップの一件の記録。取得日時、整合性の確認の結果、保持の期限。設置のデータの外にあり、`BackupStore` が保存する |

- `Media` と `MediaDerivatives` は別の集約。保存の競合は [media.md](media.md) の「MediaDerivativesRepository」が定める
- `Submission` と `SubmissionNotification` は別の集約。作成と削除の対応は [form.md](form.md) の `SubmissionRepository` の「回答と通知の対応」が定める
- `ChangeNotification` と `WebhookDelivery` は別の集約。それぞれの規則は [webhook.md](webhook.md) の「ChangeNotification」「WebhookDelivery」が定める
- Media の `StorageReservation`（保管の予約）、`ObjectRemoval`（消去待ち）、メディアの使用箇所と、スキーマの反映の準備領域の使用箇所（`MediaUsageRepository`）は、Media のポートが保存する記録で、集約に数えない。どれもワークスペースの区画にある
- ワークスペースの名簿（`WorkspaceDirectory`）と所属の索引（`MembershipIndex`）は Workspace のポートが、宣言したトークンの索引（`DeclaredTokenIndex`）は Token のポートが、設置の区画に保存する記録で、集約に数えない
- `EditingPresence` は `EditingPresenceStore` が、`Recovery` は `RecoveryJournal` が、`BackupRecord` は `BackupStore` が読み書きする

Schema ドメインのスキーマ・移行定義・識別子・検証・差分からの生成は、保存を伴わない純粋な値と関数として定める。CMS の反映時の検証と、CMS に接続しない開発環境の検証・型・識別子・移行定義の生成は、同じ関数を使う。

## 共通の型とポート

すべてのドメインが使う。定義は [common.md](common.md) だけが持ち、この表は名前と役割を示す。置き場所は `docs/backend_implementation_example.md` のファイル配置に従う。型、`TransactionalRepository`、`SingletonRepository` は `packages/core/src/domain/common/`、各集約のリポジトリのポートは `packages/core/src/domain/${domain}/ports/`、下の表の種類が「ポート」のものは、ユースケースが使うポートで、`packages/core/src/application/` に置く。ドメイン層は application のポートを使わない。

| 名前 | 種類 | 役割 |
| --- | --- | --- |
| 集約の ID | 値オブジェクト | 全集約の ID と、公開内容を変える操作の ID（`OperationId`） |
| `EmailAddress` | 値オブジェクト | メールアドレス |
| `Permission` | 文字列リテラルの直和型 | 操作に必要な権限の語彙。権限ごとの意味と権限の組は [workspace.md](workspace.md) |
| `Actor` | 直和型 | 操作の実行者 |
| `TokenDesignation` / `MemberTokenOrigin` | 直和型 | トークンを指す表示の元（アプリに紐付くトークンの名称、メンバーに紐付くトークンの発行の経路） |
| `FailureHandling` | 直和型と遷移 | 送信失敗の対処の状態 |
| `Pagination` / `PaginationResult<T>` | 値オブジェクトと型 | ページの指定と、ページの結果 |
| `BatchTargets<T>` | 値オブジェクト | 一度の操作で扱う対象の列 |
| `PublishedChangeNotice` / `PublishedChangeList` | 値 | 表示アプリへの通知に載せる公開内容の変更 |
| 期限つきの一度きりの値 | 規則 | 照合値で引く値を持つ集約に共通の規則 |
| `DeclarationIssue` | 型 | 初期構成の宣言の問題の一件 |
| `DomainKind` / `DomainCount` / `StatusItem` / `ResumeItemDefinition` / `ResumeItem` | 型 | 各ドメインが状態確認、利用終了の消去、利用再開前の処理に公開する項目の一件 |
| `KeyBinding<TId>` | 直和型と関数 | キーで宣言した設定の、対象の ID への紐付け |
| `ProtectedSecret` | 値オブジェクト | `SecretCipher` で保護した秘密情報 |
| `UsageLimit` / `UsageStanding` | 型と直和型 | 利用量の制限の上限と、キーの制限の状態 |
| `TransactionalRepository<TEntity, TId>` / `SingletonRepository<TEntity>` | ポートの基底 | 楽観ロックで保存するリポジトリの基底と、すべてのリポジトリに共通の規則 |
| `WriteScope` | 直和型 | 一つの UnitOfWork が書く区画と、確定の時点の条件 |
| `UnitOfWorkProvider` | ポート | 一つの区画の書き込みとドメインイベントの記録を、一つの確定にまとめる |
| `PartitionReader` | ポート | UnitOfWork の外の読み取りに使うリポジトリを、区画ごとに返す |
| `Clock` | ポート | 現在時刻 |
| `IdGenerator` | ポート | ID の生成と形式の検査 |
| `SecretCipher` | ポート | 後で元の値が要る秘密情報の保護 |
| `SecretTokenIssuer` | ポート | 推測できない値の生成と、保存する照合値の計算 |
| `SystemMailer` | ポート | 設置全体のメール送信。ワークスペースの送信接続設定（Form の `MailSender`）と別 |
| `UsageLimiter` | ポート | 利用量の制限 |
| `QuarantinedEventQuery` | ポート | 隔離されたドメインイベントの件数の、ワークスペースの区画ごとの照会 |
| ドメインイベント | 仕組み | 集約の振る舞いが記録し、消費者へ配送する出来事 |

エラーの種類は次のとおり。presentation は種類から応答を決める。理由のコードは、各ドメインの `ErrorCode` と、入口だけが返す理由コードの表（[../usecases/app-api.md](../usecases/app-api.md) の「入口の理由コード」）がすべて列挙する。共通のコードは [common.md](common.md) の「エラー」にある。

| 種類 | 意味 |
| --- | --- |
| `BusinessRuleError` | 集約の状態に依る規則の違反、利用者の入力に由来しない値の不変条件の違反、値オブジェクトの生成では確かめない、入力の定義の間の整合の違反（スキーマの定義どうしの不整合、移行定義どうしの不整合など、個々の値は形に合うが、組として成り立たない入力） |
| `ValidationError` | 入力の形の誤りと、利用者の入力から値オブジェクトを作るときの違反（メールアドレス、名前、パスワード、検索の条件など）。項目ごとの理由を持つ |
| `NotFoundError` | 対象が存在しない。許可されていないワークスペースの対象も含む。対象の種類（[common.md](common.md) の「エラー」の `NotFoundTarget`）と、利用者が指定したキーのうち、存在しないものをすべて持つ。一度きりの値（[common.md](common.md) の「期限つきの一度きりの値」）で引く対象（回復依頼、招待、引き換えの値など）は、対象の種類だけを持ち、キーを持たない |
| `ConflictError` | 楽観ロックの競合、一意性・参照整合性・公開内容の整合性の違反、`WriteScope` の条件を満たさないこと（確定の時点でスキーマが変わった、ワークスペースが `active` でない）、復旧の置き換えと重なった呼び出しと UnitOfWork |
| `UnauthenticatedError` | セッション・トークンが無い、失効、期限切れ。ログインの資格情報と確認コードの照合の失敗 |
| `ForbiddenError` | 権限・許可・範囲の不足、受け付けない種類の実行者 |
| `RateLimitedError` | トークンの利用量の制限、フォームの送信制限、ログインの試行の制限 |
| `UnavailableError` | スキーマの反映の適用中、設置が稼働中でない間、メディアの保管先の容量の不足 |
| `SystemError` | 上記以外の失敗 |

## ドメインをまたぐ規則

### 利用できるワークスペース

名簿で開設済みでないワークスペースは、どの入口も受け付けない。入口ごとの結果は、この節の表だけが定める。各入口のドメインとユースケースは、この節を参照し、別の扱いを書かない。

- 利用できるワークスペースは、Workspace の `WorkspaceAvailability.of`（[workspace.md](workspace.md)）が真を返すワークスペース。判定は、この関数だけが持つ
- 入口のユースケースは、設置の区画から名簿の項目（`WorkspaceDirectory.findById(workspaceId)`）を `PartitionReader` で読み、その状態と、区画の `Workspace` の状態から、`WorkspaceAvailability.of` で可否を求める。表の振る舞いは、この結果でワークスペースが利用できるかどうかを決め、別の判定を持たない。Token と Media の振る舞いは、ユースケースが求めた真偽を受け取り、Workspace の型を使わない
- 値からワークスペースを求める入口（示されたトークンを確かめる段、プレビューの引き換え、スクリプトの認証の受け取り、招待の内容の確認と受諾）は、区画を読む前に名簿を確かめる。名簿で開設済みでなければ（項目が無い場合を含む）、区画を読まずに、表の結果を返す。名簿に項目があるワークスペースへの結果は、利用終了の段（名簿だけが終了中、区画の `Workspace` が `closing`、区画を消去した後）によらない
- 名簿に項目が無いワークスペース（利用終了の完了の後、存在しない ID、偽造の値）への結果は、表の右の列が定める。どの入口も、項目が無い場合に、項目がある場合と同じ結果を返す
- 値からワークスペースの ID を求められない値の扱いは、「ワークスペースを知らない要求」に従う。名簿を読まない
- 受け付けない種類の実行者の拒否、メンバーの認可の拒否、トークンの認可のうち `TokenAuthorization.resolve` の拒否、利用できないワークスペースへの表の結果は、スキーマの反映の適用中（`SCHEMA_APPLYING`）など、ワークスペースの区画の状態に依るほかの拒否より先に返す。`TokenAuthorization.authorize` の拒否（許可、範囲、権限）は、各ユースケースの拒否の順の位置で返す。各ユースケースが定める拒否の順は、これに従う

| 入口 | 可否を決める振る舞い | 名簿に項目があり、利用できないワークスペースへの結果 | 名簿に項目が無いワークスペースへの結果 |
| --- | --- | --- | --- |
| メンバーの認可（管理画面の操作、プレビューを開く操作、スクリプトの認証の承認） | Workspace の `MemberAuthorization.resolve` | `not_available`。`NotFoundError` | 同じ |
| 示されたトークンを確かめる段 | Token の `authenticateToken`（[token.md](token.md) の「TokenAuthentication」） | `workspace_not_available`。エラーは [token.md](token.md) の「エラーコード」が定める。区画のトークンを引かない | 同じ |
| トークンの認可 | Token の `TokenAuthorization.resolve` | `workspace_not_available`。エラーは [token.md](token.md) の「エラーコード」が定める | 同じ。確かめる段を通った後に、利用終了が完了した要求が当たる |
| フォームの受付 | トークンの認可を通る | トークンの認可と同じ。応答への写し方は [../usecases/app-api.md](../usecases/app-api.md) | トークンの認可と同じ |
| メディアの配信 | Media の `MediaDelivery.decide` | `deny` | 同じ |
| プレビューの引き換え | Token の `PreviewExchangeProcess` | `refused`（`member_unavailable`）。エラーは [token.md](token.md) の「エラーコード」が定める | 同じ |
| スクリプトの認証の受け取り | Token の `DeviceAuthorizationProcess` | `refused`（`DeviceRefusal` の `member_unavailable`）。エラーにしない。前段を確定した後に利用できなくなった受け取りの結果は、[token.md](token.md) の「DeviceAuthorizationProcess」が定める | 同じ |
| 招待の内容の確認と受諾 | Workspace の `InvitationProcess` | `BusinessRuleError`（`InvitationInvalid`） | 同じ |
| ワークスペースに属さない操作が示す所属ワークスペース（所属するワークスペースの一覧、ログイン後の行き先、スクリプトの認証の依頼の確認が示す、選べるワークスペースと、承認の時点のワークスペース） | Workspace の `MemberAuthorization.resolve`。ワークスペースごとに求める | `not_available`。そのワークスペースを示さない。エラーにしない | 同じ |

- トークンの認可の結果の種類（`workspace_not_available` と `member_removed` の別）は [token.md](token.md) の「TokenAuthorization」「TokenAccess」が、初期構成で宣言したトークンの値への結果は、同じ文書の「TokenAuthentication」「DeclaredTokenIndex」「利用終了の消去」が定める
- 可否は、名簿の項目を読んだ時点で決まる。読んだ後に名簿が終了中になった要求の書き込みが確定するかどうかは、「ワークスペースの作成と利用終了」が定める
- イベントの消費者と定期ジョブは、入口でない。扱いは「イベントの消費者と定期ジョブ」が定める
- 通知を作る、または送る、イベントの消費者と定期ジョブのユースケース（Webhook と Form）は、通知を作る前と送る前に、入口と同じに名簿の項目を読む。利用できないワークスペースでは、対象が無い場合と同じに、エラーにせずに終える。通知を作らず、送らず、送信失敗も記録しない

### 認可

操作の可否は、ユースケースが `Actor` から求めた権限の組で判定する。権限の語彙（`Permission`）の型は [common.md](common.md) が、権限ごとの意味、権限の組、標準の役割との対応は Workspace が定める。ワークスペースを対象にするユースケースの入力は、`Actor` と `WorkspaceId`。

- `member`: そのメンバーが現在持つ権限
- `appToken` と `memberToken`: トークンの許可と範囲と、Token の `TokenAuthorization.resolve`（[token.md](token.md) の「TokenAuthorization」）が求める権限の組
- `operator` と `system`: 権限の組を持たない。受け付ける実行者の種類だけで判定する
- 管理画面とトークンのどちらの経路も、同じユースケースを通る。認可と入力検証はユースケースとドメインにあり、経路ごとに持たない
- ワークスペースに属さない操作（所属するワークスペースの一覧、招待の内容の確認と受諾、スクリプトの認証の依頼の確認・承認しない操作、自分のアカウントの認証の設定）は、セッションで確かめたアカウント（`AccountId`）を受け取る。メンバーの認可の手続きを行わず、トークンでは行えない。これらの操作が示す所属ワークスペース（所属するワークスペースの一覧、ログイン後の行き先、スクリプトの認証の依頼の確認が示すワークスペース）は、[../usecases/workspace.md](../usecases/workspace.md) の「所属ワークスペースを求める手続き（`resolveAccountWorkspaces`）」が求める。結果は「利用できるワークスペース」の表に従う。この手続きと、メンバーの認可の手続きは、読み取りだけを行う
- 各ユースケースは、受け付ける実行者の種類（`member` / `appToken` / `memberToken` / `operator` / `system`）を定める。受け付けない種類の実行者の要求は `ForbiddenError`（理由 `ACTOR_NOT_ALLOWED`）。`Actor` を入力に持たないユースケースは、この判定を持たない

`Actor` は入口が求めて、ユースケースへ渡す。入口は、実行者が誰かを確かめるだけで、操作ごとの認可を行わない。

| 入口 | 確かめること | 作る `Actor` |
| --- | --- | --- |
| 管理画面 | セッションのアカウントと、対象のワークスペースのメンバー | `member` |
| アプリ向けの HTTP API | 示されたトークン。Token の `authenticateToken`（[../usecases/token.md](../usecases/token.md)）が確かめる | `authenticateToken` が返す `appToken` または `memberToken` |
| 運用コマンド（`apps/cli`） | 実行基盤の管理の資格で設置に接続できること。接続できた実行者を、ホスティング運営者として扱う | `operator` |
| 定期ジョブとイベントの消費者 | なし | `system` |

- 運用コマンドの入口は `apps/cli`。実行基盤の管理の資格で設置に接続し、ユースケースを直接呼ぶ。HTTP の経路を持たない。運営者であることは、実行基盤の管理の資格が担保する
- `operator` の `Actor` を作るのは、運用コマンドの入口だけ。`apps/web` の入口（管理画面、アプリ向けの HTTP API、ワーカー）は `operator` を作らない
- 設置の状態がそのコマンドを受け付けることの確かめ方は、「設置の状態」が定める

操作ごとの認可（必要な権限・許可と、対象の範囲）は、各ユースケースが行う。ワークスペースの区画を書くユースケースは、操作を確定する UnitOfWork の中で、書き込みの前に行う。設置の区画を書くユースケース（スクリプトの認証の依頼の承認など）は、UnitOfWork の前に行う。読み取りだけのユースケースは、返す内容を読む前に行う。対象の属性で可否が決まる読み取り（その操作を行ったメンバーにだけ示す通知の送信の状態など）は、対象を読んでから判定し、判定の前に対象の内容を返さない。認可は操作のたびに行い、結果を要求をまたいで保持しない。権限の変更、利用の解除、所有者の変更、多要素認証の必須化、トークンの失効は、確定した次の操作から現れる。

メンバーの認可の手続き。`Actor` が `member` のユースケースは、次を行う。

1. 設置の区画から名簿の項目（`WorkspaceDirectory.findById(workspaceId)`）を、ワークスペースの区画から `WorkspaceRepository.find()`、`AccessPolicyRepository.find()`、`MemberRepository.findById(actor.memberId)` を、設置の区画からそのメンバーのアカウントの `AccountRepository.findById` を読む。書き込みを持つユースケースは、ワークスペースの区画の UnitOfWork の中で、`Workspace`、`AccessPolicy`、`Member` を文脈のリポジトリから、`Account` と名簿の項目を `PartitionReader` から読む。設置の区画を書くユースケースは、どれも UnitOfWork の前に読む
2. 読んだ名簿の状態、`Workspace`、`Member`、`Account.isMfaEnabled` の結果を `MemberAuthorization.resolve` に渡す
3. 結果が `not_available` なら `NotFoundError`。対象の種類は `NotFoundTarget` の `workspace`、キーはワークスペースの ID。ワークスペースの情報を示さない
4. 結果が `mfa_setup_required` なら `ForbiddenError`（理由 `MFA_SETUP_REQUIRED`）
5. 結果が `granted` で、`MemberGrant.permissions(grant)` と、ユースケースが必要とする権限に `PermissionSet.missing` が空でなければ、不足する権限を添えた `ForbiddenError`（理由 `PERMISSION_MISSING`）。所有者だけの操作で `MemberGrant.isOwner(grant)` が偽なら `ForbiddenError`（理由 `OWNER_ONLY`）

トークンの認可の手続き。トークンの許可を必要とするユースケースは、Token の `authorizeTokenOperation`（[../usecases/token.md](../usecases/token.md)。`TokenAuthorization.resolve` と `TokenAuthorization.authorize`）で、操作が必要とする許可と、対象が範囲にあることを判定する。

- 対象の解決、`TokenAuthorization` が受け取るもの、拒否の理由は [token.md](token.md) の「TokenAuthorization」が、エラーは同じ文書の「エラーコード」が定める

メディアの配信は、`Actor` を受け取らない。入口が渡すものは [../usecases/app-api.md](../usecases/app-api.md) の「メディアの配信」が、ユースケースが読むものは [../usecases/media.md](../usecases/media.md) の「メディアを配信する（`deliverMedia`）」が定める。非公開のメディアへの、アカウントがある要求では、配信先のワークスペースのそのアカウントのメンバーについてメンバーの認可の手続きの手順 2 を行い、結果が `granted` かどうかを `MediaAccess.resolve` に渡す。配信の資格は `MediaAccess.resolve` が、配信の可否は `MediaDelivery.decide` が決める（[media.md](media.md)）。ワークスペースが利用できるかどうかの扱いは「利用できるワークスペース」に従う。

認可の後に利用終了の開始が確定した操作は、UnitOfWork の確定が `ConflictError`（理由 `WORKSPACE_NOT_ACTIVE`）で拒否される。入口の種類ごとの写し方は、「競合の扱い」の表が定める。

各ユースケースのファイルは、必要な権限と許可（所有者だけの操作はその旨）と、受け付ける実行者の種類だけを書く。手続きは、この節への参照で示す。

### 変更の無い変更

現在と同じ値への変更は、成功として現在の状態を返し、何も書かない。集約の `version` は進まない。どの変更が当たるかは、各集約の振る舞いが定める。

### 競合の扱い

競合と失敗した書き込みをやり直さないことは、[common.md](common.md) の「TransactionalRepository」の、やり直さない規則だけが定める。

UnitOfWork の確定の時点の `ConflictError` の写し方は、入口の種類ごとに、この表だけが定める。各ユースケースは、表のどの種類に当たるかで写し方が決まり、エラーの対応を自分のファイルに持たない。どの種類も、確定をやり直さない。

| 入口の種類 | 当たるユースケース | 理由 `WORKSPACE_NOT_ACTIVE` | ほかの理由 |
| --- | --- | --- | --- |
| 要求の入口 | 管理画面とアプリ向けの HTTP API から呼ぶユースケースのうち、下の二つに当たらないもの | 認可の該当なしと同じ結果（`NotFoundError`）。利用できないワークスペースへの操作と同じに見える | そのまま返す。エラーを結果の型に写すユースケースは、その写し方に従う |
| 値の引き換え | 示された値が指す集約の手続きを確定するユースケース（プレビューの引き換え、スクリプトの認証の受け取り、招待の受諾） | その手続きが定める結果の直和の一つ。定義元は、Token の `PreviewExchangeProcess` と `DeviceAuthorizationProcess`、Workspace の `InvitationProcess` | その手続きが結果を定める理由は、その結果。定めない理由は、そのまま返す |
| アカウントの有無を明かさない要求 | 応答をアカウントの有無で分けないユースケース（Identity の `requestAccessRecovery`） | 当たらない（設置の区画だけを書く） | エラーにせず、要求を受け付けた結果 |
| 運用コマンド | `operator` のユースケース | そのまま返す | そのまま返す |
| イベントの消費者と定期ジョブ | `system` のユースケース | 「イベントの消費者と定期ジョブ」に従う | 「イベントの消費者と定期ジョブ」に従う |

所属の索引の修復の UnitOfWork は、どの入口から呼ばれても、この表に当たらない。写し方は [workspace.md](workspace.md) の「MembershipRepair」が定める。

### URL の組み立て

対象は、人に渡す一度きりの値を含む、CMS の画面の URL。招待の URL、回復用の URL、スクリプトの認証の承認用の URL が当たる。次の分担で組み立てる。

- ユースケースは、値を返し、これらの URL を出力に持たない
- 経路の形と、URL を作る関数は、値を発行するドメインが持つ。招待は Workspace の `InvitationLink.url`、回復は Identity の `RecoveryLink.url`、承認は Token の `DeviceApprovalLink.url`（[token.md](token.md)）。関数は、[common.md](common.md) の「期限つきの一度きりの値」の URL の組み立てに従う
- 画面、運用コマンドの結果、HTTP API の応答に示す URL は、入口が、ユースケースの返した値から、この関数で組み立てる
- メールの本文に入れる URL は、ユースケースが、設置の公開の URL と値から、同じ関数で作る。設置の公開の URL は、`InstallationEnvironment.publicUrl()` が返す（[installation.md](installation.md)）。入口は、URL を組み立てる関数をユースケースへ渡さない
- 設置全体のメール送信を使えない設置（`SystemMailer.isConfigured` が偽）での値の扱いは、値を持つ集約（[workspace.md](workspace.md) の「Invitation」、[identity.md](identity.md) の「RecoveryRequest」）が定める。ユースケースが値を出力に返した場合は、入口が URL を組み立てて示す

表示アプリの URL は、この規則の対象でない。プレビュー先の URL（表示アプリの URL、モデルの経路、引き換えの値から Token の `PreviewDestination` が作る）と、プレビュー先の設定の URL は、ユースケースが入出力に持つ。

### モデルとフォームの ID

`WorkspaceSchema` は、モデルとフォームごとに、キーとは別の変わらない `ModelId` / `FormId` を割り当てる。

- 割り当ての規則は、[schema.md](schema.md) の「モデルとフォームの ID の割り当て」が定める
- モデル・フォームに紐付く運用設定（`FormSettings`、トークンの範囲）は、`ModelId` / `FormId` で対象を指す
- 初期構成でキーを指定して宣言した運用設定の紐付けは、[common.md](common.md) の「キーで宣言した設定の紐付け」が定める

### 公開内容の整合性と通し番号

公開内容を変える操作は、Content の `PublicationOperation`（[content.md](content.md) の「操作の種類」）が定める。

公開内容の整合性は、ポートが担保する。Content と Media のポートの契約が、UnitOfWork の確定の時点で、次の五つの違反を `ConflictError` として拒否する。並行した書き込みでも成り立つ。拒否する確定の条件は、表の定義元のポートの契約だけが定める。

| 理由 | 定義元 |
| --- | --- |
| `UNIQUE_VALUE_TAKEN`、`REFERENCE_NOT_PUBLISHED`、`PUBLISHED_REFERENCE_REQUIRED` | Content の `ContentRepository`（[content.md](content.md)） |
| `MEDIA_IN_PUBLIC_USE` | Media の `MediaRepository`（[media.md](media.md)） |
| `MEDIA_NOT_PUBLIC` | Media の `MediaUsageRepository`（[media.md](media.md)）。Content のユースケースが、内容の保存と同じ UnitOfWork で使用箇所を書く |

- ユースケースは、事前に読んで利用者に違反を示すが、整合性はポートの拒否で決まる
- 利用者の確認を伴う操作の確定の要求は、その操作が確認の後の変化として挙げる対象（[../scenario/index.md](../scenario/index.md) の「確認した内容での確定」）の `version` を添える。確認の後にその対象が変わった確定は `ConflictError`（理由 `STALE_VERSION`）、確認の後にほかの公開内容が変わって整合性を満たさなくなった確定は上の理由で拒否する。拒否された確定は、確認を示し直し、変わった対象があれば示す

公開内容を変える操作は、一回ごとに、公開の変更の記録を一件追記し、ドメインイベント `content.publication_changed` を一つ記録する。操作と同じ UnitOfWork で確定することは「同じ UnitOfWork で確定するもの」が定める。

- 記録の型、通し番号の割り当て、追記の規則は、[content.md](content.md) の「PublicationLedger」と `PublicationLedgerRepository` の契約だけが定める。記録が持つ変更の一覧と参照元の一覧は、`PublishedChangeList`（[common.md](common.md) の「PublishedChangeNotice」）
- イベントの型は [content.md](content.md) の「ドメインイベント」が、消費者が行うことは「ドメインイベントで行うもの」が定める
- 通し番号は、表示アプリが変更の順序を判別する情報になる。表示アプリから見た通し番号の規則と、バックアップからの復旧の後の通し番号の規則は、[webhook.md](webhook.md) の「通し番号と復旧」だけが定める
- スキーマの反映の確定が確定できない場合の扱いは、[schema.md](schema.md) の「確定できない場合の扱い」が定める

操作の ID（`OperationId`）は、公開内容を変える操作の一回を指す。

- 公開内容を変える操作を始める主体が、`IdGenerator.generate` で作る。要求の入口、またはスキーマの反映の確定のユースケース（Schema の `commitSchemaApply`）が当たる。入口は、作った ID を、公開内容を変える Content のユースケースへ入力として渡す
- 管理画面の操作では、ユースケースが、入力に受けた操作の ID を出力に返す。メンバーは、その ID で、その操作の通知の送信の状態（Webhook）を引く。通し番号で引かない
- `SchemaApplyJobId` は、受け付けた反映・事前確認の一回を表す集約 `SchemaApplyJob` の ID。単独の「ジョブ」は、この集約だけを指す。定期的に起動する処理は、常に「定期ジョブ」と書く
- 操作の ID は、公開の変更の記録と `ChangeNotification` が持つ

### 参照の記録と保護

コンテンツの内容（公開内容・下書き・履歴の版の、すべての項目の値）は、メディアと他のコンテンツを ID で参照する。何が参照になるかは、[content.md](content.md) の `ContentReferences.of` と `MediaUsages` が定める。コンテンツへの参照は Content のリポジトリが、メディアへの参照は Media の `MediaUsageRepository` が、内容の保存と同じ UnitOfWork で記録する。

- メディアの使用箇所の記録は、Media のポート `MediaUsageRepository`（[media.md](media.md) の「MediaUsageRepository」）が持つ一つだけ。メディアごとの使用箇所の読み取りも、このポートが持つ。Content の参照の記録は、メディアへの参照を持たない
- Content のドメインサービスが、内容の変更から、持ち主ごとの使用箇所（`holderId` と `MediaUse` の列）を求める。Content のユースケースが、内容の保存と同じ UnitOfWork で、`MediaUsageRepository` の一括のメソッドを呼ぶ。使用箇所を求めるのは Content の `MediaUsages`。メソッドの契約は [media.md](media.md) の「MediaUsageRepository」が定める
- Content のポート（`ContentRepository`、`RevisionRepository`、`ContentStagingArea`）の契約と実装は、メディアの使用箇所を読み書きしない
- スキーマの反映は、変換結果の使用箇所を、Media のポートの準備領域に書き、確定が、準備領域の使用箇所へ切り替える
- `holderId` の形は、Content のドメインが定める。内容の持ち主（コンテンツの公開内容・下書き、履歴の版、準備領域の変換結果）から作り、`holderId` から持ち主へ戻せる
- コンテンツの内容が持つメディアへの参照の型は、Content の `MediaReference`。`MediaUsageRepository` に渡す使用の型は、Media が定める
- メディアの保護は、Media の記録だけから決まり、確定の時点でポートが拒否する。拒否する確定と理由は、[media.md](media.md) の「MediaRepository」と「MediaUsageRepository」が定める
- 使用箇所の一覧（`listMediaUsages`）は Content のユースケース。`MediaUsageRepository` の読み取りを使い、`holderId` から、使用箇所のコンテンツと位置を求める
- 完全削除からの参照先の保護は、Content の参照の記録から決まり、[content.md](content.md) の `ContentRepository` の契約が定める。参照切れは、同じ文書の「Content」の「不変条件」と「参照の解決」が定める
- どれもポートが担保し、並行した書き込みでも成り立つ。ユースケースは、事前に記録を読んで利用者に使用箇所・参照元と解消方法を示す

### スキーマの反映

反映は、Schema のユースケースが、受け付けた要求の一回を表す `SchemaApplyJob` で進める。反映の要求、準備、確定、失敗、止まった反映の扱い、適用の事前確認の規則は、[schema.md](schema.md) の「WorkspaceSchema」「SchemaApplyJob」「Preparation」と [../usecases/schema.md](../usecases/schema.md) が定める。この節は、ドメインをまたぐ分担だけを定める。

- 適用中（[schema.md](schema.md) の「WorkspaceSchema」）に確定しない書き込みは、[common.md](common.md) の「WriteScope」と [schema.md](schema.md) の「WriteScope とスキーマ」が定める
- 準備は、変換結果を、取得から見えない準備領域へ書く。コンテンツと履歴の変換結果の準備領域は Content のポート（[content.md](content.md) の「ContentStagingArea」）が、変換結果のメディアの使用箇所の準備領域は Media のポート（「参照の記録と保護」）が持つ
- 確定は一つの UnitOfWork で行う。確定する書き込みは [../usecases/schema.md](../usecases/schema.md) の「反映を確定する（`commitSchemaApply`）」が、ほかのドメインと同じ確定で行うものは「同じ UnitOfWork で確定するもの」が定める
- 取得（公開取得、プレビュー取得、管理画面の読み取り）は、一つの取得結果に反映前後の構造を混在させない。読み方と、読み直しても揃わない取得の扱いは [content.md](content.md) の「取得と検索の規則」の「スキーマの反映との関係」が定める

### ワークスペースの作成と利用終了

区画の順は「区画をまたぐ手順」の表が、段の中身と、途中で止まったときの状態は、「手順ごとの規則」の表が挙げる定義元が定める。

- ワークスペースの作成は、ワークスペースの区画の一つの UnitOfWork で、`Workspace` と、各ドメインの「初期構成の宣言」の生成が作る集約を作る
- 所有者の指定と、所有者がいないワークスペースの扱いは、[workspace.md](workspace.md) の「AccessPolicy」が定める
- 名簿が終了中で、区画が `active` の間は、入口を通過済みの要求の書き込みと、イベントの消費者の書き込みが確定し得る。通知を作る、または送る消費者の扱いは「利用できるワークスペース」が定める。区画が `closing` と `closed` の間に確定する UnitOfWork は、[common.md](common.md) の「WriteScope」に従う。ほかの文書は、この規則を書き直さない
- 利用終了の区画の消去は、参照する側を、参照される側より先に消去する。各ドメインの手順は、ドメインのファイルの「利用終了の消去」が、呼ぶ順は「手順を呼ぶ順」が定める
- 最後の段の中身は、[workspace.md](workspace.md) の「利用終了の消去」が定める
- 利用終了の途中のワークスペースの対象が取る状態と、そこから出る遷移は、`spec/flows/` の各フローが持つ
- そのワークスペースを指すスクリプトの認証の依頼の扱いは、[token.md](token.md) の「利用終了の消去」が定める

### イベントの消費者と定期ジョブ

イベントの消費者と定期ジョブが、処理を行えない対象に当たったときの扱いは、この節だけが定める。各ドメインのユースケースとテストケースは、この規則を参照し、別の扱いを書かない。

- 次の四つの場合は、エラーにせず、その対象の処理を終える。対象の集約が既に無い。`WriteScope` の条件が `none` でない UnitOfWork で処理する対象で、区画の `Workspace` が `active` でない（`closing`、`closed`）、または区画に `Workspace` が無い。`stableSchema` の UnitOfWork で処理する対象で、読んだ `WorkspaceSchema` が適用中。UnitOfWork の確定が `ConflictError`（理由 `WORKSPACE_NOT_ACTIVE`）で拒否された
- 区画の外に実体を既に保管した処理が、上の場合に当たって終えるときの実体の扱いは、[media.md](media.md) の「実体と記録の整合」と [../usecases/media.md](../usecases/media.md) の「実体の保管」が定める
- 利用終了の途中のワークスペースの対象も処理する定期ジョブは、「定期ジョブ」の表の、対象が終了中を含む定期ジョブ
- 入口でない。消費者は、区画の `Workspace` を見る。通知を作る、または送るユースケースの扱いは「利用できるワークスペース」が定める
- ほかの失敗（上の四つに当たらない競合、保存の失敗、外部の相手の失敗）は、ユースケースがエラーを投げて終える。一件の失敗の後に残りの対象を続ける定期ジョブのユースケースだけは、下のとおり、投げずに終える。同じ配送と同じ起動の中でやり直さない。投げた後の扱いは、消費者と定期ジョブで違う
  - イベントの消費者: イベントは配送の対象に残り、再配送が続きを進める
  - 定期ジョブ: ユースケースは、一つのワークスペース（設置の区画が対象の定期ジョブでは設置）を入力に受ける。定期ジョブは、ユースケースが投げたエラーで、ほかのワークスペースの処理を止めない。失敗した対象は変わらずに残り、定期ジョブの次の起動が扱う
  - 一つのワークスペースの中に複数の対象（通知一件、コンテンツ一件など）を持つ定期ジョブのユースケースが、一件の失敗の後に残りの対象を続けるかどうかは、各ユースケースが定める。続けるユースケースは、その失敗でエラーを投げず、残りの対象を処理して終える。続けないユースケースは、エラーを投げて終える。どちらも、失敗した対象は変わらずに残り、次の起動が扱う

### 設置の状態

設置の状態（`InstallationState`）の定義、状態と設置の設定ごとに受け付けるもの、受け付けないときの結果は、[installation.md](installation.md) の「設置の状態」だけが定める。状態の間の遷移と主体は、`spec/flows/` の F-24 が持つ。利用者に見える規則は `spec/scenario/hosting.md` の「設置の状態」にある。この節は、入口と各ドメインの分担だけを定める。

- 利用者の要求の入口とワーカーの入口は、処理を始める前に Installation の「設置の状態を確かめる（`checkInstallationAvailability`）」を呼び、受け付けない結果では処理を始めない。運用コマンドの入口は、ほかのドメインの運用コマンド（管理者復旧）のユースケースを呼ぶ前に、そのコマンドを渡して呼ぶ。Installation の運用コマンドは、入口が確かめず、ユースケースが自分で確かめる。確かめ方は、[../usecases/installation.md](../usecases/installation.md) の「設置の状態の確認」が定める
- 各ドメインのユースケースは、設置の状態を知らない

### 同じ UnitOfWork で確定するもの

次の、ドメインをまたぐ書き込みは、原因の書き込みと同じ UnitOfWork で確定する。一つのドメインの中で閉じる組は、各ドメインのファイルとユースケースのトランザクション境界が定める。

- 公開内容を変える操作と、それに伴うメディアの公開、メディアの使用箇所の記録、公開の変更の記録の追記、`content.publication_changed` の記録
- スキーマの反映の確定と、メディアの使用箇所の切り替え、追加したフォームの `FormSettings` の作成または紐付け、削除したフォームの `FormSettings` の削除済みへの遷移、トークンの範囲の紐付けと、削除したモデル・フォームの範囲からの除外
- 操作履歴に残す操作と、その `AuditLogEntry` の追記

どれも、一つの区画の中の書き込み。二つの区画にまたがる書き込み（所属の索引とメンバー、名簿と `Workspace`、スクリプトの認証の依頼とトークン）は、同じ UnitOfWork で確定せず、「区画をまたぐ手順」の順に確定する。

### ドメインイベントで行うもの

イベントと消費者の定義は、この表だけが持つ。ユースケースは、ドメインの名前と識別子の組で指す（「ユースケースの所属」）。

| イベント | 発生元 | 記録するユースケース | 消費者 | 消費者が行うこと |
| --- | --- | --- | --- | --- |
| `content.publication_changed` | Content | `publishContent`、`unpublishContent`、`trashContent`、`purgeContent`、`rewriteValues`（Content）、`commitSchemaApply`（Schema） | `createChangeNotification`（Webhook） | 公開の変更の記録から、表示アプリへの通知を作る（Webhook の「ChangeNotification」） |
| `webhook.delivery_requested` | Webhook | `createChangeNotification`、`requestDeliveryRetry`（Webhook） | `sendRequestedDelivery`（Webhook） | 表示アプリへの通知の送信を試みる（Webhook の「DeliveryDispatch」） |
| `form.submission_accepted` | Form | `acceptSubmission`（Form） | `sendSubmissionNotification`（Form） | 回答の通知メールを送る |
| `form.notification_retry_requested` | Form | `retryNotifications`（Form） | `sendSubmissionNotification`（Form） | 手動再試行の通知メールを送る |
| `media.registered` | Media | `registerMedia`（Media） | `applyDerivativeDefinitions`（Media） | 現行スキーマの派生形式の定義を、その画像に適用する |
| `media.derivative_regeneration_requested` | Media | `retryDerivative`、`applyDerivativeDefinitions`、`restartStalledDerivatives`（Media） | `generateDerivative`（Media） | 一つの派生形式の、一つの世代を生成する |
| `schema.applied` | Schema | `commitSchemaApply`（Schema） | `applyDerivativeDefinitions`（Media） | 現行スキーマの派生形式の定義を、登録済みの画像に適用する |
| `schema.apply_requested` | Schema | `requestSchemaApply`、`requestDirectSchemaApply`、`requestSchemaDryRun`、`prepareSchemaApply`（Schema） | `prepareSchemaApply`、`commitSchemaApply`、`failSchemaApply`（Schema） | 反映・事前確認を進める（[../usecases/schema.md](../usecases/schema.md) の「イベントの消費者と定期ジョブ」） |

- イベントが属する区画、消費者が処理する区画、ペイロードの規則は、[common.md](common.md) の「ドメインイベント」に従う。イベントを作る振る舞いと、各イベントのペイロードは、発生元のドメインの「ドメインイベント」が定める
- 消費者が行うことの規則（UnitOfWork の分け方、再配送の受け方）は、消費者のユースケースが定める。どの消費者も、[common.md](common.md) の「ドメインイベント」の配送の規則に従う
- 隔離されたイベントの扱いは、[common.md](common.md) の「ドメインイベント」と「QuarantinedEventQuery」に従う。件数は、Installation の「状態を確認する（`checkInstallationStatus`）」が示す

### 定期ジョブ

定期ジョブの定義は、この表だけが持つ。実行者は `system`。起動の周期は実行基盤の設定で、ここに書く値を上限とする。「対象」は、定期ジョブを行う区画と、ワークスペースの区画が対象の定期ジョブでは、対象にする名簿の状態。ワークスペースごとの進め方は、「区画をまたぐ手順」の定期ジョブが定める。

| 定期ジョブ | ドメイン | ユースケース | 行うこと | 対象 | 周期 |
| --- | --- | --- | --- | --- | --- |
| 表示アプリへの通知の再試行 | Webhook | `retryDueDeliveries` | 送る時期を過ぎた送信待ちを送る（Webhook の `WebhookDelivery.isOverdue`） | 開設済み | 1 分 |
| 履歴の版の整理 | Content | `cleanUpExpiredRevisions` | 保持方針の期限を過ぎた履歴の版を削除する（Content の「ContentRetentionPolicy」） | 開設済み | 1 日 |
| ごみ箱の完全削除 | Content | `purgeExpiredTrash` | 保持方針の期限を過ぎたごみ箱のコンテンツを完全削除する（Content の「ContentRetentionPolicy」） | 開設済み | 1 日 |
| 回答の自動整理 | Form | `purgeExpiredSubmissions`、`disposeRemovedFormSettings` | 保持期限を過ぎた回答と、回答が残らない削除済みフォームの運用設定を削除する | 開設済みと終了中 | 1 日 |
| 止まった通知の検出 | Form | `detectStalledNotifications` | 止まった通知（`SubmissionNotification.stalled`）を進める | 開設済み | 5 分 |
| 編集中の記録の掃除 | Content | `cleanUpEditingPresence` | 失効した `EditingPresence` を削除する | 区画に結び付かない。ワークスペースごとに分けない | 1 時間 |
| 止まった反映の検出 | Schema | `failStalledSchemaApplies` | 止まった反映と事前確認を失敗させる | 開設済み | 5 分 |
| 止まった派生形式の生成のやり直し | Media | `restartStalledDerivatives` | 止まった派生形式の生成を、もう一度始めるか、失敗にする | 開設済み | 10 分 |
| メディアの実体の消去 | Media | `collectStorageReservations`、`purgeRemovedObjects` | 回収できる保管の予約を回収し、消去待ちの実体を消去する（Media の「StorageReservation」「ObjectRemoval」） | 開設済みと終了中 | 1 時間 |
| 所属の索引の修復 | Workspace | `repairMembershipIndex` | 所属の索引の項目を、名簿の状態と区画の `Member` に合わせる（Workspace の `MembershipRepair.decide`） | 設置の区画。区画の進め方は「区画をまたぐ手順」の「所属の索引の修復」 | 1 時間 |
| 利用終了の完了 | Installation | `completeWorkspaceClosures` | 利用終了の途中のワークスペースの利用終了を完了させる（「区画をまたぐ手順」の「利用終了の完了」） | 終了中 | 1 時間 |
| 期限切れの記録の掃除 | Identity、Workspace、Token | `purgeExpiredSessionsAndRecoveryRequests`（Identity）、`deleteExpiredDeviceAuthorizations`（Token）、`purgeExpiredInvitations`（Workspace）、`deleteExpiredTokenRecords`（Token） | 保持期間を過ぎたセッション、回復依頼、スクリプトの認証の依頼、招待、プレビューの引き換え、トークンを削除する | セッション、回復依頼、スクリプトの認証の依頼は設置の区画。招待、プレビューの引き換え、トークンは開設済みと終了中 | 1 日 |

- 期限切れの記録の掃除の対象と手順は、[common.md](common.md) の「期限つきの一度きりの値」が定める
- 定期ジョブの名前は、この表の名前が正。各ドメインの文書は、定期ジョブをこの名前で指す。定期ジョブの一回は「起動」、運用コマンドの一回は「実行」と呼ぶ
- 「止まった」の判定（閾値と回数）は、Form の `SubmissionNotification`、Media の `DerivativePolicy`、Schema の `ApplyStall` だけが定める。止まった対象の進め方は、各定期ジョブのユースケースと、そのドメインの節（Form の `SubmissionNotification`、Media の `DerivativePolicy`、Schema の `SchemaApplyJob`）が定める。表示アプリへの通知の自動の再試行の回数と間隔は、Webhook の `DeliveryPolicy` だけが定める
- バックアップの取得は、ホスティング運営者が、自分の環境で運用コマンド「バックアップを取得する」を定期的に起動して行う

## パッケージ

| パッケージ | 内容 |
| --- | --- |
| `packages/core`（`@repo/core`） | 全ドメインの domain / application / adapters |
| `apps/web`（`@repo/web`） | 管理画面、アプリ向けの HTTP API、ワーカー |
| `packages/client`（`@repo/client`） | 表示アプリとスクリプトが使う、型付きの取得・送信・コンテンツ管理のクライアントと、スキーマの記法。Schema ドメインの値と関数を `@repo/core` から使う |
| `apps/cli`（`@repo/cli`） | 開発者のコマンド（検証、型と識別子、移行定義の生成、反映、適用状況の確認、認証）と、ホスティング運営者の運用コマンド |

## 設計で決めた事項

要件が設計へ引き継いだ事項（`spec/requirements.md` の 12）と、それを決めた場所。節の名前は、各ファイルの見出し。

| 領域 | 決めた場所 |
| --- | --- |
| 実行基盤 | `AGENTS.md` の「Reference runtimes」と `docs/runtime_*.md`。spec は実行基盤を選ばず、ポートの契約までを定める。データの置き場所の分かれ方は、この文書の「区画」と [common.md](common.md) の「WriteScope」。一つの UnitOfWork が確定できる書き込みの量は [common.md](common.md) の「UnitOfWorkProvider」。利用者に見える上限と値は [../scenario/app-integration.md](../scenario/app-integration.md) の APP-12 が定め、技術設計の層では各ドメインの定数が持つ |
| スキーマと型 | [schema.md](schema.md) の「キーと名前」「Schema」「ModelDefinition」「ContentField」（テキストの書式パターン）「FormDefinition」「SchemaValidation」「SchemaFingerprint」。スキーマから型への対応の規則は [schema.md](schema.md) の「型の対応」。スキーマの記法の構成要素、型の導出、API の形式は `../usecases/app-api.md`。CMS に接続しない検証と型は [../usecases/schema.md](../usecases/schema.md) の「CMS に接続しない操作」。反映先は、トークンの発行元のワークスペース（[token.md](token.md) の「Token」） |
| 編集と検索 | [../pages/index.md](../pages/index.md) の「共通の規則」（確認と確定、未保存の入力）と「コンテンツ」。[content.md](content.md) の「EditingPresence」「ContentDiff」「管理の検索と絞り込み」。表示名称がない場合の表示は [schema.md](schema.md) の「Schema」の `displayName`。プレビュー先の経路は [schema.md](schema.md) の「PreviewPath」、許可の渡し方は [token.md](token.md) の「PreviewExchange」 |
| コンテンツの管理 | [content.md](content.md) の「Content」「Revision」「RewriteRules」「一度に扱う対象」。対象の列の規則は [common.md](common.md) の「BatchTargets」。指定できるコンテンツID の形式は [common.md](common.md) の「Clock と IdGenerator」 |
| メディア | [media.md](media.md) の「MediaFormat と MediaFile」「MediaLimits」「SharedDescription」「DerivativeState」「DerivativeGeometry」「StorageReservation」「実体と記録の整合」「MediaAccess」「MediaDelivery」「MediaDeliveryLinks」。使用箇所は [media.md](media.md) の「MediaUsageRepository」、[content.md](content.md) の「MediaReference」「参照の記録と保護」。派生形式の定義は [schema.md](schema.md) の「ImageDerivative」 |
| 取得と検索の規則 | [content.md](content.md) の「取得と検索の規則」「ContentDelivery」「Expansion」。接続方式は [token.md](token.md) の「OriginRestriction」と `../usecases/app-api.md` |
| スキーマ移行 | [schema.md](schema.md) の「Migration」「StructureMigration」「MigrationGeneration」「ValueConversion」「ValueMigration」「SchemaFingerprint」「WorkspaceSchema」「SchemaVersionRecord」「SchemaApplyJob」「Preparation」。一つの単位で確定する方式は、この文書の「スキーマの反映」と [content.md](content.md) の「ContentMigration」「ContentStagingArea」。非推奨は [schema.md](schema.md) の「ModelDefinition」「ContentField」 |
| 手元の設置と初期構成 | [installation.md](installation.md) の「初期構成の読み込み」「BootReadiness」と、この文書の「手順を呼ぶ順」、各ドメインのファイルの「初期構成の宣言」。宣言の問題の型は [common.md](common.md) の「DeclarationIssue」。キーによる紐付けは [common.md](common.md) の「キーで宣言した設定の紐付け」と、この文書の「モデルとフォームの ID」。直接反映は [schema.md](schema.md) の「WorkspaceSchema」「MigrationGeneration」。手元での起動と、保存先を空にする手順は [../scenario/schema-development.md](../scenario/schema-development.md) の SDEV-04・SDEV-06。初期構成の読み込みは [../scenario/hosting.md](../scenario/hosting.md) の HOST-02 |
| フォームと通知 | [form.md](form.md) の「TemplateText」「NotificationTemplate」「NotificationRendering」「NotificationReferenceCheck」「Submission」「DeliveryFailureReason」「トランザクション境界」。対処の状態は [common.md](common.md) の「FailureHandling」。配送と処理の再開は、この文書の「ドメインイベントで行うもの」「定期ジョブ」。隔離されたイベントの件数は [common.md](common.md) の「QuarantinedEventQuery」。失敗を知らせる経路は [../pages/index.md](../pages/index.md) の「未対処の失敗と不整合」 |
| 認証と権限 | [common.md](common.md) の「EmailAddress」。[identity.md](identity.md) の「Authentication」「MfaEnrollment」「LoginAttemptPolicy」「PasswordHasher」「TotpAuthenticator」。[common.md](common.md) の「Permission」。[workspace.md](workspace.md) の「Permission」「StandardRole」「MemberAuthorization」「MembershipIndex」「MembershipRepair」。期限と照合値の規則は [common.md](common.md) の「期限つきの一度きりの値」と「SecretTokenIssuer」。運用コマンドの入口は、この文書の「認可」、確定の時点の競合の写し方は「競合の扱い」。URL の組み立ては、この文書の「URL の組み立て」と、[workspace.md](workspace.md) の「InvitationLink」、[identity.md](identity.md) の「RecoveryLink」 |
| アプリの接続 | [token.md](token.md) の「TokenValue」「TokenGrant」「TokenScope」「OriginRestriction」「TokenLifetimePolicy」「DeviceAuthorization」「TokenAuthorization」「DeviceApprovalLink」「DeclaredTokenIndex」。利用量の制限は [common.md](common.md) の「UsageLimiter」。入口と認可の分担は、この文書の「認可」。呼び出しの形式は `../usecases/app-api.md` |
| 保持と復旧 | [content.md](content.md) の「ContentRetentionPolicy」、[form.md](form.md) の「RetentionPolicy」、[media.md](media.md) の「ObjectRemoval」、[media.md](media.md) の「MediaClosure」、[installation.md](installation.md) の「BackupPolicy」「BackupRecord」「Recovery」「BackupStore」「RecoveryJournal」「ResumeReview」「WorkspaceClosure」「StatusAssessment」「更新手順」。設置の状態は、この文書の「設置の状態」と [installation.md](installation.md) の「設置の状態」。利用終了の手順と完了、区画の間の食い違いの直し方は、この文書の「区画をまたぐ手順」「ワークスペースの作成と利用終了」 |
| 表示アプリへの通知 | [webhook.md](webhook.md) の「ChangeNotification」「WebhookDelivery」「NotificationPayload」「DeliveryPolicy」「DeliveryDispatch」「DeliveryBatch」「WebhookSender」と、宛先の集約。変更の内容は [content.md](content.md) の「PublicationChange」「PublicationLedger」と [common.md](common.md) の「PublishedChangeNotice」。公開の変更の記録、通知の元になるイベント、操作の ID は、この文書の「公開内容の整合性と通し番号」 |
| 検証 | `spec/testcases/`（ドメインごとのユースケーステスト、`common/` の共通の規則のテスト、`ports/` のポート適合テスト）。ポート適合テストの対象と条件は [common.md](common.md) の「TransactionalRepository」「SingletonRepository」「WriteScope」「UnitOfWorkProvider」 |
