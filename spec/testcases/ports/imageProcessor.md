# ImageProcessor

契約は [media.md](../../domains/media.md) の「ImageProcessor」が定める。[common.md](../../domains/common.md) が定める試験用の相手に対して実行する。出力の画像形式を指定するケースは、その形式の変換に対応するアダプターでだけ実行する。「`avif` に対応しないアダプター」は、対応する出力の画像形式として、変換できる形式から `avif` を除いた集合を渡して作ったアダプター。入力の画像は、テストが用意するバイト列で与える。出力の検証は、`measure` と `MediaFormat.detect` で行う。`outputSize` は、断りが無い限り `DerivativeGeometry.outputSize` で求めた値を渡す。前提条件の「向き」は、画像が持つ向きの情報を指す。契約がアダプターごとの検証の対象にする項目は、このスイートの対象にしない。

## 対応する出力の画像形式

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-imageProcessor-01 | アダプターが対応する出力の画像形式ごとに、その形式を `definition.output.format` にした定義と、読み取れる JPEG がある | それぞれ `transform` を呼ぶ | どれも `ok: true` を返す。`MediaFormat.detect` は出力をその形式と判定し、`measure` は `outputSize` を返す |
| TC-ports-imageProcessor-02 | 試験用の相手と、読み取れる JPEG、アダプターが対応する出力の画像形式の定義がある | 同じ入力で `transform` を、`measure` の呼び出しを挟んで 2 回呼ぶ | どちらも `ok: true` を返す。対応する出力の画像形式は、呼び出しによらず変わらない |

## measure

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-imageProcessor-03 | 幅 800・高さ 600 の JPEG、PNG、WebP、AVIF がある | それぞれ、その形式を `format` にして `measure` を呼ぶ | どれも `{ readable: true, width: 800, height: 600 }` を返す |
| TC-ports-imageProcessor-07 | 幅 1・高さ 1 の PNG がある | `measure(bytes, "png")` を呼ぶ | `{ readable: true, width: 1, height: 1 }` を返す |
| TC-ports-imageProcessor-08 | 保存した画素が幅 800・高さ 600 で、向きが 90 度、180 度、270 度の回転を指す JPEG がある | それぞれ `measure(bytes, "jpeg")` を呼ぶ | 90 度と 270 度は `{ readable: true, width: 600, height: 800 }`、180 度は `{ readable: true, width: 800, height: 600 }` を返す |
| TC-ports-imageProcessor-11 | 向きの情報を持たない JPEG がある | `measure(bytes, "jpeg")` を呼ぶ | 保存した画素の幅と高さを返す |
| TC-ports-imageProcessor-12 | 長さ 0 のバイト列、JPEG の先頭 3 バイト（`FF D8 FF`）だけのバイト列、PNG の先頭 8 バイトの後に無作為なバイト列を続けたもの、画像でない文字列のバイト列、PDF のバイト列がある | それぞれ、四つの `format` で `measure` を呼ぶ | どれも `{ readable: false }` を返す。例外を投げない |
| TC-ports-imageProcessor-16 | 読み取れる JPEG がある | 同じ `bytes` で `measure` を 2 回呼ぶ | 同じ結果を返す |
| TC-ports-imageProcessor-17 | 読み取れる JPEG、PNG、WebP、AVIF がある | それぞれ、内容と違う形式を `format` にして `measure` を呼ぶ（JPEG は `png`、`webp`、`avif`、ほかは `jpeg`） | どれも `{ readable: false }` を返す。例外を投げない |

## transform の大きさ

