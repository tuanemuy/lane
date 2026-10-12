# ごみ箱のコンテンツを一覧する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-listTrashedContents-01 | モデル M1 の C1 と、モデル M2 の C2 がごみ箱にある。M1 の A は通常の管理対象にある。ごみ箱を保持する日数は 30 | メンバーが、モデルを指定せずに、ごみ箱を一覧する | C1 と C2 を返し、A を含めない。総件数は 2。コンテンツごとに、モデル、コンテンツID、`version`、一覧表示用の項目の最新の内容の値、移動した日時、移動した実行者の名称、移動した日時の 30 日後の時点（自動整理で完全削除される時期）を返す |
| TC-content-listTrashedContents-02 | M1 の C1 と、M2 の C2 がごみ箱にある | メンバーが、M1 のキーで絞り込んで一覧する | C1 だけを返す。総件数は 1 |
| TC-content-listTrashedContents-03 | C1 がごみ箱にある。C1 を移動した後に、ごみ箱を保持する日数を 30 から 10 に変更した | メンバーが、ごみ箱を一覧する | C1 の完全削除される時期として、移動した日時の 10 日後の時点を返す |
| TC-content-listTrashedContents-04 | ごみ箱にコンテンツが無い | メンバーが、ごみ箱を一覧する | エラーにしない。空の一覧を返す |
| TC-content-listTrashedContents-05 | M1 の C1 と、M2 の C2 がごみ箱にある。トークンは `content_manage` の許可を持ち、範囲は M1 だけを含む | トークンで、モデルを指定せずに、ごみ箱を一覧する | C1 だけを返す。総件数は 1 |
| TC-content-listTrashedContents-06 | トークンは `content_manage` の許可を持ち、範囲は M1 だけを含む | トークンで、M2 のキーで絞り込んで、ごみ箱を一覧する | `ForbiddenError`（`OUT_OF_SCOPE`）を返す |
| TC-content-listTrashedContents-07 | トークンは `content_manage` の許可を持たない | トークンで、ごみ箱を一覧する | `ForbiddenError`（`GRANT_MISSING`）を返す |
| TC-content-listTrashedContents-08 | ごみ箱の C1 のモデルは非推奨 | メンバーが、ごみ箱を一覧する | C1 を返し、モデルが非推奨であることを返す |
| TC-content-listTrashedContents-09 | 現行スキーマに、キー `gone` のモデルが無い | メンバーが、`gone` で絞り込んで、ごみ箱を一覧する | `NotFoundError` を返す |
