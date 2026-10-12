# TokenRepository

契約は [token.md](../../domains/token.md) の「ポート」と「トランザクション境界」、[common.md](../../domains/common.md) の `TransactionalRepository` と `UnitOfWorkProvider` が定める。

## 前提

- トークンは `Token.issueForApp` / `Token.issueForMember` で作り、`Token.revoke` / `Token.applyScopeChange` で変える
- 書き込み（`insert` / `insertMany` / `save` / `saveMany` / `deleteLapsed` / `deleteAll`）は `UnitOfWorkProvider.run` の中で、文脈のリポジトリで呼ぶ。`WriteScope` は、そのワークスペースの `workspace` で、条件は `none`。断りが無ければ、一つの操作を一つの UnitOfWork で確定し、読み取りは UnitOfWork の外で呼ぶ
- `recordUse`（UnitOfWork の外の記録）は、`PartitionReader.workspace` から得るリポジトリで呼ぶ
- 「保存済み」は、`insert` を確定した状態を指す。`v` は、直前の `findById` が返した `expectedVersion` を指す
- 「並行して」は、二つの UnitOfWork を同時に始め、どちらも確定まで進めることを指す
- 競合と違反は、メソッドの呼び出しか確定のどちらかで現れる。`run` が `ConflictError` を投げることと、その UnitOfWork の書き込みが一つも反映されないことで確かめる
- ID の順は、文字列のコードポイントの昇順
- リポジトリは、ワークスペースの区画に結び付き、メソッドはワークスペースの ID を引数に取らない。W1、W2 は別のワークスペース。トークンは、ワークスペースの ID のフィールドを持たない。「W1 の T」は、W1 の区画に保存した T を指す。断りの無い呼び出しは、W1 の区画のリポジトリに対して行う。W2 のトークンの書き込みと、「W2 の区画」と書いた呼び出しは、W2 の区画のリポジトリに対して行う
- M1、M2 は別のメンバー。メンバーとアカウントの ID は `IdGenerator.generate` の値から作る。断りが無ければ、メンバーとアカウントを確定する必要は無い。「在籍する」メンバーは、同じ区画の `memberRepository.insert` で確定したメンバーを指す
- t0 < t1 < t2 < t3 は固定の日時で、t0 から t3 までは `OneTimeValueRetention.ms` より短い。断りが無ければトークンは t0 に発行する
- 「削除してある」は、`deleteLapsed` または `deleteAll` を確定して、そのトークンを削除した状態を指す
- 断りが無ければ、トークンはアプリに紐付くサーバー用で、範囲は `workspace`、期限は `never`、W1 の区画に保存し、発行者は M1

