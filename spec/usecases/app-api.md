# アプリ向けの HTTP API と `packages/client`

表示アプリとスクリプトが CMS を呼ぶ HTTP API の規約と、`packages/client`（`@repo/client`）が公開するものを定める。`apps/web` のアプリ向けの HTTP API、`packages/client`、`apps/cli` は、この文書に従う。SDK を使わない表示技術は、同じ規約で HTTP API を直接呼ぶ。

ユースケースの入出力と認可は各ドメインのユースケースのファイルが、振る舞いの結果（エラーを含む）は各ドメインのユースケースのテストが定める。型と振る舞いは [../domains/](../domains/index.md) が定める。表示アプリへの通知の要求の形（本文、ヘッダー、署名）は [../domains/webhook.md](../domains/webhook.md) の「NotificationPayload」と「WebhookSender」が定める。入口が持つ規則（要求と応答の形、トークンと利用元、スキーマの識別子、エラーの形、利用できないワークスペースの応答、フォームの受付結果、要求からユースケースの入力への写し方、操作の ID、一覧と展開の指定、参照の位置ごとの形、メディアの配信の応答）と、`apps/cli` がスキーマの反映の要求に添える値のテストケースは、`../testcases/app-api/` が持つ。

## HTTP API の規約

### 要求と応答の形

- 経路は `/api/v1` で始まる。対象のワークスペースは、トークンから決まる（「トークン」）。経路と本文は、ワークスペースを指定しない。メディアの配信だけは `/media` で始まり、「メディアの配信」が定める
- 読み取りは `GET`、書き込みは `POST` を使う。入力に移行定義の列を取る適用状況の確認だけは、読み取りを `POST` で行う。`POST` の本文は JSON（UTF-8）で、対応するユースケースの入力から、入口が作る値（実行者の `Actor`、`WorkspaceId`、操作の ID、入力の帰属の `InputSource`、`conditionKeys`、メディアの受け付ける種類の `MediaAcceptance`）と、経路とヘッダーが持つ値を除いたものを持つ。メディアの登録だけは `multipart/form-data` を使う
- 応答の本文は JSON（UTF-8）で、次のどちらかの形を取る

```ts
type SuccessBody<T> = { data: T; fingerprints?: FingerprintMatch[] };
type ErrorBody = { error: ApiError };
```

- 成功の応答の状態は 200。`data` は、対応するユースケースの出力から、操作の ID と識別子ごとの一致（`fingerprints` に置く。「スキーマの識別子」）を除いたもの。[../domains/index.md](../domains/index.md) の「URL の組み立て」の対象の値は、入口が組み立てた URL に置き換え、値そのものを持たない。出力を持たないユースケースでは `null`。フォームの送信は「フォームの受付結果」に従う
- 日時は `YYYY-MM-DDTHH:mm:ss.sssZ` の文字列、日付は `YYYY-MM-DD` の文字列で表す
- 項目の値は、[../domains/schema.md](../domains/schema.md) の「型の対応」が定める形の JSON で表す。取得のユースケースの出力は、[../domains/content.md](../domains/content.md) の `ContentDelivery.arrange` の結果（`ArrangedDelivery`）で、入口は、それを直列化する
- 値の位置（`ValuePath`）は、項目のキーと 0 始まりの位置を `.` でつないだ文字列で表す（例: `sections.0.title`）

### トークン

- トークンの値は、`Authorization` ヘッダーに `Bearer` 方式で添える。クエリ、本文、Cookie に載せない
- `Authorization` ヘッダーが無い要求、`Bearer` 方式でない要求は、示されたトークンを確かめる段に、トークンを示していない要求として渡す。結果のエラーは [../domains/token.md](../domains/token.md) の「エラーコード」が定める
- 利用元は、要求の `Origin` ヘッダーの値。`Origin` ヘッダーが無い要求は、利用元を示さない要求（`null`）
- トークンを示す経路の入口は、トークンの値と利用元を `authenticateToken`（[token.md](token.md)）に渡す。入口が行うことと、ユースケースへ渡す `Actor` は、[../domains/index.md](../domains/index.md) の「認可」に従う
- ブラウザからの呼び出しのために、すべての経路がオリジン間の要求を受け付ける。事前確認の要求（`OPTIONS`）は、トークンを確かめずに、`Authorization`、`Content-Type`、`X-Lane-Fingerprints` ヘッダーを許可して応える。利用元の可否は、事前確認ではなく、`authenticateToken` が決める
- `/api/v1` の経路のうちトークンを示さない要求は、プレビューの引き換え、スクリプトの認証の開始、スクリプトの認証の結果の受け取りの三つ。それぞれ、本文の値（引き換えの値、受け取りの値）で認可する

### 利用できないワークスペース

利用できないワークスペースへの要求の、入口ごとの結果は、[../domains/index.md](../domains/index.md)「利用できるワークスペース」が定める。応答の形は、結果がエラーなら「エラー」の表が、結果なら「要求と応答の形」が決める。

### スキーマの識別子

前提とするスキーマの識別子は、`X-Lane-Fingerprints` ヘッダーに添える。値は、`対象=識別子` を `,` で区切った並び。`,` の前後の空白は読み飛ばす。

| 対象 | 書き方 |
| --- | --- |
| スキーマ全体 | `schema=<識別子>` |
| モデル | `model.<モデルのキー>=<識別子>` |
| フォーム | `form.<フォームのキー>=<識別子>` |

```text
X-Lane-Fingerprints: model.article=3f2a…, model.author=9c41…
```

- 識別子は `SchemaFingerprint`（形は [../domains/schema.md](../domains/schema.md)「正規化した JSON と要約」）。入口は、ヘッダーの値を `ClaimedFingerprints` にしてユースケースへ渡す。ヘッダーが無い要求は、識別子を添えない要求
- 入口が拒否するヘッダーの値は、「入口の理由コード」の表が定める
- 識別子を照合する要求は、公開取得、プレビュー取得、フォームの送信（`acceptSubmission`）。ほかの要求は、ヘッダーを読まず、応答に `fingerprints` を持たない
- 照合する要求の成功の応答は、`fingerprints` に、添えられた識別子ごとの `FingerprintMatch`（`target` と `matches`）を持つ。識別子を添えない要求と、経路がエラーを受付失敗に写したフォームの送信（「フォームの受付結果」）では空の配列
- 不一致を理由に要求を拒否しない。応答の状態と `data` は、識別子の一致によって変わらない。`matches` は、`WorkspaceSchema.matchFingerprints`（[../domains/schema.md](../domains/schema.md)）の結果
- エラーの応答は `fingerprints` を持たない

