# Workspace のユースケース

招待、メンバー、権限、所有者、多要素認証の必須、メンバーの認可、最後に利用したワークスペースの記録を実現する。型・振る舞い・ポートは [../domains/workspace.md](../domains/workspace.md) が定める。規則は domains が、振る舞いの結果は `spec/testcases/workspace/` のユースケーステストが定め、ユースケースは、振る舞いとポートを組み合わせる。見出しの識別子は、`spec/testcases/workspace/` のファイル名と同じ。

ほかのドメインに属するユースケースは [../domains/index.md](../domains/index.md) の「ユースケースの所属」が定める。

## 共通

- 認可: メンバーの認可の手続きと、受け付けない種類の実行者の扱いは、[../domains/index.md](../domains/index.md) の「認可」が定める。各ユースケースは、必要な権限（所有者だけの操作はその旨）と、受け付ける実行者の種類だけを書く。セッションのアカウント（`AccountId`）または招待の値を入力に取るユースケースは、`Actor` を取らない
- 利用できるワークスペース: メンバーの認可と、招待の内容の確認と受諾は、[../domains/index.md](../domains/index.md) の「利用できるワークスペース」に従う。ユースケースは、設置の区画から名簿の項目を読んで、`MemberAuthorization.resolve` と `InvitationProcess.availability` に渡す
- `WriteScope`: 書き込みを持つユースケースは、トランザクション境界に、UnitOfWork ごとの `WriteScope` を書く（[../domains/common.md](../domains/common.md) の「WriteScope」）。「ワークスペースの区画（`active`）」は `{ kind: "workspace", workspaceId, condition: { kind: "active" } }`、「ワークスペースの区画（`none`）」は条件が `none` のもの、「設置の区画」は `{ kind: "installation" }` を指す
- 区画: 集約とポートの区画は [../domains/index.md](../domains/index.md) の「集約とポートの置き場所」と「集約」の表。ワークスペースの区画の UnitOfWork の `fn` の中と、UnitOfWork を開かない読み取りで、別の区画を読むときは `PartitionReader` を使う。二つの区画を書くユースケースは、区画ごとの UnitOfWork を [../domains/index.md](../domains/index.md) の「区画をまたぐ手順」の順に確定する
- 所属の索引の修復: 修復を行うユースケース（[../domains/index.md](../domains/index.md) の「区画をまたぐ手順」）は、下の「所属の索引の修復」の手続きを使う
- ID の列での読み取り: 一度に渡せる ID の数に上限がある `findByIds`（`AccountRepository`、`MemberRepository`、`WorkspaceDirectory`）には、ID の列を、そのメソッドの上限ごとに分けて渡す。1 ページの件数と上限の大小に依らない
- 操作履歴: 残すユースケースは、[../domains/audit.md](../domains/audit.md) の「トランザクション境界」に従って追記する。実行者と対象の名称は、各ユースケースの「操作履歴」と「使用するドメインの振る舞い・ポート」が挙げるエンティティから求める。`changes` の項目は次のとおりで、どれも `ValueChange` で渡す（[../domains/audit.md](../domains/audit.md) の「AuditOperation と AuditTarget」）
  - `member.permissions_changed`: `permissions`。変更前後の、メンバーが保持する権限の組（`Permission` の列。語彙の定義順）
  - `workspace.owner_changed`: `owner`。変更前後の所有者のアカウントの名前。所有者がいないワークスペースへの指定（初期所有者の招待の受諾）は、変更前が `null`
- イベント: Workspace のユースケースは、ドメインイベントを記録しない
- 権限の組の入力: 標準の役割（`StandardRole.permissions`）、または権限の列（`PermissionSet.create`）で受け取る。出力は、権限の組と、`StandardRole.of` が返す役割
- 招待の値: 値を受けたユースケースは、[../domains/index.md](../domains/index.md) の「ワークスペースを知らない要求」と、[../domains/common.md](../domains/common.md) の「期限つきの一度きりの値」に従う。値の扱いは [../domains/workspace.md](../domains/workspace.md) の「値オブジェクト」
- 招待の URL: メールの本文に入れる招待の URL は、[../domains/index.md](../domains/index.md) の「URL の組み立て」に従い、`InvitationLink.url` で作る

