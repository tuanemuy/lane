# Workspace

ワークスペースごとに、誰が何を行えるかを保つ。境界と依存方向は [index.md](index.md) の「依存方向」が、集約とポートの区画は「集約とポートの置き場所」が定める。アカウントは `AccountId` で参照する。

Workspace はドメインイベントを発行しない。エンティティの振る舞いは、次の状態のエンティティだけを返す。

## ユビキタス言語

| 英語名 | 日本語名 | 定義 |
| --- | --- | --- |
| Workspace | ワークスペース | スキーマ、コンテンツ、メディア、フォーム、メンバー、アクセス権を管理する単位。ID と名称は名簿の項目が持ち、データはワークスペースの区画が持つ。区画の集約 `Workspace` は、区画での、そのワークスペースの状態（`active`、`closing`、`closed`）を持つ |
| Member | メンバー | アカウントの、一つのワークスペースへの所属。権限の組を持つ |
| Permission | 権限 | メンバーに付与する、操作の許可の単位。ワークスペース単位で付与する |
| Permission set | 権限の組 | メンバーが持つ権限の集合。空の組は閲覧だけを行える |
| Standard role | 標準の役割 | 権限の組み合わせに付けた名前（管理者、編集者、閲覧者）。メンバーは役割を保持せず、権限の組だけを保持する |
| Owner | 所有者 | ワークスペースが指定する一人のメンバー。すべての権限を持ち、所有者の変更と多要素認証の必須化を行える。役割ではない |
| MFA requirement | 多要素認証の必須 | 多要素認証を設定したメンバーだけがワークスペースを利用できる設定 |
| Invitation | 招待 | 宛先のメールアドレス、参加するワークスペース、付与する内容、有効期限を持つ、参加の許可 |
| Initial owner invitation | 初期所有者の招待 | ホスティング運営者が発行する招待。権限の組を持たず、参加した人を所有者に指定する |
| Invitation token | 招待の値 | 招待の URL に含める一度きりの値。CMS は照合値だけを保存する |
| Invitation URL | 招待の URL | 設置の公開の URL と招待の値から作る URL。招待された人が開いて参加する |
| Removal | 利用の解除 | メンバーをワークスペースから外すこと |
| Member access | メンバーの認可結果 | あるアカウントがあるワークスペースで、その時点で行える操作の範囲 |
| Retiring | 利用終了の途中 | 名簿で終了中（`retiring`）のワークスペースの状態。ホスティング運営者が利用終了を実行してから、名簿から外すまで続く |
| Closing workspace | 区画が `closing` | 区画の `Workspace` の状態。利用終了の開始から、区画の消去を終えるまで |
| Closed workspace | 区画が `closed` | 区画の `Workspace` の状態。利用終了の区画の消去を終えた区画に残る、終端の状態 |
| Access policy | アクセスの方針 | ワークスペースの所有者の指定と、多要素認証の必須 |
| Joining guard | 参加の囲い | アカウントごとの、そのワークスペースへの参加の世代。所属の索引の修復が、進行中の参加を取り下げるために進める |
| Workspace directory | ワークスペースの名簿 | 設置のワークスペースごとの、ID、名称、名簿の状態の記録。設置の区画にある。設置をまたぐ処理が対象のワークスペースを求めるために使う |
| Directory status | 名簿の状態 | 準備中（`provisioning`）、開設済み（`open`）、終了中（`retiring`） |
| Membership index | 所属の索引 | アカウントとワークスペースの組ごとの記録。設置の区画にある。項目は、参加の途中（`joining`）、参加済み（`joined`）のいずれかと、項目の世代を持つ。所属の正は `Member` が持つ。索引は、ワークスペースに属さない操作が、アカウントの所属するワークスペースを求めるために使う |
| Membership | 所属の索引の項目 | 所属の索引の一件 |
| Membership generation | 項目の世代 | 所属の索引の項目を見分ける値。項目を置く書き込みごとに新しい値になる。参加の囲いの世代（`JoiningGuard.generation`）と別のもの |

## エンティティ

### Workspace

ワークスペースの、区画での状態。区画が `active` か、`closing` か、`closed` かだけを持つ。

フィールド。

```ts
type WorkspaceBase = Readonly<{
  version: number;
  createdAt: Date;
  updatedAt: Date;
}>;

type ActiveWorkspace = WorkspaceBase & Readonly<{ status: "active" }>;
type ClosingWorkspace = WorkspaceBase & Readonly<{ status: "closing" }>;
type ClosedWorkspace = WorkspaceBase & Readonly<{ status: "closed" }>;
type Workspace = ActiveWorkspace | ClosingWorkspace | ClosedWorkspace;
type WorkspaceStatus = Workspace["status"];
```

| フィールド | 制約 |
| --- | --- |
| `version` | 生成時は 0。状態を変える振る舞いごとに 1 増える。現在と同じ値への変更は [index.md](index.md) の「変更の無い変更」 |

`closed` は、利用終了の区画の消去を終えた区画に残る終端の状態（「利用終了の消去」）。

`Workspace` は、区画に一つの集約で、自分の ID を持たない。ワークスペースの ID は、区画が決め、要る場所には、ユースケースが、区画を求めた ID を渡す。名称は「WorkspaceDirectoryEntry」が持つ。所有者の指定と多要素認証の必須は、`AccessPolicy` が持つ。

振る舞い。

```ts
const Workspace: {
  create(now: Date): ActiveWorkspace;
  beginClosing(workspace: Workspace | null, now: Date): ClosingWorkspace | ClosedWorkspace;
  close(workspace: ClosingWorkspace | ClosedWorkspace, now: Date): ClosedWorkspace;
};
```

| 振る舞い | 処理 |
| --- | --- |
| `create` | `active` のワークスペースを返す |
| `beginClosing` | `active` のワークスペースを `closing` にする。`null`（区画に `Workspace` が無い）には、`version` が 0 で、`createdAt` と `updatedAt` が `now` の、`closing` のワークスペースを返す。区画を作る前に止まった作成の利用終了が当たる。`closing` と `closed` のワークスペースは、渡されたワークスペースをそのまま返す。利用終了の続きが、これを通る。入力された名称の確認は、名簿の項目の `WorkspaceDirectoryEntry.confirmName` が行う |
| `close` | `closing` のワークスペースを `closed` にする。`closed` のワークスペースは、渡されたワークスペースをそのまま返す。利用終了の区画の消去の最後だけが使う（「利用終了の消去」） |

不変条件。

- `closing` のワークスペースは `active` に戻らない
- `closed` は終端。`closed` のワークスペースから出る遷移を持つ振る舞いは、どれも、渡されたワークスペースをそのまま返す。`closed` の `Workspace` がある区画に、`Workspace` を作り直す書き込みは確定しない（`WorkspaceRepository.insert`）
- 区画の `Workspace` は、最初の書き込み（作成の `insert`、または `Workspace` が無い区画への利用終了の開始の `insert`）の後、区画に残り続ける
- `active` の `Workspace` がある区画は、`AccessPolicy` を持つ

ライフサイクル。

- 生成: ホスティング運営者が運用コマンドで作成したとき、または初期構成の読み込みで宣言されたとき。どちらも、名簿に準備中の項目を登録した後に、その ID のワークスペースの区画に、`AccessPolicy` と同じ UnitOfWork で作る（「初期構成の宣言」）
- 生成（`closing`）: 利用終了の開始が、`Workspace` が無い区画に、`beginClosing(null, now)` の結果を作る（「利用終了の消去」）
- 遷移: `active` →（`beginClosing`）→ `closing`。`closing` →（`beginClosing`）→ `closing`（変わらない）
- 遷移: `closing` →（`close`）→ `closed`（「利用終了の消去」）
- 終端: `closed`。区画に残る

#### 利用中の条件

[common.md](common.md) の「WriteScope」の条件が見る `Workspace` の状態。条件の結果と、条件の割り当ては、そこが定める。

- `active` の `Workspace` は、条件 `active` を満たす。`closing` と `closed` の `Workspace` は満たさない。`AccessPolicy`（所有者の指定と設定）は、条件に関わらない
- `Workspace` を `active` でなくする遷移は、`beginClosing` だけ

### AccessPolicy

ワークスペースのアクセスの方針。所有者の指定と、多要素認証の必須を持つ。区画の `Workspace` が `active` の間、区画に一つある。

フィールド。

```ts
type OwnerDesignation =
  | Readonly<{ status: "designated"; memberId: MemberId }>
  | Readonly<{ status: "vacant" }>;

type AccessPolicy = Readonly<{
  owner: OwnerDesignation;
  mfaRequired: boolean;
  version: number;
  createdAt: Date;
  updatedAt: Date;
}>;
type OwnedPolicy = AccessPolicy & Readonly<{ owner: Extract<OwnerDesignation, { status: "designated" }> }>;
type VacantPolicy = AccessPolicy & Readonly<{ owner: Extract<OwnerDesignation, { status: "vacant" }> }>;
```

| フィールド | 制約 |
| --- | --- |
| `owner` | 所有者の指定。`designated` は、区画のメンバーを指す。`vacant` は、所有者がいない。生成時は `vacant` |
| `mfaRequired` | 生成時は `false`。所有者の指定と独立で、所有者の指定を変える振る舞いは、この値を変えない |
| `version` | 生成時は 0。状態を変える振る舞いごとに 1 増える。現在と同じ値への変更は [index.md](index.md) の「変更の無い変更」 |

所有者がいるワークスペースの方針は `OwnedPolicy`、所有者がいない（`vacant`）ワークスペースの方針は `VacantPolicy`。所有者がいない状態は、所有者の指定が `vacant` であることの一つだけ。初期所有者が未参加のワークスペースと、復旧で所有者を失ったワークスペースが当たる。二つは同じ状態で、認可、招待、状態確認で同じ扱いを受ける。

`AccessPolicy` は、区画に一つの集約で、ID を持たない。

振る舞い。

```ts
const AccessPolicy: {
  create(now: Date): VacantPolicy;
  designateOwner(policy: VacantPolicy, owner: Member, now: Date): OwnedPolicy;
  changeOwner(policy: OwnedPolicy, newOwner: Member, now: Date): OwnedPolicy;
  clearOwner(policy: AccessPolicy, now: Date): VacantPolicy;
  isVacant(policy: AccessPolicy): policy is VacantPolicy;
  isOwned(policy: AccessPolicy): policy is OwnedPolicy;
  isOwner(policy: AccessPolicy, memberId: MemberId): boolean;
  requireMfa(policy: OwnedPolicy, ownerMfaEnabled: boolean, now: Date): OwnedPolicy;
  liftMfaRequirement(policy: AccessPolicy, now: Date): AccessPolicy;
};
```

| 振る舞い | 処理 |
| --- | --- |
| `create` | `mfaRequired` が `false` で、所有者の指定が `vacant` の方針を返す |
| `designateOwner` | 所有者の指定を、`owner.id` の `designated` にする。`mfaRequired` は変えない。所有者がいる方針は、引数に取れない。呼ぶ側が `isVacant` で絞る |
| `changeOwner` | `newOwner.id` が現在の所有者なら `AlreadyOwner`。所有者の指定を置き換える |
| `clearOwner` | 所有者の指定を `vacant` にする。`mfaRequired` は変えない。既に `vacant` なら、渡された方針をそのまま返す。「利用再開前の処理」だけが使う |
| `isVacant` | 所有者の指定が `vacant` なら真 |
| `isOwned` | 所有者の指定が `designated` なら真。所有者だけの操作のユースケースが、認可の手続きの後に型を絞るために使う |
| `isOwner` | 所有者の指定が `designated` で、その `memberId` が `memberId` と一致すれば真 |
| `requireMfa` | 所有者がいる方針だけを受け取る。`ownerMfaEnabled`（所有者のアカウントが多要素認証を設定しているか）が偽なら `BusinessRuleError`（`OwnerMfaNotConfigured`）。真なら `mfaRequired` を `true` にする。既に必須なら、所有者の設定を確かめずに、渡された方針をそのまま返す |
| `liftMfaRequirement` | `mfaRequired` を `false` にする。必須でなければ、渡された方針をそのまま返す |

不変条件。

- 所有者は、最大一人。所有者の指定が `designated` なら、その区画のメンバーを指す
- メンバーの操作では、所有者の指定は `vacant` にならない（所有者は解除できず、所有者の変更は別のメンバーを指定する）
- 所有者の指定が `vacant` のワークスペースでも、メンバーの認可、トークン、配信、受付、通知は続く。所有者だけの操作（`OWNER_ONLY`）は、所有者がいないので、誰も行えない

ライフサイクル。

- 生成: 区画の `Workspace` の生成と同じ UnitOfWork（「初期構成の宣言」）。運用コマンドで作成したワークスペースは、`create` の結果を保存し、初期所有者の招待の受諾を待つ。初期構成で宣言したワークスペースは、宣言した所有者を同じ UnitOfWork で指定し、所有者の指定が `designated` で保存する
- 遷移: `vacant` →（`designateOwner`）→ `designated`。初期所有者の招待の受諾と、初期構成の読み込みが行う。どちらも `MemberAdministration.designateInitialOwner` を通る
- 遷移: `designated` →（`changeOwner`）→ `designated`。所有者の変更
- 遷移: `designated` →（`clearOwner`）→ `vacant`。「利用再開前の処理」だけが行う
- 削除: 利用終了の区画の消去（「利用終了の消去」）

### Member

フィールド。

```ts
type Member = Readonly<{
  id: MemberId;
  accountId: AccountId;
  permissions: PermissionSet;
  version: number;
  createdAt: Date;
  updatedAt: Date;
}>;
```

| フィールド | 制約 |
| --- | --- |
| `accountId` | 必須。生成後に変わらない。区画の中で一意 |
| `permissions` | 必須。メンバーが保持する権限の組。空の組を取れる |
| `version` | 生成時は 0。状態を変える振る舞いごとに 1 増える |

メンバーは、区画の集約で、ワークスペースの ID を持たない。所属するワークスペースは、区画が決める。メンバーは、在籍している間だけ存在する。在籍以外の状態を持たないので、状態の判別子を持たない。メンバーは、自分が所有者かどうかを持たない。所有者の指定は `AccessPolicy` だけが持ち、所有者の権限は `MemberAuthorization.grantOf` が導く。

振る舞い。

```ts
const Member: {
  create(
    params: {
      id: string;
      accountId: AccountId;
      permissions: PermissionSet;
    },
    now: Date,
  ): Member;
  changePermissions(member: Member, permissions: PermissionSet, now: Date): Member;
};
```

| 振る舞い | 処理 |
| --- | --- |
| `create` | `MemberId` を生成し、渡された権限の組を持つメンバーを返す |
| `changePermissions` | 権限の組を置き換える。同じ組への変更は [index.md](index.md) の「変更の無い変更」 |

不変条件。

- 一つのアカウントは、一つのワークスペースに最大一人のメンバーとして所属する
- 権限はワークスペース単位で持つ。メンバーの権限は、そのワークスペースのすべてのモデル、フォーム、コンテンツ、項目に掛かる

ライフサイクル。

- 生成: 招待の受諾、または初期構成の読み込み。どちらも、設置の区画でアカウントを確定し、所属の索引に `joining` の項目を置いた後に、ワークスペースの区画で作る（「`Member` と項目の対応」）
- 変更: 権限の変更。所有者の変更では、元の所有者に管理者の権限の組を書く
- 削除: 利用の解除（ユースケースが `MemberAdministration.assertRemovable` の後にリポジトリの `delete` を呼ぶ）、ワークスペースの利用終了（「利用終了の消去」）、バックアップからの復旧の後のアカウントが無いメンバーの削除（「利用再開前の処理」）

### Invitation

フィールド。

```ts
type InvitationGrant =
  | Readonly<{ kind: "member"; permissions: PermissionSet }>
  | Readonly<{ kind: "initial_owner" }>;

type InvitationBase = Readonly<{
  id: InvitationId;
  email: EmailAddress;
  grant: InvitationGrant;
  tokenDigest: InvitationTokenDigest;
  version: number;
  createdAt: Date;
  expiresAt: Date;
}>;

type PendingInvitation = InvitationBase & Readonly<{ status: "pending" }>;
type AcceptedInvitation = InvitationBase &
  Readonly<{ status: "accepted"; acceptedAt: Date; memberId: MemberId }>;
type RevokedInvitation = InvitationBase & Readonly<{ status: "revoked"; revokedAt: Date }>;
type Invitation = PendingInvitation | AcceptedInvitation | RevokedInvitation;

type InvitationStanding = "usable" | "expired" | "accepted" | "revoked";
```