### エラー

エラーの応答は、エラーの種類（[../domains/index.md](../domains/index.md) の表）から状態と `kind` を決める。

```ts
type ApiError =
  | {
      kind: "business" | "notFound" | "conflict" | "unauthenticated" | "forbidden" | "unavailable";
      code: string | null;
      message: string;
      details?: unknown;
    }
  | {
      kind: "validation";
      code: string | null;
      message: string;
      fieldErrors: Record<string, string[]>;
      details?: unknown;
    }
  | { kind: "rateLimited"; code: string | null; message: string; retryAt: string }
  | { kind: "system" | "unknown"; code: null; message: string };
```

| 種類 | `kind` | 状態 |
| --- | --- | --- |
| `BusinessRuleError` | `business` | 422 |
| `ValidationError` | `validation` | 422 |
| `NotFoundError` | `notFound` | 404 |
| `ConflictError` | `conflict` | 409 |
| `UnauthenticatedError` | `unauthenticated` | 401 |
| `ForbiddenError` | `forbidden` | 403 |
| `RateLimitedError` | `rateLimited` | 429 |
| `UnavailableError` | `unavailable` | 503 |
| `SystemError` | `system` | 500 |
| 上記のどれでもない失敗 | `unknown` | 500 |

- `code` は、各ドメインの `ErrorCode` と [../domains/common.md](../domains/common.md) の共通のコードと、「入口の理由コード」の表が定める理由コード。アプリは、`kind` と `code` で分岐し、`message` で分岐しない
- `system` と `unknown` は、`code` を `null`、`message` を固定の文言にして返す。内部の理由を応答に含めない
- `rateLimited` は、`retryAt`（再び試せる時刻。日時の文字列）と、同じ時刻までの秒数を持つ `Retry-After` ヘッダーを持つ
- `details` は、ユースケースがエラーに添えた内容。形は、その内容を定めるドメインの型の JSON

| `kind` と `code` | `details` |
| --- | --- |
| `forbidden`、`GRANT_MISSING` | 必要な許可（`grant`） |
| `forbidden`、`OUT_OF_SCOPE` | 範囲に無い対象（`target`） |
| `forbidden`、`PERMISSION_MISSING` | 不足する権限（`permissions`）と、該当する対象 |
| `notFound` | `{ type, keys }`。`type` は対象の種類（文字列は [../domains/common.md](../domains/common.md)「エラー」の規則で決まる。コンテンツは `content`）。`keys` は、`NotFoundError` が持つキーの文字列の列で、キーを持たない `NotFoundError` では空の列 |
| `business`、`DeliveryRejected` | `DeliveryFailure` |
| `business`、`conflict`（Content 以外の、複数の対象を取る操作） | 該当する対象ごとの理由 |
| Content の複数の対象を扱う操作のエラー（`kind` によらない） | `{ rejections }`。[../domains/content.md](../domains/content.md) の「拒否の順」の `rejections` の JSON で、形は下の `TargetRejection` の列 |

```ts
type TargetRejection = {
  type: "content" | "revision" | "media";
  id: string;
  kind: ApiError["kind"];
  code: string | null;
  details?: unknown;
};
```

- `type` は対象の種類（文字列は `notFound` の `type` と同じ規則）、`id` は対象の ID。`kind`・`code`・`details` は、その対象に当たるいちばん上の拒否を、この表と上の種類の表で写したもの。`validation` の対象は、その対象の `fieldErrors`（キーにコンテンツID を付けない）を `details` の `fieldErrors` に持つ
- 列は入力の順。応答の `kind`・`code` は、列の中のいちばん上の行の拒否のもの。`validation` の応答の `fieldErrors` は、下の表のとおり、条件を満たさない対象すべての位置を持つ

項目ごとの入力エラーは、`validation` の `fieldErrors` で返す。キーは誤りの位置、値はその位置の理由コードの配列。

| 入力 | `fieldErrors` のキー | 理由コード |
| --- | --- | --- |
| コンテンツの値（作成、下書きの保存、値の書き換え） | 値の位置（`ValuePath` の文字列）。複数の対象を取る操作では、先頭にコンテンツID と `:` を付ける | `EditViolation` の `reason`（[../domains/content.md](../domains/content.md) の「ContentIssue」） |
| コンテンツの一覧の条件 | `QueryIssue` の `subject` ごとに、`filter` は `filter.<位置>`、`filters` は `filter`、`sort` は `sort`、`search` は `search`、`expansion` は `expand.<位置>`、`depth` は `depth` | `QueryIssue` の `reason` |
| 単件の対象の指定 | `unique.<項目のキー>`（`QueryIssue` の `subject` は `lookup`） | `QueryIssue` の `reason` |
| 管理の一覧の状態の絞り込み | クエリの名前（`publication`、`hasDraft`、`draftHasIssues`、`publishedHasViolations`） | `invalidQueryValue` |
| 管理の一覧の置き場所とモデル（`GET /api/v1/manage/contents`） | クエリの名前（`location`、`model`） | `location` は `invalidQueryValue`。`model` は `modelRequired` |
| 一覧のページの指定（コンテンツの一覧を含む） | `page`、`limit` | `InvalidPagination` |
| 要求に書いた ID（呼び出し側が指定する ID と、既存の対象を指す ID） | 本文の ID は、本文のプロパティの名前を `.` でつないだ位置。経路の ID は、経路の変数の名前（`contentId`、`jobId` など） | `InvalidId`。境界で拒否する ID は [../domains/common.md](../domains/common.md)「Clock と IdGenerator」が定める。`jobId` を省いた要求も `InvalidId` |
| `X-Lane-Fingerprints` ヘッダー（「スキーマの識別子」） | `X-Lane-Fingerprints` | `duplicateTarget`、`invalidTarget`、`invalidFingerprint` |
| 複数の対象の列 | 列のプロパティの名前 | `InvalidTargetCount` |
| そのほかの本文 | 本文のプロパティの名前を `.` でつないだ位置 | 値を定めるドメインの理由コード |

