# アプリ向けの HTTP API の入口

`apps/web` のアプリ向けの HTTP API の入口と、メディアの配信の入口が持つ規則の代表ケース。契約は [app-api.md](../../usecases/app-api.md) の「要求と応答の形」「トークン」「スキーマの識別子」「エラー」「入口の理由コード」「利用できないワークスペース」「フォームの受付結果」「経路の命名」「要求とユースケースの対応」「メディアの配信」「一覧の指定」「管理の一覧の指定」「参照の展開の指定」が定める。ユースケースの判定は、各ユースケースのテストが持つ。ここでは、ユースケースの結果を応答へ写す規則だけを確かめる。断りが無い限り、ユースケース（`authenticateToken` を含む）の結果は、試験用のユースケースが返す。

## トークンと利用元

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-app-api-httpEntry-01 | なし | `Authorization` ヘッダーに `Bearer` 方式で値 v を、`Origin` ヘッダーに `https://app.example` を示して、公開取得を呼ぶ | `authenticateToken` に、トークンの値 v と、利用元 `https://app.example` を渡す |
| TC-app-api-httpEntry-02 | なし | `Authorization` ヘッダーにトークンを示し、`Origin` ヘッダーを持たない要求で、公開取得を呼ぶ | `authenticateToken` に、利用元として `null` を渡す |
| TC-app-api-httpEntry-03 | `authenticateToken` は、アプリに紐付くトークンと、発行元のワークスペースの ID を返す | トークンを示して、公開取得を呼ぶ | 取得のユースケースに、`appToken` の `Actor` と、そのワークスペースの ID を渡す |
| TC-app-api-httpEntry-04 | なし | 事前確認の要求（`OPTIONS`）で、公開取得の経路を呼ぶ | `authenticateToken` を呼ばずに、`Authorization`、`Content-Type`、`X-Lane-Fingerprints` ヘッダーを許可して応える |
| TC-app-api-httpEntry-05 | なし | `Authorization` ヘッダーを持たない要求で、本文 `{ code }` を持つ `POST /api/v1/preview-exchanges/redeem` を呼ぶ | `authenticateToken` を呼ばずに、`redeemPreviewExchange` に引き換えの値を渡す |

## スキーマの識別子

f1、f2 は `SchemaFingerprint` の形の値。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-app-api-httpEntry-06 | `listPublishedContents` は、出力と、識別子ごとの一致（`article` は一致、`author` は不一致）を返す | トークンと、`X-Lane-Fingerprints: model.article=f1, model.author=f2` を示して、公開取得の一覧を呼ぶ | `listPublishedContents` に、モデル `article` の識別子 f1 と、モデル `author` の識別子 f2 を持つ `ClaimedFingerprints` を渡す。状態 200。応答の `fingerprints` は、二つの対象の `FingerprintMatch`（`article` の `matches` は真、`author` は偽）を持つ。`data` は、不一致によって変わらない |
| TC-app-api-httpEntry-07 | `listPublishedContents` は、出力を返す | トークンを示し、`X-Lane-Fingerprints` ヘッダーを持たない要求で、公開取得の一覧を呼ぶ | `listPublishedContents` に、識別子を添えない要求として渡す。応答の `fingerprints` は空の配列 |
| TC-app-api-httpEntry-08 | `saveDraft` は、出力を返す | トークンと、`X-Lane-Fingerprints: schema=f1` を示して、下書きの保存を呼ぶ | 入口は、ヘッダーを読まない。応答は `fingerprints` を持たない |
| TC-app-api-httpEntry-09 | `listPublishedContents` は、`NotFoundError` を返す | トークンと、`X-Lane-Fingerprints: schema=f1` を示して、公開取得の一覧を呼ぶ | 状態 404 のエラーの形。応答は `fingerprints` を持たない |

