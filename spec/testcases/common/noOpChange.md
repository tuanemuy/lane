# 変更の無い変更

[../../domains/index.md](../../domains/index.md) の「変更の無い変更」の代表ケース。どの変更が当たるかは、各集約の振る舞いのテストが持つ。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-common-noOpChange-01 | フォーム F の `active` の設定。有効な通知先 A がある | `notification:manage` を持つメンバーが、A を有効にする（Form の `setRecipientEnabled`） | 成功し、A が有効の通知先を返す。F の設定の `version` は進まない。操作履歴は残らない |
