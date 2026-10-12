# DeviceAuthorizationRepository

契約は [token.md](../../domains/token.md) の「ポート」と「トランザクション境界」、[common.md](../../domains/common.md) の `TransactionalRepository` と `UnitOfWorkProvider` が定める。

## 前提

- 依頼は `DeviceAuthorization.start` で作り、`approve` / `deny` / `redeem` で変える
- リポジトリは、設置の区画にある。書き込み（`insert` / `save` / `deleteLapsed`）は `UnitOfWorkProvider.run` の中で、文脈のリポジトリで呼ぶ。`WriteScope` は `{ kind: "installation" }`。断りが無ければ、一つの操作を一つの UnitOfWork で確定し、読み取りは UnitOfWork の外で呼ぶ
- 「保存済み」は、`insert` を確定した状態を指す。`v` は、直前の読み取りが返した `expectedVersion` を指す
- 「並行して」は、二つの UnitOfWork を同時に始め、どちらも確定まで進めることを指す
- 競合と違反は、メソッドの呼び出しか確定のどちらかで現れる。`run` が `ConflictError` を投げることと、その UnitOfWork の書き込みが一つも反映されないことで確かめる
- ID の順は、文字列のコードポイントの昇順
- W1、W2 は別のワークスペースの ID。`IdGenerator.generate` の値から作る。ワークスペースの区画に何も確定していなくてよい。メンバーとアカウントの ID も同じ
- e1 < e2 < e3 は、固定の日時
- 「削除してある」は、`deleteLapsed` を確定して、その依頼を削除した状態を指す
- 「承認済み（W1）」は、`approval.workspaceId` が W1 の `approved` の依頼を指す
- `redeem` に渡すトークンの ID は `IdGenerator.generate` の値から作る。トークンを確定する必要は無い
- 同じ区画のほかの書き込みには、`accountRepository` を使う

## insert

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-deviceAuthorizationRepository-01 | 依頼が無い | `pending` の依頼 D を `insert` する | `findById(D.id)` が、`grants`、`userCode`、二つの照合値、`expiresAt` を含めて D と等しいエンティティを返す |
| TC-ports-deviceAuthorizationRepository-02 | D が保存済み | D と同じ ID で、二つの照合値が別の依頼を `insert` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。`findById(D.id)` は D を返す |
| TC-ports-deviceAuthorizationRepository-03 | D が保存済み（`deny` して保存した D でも同じ） | D と承認の値の照合値が同じで、ID と受け取りの値の照合値が別の依頼 F を `insert` する | `run` が `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）を投げる。`findById(F.id)` は `null` |
| TC-ports-deviceAuthorizationRepository-04 | D が保存済み | D と受け取りの値の照合値が同じで、ID と承認の値の照合値が別の依頼 F を `insert` する | `run` が `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）を投げる。`findById(F.id)` は `null` |
| TC-ports-deviceAuthorizationRepository-06 | D が保存済み | D と `userCode` が同じで、ID と二つの照合値が別の依頼 F を `insert` する | 確定する。`findById(F.id)` は F を返す |
| TC-ports-deviceAuthorizationRepository-07 | D を保存し、削除してある | D と二つの照合値が同じで、ID が別の依頼 F を `insert` する | 確定する。`findByApprovalDigest` が F を返す |
| TC-ports-deviceAuthorizationRepository-08 | 依頼が無い | 承認の値の照合値が同じで、ID が別の D と F を、並行して `insert` する | 一方だけが確定し、他方の `run` は `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）を投げる。`findById` は、確定した方だけを返す |
| TC-ports-deviceAuthorizationRepository-09 | 依頼が無い | 受け取りの値の照合値が同じで、ID が別の D と F を、並行して `insert` する | 一方だけが確定し、他方の `run` は `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）を投げる。`findById` は、確定した方だけを返す |

