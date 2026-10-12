# コンテンツを公開する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-publishContent-01 | A は未公開で、入力エラーの無い下書きを持つ。`version` は 1。A を公開内容から参照するコンテンツは無い | 公開権限を持つメンバーが、A と確認した `version` 1 を指定して公開する | A は公開中で、下書きを持たない。公開内容の値は、公開前の下書きの値。公開内容の更新日時は公開の時刻で、`violations` は空。入力の操作の ID、A の `version` 2 と「公開」を返す。履歴の版は作られない。公開の変更の記録を一件追記する（入力の操作の ID、`operation` は `published`、実行者、`changeList.changes` は A の `published`、`changeList.referrers` は空、確定の時点のスキーマの版）。`content.publication_changed` を一つ記録する。ペイロードは、操作の ID だけ |
| TC-content-publishContent-02 | B は公開中で、下書きを持つ | 公開権限を持つメンバーが、B と確認した `version` を指定して公開する | B の公開内容は下書きの値に置き換わり、B は下書きを持たない。置き換え前の公開内容は、公開内容だった版として履歴に残る。版にした操作は公開。B について「更新」を返す。公開の変更の記録の `changeList.changes` の B の変更は `updated` |
| TC-content-publishContent-03 | B は公開中で、下書きは一意な項目 `slug` を「old」から「new」に変える | 公開権限を持つメンバーが、B を公開する | 公開の変更の記録の `changeList.changes` の B の変更は、`uniqueValues` に `slug` の変更前「old」と変更後「new」を持つ |
| TC-content-publishContent-04 | B は公開中で、下書きを持つ。公開中の R が、公開内容で B を参照している | 公開権限を持つメンバーが、B だけを公開する | 公開の変更の記録の `changeList.referrers` は、R のコンテンツID とモデルのキーを持つ |
| TC-content-publishContent-05 | A の下書きは、非公開のメディア Md を使う。メンバーは、公開権限とメディア公開権限を持つ | メンバーが、A を公開する | A は公開中になり、Md は公開になる。公開したメディアとして Md を返す。Md の使用箇所は、A の公開内容での使用になる |
| TC-content-publishContent-06 | A と B は未公開で、下書きが互いを必須の参照で参照する | 公開権限を持つメンバーが、A と B を一つの操作で公開する | A と B は、どちらも公開中になる。公開の変更の記録は一件で、`changeList.changes` は A と B の変更（どちらも `published`）を持ち、`changeList.referrers` は A と B を含まない。`content.publication_changed` は一つ |
| TC-content-publishContent-07 | A の下書きは、A 自身を参照する | 公開権限を持つメンバーが、A を公開する | A は公開中になる |
| TC-content-publishContent-08 | トークンは `content_manage` の許可と公開権限（`content:publish`）を持ち、範囲に A のモデルを含む | トークンで、A と読み取ったときの `version` を指定して公開する | A は公開中になる。公開内容を作った実行者として、トークンと、トークンを指す値（名称または発行の経路）と、発行者または紐付くメンバーの名前を記録する |
| TC-content-publishContent-09 | A の下書きは、文字数の制約に違反する項目を持つ | 公開権限を持つメンバーが、A を公開する | `BusinessRuleError`（`PublishRejected`）を返す。A の入力エラー（項目と理由）を添える。A は未公開のまま、下書きは変わらない。公開の変更の記録を追記せず、`content.publication_changed` を記録しない |
| TC-content-publishContent-11 | A の下書きは、未公開の T を任意の参照で参照する | 公開権限を持つメンバーが、A だけを公開する | `BusinessRuleError`（`PublishRejected`）を返す。A、項目、参照先 T、理由（公開内容を持たない）を添える。A は未公開のまま |
| TC-content-publishContent-13 | メンバーが A を確認した後、ほかの操作が、A の下書きが参照する T の公開を取り消した。A の `version` は変わっていない | メンバーが、A と確認した `version` を指定して公開する | `BusinessRuleError`（`PublishRejected`）を返す。A、項目、参照先 T を添える。A は未公開のまま |
| TC-content-publishContent-14 | A の下書きの一意な項目 `slug` の値が「x」。同じモデルの公開中の B の公開内容の `slug` も「x」 | 公開権限を持つメンバーが、A を公開する | `BusinessRuleError`（`PublishRejected`）を返す。A、`slug`、値「x」、相手の B を添える。A は未公開のまま |
| TC-content-publishContent-15 | モデル M1 の A と、モデル M2 の C の下書きは、どちらも一意な項目 `slug` の値が「x」 | 公開権限を持つメンバーが、A と C を一つの操作で公開する | A と C は公開中になる |
| TC-content-publishContent-16 | 同じモデルの A と C の下書きの `slug` が、どちらも「x」 | 公開権限を持つメンバーが、A と C を一つの操作で公開する | `BusinessRuleError`（`PublishRejected`）を返す。A と C は、どちらも未公開のまま |
| TC-content-publishContent-17 | B は公開中で、下書きを持たない | 公開権限を持つメンバーが、B と現在の `version` を指定して公開する | `BusinessRuleError`（`PublishRejected`）を返す。B が下書きを持たないことを添える。B は変わらない |
| TC-content-publishContent-18 | A のモデルは非推奨。A は入力エラーの無い下書きを持つ | 公開権限を持つメンバーが、A を公開する | `BusinessRuleError`（`ModelDeprecated`）を返す。A は未公開のまま |
| TC-content-publishContent-19 | A は入力エラーの無い下書きを持ち、非公開のメディア Md を使う。C の下書きは入力エラーを持ち、未公開の T を参照する | 公開権限とメディア公開権限を持つメンバーが、A と C を一つの操作で公開する | `BusinessRuleError`（`PublishRejected`）を返す。C の入力エラーと、T への参照の違反を、どちらも添える。A と C は未公開のまま。Md は非公開のまま。公開の変更の記録を追記せず、`content.publication_changed` を記録しない |
| TC-content-publishContent-20 | 同じモデルの A と C の下書きの `slug` が、どちらも「x」。A の公開の事前の検証の後、確定までに、C の公開が確定した | 公開権限を持つメンバーが、A を公開する | `ConflictError`（`UNIQUE_VALUE_TAKEN`）を返す。A は未公開のまま。C は公開中のまま |
| TC-content-publishContent-21 | A の下書きは、公開中の T を必須の参照で参照する。A の公開の事前の検証の後、確定までに、T の公開取り消しが確定した | 公開権限を持つメンバーが、A を公開する | `ConflictError`（`REFERENCE_NOT_PUBLISHED`）を返す。A は未公開のまま。公開の変更の記録を追記せず、`content.publication_changed` を記録しない |
| TC-content-publishContent-22 | メンバーが A を `version` 1 で確認した後、ほかのメンバーが A の下書きを保存し、`version` は 2 | メンバーが、A と `version` 1 を指定して公開する | `ConflictError`（`STALE_VERSION`）を返す。A を添える。A は未公開のまま |
| TC-content-publishContent-23 | メンバーが A と B を確認した後、ほかのメンバーが A を単独で公開した | メンバーが、A と B を、確認したときの `version` で公開する | `ConflictError`（`STALE_VERSION`）を返す。`PublishRejected` にしない。B は公開されない |
| TC-content-publishContent-24 | メンバーが A を確認した後、A がごみ箱へ移動され、`version` が進んだ | メンバーが、A と確認した `version` を指定して公開する | `ConflictError`（`STALE_VERSION`）を返す。A は公開されない |
| TC-content-publishContent-25 | C はごみ箱にある | `content_manage` の許可と必要な権限を持つトークンで、C と現在の `version` を指定して公開する | `BusinessRuleError`（`ContentInTrash`）を返す。C は公開されない |
| TC-content-publishContent-26 | A と B を確認した後、B が完全削除されている | 公開権限を持つメンバーが、A と B を一つの操作で公開する | `NotFoundError` を返す。B を示す。A は公開されない |
| TC-content-publishContent-27 | A の下書きは、非公開のメディア Md を使う。メンバーは、公開権限を持ち、メディア公開権限を持たない | メンバーが、A を公開する | `ForbiddenError`（`PERMISSION_MISSING`）を返す。Md と、不足する権限（`media:publish`）を添える。A は未公開のまま。Md は非公開のまま |
| TC-content-publishContent-28 | メンバーは、編集権限を持ち、公開権限を持たない | A を公開する | `ForbiddenError`（`PERMISSION_MISSING`）を返す。不足する権限（`content:publish`）を添える。A は未公開のまま |
| TC-content-publishContent-31 | A の下書きは、公開のメディア Md を使う。メンバーが A を確認した後、Md が非公開にされた。メンバーはメディア公開権限を持たない | 公開権限を持つメンバーが、A を公開する | `ForbiddenError`（`PERMISSION_MISSING`）を返す。Md を添える。A は未公開のまま |
| TC-content-publishContent-32 | A の下書きは、公開のメディア Md を使う。A の公開の事前の検証の後、確定までに、Md が非公開にされた | 公開権限を持つメンバーが、A を公開する | `ConflictError`（`MEDIA_NOT_PUBLIC`）を返す。A は未公開のまま。公開の変更の記録を追記せず、`content.publication_changed` を記録しない |
| TC-content-publishContent-34 | A は未公開で、入力エラーの無い下書きを持つ | 公開権限を持つメンバーが、A を公開する。入口が渡す操作の ID は X | 出力の操作の ID は X。追記した公開の変更の記録の `operationId` と、`content.publication_changed` のペイロードの `operationId` は X。出力は通し番号を持たない |
| TC-content-publishContent-35 | A のモデルは非推奨で、A は入力エラーの無い下書きを持つ。C の下書きは入力エラーを持つ | 公開権限を持つメンバーが、A と C を一つの操作で公開する | `BusinessRuleError`（`PublishRejected`）を返す。C の違反を添える。`rejections` は、A の `ModelDeprecated` と、C の `PublishRejected` を、この順に持つ。A も C も公開されない |
| TC-content-publishContent-36 | A の下書きが使う非公開のメディアの数は、`ContentBatch.maxPublishedMedia` を超える。メンバーは、公開権限とメディア公開権限を持つ | メンバーが、A を公開する | `BusinessRuleError`（`PublishRejected`）を返す。新たに公開するメディアの数と上限を添える。A は未公開のまま。どのメディアも非公開のまま |