元の画像は、幅 800・高さ 600 の JPEG とする。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-imageProcessor-21 | `size` が `{ fit: "width", width: 400 }`、`{ fit: "height", height: 150 }`、`{ fit: "contain", width: 300, height: 300 }`、`{ fit: "cover", width: 300, height: 300 }`、`{ fit: "cover", width: 200, height: 400 }` の定義がある | それぞれ、`outputSize` を `{ width: 400, height: 300 }`、`{ width: 200, height: 150 }`、`{ width: 300, height: 225 }`、`{ width: 300, height: 300 }`、`{ width: 200, height: 400 }` にして `transform` を呼ぶ | どれも `ok: true` を返す。出力の `measure` は、それぞれ `outputSize` の幅と高さを返す |
| TC-ports-imageProcessor-26 | `size` が `{ fit: "width", width: 1600 }`、`{ fit: "cover", width: 1000, height: 1000 }`、`{ fit: "width", width: 1 }`、`{ fit: "width", width: 333 }`、`{ fit: "width", width: 800 }` の定義がある | それぞれ、`outputSize` を `{ width: 1600, height: 1200 }`、`{ width: 1000, height: 1000 }`、`{ width: 1, height: 1 }`、`{ width: 333, height: 250 }`、`{ width: 800, height: 600 }` にして `transform` を呼ぶ | どれも `ok: true` を返す。出力の幅と高さは、それぞれ `outputSize` と一致する |
| TC-ports-imageProcessor-31 | `size` が `{ fit: "width", width: 400 }` の定義と、`{ fit: "contain", width: 300, height: 300 }` の定義がある | `definition.size` から求まる値と異なる `outputSize`（それぞれ `{ width: 123, height: 77 }`、`{ width: 300, height: 300 }`）を渡して `transform` を呼ぶ | どちらも `ok: true` を返す。出力の幅と高さは、渡した `outputSize` と一致する |

## transform の画像形式

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-imageProcessor-37 | PNG、WebP、AVIF の元の画像と、`output` が `{ format: "jpeg", quality: 80 }` の定義がある | それぞれ、`sourceFormat` を元の形式にして `transform` を呼ぶ | どれも `ok: true` を返す。出力は `jpeg` で、大きさは `outputSize` |
| TC-ports-imageProcessor-38 | JPEG の元の画像と、`quality` が 1 の定義、`quality` が 100 の定義がある | それぞれ `transform` を呼ぶ | どちらも `ok: true` を返す。出力は定義の形式で、大きさは `outputSize` |
| TC-ports-imageProcessor-39 | 透過を持つ PNG と WebP の元の画像と、`output` が `{ format: "jpeg", quality: 80 }` の定義がある | それぞれ `transform` を呼ぶ | どちらも `ok: true` を返す。`MediaFormat.detect` は出力を `jpeg` と判定し、`measure(bytes, "jpeg")` は `outputSize` を返す |
| TC-ports-imageProcessor-41 | 透過を持つ PNG の元の画像と、`output` が `{ format: "png" }` の定義がある | `transform` を呼ぶ | `ok: true` を返す。出力は `png` で、大きさは `outputSize` |

## transform の向き

出力の大きさは `outputSize` で決まる。ここで確かめるのは、出力が向きの情報を適用済みで、出力の `measure` が `outputSize` を返すこと。画素の向きそのものは、ポートのメソッドで観測できないので、対象にしない。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-imageProcessor-42 | 保存した画素が幅 800・高さ 600 で、向きが 90 度の回転を指す JPEG と、`size` が `{ fit: "width", width: 300 }` の定義、`{ fit: "cover", width: 200, height: 200 }` の定義がある | それぞれ、`outputSize` を `{ width: 300, height: 400 }`（向きを適用した幅 600・高さ 800 から求めた値）、`{ width: 200, height: 200 }` にして `transform` を呼ぶ | どちらも `ok: true` を返す。出力の `measure` は、それぞれ幅 300・高さ 400、幅 200・高さ 200 を返す |

