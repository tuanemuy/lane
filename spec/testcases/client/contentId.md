# 名前からコンテンツID を求める関数（contentId）

`packages/client` の `contentId`。契約は [app-api.md](../../usecases/app-api.md) の「名前からコンテンツID を求める関数」が定める。CMS へ接続しない。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-client-contentId-01 | なし | `contentId("article/hello-world")` を二度呼ぶ | 二度とも同じ値を返す |
| TC-client-contentId-02 | なし | `contentId("article/hello-world")` と `contentId("article/hello-world-2")` を呼ぶ | 違う値を返す |
| TC-client-contentId-03 | なし | `contentId("article/hello-world")` の結果を `IdGenerator.parse` に通す | `GeneratedId` を返す |
| TC-client-contentId-04 | なし | `contentId("article/hello-world")` を呼ぶ | `196e4f5b-3653-5560-baba-087207507c7d` を返す |