`InvalidTargetCount` の `validation` は、`details` に上限の値（`limit`）を持つ。値は、`BatchTargets`（[../domains/common.md](../domains/common.md)）がエラーに添える `max`。

`QueryIssue` の `tooManyFilters` または `depthExceeded` を含む `validation` は、`details` の `limits` に、その `fieldErrors` のキー（`filter`、`depth`）ごとの上限の値を持つ。値は、`QueryIssue` の `limit`（[../domains/content.md](../domains/content.md) の「ContentDelivery」）。

#### 入口の理由コード

入口だけが返す理由コードは、この表がすべてである。入口は、表の要求を `validation` として拒否し、ユースケースを呼ばない。`fieldErrors` のキーは「エラー」の表が定める。表の要求と、要求に書いた ID の拒否の `validation` は、`code` が `null` で、理由を `fieldErrors` に持つ。

| 理由コード | 拒否する要求 |
| --- | --- |
| `invalidQueryValue` | 管理の一覧の状態の絞り込みの値が、「管理の一覧の指定」の書き方に合わない。管理の一覧の `location` が、`active` でも `trashed` でもない |
| `modelRequired` | 管理の一覧の `location` が `active` で、`model` を省いた |
| `duplicateTarget` | `X-Lane-Fingerprints` が、同じ対象を二度書いた |
| `invalidTarget` | `X-Lane-Fingerprints` が、「スキーマの識別子」の書き方に合わない対象を持つ |
| `invalidFingerprint` | `X-Lane-Fingerprints` が、`SchemaFingerprint` の形でない識別子を持つ |

### フォームの受付結果

フォームの送信の応答は、受付完了も受付失敗も、状態 200 の成功の形で返す。

```ts
type SubmissionBody = { data: ReceptionResult; fingerprints: FingerprintMatch[] };
```

- `data` は、[../domains/form.md](../domains/form.md)「ReceptionResult」の JSON。日時（`retryAt`）は日時の文字列。`acceptSubmission` が返した `ReceptionResponse` では、その `result`。`fingerprints` は「スキーマの識別子」が定める
- 項目の値は、本文の `values` に入れる。経路は、本文が `values` のほかに持つプロパティの名前を、本文の順に並べた列を、`conditionKeys` として `acceptSubmission` に渡し、値を読まない。`values` のほかのプロパティを持たない本文では、空の列を渡す。本文から `conditionKeys` への写し方は、ここだけが定める。`conditionKeys` が空でない送信の判定（入力不備の受付失敗）は、[../domains/form.md](../domains/form.md) の「受付」が定める
- `acceptSubmission` と入口の `authenticateToken` が投げたエラーのうち、[../domains/form.md](../domains/form.md)「受付失敗への写し方」が受付失敗に写すものは、写した受付失敗を `data` にする。ほかのエラーと、どの種類でもない失敗は、「エラー」の表に従う
- 入口は、本文が JSON のオブジェクトでない送信と、`values` が JSON のオブジェクトでない送信（`values` を持たない送信を含む）を、`validation` として拒否し、`acceptSubmission` を呼ばない。`values` の各項目の値は、JSON の値のまま `SubmissionInput` として渡し、形を確かめない。値の判定は、[../domains/form.md](../domains/form.md)「SubmissionInput と InquiryFacts」が定める。`values` のほかのプロパティを持つことは、`validation` にしない

### 経路の命名

- 区切りは、小文字の英字と `-` の語。モデル・フォーム・項目のキー、ID、一意な値は、URL のパスの一区切りとしてパーセントエンコードして入れる
- 集まりは複数形の名詞で表し、その一件は、後ろに ID を続ける。ワークスペースに一つのもの（`schema`）、単一モデルのコンテンツ（`single`）、要求が示したトークン自身（`token`）は、単数形の名詞で表す
- 公開取得は `/published`、プレビュー取得は `/preview`、コンテンツ管理は `/manage`、スキーマの反映は `/schema` で始める。公開取得とプレビュー取得は、この先頭だけが異なる
- 新しいものを作る要求（作成、送信の受付、要求の受け付け、記録の追加）は、集まりへの `POST` にする（`/manage/models/{model}/contents`、`/forms/{form}/submissions`、`/schema/applies`）
- 既にあるものを変える操作は、操作の対象の後ろに操作の動詞を続けた経路への `POST` にする。対象は、一件の対象ではその一件の経路（一件の下書きは、その後ろに `draft`）、複数の対象を取る操作と、対象を本文の値（引き換えの値、受け取りの値）で指す操作では集まりの経路（`/manage/contents/{contentId}/restore`、`/manage/contents/{contentId}/draft/discard`、`/manage/contents/publish`、`/preview-exchanges/redeem`、`/token/revoke`）

### 要求とユースケースの対応

ユースケースは、表の前に示すファイルの、見出しの識別子で指す。経路を持つのは、トークンを受け付けるユースケースと、トークンを示さずに本文の値で認可するユースケース（プレビューの引き換え、スクリプトの認証の開始と結果の受け取り）と、メディアの配信。ほかのユースケースは、管理画面、運用コマンド（[../domains/index.md](../domains/index.md) の「認可」）、定期ジョブ、イベントの消費者、開発環境のコマンド（[schema.md](schema.md) の、保存を伴わないユースケース。CMS に接続しない）が、HTTP API を通らずに呼ぶ。トークンを示す経路の入口が行うことは、「トークン」が定める。

取得（[content.md](content.md)）。

| 要求 | ユースケース |
| --- | --- |
| `GET /api/v1/published/models/{model}/contents` | `listPublishedContents` |
| `GET /api/v1/published/models/{model}/contents/{contentId}` | `getPublishedContent`（コンテンツID） |
| `GET /api/v1/published/models/{model}/unique/{field}/{value}` | `getPublishedContent`（一意な項目のキーと値） |
| `GET /api/v1/published/models/{model}/single` | `getPublishedContent`（単一モデル） |
| `GET /api/v1/preview/models/{model}/contents` | `listPreviewContents` |
| `GET /api/v1/preview/models/{model}/contents/{contentId}` | `getPreviewContent`（コンテンツID） |
| `GET /api/v1/preview/models/{model}/unique/{field}/{value}` | `getPreviewContent`（一意な項目のキーと値） |
| `GET /api/v1/preview/models/{model}/single` | `getPreviewContent`（単一モデル） |

