# Token のユースケース

トークンの発行・範囲に指定できる対象・一覧・失効、ブラウザ用トークンの利用元の限定と利用量の制限の変更、示されたトークンの確認、トークンの要求の操作ごとの認可、スクリプトの認証、プレビューを開く操作と引き換え、プレビュー先の表示アプリの URL の設定を定める。型・振る舞い・ポートは [../domains/token.md](../domains/token.md) が定める。

見出しの括弧の中は、テストケース（`spec/testcases/token/`）のファイル名と同じ識別子である。

## 共通

- 認可は [../domains/index.md](../domains/index.md) の「認可」に従う。各ユースケースは、必要な権限と、受け付ける実行者の種類だけを書く
- 利用できるワークスペースと、値からワークスペースを求める入口の読み方は、[../domains/index.md](../domains/index.md) の「利用できるワークスペース」と「ワークスペースを知らない要求」に従う
- 変更の無い変更は、[../domains/index.md](../domains/index.md) の「変更の無い変更」に従う。当たる振る舞いは [../domains/token.md](../domains/token.md) の「Token」が定める
- 確定の時点の `ConflictError` は、[../domains/index.md](../domains/index.md) の「競合の扱い」に従う
- `WriteScope`: 書き込みを持つユースケースは、トランザクション境界に、UnitOfWork ごとの `WriteScope` を書く（[../domains/common.md](../domains/common.md) の「WriteScope」）。「ワークスペースの区画（`active`）」「ワークスペースの区画（`schemaVersion`）」は、その条件の `workspace` の `WriteScope`、「設置の区画」は `{ kind: "installation" }` を指す
- 保持期間を過ぎた集約は、[../domains/common.md](../domains/common.md) の「期限つきの一度きりの値」に従う。ID でトークンを指すユースケースは、`Token.isAddressable`（[../domains/token.md](../domains/token.md) の「Token」）で、操作の対象にできるトークンかを判定する
- 操作履歴: 操作の書き込みと同じ UnitOfWork で追記する（[../domains/audit.md](../domains/audit.md) の「トランザクション境界」）。トークンを指す値（対象と実行者）は `Token.designation`、関係するメンバーのアカウントは `Token.relatedAccountId` で求める
- トークンの値の生成: `SecretTokenIssuer.issueForWorkspace`（発行元のワークスペースの ID を渡す）、`TokenValue.compose`、`SecretTokenIssuer.digest`、`IdGenerator.generate`。値と照合値の規則は [../domains/token.md](../domains/token.md) の「TokenValue」が定める

## トークンを手動発行する（`issueToken`）

名称、種類、許可、範囲、利用元の限定、利用量の制限、有効期限を決めて、アプリに紐付くトークンを発行する。

- **シナリオ**: WS-06
- **認可**: アプリ連携の権限（`integration:manage`）。実行者は `member` だけ
- **操作履歴**: `token.issued`

### 入出力

- **入力**: `AppTokenInput` の内容。範囲は、ワークスペース全体、またはモデル・フォームのキーの集合。キーは、「トークンの範囲に指定できる対象を一覧する」が返したもの。有効期限は、期限なし、期限、どちらも指定していない、のいずれか
- **出力**: 発行したトークンの内容と、トークンの値

### 使用するドメインの振る舞い・ポート

- `Clock.now`、`Token.assertIssuable`（値の生成と保護の前に呼ぶ。発行者の権限の組は、認可の手続きの結果）
- 範囲が集合の発行では、`WorkspaceSchemaRepository.find`（`run` の前に読む）、`TokenScope.resolveKeys`（入力のキーを、読んだスキーマの `modelIds` / `formIds` で ID にする）
- トークンの値の生成。ブラウザ用は `SecretCipher.protect`、`ProtectedTokenValue.wrap`
- `Token.issueForApp`（`origin: "manual"`。集合の範囲は、`resolveKeys` が返した ID）、`TokenRepository.insert`
- 操作履歴の追記

### トランザクション境界

- **UnitOfWork の要否**: 必要
- `WriteScope`: 範囲が集合の発行は、ワークスペースの区画（`schemaVersion`）。スキーマの読み方と条件は [../domains/schema.md](../domains/schema.md) の「WriteScope とスキーマ」に従う。範囲がワークスペース全体の発行は、ワークスペースの区画（`active`）で、スキーマを読まない
- UnitOfWork: 認可の読み取り、`Token` の `insert`、`AuditLogEntry` の追記
- 使うリポジトリ: `workspaceRepository`、`accessPolicyRepository`、`memberRepository`、`tokenRepository`、`auditLogRepository`。アカウントは `PartitionReader` で読む

