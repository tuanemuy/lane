# 版の復元の確認を求める

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-reviewRevisionRestore-01 | A は通常の管理対象にあり、下書きを持つ。`version` は 3。A の履歴に R1 がある | メンバーが、A と R1 を指定して復元の確認を求める | A の `version` 3 と、R1 と最新の内容の項目ごとの差分を返す。A と R1 は変わらない |
| TC-content-reviewRevisionRestore-02 | R1 は、任意の参照で T を参照する。T は完全削除されている | メンバーが、A と R1 を指定して復元の確認を求める | エラーにしない。復元する値は、T への参照を版の値のまま持つ。未設定にしない |
| TC-content-reviewRevisionRestore-03 | A のモデルは非推奨 | メンバーが、A と R1 を指定して復元の確認を求める | エラーにしない。復元できない理由として、モデルが非推奨であることを返す |
| TC-content-reviewRevisionRestore-04 | R1 は、履歴の版の整理で削除されている | メンバーが、A と R1 を指定して復元の確認を求める | `NotFoundError` を返す |
| TC-content-reviewRevisionRestore-05 | A はごみ箱にある | メンバーが、A と R1 を指定して復元の確認を求める | エラーにしない。復元できない理由として、ごみ箱にあることを返す |
| TC-content-reviewRevisionRestore-06 | R9 は、別のコンテンツ B の履歴の版 | メンバーが、A と R9 を指定して復元の確認を求める | `NotFoundError` を返す |
| TC-content-reviewRevisionRestore-07 | A のモデルの項目 `legacy` は非推奨。A の最新の内容の `legacy` は「a」、R1 の `legacy` は「b」 | メンバーが、A と R1 を指定して復元の確認を求める | 復元する値の `legacy` は「a」。差分は、`legacy` を含まない |