| フィールド | 制約 |
| --- | --- |
| `email` | 招待の宛先。生成後に変わらない。`EmailAddress` は [common.md](common.md) が定める |
| `grant` | `member` はメンバー管理の権限を持つメンバーによる招待で、付与する権限の組を持つ。`initial_owner` はホスティング運営者による初期所有者の招待 |
| `tokenDigest` | 招待の値の照合値。生成後に変わらない。招待の値そのものは保存しない |
| `expiresAt` | `createdAt` の `InvitationPolicy.validMs` 後。生成後に変わらない |
| `version` | 生成時は 0。`accept`、`revoke` で 1 増える。`revoked` の招待への `revoke` は進めない |

招待の値は、[common.md](common.md) の「期限つきの一度きりの値」に従う。

振る舞い。

```ts
const Invitation: {
  issue(
    params: {
      id: string;
      email: string;
      grant: InvitationGrant;
      tokenDigest: InvitationTokenDigest;
    },
    now: Date,
  ): PendingInvitation;
  accept(invitation: PendingInvitation, memberId: MemberId, now: Date): AcceptedInvitation;
  revoke(invitation: Invitation, now: Date): RevokedInvitation;
  standing(invitation: Invitation, now: Date): InvitationStanding;
  lapsedAt(invitation: Invitation): Date;
  isManagedByMembers(invitation: Invitation): boolean;
};
```

| 振る舞い | 処理 |
| --- | --- |
| `issue` | `InvitationId` と `EmailAddress` を生成し、`pending` の招待を返す |
| `accept` | `pending` の招待だけを受け取り、`accepted` にして、参加したメンバーを記録する。期限を判定しない。`InvitationProcess.accept` が、`InvitationProcess.availability` で期限内の招待に絞ってから呼ぶ |
| `revoke` | `pending` の招待を、期限によらず `revoked` にする。`revoked` の招待は、渡された招待をそのまま返し、`version` を進めない（変更の無い変更）。`accepted` の招待は `BusinessRuleError`（`InvitationAccepted`） |
| `standing` | `pending` で期限内なら `usable`、期限切れなら `expired`（期限の境界は [common.md](common.md) の「期限つきの一度きりの値」）。それ以外は `status` を返す |
| `lapsedAt` | 保持期間の起点。値は [common.md](common.md) の「期限つきの一度きりの値」に従う |
| `isManagedByMembers` | `grant` が `member` なら真、`initial_owner` なら偽 |

招待を受諾できるかどうかは `InvitationProcess.availability` が、取り消せる状態は `revoke` が判定する。`standing` は、招待の値の状態（期限内の `pending` か、期限切れか、使用済みか、失効済みか）だけを表し、受諾する人とワークスペースの状態を見ない。ユースケースは状態を判定しない。

不変条件。

- `accepted` の招待は取り消せない
- 招待の宛先、付与する内容、有効期限、招待の値は、発行の後に変わらない。同じ宛先へ招待し直すと、同じ種類の前の `pending` の招待は `revoked` になり、新しい招待が置き換える
- 招待の値は、発行の時にだけ得られる。招待の URL にしてメールで送るか、設置全体のメール送信を使えない設置では、発行した人に一度だけ返す
- 招待の一意性は、種類（`grant` の `kind`）ごとに分かれる。同じワークスペースの同じ宛先への、`grant` が `member` の `pending` の招待は、最大一つである。一つのワークスペースの `pending` の初期所有者の招待は、宛先によらず、最大一つである
- メンバーの招待と初期所有者の招待は、互いを妨げず、互いを失効させない。同じ宛先に、`pending` のメンバーの招待と `pending` の初期所有者の招待が、一つずつあり得る。メンバーの招待の発行は初期所有者の招待を変えず、初期所有者の招待の発行と発行し直しはメンバーの招待を変えない
- 招待は、区画の集約で、ワークスペースの ID を持たない。参加するワークスペースは、区画が決める。招待の値が、ワークスペースの ID を含む
- 所有者は、`member` の招待では指定できない。`grant` が `member` の招待は、権限の組だけを与える
- 初期所有者の招待（`grant` が `initial_owner`）は、ホスティング運営者だけが発行し、発行し直す。メンバーによる一覧と取り消しの対象は、`isManagedByMembers` が真の招待に限る。偽の招待へのメンバーの操作は、存在しない対象として扱う（`NotFoundError`）

ライフサイクル。

- 生成: メンバー管理の権限を持つメンバーが招待したとき（`member`）。ホスティング運営者がワークスペースを作成したとき、および初期所有者の招待を発行し直したとき（`initial_owner`。「初期構成の宣言」の初期所有者の招待の発行し直し）
- 遷移: `pending` →（`accept`）→ `accepted`。`pending` →（`revoke`）→ `revoked`
- 削除: ワークスペースの利用終了で、そのワークスペースの招待をすべて削除する。定期ジョブ「期限切れの記録の掃除」が、[common.md](common.md) の「期限つきの一度きりの値」の対象を削除する

### JoiningGuard

参加の囲い。アカウントごとに、そのワークスペースへの進行中の参加を見分ける世代を持つ。参加の後段は、その参加が取り下げられていない（`JoiningGuard.withdrawn` が偽の）場合だけ確定する。取り下げは「MembershipRepair」の「取り下げ」。招待の種類と、参加の経路によらない。

フィールド。

```ts
type JoiningGuard = Readonly<{
  accountId: AccountId;
  generation: number;
  version: number;
}>;

type JoiningTicket = Readonly<{ generation: number }>;
```

| フィールド | 制約 |
| --- | --- |
| `accountId` | 必須。区画の中で一意。集約を識別する |
| `generation` | 参加の世代。生成時は 1。`advance` で 1 増える。区画に囲いが無いアカウントの世代は 0 |
| `version` | 生成時は 0。`advance` で 1 増える |

振る舞い。

```ts
const JoiningGuard: {
  ticket(guard: JoiningGuard | null): JoiningTicket;
  advance(guard: JoiningGuard | null, accountId: AccountId): JoiningGuard;
  withdrawn(joining: JoiningTicket, guard: JoiningGuard | null, member: Member | null): boolean;
};
```

| 振る舞い | 処理 |
| --- | --- |
| `ticket` | 囲いの世代を返す。`null`（区画に囲いが無い）なら 0 |
| `advance` | 世代を 1 進めた囲いを返す。`null` なら、`accountId` の、世代が 1 の囲いを作る |
| `withdrawn` | 参加が取り下げられたかどうか。`member`（受諾する人の、区画の `Member`）が `null` で、`ticket(guard).generation` が `joining.generation` と違えば真。`joining` は、受諾の事前の確認が前段より前に読んだ世代。`guard` は、後段が読んだ囲い。取り下げられたかどうかの判定は、この関数だけが持つ |

- 世代を進めるのは、参加の後段（受諾）と、取り下げの二つ。どちらも、読んだ囲いに `advance` を適用して保存するので、同じアカウントの並行した後段と取り下げは、一方だけが確定する（`JoiningGuardRepository`）

ライフサイクル。

- 生成: そのアカウントの、最初の参加の後段、または最初の取り下げ
- 遷移: `advance` で世代が進む
- 削除: ワークスペースの利用終了（「利用終了の消去」）

## 値オブジェクト

ブランド付きの型は、`create` だけが生成する。利用者の入力から作る値オブジェクト（`WorkspaceName`、`Permission`、`PermissionSet`、招待の宛先の `EmailAddress`）の違反は、`ValidationError`（項目ごとの理由。理由は表のコード）になる。利用者の入力に由来しない値（照合値）の違反は `BusinessRuleError`（`InvalidStoredValue`）になる。集約の ID、`Actor`、`Versioned`、`ExpectedVersion`、`TransactionalRepository`、`SingletonRepository`、`Pagination`、`PaginationResult`、`EmailAddress` は [common.md](common.md) の定義を使う。

| 名前 | 型 | バリデーションルール | 等価性 |
| --- | --- | --- | --- |
| `WorkspaceName` | `string`（ブランド） | 前後の空白を除いた値を保持する。除いた値が 1 文字以上 100 文字以下。違反は `InvalidWorkspaceName` | 値の一致 |
| `Permission` | 文字列リテラルの直和型 | 下の語彙のいずれか。違反は `UnknownPermission` | 値の一致 |
| `PermissionSet` | `readonly Permission[]`（ブランド） | 重複を除き、語彙の定義順に並べて保持する | 要素の一致 |
| `StandardRole` | `"admin" \| "editor" \| "viewer"` | 下の対応 | 値の一致 |
| `InvitationTokenDigest` | `string`（ブランド） | 空でない。招待の値の照合値 | 値の一致 |
| `MemberGrant` | 直和型 | 下の定義。所有者か、権限の組を持つメンバーか。`MemberAuthorization.grantOf` が生成する | — |
| `MemberAccess` | 直和型 | `MemberAuthorization.resolve` が生成する | — |
| `InvitationPolicy` | 定数 | `validMs` は招待の有効期間。値は [../scenario/app-integration.md](../scenario/app-integration.md) の APP-12 の表（「招待の有効期間」）が定め、技術設計の層では、この定数だけが持つ | — |
| `InvitationLink` | 定数と関数 | 下の定義 | — |
| `DirectoryStatus` | `"provisioning" \| "open" \| "retiring"` | 下の定義 | 値の一致 |
| `WorkspaceDirectoryEntry` | 直和型 | 下の定義 | `id` の一致 |
| `Membership` | 直和型 | 下の定義。所属の索引の項目。どの状態も世代を持つ。`joining` の項目だけが、参加を始めた日時を持つ | `accountId` と `workspaceId` の組の一致 |
| `MembershipGeneration` | `number`（ブランド） | 所属の索引の項目の世代。`MembershipIndex` が割り当てる。呼び出し側は、読んだ項目の値をそのまま渡す | 値の一致 |
| `MembershipPolicy` | 定数 | `joiningMs` は、`joining` の項目を、参加の途中として保つ期間。値は [../scenario/app-integration.md](../scenario/app-integration.md) の APP-12 の表（「参加を取り下げるまでの時間」）が定め、技術設計の層では、この定数だけが持つ | — |

招待の値の扱いは [common.md](common.md) の「期限つきの一度きりの値」と「ProtectedSecret」に従う。

### Permission

権限の語彙。型の定義は [common.md](common.md) の「Permission」にある。Workspace は、値と権限の名前の対応と、権限の組の判定を定める。権限ごとに行える操作は [../scenario/index.md](../scenario/index.md) の「権限」の表が定める。

| 値 | 権限 |
| --- | --- |
| `content:edit` | 編集権限 |
| `content:publish` | 公開権限 |
| `content:delete` | 削除権限 |
| `media:publish` | メディア公開権限 |
| `schema:change` | スキーマ変更権限 |
| `submission:read` | 回答の閲覧権限 |
| `submission:delete` | 回答の削除権限 |
| `notification:manage` | 通知設定の権限 |
| `intake:manage` | 受付管理の権限 |
| `member:manage` | メンバー管理の権限 |
| `integration:manage` | アプリ連携の権限 |
| `workspace:manage` | ワークスペース設定の権限 |

- 権限はどの組み合わせでも付与できる。包含の判定は `PermissionSet.allows` が行う
- 各操作が必要とする権限は、その操作のユースケースが `Permission` の値で指定する。公開中のコンテンツのごみ箱への移動のように複数の権限を必要とする操作は、必要な権限をすべて指定する

### PermissionSet

```ts
const PermissionSet: {
  create(values: readonly string[]): PermissionSet;
  empty(): PermissionSet;
  all(): PermissionSet;
  allows(set: PermissionSet, permission: Permission): boolean;
  missing(set: PermissionSet, required: readonly Permission[]): readonly Permission[];
  intersect(a: PermissionSet, b: PermissionSet): PermissionSet;
  equals(a: PermissionSet, b: PermissionSet): boolean;
};
```

| 関数 | 処理 |
| --- | --- |
| `create` | 語彙にない値は `ValidationError`（`UnknownPermission`）。重複を除き、語彙の定義順に並べる |
| `all` | 語彙のすべての権限を持つ組 |
| `allows` | 組が `permission` を持てば真。`submission:read` は、組が `submission:delete` を持つ場合も真 |
| `missing` | `required` のうち、`allows` が偽になる権限を、`required` の順で返す。空なら不足が無い |
| `intersect` | 両方の組で `allows` が真になる権限の組 |
| `equals` | 保持する要素が一致すれば真 |

### StandardRole

```ts
const StandardRole: {
  permissions(role: StandardRole): PermissionSet;
  of(set: PermissionSet): StandardRole | null;
};
```

| 役割 | 値 | 権限の組 |
| --- | --- | --- |
| 管理者 | `admin` | すべての権限 |
| 編集者 | `editor` | `content:edit`、`content:publish`、`content:delete`、`media:publish` |
| 閲覧者 | `viewer` | 空の組 |

- `permissions` は、役割に対応する権限の組を返す
- `of` は、権限の組がいずれかの役割の組と `equals` で一致すれば、その役割を返す。一致しなければ `null`（個別に組み合わせた権限）
- 役割は権限の組の入力と表示にだけ使う。認可は権限の組で判定する

### MemberAccess

```ts
type MemberGrant =
  | Readonly<{ kind: "owner" }>
  | Readonly<{ kind: "member"; permissions: PermissionSet }>;

const MemberGrant: {
  permissions(grant: MemberGrant): PermissionSet;
  isOwner(grant: MemberGrant): boolean;
};

type MemberAccess =
  | Readonly<{ status: "granted"; memberId: MemberId; grant: MemberGrant }>
  | Readonly<{ status: "mfa_setup_required"; memberId: MemberId }>
  | Readonly<{ status: "not_available" }>;
```

`MemberGrant` は、メンバーが行える操作の範囲。

| `kind` | 意味 |
| --- | --- |
| `owner` | 所有者。すべての権限の操作と、所有者だけの操作を行える。メンバーが保持する権限の組に依らないので、権限の組を持たない |
| `member` | 所有者でないメンバー。`permissions`（メンバーが保持する権限の組）の範囲で操作を行える |

- `MemberGrant.permissions` は、`owner` なら `PermissionSet.all()`、`member` なら `permissions` を返す。権限の判定（`PermissionSet.missing`）と、メンバーの一覧に示す権限の組は、この結果を使う
- `MemberGrant.isOwner` は、`owner` なら真

| 状態 | 意味 |
| --- | --- |
| `granted` | 利用できる。`grant` の範囲で操作を行える |
| `mfa_setup_required` | メンバーだが、多要素認証を必須にしたワークスペースで、多要素認証を設定していない |
| `not_available` | 利用できないワークスペース（`WorkspaceAvailability.of` が偽）、またはメンバーでない |

各状態をどのエラーにするかは、[index.md](index.md) の「認可」の手続きが定める。

### WorkspaceDirectoryEntry

ワークスペースの名簿の一件。設置の区画にあり、`WorkspaceDirectory` が保存する。集約に数えない。

```ts
type DirectoryStatus = "provisioning" | "open" | "retiring";

type WorkspaceCreation<TFinished extends boolean = boolean> = Readonly<{
  initialOwner: EmailAddress;
  finished: TFinished;
}>;

type DirectoryEntryBase = Readonly<{
  id: WorkspaceId;
  name: WorkspaceName;
}>;

type WorkspaceDirectoryEntry =
  | (DirectoryEntryBase & Readonly<{ status: "provisioning"; creation: WorkspaceCreation<false> | null }>)
  | (DirectoryEntryBase & Readonly<{ status: "open"; creation: WorkspaceCreation | null }>)
  | (DirectoryEntryBase & Readonly<{ status: "retiring"; creation: WorkspaceCreation | null }>);

type DirectoryStatusChange =
  | Readonly<{ status: "open" }>
  | Readonly<{ status: "retiring" }>;

const WorkspaceDirectoryEntry: {
  statusFor(workspace: Workspace | null, current: DirectoryStatus): DirectoryStatus;
  confirmName(entry: WorkspaceDirectoryEntry, confirmationName: string): void;
};
```

| 状態 | 意味 |
| --- | --- |
| `provisioning` | 準備中。名簿に登録し、開設済みにしていない |
| `open` | 開設済み。作成が、区画の `Workspace` を確定した後に、開設済みにした。初期所有者が未参加の間（所有者の指定が `vacant`）も `open` |
| `retiring` | 終了中。利用終了が名簿の状態を変えてから、名簿から外すまで続く |

