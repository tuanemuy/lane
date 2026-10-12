# 履歴の版とともにメディアを削除する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-deleteMediaWithRevisions-01 | 非公開のメディア Md を、A の履歴の版 R1 と、B の履歴の版 R2 だけが使う。メンバーは、削除権限と、メディアの削除に必要な権限を持つ | メンバーが、Md と、対象に加える版 R1、R2 を指定して削除する | R1、R2、Md を同じ確定で削除する。削除した版の ID として R1 と R2 を返す。A と B の公開内容、下書き、ほかの版は変わらない。`content.publication_changed` を記録しない。確定の後に、Md の実体を消去する |
| TC-content-deleteMediaWithRevisions-02 | Md を、A の履歴の版 R1 と、C の下書きが使う | 削除権限と、メディアの削除に必要な権限を持つメンバーが、Md と R1 を指定して削除する | `ConflictError`（`MEDIA_IN_USE`）を返す。R1 も Md も残る |
| TC-content-deleteMediaWithRevisions-03 | Md を、A の履歴の版 R1 と R2 が使う | 削除権限と、メディアの削除に必要な権限を持つメンバーが、Md と R1 だけを指定して削除する | `ConflictError`（`MEDIA_IN_USE`）を返す。R1 も Md も残る |
| TC-content-deleteMediaWithRevisions-04 | Md を、A の履歴の版 R1 だけが使う。メンバーが確認した後に、R1 は履歴の版の整理で無くなった | 削除権限と、メディアの削除に必要な権限を持つメンバーが、Md と R1 を指定して削除する | `NotFoundError` を返す。R1 の ID を添える。Md は残る |
| TC-content-deleteMediaWithRevisions-05 | Md を、A の履歴の版 R1 だけが使う。メンバーは、メディアの削除に必要な権限を持ち、削除権限を持たない | メンバーが、Md と R1 を指定して削除する | `ForbiddenError`（`PERMISSION_MISSING`）を返す。`content:delete` を示す。何も削除しない |
| TC-content-deleteMediaWithRevisions-06 | Md を、A の履歴の版 R1 だけが使う。メンバーは、削除権限を持ち、メディアの削除に必要な権限を持たない | メンバーが、Md と R1 を指定して削除する | `ForbiddenError`（`PERMISSION_MISSING`）を返す。その権限を示す。何も削除しない |
| TC-content-deleteMediaWithRevisions-07 | ワークスペースはスキーマの反映の適用中 | 削除権限と、メディアの削除に必要な権限を持つメンバーが、Md と R1 を指定して削除する | `UnavailableError`（`SCHEMA_APPLYING`）を返す。何も削除しない |
| TC-content-deleteMediaWithRevisions-08 | 指定したメディアの ID のメディアが無い | 削除権限を持つメンバーが、そのメディアと R1 を指定して削除する | `NotFoundError` を返す。R1 は残る |
| TC-content-deleteMediaWithRevisions-09 | Md を、A の履歴の版 R1 だけが使う。メンバーが Md の削除を確認した後に、Md が公開に変わり、`version` が進んだ | 削除権限と、メディアの削除に必要な権限を持つメンバーが、確認したときの Md の `version` と R1 を指定して削除する | `ConflictError`（`STALE_VERSION`）を返す。R1 も Md も残る |
| TC-content-deleteMediaWithRevisions-10 | Md を、A の履歴の版 R1 だけが使う。削除が R1 を読んだ後、確定までに、履歴の版の整理が R1 を削除した | 削除権限と、メディアの削除に必要な権限を持つメンバーが、Md と R1 を指定して削除する | `ConflictError`（`REVISION_MISSING`）を返す。R1 の ID を示す。Md は残る |