## トークンの範囲に指定できる対象を一覧する（`listScopeTargets`）

手動発行の範囲に指定できる、現行スキーマのモデルとフォームと、実行者が与えられない許可を返す。

- **シナリオ**: WS-06
- **認可**: アプリ連携の権限。実行者は `member` だけ

### 入出力

- **入力**: 無し（`Actor` と `WorkspaceId` だけ）
- **出力**: 現行スキーマのモデルの列と、フォームの列。要素は、キーと表示名称。与えられない許可の列（`GrantRefusal`）

### 使用するドメインの振る舞い・ポート

- `WorkspaceSchemaRepository.find`、Schema の `Schema.displayName`
- `GrantPolicy.refusals`（すべての許可と、認可の手続きの結果の実行者の権限の組を渡す）

### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みが無い

## トークンを一覧する（`listTokens`）

ワークスペースのトークンを、見直しの条件で絞り込んで一覧する。

- **シナリオ**: WS-07
- **認可**: アプリ連携の権限。実行者は `member` だけ

### 入出力

- **入力**: 絞り込み（有効期限が近い、長く使われていない、関係するメンバーが解除された）と、ページング
- **出力**: トークンの列と総件数。トークンごとに、最終利用日時、見直しの区別（期限切れ、範囲が空、有効期限が近い、長く使われていない、関係するメンバーが解除された）、関係するメンバーを持つ。載るトークンは、[../domains/token.md](../domains/token.md) の「一覧に載るトークン」が定める

### 使用するドメインの振る舞い・ポート

- `Clock.now`、`TokenListFilter`（`expiringSoon`、`longUnused`、`relatedMemberRemoved`）、`TokenRepository.findPage`（利用者が指定したページで一回）
- `TokenReviewPolicy.isRelatedMemberRemoved`、`TokenReviewPolicy.isLongUnused`（トークンごとの区別。`findPage` がトークンに添えた `relatedMemberPresent` と `lastUsedAt` を渡す）
- `TokenReviewPolicy.isExpiringSoon`、`Token.standing`、`Token.grants`、`TokenScope.isEmpty`
- `Token.relatedAccountId`、`AccountRepository.findByIds`（関係するメンバーの名前）
- `WorkspaceSchemaRepository.find`（範囲の要素の ID から、モデル・フォームのキーを求める）

### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みが無い

## 紐付けを待つトークンの範囲を一覧する（`listPendingTokenScopes`）

初期構成で宣言したトークンのうち、範囲に紐付け待ちの要素を持つものを一覧する。

- **シナリオ**: HOST-03
- **認可**: アプリ連携の権限。実行者は `member` だけ

### 入出力

- **入力**: ページング
- **出力**: トークンの名称と、紐付けを待つモデル・フォームのキー

### 使用するドメインの振る舞い・ポート

- `Clock.now`、`TokenRepository.findPage`（`TokenListFilter` の `pendingScopeOnly: true`。返る `ListedToken` の `token` を使う）、`TokenScope.pendingTargets`

### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みが無い

## ブラウザ用トークンの値を確かめる（`revealBrowserTokenValue`）

発行済みのブラウザ用トークンの値を示す。

- **シナリオ**: WS-06、WS-07
- **認可**: アプリ連携の権限。実行者は `member` だけ

### 入出力

- **入力**: トークンの ID
- **出力**: トークンの値、または失効済みであること

### 使用するドメインの振る舞い・ポート

- `Clock.now`、`TokenRepository.findById`、`Token.isAddressable`、`Token.browserValue`
- `SecretCipher.reveal`（`browserValue` が `available` のとき）

### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みが無い

## ブラウザ用トークンの利用元の限定と利用量の制限を変える（`changeBrowserTokenRestrictions`）

発行済みのブラウザ用トークンの利用元の限定と利用量の制限を、入力した値に置き換える。トークンの値は変わらない。

- **シナリオ**: WS-07
- **認可**: アプリ連携の権限。実行者は `member` だけ
- **操作履歴**: `token.restrictions_changed`

### 入出力

- **入力**: トークンの ID、利用元の限定（利用元の列。空の列は限定しない）、利用量の制限（無しを含む）
- **出力**: 変更後のトークンの内容、または失効済みであること

### 使用するドメインの振る舞い・ポート