- 入口が受け付けるかどうかは `WorkspaceAvailability.of` が、対象にする定期ジョブは [index.md](index.md) の「定期ジョブ」の表が、状態確認の示し方は「状態確認の項目」が定める
- `creation` は、運用コマンドの作成が登録した項目の、作成の指定と進み具合。`initialOwner` は指定した初期所有者のメールアドレス、`finished` は、作成を終える段（「初期構成の宣言」の初期所有者の招待の発行し直し）が確定したか。作成を終える段は開設済みにした後にだけ来るので、準備中の項目の `finished` は偽だけを取る。初期構成で宣言したワークスペースの項目は `null`。作成の実行し直しは、`id` と `name` と `initialOwner` で同じ作成かを確かめ、`finished` で続けるかを決める（「初期構成の宣言」の運用コマンドの作成）
- `id`、`name`、`creation.initialOwner` は、登録の後に変わらない。名称は、名簿の項目だけが持つ。招待の内容、招待のメール、利用終了の確認は、名簿の項目の名称を使う
- 「開始」の語は、区画の `Workspace` を `closing` にする段だけを指す。開始の確定は、scenario の HOST-09 の「書き込みを止めた時点」に当たる
- 名簿を終了中にする手順（利用終了の名簿の状態の変更、「利用再開前の処理」）は、どれも、`{ status: "retiring" }` を `WorkspaceDirectory.changeStatus` に渡す。区画の `Workspace` を読まずに行える
- 遷移: `provisioning` → `open`（作成が、開設済みにした）。`provisioning` / `open` → `retiring`（利用終了の、名簿の状態の変更）。`retiring` から出る遷移は、名簿からの除外だけ
- `Workspace` の状態の正は、区画の `Workspace` が持つ。名簿の状態と区画の `Workspace` を確定する順は、[index.md](index.md) の「区画をまたぐ手順」が定める。途中で止まった手順の食い違いは、手順の続きが直す。バックアップからの復旧の後の食い違いは、「利用再開前の処理」が、下の `statusFor` の表で直す

`statusFor` は、区画の `Workspace` から、名簿が持つべき状態を求める。「利用再開前の処理」が使う。名簿の状態の直し方は、この表だけが定める。

| `workspace` | `current` | 結果 |
| --- | --- | --- |
| どれでも | `retiring` | `retiring` |
| `active` | `provisioning` または `open` | `open` |
| `closing` または `closed` | `provisioning` または `open` | `retiring` |
| `null`（区画に `Workspace` が無い） | `provisioning` | `provisioning` |
| `null` | `open` | `retiring` |

`confirmName` は、`confirmationName` から生成した `WorkspaceName` が `entry.name` と一致しなければ `BusinessRuleError`（`WorkspaceNameMismatch`）を投げる。項目の状態を見ない。利用終了の対象の確認が使う。名称の確認は、この振る舞いだけが行う。

### Membership

所属の索引の一件。アカウントとワークスペースの組が鍵。設置の区画にあり、`MembershipIndex` が保存する。集約に数えない。

```ts
type MembershipKey = Readonly<{ accountId: AccountId; workspaceId: WorkspaceId }>;

type MembershipBase = MembershipKey & Readonly<{ generation: MembershipGeneration }>;

type Membership =
  | (MembershipBase & Readonly<{ status: "joining"; since: Date }>)
  | (MembershipBase & Readonly<{ status: "joined" }>);

const MembershipPolicy: {
  readonly joiningMs: number;
  joiningLapsed(since: Date, now: Date): boolean;
};
```

| 状態 | 意味 |
| --- | --- |
| `joining` | 参加の途中。参加の前段を確定し、メンバーの作成を確かめていない。`since` は、前段の日時 |
| `joined` | 参加済み。メンバーの作成を確かめた |

- 一つの組の項目は最大一つ
- 世代（`generation`）は、項目を見分ける値。項目を置く書き込み（既にある項目の置き換えを含む）は、その設置の索引でそれまでに割り当てたどの世代とも違う値を割り当て、既にある項目の状態だけを変える書き込みは世代を引き継ぎ、項目を外す書き込みは、読んだ項目と同じ世代の項目だけを外す。メソッドごとの割り当ては `MembershipIndex` の契約が、この規則から導く `Member` との対応は「`Member` と項目の対応」が定める
- 項目ごとの遷移は、下の「項目の遷移」の表がすべて
- 参加の期限は、`since` の `MembershipPolicy.joiningMs` 後。`MembershipPolicy.joiningLapsed(since, now)` は、`now` がその時刻以降（同じ時刻を含む）なら真を返す。期限の判定は、この関数だけが持ち、所属の索引の修復（`MembershipRepair.decide`）だけが使う。`since` は、期限の起点としてだけ使う。期限は、修復が参加の取り下げを始める時期だけを決め、参加の後段は、取り下げられていなければ、期限の後も確定する
- 最後に利用したワークスペースは、アカウントごとに一つの値（ワークスペースの ID、または無し）で、索引が項目と別に持つ。値を置き換えるのは `MembershipIndex.markLastUsed` だけで、項目の書き込みは値を変えない。値が指すワークスペースを利用できるかどうかは、`InitialWorkspace.choose` だけが見る
- 所属の正は、ワークスペースの区画の `Member` が持つ。メンバーの認可（`MemberAuthorization.resolve`）は、`Member` だけで決まり、索引を読まない。索引と `Member` の食い違いは、`MembershipRepair`、利用終了、「利用再開前の処理」が直す
- 初期構成で宣言した所有者の項目も、`joining` で置き、名簿を開設済みにする段で `joined` にする

項目の遷移。書き込みは `MembershipIndex` のメソッド。

| 遷移 | 書き込み | 行う手順 |
| --- | --- | --- |
| （無し）→ `joining` | `beginJoining` | 参加の前段。初期構成の読み込みの、設置の区画の最初の段 |
| `joining`、`joined` → `joining`（新しい世代） | `beginJoining` | 項目があるアカウントの、受諾の前段 |
| `joining` → `joined` | `confirm` | 参加の確かめ直し。初期構成の読み込みの、名簿を開設済みにする段。修復の `confirm`。「利用再開前の処理」の確定 |
| （無し）→ `joined` | `confirm` | 「利用再開前の処理」の確定。項目が外れた後に確定した、参加の確かめ直し（利用終了の最後の段の後、または、名簿で開設済みのワークスペースで、後段の後に利用の解除と取り下げで項目が外れた後） |
| `joining`、`joined` →（無し） | `remove` | 修復の取り下げの項目の除外（`MembershipRepair.decide` が `withdraw` を返す項目）。修復の `remove`（`decide` が `remove` を返す項目）。「利用再開前の処理」の除外 |
| `joining`、`joined` →（無し） | `removeWorkspace` | 利用終了の最後の段。そのワークスペースの項目を、状態によらずすべて外す |

### InvitationLink

招待の URL の形。URL の組み立ての分担は [index.md](index.md) の「URL の組み立て」が定める。

```ts
const InvitationLink: {
  readonly path: "/invitations";
  url(publicUrl: string, token: string): string;
};
```

- `url` は、`path` と招待の値（`SecretTokenIssuer.issueForWorkspace` の `value`）から、[common.md](common.md) の「期限つきの一度きりの値」の URL の組み立てで URL を作る。`publicUrl` は、設置の公開の URL
- 経路の形は、この定数だけが持つ
- 保存を伴わない純粋な関数

## ドメインサービス

### WorkspaceAvailability

責務: ワークスペースを利用できるかどうかを、名簿の状態と区画の `Workspace` の状態から求める。利用できるかどうかの判定は、この関数だけが持つ。入口ごとの結果は、[index.md](index.md) の「利用できるワークスペース」の表が定める。

```ts
const WorkspaceAvailability: {
  of(directory: DirectoryStatus | null, workspaceStatus: WorkspaceStatus | null): boolean;
};
```

- `directory` が `open` で、`workspaceStatus` が `active` のときだけ真。それ以外は偽
- `directory` は、名簿のそのワークスペースの状態（`WorkspaceDirectory.findById` の項目の `status`）。`null` は、名簿に項目が無いこと。`workspaceStatus` は、区画の `Workspace` の `status`。`null` は、区画に `Workspace` が無いこと
- `directory` が `open` でなければ、`workspaceStatus` によらず偽。呼び出し側は、区画を読まずに `null` を渡せる
- 名簿が `open` でないワークスペースは、区画の `Workspace` が `active` でも偽。作成の途中（名簿が準備中で、区画に `Workspace` がある）と、利用終了の名簿の状態の変更の後（名簿が終了中で、区画の `Workspace` が `active`）が当たる
- 区画の `Workspace` が `closing` または `closed` のワークスペースは、名簿の状態によらず偽
- 呼ぶのは `MemberAuthorization.resolve` と `InvitationProcess.availability`

依存するポート: 無し。

### MemberAuthorization

責務: ワークスペース、アクセスの方針、メンバー、多要素認証の設定から、そのメンバーがその時点で行える操作の範囲を求める。

```ts
const MemberAuthorization: {
  grantOf(policy: AccessPolicy, member: Member): MemberGrant;
  resolve(input: {
    directory: DirectoryStatus | null;
    workspace: Workspace | null;
    policy: AccessPolicy | null;
    member: Member | null;
    mfaEnabled: boolean;
  }): MemberAccess;
};
```

`grantOf` は、多要素認証の状態によらず、メンバーが行える操作の範囲を求める。

- `AccessPolicy.isOwner` が真なら `owner`。メンバーが保持する権限の組を見ない。所有者がいない（`vacant`）ワークスペースでは、どのメンバーにも偽
- それ以外は、`member.permissions` を持つ `member`
- メンバーの一覧と、`resolve` の `granted` が使う

`resolve` は、上の行から順に確かめ、初めに当たった行の結果を返す。

| 条件 | 結果 |
| --- | --- |
| `WorkspaceAvailability.of(directory, workspace の status)` が偽（`workspace` が `null` なら、`null` を渡す）、`policy` が `null`、または `member` が `null` | `not_available` |
| `policy.mfaRequired` が真で、`mfaEnabled` が偽 | `mfa_setup_required` |
| それ以外 | `grantOf` の結果を `grant` にした `granted` |

- `workspace`、`policy`、`member` は、同じ区画から読んだもの。呼び出すユースケースが、区画を一つ決めて読む
- `directory` は、名簿のそのワークスペースの状態。項目が無ければ `null`。呼び出すユースケースが、設置の区画から読んで渡す。ワークスペースを利用できるかどうかは `WorkspaceAvailability.of` だけが決める
- 所有者の権限は、`grantOf` の導出だけで決まり、保存した値として持たない
- 多要素認証の判定は、所有者とほかのメンバーに同じ規則を適用する。多要素認証を必須にしたワークスペースで、所有者が多要素認証を設定していなければ、所有者の結果も `mfa_setup_required` になる
- 読むものと、結果のエラーへの対応、呼ぶ時点は、[index.md](index.md) の「認可」のメンバーの認可の手続きが定める

依存するポート: 無し。

### InitialWorkspace

責務: ログインの後に開くワークスペースを、アカウントが利用できるワークスペースから一つ選ぶ。

```ts
const InitialWorkspace: {
  choose(input: {
    lastUsed: WorkspaceId | null;
    workspaces: readonly Readonly<{ id: WorkspaceId; name: WorkspaceName }>[];
  }): WorkspaceId | null;
};
```

- `workspaces` は、所属ワークスペースを求める手続き（[../usecases/workspace.md](../usecases/workspace.md) の「共通」）が返すワークスペース。名称は、名簿の項目の名称。`lastUsed` は、そのアカウントが最後に利用したワークスペースの ID（`MembershipIndex.findLastUsed`）。無ければ `null`
- 所属ワークスペースの並びは、名称の、文字列のコードポイントの昇順。名称が同じワークスペースは、`WorkspaceId` の文字列のコードポイントの昇順。所属ワークスペースを求める手続きの出力と、下の先頭は、この並びを使う。並びの規則は、この箇条だけが定める
- `lastUsed` が `workspaces` にあれば、それを返す
- `lastUsed` が `null`、または `workspaces` に無い（解除された、利用を終了した）場合は、`workspaces` のうち、上の並びで先頭のものを返す
- `workspaces` が空なら `null`

依存するポート: 無し。

### MembershipRepair

責務: 所属の索引の項目と、ワークスペースの区画の `Member` の食い違いを、どちらへ直すかを決める。正は `Member` が持つ。修復の規則は、この節だけが定める。ほかの文書は、`MembershipRepair.decide` を参照する。修復を行うユースケースは、[index.md](index.md) の「区画をまたぐ手順」の所属の索引の修復の行が定める。どのユースケースも同じ表を使い、表の行は、呼び出すユースケースで分かれない。`Member` と合わない項目が残る状態から出ることは、すべての項目を走査する `repairMembershipIndex` が担う。

```ts
type MembershipRepairAction = "confirm" | "remove" | "withdraw" | "keep";

const MembershipRepair: {
  decide(
    input: {
      membership: Membership | null;
      member: Member | null;
      directory: DirectoryStatus | null;
    },
    now: Date,
  ): MembershipRepairAction;
};
```

- `membership` は、所属の索引の、そのアカウントとワークスペースの項目（`MembershipIndex.find` または `MembershipIndex.scan` の結果）。`directory` は、名簿のそのワークスペースの状態で、名簿に無ければ `null`。`member` は、そのワークスペースの区画の、そのアカウントの `Member`。`directory` が `open` でないときは、区画を読まずに `null` を渡してよい
- 呼び出すユースケースは、索引の項目を読んだ後に、区画の `Member` を読む

上の行から順に確かめ、初めに当たった行の結果を返す。

| `directory` | `member` | `membership` | 結果 |
| --- | --- | --- | --- |
| `provisioning`、`retiring` | 見ない | どれでも | `keep`。準備中は、初期構成の読み込みが `joined` にする。終了中は、利用終了の最後の段（「利用終了の消去」）が外す |
| `null` | 見ない | 無い | `keep` |
| `null` | 見ない | ある（状態によらない） | `remove`（項目を外す）。名簿に無いワークスペースの項目。修復を行う二つのユースケースのどちらも、この行で外す。`removeMember` の後段は、前段の確定の後に利用終了が最後の段まで完了し、その後に、解除したアカウントの組の項目が置かれていた場合にだけ当たる。それ以外の項目は、定期ジョブ「所属の索引の修復」が外す |
| `open` | ある | `joining` | `confirm`（項目を `joined` にする） |
| `open` | ある | `joined` | `keep` |
| `open` | ある | 無い | `confirm`（項目を `joined` で置く）。項目を読んだ後に参加が確定した組が当たる。その組の項目は既にあり、`confirm` は、それを `joined` にする |
| `open` | 無い | 無い | `keep` |
| `open` | 無い | `joined` | `withdraw`（取り下げてから、項目を外す）。利用を解除されたメンバーの項目が当たる。期限を待たない |
| `open` | 無い | `joining` | `MembershipPolicy.joiningLapsed(since, now)` が真なら `withdraw`。偽なら `keep`。期限の前は、参加の後段を待つ |

- 結果が `confirm` のとき、呼び出すユースケースは、その組を `MembershipIndex.confirm` に渡す
- 結果が `remove` のとき、呼び出すユースケースは、`decide` に渡した項目（`membership`）を `MembershipIndex.remove` に渡す
- 結果が `withdraw` のとき、呼び出すユースケースは、下の「取り下げ」を行う。名簿で開設済みのワークスペースの項目を外すのは、取り下げの確定の後だけ
- 後段が確定しなかった参加は、前段が置いた `joining` の項目を自分で外さない。項目は、期限まで残り、この決定が取り下げて外す。並行した別の参加が `Member` を作っていれば、この決定が `joined` にする
- 修復の UnitOfWork（`confirm` と `remove` の UnitOfWork、取り下げの確定、項目の除外）が `ConflictError` で確定しなかった場合、修復を行うユースケースは、エラーにせず、その組を確定しなかったものとして扱う。理由（囲いの競合、`WORKSPACE_NOT_ACTIVE` を含む）によらない。項目は変わらずに残り、次の修復が直す。`removeMember` の後段は、修復が確定しなくても正常に終わる。`repairMembershipIndex` は、確定しなかった組を件数に数え、残りの項目を続ける
- 索引、名簿、区画の読み取りの失敗は、確定しなかった修復に当たらない。修復を行うユースケースは、エラーを投げて終える

#### 取り下げ

名簿で開設済みのワークスペースの項目を外す前に、その項目を読むより前に始まった参加が `Member` を作れないようにする手順。二つの UnitOfWork を、この順に確定する。

- 取り下げの確定。ワークスペースの区画の一つの UnitOfWork（条件 `active`）。そのアカウントの参加の囲いを `JoiningGuardRepository.findById` で読み、その後に、`MemberRepository.findByAccount` で、そのアカウントの `Member` を読み直す。`Member` があれば、何も書かずに終え、呼び出すユースケースは、その組を `MembershipIndex.confirm` に渡す。`Member` が無ければ、読んだ囲い（無ければ `null`）に `JoiningGuard.advance` を適用し、囲いが無かった場合は `insert`、あった場合は読んだ `expectedVersion` で `save` する。書き込みは 1 回
- 項目の除外。取り下げの確定の後に、設置の区画の一つの UnitOfWork で、`decide` に渡した項目を `MembershipIndex.remove` に渡す。読んだ項目と同じ世代の項目だけが外れる
- 取り下げの確定が確定しなかった場合（参加の後段との囲いの競合、`WORKSPACE_NOT_ACTIVE` を含む）、項目を外さない。項目は残り、次の修復が、`Member` を読み直して決める
- 取り下げの確定の後、項目の除外の前に止まった場合、項目は残り、次の修復が、同じ手順を行う。囲いの世代は、もう一度進む
- 取り下げは、招待を変えない。取り下げの後に事前の確認から始めた受諾は、新しい世代を読み、確定する
- 取り下げの確定より前に事前の確認を行った受諾の後段は、確定しない（「InvitationProcess」の「事前の確認の後の変化」、または囲いの競合）

