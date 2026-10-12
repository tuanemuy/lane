# MediaDeliveryLinks

契約は [../../domains/media.md](../../domains/media.md) の「ポート」が定める。`ref` は `MediaObjectRef`（`MediaObjectRef.of` で作る）。ワークスペース W のメディア M の元のファイルを指す `ref` を O(W, M)、キー k・世代 g の派生形式を指す `ref` を D(W, M, k, g) と書く。g、g1、g2 は世代（`DerivativeGeneration`）で、g1 と g2 は互いに違う。期限の日時を e1 < e2 とする。前提条件が空のケースは、状態を要しない。`resolve` に渡す値は、`publicUrl` と `previewUrl` が返した `MediaUrl`、または `parse` が返した `MediaUrl`。許可を偽造できないことと、許可の計算の失敗（`SystemError`）は、契約のとおり、このスイートの対象にしない。`UnitOfWorkProvider.run` は、W の `WriteScope`（`{ kind: "workspace", workspaceId: W, condition: { kind: "none" } }`）で呼ぶ。

## publicUrl

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaDeliveryLinks-01 | | `publicUrl(O(W, M))` を呼ぶ | 空でない `MediaUrl` を返す |
| TC-ports-mediaDeliveryLinks-02 |  | `publicUrl(O(W, M))` を、`previewUrl` と `resolve` の呼び出しを挟んで 2 回呼ぶ | 同じ値を返す |
| TC-ports-mediaDeliveryLinks-03 | 同じ設定で作った、ポートの別のインスタンスがある | 二つのインスタンスで `publicUrl(O(W, M))` を呼ぶ | 同じ値を返す |
| TC-ports-mediaDeliveryLinks-05 |  | `publicUrl` を、O(W, M) と D(W, M, k, g1)、D(W, M, k, g1) と D(W, M, k, g2)、O(W, M1) と O(W, M2)、O(W1, M) と O(W2, M) の組で呼ぶ | どの組も、異なる値を返す |
| TC-ports-mediaDeliveryLinks-08 | `MediaRepository` と `MediaStorage` に、M の記録も実体も無い | `publicUrl(O(W, M))` を呼ぶ | `MediaUrl` を返す。記録と実体の有無を確かめない |

## previewUrl

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaDeliveryLinks-09 | | `previewUrl(O(W, M), e1)` を呼ぶ | 空でない `MediaUrl` を返す |
| TC-ports-mediaDeliveryLinks-10 |  | `previewUrl(O(W, M1), e1)` を呼び、`publicUrl(O(W, M1))`、`previewUrl(O(W, M1), e2)`、`previewUrl(O(W, M2), e1)` の結果と比べる | どれとも異なる値を返す |
| TC-ports-mediaDeliveryLinks-13 | 現在時刻より前の日時 e0 がある | `previewUrl(O(W, M), e0)` を呼ぶ | `MediaUrl` を返す。期限を判定しない |
| TC-ports-mediaDeliveryLinks-14 | `MediaRepository` と `MediaStorage` に、M の記録も実体も無い | `previewUrl(O(W, M), e1)` を呼ぶ | `MediaUrl` を返す |

## parse

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaDeliveryLinks-15 | `publicUrl(O(W, M))` の結果 u1 と、`previewUrl(D(W, M, k, g), e1)` の結果 u2 がある | u1、u2 と同じ文字列を、それぞれ 2 回 `parse` に渡す | それぞれ、u1、u2 と等しい `MediaUrl` を返す |
| TC-ports-mediaDeliveryLinks-17 | `previewUrl(O(W, M), e0)` の結果 u がある。e0 は現在時刻より前 | u と同じ文字列を `parse` に渡す | `MediaUrl` を返す。期限を確かめない |
| TC-ports-mediaDeliveryLinks-18 | `MediaRepository` と `MediaStorage` に、M の記録も実体も無い。`publicUrl(O(W, M))` の結果 u がある | u と同じ文字列を `parse` に渡す | `MediaUrl` を返す。記録と実体の有無を確かめない |
| TC-ports-mediaDeliveryLinks-19 | このポートが発行していない文字列 `not-a-media-link`、`publicUrl(O(W, M))` の結果から、スキームとホストを除いた文字列、ほかのサイトの URL `https://example.invalid/image.jpg`、空の文字列がある | それぞれ `parse` に渡す | どれも `null` を返す。例外を投げない |

