# 入力を最新の内容と比較する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-compareInputWithLatest-01 | メンバー甲が A を `version` 3 で読み取った後、メンバー乙が A の下書きを保存し、A の `version` は 4 | 甲が、A と自分の入力を指定して比較する | 乙が保存した最新の内容の値と `version` 4、甲の入力との項目ごとの差分、最後に内容を変えた操作として、下書きの保存、乙の名称、保存の日時を返す。A は変わらない |
| TC-content-compareInputWithLatest-02 | 甲が A を読み取った後、トークンによる値の書き換えで A の下書きが変わった | 甲が、A と自分の入力を指定して比較する | 最後に内容を変えた操作として、値の書き換え、トークンを指す値（名称または発行の経路）と発行者または紐付くメンバーの名前、書き換えの日時を返す |
| TC-content-compareInputWithLatest-03 | 甲が、公開中で下書きを持つ A を読み取った後、乙が A の下書きを破棄した | 甲が、A と自分の入力を指定して比較する | 最新の内容として公開内容の値を返す。最後に内容を変えた操作として、下書きの破棄、乙の名称、破棄の日時を返す |
| TC-content-compareInputWithLatest-04 | 甲の入力の値は、A の最新の内容の値と等しい | 甲が、A と自分の入力を指定して比較する | 差分は空 |
| TC-content-compareInputWithLatest-05 | A はごみ箱へ移動されている | 編集権限を持つメンバーが、A と入力を指定して比較する | `BusinessRuleError`（`ContentInTrash`）を返す |
| TC-content-compareInputWithLatest-06 | A は完全削除されている | 編集権限を持つメンバーが、A と入力を指定して比較する | `NotFoundError` を返す |
| TC-content-compareInputWithLatest-07 | メンバーは編集権限を持たない | A と入力を指定して比較する | `ForbiddenError`（`PERMISSION_MISSING`）を返す。不足する権限（`content:edit`）を添える |
| TC-content-compareInputWithLatest-08 | 単一モデル S の最初の保存を、メンバー甲と乙が同時に行い、乙の保存がコンテンツ S1 を作った。甲の保存は、二つ目になる操作として拒否された | 甲が、S1 と自分の入力を指定して比較する | S1 の最新の内容の値と `version`、甲の入力との項目ごとの差分、最後に内容を変えた操作として、下書きの保存、乙の名称、保存の日時を返す |