## insert

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-tokenRepository-01 | トークンが無い | トークン T を `insert` する | `findById(T.id)` が T と等しいエンティティを返す。`findByValueDigest(T.valueDigest)` の `lastUsedAt` は `null` |
| TC-ports-tokenRepository-02 | トークンが無い | ブラウザ用の T（利用元の限定が `listed` で 2 件、利用量の制限あり、範囲が紐付け済みのモデルとフォームの `selection`、期限が `at`）と U（利用元の限定が `any`、利用量の制限が `null`）を `insert` する | `findById` が、`profile`、`scope`、`expiry`、`issuerPermissions` を含めて、T と U のそれぞれと等しいエンティティを返す |
| TC-ports-tokenRepository-04 | トークンが無い | `origin` が `bootstrap` で、範囲が紐付け待ちのモデルとフォームの `selection` のトークン T を `insert` する | `findById(T.id)` が、紐付け待ちの要素のキーを含めて T と等しいエンティティを返す |
| TC-ports-tokenRepository-05 | トークンが無い | `origin` が `device` の、メンバーに紐付くトークン T と、`origin` が `preview` の、メンバーに紐付くトークン U を `insert` する | `findById` が、T と U のそれぞれと等しいエンティティを返す |
| TC-ports-tokenRepository-07 | T が保存済み | T と同じ ID で、照合値が別のトークンを `insert` する | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。`findById(T.id)` は T を返す |
| TC-ports-tokenRepository-08 | W1 の T が保存済み（`revoke` して保存した T でも同じ） | T と同じ照合値で、ID が別のトークン U を `insert` する | `run` が `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）を投げる。`findById(U.id)` は `null` |
| TC-ports-tokenRepository-09 | W1 の T が保存済み | W2 の区画で、T と同じ照合値で、ID が別のトークン U を `insert` する | `run` が確定する。W1 の区画の `findByValueDigest` は T を、W2 の区画の `findByValueDigest` は U を返す |
| TC-ports-tokenRepository-11 | T を保存し、削除してある | T と同じ照合値で、ID が別のトークン U を `insert` する | 確定する。`findByValueDigest` が U を返す |
| TC-ports-tokenRepository-12 | トークンが無い | 照合値が同じで ID が別のトークン T と U を、並行して `insert` する | 一方だけが確定し、他方の `run` は `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）を投げる。`findByValueDigest` は確定した方を返す |
| TC-ports-tokenRepository-13 | トークンが無い | 一つの UnitOfWork で、照合値が同じで ID が別のトークン T と U を `insert` する | `run` が `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）を投げる。`findById(T.id)` と `findById(U.id)` は `null` |
| TC-ports-tokenRepository-14 | W1 の T が保存済み | W2 の区画で、T と同じ ID で、照合値が別のトークン U を `insert` する | `run` が確定する。W1 の区画の `findById(T.id)` は T を、W2 の区画の `findById(T.id)` は U を返す |

## findById

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-tokenRepository-15 | トークンが無い | 未使用の ID で `findById` を呼ぶ | `null` を返す |
| TC-ports-tokenRepository-17 | W2 の T が保存済み | W1 の区画で `findById(T.id)` を呼ぶ | `null` を返す。ほかのワークスペースのトークンを返さない |
| TC-ports-tokenRepository-18 | T を保存し、削除してある | `findById(T.id)` と `findByValueDigest(T.valueDigest)` を呼ぶ | どちらも `null` を返す |

## save

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-tokenRepository-19 | T が保存済み。`findById` で T と `v` を得ている | `save(Token.revoke(T, t1), v)` を呼ぶ | `findById(T.id)` が、`status` が `revoked`、`revokedAt` が t1、`version` が T より 1 大きいエンティティを返す |
| TC-ports-tokenRepository-20 | 範囲が紐付け待ちのモデル 1 件の T と、範囲が紐付け済みのモデル 1 件の U が保存済み。`findById` でそれぞれの `v` を得ている | T に、そのキーのモデルを作成した `ScopeChange`、U に、そのモデルを削除した `ScopeChange` で `Token.applyScopeChange` を適用し、それぞれの結果を `save` する | `findById` が、要素が紐付け済み（`ModelId`）になった T と、範囲が要素の無い `selection` の U を返す |
| TC-ports-tokenRepository-22 | ブラウザ用の T が保存済み。`findById` で T と `v` を得ている | `Token.changeRestrictions` で利用元の限定と利用量の制限を変えた結果を `save` する | `findById(T.id)` が、変更後の `profile.origins` と `profile.usageLimit`、T より 1 大きい `version` のエンティティを返す。`valueDigest` と `profile.protectedValue` は T と等しい |
| TC-ports-tokenRepository-23 | T が保存済み。`save` を一度確定してある | 確定の後に `findById` で得た `v` で、もう一度 `save` する | 確定する |
| TC-ports-tokenRepository-24 | T が保存済み。`findById` で T と `v` を得た後、別の UnitOfWork が `Token.revoke(T, t1)` を `save` して確定した | `save(Token.revoke(T, t2), v)` を呼ぶ | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById(T.id)` の `revokedAt` は t1 |
| TC-ports-tokenRepository-25 | 範囲が `selection` の T が保存済み。二つの呼び出し側が、同じ `v` を得ている | 失効の `save` と、範囲の更新の `save` を、並行して呼ぶ | 一方だけが確定し、他方の `run` は `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById(T.id)` は確定した方の内容を返す |
| TC-ports-tokenRepository-26 | T が保存済み。`recordUse(T.id, t1)` の後に `findById` で T と `v` を得て、その後 `recordUse(T.id, t2)` を呼んだ | `save(Token.revoke(T, t3), v)` を呼ぶ | 確定する。`findById(T.id)` の `status` は `revoked`。`findByValueDigest(T.valueDigest)` の `lastUsedAt` は t2 |
| TC-ports-tokenRepository-28 | T が保存済み。`findById` で T と `v` を得た後、別の UnitOfWork が `deleteAll(10)` を確定した | `save(Token.revoke(T, t1), v)` を呼ぶ | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById(T.id)` は `null` |

## insertMany

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-tokenRepository-29 | 無し | `insertMany([])` を呼び、確定する | `run` が確定する。何も保存しない |
| TC-ports-tokenRepository-30 | 無し | 照合値が互いに別の A、B、C を、一回の `insertMany` で保存し、確定する | `findById` は、A、B、C をそれぞれ、全フィールドが一致して返す。`findByValueDigest` は、それぞれのトークンを返す |
| TC-ports-tokenRepository-31 | 無し | A、B を `insertMany` した後、`fn` が例外を投げる | `findById(A.id)` と `findById(B.id)` は `null` |
| TC-ports-tokenRepository-32 | T が保存済み | T と同じ照合値の A と、照合値が別の B を、一回の `insertMany` に渡す | `run` が `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）を投げる。`findById(A.id)` と `findById(B.id)` は `null` |
| TC-ports-tokenRepository-33 | 無し | 照合値が同じ A、B を、一回の `insertMany` に渡す | `run` が `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）を投げる。どちらも保存しない |
| TC-ports-tokenRepository-34 | 無し | ID が同じで照合値が別の A、B を、一回の `insertMany` に渡す | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。どちらも保存しない |
| TC-ports-tokenRepository-35 | T が保存済み | T と同じ ID のトークンと、新しい B を、一回の `insertMany` に渡す | `run` が `ConflictError`（理由 `ALREADY_EXISTS`）を投げる。`findById(B.id)` は `null` |
| TC-ports-tokenRepository-36 | 無し | 範囲が紐付け待ちの要素を持つ、発行の経路が `bootstrap` のトークン 2 件を `insertMany` し、確定する | `findSelectionScoped()` は、その 2 件を返す |
| TC-ports-tokenRepository-37 | 無し | 2,000 件のトークンを、一回の `insertMany` で保存し、確定する | `run` が確定する。`findPage` の `count` は 2,000 |

## saveMany

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-tokenRepository-38 | 無し | `saveMany([])` を呼び、確定する | `run` が確定する。何も変わらない |
| TC-ports-tokenRepository-39 | 範囲が紐付け待ちの要素 1 件の A、B、C が保存済みで、`findSelectionScoped()` で読んでいる | それぞれに `Token.applyScopeChange` を適用した結果を、読んだ `expectedVersion` とともに、一回の `saveMany` に渡し、確定する | `findById` は、A、B、C を紐付け済みの範囲で返す |
| TC-ports-tokenRepository-40 | A、B が保存済みで、それぞれの `v` を得ている | 失効させた A と B を、一回の `saveMany` に渡し、確定する。その後、A を、前に得た `v` で `save` する | `saveMany` の `run` が確定する。続く `save` の `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。前に得た `v` は、`saveMany` が保存した `version` と違う |
| TC-ports-tokenRepository-41 | A、B、C が保存済みで、それぞれの `v` を得た後、別の UnitOfWork が C の失効の `save` を確定した | 範囲を変えた A、B、C を、前に得た `v` とともに、一回の `saveMany` に渡す | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById` は、A と B を変更前の範囲で、C を `revoked` で返す。どれも保存しない |
| TC-ports-tokenRepository-42 | 期限が t1 の A と、期限が `never` の B が保存済みで、それぞれの `v` を得た後、別の UnitOfWork が `deleteLapsed(t2, 10)` を確定して A を削除した | 変更した A と B を、一回の `saveMany` に渡す | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById(B.id)` は、変更前の B を返す |
| TC-ports-tokenRepository-43 | A、B が保存済みで、それぞれの `v` を得ている | 範囲を変えた A を二度と、B を、一回の `saveMany` に渡す | `run` が `SystemError`（理由 `INVALID_QUERY`）を投げる。`findById` は、A と B を変更前の内容で返す |
| TC-ports-tokenRepository-44 | A、B が保存済み | A、B を一回の `saveMany` に渡した後、`fn` が例外を投げる | `findById` は、A と B を変更前の内容で返す |
| TC-ports-tokenRepository-45 | A が保存済みで、`v` を得ている | 並行して、一方の UnitOfWork が、範囲を変えた A を `saveMany` に渡し、もう一方が、A の失効の `save` を `v` で呼ぶ | 一方だけが確定し、もう一方は `ConflictError`（理由 `STALE_VERSION`）を投げる |
| TC-ports-tokenRepository-46 | T が保存済みで、`v` を得た後、`recordUse(T.id, t2)` を呼んでいる | 範囲を変えた T を、`v` で `saveMany` に渡し、確定する | `run` が確定する。`findByValueDigest(T.valueDigest)` の `lastUsedAt` は t2 |
| TC-ports-tokenRepository-47 | 範囲が `selection` のトークンが 2,000 件保存済みで、`findSelectionScoped()` で読んでいる | 2,000 件のそれぞれの範囲を変え、一回の `saveMany` に渡し、確定する | `run` が確定する。`findSelectionScoped()` は、2,000 件を変更後の範囲で返す |