#### `Member` と項目の対応

名簿に項目があるワークスペースの区画に `Member` があるアカウントは、所属の索引に、そのワークスペースの項目を持つ。稼働中、この対応は崩れない。根拠は、項目の世代の規則と、利用終了の順の二つ。

- 規則: `Member` を作る確定は、その組に新しい世代の項目を置いた確定の後に行う。項目を外す確定は、その項目を読んだ後に、読むより前に始まった参加が `Member` を作れないことを区画で確定してから、読んだ世代の項目だけを外す
- `Member` を作る確定は、参加の後段と、初期構成の読み込みの区画の段の二つ。参加の後段は、前段（`MembershipIndex.beginJoining`）の確定の後に行う。初期構成の読み込みは、所有者の項目を置く段の後に `Member` を作る
- 名簿に項目があるワークスペースの項目を外す確定は、取り下げの項目の除外、利用終了の最後の段、「利用再開前の処理」の所属の索引の除外の三つ
- 取り下げは、取り下げの確定で、`Member` が無く、項目を読むより前に始まった参加が `Member` を作れないこと（参加の囲いの世代が進んだこと）を確定する
- 利用終了の最後の段は、名簿からの除外と同じ確定で、そのワークスペースの項目をすべて外す。その確定の後、そのワークスペースは名簿に無く、この対応の対象の外になる
- 名簿の状態の変更より前に名簿を読んだ修復の取り下げは、取り下げの確定で `Member` が無いことを確かめた世代の項目だけを外し、その後に `Member` を作った参加の項目は、世代が違う
- 「利用再開前の処理」の除外は、利用再開前の設置で行い、世代の規則に依らない。利用再開前の設置では、[installation.md](installation.md) の「設置の状態」の `admit` の表が参加を受け付けないので、進行中の参加が無く、項目を読んだ後に `Member` を作る確定が無い。処理は、`joined` の項目が無い `Member` の組を確定し、区画に `Member` が無い項目だけを外す。処理を終えた設置では、名簿で開設済みのワークスペースに `Member` があるアカウントは、そのワークスペースの項目を持ち、稼働中の対応は、この状態から始まる
- 導出: 世代 g の項目を外す手順は、g の項目を読み、`Member` が無いことを区画で確定し（時点 T）、g の項目を外す。T より前に `Member` を作った参加があれば、T の確定が `Member` を読み、項目を外さない。T より後に `Member` を作る参加は、g の項目を読んだ後に前段を確定し、その前段は、g と違う世代の項目を置く。前段が項目の除外より先なら、索引の項目は g でないので外れない。後なら、外れた後に項目を置く。どの順でも、`Member` を作る時点と、その後に、その組の項目がある
- 並行の一例（利用を解除されたアカウントの参加し直し）: 修復が `joined` の項目（世代 g1）と、`Member` が無いことを読む。そのアカウントの受諾が、前段（項目は世代 g2 の `joining`）、後段（`Member` を作る）、確かめ直し（項目は世代 g2 の `joined`）を確定する。修復の取り下げの確定は、`Member` を読み、何も書かずに終える。修復が取り下げの確定を受諾の事前の確認より先に確定していた場合も、項目の除外は世代 g1 の項目だけを外し、世代 g2 の項目を外さない
- 名簿に無いワークスペースの項目の `remove` は、この対応の外にある
- 稼働中、アカウントが無い `Member` は生じない（アカウントは削除されない。[identity.md](identity.md) の「Account」のライフサイクル）。生じるのは、バックアップからの復旧の後だけで、「利用再開前の処理」が直す

依存するポート: 無し。

### MemberAdministration

責務: メンバーの権限の変更、利用の解除、所有者の指定と変更を行う。メンバーの操作で、所有者がいなくならないようにする。

```ts
type OwnershipTransfer = Readonly<{ policy: OwnedPolicy; formerOwner: Member }>;

type InitialOwnerDesignation = Readonly<{
  policy: OwnedPolicy;
  owner: Member;
  created: boolean;
}>;

const MemberAdministration: {
  changePermissions(
    input: { policy: AccessPolicy; member: Member; permissions: PermissionSet },
    now: Date,
  ): Member;
  assertRemovable(input: { policy: AccessPolicy; member: Member }): void;
  transferOwnership(
    input: { policy: OwnedPolicy; formerOwner: Member; newOwner: Member },
    now: Date,
  ): OwnershipTransfer;
  designateInitialOwner(
    input: {
      policy: VacantPolicy;
      existing: Member | null;
      memberId: string;
      accountId: AccountId;
    },
    now: Date,
  ): InitialOwnerDesignation;
};
```

| メソッド | 処理 |
| --- | --- |
| `changePermissions` | `AccessPolicy.isOwner` が真のメンバーなら `BusinessRuleError`（`OwnerPermissionsDerived`）。それ以外は `Member.changePermissions` を呼ぶ |
| `assertRemovable` | `AccessPolicy.isOwner` が真のメンバーなら `BusinessRuleError`（`OwnerCannotBeRemoved`） |
| `transferOwnership` | `formerOwner.id` が `policy` の所有者の指定と違えば `BusinessRuleError`（`OwnerMismatch`）。`AccessPolicy.changeOwner` を呼ぶ（現在の所有者は `AlreadyOwner`）。`formerOwner` の権限の組を `PermissionSet.all()`（管理者）にして返す。新しい所有者の `Member` は変えない。新しい所有者の多要素認証の設定を確かめない |
| `designateInitialOwner` | 所有者がいない方針だけを受け取り、所有者を指定する。`existing`（そのアカウントの、区画の `Member`）があれば、そのメンバーを所有者に指定し、`created` は偽。メンバーが保持する権限の組は変えない。`existing` が `null` なら、`memberId` と `accountId` で、空の権限の組を持つメンバーを `Member.create` で作って指定し、`created` は真。どちらも `AccessPolicy.designateOwner` を呼ぶ。`mfaRequired` は変えない |

- `designateInitialOwner` は、所有者の指定の一つの振る舞い。残っているメンバーに宛てた初期所有者の招待も、受諾できる
- 所有者がいる方針への所有者の指定は、型に無い。受諾は、`InvitationProcess.availability` が `AccessPolicy.isVacant` で絞った `VacantPolicy` を渡す
- `policy` と各 `Member` は、同じ区画から読んだもの

依存するポート: 無し。

### InvitationProcess

責務: 招待を発行し、招待を受諾できるかどうかを決め、招待の内容を示し、招待と引き換えにメンバーを作る。受諾の確定の時点の競合を、どの結果にするかを決める。

```ts
type InvitationIssueResult =
  | Readonly<{ kind: "issued"; invitation: PendingInvitation }>
  | Readonly<{
      kind: "superseded";
      invitation: PendingInvitation;
      superseded: RevokedInvitation;
      replaced: boolean;
    }>;

type InvitationTokenParams = Readonly<{ id: string; tokenDigest: InvitationTokenDigest }>;

type InvitationViewer = Readonly<{ email: EmailAddress; member: Member | null }>;

type InvitationAcceptor =
  | Readonly<{ kind: "new_account"; accountId: AccountId; joining: JoiningTicket }>
  | Readonly<{
      kind: "existing_account";
      accountId: AccountId;
      viewer: InvitationViewer;
      joining: JoiningTicket;
    }>;

type AcceptanceTarget =
  | Readonly<{ kind: "member"; permissions: PermissionSet }>
  | Readonly<{ kind: "initial_owner"; policy: VacantPolicy }>;

type InvitationAvailability =
  | Readonly<{ status: "invalid" }>
  | Readonly<{ status: "expired" | "accepted" | "revoked" }>
  | Readonly<{ status: "email_mismatch"; invitation: PendingInvitation }>
  | Readonly<{ status: "acceptable"; invitation: PendingInvitation; target: AcceptanceTarget }>;

type InvitationContent = Readonly<{
  workspaceName: WorkspaceName;
  email: EmailAddress;
  grant: InvitationGrant;
  expiresAt: Date;
}>;

type InvitationInspection =
  | (Readonly<{ status: "acceptable" }> & InvitationContent)
  | (Readonly<{ status: "email_mismatch" }> & InvitationContent)
  | Readonly<{ status: "expired" | "accepted" | "revoked" | "invalid" }>;

type InvitationAcceptance = Readonly<{
  invitation: AcceptedInvitation;
  member: Member;
  guard: JoiningGuard;
}>;

type InvitationAcceptResult =
  | (InvitationAcceptance & Readonly<{ kind: "member_joined" }>)
  | (InvitationAcceptance & Readonly<{ kind: "owner_joined"; policy: OwnedPolicy }>)
  | (InvitationAcceptance & Readonly<{ kind: "owner_designated"; policy: OwnedPolicy }>);

const InvitationProcess: {
  inviteMember(
    input: {
      workspace: ActiveWorkspace;
      email: string;
      permissions: PermissionSet;
      alreadyMember: boolean;
      pending: PendingInvitation | null;
      token: InvitationTokenParams;
    },
    now: Date,
  ): InvitationIssueResult;
  issueInitialOwner(
    input: {
      workspace: ActiveWorkspace;
      policy: AccessPolicy;
      email: string;
      pending: PendingInvitation | null;
      token: InvitationTokenParams;
    },
    now: Date,
  ): InvitationIssueResult;
  availability(
    input: {
      invitation: Invitation | null;
      directory: DirectoryStatus | null;
      workspace: Workspace | null;
      policy: AccessPolicy | null;
      viewer: InvitationViewer | null;
    },
    now: Date,
  ): InvitationAvailability;
  inspect(
    input: {
      invitation: Invitation | null;
      directory: WorkspaceDirectoryEntry | null;
      workspace: Workspace | null;
      policy: AccessPolicy | null;
      viewer: InvitationViewer | null;
    },
    now: Date,
  ): InvitationInspection;
  accept(
    input: {
      invitation: Invitation | null;
      directory: DirectoryStatus | null;
      workspace: Workspace | null;
      policy: AccessPolicy | null;
      guard: JoiningGuard | null;
      member: Member | null;
      acceptor: InvitationAcceptor;
      memberId: string;
    },
    now: Date,
  ): InvitationAcceptResult;
  rejectionOnConflict(input: {
    acceptor: InvitationAcceptor;
    guard: JoiningGuard | null;
    member: Member | null;
  }): "JoiningLapsed" | "InvitationInvalid";
};
```

| メソッド | 処理 |
| --- | --- |
| `inviteMember` | `alreadyMember`（宛先のアカウントが既にこのワークスペースのメンバーである）が真なら `BusinessRuleError`（`AlreadyMember`）。`grant` が `member` の招待を `Invitation.issue` で作る。`pending`（同じ宛先の、`grant` が `member` の `pending` の招待）が無ければ `issued`。あれば、期限によらず `Invitation.revoke` で失効させ、`superseded` で返す。新しい招待が前の招待を置き換える。`replaced` は、失効させた招待が、失効の時点で期限内（`Invitation.standing` が `usable`）だったとき真 |
| `issueInitialOwner` | `active` のワークスペースだけを受け取る。対象は、所有者がいない（`vacant`）ワークスペース（`AccessPolicy.isVacant` が真）。所有者がいるワークスペースなら `BusinessRuleError`（`InitialOwnerAlreadyJoined`）。`grant` が `initial_owner` の招待を作る。`pending`（このワークスペースの `pending` の初期所有者の招待）の扱いと、結果の `issued` / `superseded`、`replaced` は、`inviteMember` と同じ。宛先は、前の招待と違ってよく、既にメンバーである人でも、メンバーでない人でも、`pending` のメンバーの招待の宛先でもよい |
| `availability` | 招待を受諾できるかどうかを、下の表で決める。受諾できるかどうかの規則は、この表だけが持つ。`inspect` と `accept` は、この結果を写す |
| `inspect` | `availability` の結果を、招待の URL を開いた人に示す内容にする。`acceptable` と `email_mismatch` は、招待の内容（`InvitationContent`）を添える。`workspaceName` は、名簿の項目の名称。ほかの結果は、状態だけを返し、ワークスペースの名称と付与する内容を返さない。`availability` には、名簿の項目の状態を渡す。招待を変えない |
| `accept` | `availability` の結果を、下の表で、エラーまたは受諾の結果にする。`availability` に渡す `viewer` は、`acceptor` が `existing_account` ならその `viewer`、`new_account` なら `null` |
| `rejectionOnConflict` | 受諾の UnitOfWork が確定の時点の競合（`ConflictError`）で確定しなかったときに返すエラーの理由を決める。下の「確定の時点の競合」 |

`availability` の判定。上の行から順に確かめ、初めに当たった行の結果を返す。

| 条件 | 結果 |
| --- | --- |
| `invitation` が `null`、`policy` が `null`、または `WorkspaceAvailability.of(directory, workspace の status)` が偽 | `invalid` |
| `Invitation.standing` が `usable` でない | その状態（`expired` / `accepted` / `revoked`） |
| `viewer` があり、`viewer.email` が `invitation.email` と一致しない | `email_mismatch` |
| `grant` が `member` で、`viewer.member` がある | `invalid`。メンバーの招待が `pending` の間に、初期所有者の招待で参加した人が当たる。その招待は、期限まで `pending` のまま残り、未使用の招待の一覧に現れる |
| `grant` が `initial_owner` で、`AccessPolicy.isVacant` が偽 | `invalid`。`pending` の初期所有者の招待は、所有者がいない（`vacant`）ワークスペースにだけあるので、保存された状態では当たらない。型を `VacantPolicy` に絞る行 |
| それ以外 | `acceptable`。`target` は、`grant` が `member` なら、招待の権限の組。`initial_owner` なら、`VacantPolicy` |

- `viewer` は、招待を示した人のアカウント。`email` はそのメールアドレス、`member` は、そのアカウントの、その区画の `Member`（メンバーでなければ `null`）。ログインしていない人と、アカウントを作って参加する人は `null`
- 宛先と異なるアカウントは、そのワークスペースのメンバーでも `email_mismatch`
- `invitation`、`workspace`、`policy`、`viewer.member` は、招待の値から求めた一つの区画から読んだもの。`directory` は、名簿のそのワークスペースの状態で、名簿に無ければ `null`
- 招待の値からワークスペースの ID を取り出せない要求（`SecretTokenIssuer.workspaceOf` が `null`）、取り出した区画に照合値の招待が無い要求、保持期間を過ぎた招待（[common.md](common.md) の「期限つきの一度きりの値」の保持期間）の要求は、`invitation` に `null` を渡す
- 招待は、利用できるワークスペースだけで成り立つ（[index.md](index.md) の「利用できるワークスペース」）。初期所有者の招待は、作成が名簿を開設済みにした後に発行する

`accept` の結果。

| `availability` の結果 | `accept` |
| --- | --- |
| `invalid` | `BusinessRuleError`（`InvitationInvalid`） |
| `expired` | `BusinessRuleError`（`InvitationExpired`） |
| `accepted`、`revoked` | `BusinessRuleError`（`InvitationInvalid`）。使用済み・失効済みの理由を示すのは `inspect` だけ |
| `email_mismatch` | `BusinessRuleError`（`InvitationEmailMismatch`） |
| `acceptable`。下の「事前の確認の後の変化」の表で理由が決まる | その理由の `BusinessRuleError`。招待を使用済みにせず、メンバーを作らない |
| `acceptable`。上の行に当たらず、`target` が `member` | `Invitation.accept` を呼び、`acceptor.accountId` と `memberId` で、招待の権限の組を持つメンバーを `Member.create` で作る。結果は `member_joined` |
| `acceptable`。上の行に当たらず、`target` が `initial_owner` | `Invitation.accept` を呼び、`target.policy` と、`existing` に `member` を渡して `MemberAdministration.designateInitialOwner` を呼ぶ。`created` が真なら `owner_joined`、偽なら `owner_designated`。結果の `member` は所有者に指定したメンバー、`policy` は、所有者を指定した方針 |

- 受諾できるかどうか（`availability`）は、セッションと入力の確認より先に確かめる。`acceptable` でない招待の受諾は、セッションの有無と入力によらず、上の表のエラーになる
- `acceptor` は、受諾する人。`new_account` は、宛先のメールアドレスでアカウントを作って参加する人で、`accountId` は作ったアカウント。`existing_account` は、セッションのアカウントで受諾する人で、`viewer` はそのメールアドレスと、事前の確認が読んだ、その区画の `Member`
- `joining` は、事前の確認が、前段より前に読んだ参加の囲いの世代（`JoiningGuard.ticket`）。どの受諾も前段を行う
- `member` は、後段が読み直した、`acceptor.accountId` の、その区画の `Member`。無ければ `null`
- `guard` は、後段が読んだ、そのアカウントの参加の囲い。受諾の結果は、`guard` に `JoiningGuard.advance` を適用した囲いを持ち、後段は、招待とともに保存する
- `JoiningLapsed` は、その参加が取り下げられたこと（`JoiningGuard.withdrawn`）を表す。やり直しは「取り下げ」に従う
- `BusinessRuleError` では、招待、メンバー、ワークスペースは変わらない
- 招待の値で指す要求は、該当なしを `InvitationInvalid` の一つで表す

