# 移行定義を生成する（generateMigration）

CMS に接続せず、認可を持たない。断りが無い限り、既存の移行定義の列は M1 で、M1 を適用した構造は、テキストの項目 title を持つモデル article。「スキーマ」は変更後のスキーマを指す。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-schema-generateMigration-01 | スキーマは、article に任意のテキストの項目 lead を加えている | 判断なしで生成する | `generated`。操作は lead の `addField`（`fill` は `null`）だけ。M1 が加えた定義の操作を含まない |
| TC-schema-generateMigration-02 | スキーマの構造は M1 を適用した構造と同じ。文字数の制約、フォーム、表示名称だけが変わっている | 生成する | `noChange` |
| TC-schema-generateMigration-03 | スキーマは、article に任意のテキストの項目 lead を加えている | 項目 title のキーの変更の判断を指定して生成する | `invalidDecisions`。その判断と理由 `unmatched` を返す。操作を生成しない |
| TC-schema-generateMigration-04 | スキーマは、article に必須の数値の項目 price を加えている | 判断なしで生成する | `needsDecisions`。price の補完値の問い（`fill`）を返す。問いの案内は空。操作を生成しない |
| TC-schema-generateMigration-05 | 同上 | 補完値 0 の判断を指定して生成する | `generated`。price の `addField` は `fill` に数値 0 を持つ |
| TC-schema-generateMigration-06 | 同上 | 補完しない判断（`null`）を指定して生成する | `generated`。price の `addField` の `fill` は `null` |
| TC-schema-generateMigration-07 | スキーマは、title を削除し、同じ階層に任意のテキストの項目 heading を加えている | 判断なしで生成する | `needsDecisions`。title のキーの変更の問い（`rename`）を、候補 heading とともに返す |
| TC-schema-generateMigration-08 | 同上 | キーの変更（heading へ）の判断を指定して生成する | `generated`。操作は title から heading への `renameField` だけ |
| TC-schema-generateMigration-09 | 同上 | キーの変更にしない判断（`null`）だけを指定して生成する | `needsDecisions`。heading に値の複写を加えるかの問い（`copy`、候補 title）と、title を取り除くことの確認の問い（`confirmLoss`、`removed`）を返す。確認の問いの案内は `irreversibleLoss` |
| TC-schema-generateMigration-10 | 同上 | キーの変更にしない判断、値の複写を加えない判断（`null`）、損失の確認を指定して生成する | `generated`。heading の `addField` と、title の `removeField`（`lossConfirmed`）をこの順に持つ |
| TC-schema-generateMigration-11 | 同上 | キーの変更にしない判断、heading の複写元を title とする判断、損失の確認を指定して生成する | `generated`。heading の `addField`、title から heading への `copyValue`、title の `removeField`（`lossConfirmed`）をこの順に持つ |
| TC-schema-generateMigration-12 | スキーマは、title を削除し、同じ階層に任意の数値の項目 price を加えている | 判断なしで生成する | `needsDecisions`。price に値の複写を加えるかの問い（`copy`、候補 title）と、title を取り除くことの確認の問い（`confirmLoss`、`removed`）を返す。`rename` の問いを返さない |
| TC-schema-generateMigration-13 | 同上 | price の複写元を title とする判断と、損失の確認を指定して生成する | `generated`。price の `addField`、title から price への `copyValue`、title の `removeField`（`lossConfirmed`）をこの順に持つ |
| TC-schema-generateMigration-14 | スキーマは、title に非推奨を付け、同じ階層に任意の数値の項目 price を加えている | 判断なしで生成する | `needsDecisions`。price に値の複写を加えるかの問い（`copy`、候補 title）を返す |
| TC-schema-generateMigration-15 | 同上 | price の複写元を title とする判断を指定して生成する | `generated`。price の `addField` と、title から price への `copyValue` をこの順に持つ。title の操作を持たない |
| TC-schema-generateMigration-16 | スキーマは、title を必須にしている | 補完値の判断を指定して生成する | `generated`。title の `changeRequired`（`required`、指定した `fill`）を持つ |
| TC-schema-generateMigration-17 | M1 の構造は、コンテンツ参照の項目を持つ。スキーマは、その項目に許可するモデルを変えている | 判断なしで生成する | `generated`。`changeReferenceModels` を持つ |
| TC-schema-generateMigration-18 | スキーマは、title を数値に変えている | 判断なしで生成する | `needsDecisions`。確認の問い（`confirmLoss`）を返す。`cause` は `kindChangeWithRule` で、変換元がテキスト、変換先が数値、変換できない値が生じ得る（`lossless` が偽）変換規則を持つ |
| TC-schema-generateMigration-19 | 同上 | 損失の確認を指定して生成する | `generated`。title の `changeFieldKind`（`lossConfirmed`）を持つ |
| TC-schema-generateMigration-20 | スキーマは、title をリッチテキストに変えている | 判断なしで生成する | `needsDecisions`。確認の問い（`confirmLoss`）を返す。`cause` は `kindChangeWithoutRule` で、変換元のテキストと変換先のリッチテキストを持つ |
| TC-schema-generateMigration-21 | 同上 | 損失の確認を指定して生成する | `generated`。同じキー title の `removeField`（`lossConfirmed`）と `addField` をこの順に持ち、`changeFieldKind` を持たない |
| TC-schema-generateMigration-22 | M1 の構造は、選択肢 a、b を持つ単一選択の項目を持つ。スキーマは a を削除している | 判断なしで生成する | `needsDecisions`。置き換え先の問い（`optionReplacement`）を、候補 b とともに返す。問いの案内は空 |
| TC-schema-generateMigration-23 | 同上 | 置き換え先 b の判断を指定して生成する | `generated`。a の `removeOption`（置き換え先は選択肢 b）を持つ |
| TC-schema-generateMigration-24 | 同上 | 置き換え先を未設定にする判断（`null`）だけを指定して生成する | `needsDecisions`。a の値を失うことの確認の問い（`confirmLoss`、`optionUnset`）を、案内 `irreversibleLoss` とともに返す。操作を生成しない |
| TC-schema-generateMigration-25 | 同上 | 置き換え先を未設定にする判断と、損失の確認を指定して生成する | `generated`。a の `removeOption`（置き換え先は未設定、`lossConfirmed`）を持つ |
| TC-schema-generateMigration-26 | 同上 | 置き換え先に、項目に無い選択肢 z を指定して生成する | `invalidDecisions`。その判断と理由 `outOfRange` を返す。操作を生成しない |
| TC-schema-generateMigration-27 | スキーマは、モデル article を削除している | article の損失の確認だけを指定して生成する | `generated`。操作は article の `removeModel`（`lossConfirmed`）だけ。title の `removeField` を持たない |
| TC-schema-generateMigration-28 | M1 の構造は、種類 quote を持つブロックの項目を持つ。スキーマは quote を削除している | 損失の確認を指定して生成する | `generated`。quote の `removeBlockType`（`lossConfirmed`）を持つ |
| TC-schema-generateMigration-29 | article は複数件モデル。スキーマは article を単一モデルに変えている | 判断なしで生成する | `needsDecisions`。確認の問い（`confirmLoss`、`cardinalityChange`）を、案内 `irreversibleLoss` と `stagedChange` とともに返す |
| TC-schema-generateMigration-30 | 同上 | 損失の確認を指定して生成する | `generated`。同じキー article の `removeModel`（`lossConfirmed`）と `addModel` をこの順に、続けて項目ごとの `addField` を持つ |
| TC-schema-generateMigration-31 | article は複数件モデル。別のモデル page が、article を許可するコンテンツ参照の項目 related を持つ。スキーマは article を単一モデルに変えている | 損失の確認を指定して生成する | `generated`。article の `removeModel` と `addModel` の後に、related の `models` を変更後のスキーマの値にする `changeReferenceModels` を持つ。操作を M1 の構造へ適用した結果は、スキーマのコンテンツの構造と一致する |
| TC-schema-generateMigration-32 | M1 の構造は、モデル直下の項目 caption と、繰り返しの項目 items を持つ。スキーマは caption を items の中へ移している | 判断なしで生成する | `needsDecisions`。モデル直下の caption を取り除くことの確認の問い（`confirmLoss`）を返す。`cause` は `movedAcrossLevels` で、items の中の caption の位置を持つ。案内は `irreversibleLoss` と `stagedChange`。`rename` の問いを返さない |
| TC-schema-generateMigration-33 | 同上 | 損失の確認を指定して生成する | `generated`。items の階層の caption の `addField` と、モデル直下の caption の `removeField`（`lossConfirmed`）をこの順に持ち、`renameField` を持たない |
| TC-schema-generateMigration-34 | スキーマは、項目の追加と、別の項目の削除と、モデルの追加を含む。必要な判断をすべて指定する | 同じ引数で二度生成する | 二度とも同じ操作の並びを返す。`addModel` が先頭に、`addField` が `removeField` より前に並ぶ |
| TC-schema-generateMigration-35 | スキーマは、M1 が加えた項目 title を持たない。既存の M1 は title を扱う記述のまま | 損失の確認を指定して生成する | `generated`。title の `removeField` を持つ。M1 を不整合として返さない |
| TC-schema-generateMigration-36 | スキーマが、同じモデルに同じキーの項目を二つ持つ | 生成する | 生成せず、`SchemaIssue` の列を返す |
| TC-schema-generateMigration-37 | 既存の列の M2 が、その時点の構造に無いキーを指す | 生成する | 生成せず、`MigrationIssue` の列（該当する操作、`unknownKey`、無いキーを指す `missing`）を返す |
| TC-schema-generateMigration-38 | 既存の列に、同じ名前の移行定義が二つある | 生成する | 生成せず、`MigrationIssue` の列（その名前と位置を示す `duplicateName`）を返す |
| TC-schema-generateMigration-39 | スキーマは、title を削除し、同じ階層に任意のテキストの項目 heading を加えている | title のキーの変更（heading へ）の判断を二つ指定して生成する | `invalidDecisions`。二つ目の判断と理由 `duplicate` を返す。操作を生成しない |