フォーム（[form.md](form.md)）。

| 要求 | ユースケース |
| --- | --- |
| `POST /api/v1/forms/{form}/submissions` | `acceptSubmission`。本文は `{ values: SubmissionInput }`。本文の写し方は「フォームの受付結果」が定める。識別子は `X-Lane-Fingerprints` ヘッダーで添える |

コンテンツ管理（[content.md](content.md)、[media.md](media.md)）。

| 要求 | ユースケース |
| --- | --- |
| `GET /api/v1/manage/contents` | `listContents`（置き場所が通常）、`listTrashedContents`（置き場所がごみ箱） |
| `GET /api/v1/manage/contents/{contentId}` | `getContentForEditing`（コンテンツID） |
| `GET /api/v1/manage/contents/{contentId}/reference-status` | `checkReferenceAndUniqueStatus` |
| `GET /api/v1/manage/models/{model}/single` | `getContentForEditing`（単一モデルのキー） |
| `POST /api/v1/manage/models/{model}/contents` | `createContent`。本文は `{ id?, values }` |
| `POST /api/v1/manage/contents/{contentId}/draft/save` | `saveDraft` |
| `POST /api/v1/manage/contents/{contentId}/draft/discard` | `discardDraft` |
| `POST /api/v1/manage/contents/publish` | `publishContent` |
| `POST /api/v1/manage/contents/unpublish` | `unpublishContent` |
| `POST /api/v1/manage/contents/trash` | `trashContent` |
| `POST /api/v1/manage/contents/{contentId}/restore` | `restoreTrashedContent` |
| `POST /api/v1/manage/contents/purge` | `purgeContent`。本文は、対象（コンテンツID と `version`）、対象に加える履歴の版（版の ID の列。省ける）。入口は、省いた要求を「指定なし」として渡す |
| `POST /api/v1/manage/contents/rewrite` | `rewriteValues`。本文は、対象と、書き換えの指定 |
| `POST /api/v1/manage/media` | `registerMedia`。`multipart/form-data` の `file` の部分に、ファイルの内容と元のファイル名を持つ。受け付ける種類（`MediaAcceptance`）は、入口が `any` を渡す。入口は、`MediaLimits.maxBytes`（[../domains/media.md](../domains/media.md)）を超える長さの `file` を読み込まずに、[../domains/media.md](../domains/media.md)「MediaAdmission」の `too_large` の拒否と同じ `ValidationError` を返し、ユースケースを呼ばない |

- `GET /api/v1/manage/contents` は、置き場所をクエリ `location`（`active` または `trashed`。指定しない場合は `active`）で、モデルをクエリ `model`（モデルのキー）で指定する。`active` は `model` と、「一覧の指定」の `filter`、`sort`、`page`、`limit` と、「管理の一覧の指定」の検索語と状態の絞り込みを使う。`trashed` は `model` を省け、`page` と `limit` を使う
- `createContent` の `id` は、[../domains/common.md](../domains/common.md) の「Clock と IdGenerator」に従って、入口が扱う。形式に合わない `id` の応答は、「エラー」の `fieldErrors` の表が定める
- `createContent` と `saveDraft` の入力の帰属（`InputSource`）は、入口が `direct` を渡す
- `publish`、`unpublish`、`trash`、`purge`、`rewrite` は、公開内容を変える操作の経路。操作の ID の扱いは [../domains/index.md](../domains/index.md)「公開内容の整合性と通し番号」に従う
- 複数の対象を取る経路は、対象の列を `BatchTargets` で受ける。0 件の列と、上限を超える列の応答は、「エラー」が定める

スキーマの反映（[schema.md](schema.md)）。`apps/cli` のコマンドが呼ぶ。

| 要求 | ユースケース |
| --- | --- |
| `POST /api/v1/schema/applies` | `requestSchemaApply`。本文は `{ jobId, baseSchemaVersion, schema, migrations }` |
| `POST /api/v1/schema/direct-applies` | `requestDirectSchemaApply`。本文は `{ jobId, baseSchemaVersion, schema, lossAcknowledged? }` |
| `POST /api/v1/schema/dry-runs` | `requestSchemaDryRun`。本文は `{ jobId, baseSchemaVersion, schema, migrations }` |
| `POST /api/v1/schema/applied-migrations` | `recordMigrationsAsApplied` |
| `GET /api/v1/schema` | `getCurrentSchema` |
| `POST /api/v1/schema/status` | `getMigrationStatus`。本文は `{ migrations }`（移行定義の列の全体）。何も書き込まない |
| `GET /api/v1/schema/versions` | `listSchemaVersionRecords` |
| `GET /api/v1/schema/results` | `listSchemaApplyResults` |
| `GET /api/v1/schema/jobs/{jobId}` | `getSchemaApplyJob` |

- スキーマと移行定義は、Schema ドメインの `Schema` と `Migration` の JSON で送る。入口が、境界のデコードで値にする
- 反映、直接反映、事前確認の要求の本文は、次の値を持つ。ジョブの ID のほかは、Schema ドメインの `ApplyRequest`（[../domains/schema.md](../domains/schema.md)）のフィールドと同じ名前。`source` と `mode` は、経路に対応するユースケースが決め、本文に持たない

| プロパティ | 内容 | 持つ経路 |
| --- | --- | --- |
| `jobId` | 要求した側が作ったジョブの ID。扱いは [../domains/common.md](../domains/common.md) の「Clock と IdGenerator」に従う。形式に合わない ID と、`jobId` が無い要求の応答は、「エラー」の `fieldErrors` の表が定める | 三つとも |
| `baseSchemaVersion` | 前提とする現行スキーマの版。0 以上の整数 | 三つとも |
| `schema` | スキーマの全体 | 三つとも |
| `migrations` | 移行定義の列の全体。適用の順 | `applies`、`dry-runs` |
| `lossAcknowledged` | 直接反映が確定の前に求める確認を済ませたこと。確認を求める対象は [../domains/schema.md](../domains/schema.md) の「反映の要求の検証」が定める。真偽値。省いた要求は `false` | `direct-applies` |

