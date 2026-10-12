# NotFoundError

[../../domains/index.md](../../domains/index.md) の「共通の型とポート」の `NotFoundError` が持つキーの代表ケース。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-common-NotFoundError-02 | ワークスペース W は利用できる。W のコンテンツ A の履歴に版 R1 がある。A の版 R2 と R3 は、履歴の版の整理で既に無い | `content:delete` を持つメンバーが、A の R1、R2、R3 を一度に削除する（Content の `deleteRevisions`） | `NotFoundError`。対象の種類 `revision` と、R2 と R3 の ID を持つ。R1 は残る |