### 所属の索引の修復

一つのアカウントとワークスペースの組について、所属の索引の項目を、区画の `Member` に合わせる手続き。どの項目をどう直すかは、`MembershipRepair.decide`（[../domains/workspace.md](../domains/workspace.md) の「MembershipRepair」）が決める。

- **入力**: アカウントの ID、ワークスペースの ID、そのワークスペースの名簿の状態（無ければ `null`）、読んだ項目（`MembershipIndex.find` または `scan` の結果。無ければ `null`）、区画のそのアカウントの `Member`（名簿で開設済みのワークスペースだけ。`MemberRepository.findByAccount` で読む。読み順は [../domains/workspace.md](../domains/workspace.md) の「MembershipRepair」）
- **出力**: `MembershipRepair.decide` の結果（`confirm` / `remove` / `withdraw` / `keep`）と、確定したかどうか
- **使用するドメインの振る舞い・ポート**: `Clock.now`、`MembershipRepair.decide`。結果が `confirm` なら `MembershipIndex.confirm`、`remove` なら `MembershipIndex.remove`（読んだ項目を渡す）、`withdraw` なら「MembershipRepair」の「取り下げ」
- **トランザクション境界**: `keep` は UnitOfWork を開かない。`confirm` と `remove` は、設置の区画の一つの UnitOfWork。複数の組を直すユースケースは、`confirm` の組と `remove` の項目をまとめ、一つの UnitOfWork で `MembershipIndex.confirm` と `MembershipIndex.remove` をそれぞれ一回呼ぶ。`withdraw` は「MembershipRepair」の「取り下げ」

### 所属ワークスペースを求める手続き（`resolveAccountWorkspaces`）

アカウントが所属し、利用できるワークスペースを求める手続き。ワークスペースに属さない操作が示す所属ワークスペース（[../domains/index.md](../domains/index.md) の「利用できるワークスペース」の表の行）は、どれも、この手続きで求める。「所属するワークスペースを一覧する」と、Token の「スクリプトの認証の内容を確かめる（`getDeviceAuthorization`）」が使う。

- **入力**: アカウントの ID
- **出力**: 索引の項目のワークスペースのうち、`MemberAuthorization.resolve` が `not_available` でないものの列（ワークスペースごとに、ID、名称、`MemberAccess`、多要素認証を必須にしているか。並びは [../domains/workspace.md](../domains/workspace.md) の「InitialWorkspace」に従う）と、そのアカウントが最後に利用したワークスペースの ID（無ければ無し）
- **使用するドメインの振る舞い・ポート**: `MembershipIndex.findByAccount`、`MembershipIndex.findLastUsed`、`WorkspaceDirectory.findByIds`、`AccountRepository.findById`、`Account.isMfaEnabled`（Identity）、名簿で開設済みのワークスペースごとに、その区画の `WorkspaceRepository.find`、`AccessPolicyRepository.find`、`MemberRepository.findByAccount`、`MemberAuthorization.resolve`
- **トランザクション境界**: UnitOfWork は不要。読み取りだけ

## メンバーの認可結果を求める（`resolveMemberAccess`）

管理画面の入口が、セッションのアカウントと対象のワークスペースから、そのアカウントのメンバーの認可結果を求める。入口は、結果のメンバーの ID から `Actor`（`member`）を作る。操作に必要な権限と、所有者だけの操作の判定は、各ユースケースがメンバーの認可の手続きで行う。

- **シナリオ**: AUTH-02、AUTH-05、WS-02、WS-03（「次の操作から適用する」）と、管理画面の全シナリオの入口
- **認可**: セッションのアカウント
- **入力**: セッションのアカウント、ワークスペースの ID
- **出力**: `MemberAccess`（`granted` / `mfa_setup_required`）
- **使用するドメインの振る舞い・ポート**: `WorkspaceDirectory.findById`（名簿の状態）、`AccountRepository.findById`、`Account.isMfaEnabled`（Identity）。ここまでは設置の区画。`WorkspaceRepository.find`、`AccessPolicyRepository.find`、`MemberRepository.findByAccount`（ワークスペースの区画）、`MemberAuthorization.resolve`
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