- `apps/cli` は、要求ごとに `jobId` を一つ作り、応答を得られなかった要求の再送に同じ `jobId` を使う。再送では、最初の要求と同じ値を使う
- `apps/cli` は、版を変える要求（`applies`、`direct-applies`、`applied-migrations`）の `baseSchemaVersion` に、SMIG の「共通の前提」が定める示した版を添える。コマンドは、示した版を、接続先とトークンの組ごとに開発環境へ保存し、示すたびに置き換える。保存する版は、次の表のとおり

| 要求 | 示して保存する版 |
| --- | --- |
| 事前確認（`dry-runs`） | 要求の前に `GET /api/v1/schema` で読み、`baseSchemaVersion` に添えた版 |
| 適用状況の確認（`status`） | `data` の現行スキーマの版 |
| 反映 | `succeeded` のジョブの `success.schemaVersion`。差分が無いことを返した要求では、`data` の現在の版 |
| 適用済みとしての記録 | `data` の記録後の版 |
| 拒否された要求、失敗で終えた反映 | 保存した版を変えない |

- 保存した版が無い開発環境では、コマンドが、要求の前に `GET /api/v1/schema` で版を読み、開発者に示してから添える
- 直接反映の確認（`SCHEMA_LOSS_NOT_ACKNOWLEDGED` の後の、`lossAcknowledged` を真にした要求）は、確認を求めた要求と同じ `baseSchemaVersion` を添える
- 一覧の要求は、「一覧の指定」の `page` と `limit` を使う

トークンの受け渡し（[token.md](token.md)）。

| 要求 | トークン | ユースケース |
| --- | --- | --- |
| `POST /api/v1/preview-exchanges/redeem` | 示さない | `redeemPreviewExchange`。本文は `{ code }` |
| `POST /api/v1/device-authorizations` | 示さない | `startDeviceAuthorization`。本文は `{ grants }`。ワークスペースを指定しない |
| `POST /api/v1/device-authorizations/redeem` | 示さない | `redeemDeviceAuthorization`。本文は `{ deviceCode }` |
| `POST /api/v1/token/revoke` | 示す | `revokeOwnToken` |

- 表示アプリは、引き換えの値を、プレビュー先の URL のクエリから読む。URL の形とクエリの名前は、[../domains/token.md](../domains/token.md) の `PreviewDestination` が定める
- `startDeviceAuthorization` の `data` は、確認用のコード、受け取りの値、依頼の期限、承認用の URL を持ち、承認の値を持たない（「要求と応答の形」）。承認用の URL は、[../domains/index.md](../domains/index.md) の「URL の組み立て」に従う

### メディアの配信

メディアの配信の入口の、経路、資格、応答の形を定める。ユースケースは [media.md](media.md) が定める。「要求とユースケースの対応」の一部で、ユースケースを見出しの識別子で指す。

| 要求 | 資格 | ユースケース |
| --- | --- | --- |
| `GET /media/{配信先}` | メンバーのセッション、またはプレビュー用の配信先。資格を示さない要求も受け付ける | `deliverMedia` |

- 配信先の URL は `MediaDeliveryLinks`（[../domains/media.md](../domains/media.md)）が作る。`/media/` より後ろの形は、このポートが定める。入口が `parse` に渡す値は、[../domains/media.md](../domains/media.md)「MediaDeliveryLinks」に従う。入口は、得た `MediaUrl` をユースケースへ渡す
- トークンを示さない。`Authorization` ヘッダーを読まない
- 入口は、要求にセッションがあれば、セッションで確かめたアカウント（`AccountId`）を渡す。セッションが無い要求は `null` を渡す。入口は、メンバーとしての認可を行わない
- 配信の資格と可否は、ユースケースが、Media の `MediaAccess.resolve` と `MediaDelivery.decide` で決める（[../domains/media.md](../domains/media.md)）。入口は、可否の規則を持たない
- 成功の応答は、状態 200 で、本文に実体の内容を持つ。JSON の形を取らない。`Content-Type` は、`deliverMedia` が返した種類（`MediaDelivery.decide` の `contentType`）。`Cache-Control` は、入口が、`deliverMedia` の返した配信の先の保持の方針（`DownstreamRetention`。[../domains/media.md](../domains/media.md) の「MediaDelivery」）から、次の表で作る。文字列を作るのは入口だけ。保持の方針と長さは、入口が決めない

| `retention` | `Cache-Control` の値 |
| --- | --- |
| `long` | `public, max-age={maxAgeSeconds}, immutable` |
| `none` | `private, no-store` |

- 入口の応答は、[../domains/media.md](../domains/media.md) の「MediaDelivery」に従う
- 利用できないワークスペースのメディアへの要求の応答は、「利用できないワークスペース」に従う
- `parse` が `null` を返した要求（ユースケースを呼ばない）と、`deliverMedia` の `NotFoundError` は、理由を区別せずに `notFound`（状態 404）のエラーの形で返す

### 一覧の指定

一覧の取得は、絞り込み、並び替え、ページ分割をクエリで指定する。意味と上限は [../domains/content.md](../domains/content.md) の「取得と検索の規則」が定める。

| クエリ | 書き方 | 対応する値 |
| --- | --- | --- |
| `filter` | `<項目のキー>:<比較>:<値>`。条件ごとに繰り返す | `FilterInput`。繰り返しの順が、条件の位置（0 始まり） |
| `sort` | `field:<項目のキー>:<順>`、`createdAt:<順>`、`updatedAt:<順>`。`<順>` は `asc` または `desc` | `ContentSort`。指定しない場合は、[../domains/content.md](../domains/content.md)「並び替え」の既定の並び |
| `page` | 整数。指定しない場合は 1 | `Pagination.page` |
| `limit` | 整数。指定しない場合は、[../scenario/app-integration.md](../scenario/app-integration.md) の APP-12 の「一覧の 1 ページの件数」の、件数を指定しない取得の値 | `Pagination.limit` |

`filter` の比較と値。項目のキーと比較は `:` を含まないので、値は 2 つ目の `:` より後ろの全体。

| 比較 | 値 | `FilterInput` |
| --- | --- | --- |
| `eq` | 項目の値の種類の表記 | `equals` |
| `range` | `<下限>..<上限>`。限らない側は空にする（`100..`、`..2026-12-31`） | `range`。`..` の前が `min`、後ろが `max`。空の側は `null` |
| `includes` | 選択肢のキー | `includes` |
| `references` | コンテンツID | `references` |