## エラーの形

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-app-api-httpEntry-10 | `revokeOwnToken` は、出力を持たずに正常に終わる | メンバーに紐付くトークンを示して `POST /api/v1/token/revoke` を呼ぶ | 状態 200。本文は成功の形で、`data` は `null` |
| TC-app-api-httpEntry-12 | ユースケースは、対象の種類がコンテンツで、キーが k の `NotFoundError` を返す | トークンを示して、単件の公開取得を呼ぶ | 状態 404。本文はエラーの形で、`kind` は `notFound`。`details` は、`type` が `content`、`keys` が k だけの列 |
| TC-app-api-httpEntry-13 | ユースケースは、理由コード c の `BusinessRuleError` を返す | トークンを示して、コンテンツ管理の経路を呼ぶ | 状態 422。`kind` は `business`、`code` は c |
| TC-app-api-httpEntry-14 | ユースケースは、値の位置（項目 `sections` の 0 番目の `title`）に理由コード r を持つ `ValidationError` を返す | トークンを示して、下書きの保存を呼ぶ | 状態 422。`kind` は `validation`。`fieldErrors` は、キー `sections.0.title` に r を持つ |
| TC-app-api-httpEntry-15 | `rewriteValues` は、コンテンツ C の値の位置 `title` に理由コード r を持つ `ValidationError` を返す | トークンを示して、`POST /api/v1/manage/contents/rewrite` を呼ぶ | 状態 422。`fieldErrors` は、キー `C:title`（C はコンテンツID）に r を持つ |
| TC-app-api-httpEntry-17 | ユースケースは、不足する権限を添えた `ForbiddenError`（`PERMISSION_MISSING`）を返す | トークンを示して、公開を呼ぶ | 状態 403。`kind` は `forbidden`。`details` は、不足する権限を持つ |
| TC-app-api-httpEntry-18 | `authenticateToken` は、再び試せる時刻 t の `RateLimitedError` を返す | トークンを示して、公開取得を呼ぶ | 状態 429。`kind` は `rateLimited`、`retryAt` は t。`Retry-After` ヘッダーは、t までの秒数を持つ。取得のユースケースを呼ばない |
| TC-app-api-httpEntry-20 | ユースケースは、内部の理由を持つ `SystemError` を返す | トークンを示して、公開取得を呼ぶ | 状態 500。`kind` は `system`、`code` は `null`、`message` は固定の文言。応答は、内部の理由を含まない |
| TC-app-api-httpEntry-21 | ユースケースは、エラーの種類のどれでもない失敗を投げる | トークンを示して、公開取得を呼ぶ | 状態 500。`kind` は `unknown`、`code` は `null`、`message` は固定の文言 |
| TC-app-api-httpEntry-22 | なし | `Authorization` ヘッダーを持たない要求で、公開取得を呼ぶ | 状態 401。`kind` は `unauthenticated`、`code` は `TOKEN_UNKNOWN` |
| TC-app-api-httpEntry-23 | なし | トークンを示して、`IdGenerator` の形式に合わない `id` を本文に持つ `POST /api/v1/manage/models/article/contents` と、`jobId` を持たない本文の `POST /api/v1/schema/applies` を呼ぶ | どちらも状態 422。`kind` は `validation`。`fieldErrors` は、それぞれキー `id` とキー `jobId` に `InvalidId` を持つ。`createContent` と `requestSchemaApply` を呼ばない |
| TC-app-api-httpEntry-73 | なし | トークンを示して、対象の列の一つの要素のコンテンツID が空の文字列の本文で `POST /api/v1/manage/contents/publish` を呼ぶ | 状態 422。`kind` は `validation`、`code` は `null`。`fieldErrors` は、その ID の本文の位置のキーに `InvalidId` を持つ。`publishContent` を呼ばない |
| TC-app-api-httpEntry-24 | なし | トークンを示して、`location` と `model` を指定せずに `GET /api/v1/manage/contents` を呼ぶ | 状態 422。`kind` は `validation`、`code` は `null`。`fieldErrors` は、キー `model` に `modelRequired` を持つ。`listContents` を呼ばない |
| TC-app-api-httpEntry-25 | なし | トークンを示して、クエリ `model=article`、`publication=draft` を持つ `GET /api/v1/manage/contents` を呼ぶ | 状態 422。`kind` は `validation`。`fieldErrors` は、キー `publication` に `invalidQueryValue` を持つ。`listContents` を呼ばない |
| TC-app-api-httpEntry-27 | なし | トークンと、同じ対象を二度書いた `X-Lane-Fingerprints` ヘッダーを示して、`GET /api/v1/published/models/article/contents` を呼ぶ | 状態 422。`kind` は `validation`。`fieldErrors` は、キー `X-Lane-Fingerprints` に `duplicateTarget` を持つ。`listPublishedContents` を呼ばない |
| TC-app-api-httpEntry-28 | ユースケースは、上限を添えた `ValidationError`（`InvalidTargetCount`）を返す | トークンを示して、対象の列を持つ公開を呼ぶ | 状態 422。`kind` は `validation`。`fieldErrors` のキーは、列のプロパティの名前。`details` は、上限の値（`limit`）を持つ |

