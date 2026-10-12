# BatchTargets

[../../domains/common.md](../../domains/common.md) の「BatchTargets」の代表ケース。保存を伴わない値オブジェクトなので、ユースケースを通さずに確かめる。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-common-BatchTargets-01 | なし | `create(["b", "a", "b", "c", "a"], 3)` を呼ぶ | `["b", "a", "c"]` を返す |
| TC-common-BatchTargets-02 | なし | `createBy` に、ID が `x`、`y`、`x` で `version` がそれぞれ 1、2、3 の列と、上限 2、ID を返す `key` を渡す | `x`（`version` 1）、`y` の列を返す |
| TC-common-BatchTargets-03 | なし | `create([], 3)` を呼ぶ | `ValidationError`（`InvalidTargetCount`）。上限 3 を添える |
| TC-common-BatchTargets-04 | なし | `create(["a", "b", "c", "d", "a"], 3)` を呼ぶ | `ValidationError`（`InvalidTargetCount`）。上限 3 を添える。列を切り詰めない |
| TC-common-BatchTargets-05 | なし | `create(["a", "b", "c", "a"], 3)` を呼ぶ | `["a", "b", "c"]` を返す |
