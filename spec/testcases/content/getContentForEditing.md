# 編集するコンテンツを取得する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-getContentForEditing-01 | A は公開中で、下書きを持つ。最後に A を変えた操作は、メンバー甲の下書きの保存 | メンバーが、A のコンテンツID で取得する | `version`、置き場所（通常）、最後の変更（下書きの保存、甲の名称、日時）、公開内容の値と `violations`、下書きの値と `issues` を返す。最新の内容は下書きと示す |
| TC-content-getContentForEditing-02 | A の公開内容の値は、スキーマの反映の移行で変換された。A の下書きは、その後に保存された | メンバーが、A を取得する | 公開内容の、移行で変換された日時と、変換後のスキーマの版を返す。下書きの、移行で変換された日時とスキーマの版は無い |
| TC-content-getContentForEditing-03 | A は公開中で、下書きを持たない | メンバーが、A を取得する | 公開内容を返し、下書きが無いことを返す。最新の内容は公開内容と示す |
| TC-content-getContentForEditing-04 | A は未公開で、下書きが必須項目の欠落を持つ | メンバーが、A を取得する | 下書きの `issues` に、該当する項目の入力エラーを返す |
| TC-content-getContentForEditing-05 | C はごみ箱にある | メンバーが、C を取得する | エラーにしない。置き場所（ごみ箱）、移動した日時、移動した実行者の名称を添えて、下書きを返す |
| TC-content-getContentForEditing-06 | 単一モデル S の通常の管理対象に、コンテンツがある | メンバーが、S のキーで取得する | S のコンテンツを返す |
| TC-content-getContentForEditing-07 | 単一モデル S の通常の管理対象に、コンテンツが無い | メンバーが、S のキーで取得する | エラーにしない。コンテンツが無いことを返す |
| TC-content-getContentForEditing-09 | トークンは `content_manage` の許可を持たない | トークンで、A を取得する | `ForbiddenError`（`GRANT_MISSING`）を返す |
| TC-content-getContentForEditing-10 | 指定したコンテンツID のコンテンツが無い | メンバーが、そのコンテンツID で取得する | `NotFoundError` を返す。コンテンツID を示す |
