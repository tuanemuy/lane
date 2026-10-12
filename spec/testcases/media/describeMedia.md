# 管理用の名称・検索情報・共有する説明を編集する（`describeMedia`）

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-media-describeMedia-01 | `content:edit` を持つメンバー。非公開のメディアがある | 管理用の名称、検索情報、代替テキスト、キャプションを指定して保存する | 四つを置き換えた保存後の管理情報を返す。登録時の記録と公開範囲は変わらない。変更後の名称と検索情報で検索に該当する |
| TC-media-describeMedia-02 | `content:edit` を持ち、`media:publish` を持たないメンバー。公開のメディアに、公開使用が記録されている | 名称と共有する説明を変えて保存する | 保存後の管理情報を返す。公開範囲は `public` のまま。使用箇所の記録は変わらない |
| TC-media-describeMedia-03 | `content:edit` を持つメンバー。非公開のメディアがある。管理情報を読んだ後、確定の前に、そのメディアの公開（`publishMany`）が確定する | 名称を変えて保存する | 保存後の管理情報を返す。メディアは `public` のまま |
| TC-media-describeMedia-06 | `content:edit` を持つメンバー | 管理用の名称を空にして保存する | `ValidationError`（理由 `InvalidMediaName`）。メディアは変わらない |
| TC-media-describeMedia-07 | `content:edit` を持つメンバー。編集の途中でメディアが削除されている | 保存する | `NotFoundError` |
| TC-media-describeMedia-08 | `content:edit` を持つメンバー。管理情報を読んでから確定までに、メディアが削除される | 保存する | `ConflictError`（理由 `STALE_VERSION`）。メディアは無いまま |
| TC-media-describeMedia-09 | `content:edit` を持たないメンバー | 保存する | `ForbiddenError`（理由 `PERMISSION_MISSING`）。不足する権限 `content:edit` を示す。メディアは変わらない |