## findById

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-deviceAuthorizationRepository-10 | 依頼が無い | 未使用の ID で `findById` を呼ぶ | `null` を返す |
| TC-ports-deviceAuthorizationRepository-11 | D を `approve` して保存してある | `findById(D.id)` を呼ぶ | `status` が `approved` で、`approval` の `workspaceId`、`member`、`approvedAt`、`tokenExpiresAt` を持つエンティティを返す |
| TC-ports-deviceAuthorizationRepository-12 | D を `deny` して保存してある | `findById(D.id)` を呼ぶ | `status` が `denied` で、`deniedAt` と、`reason` が `denied_by_account` の `denial` を持つエンティティを返す |
| TC-ports-deviceAuthorizationRepository-13 | 権限が足りない `approve` の結果（`denied`）を保存してある | `findById(D.id)` を呼ぶ | `denial` の `reason` が `insufficient_permissions` で、`refusals` の各要素の `grant` と `requiresAnyOf` が保存した値と等しいエンティティを返す |
| TC-ports-deviceAuthorizationRepository-14 | D を `approve`、`redeem` の順に保存してある | `findById(D.id)` を呼ぶ | `status` が `redeemed` で、`approval`、`redeemedAt`、`tokenId` を持つエンティティを返す |
| TC-ports-deviceAuthorizationRepository-15 | D を保存し、削除してある | `findById(D.id)`、`findByApprovalDigest(D.approvalDigest)`、`findByDeviceCodeDigest(D.deviceCodeDigest)` を呼ぶ | どれも `null` を返す |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-deviceAuthorizationRepository-16 | `pending` の D が保存済み。`findById` で D と `v` を得ている | `approve` の結果を `save` する | `findById(D.id)` が `approved` のエンティティを返す |
| TC-ports-deviceAuthorizationRepository-17 | `pending` の D が保存済み。`findById` で D と `v` を得ている | `deny` の結果、または権限が足りない `approve` の結果（`denied`）を `save` する | `findById(D.id)` が `denied` のエンティティを返す。`denial` の `reason` は、それぞれ `denied_by_account` と `insufficient_permissions` |
| TC-ports-deviceAuthorizationRepository-18 | 承認済みの D が保存済み。`findById` で D と `v` を得ている | `redeem` の結果を `save` する | `findById(D.id)` が、`tokenId` が渡した ID の `redeemed` のエンティティを返す |
| TC-ports-deviceAuthorizationRepository-19 | `pending` の D が保存済み。`findById` で D と `v` を得た後、別の UnitOfWork が D の `deny` を `save` して確定した | `v` で `approve` の結果を `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById(D.id)` は `denied` のエンティティを返す |
| TC-ports-deviceAuthorizationRepository-20 | `pending` の D が保存済み。二つの呼び出し側が、同じ `v` を得ている | `approve` の `save` と、`deny` の `save` を、並行して呼ぶ | 一方だけが確定し、他方の `run` は `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById(D.id)` は確定した方の状態を返す |
| TC-ports-deviceAuthorizationRepository-21 | 承認済みの D が保存済み。二つの呼び出し側が、同じ `v` を得ている | それぞれの UnitOfWork で、違うトークンの ID を渡した `redeem` の `save` を、並行して呼ぶ | 一方だけが確定し、他方の `run` は `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById(D.id)` の `tokenId` は、確定した方が渡した ID |
| TC-ports-deviceAuthorizationRepository-22 | `pending` の D が保存済み。`findById` で D と `v` を得た後、別の UnitOfWork が、D を削除する `deleteLapsed` を確定した | `v` で `approve` の結果を `save` する | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById(D.id)` は `null` |

## findByApprovalDigest

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-deviceAuthorizationRepository-24 | 依頼が無い。または、D が保存済み | 未使用の照合値（D のどちらの照合値とも違う値）で `findByApprovalDigest` を呼ぶ | `null` を返す |
| TC-ports-deviceAuthorizationRepository-26 | `pending` の D と、照合値が別の F が保存済み | `findByApprovalDigest(D.approvalDigest)` を呼ぶ | D と `expectedVersion` を返す |
| TC-ports-deviceAuthorizationRepository-28 | `approved`、`denied`、`redeemed` の依頼と、`expiresAt` が過去の日時になる依頼が保存済み | それぞれの承認の値の照合値で `findByApprovalDigest` を呼ぶ | どれも、その状態のまま依頼を返す |
| TC-ports-deviceAuthorizationRepository-32 | `pending` の D が保存済み | `findByApprovalDigest` が返した `expectedVersion` で、`approve` の結果を `save` する | 確定する。`findById(D.id)` は `approved` のエンティティを返す |

## findByDeviceCodeDigest

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-deviceAuthorizationRepository-33 | 依頼が無い。または、D が保存済み | 未使用の照合値（D のどちらの照合値とも違う値）で `findByDeviceCodeDigest` を呼ぶ | `null` を返す |
| TC-ports-deviceAuthorizationRepository-35 | `pending` の D と、照合値が別の F が保存済み | `findByDeviceCodeDigest(D.deviceCodeDigest)` を呼ぶ | D と `expectedVersion` を返す |
| TC-ports-deviceAuthorizationRepository-37 | `approved`、`denied`、`redeemed` の依頼と、`expiresAt` が過去の日時になる依頼が保存済み | それぞれの受け取りの値の照合値で `findByDeviceCodeDigest` を呼ぶ | どれも、その状態のまま依頼を返す |
| TC-ports-deviceAuthorizationRepository-41 | 承認済みの D が保存済み | `findByDeviceCodeDigest` が返した `expectedVersion` で、`redeem` の結果を `save` する | 確定する。`findById(D.id)` は `redeemed` のエンティティを返す |

## deleteLapsed

保持期間の起点は `DeviceAuthorization.lapsedAt`。対象と順序は [common.md](../../domains/common.md) の「期限つきの一度きりの値」が定める。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-deviceAuthorizationRepository-42 | 依頼が無い | `deleteLapsed(e2, 10)` を呼ぶ | 0 を返す |
| TC-ports-deviceAuthorizationRepository-43 | `expiresAt` が e1、e2、e3 の D、F、G | `deleteLapsed(e2, 10)` を呼ぶ | 2 を返す。`findById` は D と F に `null`、G に G を返す |
| TC-ports-deviceAuthorizationRepository-46 | `expiresAt` が e1 の、`pending`、承認済み（W1）、承認済み（W2）、`denied`、`redeemed` の依頼が 1 件ずつ | `deleteLapsed(e2, 10)` を呼ぶ | 5 を返す。5 件とも `findById` は `null` |
| TC-ports-deviceAuthorizationRepository-48 | `expiresAt` が e1、e2、e3 の D、F、G | `deleteLapsed(e3, 2)` を呼ぶ | 2 を返す。`findById` は D と F に `null`、G に G を返す |
| TC-ports-deviceAuthorizationRepository-49 | `expiresAt` が e1 の依頼が 3 件 | `deleteLapsed(e2, 2)` を呼ぶ | 2 を返す。`findById` は、ID の昇順の先頭 2 件に `null`、3 件目に依頼を返す |
| TC-ports-deviceAuthorizationRepository-50 | `expiresAt` が e3 以前の依頼が 4 件 | `deleteLapsed(e3, 3)` を三度続けて、別の UnitOfWork で呼ぶ | 順に 3、1、0 を返す。4 件とも `findById` は `null` |
| TC-ports-deviceAuthorizationRepository-53 | `expiresAt` が e1 の依頼が 2 件 | `deleteLapsed(e2, BoundedDeletion.maxLimit)` を呼ぶ | 2 を返す |
| TC-ports-deviceAuthorizationRepository-54 | `expiresAt` が e1 の D | `deleteLapsed(e2, 0)` と、`BoundedDeletion.maxLimit` を超える `limit` の `deleteLapsed(e2, limit)` を、それぞれ呼ぶ | どちらも `SystemError`（理由 `INVALID_QUERY`）になる。`findById(D.id)` は D を返す |

## UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-deviceAuthorizationRepository-56 | 承認済みの D が保存済み | 一つの UnitOfWork で、D の `redeem` の `save` と、アカウント A の `accountRepository.insert` を呼び、確定する | `findById(D.id)` は `redeemed` の D、`accountRepository.findById(A.id)` は A を返す |
| TC-ports-deviceAuthorizationRepository-57 | 承認済みの D が保存済み | 一つの UnitOfWork で、同じ 2 つの書き込みを呼んだ後、`fn` が例外を投げる | `findById(D.id)` は `approved` の D、`accountRepository.findById(A.id)` は `null` を返す |
| TC-ports-deviceAuthorizationRepository-58 | `pending` の D が保存済み。`findById` で `v` を得た後、別の UnitOfWork が D の `deny` を確定した | 一つの UnitOfWork で、新しい依頼 F の `insert` と、`v` での D の `save` を呼ぶ | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById(F.id)` は `null` |
| TC-ports-deviceAuthorizationRepository-59 | `expiresAt` が e1 の依頼が 2 件 | 一つの UnitOfWork の中で、`deleteLapsed(e2, 10)` を呼んだ後、`fn` が例外を投げる | 2 件とも `findById` が返る |