事前の確認の後の変化。事前の確認の後に、受諾する人の `Member` と参加の囲いが変わった受諾が、確定しない理由を決める。上の行から順に確かめ、初めに当たった行の理由にする。`accept` と `rejectionOnConflict` が使う。

| `grant`、`acceptor`、`guard`、`member` | 理由 |
| --- | --- |
| `acceptor` が `existing_account` で、`viewer.member` があり、`member` が `null` | `InvitationInvalid`。事前の確認の後に、その人の利用が解除された |
| `JoiningGuard.withdrawn(acceptor.joining, guard, member)` が真 | `JoiningLapsed`。その参加が取り下げられた |
| `grant` が `member` で、`member` がある | `InvitationInvalid`。事前の確認の後に、その人が別の受諾でメンバーになった |
| それ以外 | 決まらない |

確定の時点の競合。招待の受諾は、[index.md](index.md) の「競合の扱い」の表の、値の引き換えに当たる。参加の前段と後段の UnitOfWork が `ConflictError` で確定しなかった受諾の結果は、`rejectionOnConflict` だけが定める。

- 「事前の確認の後の変化」の表で理由が決まれば、その理由。決まらなければ `InvitationInvalid`。`ConflictError` の理由によらない（`WORKSPACE_NOT_ACTIVE`、`OWNER_NOT_MEMBER`、並行した取り消し、置き換え、同じアカウントの別の受諾に負けた場合を含む）
- ユースケースは、競合の後に、そのアカウントの囲いと `Member` を読み直して `guard` と `member` に渡し、結果の理由の `BusinessRuleError` を返す。確定をやり直さない
- 確かめ直しの UnitOfWork が確定しなかった受諾は、後段の確定で成立している。結果は、確定しなかった理由によらず、エラーにせず、成立した受諾の結果にする。セッションを始めないので、アカウントを作った受諾は、ログインが要ることを返す。残った `joining` の項目は、`MembershipRepair.decide` が `joined` にする

依存するポート: 無し。

### エラーコード

Workspace が返す理由のコードを、すべて列挙する。共通のコード（`STALE_VERSION`、`ALREADY_EXISTS`、`SECRET_DIGEST_TAKEN`、`WORKSPACE_NOT_ACTIVE`、`PERMISSION_MISSING`、`OWNER_ONLY`、`ACTOR_NOT_ALLOWED`、`MFA_SETUP_REQUIRED`、`INVALID_QUERY`）は [common.md](common.md) にある。`WORKSPACE_NOT_ACTIVE` の意味は、[common.md](common.md) の「WriteScope」が定める。`MEMBERSHIP_ACCOUNT_MISSING` は、所属の索引（`MembershipIndex`）が返す。

```ts
type WorkspaceErrorCode =
  // ValidationError
  | "InvalidWorkspaceName"
  | "UnknownPermission"
  // BusinessRuleError
  | "InvalidStoredValue"
  | "AlreadyOwner"
  | "OwnerMismatch"
  | "OwnerMfaNotConfigured"
  | "WorkspaceNameMismatch"
  | "OwnerPermissionsDerived"
  | "OwnerCannotBeRemoved"
  | "AlreadyMember"
  | "InitialOwnerAlreadyJoined"
  | "InvitationInvalid"
  | "InvitationExpired"
  | "InvitationAccepted"
  | "InvitationEmailMismatch"
  | "JoiningLapsed"
  // ConflictError
  | "WORKSPACE_CLOSING"
  | "OWNER_NOT_MEMBER"
  | "WORKSPACE_NOT_EMPTY"
  | "ALREADY_MEMBER"
  | "MEMBER_IS_OWNER"
  | "PENDING_INVITATION_EXISTS"
  | "MEMBERSHIP_ACCOUNT_MISSING"
  // DeclarationIssue の理由
  | "WorkspaceNameDuplicated";
```

| 種類 | 理由 | 意味 |
| --- | --- | --- |
| `ValidationError` | `InvalidWorkspaceName` / `UnknownPermission` | 利用者の入力から値オブジェクトを作れない。招待の宛先の違反は、共通の `InvalidEmailAddress`（[common.md](common.md)） |
| `DeclarationIssue.code` | `WorkspaceNameDuplicated` | 初期構成で、ワークスペースの名称が重なる。エラーとして投げない |
| `BusinessRuleError` | `InvalidStoredValue` | 利用者の入力に由来しない値（照合値）が空 |
| `BusinessRuleError` | `AlreadyOwner` | 新しい所有者が、現在の所有者 |
| `BusinessRuleError` | `OwnerMismatch` | 所有者の変更に渡した元の所有者のメンバーが、所有者の指定と一致しない |
| `BusinessRuleError` | `OwnerMfaNotConfigured` | 所有者が多要素認証を設定していない状態での必須化 |
| `BusinessRuleError` | `WorkspaceNameMismatch` | 利用終了の確認の名称が一致しない |
| `BusinessRuleError` | `OwnerPermissionsDerived` / `OwnerCannotBeRemoved` | 所有者の権限の変更、所有者の利用の解除 |
| `BusinessRuleError` | `AlreadyMember` / `InitialOwnerAlreadyJoined` | 招待を発行できない。`AlreadyMember` は、宛先が既にメンバーであるメンバーの招待。`InitialOwnerAlreadyJoined` は、所有者がいるワークスペースへの初期所有者の招待。`InvitationProcess.issueInitialOwner` だけが返す |
| `BusinessRuleError` | `InvitationInvalid` / `InvitationExpired` / `InvitationEmailMismatch` | 招待を受諾できない。`InvitationProcess.accept` と `InvitationProcess.rejectionOnConflict` だけが返す。`InvitationInvalid` は、示された値で受諾できる招待が無いことを表す |
| `BusinessRuleError` | `JoiningLapsed` | その参加が取り下げられた（`JoiningGuard.withdrawn`）。`InvitationProcess.accept` と `InvitationProcess.rejectionOnConflict` が返す |
| `BusinessRuleError` | `InvitationAccepted` | 参加済みの招待を取り消せない。`Invitation.revoke` だけが返す |
| `ConflictError` | `WORKSPACE_CLOSING` | 利用終了が先に進んだワークスペース（利用終了を完了した ID を含む）への、ワークスペースの作成と、初期所有者の招待の発行。条件は「初期構成の宣言」の発行し直しと、「作成と利用終了の並行」が定める。運用コマンドの二つだけが返す |
| `ConflictError` | `OWNER_NOT_MEMBER` / `MEMBER_IS_OWNER` | 所有者がメンバーでなくなる書き込み |
| `ConflictError` | `WORKSPACE_NOT_EMPTY` | アクセスの方針、メンバー、招待、参加の囲いのいずれかが残るワークスペースを `closed` にする保存 |
| `ConflictError` | `ALREADY_MEMBER` | 同じワークスペースに同じアカウントのメンバーがいる |
| `ConflictError` | `PENDING_INVITATION_EXISTS` | 同じ種類の `pending` の招待がある。同じ宛先の `pending` のメンバーの招待、または `pending` の初期所有者の招待 |
| `ConflictError` | `MEMBERSHIP_ACCOUNT_MISSING` | 所属の索引の項目のアカウントが、確定の時点で存在しない |

## ポート

`MemberRepository` は `TransactionalRepository` を、`InvitationRepository` と `JoiningGuardRepository` は、一件ずつ削除する呼び手が無いので `Omit<TransactionalRepository<TEntity, TId>, "delete">` を、`WorkspaceRepository` と `AccessPolicyRepository` は `SingletonRepository` を継ぐ。どのポートも、[common.md](common.md) の「TransactionalRepository」の、すべてのリポジトリのメソッドに適用する規則に従う。`InvitationRepository` は、加えて「期限つきの一度きりの値」に従う。下の表は、これに加える契約だけを書く。

`WorkspaceRepository`、`AccessPolicyRepository`、`MemberRepository`、`InvitationRepository`、`JoiningGuardRepository` は、[common.md](common.md) の「TransactionalRepository」のワークスペースの区画のリポジトリ。ポートが担保する一意性と参照整合性は、その区画の中に限る。ワークスペースをまたぐ読み取り（名称での検索、設置のワークスペースの一覧、アカウントが所属するワークスペース）を持たない。それらは、設置の区画の `WorkspaceDirectory` と `MembershipIndex` から求める。二つは設置の区画にあり、対象のワークスペースを引数で受ける。

### WorkspaceRepository

目的: 区画のワークスペースを保存する。区画に `Workspace` は一つ。

```ts
interface WorkspaceRepository extends SingletonRepository<Workspace> {}
```

メソッドは `SingletonRepository` の三つ（`insert`、`find`、`save`）。利用終了の区画の消去は、`closed` の `Workspace` を `save` して終わり、`Workspace` は区画に残る。

| メソッド | 契約 |
| --- | --- |
| `insert` | 区画の `Workspace` を作る。区画に `Workspace` が既にあれば、その状態（`active` / `closing` / `closed`）によらず、`SingletonRepository` の `ConflictError`（理由 `ALREADY_EXISTS`）。並行した二つの `insert`（作成の区画の段と、`Workspace` が無い区画への利用終了の開始）は、一方だけが確定する。ID と名称を持たない。呼び出し側が、名簿（`WorkspaceDirectory.register`）に登録した ID の区画で `insert` する |
| `save` | 楽観ロックを必要とする。`closed` のワークスペースの保存は、確定の時点でアクセスの方針、メンバー、招待、参加の囲いのいずれかが区画に残っていれば `ConflictError`（理由 `WORKSPACE_NOT_EMPTY`）。同じ UnitOfWork で残りを削除する書き込みは成功する。確定の時点で残っていなければよい |
| `find` | 状態によらず返す。利用終了の区画の消去を終えた区画では、`closed` のワークスペースを返す。区画に `Workspace` が無ければ `null`（作成の前） |

### AccessPolicyRepository

目的: 区画のアクセスの方針を保存する。区画に一つ。

```ts
interface AccessPolicyRepository extends SingletonRepository<AccessPolicy> {
  deleteAll(limit: number): Promise<number>;
}
```

メソッドの分類: `deleteAll` は書き込み。

| メソッド | 契約 |
| --- | --- |
| `insert` | 区画に方針が既にあれば `ConflictError`（理由 `ALREADY_EXISTS`）。所有者の指定が `designated` の方針は、`save` と同じく、所有者が区画のメンバーであることを確定の時点で検査する（理由 `OWNER_NOT_MEMBER`） |
| `save` | 楽観ロックを必要とする。所有者の指定が `designated` の方針の `owner.memberId` が、確定の時点で区画のメンバーを指すことを、ポートが担保する。指さなければ `ConflictError`（理由 `OWNER_NOT_MEMBER`）。判定は、その区画に保存されたメンバーの ID で行う。所有者の指定が `vacant` の方針は、この検査の対象でない。同じ UnitOfWork で `insert` したメンバーを所有者に指定する書き込みは成功する。同じ確定が `OWNER_NOT_MEMBER` と `MemberRepository.delete` の `MEMBER_IS_OWNER` の両方に当たる場合は、どちらかの理由の `ConflictError` を返す。どちらの理由になるかは約束しない |
| `find` | 区画の方針を返す。無ければ `null` |
| `deleteAll` | 区画の消去（[common.md](common.md)）。対象は、区画の方針。楽観ロックを使わない |

### MemberRepository

目的: メンバーを保存し、ID とアカウントで引く。

```ts
interface MemberRepository extends TransactionalRepository<Member, MemberId> {
  findByAccount(accountId: AccountId): Promise<Versioned<Member> | null>;
  findByIds(ids: readonly MemberId[]): Promise<readonly Member[]>;
  findPage(pagination: Pagination): Promise<PaginationResult<Member>>;
  deleteAll(limit: number): Promise<number>;
}
```

メソッドの分類: `findByAccount`、`findByIds`、`findPage` は読み取り。`deleteAll` は書き込み。

| メソッド | 契約 |
| --- | --- |
| `insert` | 区画の中でアカウントが一意であることを、ポートが担保する。重複は `ConflictError`（理由 `ALREADY_MEMBER`）。並行した `insert` でも一方だけが成功する。受諾は、区画にそのアカウントの `Member` があれば、メンバーを作らない（`InvitationProcess.accept`）ので、この違反は並行した受諾でだけ生じる。アカウントの存在を、ポートは確かめない。アカウントは設置の区画にあり、区画をまたぐ参照整合性は、呼び出し側が順序で担保する。呼び出し側は、設置の区画でアカウントと所属の索引の項目を確定してから、`insert` する（[index.md](index.md) の「区画をまたぐ手順」の参加） |
| `save` | 楽観ロックを必要とする。同じメンバーへの並行した権限の変更は、一方だけが成功する |
| `delete` | 楽観ロックを必要とする。区画のアクセスの方針が所有者に指定するメンバーの削除を、ポートが拒否する（`ConflictError`、理由 `MEMBER_IS_OWNER`）。確定の時点の指定で判定し、並行した所有者の変更に対しても成り立つ。同じ UnitOfWork で、所有者の指定を `vacant` にした方針（`AccessPolicy.clearOwner` の結果）を保存する書き込みは成功する。所有者がいない（`vacant`）方針と、方針が無い区画では、どのメンバーも削除できる |
| `findById` | 区画のメンバーを返す。ほかのワークスペースのメンバーの ID には `null` |
| `findByAccount` | 区画の、そのアカウントのメンバーを返す。無ければ `null` |
| `findByIds` | 渡した ID のうち、区画に存在するメンバーを、ID の昇順で返す。ほかのワークスペースのメンバーは返さない。渡せる ID の上限は 100 件。空の集合には空の列を返す |
| `findPage` | 区画のメンバーを、作成日時の早い順で返す。`count` は、区画のメンバーの総件数 |
| `deleteAll` | 区画の消去（[common.md](common.md)）。対象は、区画のすべてのメンバー。所有者に指定されたメンバーを含む。楽観ロックを使わない。`Workspace` が `closed` の区画では、0 を返す |

### InvitationRepository

目的: 招待を保存し、招待の値の照合値と宛先で引く。

```ts
interface InvitationRepository
  extends Omit<TransactionalRepository<Invitation, InvitationId>, "delete"> {
  findByTokenDigest(digest: InvitationTokenDigest): Promise<Versioned<Invitation> | null>;
  findPendingMemberByEmail(email: EmailAddress): Promise<Versioned<PendingInvitation> | null>;
  findPendingInitialOwner(): Promise<Versioned<PendingInvitation> | null>;
  findPendingPage(pagination: Pagination): Promise<PaginationResult<PendingInvitation>>;
  deleteAll(limit: number): Promise<number>;
  deleteLapsed(before: Date, limit: number): Promise<number>;
}
```

メソッドの分類: `findByTokenDigest`、`findPendingMemberByEmail`、`findPendingInitialOwner`、`findPendingPage` は読み取り。`deleteAll` と `deleteLapsed` は書き込み。

| メソッド | 契約 |
| --- | --- |
| `insert` | 「Invitation」の不変条件の、種類ごとの `pending` の招待の一意性を、ポートが担保する。確定の時点で違反する書き込みは `ConflictError`（理由 `PENDING_INVITATION_EXISTS`）。同じ宛先の、`pending` のメンバーの招待と `pending` の初期所有者の招待は、どちらの順の `insert` も成功する。期限を過ぎた `pending` も `pending` として数える。同じ UnitOfWork で、同じ種類の前の `pending` を `revoked` に `save` し、新しい `pending` を `insert` する書き込みは成功する |
| `save` | 楽観ロックを必要とする。同じ招待への並行した受諾と取り消しは、一方だけが成功する |
| `findById` | 区画の招待を返す。ほかのワークスペースの招待の ID には `null` |
| `findByTokenDigest` | 区画の招待を、照合値の一致で引く。状態によらず返す。無ければ `null`。呼び出し側は、招待の値から `SecretTokenIssuer.workspaceOf` で求めたワークスペースの区画のリポジトリで呼ぶ |
| `findPendingMemberByEmail` | 区画の、その宛先の、`grant` が `member` の `pending` の招待を返す。期限を過ぎたものも返す。初期所有者の招待は返さない。無ければ `null` |
| `findPendingInitialOwner` | 区画の、`pending` の初期所有者の招待を、宛先によらず返す。期限を過ぎたものも返す。メンバーの招待は返さない。無ければ `null` |
| `findPendingPage` | 区画の、`grant` が `member` の `pending` の招待を、作成日時の新しい順で返す。期限を過ぎたものも含む。初期所有者の招待は含まない。`count` は、該当する総件数 |
| `deleteAll` | 区画の消去（[common.md](common.md)）。対象は、区画のすべての招待。状態によらない。楽観ロックを使わない。`Workspace` が `closed` の区画では、0 を返す |
| `deleteLapsed` | 一度きりの値の削除（[common.md](common.md) の「期限つきの一度きりの値」）。`lapsedAt` は `Invitation.lapsedAt`。区画が無いワークスペースでは 0 を返す。参加済みのメンバーは変えない。定期ジョブ「期限切れの記録の掃除」が使う |