- `Clock.now`、`TokenRepository.findById`、`Token.isAddressable`、`Token.changeRestrictions`、`TokenRepository.save`
- 操作履歴の追記。`token.restrictions_changed` の `changes` は、`changeRestrictions` の変更前後の値から作る三つの項目。`origins` は利用元の列（`OriginRestriction` の並びの順。限定しない場合は空の列）、`usageLimitMax` と `usageLimitWindowMs` は利用量の制限の上限と期間（ミリ秒。制限が無い場合は `null`）

### トランザクション境界

- **UnitOfWork の要否**: 必要
- `WriteScope`: ワークスペースの区画（`active`）
- UnitOfWork: 認可の読み取り、`Token` の読み取りと `save`、`AuditLogEntry` の追記
- 使うリポジトリ: `workspaceRepository`、`accessPolicyRepository`、`memberRepository`、`tokenRepository`、`auditLogRepository`。アカウントは `PartitionReader` で読む

## トークンを失効させる（`revokeToken`）

一覧から選んだトークンを失効させる。アプリに紐付くトークンとメンバーに紐付くトークンの両方を対象にする。

- **シナリオ**: WS-07
- **認可**: アプリ連携の権限。実行者は `member` だけ
- **操作履歴**: `token.revoked`

### 入出力

- **入力**: トークンの ID
- **出力**: 失効したトークンの内容、または失効済みであること

### 使用するドメインの振る舞い・ポート

- `Clock.now`、`TokenRepository.findById`、`Token.isAddressable`、`Token.revoke`、`TokenRepository.save`
- 操作履歴の追記

### トランザクション境界

- **UnitOfWork の要否**: 必要
- `WriteScope`: ワークスペースの区画（`active`）
- UnitOfWork: 認可の読み取り、`Token` の読み取りと `save`、`AuditLogEntry` の追記
- 使うリポジトリ: `workspaceRepository`、`accessPolicyRepository`、`memberRepository`、`tokenRepository`、`auditLogRepository`。アカウントは `PartitionReader` で読む

## 自分に紐付くトークンを失効させる（`revokeOwnToken`）

認証を解除するコマンドが、実行環境に保管したトークンを示して、そのトークンを失効させる。

- **シナリオ**: AUTH-06
- **認可**: 実行者は `memberToken` だけ。入口が「示されたトークンを確かめる」で作った `Actor` を渡す。失効させられるトークンと、許可と紐付くメンバーの扱いは、[../domains/token.md](../domains/token.md) の「自分に紐付くトークンの失効」に従う
- **操作履歴**: `token.revoked`

### 入出力

- **入力**: `Actor`、`WorkspaceId`（入口が「示されたトークンを確かめる」から得たもの）
- **出力**: 無し

### 使用するドメインの振る舞い・ポート

- `TokenRepository.findById`（`actor.tokenId`）
- `Clock.now`、`Token.revoke`、`TokenRepository.save`
- 操作履歴の追記（実行者は `Actor` の `memberToken`。紐付くメンバーの名前は、`Token.relatedAccountId` を `AccountRepository.findById` に渡して読む）
- 名簿を読まない。入口が確かめた後に名簿が終了中になった要求は、[../domains/index.md](../domains/index.md) の「利用できるワークスペース」と「競合の扱い」に従う

### トランザクション境界

- **UnitOfWork の要否**: 必要
- `WriteScope`: ワークスペースの区画（`active`）
- UnitOfWork: `Token` の読み取りと `save`、`AuditLogEntry` の追記
- 使うリポジトリ: `tokenRepository`、`auditLogRepository`。アカウントは `PartitionReader` で読む

## 示されたトークンを確かめる（`authenticateToken`）

アプリ向けの要求の入口が、要求が示したトークンの値、利用元、利用量を確かめ、`Actor` を求める。操作ごとの認可は「トークンの要求を操作ごとに認可する」が行う。

- **シナリオ**: APP-01
- **認可**: トークンの値。入口だけが呼ぶ

### 入出力

- **入力**: トークンの値（示されなければ無し）、要求が示した利用元（示されなければ `null`）。対象のワークスペースは、トークンの値から求める
- **出力**: `Actor`（アプリに紐付くトークンは `appToken`、メンバーに紐付くトークンは `memberToken`）と、発行元の `WorkspaceId`。入口は、この二つをユースケースへ渡す

### 使用するドメインの振る舞い・ポート