## findByValueDigest

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-tokenRepository-48 | トークンが無い | 未使用の照合値で `findByValueDigest` を呼ぶ | `null` を返す |
| TC-ports-tokenRepository-49 | T と、照合値が別の U が保存済み | `findByValueDigest(T.valueDigest)` を呼ぶ | T と `expectedVersion` を返す |
| TC-ports-tokenRepository-51 | `revoke` して保存した T と、t0 に発行した、期限が t1 の U が保存済み | それぞれの照合値で `findByValueDigest` を呼ぶ | どちらも、その状態のままトークンを返す |
| TC-ports-tokenRepository-53 | W2 の T が保存済み | W1 の区画と W2 の区画で、それぞれ `findByValueDigest(T.valueDigest)` を呼ぶ | W1 の区画は `null`、W2 の区画は T を返す |
| TC-ports-tokenRepository-55 | T が保存済み | `findByValueDigest` が返した `expectedVersion` で、`save(Token.revoke(T, t1), expectedVersion)` を呼ぶ | 確定する。`findById(T.id)` は `revoked` のエンティティを返す |
| TC-ports-tokenRepository-56 | T が保存済み。`findByValueDigest` で `expectedVersion` を得た後、別の UnitOfWork が T の失効の `save` を確定した | その `expectedVersion` で `save` を呼ぶ | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる |

