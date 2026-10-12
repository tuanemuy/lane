# コンテンツを完全削除する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-purgeContent-01 | C はごみ箱にあり、履歴の版を持つ。A は通常の管理対象にある未公開のコンテンツ。C の下書きは、メディア Md を使い、コンテンツ T を任意の参照で参照する | 削除権限を持つメンバーが、C と A を、確認した `version` を指定して完全削除する | C と A は、下書きと履歴の版とともに無くなる。削除した対象として C と A を返す。C と A は、取得、一覧、ごみ箱、参照先の候補に現れない。Md と T は残る。C の下書きと履歴の版のメディアの使用箇所は無くなる。公開の変更の記録を追記せず、`content.publication_changed` を記録しない |
| TC-content-purgeContent-02 | B は公開中で、履歴の版を持つ。メンバーは、削除権限と公開権限を持つ | メンバーが、B を完全削除する | B は、公開内容、下書き、履歴の版とともに無くなる。入力の操作の ID と、公開中だった対象として B を返す。公開の変更の記録を一件追記する（入力の操作の ID、`operation` は `purged`、実行者、`changeList.changes` は B の `unpublished`、確定の時点のスキーマの版）。`content.publication_changed` を一つ記録する |
| TC-content-purgeContent-03 | A と D は未公開で、下書きが互いを必須の参照で参照する | 削除権限を持つメンバーが、A と D を一つの操作で完全削除する | A と D は、どちらも無くなる |
| TC-content-purgeContent-04 | C1 と C2 はごみ箱にあり、保持方針の期限を過ぎている。下書きが互いを必須の参照で参照し、ごみ箱の完全削除は、どちらもごみ箱に残した | 削除権限を持つメンバーが、C1 と C2 を一つの操作で完全削除する | C1 と C2 は、どちらも無くなる |
| TC-content-purgeContent-05 | 対象でない G が、下書きの任意の参照で A を参照する | 削除権限を持つメンバーが、A だけを完全削除する | A は無くなる。G は残り、G の下書きは変わらない |
| TC-content-purgeContent-06 | 単一モデル S のコンテンツ S1 は未公開 | 削除権限を持つメンバーが、S1 を完全削除する | S1 は無くなる。S は、通常の管理対象にコンテンツを持たない |
| TC-content-purgeContent-07 | 対象でない D が、下書きの必須の参照で A を参照する。C はごみ箱にあり、参照されていない | 削除権限を持つメンバーが、A と C を一つの操作で完全削除する | `BusinessRuleError`（`PurgeRejected`）を返す。対象 A、参照元 D、内容の種類（下書き）、参照元の置き場所（通常）を添える。A と C は、どちらも残る |
| TC-content-purgeContent-09 | ごみ箱にある F が、下書きの本文のリンクで A を参照する | 削除権限を持つメンバーが、A だけを完全削除する | `BusinessRuleError`（`PurgeRejected`）を返す。対象 A、参照元 F、内容の種類（下書き）、参照元の置き場所（ごみ箱）を添える。A と F は残る |
| TC-content-purgeContent-10 | B は公開中。対象でない公開中の R2 が、公開内容の必須の参照で B を参照する。メンバーは、削除権限と公開権限を持つ | メンバーが、B だけを完全削除する | `BusinessRuleError`（`WithdrawRejected`）を返す。B は公開中のまま残る。公開の変更の記録を追記せず、`content.publication_changed` を記録しない |
| TC-content-purgeContent-11 | A の完全削除の事前の確認の後、確定までに、A を必須の参照で参照する D の下書きの保存が確定した | 削除権限を持つメンバーが、A を完全削除する | `ConflictError`（`REFERENCED`）を返す。A は残る |
| TC-content-purgeContent-12 | A は未公開、B は公開中。メンバーは、削除権限を持ち、公開権限を持たない | メンバーが、A と B を、現在の `version` で一つの操作で完全削除する | `ForbiddenError`（`PERMISSION_MISSING`）を返す。不足する権限（`content:publish`）と、該当する対象 B を添える。A と B は、どちらも残る |
| TC-content-purgeContent-13 | メンバーが未公開の A を確認した後、A が公開され、A の `version` が進んだ。メンバーは、削除権限を持ち、公開権限を持たない | メンバーが、A と確認したときの `version` を指定して完全削除する | `ConflictError`（`STALE_VERSION`）を返す。`ForbiddenError` にしない。A は公開中のまま残る |
| TC-content-purgeContent-14 | A と C を確認した後、C がほかの操作で完全削除されている | 削除権限を持つメンバーが、A と C を一つの操作で完全削除する | `NotFoundError` を返す。C を添える。A は残る |
| TC-content-purgeContent-15 | メンバーが A を `version` 1 で確認した後、ほかの操作で A の下書きが変わり、`version` は 2 | メンバーが、A と `version` 1 を指定して完全削除する | `ConflictError`（`STALE_VERSION`）を返す。A は残る |
| TC-content-purgeContent-17 | メンバーは、編集権限を持ち、削除権限を持たない | A を完全削除する | `ForbiddenError`（`PERMISSION_MISSING`）を返す。不足する権限（`content:delete`）を添える。A は残る |
| TC-content-purgeContent-18 | トークンは `content_manage` の許可と削除権限を持ち、範囲に A のモデルを含む。A は未公開 | トークンで、A と読み取ったときの `version` を指定して完全削除する | A は無くなる |
| TC-content-purgeContent-20 | 対象でない E の履歴の版 R1 が、必須の参照で A を参照する。E は公開中で、下書きを持つ | 削除権限を持つメンバーが、A と、対象に加える版 R1 を指定して完全削除する | A と R1 は、同じ確定で無くなる。削除した版として R1 を返す。E の公開内容、下書き、ほかの版は変わらない |
| TC-content-purgeContent-21 | 対象でない E の履歴の版 R1 と R2 が、必須の参照で A を参照する | 削除権限を持つメンバーが、A と、対象に加える版 R1 だけを指定して完全削除する | `BusinessRuleError`（`PurgeRejected`）を返す。参照元 E、内容の種類（履歴の版）、R2 を添える。A、R1、R2 は残る |
| TC-content-purgeContent-22 | 対象でない E の履歴の版 R1 が、必須の参照で A を参照する。メンバーが確認した後に、R1 は履歴の版の整理で無くなった | 削除権限を持つメンバーが、A と、対象に加える版 R1 を指定して完全削除する | `NotFoundError` を返す。R1 の ID を添える。A は残る |
| TC-content-purgeContent-23 | C はごみ箱にあり、C のモデルは非推奨 | 削除権限を持つメンバーが、C を完全削除する | C は無くなる |
| TC-content-purgeContent-24 | B は公開中。B の完全削除の事前の確認の後、確定までに、B を必須の参照で参照する R2 の公開が確定した。メンバーは、削除権限と公開権限を持つ | メンバーが、B を完全削除する | `ConflictError`（`PUBLISHED_REFERENCE_REQUIRED`）を返す。B は公開中のまま残る |
| TC-content-purgeContent-25 | 対象でない E の履歴の版 R1 が、必須の参照で A を参照する。完全削除が R1 を読んだ後、確定までに、履歴の版の整理が R1 を削除した | 削除権限を持つメンバーが、A と、対象に加える版 R1 を指定して完全削除する | `ConflictError`（`REVISION_MISSING`）を返す。R1 の ID を示す。A は残る |
| TC-content-purgeContent-26 | トークンは `content_manage` の許可と削除権限を持ち、範囲は A のモデルを含み、E のモデルを含まない。対象でない E の履歴の版 R1 が、必須の参照で A を参照する | トークンで、A と、対象に加える版 R1 を指定して完全削除する | `ForbiddenError`（`OUT_OF_SCOPE`）を返す。A と R1 は残る |
| TC-content-purgeContent-27 | A は未公開で、履歴の版 R1 を持つ | 削除権限を持つメンバーが、A と、対象に加える版 R1 を指定して完全削除する | エラーにしない。A は、R1 を含む履歴の版とともに無くなる。削除した版として R1 を返す |