- `TokenValue.secretPart`、`SecretTokenIssuer.workspaceOf`（値が含む発行元のワークスペースの ID）、`SecretTokenIssuer.digest`
- `DeclaredTokenIndex.findByDigest`（初期構成で宣言したトークンの発行元のワークスペースの ID）
- 求めたワークスペースごとに、`WorkspaceDirectory.findById`、`WorkspaceRepository.find`、`WorkspaceAvailability.of`、利用できるワークスペースの区画の `TokenRepository.findByValueDigest`
- `TokenAuthentication.identify`（確かめるトークンを決める規則は、この振る舞いだけが持つ）。二つの発行元（`embedded`、`declared`）に渡す値の意味は、[../domains/token.md](../domains/token.md) の「TokenAuthentication」が定める。`embedded` からトークンを得た値は、宣言したトークンの索引を引かず、`declared` に `found: false` を渡す。保持期間を過ぎたトークンは、`null` として渡す
- `Clock.now`、`TokenAuthentication.verify`
- `TokenUsagePolicy.key`、`UsageLimiter.check`、`UsageLimiter.record`（`verified` で `usageLimit` が `null` でない要求）
- `Token.shouldRecordUse`（`findByValueDigest` が添えた `lastUsedAt` を渡す）、`TokenRepository.recordUse`（`PartitionReader` から得たリポジトリで呼ぶ）
- 読み取りは、名簿の項目と宣言したトークンの索引を `PartitionReader.installation()` から、`Workspace` とトークンを `PartitionReader.workspace(workspaceId)` のリポジトリから行う

### トランザクション境界

- **UnitOfWork の要否**: 不要。UnitOfWork で確定する書き込みが無い。最終利用日時（`TokenRepository.recordUse`）と利用量（`UsageLimiter.record`）は、UnitOfWork の外の記録で書く

## トークンの要求を操作ごとに認可する（`authorizeTokenOperation`）

トークンの認可の手続き。呼ぶユースケースと時点は [../domains/index.md](../domains/index.md) の「認可」が定める。入口は呼ばない。

- **シナリオ**: APP-01、AUTH-06
- **認可**: 実行者は `appToken` と `memberToken`。この手続きが認可を行う

### 入出力

- **入力**: `Actor`、`WorkspaceId`、操作が必要とする `AccessRequirement`（許可、対象、権限）
- **出力**: `GrantedTokenAccess`。操作履歴と履歴の版に実行者を残すユースケースは、トークンを読み直さず、トークンを指す値に `designation` を使い、関係するメンバーの名前を、`relatedAccountId` を `AccountRepository.findById` に渡して読む

手続きは、二つの段からなる。前の段は `TokenAuthorization.resolve`、後の段は `TokenAuthorization.authorize`。段の間の対象の解決と、`AccessRequirement.target` に渡すものは、[../domains/token.md](../domains/token.md) の「TokenAuthorization」が定める。

### 使用するドメインの振る舞い・ポート

- `TokenRepository.findById`（`actor.tokenId`）
- `Clock.now`、`WorkspaceDirectory.findById`（名簿の項目。設置の区画から `PartitionReader` で読む）、`WorkspaceRepository.find`、`WorkspaceAvailability.of`。トークンの種類によらず読み、求めた真偽を `TokenAuthorization.resolve` に渡す
- `memberToken` では、`AccessPolicyRepository.find`、`MemberRepository.findById`（`token.member.memberId`）、`AccountRepository.findById`、`Account.isMfaEnabled`、`MemberAuthorization.resolve`。`appToken` では、発行者の `Member` を読まない
- `TokenAuthorization.resolve`、`TokenAuthorization.authorize`

### トランザクション境界

- **UnitOfWork の要否**: この手続きは UnitOfWork を開かず、何も書かない。行う場所は [../domains/index.md](../domains/index.md) の「認可」が定める
- 読み方は [../domains/index.md](../domains/index.md) の「認可」のメンバーの認可の手続きの手順 1 に従う。書き込みを持つユースケースの UnitOfWork の中では、`Token` も文脈のリポジトリから読む
- トークンは、入力の `WorkspaceId` の区画で読む

## スクリプトの認証を始める（`startDeviceAuthorization`）

スクリプトの実行環境のコマンドが、必要な許可を指定して、スクリプトの認証の依頼を作る。依頼の時点ではワークスペースを指定しない。

- **シナリオ**: AUTH-06
- **認可**: 不要。実行者を持たない（セッションもトークンも示さない）

### 入出力

- **入力**: 要求する許可
- **出力**: 承認の値、確認用のコード（`UserCode.format`）、受け取りの値、依頼の期限。承認用の URL の組み立ては、[../domains/index.md](../domains/index.md) の「URL の組み立て」に従う

### 使用するドメインの振る舞い・ポート