## 所属するワークスペースを一覧する（`listMyWorkspaces`）

アカウントが利用できるワークスペースと、ログインの後に開くワークスペースを返す。

- **シナリオ**: AUTH-02、AUTH-03（多要素認証の解除の前に示す、必須にしたワークスペース）、AUTH-05
- **認可**: セッションのアカウント
- **入力**: セッションのアカウント
- **出力**: 「共通」の「所属ワークスペースを求める手続き」の列と、ログインの後に開くワークスペースの ID（`InitialWorkspace.choose` の結果）
- **使用するドメインの振る舞い・ポート**: 「所属ワークスペースを求める手続き」、`InitialWorkspace.choose`（`lastUsed` は、手続きが返した、最後に利用したワークスペース）
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

## 最後に利用したワークスペースを記録する（`recordLastWorkspace`）

メンバーが開いたワークスペースを、次のログインで開くワークスペースとして、所属の索引に記録する。

- **シナリオ**: AUTH-02、AUTH-05
- **認可**: 実行者は `member` だけ。権限は不要
- **入力**: `Actor`、ワークスペースの ID
- **出力**: なし
- **使用するドメインの振る舞い・ポート**: メンバーの認可の手続き（[../domains/index.md](../domains/index.md) の「認可」）、`MembershipIndex.findLastUsed`、`MembershipIndex.markLastUsed`。組は、認可の手続きで読んだメンバーのアカウントの ID と、ワークスペースの ID
- **トランザクション境界**: 設置の区画の一つの UnitOfWork（`membershipIndex.markLastUsed`）。認可の手続きの読み取りと `MembershipIndex.findLastUsed` は、UnitOfWork の前に `PartitionReader` で行う。`findLastUsed` が既にそのワークスペースを返した場合は、UnitOfWork を開かない

## メンバーを招待する（`inviteMember`）

宛先と権限の組を決めて招待を作り、招待の URL を宛先へ送る。

- **シナリオ**: WS-01、AUTH-01
- **認可**: `member`。`member:manage`
- **操作履歴**: `member.invited`
- **入力**: `Actor`、ワークスペースの ID、宛先のメールアドレス、権限の組
- **出力**: 作った招待（有効期限を含む）、`InvitationIssueResult` の種類（`issued` / `superseded` と `replaced`）、送信の結果（`SystemMailResult` の `accepted` / `not_configured` / `failed`）。`not_configured` のときは招待の値
- **使用するドメインの振る舞い・ポート**: `EmailAddress.create`、`PermissionSet.create`、`StandardRole.permissions`、`AccountRepository.findByEmail`（Identity。`PartitionReader` で読む）、`MemberRepository.findByAccount`、`InvitationRepository.findPendingMemberByEmail`、`SecretTokenIssuer.issueForWorkspace`、`InvitationTokenDigest.create`、`IdGenerator.generate`、`InvitationProcess.inviteMember`、`InvitationRepository.save`・`insert`、`AuditLogRepository.append`、設置の公開の URL（`InstallationEnvironment`）、`InvitationLink.url`、`SystemMailer.send`（本文のワークスペースの名称は、認可の手続きで読んだ名簿の項目の名称）
- **トランザクション境界**: ワークスペースの区画（`active`）の一つの UnitOfWork（認可の読み取り、新しい招待の `insert`、`superseded` の場合は前の招待の `save`、`auditLogRepository.append`）。`SystemMailer.send` は、UnitOfWork の確定の後に呼ぶ

## 招待を取り消す（`revokeInvitation`）

未使用の招待を失効させる。

- **シナリオ**: WS-01
- **認可**: `member`。`member:manage`
- **操作履歴**: `member.invitation_revoked`
- **入力**: `Actor`、ワークスペースの ID、招待の ID
- **出力**: なし
- **使用するドメインの振る舞い・ポート**: `InvitationRepository.findById`、`Invitation.isManagedByMembers`、`Invitation.revoke`、`InvitationRepository.save`、`AuditLogRepository.append`
- **トランザクション境界**: ワークスペースの区画（`active`）の一つの UnitOfWork（認可の読み取り、`invitationRepository` の読み取りと `save`、`auditLogRepository.append`）