### JoiningGuardRepository

目的: 参加の囲いを保存し、アカウントで引く。

```ts
interface JoiningGuardRepository
  extends Omit<TransactionalRepository<JoiningGuard, AccountId>, "delete"> {
  deleteAll(limit: number): Promise<number>;
}
```

メソッドの分類: `deleteAll` は書き込み。

| メソッド | 契約 |
| --- | --- |
| `insert` | 区画に同じアカウントの囲いがあれば `ConflictError`（理由 `ALREADY_EXISTS`）。並行した二つの `insert` は、一方だけが確定する |
| `findById` | 区画の、そのアカウントの囲いを返す。無ければ `null` |
| `save` | 楽観ロックを必要とする。同じ囲いへの並行した `advance` の保存（参加の後段と取り下げ、二つの後段）は、一方だけが成功する |
| `deleteAll` | 区画の消去（[common.md](common.md)）。対象は、区画のすべての囲い。楽観ロックを使わない |

- アカウントの存在を、ポートは確かめない

### WorkspaceDirectory

目的: ワークスペースの名簿を保存する。設置のワークスペースごとに、ID、名称、名簿の状態を持つ。設置をまたぐ処理（所属するワークスペースの一覧、定期ジョブ、状態確認、ワークスペースの作成と利用終了）が、対象のワークスペースを求めるために使う。設置の区画にあり、`InstallationContext` と `PartitionReader.installation()` が `workspaceDirectory` として公開する。集約を保存しないので、`TransactionalRepository` を継がず、楽観ロックを使わない。持つメソッドは、下のインターフェースがすべて宣言する。置き場所は `packages/core/src/domain/workspace/ports/`。

```ts
type DirectoryRegistration = Readonly<{
  id: WorkspaceId;
  name: WorkspaceName;
  initialOwner: EmailAddress | null;
}>;

interface WorkspaceDirectory {
  register(entries: readonly DirectoryRegistration[]): Promise<void>;
  changeStatus(id: WorkspaceId, change: DirectoryStatusChange): Promise<void>;
  finishCreation(id: WorkspaceId): Promise<void>;
  remove(id: WorkspaceId): Promise<void>;
  findById(id: WorkspaceId): Promise<WorkspaceDirectoryEntry | null>;
  findByIds(ids: readonly WorkspaceId[]): Promise<readonly WorkspaceDirectoryEntry[]>;
  findByName(name: WorkspaceName): Promise<readonly WorkspaceDirectoryEntry[]>;
  findPageByStatus(
    statuses: readonly DirectoryStatus[],
    pagination: Pagination,
  ): Promise<PaginationResult<WorkspaceDirectoryEntry>>;
}
```

メソッドの分類: `register`、`changeStatus`、`finishCreation`、`remove` は書き込み。`findById`、`findByIds`、`findByName`、`findPageByStatus` は読み取り。

| メソッド | 契約 |
| --- | --- |
| `register` | 渡した項目を、準備中（`provisioning`）で登録する。`initialOwner` があれば、項目の `creation` を、その `initialOwner` と `finished` が偽にする。無ければ `creation` は `null`。一回の呼び出しで複数の項目を登録する。名称は、ほかの項目と重なってよい。名簿に、同じ ID で、名称または初期所有者（`creation` の `initialOwner`。`creation` が `null` なら無し）の違う項目があれば `ConflictError`（理由 `ALREADY_EXISTS`）。同じ ID で、名称と初期所有者が同じ項目が既にある場合は、その項目を変えずに通過する（状態を `provisioning` に戻さず、`finished` を変えない）。一件でも違反があれば、どの項目も登録しない。同じ ID で名称または初期所有者の違う並行した `register` は、一方だけが確定する。渡せる件数の上限は 1,000 件。空の列では何もしない |
| `changeStatus` | その ID の項目の状態を置き換える。同じ状態への変更と、存在しない ID への呼び出しは、何も変えずに通過する。`retiring` の項目を `open` にする呼び出しは、何も変えずに通過する。`retiring` の項目は、`remove` まで `retiring` のまま。並行した `open` と `retiring` への変更は、確定の順によらず `retiring` になる |
| `finishCreation` | 準備中でない、その ID の項目の `creation.finished` を真にする。状態は変えない。準備中の項目、`creation` が `null` の項目、既に真の項目、存在しない ID では、何も変えずに通過する |
| `remove` | その ID の項目を、状態によらず、名簿から外す。存在しない ID では、何も変えずに通過する。区画のデータは、ポートの検査の対象でない。呼び出し側が「利用終了の消去」の条件を確かめる |
| `findById` | その ID の項目を、状態によらず返す。無ければ `null` |
| `findByIds` | 渡した ID のうち、名簿にある項目を、状態によらず、ID の昇順で返す。渡せる ID の上限は 100 件。空の集合には空の列を返す |
| `findByName` | その名称の項目を、`WorkspaceName` の一致で引き、状態によらず、ID の昇順で返す。無ければ空の列 |
| `findPageByStatus` | 状態が `statuses` のいずれかの項目を、ID の昇順で返す。`count` は該当する総件数 |

- 書き込み（`register`、`changeStatus`、`finishCreation`、`remove`）は、設置の区画の UnitOfWork の中で呼び、同じ UnitOfWork のほかの書き込み（`Installation`、アカウント、所属の索引）と、全部成功か全部反映されないかを共にする
- 一意性・整合性: ID の一意性を、ポートが担保する。区画の `Workspace` の存在と、名簿の状態が `Workspace` の状態に合うことは、別の区画にあり、呼び出し側が「区画をまたぐ手順」の順序で担保する
- 並行性: 楽観ロックを使わない。`register` の一意性は、並行した呼び出しでも成り立つ
- 可視性: 確定した書き込みは、以後のすべての読み取りに反映する
- エラーケース: 範囲を外れた件数の指定は `SystemError`（理由 `INVALID_QUERY`）

### MembershipIndex

目的: 所属の索引を保存する。アカウントとワークスペースの組ごとに、項目の状態（`joining` / `joined`）と世代を持ち、アカウントごとに、最後に利用したワークスペースを持つ。ワークスペースに属さない操作（所属するワークスペースの一覧、ログインの後に開くワークスペース）、最後に利用したワークスペースの記録、所属の索引の修復、利用終了、利用再開前の処理が、アカウントとワークスペースの対応を求めるために使う。設置の区画にあり、`InstallationContext` と `PartitionReader.installation()` が `membershipIndex` として公開する。集約を保存しないので、`TransactionalRepository` を継がず、楽観ロックを使わない。持つメソッドは、下のインターフェースがすべて宣言する。置き場所は `packages/core/src/domain/workspace/ports/`。

```ts
type MembershipPage = Readonly<{
  items: readonly Membership[];
  next: AccountId | null;
}>;

type MembershipScanPage = Readonly<{
  items: readonly Membership[];
  next: MembershipKey | null;
}>;

interface MembershipIndex {
  beginJoining(keys: readonly MembershipKey[], since: Date): Promise<void>;
  confirm(keys: readonly MembershipKey[]): Promise<void>;
  remove(memberships: readonly Membership[]): Promise<void>;
  markLastUsed(key: MembershipKey): Promise<void>;
  removeWorkspace(workspaceId: WorkspaceId): Promise<void>;
  find(key: MembershipKey): Promise<Membership | null>;
  findByAccount(accountId: AccountId): Promise<readonly Membership[]>;
  findLastUsed(accountId: AccountId): Promise<WorkspaceId | null>;
  findPageByWorkspace(
    workspaceId: WorkspaceId,
    page: Readonly<{ after: AccountId | null; limit: number }>,
  ): Promise<MembershipPage>;
  scan(page: Readonly<{ after: MembershipKey | null; limit: number }>): Promise<MembershipScanPage>;
}
```

メソッドの分類: `beginJoining`、`confirm`、`remove`、`markLastUsed`、`removeWorkspace` は書き込み。`find`、`findByAccount`、`findLastUsed`、`findPageByWorkspace`、`scan` は読み取り。

| メソッド | 契約 |
| --- | --- |
| `beginJoining` | 参加の開始。渡した組ごとに、新しい世代を割り当てた `joining`（`since` は渡した日時）の項目を置く。その組の項目が `joining` または `joined` なら、新しい世代の `joining` の項目に置き換える。割り当てる世代は、その設置の索引で、それまでに割り当てたどの世代とも違う値（外した項目の世代を含む）。一回の呼び出しで複数の組を扱う。渡せる組の上限は 1,000 件。同じ組の重複は一つとして扱う。空の列では何もしない |
| `confirm` | 参加の確定。渡した組ごとに、`joining` の項目を、世代を引き継いだ `joined` にする。項目が無ければ、新しい世代を割り当てた `joined` の項目を置く。`joined` の項目は、世代を含めて変えない。渡せる組の上限、同じ組の重複、空の列の扱いは `beginJoining` と同じ |
| `remove` | 除外。渡した項目ごとに、索引のその組の項目が、渡した項目と同じ世代の場合だけ、外す。比べるのは世代だけ。渡した後に `confirm` が `joined` にした項目は、同じ世代なので外す。項目が無い組と、世代が違う組は、何も変えずに通過する。呼び出し側は、読んだ項目（`find`、`findByAccount`、`findPageByWorkspace`、`scan` の結果）を渡す。渡せる項目の上限、同じ組の重複、空の列の扱いは `beginJoining` と同じ |
| `markLastUsed` | そのアカウントの、最後に利用したワークスペースを、渡した組のワークスペースにする。その組の項目の有無と状態によらない。既にそのワークスペースなら、何も変わらない |
| `removeWorkspace` | そのワークスペースのすべての項目を、状態と世代によらず外す。該当する項目が無ければ、何も変えずに通過する。件数の上限を持たない一括のメソッド（[common.md](common.md) の「UnitOfWorkProvider」の「書き込みの量」）。項目の件数によらず、一回の呼び出しで確定する。名簿からの除外（`WorkspaceDirectory.remove`）と同じ UnitOfWork で確定し、利用終了の完了を一つの確定にするので、件数で区切らない |
| `find` | その組の項目を返す。無ければ `null` |
| `findByAccount` | そのアカウントのすべての項目を、状態によらず、`workspaceId` の昇順で返す。件数を区切らない。無ければ空の列 |
| `findLastUsed` | そのアカウントの、最後に `markLastUsed` に渡したワークスペースの ID を返す。無ければ `null`。項目の書き込み（`beginJoining`、`confirm`、`remove`、`removeWorkspace`）は、この値を変えない |
| `findPageByWorkspace` | そのワークスペースの項目を、状態によらず、`accountId` の昇順で返す。`after` より後ろの `accountId` から `limit` 件まで。`next` は、続きがあれば最後に返した項目の `accountId`、無ければ `null`。`limit` の上限は 100。読み進める間に項目が変わっても、`after` より後ろの項目を飛ばさない |
| `scan` | 索引のすべての項目を、状態とワークスペースによらず、`accountId` の昇順、同じアカウントは `workspaceId` の昇順で返す。`after` より後ろの組から `limit` 件まで。`next` は、続きがあれば最後に返した項目の組、無ければ `null`。`limit` の上限は 100。読み進める間に項目が変わっても、`after` より後ろの項目を飛ばさない。定期ジョブ「所属の索引の修復」が使う |

- 書き込みは、設置の区画の UnitOfWork の中で呼び、同じ UnitOfWork のほかの書き込み（アカウント、名簿、セッション）と、全部成功か全部反映されないかを共にする
- 一意性・整合性: 一つの組の項目が最大一つであること、割り当てる世代が、その設置の索引で一意であること、一つのアカウントの最後に利用したワークスペースが最大一つであることを、ポートが担保する。`confirm`、`remove`、`markLastUsed`、`removeWorkspace` は、もう一度呼んでも結果が変わらない。`beginJoining` は、呼ぶたびに新しい世代を割り当てる。どの書き込みも、既にある項目を理由にしたエラーを返さない
- 参照整合性: 項目のアカウントが、設置の区画に存在することを、ポートが、UnitOfWork の確定の時点で担保する。`beginJoining` と `confirm` が、存在しないアカウントの項目を置く確定は、`ConflictError`（理由 `MEMBERSHIP_ACCOUNT_MISSING`）。同じ UnitOfWork で `insert` したアカウントの項目は置ける。ワークスペースの存在（名簿の項目、区画の `Workspace` と `Member`）は、ポートの検査の対象でない。名簿に無いワークスペースの項目と、`Member` に合わない項目は、`MembershipRepair`、利用終了、「利用再開前の処理」が直す
- 並行性: 楽観ロックを使わない。同じ組への並行した書き込みは、どちらも確定し、確定の順に適用した結果になる。`remove` は、確定の時点の項目の世代を、渡した項目の世代と比べる。呼び出し側が読んだ後に確定した `beginJoining` の項目は、世代が違うので外さない。同じアカウントへの並行した `markLastUsed` は、どちらも確定し、値は、後に確定した側のワークスペースになる
- 可視性: 確定した書き込みは、以後のすべての読み取りに反映する
- 索引の書き込みは `Account` を書かない。`Account` の `version` との関係は [identity.md](identity.md) の「Account」の不変条件に従う
- エラーケース: 範囲を外れた件数の指定は `SystemError`（理由 `INVALID_QUERY`）

## トランザクション境界

共通の `UnitOfWorkProvider.run(scope, fn)` を使う（[common.md](common.md) の「WriteScope」「UnitOfWorkProvider」）。`WorkspaceContext` が `workspaceRepository`、`accessPolicyRepository`、`memberRepository`、`invitationRepository`、`joiningGuardRepository` を、`InstallationContext` が `workspaceDirectory` と `membershipIndex` を公開する。一つの UnitOfWork は、一つの区画だけを書く。UnitOfWork ごとの `WriteScope`、確定の単位、書き込みの回数は、各ユースケース（[../usecases/workspace.md](../usecases/workspace.md)）と、修復の手続きと、Installation に公開する手順の節（「取り下げ」「初期構成の宣言」「利用再開前の処理」「利用終了の消去」「作成と利用終了の並行」）が定める。この節は、整合性を担保する場所だけを定める。

- 区画の中の整合性は、ポートが、確定の時点で担保する。所有者がメンバーであること、所有者であるメンバーを削除しないこと、メンバーの一意性、招待の種類ごとの一意性、参加の囲いの一意性が当たる。ドメインサービスの事前の確認は、利用者に理由を示すために行う
- 区画をまたぐ整合性は、手順の順序が担保する。メンバーのアカウントの存在、名簿と `Workspace` の対応、索引と `Member` の対応が当たる。順は [index.md](index.md) の「区画をまたぐ手順」が、段の中身は、その「手順ごとの規則」の表の定義元が定める
- 索引の項目のアカウントの存在は、同じ設置の区画にあるので、`MembershipIndex` が担保する
- ワークスペースの区画の UnitOfWork の条件の意味は、[common.md](common.md) の「WriteScope」が定める。条件が見る `Workspace` の状態は「利用中の条件」

## 初期構成の宣言

初期構成の読み込みとワークスペースの作成で、Workspace が受ける宣言と、その検証・生成の中身。宣言の形の全体と、`path` に渡す位置は [installation.md](installation.md) が、区画をまたぐ段の順は [index.md](index.md) の「区画をまたぐ手順」が、ドメインを呼ぶ順は「手順を呼ぶ順」が定める。この節は、順を持たず、index.md の段の名前ごとに、Workspace が行う中身を書く。`DeclarationIssue` と検証の性質は [common.md](common.md) の「DeclarationIssue」が定める。

```ts
type WorkspaceDeclaration = Readonly<{ name: string }>;

const WorkspaceDeclaration: {
  validate(
    entries: readonly Readonly<{ path: string; declaration: WorkspaceDeclaration }>[],
  ): readonly DeclarationIssue[];
};
```

宣言は、ワークスペースの名称。`entries` は、宣言のすべてのワークスペース。

検証。

| 規則 | `path` | `code` |
| --- | --- | --- |
| `WorkspaceName.create` | 渡された `path` に `.name` を続ける | 投げた `ValidationError` の理由 |
| 名称（`WorkspaceName` の等価性）が、宣言の中で重ならない | 重なる二件目以降の `path` に `.name` を続ける | `WorkspaceNameDuplicated` |