## 利用できないワークスペース

入口ごとの結果は、[index.md](../../domains/index.md) の「利用できるワークスペース」の表が定め、そのテストは定義元のテストが持つ。ここでは、フォームの送信の応答の形だけを確かめる。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-app-api-httpEntry-30 | `authenticateToken` は、利用できないワークスペースの `NotFoundError` を返す | トークンを示して、フォームの送信を呼ぶ | 状態 404 のエラーの形。受付結果の形にしない。`acceptSubmission` を呼ばない |

## フォームの受付結果

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-app-api-httpEntry-31 | `acceptSubmission` は、受付完了と、識別子の照合の結果を返す | トークンと識別子を示して、`{ values }` を本文に持つ送信を呼ぶ | 状態 200。`data` は `{ outcome: "accepted" }`。`fingerprints` は、ユースケースが返した照合の結果 |
| TC-app-api-httpEntry-32 | `acceptSubmission` は、理由 `closed` の受付失敗を返す | トークンを示して、送信を呼ぶ | 状態 200 の成功の形。`data` は、`outcome` が `rejected`、`rejection.reason` が `closed` |
| TC-app-api-httpEntry-33 | `authenticateToken` は、再び試せる時刻 t の `RateLimitedError` を返す | トークンと識別子を示して、送信を呼ぶ | 状態 200 の成功の形。`data` は、`outcome` が `rejected`、`rejection` が `{ reason: "rateLimited", retryAt: t }`。`fingerprints` は空の配列。エラーの形にしない |
| TC-app-api-httpEntry-34 | `acceptSubmission` は、再び試せる時刻 t の `RateLimitedError`（`SUBMISSIONS_LIMITED`）を返す | トークンを示して、送信を呼ぶ | 状態 200 の成功の形。`rejection` は `{ reason: "rateLimited", retryAt: t }` |
| TC-app-api-httpEntry-35 | `acceptSubmission` は、`ConflictError`（`SCHEMA_CHANGED`）を返す | トークンを示して、送信を呼ぶ | 状態 200 の成功の形。`rejection.reason` は `busy` |
| TC-app-api-httpEntry-36 | `acceptSubmission` は、`SystemError` を返す | トークンを示して、送信を呼ぶ | 状態 200 の成功の形。`rejection.reason` は `storageFailed` |
| TC-app-api-httpEntry-37 | `acceptSubmission` は、`ForbiddenError`（`GRANT_MISSING`）を返す | トークンを示して、送信を呼ぶ | 状態 403 のエラーの形。受付結果の形にしない |
| TC-app-api-httpEntry-38 | `acceptSubmission` は、エラーの種類のどれでもない失敗を投げる | トークンを示して、送信を呼ぶ | 状態 500 のエラーの形。`kind` は `unknown`。受付結果の形にしない |
| TC-app-api-httpEntry-39 | `authenticateToken` は、`SystemError` を返す | トークンを示して、送信を呼ぶ | 状態 500 のエラーの形。`kind` は `system`。受付結果の形にしない（`storageFailed` にしない）。`acceptSubmission` を呼ばない |
| TC-app-api-httpEntry-40 | なし | トークンを示して、`values` のほかに `formId` と `recipient` のプロパティをこの順に持つ本文で、送信を呼ぶ | `acceptSubmission` に、`conditionKeys` として `formId`、`recipient` の列を渡す。二つのプロパティの値を渡さない |
| TC-app-api-httpEntry-42 | なし | トークンを示して、JSON の配列を本文に持つ送信と、`values` が JSON の配列の本文の送信を呼ぶ | どちらも状態 422。`kind` は `validation`。`acceptSubmission` を呼ばない |
| TC-app-api-httpEntry-44 | なし | トークンを示して、`values` の項目 `tags` の値が数の配列の本文で、送信を呼ぶ | `acceptSubmission` に、`tags` の値を数の配列のまま渡す。`validation` にしない |

## 操作の ID

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-app-api-httpEntry-46 | `publishContent` は、入力に受けた操作の ID を含む出力を返す | トークンを示して、`POST /api/v1/manage/contents/publish` を二度呼ぶ | 入口は、要求ごとに `IdGenerator.generate` で作った、互いに違う操作の ID を `publishContent` に渡す。二つの応答の `data` は、操作の ID と通し番号を持たない |
| TC-app-api-httpEntry-47 | なし | トークンを示して、`id` を省いた本文で `POST /api/v1/manage/models/article/contents` を呼ぶ | `createContent` に、入口が `IdGenerator.generate` で作ったコンテンツID を渡す |