## 未使用の招待を一覧する（`listPendingInvitations`）

ワークスペースの、メンバーによる `pending` の招待を返す。

- **シナリオ**: WS-01
- **認可**: `member`。`member:manage`
- **入力**: `Actor`、ワークスペースの ID、ページの指定
- **出力**: `InvitationRepository.findPendingPage` の招待の列と総件数。招待ごとに、`Invitation.standing`（`usable` / `expired`）と、`StandardRole.of` の役割
- **使用するドメインの振る舞い・ポート**: `Pagination.create`、`InvitationRepository.findPendingPage`、`Invitation.standing`、`StandardRole.of`
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

## 招待の内容と状態を確かめる（`inspectInvitation`）

招待の値を示した人に、参加するワークスペース、付与される内容、招待を受諾できるかを返す。

- **シナリオ**: AUTH-01、HOST-04
- **認可**: 招待の値。セッションは任意
- **入力**: 招待の値、セッションのアカウント（ログインしている場合）
- **出力**: `InvitationInspection`（`acceptable` / `email_mismatch` / `expired` / `accepted` / `revoked` / `invalid`）。`acceptable` の場合は、加えて、宛先のアカウントが既にあるか
- **使用するドメインの振る舞い・ポート**: `SecretTokenIssuer.workspaceOf`、`SecretTokenIssuer.digest`、`InvitationTokenDigest.create`、`WorkspaceDirectory.findById`（名簿の項目。ワークスペースの名称を持つ）、取り出したワークスペースの区画の `InvitationRepository.findByTokenDigest`、`WorkspaceRepository.find`、`AccessPolicyRepository.find`、ログインしている場合は、`AccountRepository.findById`（Identity。セッションのアカウントのメールアドレス）と、同じ区画の `MemberRepository.findByAccount`、`Clock.now`、`Invitation.lapsedAt`、`OneTimeValueRetention.isForgotten`、`InvitationProcess.inspect`（`viewer` は、セッションのアカウントのメールアドレスと、そのアカウントの区画の `Member`。ログインしていなければ `null`）、`StandardRole.of`、`AccountRepository.findByEmail`（Identity。`acceptable` の場合だけ）
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

## 招待を受諾する（`acceptInvitation`）

招待と引き換えに、招待された人をワークスペースのメンバーにする。アカウントを持たない人は、同時にアカウントを作り、セッションを始める。初期所有者の招待では、受諾した人を所有者に指定する。

- **シナリオ**: AUTH-01、HOST-04
- **認可**: 招待の値。アカウントを持つ人は、加えてセッションのアカウント
- **操作履歴**: `member.joined`、`workspace.owner_changed`。どの記録も、実行者と対象は、参加したメンバー、または所有者に指定したメンバー
- **入力**: 招待の値と、次のどちらか
  - アカウントを持たない人: 名前、パスワード
  - アカウントを持つ人: セッションのアカウント