- `WorkspaceNameDuplicated` は、この検証だけが返す理由。エラーとして投げない
- 宣言の照合値から除く値: 無い
- 前提: 先に終えているべきドメインは Identity。所有者の索引の項目は、Identity の生成が作る所有者のアカウントを指し、`MembershipIndex` が、項目のアカウントの存在を担保する。Workspace の生成は、区画と `Workspace` を作るので、区画に集約を作るほかのドメインより先に行う

初期構成の読み込みの中身。index.md の「区画をまたぐ手順」の「初期構成の読み込み」の行の段ごとに、次を行う。

- 設置の区画の最初の段: 宣言したワークスペースごとに `IdGenerator.generate` で ID を決め、すべてのワークスペースを `WorkspaceDirectory.register` の一回の呼び出しで、準備中で登録する（`initialOwner` は `null`）。宣言した所有者ごとに、Identity の生成が返した `AccountId` とワークスペースの ID の組を、`MembershipIndex.beginJoining` の一回の呼び出しで、`joining` の項目にする。`since` は、起動の初めに求めた現在時刻
- ワークスペースごとの区画の段（条件 `none`）: `Workspace.create` と `AccessPolicy.create` を呼ぶ。所有者の `AccountId` と、`IdGenerator.generate` の ID で、`MemberAdministration.designateInitialOwner`（`existing` は `null`）を呼ぶ。招待を経ない。`active` の `Workspace` を `WorkspaceRepository.insert`、所有者を指定した方針を `AccessPolicyRepository.insert`、所有者の `Member` を `MemberRepository.insert` で保存する。生成は、所有者の `MemberId` を返し、ほかのドメインの生成が使う
- ワークスペースごとの設置の区画の段: `WorkspaceDirectory.changeStatus(id, { status: "open" })` と、所有者の組の `MembershipIndex.confirm` を、同じ UnitOfWork で行う
- やり直し: 区画の段は、`WorkspaceRepository.find` が `Workspace` を返す区画を通過する。設置の区画の段は、開設済みの項目と `joined` の項目を変えずに通過する。続きに使うワークスペースの ID と所有者のアカウントは、名簿（`findByName`）とアカウント（`AccountRepository.findByEmail`）から読む。読み込みの途中の名簿は宣言したワークスペースの項目だけを持ち、宣言の名称は宣言の中で重ならない（`WorkspaceNameDuplicated`）ので、`findByName` が返す項目は一つ
- 途中で止まったとき: 名簿に準備中の項目と、所有者の `joining` の項目が残る。所属の索引の修復での項目の扱いは、`MembershipRepair.decide` に従う

ワークスペースの作成（運用コマンド）の中身。宣言を取らず、ワークスペースの ID、名称、初期所有者のメールアドレスを入力にする。作成は、名簿を開設済みにするまでを行い、続けて、招待の発行と作成を終える段を行う。index.md の「区画をまたぐ手順」の「ワークスペースの作成」の行の段ごとに、次を行う。

- ID: 作成は ID で冪等にする（`AGENTS.md` の Idempotent create）。運用コマンドの入口が、初めの実行の始めに `IdGenerator.generate` で ID を決めて示し、ユースケースに渡す。途中で止まった作成は、同じ ID で実行し直すと、止まった段から続け、作成を終える段まで行う。どの段で止まっても同じ。名称は表示の名前で、続きを見つけるのに使わない
- 名簿への登録の段: `WorkspaceDirectory.findById` で、その ID の項目を読む。登録するときは、入力の初期所有者のメールアドレスを `initialOwner` に渡す。項目の状態で、次のとおり分かれる。事前の読み取りと `register` の間に、同じ ID で名称または初期所有者の違う登録が確定した場合は、`register` の契約のとおり、`ConflictError`（理由 `ALREADY_EXISTS`）で拒否する

| その ID の項目 | 作成 |
| --- | --- |
| 無い。その区画の `WorkspaceRepository.find`（`PartitionReader`）が `null` | その ID と名称で `WorkspaceDirectory.register` を呼び、区画の段へ進む |
| 無い。区画に `Workspace` がある（利用終了を完了した ID） | 何も書かずに `ConflictError`（理由 `WORKSPACE_CLOSING`） |
| 準備中、開設済みで、名称または初期所有者（`creation.initialOwner`。`creation` が `null` の項目は初期所有者が無い）が入力と違う | 何も書かずに `ConflictError`（理由 `ALREADY_EXISTS`） |
| 開設済みで、`creation.finished` が真 | 何も書かずに、作成済みであることを返す |
| 準備中、または開設済みで `creation.finished` が偽 | 登録せずに区画の段から続ける（作成の続き） |
| 終了中 | 何も書かずに `ConflictError`（理由 `WORKSPACE_CLOSING`） |

- 区画の段: UnitOfWork を開く前に、その区画の `WorkspaceRepository.find`（`PartitionReader`）で `Workspace` を読む。読んだ結果で、次のとおり分かれる。区画の段は、招待を作らない

| 読んだ `Workspace` | 区画の段 |
| --- | --- |
| 無い | 条件 `none` の一つの UnitOfWork。`Workspace.create` の結果を `WorkspaceRepository.insert`、`AccessPolicy.create` の結果（所有者の指定が `vacant`）を `AccessPolicyRepository.insert` で保存する |
| `active`（作成の続き） | UnitOfWork を開かずに、名簿を開設済みにする段へ進む |
| `closing` または `closed` | UnitOfWork を開かない。結果は「作成と利用終了の並行」が定める |

- 名簿を開設済みにする段: `WorkspaceDirectory.changeStatus(id, { status: "open" })`。終了中の項目と、項目が無い ID は、何も変えずに通過する
- 招待の発行と作成を終える段: 名簿を開設済みにする段の確定の後に、そのワークスペースの ID と、指定した初期所有者のメールアドレスで、下の「初期所有者の招待の発行し直し」の手順を行う。作成は、手順の結果とエラーを、そのまま自分の結果にする。作成は、作成を終える段の確定で終わる
- 途中で止まったとき（名簿を開設済みにする段の確定の前）: 名簿に準備中の項目が残る。区画の段が確定していれば、区画に、`active` の `Workspace` と、所有者がいない（`vacant`）方針がある。招待は発行されていない。状態確認の項目は `provisioning` で、対処は「状態確認の項目」の表が定める。CMS は、作成を自動で進めない
- 途中で止まったとき（名簿を開設済みにする段の確定の後、作成を終える段の確定の前）: 名簿で開設済みの、所有者がいない（`vacant`）ワークスペースが残り、名簿の項目の `creation.finished` は偽のまま。招待の発行が確定していれば、届けていない招待がある。状態確認の項目は `owner_vacant` で、対処は「状態確認の項目」の表が定める
- 作成の実行中に始まった利用終了の扱いは、「作成と利用終了の並行」が定める

初期所有者の招待の発行し直し（運用コマンド）の中身。ワークスペースの ID と宛先のメールアドレスを入力にする。初期所有者の招待を発行する手順は、この一つ。作成が開設の後に行う発行も、この手順を使う。手順は、対象の確認、発行、招待を届ける、作成を終える段。

- 対象の確認（UnitOfWork の外）: `WorkspaceDirectory.findById` で名簿の項目を読む。項目が無い、または準備中（`provisioning`）なら `NotFoundError`。終了中（`retiring`）なら `ConflictError`（理由 `WORKSPACE_CLOSING`）
- 発行（ワークスペースの区画の一つの UnitOfWork。条件 `active`）: `WorkspaceRepository.find`、`AccessPolicyRepository.find`、`InvitationRepository.findPendingInitialOwner`、`InvitationProcess.issueInitialOwner` の結果の、前の招待の `save` と新しい招待の `insert`、招待の作成の操作履歴。読んだ `Workspace` が `active` でなければ、何も書かずに `ConflictError`（理由 `WORKSPACE_CLOSING`）。対象の確認の後に利用終了が進んだ発行が当たる。読むのは初期所有者の招待だけ
- 招待を届ける: 発行の確定の後に行う。設置全体のメール送信を使える設置では、名簿の項目の名称、`InvitationLink.url` の招待の URL、有効期限を本文に持つメールを送る。使えない設置では、招待の値を返す
- 作成を終える段: 招待を届けた後（送信の失敗を含む）に、対象の確認で読んだ項目の `creation` があり、`finished` が偽なら、設置の区画の一つの UnitOfWork で `WorkspaceDirectory.finishCreation(id)` を呼ぶ。それ以外は、UnitOfWork を開かない。発行がエラーを返した手順は、この段を行わない
- 招待の値は、実行のたびに `SecretTokenIssuer.issueForWorkspace` で発行する。保存するのは照合値だけで、前の実行が発行した値は得られない。届ける前に止まった発行は、発行し直しで続け、前の招待は失効する
- 対象と宛先は `InvitationProcess.issueInitialOwner` が定める。受諾は `InvitationProcess.accept` が定める

## 利用再開前の処理

バックアップから復旧した設置で、利用の再開の前に、設置の区画とワークスペースの区画の食い違いを直す処理。写しの時点と範囲は [installation.md](installation.md) の「BackupStore」が定める。操作履歴を残さない。

- 前提: 先に終えているべきドメインは無い。Workspace の中では、名簿の状態、アカウントが無いメンバー、所属の索引の順に直す。後の二つの対象は、開設済みのワークスペース（下の「残件数と確認」の名簿の状態）。索引は `Member` に合わせるので、メンバーの削除を先に行う

名簿の状態。正は、ワークスペースの区画の `Workspace` が持つ。

- 読み取り（UnitOfWork の外）: `WorkspaceDirectory.findPageByStatus` で、すべての状態の項目を、最後のページまで読む。項目ごとに、その区画の `WorkspaceRepository.find` で `Workspace` を読み、`WorkspaceDirectoryEntry.statusFor` で、名簿が持つべき状態を求める
- 書き込み（設置の区画。項目一件ごとに別の UnitOfWork）: 求めた状態が項目の状態と違う項目ごとに、`WorkspaceDirectory.changeStatus` を呼ぶ。開設済みにする変更は `{ status: "open" }`、終了中にする変更は `{ status: "retiring" }`
- どの項目をどの状態に直すかは、`statusFor` の表だけが定める
- 利用再開前の処理は、区画のデータを消去せず、名簿から外さない

アカウントが無いメンバー。設置の区画の写しより後に、新しいアカウントで参加したメンバーは、復旧の後、アカウントが無い。対象は、開設済みのワークスペース。

- 読み取り（UnitOfWork の外）: その区画の `AccessPolicyRepository.find` と `MemberRepository.findPage`（最後のページまで）で、方針とメンバーを読み、`AccountRepository.findByIds` でそのアカウントを読む（ID の列の渡し方は [../usecases/workspace.md](../usecases/workspace.md) の「共通」の「ID の列での読み取り」）。アカウントが無いメンバーが、削除する対象
- 書き込み（ワークスペースの区画。条件 `none`。メンバー一件ごとに別の UnitOfWork）: `MemberRepository.findById` と `MemberRepository.delete`。`AccessPolicy.isOwner` が真のメンバーでは、`AccessPolicy.clearOwner` の結果の `AccessPolicyRepository.save` を、同じ UnitOfWork で行う
- アカウントが無いメンバーは、どのアカウントからも利用できないので、削除する。アカウントを作り直さない
- 所有者のアカウントが無いワークスペースは、`active` のまま、所有者がいない（`vacant`）ワークスペースになる。状態確認の項目は `owner_vacant`（「状態確認の項目」）
- 削除したメンバーに紐付くトークンと、そのメンバーが発行したトークンは、メンバーの利用の解除の後と同じ扱いになる

所属の索引。正は、ワークスペースの区画の `Member` が持つ。対象は、開設済みのワークスペース。利用の再開の前は、進行中の参加が無いので、`MembershipRepair.decide` を使わず、参加の期限を待たずに、`Member` の有無だけで直す。

- 確定する対象の読み取り（UnitOfWork の外）: その区画の `MemberRepository.findPage`（最後のページまで）でメンバーを読み、メンバーごとに `MembershipIndex.find` で項目を読む。`joined` の項目が無いメンバー（項目が無い、または `joining`）が、確定する対象
- 確定（設置の区画。呼び出しごとに別の UnitOfWork）: 確定する対象の組を、メンバーのページごとに `MembershipIndex.confirm` に渡す
- 外す対象の読み取り（UnitOfWork の外）: `MembershipIndex.findPageByWorkspace` で、`next` が `null` になるまで項目を読み、項目ごとに、その区画の `MemberRepository.findByAccount` でメンバーを読む。メンバーが無い項目が、項目の状態によらず、外す対象
- 除外（設置の区画。呼び出しごとに別の UnitOfWork）: 外す対象の、読んだ項目を、ページごとに `MembershipIndex.remove` に渡す
- 開設済みでないワークスペースの項目は直さない。名簿に無いワークスペースの項目は `MembershipRepair.decide` に従う

残件数と確認。項目は、[common.md](common.md) の「各ドメインが公開する項目」の `ResumeItemDefinition`（`domain` は `workspace`）の定数の列で公開し、件数は、下の表の求め方で、ユースケースが上の各処理の読み取りから求める。

```ts
type WorkspaceResumeItemKind =
  | "directory_to_fix"
  | "directory_provisioning"
  | "directory_retiring"
  | "member_to_remove"
  | "owner_vacant"
  | "membership_to_confirm"
  | "membership_to_remove";

const WorkspaceResumeItems: {
  definitions: readonly (ResumeItemDefinition & Readonly<{ kind: WorkspaceResumeItemKind }>)[];
};
```

| `kind` | 件数の求め方 | `unit` | `countsAsRemaining` | `shownInReview` |
| --- | --- | --- | --- | --- |
| `directory_to_fix` | 直す名簿の項目の数。名簿の状態が `statusFor` の結果と違う項目 | `installation` | 真 | 偽 |
| `directory_provisioning` | 準備中の項目の数。項目ごとに、ワークスペースの ID と名称を示す。作成の途中として示す | `installation` | 偽 | 真 |
| `directory_retiring` | 終了中の項目の数。項目ごとに、ワークスペースの ID と名称を示す。利用終了の途中として示す | `installation` | 偽 | 真 |
| `member_to_remove` | 削除するメンバーの数。アカウントが無いメンバー | `workspace` | 真 | 真 |
| `owner_vacant` | 利用の再開の後に所有者がいない（`vacant`）ワークスペースの数。開設済みのワークスペースのうち、所有者のメンバーのアカウントが無いワークスペースと、所有者の指定が既に `vacant` のワークスペース。復旧点で作成の途中で、区画まで作り終えていたワークスペースを含む。状態確認の `owner_vacant` が、利用の再開の後に示す集合と同じ。項目ごとに、ワークスペースの ID と名称を示す。利用は続くことと、「状態確認の項目」の `owner_vacant` の対処を示す | `installation` | 偽 | 真 |
| `membership_to_confirm` | 所属の索引で確定する対象の数 | `workspace` | 真 | 偽 |
| `membership_to_remove` | 所属の索引から外す対象の数 | `workspace` | 真 | 偽 |

- `shownInReview` が偽の項目の扱いは、[common.md](common.md) の「各ドメインが公開する項目」の `ResumeItemDefinition` が定める（HOST-08 の手順 5 は、名簿と所属の索引の食い違いを示さない）

- 件数は、書き込まずに、上の各処理の読み取りで数える。この節の名簿の状態（開設済み、準備中、終了中）は、名簿のすべての項目に `statusFor` を適用した後の状態を指す。処理を行う実行と、利用再開前の確認は、どちらもこの状態で対象を決める。`unit` が `workspace` の項目は、開設済みのワークスペースごとに求める。`unit` が `installation` の項目は、設置の全体で一つの件数を求め、該当するワークスペースの ID と名称の列を添える
- 残件数への振り分けは [installation.md](installation.md) の「ResumeReview」が定める。処理を終えると、`countsAsRemaining` が真の項目は 0 になる
- `owner_vacant` は、処理の後も残る。所有者のアカウントが無いメンバーの削除は `member_to_remove` に数える
- `directory_provisioning` のワークスペース（復旧点で作成の途中で、区画まで作り終えていなかったもの。区画に `Workspace` が無い準備中の項目）は、利用の再開の後に、状態確認が作成の途中（「状態確認の項目」の `provisioning`）として示す

## 利用終了の消去

Workspace が、ワークスペースの利用終了に公開する手順の中身。手順は、対象の確認、名簿の状態の変更、開始、区画の消去、最後の段、終了前の確認。どの手順をどの順に、どのユースケース（`closeWorkspace`、`completeWorkspaceClosures`）が行うかは、[index.md](index.md) の「区画をまたぐ手順」が定め、この節は順を持たない。消去済みの手順は 0 件で通過する。

