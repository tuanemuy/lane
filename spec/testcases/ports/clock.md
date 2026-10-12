# Clock

`Clock` の契約（[../../domains/common.md](../../domains/common.md)）を、すべてのアダプターが通すケースで定める。

## now

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-clock-01 | なし | `now` を呼ぶ | 有効な日時を指す `Date` を返す |
