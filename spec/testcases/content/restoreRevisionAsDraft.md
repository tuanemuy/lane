# 版を下書きとして復元する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-restoreRevisionAsDraft-01 | A は公開中で、下書きを持つ。`version` は 3。A の履歴に、公開内容だった版 R1 がある | 編集権限を持つメンバーが、A、`version` 3、R1 を指定して復元する | A の下書きの値は R1 の値になる。下書きを作った操作は履歴からの復元。置き換え前の下書きは、下書きだった版として履歴に残る。A は公開中のまま、公開内容と、公開内容の更新日時は変わらない。R1 は履歴に残る。`version` 4 と、下書きの入力エラーを返す。公開の変更の記録を追記せず、`content.publication_changed` を記録しない |
| TC-content-restoreRevisionAsDraft-02 | A は公開中で、下書きを持たない。A の履歴に R1 がある | 編集権限を持つメンバーが、A と R1 を指定して復元する | A に、R1 の値の下書きができる。新しい履歴の版は作られない。公開内容は変わらない |
| TC-content-restoreRevisionAsDraft-03 | A は未公開。A の履歴に、下書きだった版 R1 がある | 編集権限を持つメンバーが、A と R1 を指定して復元する | A の下書きの値は R1 の値になる。A は未公開のまま |
| TC-content-restoreRevisionAsDraft-04 | R1 の値は、現行スキーマの必須の項目を欠く | 編集権限を持つメンバーが、A と R1 を指定して復元する | エラーにしない。復元し、必須項目の欠落を下書きの入力エラーとして返す |
| TC-content-restoreRevisionAsDraft-05 | R1 は、任意の参照で T を参照する。T は完全削除されている | 編集権限を持つメンバーが、A と R1 を指定して復元する | 復元する。T への参照は、版の値のまま、参照切れとして下書きに残る。未設定にならない |
| TC-content-restoreRevisionAsDraft-06 | R1 は、メンバーが確認した後に、履歴の版の整理で削除されている | 編集権限を持つメンバーが、A と R1 を指定して復元する | `NotFoundError` を返す。A の下書きは変わらない |
| TC-content-restoreRevisionAsDraft-07 | メンバーが A を `version` 3 で確認した後、ほかの操作で A の最新の内容が変わり、`version` は 4 | メンバーが、A、`version` 3、R1 を指定して復元する | `ConflictError`（`STALE_VERSION`）を返す。A の下書きは変わらない |
| TC-content-restoreRevisionAsDraft-08 | R9 は、別のコンテンツ B の履歴の版 | 編集権限を持つメンバーが、A と R9 を指定して復元する | `NotFoundError` を返す。A の下書きは変わらない |
| TC-content-restoreRevisionAsDraft-09 | A はごみ箱にある | 編集権限を持つメンバーが、A、現在の `version`、R1 を指定して復元する | `BusinessRuleError`（`ContentInTrash`）を返す。A は変わらない |
| TC-content-restoreRevisionAsDraft-10 | A のモデルは非推奨 | 編集権限を持つメンバーが、A と R1 を指定して復元する | `BusinessRuleError`（`ModelDeprecated`）を返す。A の下書きは変わらない |
| TC-content-restoreRevisionAsDraft-13 | メンバーは編集権限を持たない | A と R1 を指定して復元する | `ForbiddenError`（`PERMISSION_MISSING`）を返す。不足する権限（`content:edit`）を添える。A の下書きは変わらない |
| TC-content-restoreRevisionAsDraft-14 | A のモデルの項目 `legacy` は非推奨。A の最新の内容の `legacy` は「a」、R1 の `legacy` は「b」。R1 の非推奨でない項目 `title` は「T1」 | 編集権限を持つメンバーが、A と R1 を指定して復元する | A の下書きの `title` は「T1」になり、`legacy` は「a」のまま |
| TC-content-restoreRevisionAsDraft-17 | メンバー甲が A を `version` 3 で確認した後、メンバー乙が R1 を復元し、A の `version` は 4 | 甲が、A、`version` 3、R1 を指定して復元する | `ConflictError` にしない。成功として現在の A を返す。`version` は 4 のまま。履歴の版は増えない |
