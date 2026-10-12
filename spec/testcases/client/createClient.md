# クライアント（createClient）

`packages/client` の `createClient` が返すクライアントの振る舞い。契約は [app-api.md](../../usecases/app-api.md) の「クライアント」が定める。CMS の応答は、試験用の入口が返す。断りが無い限り、スキーマ S は、モデル article とフォーム contact を持ち、クライアントは S を指定して作る。「不一致の応答」は、照合の結果に `matches` が偽の対象を持つ応答を指す。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-client-createClient-01 | `fingerprints` を指定しない | `published.list` で article を取得する | 要求は、`SchemaFingerprint.of(S)` のスキーマ全体の識別子を添える。モデルごとの識別子を添えない |
| TC-client-createClient-02 | `fingerprints` は `target` | `published.list` で article を取得する | 要求は、article の識別子を添える。スキーマ全体と contact の識別子を添えない |
| TC-client-createClient-03 | `fingerprints` は `target`。article は、モデル author と page を許可する参照の項目を持つ | その項目を展開に指定して、article を取得する | 要求は、article、author、page の識別子を添える |
| TC-client-createClient-05 | `onFingerprintMismatch` を指定せず、`onMismatch` を指定している。CMS は、article の取得に、不一致の応答を返す | `published.list` で article を取得する | 取得結果を返し、`onMismatch` に、不一致の対象を渡す。失敗にしない |
| TC-client-createClient-07 | `onFingerprintMismatch` は `fail`。CMS は、不一致の応答を返す | `published.list` で article を取得する | `FingerprintMismatchError` を投げる。失敗は、不一致の対象を示す。取得結果を返さない |
| TC-client-createClient-08 | `onFingerprintMismatch` は `fail`。CMS は、一致の応答を返す | `published.list` で article を取得する | 取得結果を返す |
| TC-client-createClient-09 | `onFingerprintMismatch` は `fail`。CMS は、回答を保存し、受付完了と不一致を返す | `forms.submit` で contact へ送信する | `FingerprintMismatchError` を投げる。失敗は、CMS が返した受付完了の受付結果を持つ |
| TC-client-createClient-10 | CMS は、対象のコンテンツが存在しない応答（`notFound` で、`details.type` が `content`）を返す | `published.get` でコンテンツID を指定して取得する | `null` を返す。失敗にしない |
| TC-client-createClient-11 | CMS は、存在しないモデルのキーの応答（`notFound` で、`details.type` が `model`）を返す | `published.get` で取得する | `LaneError` を投げる。`error.kind` は `notFound` |
| TC-client-createClient-12 | CMS は、`kind` が `validation` のエラーの応答（`code`、`message`、`fieldErrors`、`details` を持つ）を返す | `manage.saveDraft` を呼ぶ | `LaneError` を投げる。`error` は、応答の `ApiError` と等しい。`error.kind` を `validation` と確かめた後に、`fieldErrors` を読める |
| TC-client-createClient-14 | CMS に到達できない | `published.list` で article を取得する | 到達できないことを示す失敗を投げる。`LaneError` にしない |
| TC-client-createClient-15 | CMS は、入力エラーによる受付失敗を返す | `forms.submit` で contact へ送信する | 失敗を投げず、理由付きの受付失敗の `Reception` を返す |
| TC-client-createClient-16 | トークン T1 を指定したクライアント | `withToken(T2)` が返すクライアントで、`preview.get` を呼ぶ | 要求は T2 を示す。接続先、スキーマ、識別子の設定は、元のクライアントと同じ。元のクライアントの要求は、T1 を示し続ける |
| TC-client-createClient-17 | CMS は、引き換えたトークンを返す | `exchangePreviewCode` を呼ぶ | 要求は、トークンを示さない。引き換えたトークンを返す |
