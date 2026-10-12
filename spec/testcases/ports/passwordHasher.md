# PasswordHasher

`PasswordHasher` の契約（[../../domains/identity.md](../../domains/identity.md)）を、すべてのアダプターが通すケースで定める。

パスワードは `PlainPassword.create` で作る。

## hash

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-passwordHasher-01 | なし | 12 文字のパスワードで `hash` を呼ぶ | `PasswordHash.create` が受け付ける、空でない照合値を返す |
| TC-ports-passwordHasher-02 | なし | 128 文字のパスワードで `hash` を呼ぶ | 照合値を返す |
| TC-ports-passwordHasher-03 | なし | 同じパスワードで `hash` を二回呼ぶ | 二つの照合値が違う |

## verify

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-passwordHasher-04 | `hash(p)` の照合値 h を得ている | `verify(p, h)` を呼ぶ | 真を返す |
| TC-ports-passwordHasher-06 | 同じ p で `hash` を二回呼び、照合値 h1、h2 を得ている | `verify(p, h1)` と `verify(p, h2)` を呼ぶ | どちらも真を返す |
| TC-ports-passwordHasher-07 | 128 文字のパスワード p の照合値 h を得ている | `verify(p, h)` を呼ぶ | 真を返す |
| TC-ports-passwordHasher-08 | 日本語と記号を含むパスワード p の照合値 h を得ている | `verify(p, h)` を呼ぶ | 真を返す |
| TC-ports-passwordHasher-09 | `hash(p)` の照合値 h を得ている | p と違うパスワードで `verify` を呼ぶ | 偽を返す |
| TC-ports-passwordHasher-10 | `hash(p)` の照合値 h を得ている | p の末尾に 1 文字を加えた値と、p の末尾の 1 文字を除いた値で、それぞれ `verify` を呼ぶ | どちらも偽を返す |
| TC-ports-passwordHasher-11 | `hash(p)` の照合値 h を得ている | p の英字の大文字と小文字を入れ替えた値で `verify` を呼ぶ | 偽を返す |
| TC-ports-passwordHasher-12 | `hash(p)` の照合値 h を得ている | `verify("", h)` を呼ぶ | 例外を投げず、偽を返す |
| TC-ports-passwordHasher-13 | `hash(p)` の照合値 h を得ている | 100,000 文字の値で `verify` を呼ぶ | 例外を投げず、偽を返す |
| TC-ports-passwordHasher-14 | `hash(p)` の照合値 h を得ている | NUL 文字と改行を含む値で `verify` を呼ぶ | 例外を投げず、偽を返す |
| TC-ports-passwordHasher-15 | なし | `hash` が生成していない形の `PasswordHash`（`PasswordHash.create("not-a-hash")`）と p で `verify` を呼ぶ | 例外を投げず、偽を返す |
| TC-ports-passwordHasher-18 | なし | `verify(p, null)`、`verify("", null)`、100,000 文字の値と `null` の `verify` を、それぞれ呼ぶ | どれも例外を投げず、偽を返す |
