# メディアを一覧・検索する（`listMedia`）

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-media-listMedia-01 | `content:edit` を持たないメンバー。ワークスペースに画像 2 件と文書 1 件がある | 絞り込みなしで一覧する | 3 件と総件数 3 を返す。メディアごとに、管理用の名称、共有する説明、登録時の記録（形式、容量、画像の幅・高さ）、公開範囲、メディアの種類（画像は `image`、文書は `document`）を返す |
| TC-media-listMedia-02 | メンバー。語 X を管理用の名称に含むメディア、検索情報に含むメディア、どちらにも含まないメディアがある | 検索の語 X で一覧する | 名称に含むメディアと検索情報に含むメディアの 2 件と、総件数 2 を返す |
| TC-media-listMedia-03 | メンバー。`failed` の派生形式を持つ画像、`generating` の派生形式を持つ画像、すべて `ready` の画像がある | 生成状況を `generating` と `failed` で絞り込む | `failed` の画像と `generating` の画像の 2 件を返す。派生形式ごとの生成状況を返す |
| TC-media-listMedia-04 | メンバー。`ready` の派生形式と `generating`（変更前の派生形式なし）の派生形式を持つ画像がある | 一覧する | 元の画像と `ready` の派生形式の配信先を返す。`generating` の派生形式は、生成状況を `generating` として返し、配信先を返さない |
| TC-media-listMedia-05 | メンバー。画像 1 件と文書 1 件がある。一覧がメディアを読んだ後、管理情報を読む前に、文書の削除が確定する | 一覧する | 画像だけを返す |
| TC-media-listMedia-06 | メンバー。検索の語に該当するメディアが無い | その語で一覧する | 空の列と総件数 0 を返す。エラーにしない |
| TC-media-listMedia-08 | メンバー | 長さの上限を超える検索の語で一覧する | `ValidationError`（理由 `InvalidMediaFilter`） |
| TC-media-listMedia-09 | メンバー。現行スキーマが派生形式 a を定義している。派生形式の記録の `states` が空の画像（定義の適用の前）と、a が `ready` の画像がある | 生成状況を `generating` で絞り込む | `states` が空の画像だけを返し、a の生成状況を `generating` として返す |
