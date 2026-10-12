# 失敗した派生形式の生成をやり直す（`retryDerivative`）

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-media-retryDerivative-01 | `content:edit` を持つメンバー。派生形式 k が `failed`（理由 `processing_failed`、または `stalled`）の画像がある | k を指定してやり直す | 保存後の派生形式の記録を返す。k は `generating` で、生成の対象（定義と世代）は失敗の前と同じ、始め直しの回数は 0。k を指す `media.derivative_regeneration_requested` を 1 件記録する。ほかの派生形式は変わらない |
| TC-media-retryDerivative-03 | `content:edit` を持つメンバー。派生形式 k が `failed` で、変更前の派生形式を持つ | k を指定してやり直す | k は `generating` で、変更前の派生形式を引き継ぐ。取得結果と配信は、変更前の派生形式を返し続ける |
| TC-media-retryDerivative-04 | `content:edit` を持つメンバー。派生形式 k が `generating` または `ready` の画像がある、または k の状態が無い | k を指定してやり直す | `BusinessRuleError`（`DerivativeNotFailed`）。状態は変わらず、イベントを記録しない |
| TC-media-retryDerivative-05 | `content:edit` を持つメンバー。文書のメディアがある | 派生形式のキーを指定してやり直す | `BusinessRuleError`（`MediaNotImage`）。イベントを記録しない |
| TC-media-retryDerivative-06 | `content:edit` を持つメンバー | 存在しないメディアの ID を指定してやり直す | `NotFoundError` |
| TC-media-retryDerivative-07 | `content:edit` を持たないメンバー。派生形式 k が `failed` の画像がある | k を指定してやり直す | `ForbiddenError`（理由 `PERMISSION_MISSING`）。不足する権限 `content:edit` を示す。k は `failed` のまま、イベントを記録しない |
| TC-media-retryDerivative-09 | `content:edit` を持つメンバー。派生形式 k が `failed` の画像がある。画像の `Media` を読んだ後、派生形式の記録を読む前に、そのメディアの削除が確定する | k を指定してやり直す | `NotFoundError`。イベントを記録しない |