- `SecretTokenIssuer.issue`（承認の値、受け取りの値、確認用のコードの元になる値。ワークスペースの ID を含まない）、`UserCode.derive`
- `IdGenerator.generate`、`Clock.now`、`DeviceAuthorization.start`
- `DeviceAuthorizationRepository.insert`

### トランザクション境界

- **UnitOfWork の要否**: 必要
- `WriteScope`: 設置の区画。ワークスペースが決まっていない
- UnitOfWork: `DeviceAuthorization` の `insert`
- 使うリポジトリ: `deviceAuthorizationRepository`

## スクリプトの認証の内容を確かめる（`getDeviceAuthorization`）

承認用の URL を開いた人に、依頼の内容と、その人が所属するワークスペースのそれぞれを対象に選べるかどうかを示す。

- **シナリオ**: AUTH-06
- **認可**: ワークスペースに属さない操作。セッションで確かめたアカウント。権限を必要としない

### 入出力

- **入力**: `AccountId`、承認の値
- **出力**: 依頼の内容（確認用のコード、要求された許可、発行するトークンの有効期限）と状態（`DeviceAuthorization.standing`。拒否の依頼は `denial`）、所属ワークスペースの列（ワークスペースごとに、ID、名称、`DeviceApprovalEligibility`）、承認の時点のワークスペース（ID と名称）

所属ワークスペースの列は、Workspace の、所属ワークスペースを求める手続き（`resolveAccountWorkspaces`。[workspace.md](workspace.md) の「共通」）の結果で、含むワークスペースと順は、その手続きが定める。

### 使用するドメインの振る舞い・ポート

- `SecretTokenIssuer.digest`、`DeviceAuthorizationRepository.findByApprovalDigest`
- `Clock.now`、`DeviceAuthorization.standing`、`DeviceAuthorization.tokenExpiresAt`、`UserCode.format`
- Workspace の、所属ワークスペースを求める手続き（`resolveAccountWorkspaces`。ワークスペースごとの ID、名称、`MemberAccess`）。読むポートと、項目から列を求める規則は、その手続きが定める
- `DeviceAuthorizationProcess.eligibility`（列のワークスペースごとに、依頼の `grants` と、そのワークスペースの `MemberAccess` を渡す）

### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みが無い。`resolveAccountWorkspaces` は、読み取りだけの手続き

## スクリプトの認証を承認する（`approveDeviceAuthorization`）

メンバーが、確認用のコードの一致を確かめた依頼を、選んだワークスペースを対象にして承認する。承認の結果（承認、または権限の不足による拒否）は `DeviceAuthorization.approve` が決める。トークンは、コマンドが結果を受け取るときに発行する。

- **シナリオ**: AUTH-06
- **認可**: 権限を必要としない。実行者は `member` だけ。対象のワークスペースは、メンバーが、`getDeviceAuthorization` が選べると示したワークスペースから選んだもの

### 入出力

- **入力**: 承認の値
- **出力**: 承認した依頼の内容（ワークスペース、許可、トークンの有効期限）、または権限の不足による拒否（`insufficient_permissions`）と不足する権限

### 使用するドメインの振る舞い・ポート

- メンバーの認可の手続き（UnitOfWork の前に、選んだワークスペースの区画の `WorkspaceRepository.find`、`AccessPolicyRepository.find`、`MemberRepository.findById` と、設置の区画の `AccountRepository.findById` を読む）
- `SecretTokenIssuer.digest`、`DeviceAuthorizationRepository.findByApprovalDigest`
- `Clock.now`、`DeviceAuthorization.approve`（認可の手続きの `granted` のメンバーの、ワークスペース、`MemberId`、`AccountId`、権限の組（`MemberGrant.permissions(grant)`）を渡す）
- `DeviceAuthorizationRepository.save`

### トランザクション境界

- **UnitOfWork の要否**: 必要
- `WriteScope`: 設置の区画。書き込みは、依頼の `save` だけ。ワークスペースの区画を書かない
- 認可は、UnitOfWork の前に行う（[../domains/index.md](../domains/index.md) の「認可」）。`approve` に渡す権限の組は、この認可が読んだもの
- UnitOfWork: 依頼の読み取りと `save`
- 使うリポジトリ: `deviceAuthorizationRepository`

## スクリプトの認証を承認しない（`denyDeviceAuthorization`）

承認用の URL を開いた人が、依頼を承認しないことを選ぶ。依頼は、理由が `denied_by_account` の拒否になる。

- **シナリオ**: AUTH-06
- **認可**: ワークスペースに属さない操作。セッションで確かめたアカウント。権限を必要としない

### 入出力