## findPage: 対象

`filter` は `{}`、`pagination` は `{ page: 1, limit: 10 }` で呼ぶ。`items` の要素は `ListedToken`（`token`、`lastUsedAt`、`relatedMemberPresent`）。「`items` は T」は、`token` が T と等しい要素を指す。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-tokenRepository-57 | W1 にトークンが無い。区画に何も確定していないワークスペース Z | W1 の区画と Z の区画で、それぞれ `findPage({}, t1, pagination)` を呼ぶ | どちらも `items` は空、`count` は 0 |
| TC-ports-tokenRepository-58 | W1 に T が 1 件 | `findPage({}, t1, pagination)` を呼ぶ | `items` は T の 1 件、`count` は 1 |
| TC-ports-tokenRepository-59 | W1 に 2 件、W2 に 1 件 | `findPage({}, t1, pagination)` を呼ぶ | `items` は W1 の 2 件、`count` は 2 |
| TC-ports-tokenRepository-61 | W1 に、`active` の T と、`revoke` して保存した U がある | `findPage({}, t1, pagination)` を呼ぶ | `items` は T だけ、`count` は 1 |
| TC-ports-tokenRepository-62 | W1 に、期限が t1 のアプリに紐付く T がある | `now` を t2 にして呼ぶ。続けて、`now` を、t1 の `TokenReviewPolicy.expiredListedMs` 後にして呼ぶ | 一つ目の `items` は T を含む。二つ目の `items` は空、`count` は 0 |
| TC-ports-tokenRepository-64 | W1 に、期限が t2 のメンバーに紐付く T がある | `now` を t1、t2、t3 にして、それぞれ呼ぶ | t1 の `items` は T を含む。t2 と t3 の `items` は空、`count` は 0 |

## findPage: origins

W1 に、`origin` が `manual`、`bootstrap`、`device`、`preview` のトークンが 1 件ずつある。`now` は、どのトークンの期限よりも前にする。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-tokenRepository-68 | 上の 4 件 | `origins: ["manual"]`、続けて `origins: ["bootstrap"]` で呼ぶ | それぞれ、その `origin` の 1 件、`count` は 1 |
| TC-ports-tokenRepository-70 | 上の 4 件 | `origins: ["device", "preview"]`、続けて `origins` に 4 種類すべてを渡して呼ぶ | 一つ目の `items` は `device` と `preview` の 2 件、`count` は 2。二つ目の `items` は 4 件、`count` は 4 |
| TC-ports-tokenRepository-72 | 上の 4 件 | `origins: []` で呼ぶ | `items` は空、`count` は 0 |
| TC-ports-tokenRepository-73 | 上の 4 件 | `origins: ["manual", "manual"]` で呼ぶ | `items` は `manual` の 1 件、`count` は 1 |

## findPage: relatedMember

W1 に、発行者が M1 のトークンが 2 件、M2 のトークンが 1 件ある。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-tokenRepository-74 | 上の 3 件 | `relatedMember: M1`、続けて、トークンを持たないメンバーの ID を `relatedMember` に渡して呼ぶ | 一つ目の `items` は M1 の 2 件、`count` は 2。二つ目の `items` は空、`count` は 0 |
| TC-ports-tokenRepository-76 | 上の 3 件に加えて、M2 に紐付く `active` のトークン（`Token.issueForMember`、期限 t2）が 1 件。`now` は t1 | `relatedMember: M2` で呼ぶ | `items` は、発行者が M2 の 1 件と、M2 に紐付く 1 件、`count` は 2 |
| TC-ports-tokenRepository-77 | 上の 3 件。M1 のトークンのうち 1 件は `origin` が `bootstrap`、ほかは `manual` | `relatedMember: M1`、`origins: ["manual"]` で呼ぶ | `items` は M1 の `manual` の 1 件、`count` は 1 |

## findPage: expiringSoon