## resolve

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaDeliveryLinks-24 | `publicUrl(O(W, M))` の結果 u1 と、`publicUrl(D(W, M, k, g))` の結果 u2 がある | それぞれ `resolve` を呼ぶ | `ref` が、それぞれ O(W, M)、D(W, M, k, g)（`workspaceId`、`mediaId`、`variant.kind`、`key`、`generation`）と等しく、`previewGrant` が `null` の値を返す |
| TC-ports-mediaDeliveryLinks-26 | `previewUrl(O(W, M), e1)` の結果 u1 と、`previewUrl(D(W, M, k, g), e1)` の結果 u2 がある | u1 と u2 を、それぞれ `resolve` に渡す。u1 と同じ文字列を `parse` に渡し、その結果も `resolve` に渡す | `ref` が、それぞれ O(W, M)、D(W, M, k, g)、O(W, M) と等しく、`previewGrant.expiresAt` が e1 と等しい値を返す |
| TC-ports-mediaDeliveryLinks-29 | `publicUrl(ref)` の結果と `previewUrl(ref, e1)` の結果がある | それぞれ `resolve` する | 同じ `ref` を返す |
| TC-ports-mediaDeliveryLinks-30 | `publicUrl(O(W1, M))`、`publicUrl(O(W2, M))`、`previewUrl(O(W1, M), e1)`、`previewUrl(O(W2, M), e1)` の結果がある | それぞれ `resolve` する | `ref.workspaceId` は、順に W1、W2、W1、W2。配信先が含むワークスペースの ID を返す |
| TC-ports-mediaDeliveryLinks-31 | `previewUrl(O(W, M), e1)` の結果と `previewUrl(O(W, M), e2)` の結果がある | それぞれ `resolve` する | `previewGrant.expiresAt` は、それぞれ e1、e2 |
| TC-ports-mediaDeliveryLinks-32 | 現在時刻より前の日時 e0 で `previewUrl(O(W, M), e0)` の結果 u がある | `resolve(u)` を呼ぶ | `ref` と、`expiresAt` が e0 の `previewGrant` を返す。期限切れを理由に許可を落とさない |
| TC-ports-mediaDeliveryLinks-33 | ミリ秒の端数を持つ日時 e で `previewUrl(O(W, M), e)` の結果 u がある | `resolve(u)` を呼ぶ | `previewGrant.expiresAt` が e と等しい |
| TC-ports-mediaDeliveryLinks-34 | キーが 64 文字の `DerivativeKey` の `ref` で `publicUrl` と `previewUrl` の結果がある | それぞれ `resolve` する | どちらも、同じ `ref` を返す |
| TC-ports-mediaDeliveryLinks-35 | 同じ設定で作った、ポートの別のインスタンスが `previewUrl(O(W, M), e1)` で発行した結果 u がある | `resolve(u)` を呼ぶ | `ref` が O(W, M) と等しく、`previewGrant.expiresAt` が e1 と等しい値を返す。発行を保存した状態に頼らない |
| TC-ports-mediaDeliveryLinks-36 | `MediaRepository` と `MediaStorage` に、M の記録も実体も無い。`publicUrl(O(W, M))` の結果 u がある | `resolve(u)` を呼ぶ | `ref` を返す。記録と実体の有無を確かめない |
| TC-ports-mediaDeliveryLinks-37 | `previewUrl(O(W, M), e1)` の結果 u がある | u の各位置の 1 文字を別の文字に置き換えた文字列、u の末尾に 1 文字を加えた文字列、u の末尾の 1 文字を除いた文字列を、それぞれ `parse` に渡し、`MediaUrl` が返れば `resolve` に渡す | どれも、`parse` が `null` を返すか、`resolve` が `previewGrant` を `null` で返す |
| TC-ports-mediaDeliveryLinks-40 | `previewUrl(O(W, M1), e1)` の結果 u1 と、`previewUrl(O(W, M2), e2)` の結果 u2 がある | u1 と u2 をそれぞれ `resolve` する | u1 は O(W, M1) と e1、u2 は O(W, M2) と e2 を返す。発行した値そのものは、互いの許可に影響しない |
| TC-ports-mediaDeliveryLinks-41 | 同じ `ref` と `expiresAt` で `previewUrl` を 2 回呼んだ結果 u1 と u2 がある | それぞれ `resolve` する | どちらも、同じ `ref` と、`expiresAt` が等しい `previewGrant` を返す |

## 状態と UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mediaDeliveryLinks-42 | | `UnitOfWorkProvider.run` の外で `publicUrl`、`previewUrl`、`parse`、`resolve` を呼ぶ | どれも成功する |
| TC-ports-mediaDeliveryLinks-43 | | `UnitOfWorkProvider.run` の `fn` の中で `previewUrl` を呼んだ後、`fn` が例外を投げる | 得た配信先を `resolve` すると、`ref` と `previewGrant` を返す |
| TC-ports-mediaDeliveryLinks-44 | | `previewUrl` と `resolve` を、異なる `ref` と期限で、並行して 10 件ずつ呼ぶ | すべて成功し、それぞれの `resolve` は、発行に渡した `ref` と期限を返す |