- **入力**: `AccountId`、承認の値
- **出力**: 無し

### 使用するドメインの振る舞い・ポート

- `SecretTokenIssuer.digest`、`DeviceAuthorizationRepository.findByApprovalDigest`
- `Clock.now`、`DeviceAuthorization.deny`、`DeviceAuthorizationRepository.save`

### トランザクション境界

- **UnitOfWork の要否**: 必要
- `WriteScope`: 設置の区画
- UnitOfWork: 依頼の読み取りと `save`
- 使うリポジトリ: `deviceAuthorizationRepository`

## スクリプトの認証の結果を受け取る（`redeemDeviceAuthorization`）

コマンドが、受け取りの値を示して、承認の結果を求める。承認済みの依頼では、依頼を受け取り済みにしてから、承認したメンバーに紐付くサーバー用トークンを発行して返す。

- **シナリオ**: AUTH-06
- **認可**: 受け取りの値。実行者を持たない（セッションもトークンも示さない）
- **操作履歴**: `token.issued`（トークンを発行したとき。実行者は承認したメンバー）

### 入出力

- **入力**: 受け取りの値
- **出力**: 結果。承認待ち、発行済み（トークンの値、有効期限、対象のワークスペース）、受け取り済み、期限切れ、拒否（`denial` または `refusal`）のいずれか。依頼に適用する振る舞いの結果は [../domains/token.md](../domains/token.md) の「DeviceAuthorizationProcess」が定める

### 使用するドメインの振る舞い・ポート

- `SecretTokenIssuer.digest`、`DeviceAuthorizationRepository.findByDeviceCodeDigest`
- `Clock.now`、`DeviceAuthorizationProcess.result`（`approved` でない依頼の結果）
- `approved` の依頼では、名簿の項目（`approval.workspaceId`）、その区画の `WorkspaceRepository.find`、`AccessPolicyRepository.find`、`MemberRepository.findById`（`approval.member.memberId`）、`AccountRepository.findById`、`Account.isMfaEnabled`、`WorkspaceAvailability.of`、`MemberAuthorization.resolve`、トークンの値の生成（種類は `server`。`approval.workspaceId` を含む値）、`DeviceAuthorizationProcess.redeem`
- `outcome` が `redeemable` のとき、`DeviceAuthorizationRepository.save`（設置の区画）。続けて、`DeviceAuthorizationProcess.refusal`（確かめ直し）、`TokenRepository.insert`、操作履歴の追記（ワークスペースの区画）

### トランザクション境界

- **UnitOfWork の要否**: `redeem` が `redeemable` を返した受け取りだけが必要。ほかの受け取りは、UnitOfWork を開かない
- UnitOfWork の前: 受け取りの値の照合値で依頼を読む。`approved` の依頼は、名簿の項目を読み、名簿で開設済みなら `approval.workspaceId` の区画の `Workspace`、`AccessPolicy`、`Member`、設置の区画のアカウントを読んで、`DeviceMemberStanding` を求める。トークンの値を生成し、`redeem` を適用する
- 前段。設置の区画。UnitOfWork は、依頼の読み直しと、`redeemed` の依頼の `save`。読み直した依頼に、同じ入力で `redeem` を適用し、読み直した `expectedVersion` で保存する
- 後段。ワークスペースの区画（`active`。`approval.workspaceId`）。前段の `save` が確定した受け取りだけが行う。UnitOfWork は、名簿の項目、`Workspace`、`AccessPolicy`、`Member`、アカウントの読み直しと、`DeviceAuthorizationProcess.refusal` による確かめ直し、`Token` の `insert`、`AuditLogEntry` の追記。名簿の項目とアカウントは `PartitionReader` で、`Workspace`、`AccessPolicy`、`Member` は文脈のリポジトリで読む
- 区画の順は、[../domains/index.md](../domains/index.md) の「区画をまたぐ手順」が定める
- 使うリポジトリ: 設置の区画の `deviceAuthorizationRepository`、ワークスペースの区画の `workspaceRepository`、`accessPolicyRepository`、`memberRepository`、`tokenRepository`、`auditLogRepository`。アカウントと名簿の項目は `PartitionReader` で読む

## プレビューを開く（`openPreview`）

メンバーが開いたコンテンツのプレビュー先を決め、表示アプリへ渡す引き換えの値を作る。

- **シナリオ**: EDIT-12
- **認可**: 権限を必要としない。実行者は `member` だけ

### 入出力