## 要求からユースケースの入力への写し方

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-app-api-httpEntry-48 | なし | トークンを示して、`GET /api/v1/published/models/article/unique/slug/a%2Fb` を呼ぶ | `getPublishedContent` に、モデルのキー `article`、一意な項目のキー `slug`、値 `a/b` を渡す |
| TC-app-api-httpEntry-49 | なし | トークンを示して、`GET /api/v1/published/models/profile/single` を呼ぶ | `getPublishedContent` に、単一モデルのキー `profile` を渡す |
| TC-app-api-httpEntry-50 | なし | トークンを示して、本文 `{ values }` で `POST /api/v1/manage/models/article/contents` を呼ぶ | `createContent` に、入力の帰属（`InputSource`）として `direct` を渡す |
| TC-app-api-httpEntry-51 | なし | トークンを示して、`POST /api/v1/manage/contents/C/restore`（C はコンテンツID）を呼ぶ | `restoreTrashedContent` に、コンテンツID C を渡す |
| TC-app-api-httpEntry-52 | なし | トークンを示して、履歴の版を持たない本文で `POST /api/v1/manage/contents/purge` を呼ぶ | `purgeContent` に、対象に加える履歴の版として「指定なし」を渡す |
| TC-app-api-httpEntry-53 | なし | トークンを示して、`file` の部分にファイルの内容と元のファイル名を持つ `multipart/form-data` の本文で `POST /api/v1/manage/media` を呼ぶ | `registerMedia` に、ファイルの内容と元のファイル名と、受け付ける種類 `any` を渡す |
| TC-app-api-httpEntry-54 | なし | トークンを示して、`IdGenerator` の形式に合う `jobId` と、`baseSchemaVersion`、`schema`、`migrations` を本文に持つ `POST /api/v1/schema/applies` を呼ぶ | `requestSchemaApply` に、`jobId` から作った `SchemaApplyJobId`、`baseSchemaVersion`、境界のデコードで値にした `Schema` と `Migration` の列を渡す |
| TC-app-api-httpEntry-55 | なし | トークンを示して、`lossAcknowledged` を持たない本文で `POST /api/v1/schema/direct-applies` を呼ぶ | `requestDirectSchemaApply` に、`lossAcknowledged` として `false` を渡す |
| TC-app-api-httpEntry-56 | `startDeviceAuthorization` は、承認の値、確認用のコード、受け取りの値、依頼の期限を返す | 本文 `{ grants }` で `POST /api/v1/device-authorizations` を呼ぶ | `data` は、確認用のコード、受け取りの値、依頼の期限と、承認の値から `DeviceApprovalLink.url` で組み立てた承認用の URL を持つ。`data` は、承認の値を、承認用の URL の外に持たない |

## 一覧と展開の指定

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-app-api-httpEntry-57 | なし | トークンを示して、クエリ `filter=price:range:100..`、`filter=title:eq:a:b`、`sort=field:price:desc`、`page=2`、`limit=5` をこの順に持つ `GET /api/v1/published/models/article/contents` を呼ぶ | `listPublishedContents` に、位置 0 の条件（項目 `price`、`range`、`min` は表記 `100`、`max` は `null`）、位置 1 の条件（項目 `title`、`equals`、値は表記 `a:b`）、並び替え（項目 `price` の降順）、`page` 2、`limit` 5 を渡す。値は、表記の文字列のまま渡す |
| TC-app-api-httpEntry-58 | なし | トークンを示して、クエリを持たない `GET /api/v1/published/models/article/contents` を呼ぶ | `listPublishedContents` に、条件なし、並び替え `createdAt` の降順、`page` 1、`limit` は APP-12 の「一覧の 1 ページの件数」の、件数を指定しない取得の値、展開の深さ 0 を渡す |
| TC-app-api-httpEntry-59 | `listPublishedContents` は、位置 1 の条件に理由 `valueKindMismatch` を持つ `ValidationError` を返す | トークンを示して、条件を二つ指定した公開取得の一覧を呼ぶ | 状態 422。`kind` は `validation`。`fieldErrors` は、キー `filter.1` に `valueKindMismatch` を持つ |
| TC-app-api-httpEntry-72 | `listPublishedContents` は、`subject` が `filters` で上限の値 f の `tooManyFilters` と、`subject` が `depth` で上限の値 d の `depthExceeded` を持つ `ValidationError` を返す | トークンを示して、公開取得の一覧を呼ぶ | 状態 422。`kind` は `validation`。`fieldErrors` は、キー `filter` に `tooManyFilters`、キー `depth` に `depthExceeded` を持つ。`details` の `limits` は、キー `filter` に f、キー `depth` に d を持つ |
| TC-app-api-httpEntry-60 | なし | トークンを示して、クエリ `expand=article:author` を持ち、`depth` を持たない単件の公開取得を呼ぶ | `getPublishedContent` に、位置 0 の展開の指定（モデル `article`、項目への経路 `author`）と、APP-12 の「展開する参照の深さ」の、深さを指定しない取得の値を渡す |
| TC-app-api-httpEntry-61 | なし | トークンを示して、クエリ `location=trashed` を持つ `GET /api/v1/manage/contents` を呼ぶ | `listTrashedContents` を呼ぶ。`listContents` を呼ばない |
| TC-app-api-httpEntry-62 | なし | トークンを示して、クエリ `model=article`、`search=hello`、`publication=published`、`hasDraft=true` を持つ `GET /api/v1/manage/contents` を呼ぶ | `listContents` に、モデルのキー `article`、検索語 `hello`、状態の絞り込み（`publication` は `published`、`hasDraft` は真、`draftHasIssues` と `publishedHasViolations` は `null`）を渡す |
| TC-app-api-httpEntry-63 | `listPreviewContents` は、一覧、総件数、結果に含めなかった対象を返す | トークンを示して、プレビュー取得の一覧を呼ぶ | `data` は、`items`、`count`、`excluded` を持つ。公開取得の一覧の `data` は、`items` と `count` を持ち、`excluded` を持たない |

