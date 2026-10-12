# ごみ箱のコンテンツを復元する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-restoreTrashedContent-01 | C はごみ箱にあり、下書きと履歴の版を持つ。`version` は 3 | 削除権限を持つメンバーが、C を指定して復元する | C は通常の管理対象にあり、未公開。コンテンツID、下書き、履歴の版は変わらない。`version` 4、公開状態（未公開）、下書きの入力エラーを返す。新しい履歴の版は作られない。メディアの使用箇所は変わらない。公開の変更の記録を追記せず、`content.publication_changed` を記録しない。C は通常の一覧に現れ、公開取得には現れない |
| TC-content-restoreTrashedContent-02 | C はごみ箱にある。C の下書きは入力エラーを持ち、未公開の T を参照し、一意な値がほかのコンテンツの公開内容と重複する | 削除権限を持つメンバーが、C を復元する | エラーにしない。復元し、下書きの入力エラーを返す |
| TC-content-restoreTrashedContent-03 | 単一モデル S のコンテンツ S1 はごみ箱にある。S の通常の管理対象に、コンテンツは無い | 削除権限を持つメンバーが、S1 を復元する | S1 は通常の管理対象にあり、未公開 |
| TC-content-restoreTrashedContent-04 | 単一モデル S のコンテンツ S1 はごみ箱にある。S の通常の管理対象に、新しいコンテンツ S2 がある | 削除権限を持つメンバーが、S1 を復元する | `BusinessRuleError`（`SingleContentExists`）を返す。S1 はごみ箱のまま。S2 は変わらない |
| TC-content-restoreTrashedContent-05 | メンバーがごみ箱の一覧で C を見た後、ほかのメンバーが C を復元した。C は通常の管理対象にある | 削除権限を持つメンバーが、C を復元する | `BusinessRuleError`（`ContentNotInTrash`）を返す。C は変わらない |
| TC-content-restoreTrashedContent-06 | C は、メンバーが確認した後に、ごみ箱の完全削除またはほかの操作で完全削除されている | 削除権限を持つメンバーが、C を復元する | `NotFoundError` を返す |
| TC-content-restoreTrashedContent-07 | メンバーがごみ箱の C の内容を開いた後、値の書き換えで C の下書きが変わり、`version` は 3 から 4 に進んだ | 削除権限を持つメンバーが、C を復元する | C は通常の管理対象にあり、未公開。下書きは、書き換えた後の値を持つ。`version` 5 を返す |
| TC-content-restoreTrashedContent-09 | メンバーは、編集権限を持ち、削除権限を持たない | C を復元する | `ForbiddenError`（`PERMISSION_MISSING`）を返す。不足する権限（`content:delete`）を添える。C はごみ箱のまま |
| TC-content-restoreTrashedContent-10 | トークンは `content_manage` の許可と削除権限を持ち、範囲に C のモデルを含む | トークンで、C を指定して復元する | C は通常の管理対象にあり、未公開 |
| TC-content-restoreTrashedContent-12 | C はごみ箱にあり、C のモデルは非推奨 | 削除権限を持つメンバーが、C を復元する | `BusinessRuleError`（`ModelDeprecated`）を返す。C はごみ箱のまま |
| TC-content-restoreTrashedContent-13 | 単一モデル S のコンテンツ S1 はごみ箱にある。S1 の復元が S の通常の管理対象を読んだ後、確定までに、S の新しいコンテンツ S2 の作成が確定した | 削除権限を持つメンバーが、S1 を復元する | `ConflictError`（`SINGLE_EXISTS`）を返す。S1 はごみ箱のまま |
| TC-content-restoreTrashedContent-14 | C はごみ箱にある。C の復元が C を読んだ後、確定までに、ほかのメンバーによる C の復元が確定した | 削除権限を持つメンバーが、C を復元する | `ConflictError`（`STALE_VERSION`）を返す。C は、先の復元のとおり、通常の管理対象にある |
