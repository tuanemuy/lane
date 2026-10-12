# スキーマの識別子を求める（computeSchemaFingerprints）

CMS に接続せず、認可を持たない。基準のスキーマ S は、画像の項目を持つモデル article、メディアもリッチテキストも持たないモデル author、article から author へのコンテンツ参照、フォーム contact、派生形式 thumb を持つ。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-schema-computeSchemaFingerprints-01 | スキーマ S | 識別子を求める | スキーマ全体、article、author、contact の識別子を返す。同じスキーマからは、同じ識別子を返す |
| TC-schema-computeSchemaFingerprints-02 | S から、article の項目の文字数の制約と一意性、表示名称、項目の順序、説明、初期値、非推奨の指定、contact の項目の `mustMatch`、派生形式 thumb の大きさと品質だけを変えたスキーマ | 識別子を求める | どの識別子も、S の識別子と等しい |
| TC-schema-computeSchemaFingerprints-04 | S から、派生形式のキーを thumb から small に変えたスキーマ | 識別子を求める | スキーマ全体と article の識別子は S と異なる。author と contact の識別子は S と等しい |
| TC-schema-computeSchemaFingerprints-05 | S から、article に項目を一つ加えたスキーマ | 識別子を求める | スキーマ全体と article の識別子は S と異なる。author と contact の識別子は S と等しい |
| TC-schema-computeSchemaFingerprints-06 | S から、author の項目を必須にしたスキーマ | 識別子を求める | スキーマ全体と author の識別子は S と異なる。author を参照する article の識別子は S と等しい |
| TC-schema-computeSchemaFingerprints-07 | S から、contact の項目のキーを変えたスキーマ | 識別子を求める | スキーマ全体と contact の識別子は S と異なる。article と author の識別子は S と等しい |
| TC-schema-computeSchemaFingerprints-08 | 同じモデルに同じキーの項目を二つ持つスキーマ | 識別子を求める | 識別子を返さず、「スキーマを検証する」と同じ不整合を返す |