- 入口は、`filter` を、項目のキー、比較、値の表記に分けて `FilterInput`（[../domains/content.md](../domains/content.md)）にし、`eq` の値と `range` の境界を、表記の文字列のままユースケースへ渡す。入口は、現行スキーマを読まない。値の種類ごとの表記と、表記を値にする振る舞い（`ContentDelivery.validateQuery`。ユースケースが、読んだ現行スキーマとともに呼ぶ）は、同じ文書の「絞り込み」が定める。表記として読めない値を `valueKindMismatch` にするのは、その振る舞いである
- 一覧の `data` は、`items`（そのページの内容）と `count`（条件に該当する総件数）を常に持つ。プレビュー取得の一覧は、結果に含めなかった対象（`excluded`）を加える
- 続きは、`page` を進めて取得する

### 管理の一覧の指定

通常の管理対象の一覧（`GET /api/v1/manage/contents` の `location` が `active`）は、「一覧の指定」に加えて、検索語と状態の絞り込みをクエリで指定する。意味は [../domains/content.md](../domains/content.md) の「管理の検索と絞り込み」が定める。管理画面の一覧（EDIT-01）と同じ条件を受ける。

| クエリ | 書き方 | 対応する値 |
| --- | --- | --- |
| `search` | 検索語 | `ContentSearch.term`。指定しない場合は、検索しない |
| `publication` | `published` または `unpublished` | `StatusFilter.publication` |
| `hasDraft` | `true` または `false` | `StatusFilter.hasDraft` |
| `draftHasIssues` | `true` または `false` | `StatusFilter.draftHasIssues` |
| `publishedHasViolations` | `true` または `false` | `StatusFilter.publishedHasViolations` |

- 指定しない状態の絞り込みは、絞り込まない（`null`）
- 書き方に合わない状態の絞り込みの値は、「入口の理由コード」の表に従う

### 参照の展開の指定

単件と一覧の取得は、展開をクエリで指定する。

| クエリ | 書き方 | 対応する値 |
| --- | --- | --- |
| `expand` | `<モデルのキー>:<項目への経路>`。項目ごとに繰り返す | `Expansion.fields`。繰り返しの順が、指定の位置（0 始まり） |
| `depth` | 整数。`expand` を指定して `depth` を指定しない場合は、APP-12 の「展開する参照の深さ」の、深さを指定しない取得の値。`expand` を指定しない場合は 0 | `Expansion.depth` |

- 項目への経路は、モデルの直下から項目までのキーを `.` でつないだもの。入れ子へ入る項目のキーと、ブロックの項目の直後に置くブロックの種類のキーを、順に並べ、最後に展開する項目のキーを置く（例: `article:author`、`article:sections.hero.related`）
- 取得の対象のモデルの項目と、展開した参照先のモデルの項目を、同じ書き方で指定する
- モデル・フォームごとの識別子を添える要求は、取得の対象のモデルの識別子と、[../domains/schema.md](../domains/schema.md)「参照の展開」が定める、展開する参照の項目が許可するモデルの識別子を添える

## `packages/client` が公開するもの

`packages/client` は、Schema ドメインの値と純粋な関数を `@repo/core` から使う。型、識別子、名前からの ID を求めるために、CMS へ接続しない。テストケースは `../testcases/client/` が持つ。クライアントは `createClient.md`、通知の署名の確認は `verifyNotification.md`、`@repo/client/next` は `nextHandlers.md`、フォームの送信のフックは `formSubmissionHook.md`、スキーマの記法は `schemaNotation.md`、移行定義の記法は `migrationNotation.md`、型の導出は `types.md`、名前からコンテンツID を求める関数は `contentId.md`。

### 入口

| 入口 | 内容 | 使う場所 |
| --- | --- | --- |
| `@repo/client` | クライアント、型の導出、識別子、名前からコンテンツID を求める関数、通知の署名の確認 | `fetch` を持つすべての実行環境（サーバー、ブラウザ、スクリプト） |
| `@repo/client/schema` | スキーマの記法 | スキーマのコード |
| `@repo/client/migration` | 移行定義の記法 | 移行定義のコード |
| `@repo/client/next` | プレビューの入口（Next.js のミドルウェア）、プレビュー用のクライアントを得る関数、プレビューを終える入口と通知の入口（Next.js のルートハンドラー） | Next.js の表示アプリ |
| `@repo/client/react` | フォームの送信の状態（入力、送信中、受付結果、項目ごとの理由）を保つフック | React の表示アプリ |

- `@repo/client/next` と `@repo/client/react` は、`@repo/client` の関数だけを通じて HTTP API を呼ぶ。ほかの表示技術のアプリは、`@repo/client` だけで同じ操作を行える
- プレビューの入口は、モデルのプレビュー先の経路への要求のうち、引き換えの値を持つ要求を受ける（APP-06）。引き換えの値をトークンに引き換え、トークンとその有効期限をサーバー側のセッションに結び付けて、要求の URL から引き換えの値のクエリを除いた URL へ移す。トークンの値をブラウザへ渡さない。CMS が引き換えを拒否した場合は、セッションにトークンを結び付けず、移さずに、引き換えを拒否されたことを示す
- プレビュー用のクライアントを得る関数は、要求のセッションに結び付いたトークンを `withToken` で使うクライアントを返す。結び付いたトークンが無い場合と、その有効期限を過ぎた場合は `null` を返す。表示アプリのサーバー側の処理は、このクライアントでプレビュー取得を行う
- プレビューを終える入口は、セッションからトークンの結び付きを外し、トークンを破棄する
- 通知の入口は、受け取った要求の本文のバイト列と、`X-Lane-Timestamp`、`X-Lane-Signature` ヘッダーの値を `verifyNotification` に渡して、署名を確かめる（APP-11）。応答の状態は 401、204、500 のどれか。どの場合にどれを返し、開発者の関数をいつ呼ぶかは、[../testcases/client/nextHandlers.md](../testcases/client/nextHandlers.md) が定める。CMS が応答の状態をどう扱うかは、[../domains/webhook.md](../domains/webhook.md)「WebhookSender」が定める
- キャッシュの破棄、重複の見分け、順序の判別は、開発者の関数が行う。通知の入口は、処理済みの通知と通し番号を保持しない。見分け方と、設置の復旧（HOST-08）の後の通し番号の扱いは、[../domains/webhook.md](../domains/webhook.md) の「NotificationPayload」と「通し番号と復旧」が定める