- **出力**: 参加したワークスペースの ID、メンバーの ID、`MemberAccess`（`granted` / `mfa_setup_required`）。アカウントを作った場合は、加えて、結果の種類 `started`（セッションの値と有効期限）/ ログインが要る（`SessionIssuance.start` の `login_required` の場合は `retryAt` を添える）
- **使用するドメインの振る舞い・ポート**:
  - 事前の確認: `SecretTokenIssuer.workspaceOf`、`SecretTokenIssuer.digest`、`InvitationTokenDigest.create`、`WorkspaceDirectory.findById`（名簿の状態）、取り出したワークスペースの区画の `InvitationRepository.findByTokenDigest`、`WorkspaceRepository.find`、`AccessPolicyRepository.find`、`MemberRepository.findByAccount`（セッションのアカウントの `Member`。`viewer.member` に渡す）、`JoiningGuardRepository.findById`（そのアカウントの参加の囲い）、`JoiningGuard.ticket`、`AccountRepository.findByEmail`（宛先のアカウントの有無）、`AccountRepository.findById`（セッションのアカウント）、`Clock.now`、`Invitation.lapsedAt`、`OneTimeValueRetention.isForgotten`、`InvitationProcess.availability`。`viewer` は、セッションのアカウントがあれば、そのメールアドレスと `Member`。無ければ `null`
  - 設置の区画（前段）: `MembershipIndex.beginJoining`（`since` は、ユースケースの初めに求めた現在時刻）。アカウントを作る場合は、加えて、`AccountName.create`、`PlainPassword.create`、`PasswordHasher.hash`、`IdGenerator.generate`、`Account.create`、`AccountRepository.insert`（Identity）
  - ワークスペースの区画（後段）: `InvitationRepository.findByTokenDigest`、`WorkspaceRepository.find`、`AccessPolicyRepository.find`、`MemberRepository.findByAccount`（`InvitationProcess.accept` の `member` に渡す）、`JoiningGuardRepository.findById`、`IdGenerator.generate`、`Clock.now`、`InvitationProcess.accept`（`acceptor` は、アカウントを作った受諾なら `new_account`、セッションのアカウントの受諾なら `existing_account`。`joining` は、事前の確認で読んだ囲いの世代）、`MemberRepository.insert`、`InvitationRepository.save`、`AccessPolicyRepository.save`、`JoiningGuardRepository.insert` または `save`、`AuditLogRepository.append`・`appendMany`
  - 設置の区画（確かめ直し）: `MembershipIndex.confirm`。アカウントを作った場合は、加えて、`LoginAttemptPolicy.key`・`limit`、`UsageLimiter.check`、`SecretTokenIssuer.issue`、`SessionTokenDigest.create`、`IdGenerator.generate`、`SessionIssuance.start`、`SessionRepository.insert`（`started` の場合）（Identity）
  - 競合の後: `JoiningGuardRepository.findById`、`MemberRepository.findByAccount`、`InvitationProcess.rejectionOnConflict`（`acceptor` は後段と同じ）
  - 出力: `Account.isMfaEnabled`、`MemberAuthorization.resolve`
- **トランザクション境界**: 区画ごとの三つの UnitOfWork を、この順に確定する（[../domains/index.md](../domains/index.md) の「区画をまたぐ手順」の参加）。事前の確認の読み取りは、UnitOfWork の前に行う
  - 前段: 設置の区画の一つの UnitOfWork（`membershipIndex.beginJoining`。アカウントを作る場合は `accountRepository.insert`）
  - 後段: ワークスペースの区画（`active`）の一つの UnitOfWork（招待、`Workspace`、方針、そのアカウントの `Member` と参加の囲いの読み直し、`InvitationProcess.accept` の結果に応じた `memberRepository.insert`・`accessPolicyRepository.save`、`invitationRepository.save`、`joiningGuardRepository` の `insert` または `save`、`auditLogRepository.append` または `appendMany`）
  - 確かめ直し: 設置の区画の一つの UnitOfWork（`membershipIndex.confirm`。`started` の場合は `sessionRepository.insert`）。`UsageLimiter.check` は、この UnitOfWork の前に、宛先のメールアドレスで呼ぶ

## メンバーを一覧する（`listMembers`）

ワークスペースのメンバーを、権限の組、所有者かどうか、多要素認証の設定の有無とともに返す。

- **シナリオ**: WS-02、WS-03、WS-04、WS-05
- **認可**: `member`。`member:manage`
- **入力**: `Actor`、ワークスペースの ID、ページの指定
- **出力**: メンバーの列と総件数、ワークスペースが多要素認証を必須にしているか、ワークスペースに所有者がいるか。メンバーごとに、権限の組（`MemberGrant.permissions`）、役割（`StandardRole.of`）、所有者かどうか、多要素認証を設定しているか。権限の変更（WS-02）と所有者の変更（WS-04）の確認は、対象のメンバーが通知設定の権限（`notification:manage`）を新たに持つかどうかを、この権限の組から求める
- **使用するドメインの振る舞い・ポート**: `Pagination.create`、`MemberRepository.findPage`、`AccountRepository.findByIds`、`Account.isMfaEnabled`（Identity。設置の区画）、`AccessPolicy.isVacant`、`MemberAuthorization.grantOf`（方針は、認可の手続きで読んだもの）、`MemberGrant.permissions`、`MemberGrant.isOwner`、`StandardRole.of`
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

## メンバーの権限を変更する（`changeMemberPermissions`）

メンバーが保持する権限の組を置き換える。

