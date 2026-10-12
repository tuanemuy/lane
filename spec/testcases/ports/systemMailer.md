# SystemMailer

`SystemMailer` の契約（[../../domains/common.md](../../domains/common.md)）を、すべてのアダプターが通すケースで定める。

- 「宣言した設置」は、設置全体のメール送信を宣言した構成のアダプターを指す。「宣言していない設置」は、宣言しない構成のアダプターを指す
- 「受理する状態」「受理しない状態」「到達できない状態」は、応答を指定できる試験用の相手に指定する状態を指す
- メール M は、宛先が `EmailAddress`、件名と本文が空でない `SystemMail` を指す

## isConfigured

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-systemMailer-01 | 宣言した設置 | `isConfigured` を二回呼び、`send(M)` が `failed` を返した後に、もう一度呼ぶ | 三回とも真を返す |
| TC-ports-systemMailer-02 | 宣言していない設置 | `isConfigured` を呼ぶ | 偽を返す |
| TC-ports-systemMailer-04 | 宣言した設置で、メール送信基盤が到達できない状態と、受理する状態 | それぞれ `isConfigured` を呼ぶ | 失敗せず、真を返す。メールが送られない |

## send

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-systemMailer-07 | 宣言した設置で、メール送信基盤が受理する状態 | 改行と日本語を含む本文のメール M で `send(M)` を呼ぶ | `accepted` を返す。メール送信基盤が、宛先・件名・本文が M と一致するメールを一通受け取る。本文はプレーンテキスト。送信者は、アダプターに設定した設置の送信者と一致する |
| TC-ports-systemMailer-10 | 宣言した設置で、メール送信基盤が受理する状態 | 同じ M で `send` を二回呼ぶ | 二回とも `accepted` を返す。メール送信基盤が二通を受け取る |
| TC-ports-systemMailer-11 | 宣言した設置で、メール送信基盤が受理する状態 | 宛先が違う二つのメールの `send` を並行して呼ぶ | どちらも `accepted` を返す。メール送信基盤が、それぞれの宛先に一通ずつ受け取る |
| TC-ports-systemMailer-12 | 宣言していない設置 | `send(M)` と、件名が空のメール、本文が空のメールの `send` を、それぞれ呼ぶ | 例外を投げず、どれも `not_configured` を返す。メールが送られない |
| TC-ports-systemMailer-14 | 宣言していない設置と、メール送信基盤が受理する状態の宣言した設置 | それぞれ `isConfigured` と `send(M)` を呼ぶ | 宣言していない設置は、`isConfigured` が偽を返し、`send` が `not_configured` を返す。宣言した設置は、`isConfigured` が真を返し、`send` が `not_configured` を返さない |
| TC-ports-systemMailer-16 | 宣言した設置で、メール送信基盤が受理しない状態と、到達できない状態 | それぞれ `send(M)` を呼ぶ | 例外を投げず、どちらも `failed` と、空でない `reason` を返す |
| TC-ports-systemMailer-18 | 宣言した設置で、メール送信基盤が受理しない状態 | `send(M)` を呼ぶ | `reason` が、M の本文と、アダプターに設定した認証情報のどちらも部分文字列として含まない |
| TC-ports-systemMailer-19 | 宣言した設置で、メール送信基盤が受理しない状態 | `send(M)` を一回呼び、`failed` を返した後、受理する状態にして `send(M)` をもう一度呼ぶ | 一回目の呼び出しで、メール送信基盤への送信の試行が一回だけ起こる。二回目が `accepted` を返し、メール送信基盤が一通を受け取る |
| TC-ports-systemMailer-21 | 宣言した設置で、メール送信基盤が受理する状態 | `UnitOfWorkProvider.run` の `fn` の中で `send(M)` を呼んだ後、`fn` が例外を投げる | `run` が失敗する。メール送信基盤が一通を受け取っている |

## エラーケース

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-systemMailer-22 | 宣言した設置で、メール送信基盤が受理する状態 | 件名が空のメール、本文が空のメール、件名と本文がどちらも空のメールで、それぞれ `send` を呼ぶ | どれも `SystemError`（理由 `INVALID_QUERY`）で失敗する。メールが送られない |
