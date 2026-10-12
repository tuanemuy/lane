# フォームの送信のフック（@repo/client/react）

`packages/client` の `@repo/client/react` が公開する、フォームの送信の状態を保つフック。契約は [app-api.md](../../usecases/app-api.md) の「入口」と「クライアント」が定める。CMS の応答は、試験用の入口が返す。断りが無い限り、フックは、フォーム contact を持つスキーマ S のクライアントと、contact を指定して使う。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-client-formSubmissionHook-01 | なし | フックを使い始める | 状態は入力。受付結果を持たない |
| TC-client-formSubmissionHook-02 | CMS は、応答を返していない | 送信内容を渡して送信する | 状態は送信中。要求は、クライアントの `forms.submit` と同じ経路と識別子を持つ |
| TC-client-formSubmissionHook-03 | CMS は、受付完了を返す | 送信する | 状態は、受付完了の受付結果を持つ |
| TC-client-formSubmissionHook-04 | CMS は、入力エラーによる受付失敗を返す | 送信する | 失敗を投げない。状態は、受付失敗の受付結果と、項目ごとの理由を持つ |
| TC-client-formSubmissionHook-05 | CMS は、送信制限による受付失敗（`rateLimited`）を返す | 送信する | 失敗を投げない。状態は、受付失敗の受付結果と、再び送信できる時刻（`retryAt`）を持つ。フォームの送信制限と、ブラウザ用トークンの利用量の制限を、同じ形で受ける |
| TC-client-formSubmissionHook-06 | 受付失敗の受付結果を持つ | 入力を直して、もう一度送信する。CMS は、受付完了を返す | 状態は送信中を経て、受付完了の受付結果を持つ。前の項目ごとの理由を持たない |