- **入力**: コンテンツID
- **出力**: 次のいずれか
  - 開ける: プレビュー先の URL（引き換えの値を含む）
  - 開けない: `PreviewDestination` の `resolved: false` の理由。`draft_has_type_errors` では、Content の結果からエラー箇所を添える

### 使用するドメインの振る舞い・ポート

- `ContentRepository.findById`、`Content.latest`、`Content.hasTypeErrors`、`UniqueValues.of`（経路に使う一意な値。Content）
- `WorkspaceSchemaRepository.find`（コンテンツの `ModelId` から、モデルの定義を求める）、Schema の `PreviewPath.resolveFor`（`PreviewPathResolution`）
- `PreviewSettingsRepository.find`、`PreviewSettings.baseUrl`
- `PreviewDestination.resolve`
- `resolved: true` のとき、`SecretTokenIssuer.issueForWorkspace`（引き換えの値と照合値。値はワークスペースの ID を含む）、`PreviewDestination.url`、`IdGenerator.generate`、`Clock.now`、`PreviewExchange.create`、`PreviewExchangeRepository.insert`

### トランザクション境界

- **UnitOfWork の要否**: 必要
- `WriteScope`: ワークスペースの区画（`active`）
- UnitOfWork: 認可、`Content`、`WorkspaceSchema`、`PreviewSettings` の読み取りと、`PreviewExchange` の `insert`
- 使うリポジトリ: `workspaceRepository`、`accessPolicyRepository`、`memberRepository`、`contentRepository`、`workspaceSchemaRepository`、`previewSettingsRepository`、`previewExchangeRepository`。アカウントは `PartitionReader` で読む

## 引き換えの値をトークンに引き換える（`redeemPreviewExchange`）

表示アプリのサーバー側の処理が、引き換えの値と引き換えに、プレビューを開いたメンバーに紐付くトークンを得る。

- **シナリオ**: APP-06、EDIT-12
- **認可**: 引き換えの値。実行者を持たない（ほかのトークンを示さない）
- **操作履歴**: `token.issued`（実行者はプレビューを開いたメンバー）

### 入出力

- **入力**: 引き換えの値
- **出力**: トークンの値と有効期限。`redeem` の結果の種類は、[../domains/token.md](../domains/token.md) の「PreviewExchangeProcess」が定める

### 使用するドメインの振る舞い・ポート

- `SecretTokenIssuer.workspaceOf`（引き換えの値からワークスペースの ID を取り出す）、`SecretTokenIssuer.digest`
- 名簿の項目、`WorkspaceRepository.find`、`WorkspaceAvailability.of`
- 利用できるワークスペースでは、その区画の `PreviewExchangeRepository.findByCodeDigest`、`AccessPolicyRepository.find`、`MemberRepository.findById`（`exchange.member.memberId`）、`AccountRepository.findById`、`Account.isMfaEnabled`、`MemberAuthorization.resolve`、トークンの値の生成（種類は `server`）
- `Clock.now`、`PreviewExchangeProcess.redeem`
- `outcome` が `redeemable` のとき、`TokenRepository.insert`、`PreviewExchangeRepository.save`、操作履歴の追記

### トランザクション境界

- **UnitOfWork の要否**: 名簿で開設済みのワークスペースの引き換えだけが必要
- UnitOfWork の前: 引き換えの値からワークスペースの ID を取り出し、名簿の項目を読む
- `WriteScope`: ワークスペースの区画（`active`）。`WorkspaceId` は、引き換えの値から取り出したもの
- UnitOfWork: `Workspace`、引き換え、`AccessPolicy`、`Member` の読み取り、`Token` の `insert`、引き換えの `save`、`AuditLogEntry` の追記
- 使うリポジトリ: `previewExchangeRepository`、`workspaceRepository`、`accessPolicyRepository`、`memberRepository`、`tokenRepository`、`auditLogRepository`。アカウントと名簿の項目は `PartitionReader` で読む

## プレビュー先の表示アプリの URL を確かめる（`getPreviewBaseUrl`）

ワークスペースの現在の URL を示す。

- **シナリオ**: WS-08
- **認可**: アプリ連携の権限。実行者は `member` だけ

### 入出力

- **入力**: 無し（`Actor` と `WorkspaceId` だけ）
- **出力**: 現在の URL、または未設定であること

### 使用するドメインの振る舞い・ポート

- `PreviewSettingsRepository.find`、`PreviewSettings.baseUrl`

### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みが無い

## プレビュー先の表示アプリの URL を設定する（`setPreviewBaseUrl`）

ワークスペースの URL を、入力した URL にする。

- **シナリオ**: WS-08
- **認可**: アプリ連携の権限。実行者は `member` だけ
- **操作履歴**: `preview.url_changed`