- **シナリオ**: WS-02
- **認可**: `member`。`member:manage`
- **操作履歴**: `member.permissions_changed`
- **入力**: `Actor`、ワークスペースの ID、対象のメンバーの ID、権限の組
- **出力**: 変更後のメンバー（権限の組と役割）
- **使用するドメインの振る舞い・ポート**: `MemberRepository.findById`、`PermissionSet.create`、`StandardRole.permissions`、`MemberAdministration.changePermissions`、`MemberRepository.save`、`AccountRepository.findById`（Identity。対象の名称）、`AuditLogRepository.append`
- **トランザクション境界**: ワークスペースの区画（`active`）の一つの UnitOfWork（認可の読み取り、`memberRepository` の読み取りと `save`、`auditLogRepository.append`）

## メンバーの利用の解除の影響を確かめる（`previewMemberRemoval`）

解除の確認に示す、対象のメンバーが発行したアプリに紐付くトークンを返す。

- **シナリオ**: WS-03（手順 2）
- **認可**: `member`。`member:manage`
- **入力**: `Actor`、ワークスペースの ID、対象のメンバーの ID
- **出力**: 対象のメンバーが発行した、アプリに紐付くトークンのうち、`Token.standing` が `active` のものの名称（`Token.label`）の列
- **使用するドメインの振る舞い・ポート**: `MemberRepository.findById`、`TokenRepository.findPage`（Token。`TokenListFilter` の `relatedMember` と、`origins` に `manual` と `bootstrap`。該当が尽きるまでページを進める。返る `ListedToken` の `token` を使う）、`Clock.now`、`Token.standing`、`Token.label`（Token）
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

## メンバーの利用を解除する（`removeMember`）

メンバーをワークスペースから外す。

- **シナリオ**: WS-03
- **認可**: `member`。`member:manage`
- **操作履歴**: `member.removed`
- **入力**: `Actor`、ワークスペースの ID、対象のメンバーの ID
- **出力**: なし
- **使用するドメインの振る舞い・ポート**: `MemberRepository.findById`、`MemberAdministration.assertRemovable`、`MemberRepository.delete`、`AccountRepository.findById`（Identity。対象の名称）、`AuditLogRepository.append`。後段は、`WorkspaceDirectory.findById`、`MembershipIndex.find`、`MemberRepository.findByAccount`、「共通」の「所属の索引の修復」
- **トランザクション境界**: 二つの段を、この順に行う（[../domains/index.md](../domains/index.md) の「区画をまたぐ手順」のメンバーの利用の解除）
  - 前段: ワークスペースの区画（`active`）の一つの UnitOfWork（認可の読み取り、`memberRepository` の読み取りと `delete`、`auditLogRepository.append`）
  - 後段: 前段の確定の後に、対象のメンバーのアカウントとワークスペースの組に、「共通」の「所属の索引の修復」を行う。UnitOfWork の分け方は、その手続きのとおり

## 所有者を変更する（`transferOwnership`）

ワークスペースの所有者を、別のメンバーへ移す。

- **シナリオ**: WS-04
- **認可**: `member`。所有者だけ
- **操作履歴**: `workspace.owner_changed`。対象は新しい所有者のメンバー
- **入力**: `Actor`、ワークスペースの ID、新しい所有者にするメンバーの ID
- **出力**: なし
- **使用するドメインの振る舞い・ポート**: `AccessPolicy.isOwned`（認可の手続きで所有者と確かめた方針の型を絞る）、`MemberRepository.findById`（新しい所有者）、`AccountRepository.findByIds`（Identity。名称）、`MemberAdministration.transferOwnership`（`formerOwner` は、認可の手続きで読んだ実行者の `Member`）、`AccessPolicyRepository.save`、`MemberRepository.save`（元の所有者）、`AuditLogRepository.append`
- **トランザクション境界**: ワークスペースの区画（`active`）の一つの UnitOfWork（認可の読み取り、`memberRepository` の読み取り、`accessPolicyRepository.save`、元の所有者の `memberRepository.save`、`auditLogRepository.append`）

## 多要素認証を設定していないメンバーを確かめる（`listMembersWithoutMfa`）

