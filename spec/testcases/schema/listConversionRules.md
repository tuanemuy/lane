# 変換規則を一覧する（listConversionRules）

CMS に接続せず、認可を持たない。入力は無い。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-schema-listConversionRules-01 | なし | 一覧する | `ConversionRule` の列を返す。規則ごとに、変換元と変換先の値の種類と、変換できない値が生じ得るかどうか（`lossless`）を持つ |
| TC-schema-listConversionRules-02 | なし | 一覧する | 数値からテキスト、日付から日時、コンテンツ参照（一件）からコンテンツ参照（複数件）の規則は、`lossless` が真 |
| TC-schema-listConversionRules-03 | なし | 一覧する | テキストから数値、複数選択から単一選択、コンテンツ参照（複数件）からコンテンツ参照（一件）の規則は、`lossless` が偽 |
| TC-schema-listConversionRules-04 | なし | 一覧する | リッチテキスト、メディア、項目のまとまり、繰り返し、ブロックと、ほかの値の種類との組み合わせを含まない。数値から日付のように、変換規則の無い組み合わせを含まない |
| TC-schema-listConversionRules-05 | なし | 一覧し、テキストの項目を数値に変えるスキーマで「移行定義を生成する」の問いを得る | 問いが持つ変換規則は、一覧のテキストから数値の規則と等しい |
