# TotpAuthenticator

`TotpAuthenticator` の契約（[../../domains/identity.md](../../domains/identity.md)）を、すべてのアダプターが通すケースで定める。

- 「秘密の値」は、`enroll` が返した `setupKey` を、`enroll` の契約の表記で解いたバイト列を指す
- 「時間枠 n の確認コード」は、秘密の値から `verify` の契約の方式で求めた、Unix 時刻を時間枠の長さで割った商が n の時間枠の確認コードを指す。テストが自分で計算する
- `at` を含む時間枠を n とする

## enroll

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-totpAuthenticator-01 | なし | メールアドレス e で `enroll` を呼ぶ | 空でない `secret`、`setupKey`、`otpauthUri` を返す |
| TC-ports-totpAuthenticator-02 | なし | `enroll` を呼ぶ | `setupKey` が、Base32 の大文字（`A`〜`Z`、`2`〜`7`）だけからなり、パディングの `=` を含まない |
| TC-ports-totpAuthenticator-03 | なし | `enroll` を呼ぶ | 秘密の値が 20 バイト以上である |
| TC-ports-totpAuthenticator-04 | なし | メールアドレス e で `enroll` を呼ぶ | `otpauthUri` が `otpauth://totp/` で始まり、ラベルに e を含む |
| TC-ports-totpAuthenticator-05 | なし | `enroll` を呼ぶ | `otpauthUri` の `secret` パラメータが、`setupKey` と同じ値である |
| TC-ports-totpAuthenticator-06 | なし | 同じメールアドレスで `enroll` を二回呼ぶ | 二回の `secret` と `setupKey` が、それぞれ違う |
| TC-ports-totpAuthenticator-07 | なし | `enroll` を呼び、返った `secret` と、時間枠 n の確認コードで `verify` を呼ぶ | 真を返す |
| TC-ports-totpAuthenticator-08 | 同じメールアドレスで `enroll` を二回呼び、一回目の結果で `verify` が真を返している | 二回目の `enroll` の後、一回目の `secret` と、一回目の時間枠 n の確認コードで `verify` を呼ぶ | 真を返す |

## verify

`enroll` の結果を得ているものとする。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-totpAuthenticator-10 | なし | 時間枠 n − 1 の確認コードで `verify` を呼ぶ | 真を返す |
| TC-ports-totpAuthenticator-11 | なし | 時間枠 n + 1 の確認コードで `verify` を呼ぶ | 真を返す |
| TC-ports-totpAuthenticator-12 | 時間枠 n − 2 の確認コードが、時間枠 n − 1、n、n + 1 の確認コードのどれとも違う | 時間枠 n − 2 の確認コードで `verify` を呼ぶ | 偽を返す |
| TC-ports-totpAuthenticator-13 | 時間枠 n + 2 の確認コードが、時間枠 n − 1、n、n + 1 の確認コードのどれとも違う | 時間枠 n + 2 の確認コードで `verify` を呼ぶ | 偽を返す |
| TC-ports-totpAuthenticator-14 | なし | 時間枠 n − 1、n、n + 1 の確認コードのどれとも違う 6 桁の数字で `verify` を呼ぶ | 偽を返す |
| TC-ports-totpAuthenticator-15 | `at` が時間枠 n の最初の瞬間（Unix 時刻が 30 秒の倍数） | 時間枠 n の確認コードと、時間枠 n − 1 の確認コードで、それぞれ `verify` を呼ぶ | どちらも真を返す |
| TC-ports-totpAuthenticator-16 | `at` が時間枠 n の最後のミリ秒 | 時間枠 n の確認コードと、時間枠 n + 1 の確認コードで、それぞれ `verify` を呼ぶ | どちらも真を返す |
| TC-ports-totpAuthenticator-17 | なし | 時間枠 n の確認コードで、同じ `at` の `verify` を二回呼ぶ | 二回とも真を返す |
| TC-ports-totpAuthenticator-18 | もう一度 `enroll` を呼び、別の秘密 s2 を得ている。一つ目の秘密の時間枠 n の確認コードが、s2 の時間枠 n − 1、n、n + 1 の確認コードのどれとも違う | s2 と、一つ目の秘密の時間枠 n の確認コードで `verify` を呼ぶ | 偽を返す |
| TC-ports-totpAuthenticator-19 | 先頭が 0 の確認コードになる `at` を選んでいる | その確認コードを、先頭の 0 を含む 6 桁で `verify` に渡す | 真を返す |