n は固定の日時。断りが無ければ、`now` は n、`filter` は `{ expiringSoon: true }` で呼ぶ。E は、n の `TokenReviewPolicy.expiringSoonMs` 後の日時。トークンは、n の 1 日前に発行する。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-tokenRepository-78 | 期限が `never` の T | `now` を n にして、`expiringSoon: true` で呼ぶ | T を含まない |
| TC-ports-tokenRepository-80 | 期限が n の 1 日後の T と、期限が E ちょうどの U | `now` を n にして、`expiringSoon: true` で呼ぶ | T と U を含む |
| TC-ports-tokenRepository-81 | 期限が E の 1 ミリ秒後の T | `now` を n にして、`expiringSoon: true` で呼ぶ | T を含まない |
| TC-ports-tokenRepository-82 | 期限が n の 1 日後の、アプリに紐付く T | `now` を、T の期限と同じ日時にして呼ぶ | T を含まない |
| TC-ports-tokenRepository-83 | 期限が n の 1 日後の、メンバーに紐付く T | `now` を n にして、`expiringSoon: true` で呼ぶ | T を含む |
| TC-ports-tokenRepository-84 | 期限が n の 1 日後の T と、期限が `never` の U | `filter` を `{}` にして呼ぶ | T と U を含む |

## findPage: longUnused

n は固定の日時。`now` は n、`filter` は `{ longUnused: true }` で呼ぶ。U は、n の `TokenReviewPolicy.unusedMs` 前の日時。U0 は、U より前の日時。断りが無ければ、期限は `never`。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-tokenRepository-85 | U ちょうどに発行し、一度も使われていない T | `now` を n にして、`longUnused: true` で呼ぶ | T を含む |
| TC-ports-tokenRepository-86 | U の 1 ミリ秒後に発行し、一度も使われていない T | `now` を n にして、`longUnused: true` で呼ぶ | T を含まない |
| TC-ports-tokenRepository-87 | U0 に発行し、`recordUse` で n の 1 日前に使った T と、U0 に発行し、`recordUse` で U ちょうどに使った V | `now` を n にして、`longUnused: true` で呼ぶ | V を含み、T を含まない |
| TC-ports-tokenRepository-89 | U0 に発行し、期限が n の 1 日後で、一度も使われていない T | `now` を n にして、`longUnused: true` で呼ぶ | T を含む |
| TC-ports-tokenRepository-90 | U0 に発行し、期限が n ちょうどで、一度も使われていない T | `now` を n にして、`longUnused: true` で呼ぶ | T を含まない |

## findPage: relatedMemberRemoved

`now` は t1、`filter` は、断りが無ければ `{ relatedMemberRemoved: true }`、`pagination` は `{ page: 1, limit: 10 }` で呼ぶ。メンバーの在籍は、同じ区画の `memberRepository` で作る。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-tokenRepository-91 | W1 に、発行者が M1 の A と、発行者が M2 の B がある。M1 は W1 に在籍し、M2 は在籍しない | `findPage({ relatedMemberRemoved: true }, t1, pagination)` を呼ぶ | `items` は B だけ、`count` は 1 |
| TC-ports-tokenRepository-93 | W1 に、M2 に紐付く、期限 t2 の T（`Token.issueForMember`）がある。M2 は W1 に在籍しない | `findPage({ relatedMemberRemoved: true }, t1, pagination)` を呼ぶ | `items` は T だけ、`count` は 1 |
| TC-ports-tokenRepository-94 | W1 に、発行者が M1 の A がある。M1 は W1 に在籍する | `memberRepository.deleteAll(10)` を確定して M1 を削除してから、`findPage({ relatedMemberRemoved: true }, t1, pagination)` を呼ぶ | `items` は A だけ、`count` は 1 |
| TC-ports-tokenRepository-95 | W1 に、発行者が M1 の A がある。M1 は、W2 に在籍し、W1 に在籍しない | W1 の区画で `findPage({ relatedMemberRemoved: true }, t1, pagination)` を呼ぶ | `items` は A だけ、`count` は 1 |
| TC-ports-tokenRepository-96 | W1 に、発行者が M2（在籍しない）で、`revoke` して保存した A がある | `findPage({ relatedMemberRemoved: true }, t1, pagination)` を呼ぶ | `items` は空、`count` は 0。一覧に載らないトークンを含めない |
| TC-ports-tokenRepository-97 | W1 に、発行者がどれも在籍しない別のメンバーのトークンが 3 件ある | `findPage({ relatedMemberRemoved: true }, t1, { page: 2, limit: 2 })` を呼ぶ | `items` は並び順の 3 件目だけ、`count` は 3 |
| TC-ports-tokenRepository-98 | W1 に、発行者が M2（在籍しない）の、`manual` の A と `bootstrap` の B がある | `findPage({ relatedMemberRemoved: true, origins: ["manual"] }, t1, pagination)` を呼ぶ | `items` は A だけ、`count` は 1 |
| TC-ports-tokenRepository-99 | W1 に、発行者が M1 の A と、発行者が M2 の B と、M1 に紐付く、期限 t2 の T（`Token.issueForMember`）がある。M1 は W1 に在籍し、M2 は在籍しない | `findPage({}, t1, pagination)` を呼ぶ | `items` は A、B、T、`count` は 3。A と T の要素の `relatedMemberPresent` は真、B の要素は偽 |

