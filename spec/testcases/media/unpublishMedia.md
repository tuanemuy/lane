# メディアを非公開化する（`unpublishMedia`）

「非公開化する」は、断りが無い限り、操作の時点のメディアの `version` を、確認したときの `version` として添える。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-media-unpublishMedia-01 | `media:publish` を持つメンバー。使用箇所の無い公開の画像があり、`ready` の派生形式を持つ。確定の前に、資格を示さない配信の要求へ、元の画像と派生形式の内容を返している | 非公開化する | `private` のメディアを返す。以後、資格を示さない配信の要求は、元の画像と派生形式のどちらも `NotFoundError` になる。消去待ちの記録はできない。実体はメディアの保管先に残る。イベントを記録しない |
| TC-media-unpublishMedia-02 | `media:publish` を持つメンバー。公開のメディアに、`public` が偽の使用箇所だけが記録されている | 非公開化する | `private` のメディアを返す。使用箇所の記録は変わらない |
| TC-media-unpublishMedia-03 | `media:publish` を持つメンバー。公開使用の無い公開のメディアがあり、ワークスペースがスキーマの反映の適用中 | 非公開化する | `private` のメディアを返す |
| TC-media-unpublishMedia-04 | `media:publish` を持つメンバー。公開のメディアに、`MediaUsageRepository.replaceMany` で `public` が真の使用箇所が記録されている。または、公開使用の無い公開のメディアを読んでから非公開化の確定までに、そのメディアの公開使用の記録が確定する | 非公開化する | `ConflictError`（理由 `MEDIA_IN_PUBLIC_USE`）。公開範囲は `public` のまま。資格を示さない配信の要求は実体を返し続ける。消去待ちの記録はできない |
| TC-media-unpublishMedia-06 | `media:publish` を持つメンバー。公開使用の無い公開のメディアがある | 非公開化の確定と、そのメディアの公開使用の記録の確定を並行して行う | 一方だけが成功する。非公開化が負けた場合は `ConflictError`（理由 `MEDIA_IN_PUBLIC_USE`）、記録が負けた場合は `ConflictError`（理由 `MEDIA_NOT_PUBLIC`）。公開使用のある非公開のメディアは生じない |
| TC-media-unpublishMedia-07 | `content:edit` を持ち、`media:publish` を持たないメンバー。公開使用の無い公開のメディアがある | 非公開化する | `ForbiddenError`（理由 `PERMISSION_MISSING`）。必要な権限 `media:publish` を示す。公開範囲は変わらない |
| TC-media-unpublishMedia-08 | `media:publish` を持つメンバー。メディアが削除されている | 非公開化する | `NotFoundError` |
| TC-media-unpublishMedia-09 | `media:publish` を持つメンバー。公開使用の無い公開のメディアの非公開化を確認した後に、別のメンバーの非公開化と、そのメディアを新たに公開する操作が確定している（メディアは `public`） | 確認したときの `version` を添えて非公開化する | `ConflictError`（理由 `STALE_VERSION`）。公開範囲は変わらない |
| TC-media-unpublishMedia-10 | `media:publish` を持つメンバー。メディアを読んでから確定までに、メディアが削除される | 非公開化する | `ConflictError`（理由 `STALE_VERSION`）。メディアは無いまま |