### スキーマの記法

`@repo/client/schema` の関数は、Schema ドメインの値を返す。キーを持つ定義（モデル、フォーム、項目、選択肢、ブロックの種類、派生形式）は、キーをプロパティの名前にしたオブジェクトで宣言する。プロパティの宣言の順が、`Schema` の配列の順（表示の順序）になる。型は、プロパティの名前から、キーを文字列のリテラルとして得る。

| 関数 | 役割 | Schema ドメインの値 |
| --- | --- | --- |
| `defineSchema({ models, forms, derivatives })` | スキーマを宣言する | `Schema` |
| `collection(options)` | 複数件を持つモデルを宣言する | `ModelDefinition`（`cardinality` は `collection`） |
| `single(options)` | 単一の情報のモデルを宣言する | `ModelDefinition`（`cardinality` は `single`） |
| `form(options)` | フォームを宣言する | `FormDefinition` |
| `field.text` / `number` / `boolean` / `date` / `datetime` / `select` / `multiSelect` / `richText` / `media` / `reference` / `referenceList` / `group` / `repeat` / `blocks` | モデルの項目を、値の種類ごとに宣言する | `ContentField`。`type.kind` は関数の名前 |
| `formField.text` / `number` / `boolean` / `date` / `datetime` / `select` / `multiSelect` / `contentReference` | フォームの項目を、値の種類ごとに宣言する | `FormField`。`type.kind` は関数の名前 |
| `option(options)` | 選択肢を宣言する | `ContentOption`、`FormOption` |
| `blockType(options)` | ブロックの種類を宣言する | `BlockTypeDefinition` |
| `format.email()` / `format.url()` / `format.pattern(source)` | テキストの書式を宣言する | `TextFormat` |
| `createdAt()` | 日付・日時の初期値に、作成時点を指定する | `TemporalDefault`（`type` は `createdAt`） |
| `derivative({ size, output })` | 画像の派生形式を宣言する | `ImageDerivative` |
| `previewPath(...parts)` | モデルのプレビュー先の経路を宣言する。`parts` は、文字列、`contentId()`、`uniqueValue(field)` の並び | `PreviewPath`。文字列は `literal`、`contentId()` は `contentId`、`uniqueValue(field)` は `uniqueValue` |

- 制約、必須、初期値、一意性は、項目の関数の `options` のプロパティで宣言する。プロパティの名前は、`ContentFieldType` と `FormFieldType` のフィールドの名前（`required`、`constraints` の `minLength`・`maxLength`・`format`・`min`・`max`、`unique`、`defaultValue`、`options`、`models`、`fields`、`blockTypes`、`mustBeTrue`、`mustMatch`）。値の種類が持てないプロパティは、型が受け付けない
- 表示・運用の定義は、`label`、`description`、`example`、`input`、`listFields`、`inquiryLabelField`、`previewPath` のプロパティで宣言する
- 非推奨は、モデル、項目、選択肢、ブロックの種類の `options` の `deprecated: true` で宣言する
- 宣言しないプロパティは、Schema ドメインの値の既定になる。`required`・`unique`・`deprecated`・`mustBeTrue` は偽、`label`・`description`・`example`・`defaultValue`・`previewPath`・`inquiryLabelField`・`mustMatch` と制約は `null`、`input` は `singleLine`、`listFields` は空。`multiSelect` の `defaultValue` は、空の組（初期値なし）
- 記法が返す値の確かめ方は、[../domains/schema.md](../domains/schema.md) の「Schema」に従う
- 記法が公開する関数は、上の表がすべて。宣言できるのは、スキーマが持つ定義（構造、制約、表示・運用の定義、非推奨）だけ

### 型の導出

型のうち、スキーマの定義から決まる部分は、`defineSchema` の結果の型から、[../domains/schema.md](../domains/schema.md) の「型の対応」の規則で導く。ほかの部分は、それを値として持つドメインの型を使う。一件の取得結果が項目の値のほかに持つもの、本文の文書、参照の要約は [../domains/content.md](../domains/content.md) の `ArrangedDelivery`、メディアの取得結果は [../domains/media.md](../domains/media.md) の `renditions`、フォームの受付結果と入力エラーの理由は [../domains/form.md](../domains/form.md) の `ReceptionResult` と `SubmissionIssue`。`packages/client` は、両方を組み合わせて、次の型を公開する。`S` はスキーマの型、`M` はモデルのキー、`F` はフォームのキー。

| 型 | 内容 |
| --- | --- |
| `Content<S, M, E>` | 一件の取得結果。`E` は展開の指定（展開する項目と深さ。`Expansion`）。参照の項目の型は、参照の位置ごとに、[../domains/content.md](../domains/content.md) の「参照先の展開」が定める形（展開した内容か要約）を取る |
| `ContentInput<S, M>` | コンテンツ管理で指定する値 |
| `ContentIssues<S, M>` | コンテンツの入力エラー（[../domains/schema.md](../domains/schema.md) の「入力エラー」）。保存の拒否は、`ApiError` の `validation`（「エラー」）が持つ |
| `Submission<S, F>` | フォームの送信内容 |
| `SubmissionIssues<S, F>` | フォームの項目ごとの入力エラー |
| `Reception<S, F>` | フォームの受付結果。受付完了と、理由付きの受付失敗の直和 |

- 取得・送信・コンテンツ管理のクライアントの引数と結果は、これらの型を持つ
- 非推奨の印（[../domains/schema.md](../domains/schema.md) の「非推奨」）の記法は `@deprecated`

### クライアント

`createClient` に、接続先、トークン、スキーマの定義を一度だけ指定する。

| 設定 | 内容 |
| --- | --- |
| `baseUrl` | 接続先の設置の URL |
| `token` | トークンの値 |
| `schema` | `defineSchema` の結果 |
| `fingerprints` | 識別子を添える単位。`schema`（スキーマ全体）または `target`（モデル・フォームごと）。指定しない場合は `schema` |
| `onFingerprintMismatch` | 不一致の応答の扱い。`fail`（失敗として扱う）または `continue`（処理を続ける）。指定しない場合は `continue` |
| `onMismatch` | 不一致を受け取る関数。`continue` のアプリが、不一致を記録するために使う |

