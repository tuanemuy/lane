# 公開コンテンツを単件で取得する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-getPublishedContent-01 | モデル M の A は公開中で、下書きを持つ。トークンは `public_read` の許可を持ち、範囲に M を含む | トークンで、M のキーと A のコンテンツID を指定して取得する | A のコンテンツID、作成日時、公開内容の更新日時、公開内容の値を返す。下書きの値と、下書きがあることを含めない |
| TC-content-getPublishedContent-02 | M の A は公開中で、公開内容の一意な項目 `slug` は「x」 | `public_read` のトークンで、M のキー、`slug`、値「x」を指定して取得する | A の公開内容を返す |
| TC-content-getPublishedContent-03 | 単一モデル S のコンテンツは公開中 | `public_read` のトークンで、S のキーを、対象を指定せずに指定して取得する | S のコンテンツの公開内容を返す |
| TC-content-getPublishedContent-04 | A の公開内容は、画像のメディア Md を使う。Md は、生成済みの派生形式と、生成中の派生形式を持つ | `public_read` のトークンで、A を取得する | Md について、元のファイルと生成済みの派生形式の、配信先、画像形式、幅、高さを返す。生成中の派生形式を返さない |
| TC-content-getPublishedContent-05 | A の公開内容は、公開中の T を必須の参照で参照する | `public_read` のトークンで、A を、展開を指定せずに取得する | T への参照を、T のモデル、コンテンツID、一意な値で返す |
| TC-content-getPublishedContent-06 | A の公開内容の本文は、公開中の T へのリンクを持つ | `public_read` のトークンで、A を取得する | T のモデル、コンテンツID、一意な値を、参照先の要約として返す |
| TC-content-getPublishedContent-07 | A の公開内容は、公開中の T をコンテンツ参照の項目 `author` で参照する。T は下書きを持つ | `public_read` のトークンで、A を、`author` の展開と深さ 1 を指定して取得する | T の公開内容を、A と同じ取得結果で返す。T の下書きの値を含めない |
| TC-content-getPublishedContent-08 | A の公開内容は、任意の参照で T を参照する。T は公開を取り消されている | `public_read` のトークンで、A を取得する | エラーにしない。T への参照を未設定として返す |
| TC-content-getPublishedContent-09 | A の公開内容は、任意の参照で T を参照する。T は公開を取り消された後、再び公開されている。A は公開し直していない | `public_read` のトークンで、A を取得する | T への参照を返す |
| TC-content-getPublishedContent-10 | A と B は公開中で、コンテンツ参照の項目 `related` で互いを参照する | `public_read` のトークンで、A を、`related` の展開と深さ 1 を指定して取得する | B の公開内容を展開して返す。B から A への参照は、モデル、コンテンツID、一意な値で返す |
| TC-content-getPublishedContent-11 | アプリが添えた M の識別子は、現行スキーマの M の識別子と一致しない | `public_read` のトークンで、識別子を添えて A を取得する | 要求を拒否しない。A の公開内容と、識別子が一致しないことを返す |
| TC-content-getPublishedContent-12 | U は未公開 | `public_read` のトークンで、U のコンテンツID を指定して取得する | `NotFoundError` を返す。U の下書きの内容と存在を明かさない |
| TC-content-getPublishedContent-13 | C はごみ箱にある | `public_read` のトークンで、C のコンテンツID を指定して取得する | `NotFoundError` を返す |
| TC-content-getPublishedContent-14 | A は、モデル M のコンテンツ | `public_read` のトークンで、別のモデル M2 のキーと、A のコンテンツID を指定して取得する | `NotFoundError` を返す |
| TC-content-getPublishedContent-16 | 現行スキーマに、キー `gone` のモデルが無い | `public_read` のトークンで、`gone` を指定して取得する | `NotFoundError` を返す。キー `gone` を示す |
| TC-content-getPublishedContent-17 | トークンは `public_read` の許可を持たない | トークンで、A を取得する | `ForbiddenError`（`GRANT_MISSING`）を返す |
| TC-content-getPublishedContent-19 | トークンは `public_read` の許可を持ち、範囲は M を含み、モデル M2 を含まない。A の項目 `author` は M2 への参照を許可する | トークンで、A を、`author` の展開を指定して取得する | `ForbiddenError`（`OUT_OF_SCOPE`）を返す |
| TC-content-getPublishedContent-20 | A の公開内容は、モデル M2 の公開中の T を項目 `author` で参照する。アプリが添えた M の識別子は現行と一致し、M2 の識別子は現行と一致しない | `public_read` のトークンで、M と M2 の識別子を添えて、A を、`author` の展開と深さ 1 を指定して取得する | 要求を拒否しない。A と、展開した T の公開内容を返す。M の識別子は一致、M2 の識別子は不一致と返す |
| TC-content-getPublishedContent-21 | M の項目 `title` は、一意にする制約を持たない | `public_read` のトークンで、M のキー、`title`、値を指定して取得する | `ValidationError` を返す。項目と理由を添える |
| TC-content-getPublishedContent-22 | M の項目 `title` はテキスト | `public_read` のトークンで、A を、`title` の展開を指定して取得する | `ValidationError` を返す。項目と理由を添える |
| TC-content-getPublishedContent-24 | A の公開内容は、公開中の T を、コンテンツ参照の項目 `author` と `reviewer` の両方で参照する | `public_read` のトークンで、A を、`author` の展開と深さ 1 を指定して取得する | `author` の位置には T の公開内容を、`reviewer` の位置には T のモデル、コンテンツID、一意な値を返す |
| TC-content-getPublishedContent-25 | トークンは `public_read` の許可を持ち、範囲は M を含み、モデル M2 を含まない。A の公開内容は、M2 の公開中の T を参照する | トークンで、A を、展開を指定せずに取得する | A の公開内容を返す。T への参照を、T のモデル、コンテンツID、一意な値で返す |
| TC-content-getPublishedContent-26 | A は公開中で、公開内容の `title` は「x」。スキーマの反映が適用中で、その準備領域に、`title` を「y」にした A の変換結果がある | `public_read` のトークンで、A を取得する | 反映前の構造で、`title` が「x」の A の公開内容を返す |
| TC-content-getPublishedContent-27 | A の公開内容は、公開中の T をコンテンツ参照の項目 `author` で、公開中の V を `reviewer` で参照する。V の公開内容は、`ContentDelivery.limits.referencedContents` 件を超える数の公開中のコンテンツを参照する | `public_read` のトークンで、A を、`author` の展開と深さ 1 を指定して取得する | 成功する。`author` の位置に T の公開内容を、`reviewer` の位置に V の要約を返す |
| TC-content-getPublishedContent-28 | A の公開内容は、`ContentDelivery.limits.referencedContents` 件を超える数の公開中のコンテンツを参照する | `public_read` のトークンで、A を取得する | `BusinessRuleError`（`DeliveryRejected`）を返す。理由（`referenceLimitExceeded`）と上限を添える。A の内容を返さない |
| TC-content-getPublishedContent-29 | A は公開中。取得の読み始めと読み終わりで、スキーマの版が違い、読み直した取得でも、読み始めと読み終わりのスキーマの版が違う | `public_read` のトークンで、A を取得する | `ConflictError`（`SCHEMA_CHANGED`）を返す。A の内容を返さない |
| TC-content-getPublishedContent-30 | A の公開内容は、公開中の T をコンテンツ参照の項目 `author` で参照する | `public_read` のトークンで、A を、`author` の展開と、`ContentDelivery.limits.expansionDepth` より 1 大きい深さを指定して取得する | `ValidationError` を返す。理由（`depthExceeded`）と上限の値 `ContentDelivery.limits.expansionDepth` を添える。A の内容を返さない |