## transform の失敗

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-imageProcessor-44 | 長さ 0 のバイト列、JPEG の先頭 3 バイトだけのバイト列、画像でない文字列のバイト列、`measure(bytes, format)` が `readable: false` を返すバイト列がある | それぞれ、`sourceFormat` を `jpeg`、`jpeg`、`png`、その `format` にして `transform` を呼ぶ | どれも `{ ok: false, reason: "source_unreadable" }` を返す。例外を投げない |
| TC-ports-imageProcessor-47 | 読み取れる PNG と JPEG がある | PNG は `sourceFormat` を `jpeg` に、JPEG は `png`、`webp`、`avif` にして、それぞれ `transform` を呼ぶ | どれも `{ ok: false, reason: "source_unreadable" }` を返す。例外を投げない |
| TC-ports-imageProcessor-50 | `avif` に対応しないアダプターと、読み取れる JPEG、`output` が `{ format: "avif", quality: 50 }` の定義がある | `transform` を呼ぶ | `{ ok: false, reason: "output_unsupported" }` を返す。例外を投げない |
| TC-ports-imageProcessor-51 | `avif` に対応しないアダプターと、長さ 0 のバイト列、読み取れる PNG、`output` が `{ format: "avif", quality: 50 }` の定義がある | どちらも `sourceFormat` を `jpeg` にして `transform` を呼ぶ | どちらも `{ ok: false, reason: "output_unsupported" }` を返す |
| TC-ports-imageProcessor-53 | `avif` に対応しないアダプターと、読み取れる JPEG、長さ 0 のバイト列、`output` が `{ format: "jpeg", quality: 80 }` の定義がある | どちらも `sourceFormat` を `jpeg` にして `transform` を呼ぶ | 読み取れる JPEG は `ok: true` を返し、出力は `jpeg` で、大きさは `outputSize`。長さ 0 のバイト列は `{ ok: false, reason: "source_unreadable" }` を返す |
| TC-ports-imageProcessor-55 | `avif` に対応しないアダプターと、読み取れる AVIF がある | `measure(bytes, "avif")` を呼ぶ | 幅と高さを `readable: true` で返す。出力の画像形式を絞っても、入力の形式は制限しない |

## 打ち切り

打ち切りの時間を短い値にしたアダプターと、応答しないよう指定した試験用の相手で確かめる。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-imageProcessor-56 | 読み取れる JPEG がある。`measure` に応答しないよう指定した試験用の相手がある | `measure(bytes, "jpeg")` を呼ぶ | 打ち切りの時間の後に `SystemError`（理由 `IMAGE_PROCESSING_TIMED_OUT`）で失敗する。`readable: false` を返さない |
| TC-ports-imageProcessor-57 | 読み取れる JPEG と、定義がある。`transform` に応答しないよう指定した試験用の相手がある | `transform` を呼ぶ | 打ち切りの時間の後に `{ ok: false, reason: "processing_failed" }` を返す。例外を投げない |
| TC-ports-imageProcessor-58 | 上の打ち切りの後 | 指定を解いて、同じ引数で `transform` を呼ぶ | `ok: true` を返す。前の呼び出しの結果が影響しない |

## 状態・並行性・UnitOfWork

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-imageProcessor-59 | 読み取れる元の画像と、読み取れない元の画像がある | それぞれ `transform` を呼び、呼び出しの前後で `source` を比べる | どちらも `source` の内容は変わらない |
| TC-ports-imageProcessor-61 | 元の画像 A（幅 800・高さ 600）、B（幅 400・高さ 400）と、読み取れないバイト列がある | A と B の `transform` を、異なる定義と `outputSize` で、読み取れないバイト列の `transform` と並行して呼ぶ | A と B は `ok: true` を返し、それぞれの出力は、自分の定義の形式と `outputSize` を持つ。読み取れないバイト列は `ok: false` を返す |
| TC-ports-imageProcessor-63 | 読み取れる画像がある | 読み取れないバイト列の `measure` の後に、読み取れる画像の `measure` を呼ぶ | 読み取れる画像の幅と高さを返す。前の呼び出しの結果が影響しない |
| TC-ports-imageProcessor-64 | 読み取れる画像がある | `UnitOfWorkProvider.run` の外で `measure` と `transform` を呼ぶ | どちらも成功する |