## findPage: pendingScopeOnly

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-tokenRepository-101 | W1 に、範囲が `workspace` の A、紐付け済みの要素だけの B、紐付け待ちの要素だけの C、両方を持つ D がある | `pendingScopeOnly: true` で呼ぶ | `items` は C と D、`count` は 2 |
| TC-ports-tokenRepository-103 | 紐付け待ちの要素 1 件の C を、`Token.applyScopeChange` で紐付け済みにして `save` を確定した | `pendingScopeOnly: true` で呼ぶ | `items` は C を含まない |

## findPage: 条件の組み合わせ

n は固定の日時。`now` は n。トークンは、n の `TokenReviewPolicy.unusedMs` 前より前の日時に発行する。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-tokenRepository-104 | W1 に、A（`manual`、期限が n の 1 日後）、B（`bootstrap`、期限が n の 1 日後）、C（`manual`、期限 `never`）がある | `origins: ["manual"]`、`expiringSoon: true` で呼ぶ | `items` は A だけ、`count` は 1 |
| TC-ports-tokenRepository-105 | W1 に、期限が n の 1 日後の A（未使用）、期限が n の 1 日後の B（`recordUse` で n の 1 日前に使用）、期限が `never` の C（未使用）がある | `expiringSoon: true`、`longUnused: true` で呼ぶ | `items` は A だけ、`count` は 1 |
| TC-ports-tokenRepository-106 | W1 に、A（`bootstrap`、範囲に紐付け待ちの要素を持つ）、B（`bootstrap`、範囲が `workspace`）、C（`manual`、範囲が紐付け済みの要素だけ）がある | `origins: ["bootstrap"]`、`pendingScopeOnly: true` で呼ぶ | `items` は A だけ、`count` は 1 |
| TC-ports-tokenRepository-107 | 上の各行の前提 | 上の各行の呼び出しで返った `items` | すべての要素で、要素の `token`、`lastUsedAt`、`relatedMemberPresent` を渡した `TokenListFilter.matches(token, filter, { now, lastUsedAt, relatedMemberPresent })` が真。W1 のトークンのうち `items` に無いものは、`matches` が偽 |

## findPage: 並び順とページング

`filter` は、断りが無ければ `{}`。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-tokenRepository-108 | W1 に、発行日時が t0、t1、t2 の 3 件を、この順と違う順で `insert` してある | `{ page: 1, limit: 10 }` で呼ぶ | `items` は発行日時が t2、t1、t0 の順 |
| TC-ports-tokenRepository-109 | W1 に、発行日時が同じ 2 件がある | `{ page: 1, limit: 10 }` で呼ぶ | `items` は ID の降順 |
| TC-ports-tokenRepository-110 | W1 に 3 件 | `{ page: 1, limit: 3 }` で呼ぶ | `items` は 3 件、`count` は 3 |
| TC-ports-tokenRepository-111 | W1 に 3 件 | `{ page: 2, limit: 3 }` で呼ぶ | `items` は空、`count` は 3 |
| TC-ports-tokenRepository-112 | W1 に 4 件 | `{ page: 1, limit: 3 }`、`{ page: 2, limit: 3 }`、`{ page: 4, limit: 1 }` で、それぞれ呼ぶ | 一つ目の `items` は並び順の先頭 3 件、二つ目と三つ目は並び順の 4 件目だけ。どれも `count` は 4 |
| TC-ports-tokenRepository-115 | W1 に、`manual` が 3 件、`bootstrap` が 2 件 | `origins: ["manual"]`、`{ page: 1, limit: 2 }` で呼ぶ | `items` は `manual` の並び順の先頭 2 件、`count` は 3 |

## findSelectionScoped

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-tokenRepository-116 | W1 にトークンが無い。または、W1 に、範囲が `workspace` のアプリに紐付くトークンだけ、またはメンバーに紐付くトークンだけがある | `findSelectionScoped()` を呼ぶ | どの場合も空の配列を返す |
| TC-ports-tokenRepository-119 | W1 に、範囲が紐付け済みの要素の `selection` の T、範囲が紐付け待ちの要素の `selection` の U、範囲が `workspace` の V がある | `findSelectionScoped()` を呼ぶ | T と U を、それぞれの `expectedVersion` とともに返す。V を含まない。順序は問わない |
| TC-ports-tokenRepository-121 | W1 に、`Token.applyScopeChange` ですべての要素を除いて保存した、範囲が空の T がある | `findSelectionScoped()` を呼ぶ | T の 1 件を返す |
| TC-ports-tokenRepository-122 | W1 に、t0 に発行した、期限が t1 で範囲が `selection` の T がある | `findSelectionScoped()` を呼ぶ | T の 1 件を返す |
| TC-ports-tokenRepository-123 | W1 に、範囲が `selection` で、`revoke` して保存した T がある | `findSelectionScoped()` を呼ぶ | 空の配列を返す |
| TC-ports-tokenRepository-124 | W2 に、範囲が `selection` の T がある | W1 の区画で `findSelectionScoped()` を呼ぶ | 空の配列を返す |
| TC-ports-tokenRepository-126 | W1 に、範囲が `selection` の T がある | 返された `expectedVersion` で、`Token.applyScopeChange` の結果を `save` する | 確定する。`findById(T.id)` は更新後の範囲を返す |