- 前提: 区画の消去で、先に消去を終えているべき相手は、ワークスペースの区画のほかのすべてのドメイン
- 前提: 最後の段を行う時点は [index.md](index.md) の「区画をまたぐ手順」が定める。条件と、条件に要る記録は、Media の「MediaClosure」（[media.md](media.md)）が定める

対象の確認。

- `WorkspaceDirectory.findById` で名簿の項目を読む。項目が無ければ `NotFoundError`。利用終了が完了したワークスペースと、存在しないワークスペースが当たる。準備中、開設済み、終了中のどの状態の項目も受け付ける
- 利用終了のユースケースが対象の確認に受けた名称を、`WorkspaceDirectoryEntry.confirmName` に渡す。一致しなければ `BusinessRuleError`（`WorkspaceNameMismatch`）で、何も変わらない

名簿の状態の変更。設置の区画の一つの UnitOfWork。

- `WorkspaceDirectory.changeStatus(workspaceId, { status: "retiring" })` を呼ぶ
- 確定した時点からの入口ごとの結果は [index.md](index.md) の「利用できるワークスペース」が、終了中のワークスペースを対象にする定期ジョブは「定期ジョブ」の表が定める
- 準備中の項目も終了中にする。作成の途中でやめるワークスペースは、これで利用終了の対象になる

開始。ワークスペースの区画の一つの UnitOfWork（条件 `none`）。ほかのどのドメインの消去よりも前に行う。

- `WorkspaceRepository.find` でワークスペースを読み、読んだ結果（無ければ `null`）を `Workspace.beginClosing` に渡す。読んだ結果で、次のとおり分かれる

| 読んだ `Workspace` | 開始 |
| --- | --- |
| 無い（区画を作る前に止まった作成） | `beginClosing` の結果（`closing`）を `WorkspaceRepository.insert` で保存する |
| `active` | `beginClosing` の結果（`closing`）を `WorkspaceRepository.save` で保存する |
| `closing`（開始を終えた利用終了の続き） | 何も書かずに通過する |
| `closed`（区画の消去を終えた利用終了の続き） | 何も書かずに通過する |

- 開始の後の区画には、`closing` または `closed` の `Workspace` がある
- 確定した時点で、ワークスペースは `closing` になる。区画の `Member` は、この確定から、区画の消去が消すまで、増えも減りもしない（`Member` を作る参加の後段と、`Member` を削除する利用の解除は、条件が `active`）

区画の消去。ワークスペースの区画の UnitOfWork（条件 `none`）。開始の後に呼ぶ。手順を呼ぶ側が、開始を先に行う。区画の消去の終わり方（イベントの消去と、`Workspace` を `closed` にする保存）は、この節だけが定める。

- `InvitationRepository.deleteAll(limit)`、`JoiningGuardRepository.deleteAll(limit)`、`AccessPolicyRepository.deleteAll(limit)`、`MemberRepository.deleteAll(limit)` を、この順に、それぞれ消し終えるまで呼ぶ。呼び方は [common.md](common.md) の「TransactionalRepository」の件数を区切る削除
- 招待、囲い、方針、メンバーを消し終えた区画で、`WorkspaceContext.deleteEvents(limit)`（[common.md](common.md) の「UnitOfWorkProvider」）を、同じ呼び方で、消し終えるまで呼ぶ。区画のほかのすべてのドメインの消去の後に行う。区画が `closing` または `closed` の区画に確定する UnitOfWork は、ドメインイベントを記録しないので、消し終えたイベントは増えない
- イベントを消し終えてから、`WorkspaceRepository.find` で読んだ `Workspace` に `Workspace.close` を呼び、結果が、渡したワークスペースと違う場合（`closing` だった場合）に、`WorkspaceRepository.save` で保存する。返した件数が `limit` より少ない `deleteEvents` の呼び出しと、`closed` にする保存は、同じ UnitOfWork で行える
- `closed` にした区画には、`closed` の `Workspace` と、Media が最後の段の前に片付ける記録（[media.md](media.md) の「MediaClosure」）のほかに、集約、記録、ドメインイベントは残らない
- `closed` の `Workspace` は、区画に残る印で、利用者のデータを持たない。持つのは、状態、`version`、作成と更新の日時だけ
- `Workspace` が既に `closed` の区画では、招待、メンバー、イベントの消去は 0 件で、`closed` にする保存を、何も書かずに通過する
- メンバーの消去は、アカウントと所属の索引を変えない
- `closed` にした後に区画に届き得る書き込みは、[common.md](common.md) の「WriteScope」の条件と `WorkspaceRepository.insert` の契約が定める。届くのは、Media の条件 `none` の書き込みだけ（[media.md](media.md) の「MediaClosure」、[../usecases/media.md](../usecases/media.md) の「書き込みの UnitOfWork」）
- 途中で止まったとき: 名簿に終了中の項目と、消去を終えていない区画が残る。`closed` にする保存の前に止まった区画の `Workspace` は、`closing` のまま。続きは、同じ手順を初めから呼ぶ

最後の段。ワークスペースを名簿から外す段。設置の区画の一つの UnitOfWork（`{ kind: "installation" }`）で、次の三つを確定する。

- `WorkspaceDirectory.remove(workspaceId)`
- `MembershipIndex.removeWorkspace(workspaceId)`。そのワークスペースの項目を、状態によらずすべて外す
- Token の「利用終了の消去」の `DeclaredTokenIndex.removeByWorkspace(workspaceId)`（[token.md](token.md)）
- アカウント（Identity）は変えない。項目が外れて、どのワークスペースの項目も持たなくなったアカウントの扱いは、[identity.md](identity.md) の「Account」のライフサイクルが定める
- UnitOfWork の書き込みは、索引の項目とトークンの数によらず 3 回（`MembershipIndex.removeWorkspace` の契約）
- この段が確定した時点で、ワークスペースは名簿から外れる。以後、そのワークスペースの ID を指定した利用終了は `NotFoundError`。利用終了は、この段の確定で完了する。完了した時点で、区画の消去の後に残った Media の記録は片付いていて、そのワークスペースの区画に残るのは `closed` の `Workspace` だけ。保管先に実体は残っていない。名簿に項目が無い区画の扱いは [installation.md](installation.md) の「BackupStore」の写しの範囲に従う。`closed` の `Workspace` は、設置全体の利用終了での保存先の消去で消える
- この段の後に確定した、そのワークスペースの項目を置く書き込み（参加の前段、確かめ直し）が残す項目は、`MembershipRepair.decide` に従う
- やり直し: 確定済みの段をもう一度呼ぶと、名簿の項目と索引の項目が無いので、何も変えずに通過する
- 途中で止まったとき: この段が確定する前は、名簿に終了中の項目と、そのワークスペースの索引の項目が残り、次の起動が、同じ手順を初めから呼ぶ

終了前の確認。データを変えない。対象は [installation.md](installation.md) の「WorkspaceClosure」が定める。

- 示すワークスペースの ID、名称、名簿の状態は、`WorkspaceDirectory.findById` の項目から得る。利用終了の途中かどうかは、名簿の状態が `retiring` かどうか

```ts
type WorkspaceClosureItemKind = "member" | "pending_invitation";

const WorkspaceClosureItems: {
  kinds: readonly WorkspaceClosureItemKind[];
  lostData: readonly ["member", "invitation", "membership"];
  remainsOutside: readonly ["sent_invitation_mail"];
};
```

| 示す項目の `kind` | 件数の求め方 |
| --- | --- |
| `member` | 失われるメンバーの数。区画の `MemberRepository.findPage` の `count` |
| `pending_invitation` | 失われる未使用の招待の数。区画の `InvitationRepository.findPendingPage` の `count` に、`findPendingInitialOwner` が招待を返せば 1 を足す |

- 利用終了で失われるデータの種類（`lostData`）: メンバーとその権限、所有者の指定、多要素認証の必須（`member`）、招待（`invitation`）、所属の索引の項目（`membership`）
- CMS の通常データの外に残る情報（`remainsOutside`）: 送信済みの招待のメール（`sent_invitation_mail`）。受信箱に残り、CMS から消去できない。メールの招待の URL では、利用終了の後に参加できない
- 消去する資格情報: Workspace の集約は、資格情報を持たない。招待の値は照合値だけを保存し、招待とともに消える
- 区画に `Workspace` が無い準備中のワークスペースと、区画の `Workspace` が `closed` のワークスペースでは、区画の件数は 0

## 作成と利用終了の並行

ワークスペースの作成（運用コマンド）と、同じワークスペースの利用終了が並行したときの規則。この節だけが定める。作成の段の中身は「初期構成の宣言」、利用終了の段の中身は「利用終了の消去」が定め、この節は、二つが重なったときの結果を定める。段の順は [index.md](index.md) の「区画をまたぐ手順」の、ワークスペースの作成と、利用終了、利用終了の完了の行にある。

並行が生じるのは、準備中の項目の作成（名簿への登録の段を終えた作成と、作成の続き）と、開設の後に作成が行う招待の発行と、そのワークスペースの利用終了。

規則は一つ。「Workspace」の不変条件から、作成の区画の段と、利用終了の開始は、どちらも区画の `Workspace` を書くので、一方だけが先に確定し、後の側は、先の側の結果に従う。

| 先に確定した側 | 後の側 |
| --- | --- |
| 作成の区画の段（`Workspace` の `insert`） | 利用終了の開始が、`active` の `Workspace` を `closing` にし、区画の消去が、作成が作った集約、招待、操作履歴を消して、`closed` にする |
| 利用終了の開始（`closing` の `Workspace` の `insert`、または `active` を `closing` にする `save`） | 作成の区画の段は、何も確定しない。区画の消去が `closed` にした後も同じ |

作成の区画の段の結果。

| 区画の段 | 作成 |
| --- | --- |
| UnitOfWork の前に、`closing` または `closed` の `Workspace` を読んだ | 何も書かず、招待を発行せずに、`ConflictError`（理由 `WORKSPACE_CLOSING`） |
| `Workspace` が無い区画を読んだ後、`insert` の確定の前に、開始が `closing` の `Workspace` を置いた | UnitOfWork は `ConflictError`（理由 `ALREADY_EXISTS`）で確定しない。エラーをそのまま返す |

- 開始の `insert` が、並行した `Workspace` の `insert`（作成の区画の段、もう一つの開始）に負けた場合は `ConflictError`（理由 `ALREADY_EXISTS`）、開始の `save` が、並行した `Workspace` の保存に負けた場合は `ConflictError`（理由 `STALE_VERSION`）で、開始の UnitOfWork は確定しない。名簿の項目は終了中なので、利用終了の続き（定期ジョブ「利用終了の完了」、または実行し直し）が、開始から行い、読んだ `Workspace` に従う（`active` なら `closing` にする）
- 確定しなかった作成を、同じ ID で実行し直すと、名簿への登録の段が、終了中の項目を読んで `WORKSPACE_CLOSING` で拒否する

招待の発行の結果。作成の区画の段が先に確定し、その後に利用終了が進んだ作成が当たる。名簿を開設済みにする段は、終了中の項目を変えずに通過し、作成は、発行し直しの手順（「初期構成の宣言」）の結果を、そのまま返す。

| 対象の確認が読んだ項目 | 作成 |
| --- | --- |
| 開設済み | 招待を発行して届ける。発行の UnitOfWork が開始に負けた場合は、発行し直しと同じ `ConflictError`（読んだ `Workspace` が `active` でなければ `WORKSPACE_CLOSING`、確定の時点なら `WORKSPACE_NOT_ACTIVE`）で、招待を発行しない |
| 終了中 | 招待を発行せず、`ConflictError`（理由 `WORKSPACE_CLOSING`）。区画は、利用終了が消去する |
| 無い | 招待を発行せず、`NotFoundError`。利用終了が、最後の段まで完了している。区画の `Workspace` は `closed` |

- 開設済みを読んで招待を発行した後に、利用終了が名簿の項目を終了中にした作成は、招待を届ける。届いた招待は、受諾できない（名簿で開設済みでないワークスペースの招待は無効）
- エラーを返した作成は、何も書き足さない。作成が書いたものは、利用終了が消去する
- 作成の続きが、終了中の項目、または利用終了を完了した ID で実行された場合は、名簿への登録の段が `WORKSPACE_CLOSING` で拒否する（「初期構成の宣言」）。この節の対象は、準備中の項目を読んだ後に、利用終了が始まった作成

作成と利用終了の段の間に復旧が完了する場合は、この節の対象でない。復旧の前提（HOST-08）と、復旧の前から続く手順の扱いは、[installation.md](installation.md) の「BackupStore」が定める。

## 状態確認の項目

状態確認に示す Workspace の項目。項目は、作成の途中、利用終了の途中、所有者がいないワークスペースの三つで、どれも設置の全体の項目（単位が設置）。設置の全体で一つの件数を求め、該当するワークスペースの ID と名称を添える。項目ごとの必要な対処は、下の表だけが定める（[index.md](index.md) の「区画をまたぐ手順」の定期ジョブと状態確認）。

```ts
type WorkspaceStatusItemKind =
  | "provisioning"
  | "retiring"
  | "owner_vacant";

type InitialOwnerInvitationState = "usable" | "expired" | "none";

const WorkspaceStatusItems: {
  kinds: readonly WorkspaceStatusItemKind[];
  requiresAction(kind: WorkspaceStatusItemKind, count: number): boolean;
  initialOwnerInvitation(
    pending: PendingInvitation | null,
    now: Date,
  ): InitialOwnerInvitationState;
};
```

| `kind` | 件数の求め方 | `requiresAction` |
| --- | --- | --- |
| `provisioning` | 作成の途中のワークスペースの数。名簿（`WorkspaceDirectory.findPageByStatus`）で準備中（`provisioning`）の項目の数。項目ごとに、ワークスペースの ID と名称を示す。区画を読まない | `count` が 1 以上のとき真。ホスティング運営者が、同じ ID で作成を実行し直すか、利用終了を実行する |
| `retiring` | 利用終了が完了していないワークスペースの数。名簿で終了中（`retiring`）の項目の数。項目ごとに、ワークスペースの ID と名称を示す。区画を読まない | 常に偽。定期ジョブ「利用終了の完了」が完了させる |
| `owner_vacant` | 所有者がいない（`vacant`）ワークスペースの数。名簿で開設済みのワークスペースのうち、区画の `AccessPolicyRepository.find` が返す方針に `AccessPolicy.isVacant` が真のものの数。項目ごとに、ワークスペースの ID と名称を示す。ワークスペースごとに添える値（[installation.md](installation.md) の `StatusItemWorkspace.detail`）は、キー `invitation` に `initialOwnerInvitation` の結果を持つ組 | `count` が 1 以上のとき真。必要な対処は、`invitation` の値で分かれる。`usable` は、招待された人の参加を待つ。招待が届いていなければ、ホスティング運営者が、初期所有者の招待を発行し直す。`expired` と `none` は、ホスティング運営者が、初期所有者の招待を発行し直す |

- `initialOwnerInvitation` は、所有者がいないワークスペースごとに、その区画の `InvitationRepository.findPendingInitialOwner` の結果から、初期所有者の招待の状態を求める。`pending` の招待が無ければ `none`。あれば、`Invitation.standing` が `usable` なら `usable`（有効）、`expired` なら `expired`（期限切れ）。招待の宛先と値は示さない
- `provisioning` と `retiring` は、ワークスペースごとに添える値を持たない
- 状態確認の対象のワークスペース（名簿で開設済みの項目）の ID と名称は、名簿から読む。ほかのドメインの、単位がワークスペースの項目は、このワークスペースごとに示す
- 作成が進行中のワークスペースも、名簿を開設済みにする段が確定するまで、`provisioning` に数える。途中で止まった作成と区別しない

## ユースケース（概要）

詳細は [../usecases/workspace.md](../usecases/workspace.md) が定める。受け付ける実行者の種類と必要な権限は、各ユースケースが定める。

- メンバーの認可結果を求める
- 所属するワークスペースを一覧する
- 最後に利用したワークスペースを記録する
- メンバーを招待する
- 招待を取り消す
- 未使用の招待を一覧する
- 招待の内容と状態を確かめる
- 招待を受諾する
- メンバーを一覧する
- メンバーの権限を変更する
- メンバーの利用の解除の影響を確かめる
- メンバーの利用を解除する
- 所有者を変更する
- 多要素認証を設定していないメンバーを確かめる
- 多要素認証を必須にする
- 多要素認証の必須を解除する
- 期限切れの招待を掃除する
- 所属の索引を修復する

名簿への登録と状態の変更（`WorkspaceDirectory`）、ワークスペースの作成（`Workspace.create` と `AccessPolicy.create`）、初期所有者の招待の発行と発行し直し（`InvitationProcess.issueInitialOwner`）、初期構成による所有者の指定（`MemberAdministration.designateInitialOwner`。招待を経ない）、利用終了（`Workspace.beginClosing` と「利用終了の消去」）、利用再開前の処理は、Installation のユースケースが、このドメインの振る舞いとポートを使って行う。
