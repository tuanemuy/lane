# UsageLimiter

`UsageLimiter` の契約（[../../domains/common.md](../../domains/common.md)）を、すべてのアダプターが通すケースで定める。

- 制限 L は `{ max: 3, windowMs: 60_000 }` を指す。断りが無ければ、`check` と `record` に L を渡す
- 時刻は、テストが `now` に渡す値で決める。t0 を基準の時刻とし、「t0 + 10 秒」のように表す
- キー k は、ケースごとに新しい、記録の無いキーを指す

## check

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-usageLimiter-01 | k に記録が無い | `check(k, L, t0)` を呼ぶ | `available` を返す |
| TC-ports-usageLimiter-02 | t0 に `record` を 1 回、または 2 回呼んでいる | `check(k, L, t0 + 1 秒)` を呼ぶ | どちらも `available` を返す |
| TC-ports-usageLimiter-04 | t0 に `record` を 3 回呼んでいる | `check(k, L, t0 + 1 秒)` を呼ぶ | `limited` を返し、`retryAt` が t0 + 60 秒である |
| TC-ports-usageLimiter-07 | t0 に `record` を 3 回呼んでいる | `check(k, L, t0 + 60 秒)` を呼ぶ | `available` を返す |
| TC-ports-usageLimiter-08 | t0 に `record` を 2 回呼んでいる | `check(k, L, t0 + 1 秒)` を 10 回呼んだ後、もう一度 `check` を呼ぶ | どれも `available` を返す |
| TC-ports-usageLimiter-10 | k に記録が無い | `check(k, L, t0)` を呼んだ後、t0 + 30 秒に `record` を 3 回呼び、`check(k, L, t0 + 31 秒)` を呼ぶ | `limited` を返し、`retryAt` が t0 + 90 秒である |

## record

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-usageLimiter-13 | t0 に `record` を 2 回呼んでいる | `record(k, L, t0 + 59 秒)` を呼び、`check(k, L, t0 + 59.5 秒)` を呼ぶ | `limited` を返し、`retryAt` が t0 + 60 秒である |
| TC-ports-usageLimiter-14 | t0 に `record` を 3 回呼んでいる | `record(k, L, t0 + 30 秒)` を呼び、`check(k, L, t0 + 31 秒)` を呼ぶ | `record` が成功する。`check` が、`retryAt` が t0 + 60 秒の `limited` を返す |
| TC-ports-usageLimiter-15 | t0 に `record` を 3 回、t0 + 30 秒に 1 回呼んでいる | `check(k, L, t0 + 61 秒)` を呼ぶ。別のキーで、t0 に `record` を 1 回、t0 + 120 秒に 2 回呼び、`check(t0 + 121 秒)` を呼ぶ | どちらも `available` を返す |
| TC-ports-usageLimiter-17 | t0 に `record` を 3 回呼んでいる | t0 + 120 秒に `record` を 3 回呼び、`check(k, L, t0 + 121 秒)` を呼ぶ | `limited` を返し、`retryAt` が t0 + 180 秒である |
| TC-ports-usageLimiter-19 | t0 に `record` を 3 回呼んでいる | t0 + 60 秒に `record` を 3 回呼び、`check(k, L, t0 + 61 秒)` を呼ぶ | `limited` を返し、`retryAt` が t0 + 120 秒である |

## 戻った now

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-usageLimiter-20 | t0 + 30 秒に `record` を 1 回呼んでいる。別のキーで、t0 に `record` を 3 回、t0 + 120 秒に 1 回呼んでいる | k で `record(k, L, t0 + 10 秒)` を 2 回呼び、`check(k, L, t0 + 31 秒)` を呼ぶ。別のキーで `record(t0 + 30 秒)` を 2 回呼び、`check(t0 + 121 秒)` を呼ぶ | `record` が成功する。k の `check` が、`retryAt` が t0 + 90 秒の `limited` を返す。別のキーの `check` が、`retryAt` が t0 + 180 秒の `limited` を返す |
| TC-ports-usageLimiter-21 | t0 + 30 秒に `record` を 3 回呼んでいる。別のキーで、t0 + 30 秒に 2 回、t0 + 31 秒に 1 回呼んでいる | k で `check(k, L, t0 + 10 秒)` を、別のキーで `check(t0 + 30.5 秒)` を呼ぶ | どちらも `limited` を返し、`retryAt` が t0 + 90 秒である |

## キーの独立

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-usageLimiter-24 | キー k1 で t0 に `record` を 3 回、k1 を接頭辞に持つキー k2 で t0 + 30 秒に `record` を 2 回呼んでいる | t0 + 31 秒に、k1 と k2 でそれぞれ `check` を呼ぶ | k1 が、`retryAt` が t0 + 60 秒の `limited` を返す。k2 が `available` を返す |

## 並行性と可視性

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-usageLimiter-27 | `{ max: 20, windowMs: 60_000 }` で、k に記録が無い | 同じ制限と同じ `now` t0 で、`record` を 20 個並行して呼び、すべての完了後に `check(k, t0 + 1 秒)` を呼ぶ | 20 個の `record` がすべて成功する。`check` が `limited` を返す |
| TC-ports-usageLimiter-28 | `{ max: 20, windowMs: 60_000 }` で、k に記録が無い | 同じ制限と同じ `now` t0 で、`record` を 19 個並行して呼び、すべての完了後に `check(k, t0 + 1 秒)` を呼ぶ | `check` が `available` を返す |
| TC-ports-usageLimiter-29 | `{ max: 20, windowMs: 60_000 }` で、k に記録が無い | 同じ制限で、`now` に t0 から t0 + 19 ミリ秒までの互いに違う時刻を渡した `record` を、20 個並行して呼び、すべての完了後に `check(k, t0 + 1 秒)` を呼ぶ | 20 個の `record` がすべて成功する。`check` が `limited` を返す |
| TC-ports-usageLimiter-30 | t0 に `record` を 2 回呼んでいる | `record(k, L, t0 + 1 秒)` の完了の直後に `check(k, L, t0 + 1 秒)` を呼ぶ。別のキーで、`UnitOfWorkProvider.run` の `fn` の中で同じに呼ぶ | どちらの `check` も `limited` を返す |
| TC-ports-usageLimiter-31 | t0 に `record` を 2 回呼んでいる | `UnitOfWorkProvider.run` の `fn` の中で `record(k, L, t0 + 1 秒)` を呼んだ後、`fn` が例外を投げる。その後、`check(k, L, t0 + 2 秒)` を呼ぶ | `run` が失敗する。`check` が `limited` を返す |

## エラーケース

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-usageLimiter-33 | なし | 空のキーで `check` と `record` をそれぞれ呼ぶ | どちらも `SystemError`（理由 `INVALID_QUERY`）で失敗する |
| TC-ports-usageLimiter-35 | なし | `{ max: 0, windowMs: 60_000 }`、`{ max: 1.5, windowMs: 60_000 }`、`{ max: 3, windowMs: 0 }`、`{ max: 3, windowMs: 0.5 }` で、`check` と `record` をそれぞれ呼ぶ | どれも `SystemError`（理由 `INVALID_QUERY`）で失敗する |