## recordUse

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-tokenRepository-127 | T が保存済み。`recordUse` を呼んでいない | UnitOfWork の外で `recordUse(T.id, t1)` を呼ぶ | `findByValueDigest(T.valueDigest)` の `lastUsedAt` は t1。`findPage` が返す T の `ListedToken` の `lastUsedAt` は t1 |
| TC-ports-tokenRepository-128 | T に `recordUse(T.id, t2)` を呼んである | `recordUse(T.id, t1)`、`recordUse(T.id, t2)`、`recordUse(T.id, t3)` を、この順に呼ぶ | `findByValueDigest(T.valueDigest)` の `lastUsedAt` は、t1 と t2 の後は t2、t3 の後は t3 |
| TC-ports-tokenRepository-131 | T が保存済み。`version` は 0 | `recordUse(T.id, t1)` を呼ぶ | `findById(T.id)` の `version` は 0 |
| TC-ports-tokenRepository-132 | T が保存済み。`recordUse` を呼んでいない | `recordUse(T.id, t1)`、`recordUse(T.id, t3)`、`recordUse(T.id, t2)` を並行して呼ぶ | すべての呼び出しが終わった後、`findByValueDigest(T.valueDigest)` の `lastUsedAt` は t3 |
| TC-ports-tokenRepository-133 | トークンが無い | 未使用の ID で `recordUse` を呼ぶ | エラーにならない。その ID の `findById` は `null` |
| TC-ports-tokenRepository-134 | T に `recordUse(T.id, t1)` を呼んだ後、`deleteAll(10)` で T を削除してある | T と同じ ID と照合値のトークンを `insert` し、確定する | `findByValueDigest(T.valueDigest)` の `lastUsedAt` は `null` |

## deleteLapsed

保持期間の起点は `Token.lapsedAt`（[token.md](../../domains/token.md) の「Token」）。対象と順序は [common.md](../../domains/common.md) の「期限つきの一度きりの値」が定める。トークンは t0 に発行する。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-tokenRepository-135 | トークンが無い | `deleteLapsed(t2, 10)` を呼ぶ | 0 を返す |
| TC-ports-tokenRepository-136 | 期限が `never` で `active` の U と、期限が t1 の T | `deleteLapsed(t2, 10)` を呼ぶ | 1 を返す。`findById(U.id)` は U を、`findById(T.id)` は `null` を返す |
| TC-ports-tokenRepository-137 | `active` の、期限が t1 の T、期限が t2 の U、期限が t3 の V と、期限が t1 の、メンバーに紐付く M | `deleteLapsed(t2, 10)` を呼ぶ | 3 を返す。`findById` は T、U、M に `null`、V に V を返す。`findByValueDigest(T.valueDigest)` も `null` |
| TC-ports-tokenRepository-140 | 期限が `never` で、`revoke` した T（t1）、U（t2）、V（t3） | `deleteLapsed(t2, 10)` を呼ぶ | 2 を返す。`findById` は T と U に `null`、V に V を返す |
| TC-ports-tokenRepository-143 | 期限が t3 で `revoke`（t1）して保存した T と、期限が t1 で `revoke`（t3）して保存した U | `deleteLapsed(t2, 10)` を呼ぶ | 2 を返す。どちらも `findById` は `null` |
| TC-ports-tokenRepository-146 | 期限が t1 の、W1 の T と W2 の U | W1 の区画で `deleteLapsed(t2, 10)` を呼ぶ | 1 を返す。W1 の区画の `findById(T.id)` は `null`。W2 の区画の `findById(U.id)` は U を返す |
| TC-ports-tokenRepository-148 | 期限が t1、t2、t3 の T、U、V | `deleteLapsed(t3, 2)` を呼ぶ | 2 を返す。`findById` は T と U に `null`、V に V を返す |
| TC-ports-tokenRepository-149 | 期限が t1 の 3 件 | `deleteLapsed(t2, 2)` を呼ぶ | 2 を返す。`findById` は、ID の昇順の先頭 2 件に `null`、3 件目にトークンを返す |
| TC-ports-tokenRepository-150 | 期限が t1 の 4 件 | `deleteLapsed(t2, 3)` を三度続けて、別の UnitOfWork で呼ぶ | 順に 3、1、0 を返す。4 件とも `findById` は `null` |
| TC-ports-tokenRepository-153 | 期限が t1 の 2 件 | `deleteLapsed(t2, BoundedDeletion.maxLimit)` を呼ぶ | 2 を返す |
| TC-ports-tokenRepository-154 | 期限が t1 の T | `deleteLapsed(t2, 0)` と、`BoundedDeletion.maxLimit` を超える `limit` の `deleteLapsed(t2, limit)` を、それぞれ呼ぶ | どちらも `SystemError`（理由 `INVALID_QUERY`）になる。`findById(T.id)` は T を返す |
| TC-ports-tokenRepository-156 | 期限が t1 の T。`findById` で `v` を得た後、別の UnitOfWork が T の失効の `save` を確定した | `deleteLapsed(t2, 10)` を呼ぶ | 1 を返す。`findById(T.id)` は `null` |