## 参照の位置ごとの形

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-app-api-httpEntry-64 | `getPublishedContent` は、参照の位置ごとの形を `ContentDelivery.arrange` で並べた出力を返す | トークンを示して、単件の公開取得を呼ぶ | `data` は、その出力の JSON。入口は、参照の位置を置き換えない |

## メディアの配信

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-app-api-httpEntry-65 | `deliverMedia` は、実体と、`retention` が `long`（`maxAgeSeconds` は n）の結果を返す | `GET /media/{配信先}` を呼ぶ | 状態 200。本文は実体の内容。`Content-Type` は実体の種類。`Cache-Control` は `public, max-age=n, immutable` |
| TC-app-api-httpEntry-66 | `deliverMedia` は、実体と、`retention` が `none` の結果を返す | `GET /media/{配信先}` を呼ぶ | 状態 200。`Cache-Control` は `private, no-store` |
| TC-app-api-httpEntry-67 | なし | `https://cms.example/media/{配信先}` へ `GET` を呼ぶ | `MediaDeliveryLinks.parse` に、要求の URL の全体 `https://cms.example/media/{配信先}` を渡す |
| TC-app-api-httpEntry-68 | `MediaDeliveryLinks.parse` が `null` を返す配信先 | `GET /media/{配信先}` を呼ぶ | 状態 404。`kind` は `notFound`。`deliverMedia` を呼ばない |
| TC-app-api-httpEntry-69 | 要求は、セッションを持つ | `GET /media/{配信先}` を呼ぶ | `deliverMedia` に、セッションで確かめたアカウントの `AccountId` を渡す。`Authorization` ヘッダーを読まない |
| TC-app-api-httpEntry-70 | 要求は、セッションを持たない | `GET /media/{配信先}` を呼ぶ | `deliverMedia` に、アカウントとして `null` を渡す |
| TC-app-api-httpEntry-71 | `deliverMedia` は、同じ配信先に、一度目は配信する結果を、二度目は配信しない結果を返す | 同じ配信先を二度呼ぶ | 二度とも `deliverMedia` を呼ぶ。二度目は状態 404 を返し、一度目の応答を返さない |
| TC-app-api-httpEntry-74 | `publishContent` は、対象 A・C のうち、A に `ConflictError`（`STALE_VERSION`）、C に `BusinessRuleError`（`PublishRejected`）が当たり、`rejections` に入力の順で両方を持つ `ConflictError` を返す | トークンを示して、対象の列 [A, C] で `POST /api/v1/manage/contents/publish` を呼ぶ | 状態 409。`kind` は `conflict`、`code` は `STALE_VERSION`。`details.rejections` は、`{ type: "content", id: A, kind: "conflict", code: "STALE_VERSION" }`、`{ type: "content", id: C, kind: "business", code: "PublishRejected", details }` をこの順に持つ |
