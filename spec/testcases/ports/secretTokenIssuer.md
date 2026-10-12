# SecretTokenIssuer

`SecretTokenIssuer` の契約（[../../domains/common.md](../../domains/common.md)）を、すべてのアダプターが通すケースで定める。

## issue

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-secretTokenIssuer-01 | なし | `issue` を呼ぶ | 空でない `value` と、空でない `digest` を返す。`value` が、英数字と `-`、`_` だけからなる 43 文字以上の文字列である |
| TC-ports-secretTokenIssuer-02 | なし | `issue` を呼び、返った `value` で `digest` を三回呼ぶ | 三回とも、`issue` の返した `digest` と同じ値を返す |
| TC-ports-secretTokenIssuer-05 | なし | `issue` を 1,000 回呼ぶ | 返った 1,000 個の `value` が互いに違う |

## issueForWorkspace と workspaceOf

W、V は、互いに違うワークスペースの ID。`IdGenerator.generate` の値から作る。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-secretTokenIssuer-06 | なし | `issueForWorkspace(W)` を呼ぶ | 空でない `value` と、空でない `digest` を返す。`value` が、英数字と `-`、`_` だけからなる |
| TC-ports-secretTokenIssuer-07 | なし | `issueForWorkspace(W)` を呼び、返った `value` で `digest` を呼ぶ | `digest(value)` が、`issueForWorkspace` の返した `digest` と同じ値を返す |
| TC-ports-secretTokenIssuer-08 | なし | `issueForWorkspace(W)` と `issueForWorkspace(V)` を呼び、それぞれの `value` で `workspaceOf` を二回ずつ呼ぶ | W の `value` は二回とも W を、V の `value` は二回とも V を返す |
| TC-ports-secretTokenIssuer-10 | なし | `issueForWorkspace(W)` を 1,000 回呼ぶ | 返った 1,000 個の `value` が互いに違う。どの `value` の `workspaceOf` も W を返す |
| TC-ports-secretTokenIssuer-11 | なし | `issueForWorkspace(W)` と `issueForWorkspace(V)` を呼ぶ | 二つの `value` が違う。二つの `digest` が違う |
| TC-ports-secretTokenIssuer-12 | なし | `issue` の `value`、`""`、空白・日本語・改行を含む値、100,000 文字の値で、それぞれ `workspaceOf` を呼ぶ | 失敗せず、どれも `null` を返す |
| TC-ports-secretTokenIssuer-16 | どのワークスペースの区画にも確定していないワークスペースの ID W | `issueForWorkspace(W)` を呼び、返った `value` で `workspaceOf` を呼ぶ | W を返す |

## digest

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-secretTokenIssuer-17 | なし | 同じ値で `digest` を二回呼ぶ | 二回とも同じ照合値を返す |
| TC-ports-secretTokenIssuer-19 | なし | `""`、100,000 文字の値、`issue` が返す文字に含まれない文字（空白、日本語、改行）を含む値で、それぞれ `digest` を呼ぶ | 失敗せず、どれも空でない照合値を返す |
| TC-ports-secretTokenIssuer-22 | なし | 値 x、x の末尾に 1 文字を加えた値、x の英字の大文字と小文字を入れ替えた値、`""`、`" "` で `digest` を呼ぶ。`issue` を 1,000 回呼ぶ | 五つの照合値が互いに違う。返った 1,000 個の `digest` が互いに違う |
