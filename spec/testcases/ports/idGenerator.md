# IdGenerator

`IdGenerator` の契約（[../../domains/common.md](../../domains/common.md)）を、すべてのアダプターが通すケースで定める。`generate` と `parse` が返す値の型は `GeneratedId`。

## generate

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-idGenerator-01 | なし | `generate` を呼び、値を `AccountId.create` に渡す | 空でない文字列を返す。`AccountId.create` が `BusinessRuleError` にならず、`AccountId` を返す |
| TC-ports-idGenerator-02 | なし | `generate` を 1,000 回呼ぶ | 返った 1,000 個の値が互いに違う |

## parse

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-idGenerator-04 | `generate` の値 v を得ている | `parse(v)` と `parse(parse(v))` を呼ぶ | `ValidationError` にならず、どちらも v を返す |
| TC-ports-idGenerator-06 | なし | バージョン 4 の `"f47ac10b-58cc-4372-a567-0e02b2c3d479"`、バージョン 1 の `"6ba7b810-9dad-11d1-80b4-00c04fd430c8"`、バージョン 5 の `"886313e1-3b8a-5372-9b90-0c9aee199e5d"`、バージョン 7 の `"017f22e2-79b0-7cc3-98c4-dc0c0c07398f"` で、それぞれ `parse` を呼ぶ | どれも渡した値をそのまま返す |
| TC-ports-idGenerator-10 | なし | `""`、`"not-a-uuid"`、末尾の 1 文字を除いた `"f47ac10b-58cc-4372-a567-0e02b2c3d47"`、16 進でない文字を含む `"g47ac10b-58cc-4372-a567-0e02b2c3d479"`、前後に空白を加えた `" f47ac10b-58cc-4372-a567-0e02b2c3d479 "`、ハイフンの無い `"f47ac10b58cc4372a5670e02b2c3d479"`、波括弧で囲んだ `"{f47ac10b-58cc-4372-a567-0e02b2c3d479}"` で、それぞれ `parse` を呼ぶ | どれも `ValidationError`（理由 `InvalidId`）で失敗する |
| TC-ports-idGenerator-15 | なし | 大文字だけの `"F47AC10B-58CC-4372-A567-0E02B2C3D479"` と、大文字を 1 文字だけ含む `"F47ac10b-58cc-4372-a567-0e02b2c3d479"` で、それぞれ `parse` を呼ぶ | どちらも `ValidationError`（理由 `InvalidId`）で失敗する |
| TC-ports-idGenerator-18 | なし | `generate` を 100 回呼ぶ | 返った値がどれも大文字を含まず、`parse` がそのまま返す |
