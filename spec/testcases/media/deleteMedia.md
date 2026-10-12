# メディアを削除する（`deleteMedia`）

「削除する」は、断りが無い限り、操作の時点のメディアの `version` を、確認したときの `version` として添える。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-media-deleteMedia-01 | `content:edit` を持つメンバー。使用箇所の無い非公開の画像があり、`ready` の派生形式を持つ | 削除する | 成功する。メディアの取得は `NotFoundError` になり、一覧と検索に現れない。派生形式の記録と管理情報も無くなる。元のファイルとすべての派生形式の実体がメディアの保管先から無くなり、消去待ちの記録は残らない |
| TC-media-deleteMedia-02 | `media:publish` を持つメンバー。使用箇所の無い公開のメディアがある。確定の前に、資格を示さない配信の要求へ内容を返している | 削除する | 成功する。以後の配信の要求は、資格を示す要求を含めて、すべて `NotFoundError` になる。実体がメディアの保管先から無くなり、消去待ちの記録は残らない |
| TC-media-deleteMedia-03 | `content:edit` を持ち、`media:publish` を持たないメンバー。使用箇所の無い公開のメディアがある | 削除する | `ForbiddenError`（理由 `PERMISSION_MISSING`）。必要な権限 `media:publish` を示す。メディアと実体は残る |
| TC-media-deleteMedia-04 | `media:publish` を持ち、`content:edit` を持たないメンバー。使用箇所の無い非公開のメディアがある | 削除する | `ForbiddenError`（理由 `PERMISSION_MISSING`）。必要な権限 `content:edit` を示す。メディアと実体は残る |
| TC-media-deleteMedia-06 | 削除に必要な権限を持つメンバー。メディアに、`MediaUsageRepository.replaceMany` で使用箇所が一つ記録されている（`public` が真の使用箇所と、偽の使用箇所のそれぞれで確かめる）。または、使用箇所の無いメディアを読んでから確定までに、そのメディアの使用箇所の記録が確定する | 削除する | `ConflictError`（理由 `MEDIA_IN_USE`）。メディアと実体は残る。使用箇所の記録は変わらない |
| TC-media-deleteMedia-07 | `content:edit` を持つメンバー。非公開のメディアの使用箇所が、`MediaUsageRepository.replaceMany`（空の `uses`）ですべて取り除かれている | 削除する | 成功する。メディアの取得は `NotFoundError` になる |
| TC-media-deleteMedia-09 | `content:edit` を持つメンバー | 存在しないメディアの ID を指定して削除する | `NotFoundError` |
| TC-media-deleteMedia-11 | `media:publish` を持つメンバー。使用箇所の無い非公開のメディアの削除を確認した後に、そのメディアが公開される | 確認したときの `version` を添えて削除する | `ConflictError`（理由 `STALE_VERSION`）。メディアは `public` のまま残り、実体も残る |
| TC-media-deleteMedia-12 | `content:edit` を持つメンバー。使用箇所の無い非公開のメディアがある。ワークスペースがスキーマの反映の適用中 | 削除する | 成功する |
| TC-media-deleteMedia-13 | `content:edit` を持つメンバー。使用箇所の無い非公開のメディアがあり、確定の後の実体の消去が失敗する | 削除する | 成功する。メディアの取得は `NotFoundError` になる。消去待ちの記録が残る。その実体の配信の要求は `NotFoundError` になる。「消去待ちの実体を消去する」が、その実体を消去する |