- クライアントは、識別子を `SchemaFingerprint.of(schema)` で求め、照合する要求に添える。`target` で添える対象は、取得では「参照の展開の指定」が、フォームの送信では送信先のフォーム
- `fail` では、`matches` が偽の対象を持つ応答の取得・送信が、不一致の対象を示す失敗（`FingerprintMismatchError`）になる。フォームの送信の失敗は、CMS が返した受付結果を持つ。受付完了の回答は保存済みで、アプリは再送を求めない
- `continue` では、結果を返し、`onMismatch` に不一致の対象を渡す
- エラーの応答は、応答の `ApiError`（「エラー」。`kind` で判別する直和）を `error` に持つ `LaneError` として投げる。`error` の `kind` を確かめると、その変種の値（`validation` の `fieldErrors`、`rateLimited` の `retryAt`、`details`）を読める。CMS に到達できない場合は、到達できないことを示す失敗を投げる
- 単件の取得は、対象のコンテンツが存在しない応答（`notFound` で、`details.type` が `content`）に `null` を返す。存在しないモデル・項目のキーの応答は、失敗として投げる
- フォームの送信は、受付失敗を投げずに、`Reception` として返す
- `withToken(token)` は、トークンだけを替えた同じ設定のクライアントを返す。プレビューで、引き換えたトークンを使う要求に使う
- ブラウザで動くコードは、ブラウザ用トークンを指定した同じクライアントで、公開取得とフォームの送信を呼ぶ

操作と要求の対応。

| 操作 | 要求 |
| --- | --- |
| `published.list` / `get` / `single`、`preview.list` / `get` / `single` | 取得の経路。`get` は、コンテンツID、または一意な項目のキーと値で対象を指定する |
| `forms.submit` | フォームの送信 |
| `manage.list` / `get` / `getSingle` / `referenceStatus` / `create` / `saveDraft` / `discardDraft` / `publish` / `unpublish` / `trash` / `restore` / `purge` / `rewrite` | コンテンツ管理の経路。`list` は、置き場所（通常、ごみ箱）を指定する。通常の一覧は、「管理の一覧の指定」の検索語と状態の絞り込みを受ける |
| `media.register` | メディアの登録 |
| `exchangePreviewCode` | プレビューの引き換え。トークンを添えない |
| `verifyNotification` | 表示アプリへの通知の署名の確認。CMS へ接続しない。秘密情報、受け取った本文のバイト列、`X-Lane-Timestamp` と `X-Lane-Signature` ヘッダーの値を受け、[../domains/webhook.md](../domains/webhook.md) の「WebhookSender」の規則で署名を計算し直す。一致すれば通知の内容（`NotificationPayload`）を返し、一致しなければ、確かめられないことを返す。時刻の古さでは拒否しない |

スキーマの反映、スクリプトの認証、トークンの失効の経路は、`apps/cli` のコマンドが呼ぶ。`packages/client` は、これらの操作を公開しない。

### 名前からコンテンツID を求める関数

```ts
function contentId(name: string): string;
```

- 同じ `name` から、常に同じコンテンツID を返す。データを投入するスクリプトは、コンテンツごとに決めた名前からコンテンツID を求め、コンテンツID を指定した作成に使う。投入をやり直しても、同じ ID になる
- 返す値は、`IdGenerator` の形式に合う（[../domains/common.md](../domains/common.md) の「Clock と IdGenerator」）
- 値は、名前ベースの UUID（バージョン 5）。名前空間は、nil UUID を名前空間に、文字列 `lane.content` から求めたバージョン 5 の UUID。名前は、`name` の UTF-8 のバイト列
- コンテンツID はワークスペースの中で一意なので、スクリプトは、モデルをまたいで重ならない名前を使う（例: `article/hello-world`）

### 移行定義の記法

`@repo/client/migration` の関数は、Schema ドメインの `Migration` と `MigrationOperation` を返す。移行定義の生成（`generateMigration`）は、この記法のコードを出力する。開発者は、出力されたコードを編集できる。

| 関数 | 役割 | Schema ドメインの値 |
| --- | --- | --- |
| `defineMigration(name, operations)` | 一つの移行定義を宣言する | `Migration` |
| `defineMigrations(migrations)` | 移行定義の列を宣言する。配列の順が、適用の順 | `readonly Migration[]` |
| `level(model, ...steps)` | 階層を指す。`steps` は、`field(key)` と `blockType(key)` の並び。文字列は `field` として扱う | `LevelPath` |

操作を宣言する関数は、`MigrationOperation` の `op` と同じ名前を持ち、その操作の値を返す。

| 関数 | 引数 |
| --- | --- |
| `addModel` | モデルのキー、単一か複数件か |
| `renameModel` | 変更前と変更後のキー |
| `removeModel` | モデルのキー、`lossConfirmed: true` |
| `addField` | 階層、項目のキー、`NewFieldType`、補完値 |
| `renameField` | 階層、変更前と変更後のキー |
| `changeRequired` | 階層、項目のキー、必須にする（補完値）か任意にするか |
| `changeReferenceModels` | 階層、項目のキー、許可するモデル |
| `changeFieldKind` | 階層、項目のキー、`ConvertedFieldType`、`lossConfirmed: true` |
| `removeField` | 階層、項目のキー、`lossConfirmed: true` |
| `addOption` | 階層、項目のキー、選択肢のキー |
| `renameOption` | 階層、項目のキー、変更前と変更後のキー |
| `removeOption` | 階層、項目のキー、選択肢のキー、置き換え先の選択肢、または未設定と `lossConfirmed: true` |
| `addBlockType` | 階層、項目のキー、ブロックの種類のキー |
| `renameBlockType` | 階層、項目のキー、変更前と変更後のキー |
| `removeBlockType` | 階層、項目のキー、ブロックの種類のキー、`lossConfirmed: true` |
| `copyValue` | 階層、複写元と複写先の項目のキー |

- データを失い得る操作の関数は、`lossConfirmed: true` を引数に持たない呼び出しを、型が受け付けない
- 補完値は `LiteralValue`。指定しない場合は `null`
- 操作を宣言する関数は、上の表がすべて。引数は、Schema ドメインの値（キー、階層、項目の型、`LiteralValue`）だけを取る
