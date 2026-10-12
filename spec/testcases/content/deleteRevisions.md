# 履歴の版を削除する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-deleteRevisions-01 | A は公開中で、下書きを持つ。A の履歴に R1、R2、R3 がある | 削除権限を持つメンバーが、A と R1、R2 を指定して削除する | R1 と R2 は無くなり、履歴の一覧に現れない。削除した版の ID として R1 と R2 を返す。R3、A の公開内容、下書き、`version` は変わらない。`content.publication_changed` を記録しない |
| TC-content-deleteRevisions-02 | A の履歴の版 R1 は、メディア Md を使い、コンテンツ T を必須の参照で参照する。Md を使う内容と、T を参照する内容は、R1 だけ | 削除権限を持つメンバーが、A と R1 を指定して削除する | R1 は無くなる。Md の使用箇所と、T の参照元に、A が現れない |
| TC-content-deleteRevisions-03 | C はごみ箱にあり、履歴の版 R1 を持つ | 削除権限を持つメンバーが、C と R1 を指定して削除する | R1 は無くなる。C はごみ箱のまま、下書きは変わらない |
| TC-content-deleteRevisions-04 | A の履歴に R1 がある。R2 は、メンバーが確認した後に、履歴の版の整理で削除されている | 削除権限を持つメンバーが、A と R1、R2 を指定して削除する | `NotFoundError` を返す。R2 の ID を添える。R1 は残る |
| TC-content-deleteRevisions-05 | A の履歴に R1 がある。R9 は、別のコンテンツ B の履歴の版 | 削除権限を持つメンバーが、A と R1、R9 を指定して削除する | `NotFoundError` を返す。R9 の ID を添える。R1 と R9 は残る |
| TC-content-deleteRevisions-06 | A は、メンバーが確認した後に完全削除されている | 削除権限を持つメンバーが、A と R1 を指定して削除する | `NotFoundError` を返す |
| TC-content-deleteRevisions-08 | メンバーは、編集権限を持ち、削除権限を持たない | A と R1 を指定して削除する | `ForbiddenError`（`PERMISSION_MISSING`）を返す。不足する権限（`content:delete`）を添える。R1 は残る |
| TC-content-deleteRevisions-09 | A の履歴に R1 と R2 がある。削除が R1 と R2 を読んだ後、確定までに、履歴の版の整理が R2 を削除した | 削除権限を持つメンバーが、A と R1、R2 を指定して削除する | `ConflictError`（`REVISION_MISSING`）を返す。R2 の ID を示す。R1 は残る |