多要素認証を必須にする確認に示す、多要素認証を設定していないメンバーを返す。

- **シナリオ**: WS-05（手順 2）
- **認可**: `member`。所有者だけ
- **入力**: `Actor`、ワークスペースの ID
- **出力**: `Account.isMfaEnabled` が偽のアカウントのメンバーの列
- **使用するドメインの振る舞い・ポート**: `MemberRepository.findPage`（最後のページまで進める）、`AccountRepository.findByIds`、`Account.isMfaEnabled`（Identity）
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

## 多要素認証を必須にする（`requireMfa`）

ワークスペースの利用に、多要素認証の設定を必須にする。

- **シナリオ**: WS-05
- **認可**: `member`。所有者だけ
- **入力**: `Actor`、ワークスペースの ID
- **出力**: なし
- **使用するドメインの振る舞い・ポート**: `AccessPolicy.isOwned`（認可の手続きで所有者と確かめた方針の型を絞る）、`AccessPolicy.requireMfa`（`ownerMfaEnabled` は、認可の手続きで読んだ所有者のアカウントの `Account.isMfaEnabled`）、`AccessPolicyRepository.save`
- **トランザクション境界**: ワークスペースの区画（`active`）の一つの UnitOfWork（認可の読み取りと `accessPolicyRepository.save`）

## 多要素認証の必須を解除する（`liftMfaRequirement`）

多要素認証を設定していないメンバーも、ワークスペースを利用できるようにする。

- **シナリオ**: WS-05
- **認可**: `member`。所有者だけ
- **入力**: `Actor`、ワークスペースの ID
- **出力**: なし
- **使用するドメインの振る舞い・ポート**: `AccessPolicy.liftMfaRequirement`、`AccessPolicyRepository.save`
- **トランザクション境界**: ワークスペースの区画（`active`）の一つの UnitOfWork（認可の読み取りと `accessPolicyRepository.save`）

## 期限切れの招待を掃除する（`purgeExpiredInvitations`）

定期ジョブ「期限切れの記録の掃除」のうち、招待の分。一つのワークスペースの区画を対象にする。対象にするワークスペースと、一つのワークスペースの失敗の扱いは、[../domains/index.md](../domains/index.md) の「定期ジョブ」と「イベントの消費者と定期ジョブ」が定める。

- **シナリオ**: なし（[../domains/index.md](../domains/index.md) の「定期ジョブ」）
- **認可**: 実行者は `system` だけ
- **入力**: `Actor`、ワークスペースの ID
- **出力**: 削除した招待の件数
- **使用するドメインの振る舞い・ポート**: `Clock.now`、`OneTimeValueRetention.cutoff`、`InvitationRepository.deleteLapsed`
- **トランザクション境界**: [../domains/common.md](../domains/common.md) の「期限つきの一度きりの値」の「掃除」。対象のポートは `InvitationRepository.deleteLapsed`

## 所属の索引を修復する（`repairMembershipIndex`）

定期ジョブ「所属の索引の修復」。所属の索引のすべての項目を走査し、項目ごとに、名簿の状態と、そのワークスペースの区画の `Member` に合わせる。

- **シナリオ**: なし（[../domains/index.md](../domains/index.md) の「定期ジョブ」）。AUTH-01 の「取り下げられた参加」が、この定期ジョブに依る
- **認可**: 実行者は `system` だけ
- **入力**: `Actor` だけ
- **出力**: `joined` にした項目、外した項目、確定しなかった項目の件数
- **使用するドメインの振る舞い・ポート**: `MembershipIndex.scan`、`WorkspaceDirectory.findByIds`（ページの項目のワークスペース）、名簿で開設済みのワークスペースの項目ごとに、その区画の `MemberRepository.findByAccount`、「共通」の「所属の索引の修復」。走査するのは項目だけで、項目が無い `Member` を探さない（[../domains/workspace.md](../domains/workspace.md) の「`Member` と項目の対応」）
- **トランザクション境界**: `MembershipIndex.scan` を、`next` が `null` になるまで、ページごとに読む。`confirm` の組と `remove` の項目は、ページごとに、設置の区画の一つの UnitOfWork（書き込みは 2 回以下）。`withdraw` は「共通」の「所属の索引の修復」