## deleteAll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-tokenRepository-157 | W1 にトークンが無い | `deleteAll(10)` を呼ぶ | 0 を返す |
| TC-ports-tokenRepository-158 | W1 に、`active` の 1 件、`revoke` した 1 件、期限を過ぎた 1 件 | `deleteAll(10)` を呼ぶ | 3 を返す。3 件とも `findById` は `null` |
| TC-ports-tokenRepository-159 | W1 に 4 件 | `deleteAll(3)` を三度続けて、別の UnitOfWork で呼ぶ | 順に 3、1、0 を返す。一回目の後は 4 件のうち 1 件だけ `findById` が返る。三回目の後は 4 件とも `null` |
| TC-ports-tokenRepository-162 | W1 に 2 件、W2 に 1 件 | `deleteAll(10)` を呼ぶ | 2 を返す。W2 の 1 件は、W2 の区画の `findById` が返る |
| TC-ports-tokenRepository-164 | W1 に 2 件 | `deleteAll(BoundedDeletion.maxLimit)` を呼ぶ | 2 を返す |
| TC-ports-tokenRepository-165 | W1 に 2 件 | `deleteAll(0)` と、`BoundedDeletion.maxLimit` を超える `limit` の `deleteAll` を、それぞれ呼ぶ | どちらも `SystemError`（理由 `INVALID_QUERY`）になる。2 件とも `findById` が返る |

## UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-tokenRepository-167 | B と、期限が t1 の C が保存済み | 一つの UnitOfWork で、A の `insert`、B の失効の `save`、`deleteLapsed(t2, 10)` を呼び、確定する | `findById` は、A を返し、B を `revoked` で返し、C に `null` を返す |
| TC-ports-tokenRepository-168 | B と、期限が t1 の C が保存済み | 一つの UnitOfWork で、同じ 3 つの書き込みを呼んだ後、`fn` が例外を投げる | `findById` は、A に `null`、B を `active` で、C をそのまま返す |
| TC-ports-tokenRepository-169 | B が保存済み | 一つの UnitOfWork で、A の `insert` と、B と同じ照合値の D の `insert` を呼ぶ | `run` が `ConflictError`（理由 `SECRET_DIGEST_TAKEN`）を投げる。`findById(A.id)` と `findById(D.id)` は `null` |
| TC-ports-tokenRepository-170 | B が保存済み。`findById` で `v` を得た後、別の UnitOfWork が B の失効の `save` を確定した | 一つの UnitOfWork で、A の `insert` と、`v` での B の `save` を呼ぶ | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById(A.id)` は `null` |
| TC-ports-tokenRepository-171 | W1 に 2 件 | 一つの UnitOfWork の中で、`deleteAll(10)` を呼んだ後、`fn` が例外を投げる | 2 件とも `findById` が返る |
| TC-ports-tokenRepository-172 | W1 に、範囲が紐付け待ちの要素 1 件の B と C がある | 一つの UnitOfWork の中で、`findSelectionScoped()` の結果のそれぞれに `Token.applyScopeChange` を適用し、一回の `saveMany` で保存して、確定する | `findById` は、B と C を紐付け済みの範囲で返す |
| TC-ports-tokenRepository-173 | W1 に、範囲が紐付け待ちの要素 1 件の B と C がある。`findSelectionScoped()` で読んだ後、別の UnitOfWork が C の失効の `save` を確定した | 一つの UnitOfWork で、読んだ B と C のそれぞれに `Token.applyScopeChange` を適用し、一回の `saveMany` で保存する | `run` が `ConflictError`（理由 `STALE_VERSION`）を投げる。`findById(B.id)` の範囲は紐付け待ちのまま |
| TC-ports-tokenRepository-175 | 期限が t1 の 2 件 | 一つの UnitOfWork の中で、`deleteLapsed(t2, 10)` を呼んだ後、`fn` が例外を投げる | 2 件とも `findById` が返る |