### 入出力

- **入力**: 表示アプリの URL
- **出力**: 保存した URL

### 使用するドメインの振る舞い・ポート

- `Clock.now`、`PreviewSettingsRepository.find`、`PreviewSettings.setUrl`、`PreviewSettingsRepository.save`
- 操作履歴の追記。`preview.url_changed` の `changes` は、項目 `baseUrl`（変更前後の URL。未設定は `null`）

### トランザクション境界

- **UnitOfWork の要否**: 必要
- `WriteScope`: ワークスペースの区画（`active`）
- UnitOfWork: 認可の読み取り、`PreviewSettings` の読み取りと `save`、`AuditLogEntry` の追記
- 使うリポジトリ: `workspaceRepository`、`accessPolicyRepository`、`memberRepository`、`previewSettingsRepository`、`auditLogRepository`。アカウントは `PartitionReader` で読む

## プレビュー先の表示アプリの URL を取り除く（`clearPreviewBaseUrl`）

ワークスペースの URL を未設定に戻す。

- **シナリオ**: WS-08
- **認可**: アプリ連携の権限。実行者は `member` だけ
- **操作履歴**: `preview.url_changed`

### 入出力

- **入力**: 無し（`Actor` と `WorkspaceId` だけ）
- **出力**: 無し

### 使用するドメインの振る舞い・ポート

- `Clock.now`、`PreviewSettingsRepository.find`、`PreviewSettings.clearUrl`、`PreviewSettingsRepository.save`
- 操作履歴の追記。`changes` は「プレビュー先の表示アプリの URL を設定する」と同じ項目

### トランザクション境界

- **UnitOfWork の要否**: 必要
- `WriteScope`: ワークスペースの区画（`active`）
- UnitOfWork: 認可の読み取り、`PreviewSettings` の読み取りと `save`、`AuditLogEntry` の追記
- 使うリポジトリ: `workspaceRepository`、`accessPolicyRepository`、`memberRepository`、`previewSettingsRepository`、`auditLogRepository`。アカウントは `PartitionReader` で読む

## 期限切れのプレビューの引き換え・トークンを削除する（`deleteExpiredTokenRecords`）

定期ジョブ「期限切れの記録の掃除」のうち、ワークスペースの区画の Token の分。一つのワークスペースの、保持期間の起点（`lapsedAt`）から保持期間を過ぎた `PreviewExchange` と `Token` を削除する。対象は [../domains/common.md](../domains/common.md) の「期限つきの一度きりの値」が、呼び出す対象のワークスペースは [../domains/index.md](../domains/index.md) の「定期ジョブ」が定める。

- **シナリオ**: EDIT-12（引き換えの値の有効期間）、WS-07（失効・期限切れのトークン）
- **認可**: 実行者は `system` だけ。周期は [../domains/index.md](../domains/index.md) の「定期ジョブ」が定める

### 入出力

- **入力**: `Actor`、ワークスペースの ID
- **出力**: 削除した引き換えとトークンの件数

### 使用するドメインの振る舞い・ポート

- `PreviewExchangeRepository.deleteLapsed`
- `TokenRepository.deleteLapsed`

### トランザクション境界

- 進め方（`before` の求め方、UnitOfWork の分け方、`WriteScope`）は、[../domains/common.md](../domains/common.md) の「期限つきの一度きりの値」の「掃除」が定める
- 使うリポジトリ: `previewExchangeRepository`、`tokenRepository`

## 期限切れのスクリプトの認証の依頼を削除する（`deleteExpiredDeviceAuthorizations`）

定期ジョブ「期限切れの記録の掃除」のうち、設置の区画の Token の分。保持期間の起点（`lapsedAt`）から保持期間を過ぎた `DeviceAuthorization` を削除する。対象は [../domains/common.md](../domains/common.md) の「期限つきの一度きりの値」が定める。ワークスペースごとに分けない。

- **シナリオ**: AUTH-06（依頼の有効期間）
- **認可**: 実行者は `system` だけ。周期は [../domains/index.md](../domains/index.md) の「定期ジョブ」が定める

### 入出力

- **入力**: `Actor` だけ
- **出力**: 削除した依頼の件数

### 使用するドメインの振る舞い・ポート

- `DeviceAuthorizationRepository.deleteLapsed`

### トランザクション境界

- 進め方は、[../domains/common.md](../domains/common.md) の「期限つきの一度きりの値」の「掃除」が定める
- 使うリポジトリ: `deviceAuthorizationRepository`
