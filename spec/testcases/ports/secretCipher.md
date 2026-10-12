# SecretCipher

`SecretCipher` の契約（[../../domains/common.md](../../domains/common.md)）を、すべてのアダプターが通すケースで定める。

「別の鍵の `SecretCipher`」は、保護に使う鍵が違う構成で用意した、同じアダプターのもう一つのインスタンスを指す。

## protect と reveal

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-secretCipher-01 | なし | `"secret-value"`、空の文字列、改行と日本語を含む 10,000 文字の値を、それぞれ `protect` し、結果を `reveal` に二回ずつ渡す | どの `reveal` も、`protect` に渡した値を返す |
| TC-ports-secretCipher-04 | なし | 同じ値で `protect` を二回呼び、それぞれの結果を `reveal` に渡す | 二つの `ProtectedSecret` が同じでも違っても、どちらの `reveal` も元の値を返す |

## 解けない値

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-secretCipher-07 | 別の鍵の `SecretCipher` が `protect(x)` した結果 s を得ている | s を `reveal` に渡す | `SystemError`（理由 `SECRET_UNAVAILABLE`）で失敗する |
| TC-ports-secretCipher-08 | `protect(x)` の結果 s を得ている | `protect` が返していない文字列（`"not-a-protected-secret"`）、空の文字列、s の末尾の 1 文字を除いた値を、それぞれ `reveal` に渡す | どれも `SystemError`（理由 `SECRET_UNAVAILABLE`）で失敗する |

## currentKeyId

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-secretCipher-11 | なし | `currentKeyId` を呼び、`protect` と `reveal` を呼んだ後、もう一度 `currentKeyId` を呼ぶ | 二回とも同じ値を返す |
| TC-ports-secretCipher-13 | なし | 別の鍵の `SecretCipher` と、それぞれ `currentKeyId` を呼ぶ | 二つの値が違う |
| TC-ports-secretCipher-14 | 同じ鍵で構成したもう一つのインスタンスが `protect(x)` した結果 s を得ている | 両方の `currentKeyId` を呼び、s を `reveal` に渡す | 二つの `currentKeyId` が同じ値を返し、`reveal` が x を返す |

## 鍵を使えない構成

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-secretCipher-15 | 保護に使う鍵が無い構成で用意したインスタンス | `protect("x")`、`currentKeyId`、別の鍵の `SecretCipher` が `protect(x)` した結果の `reveal` を、それぞれ呼ぶ | どれも `SystemError`（理由 `SECRET_UNAVAILABLE`）で失敗する |
