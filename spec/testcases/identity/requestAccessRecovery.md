# アクセス回復を依頼する

ユースケースは [../../usecases/identity.md](../../usecases/identity.md) が定める。

- A は、登録済みのアカウント
- 断りが無ければ、設置は設置全体のメール送信を使え、送信は `accepted` になる

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-identity-requestAccessRecovery-01 | A に `pending` の回復依頼が無い | A のメールアドレスで依頼する | `requested` を返す。A の、`channel` が `email` で、有効期限が現在時刻の `RecoveryPolicy.emailMs` 後の `pending` の回復依頼を一つ作る。設置の公開の URL と回復の値から `RecoveryLink.url` で作った回復用の URL と、有効期限を含むメールを、A のメールアドレスへ一通送る。出力に回復の値を含めない。A のパスワードと多要素認証の設定は変わらない |
| TC-identity-requestAccessRecovery-02 | A に、`channel` が `email` の `pending` の回復依頼 R がある | A のメールアドレスで依頼する | `requested` を返す。R は `revoked` になり、新しい `pending` の回復依頼を作る。R の値で「回復の値の状態を確かめる」が `revoked` を返す |
| TC-identity-requestAccessRecovery-03 | 入力するメールアドレスのアカウントが無い | 登録のないメールアドレスで依頼する | `requested` を返す。回復依頼を作らず、メールを送らない |
| TC-identity-requestAccessRecovery-05 | メールの送信が `failed` になる | A のメールアドレスで依頼する | `requested` を返す。作った回復依頼は `pending` のまま残る |
| TC-identity-requestAccessRecovery-07 | 設置全体のメール送信を使えない設置。A に、管理者復旧が発行した `pending` の回復依頼 R がある | A のメールアドレスで依頼する | `mail_not_configured` を返す。R は `pending` のまま変わらない。回復依頼を作らず、メールを送らない |
| TC-identity-requestAccessRecovery-08 | 設置全体のメール送信を使えない設置。入力するメールアドレスのアカウントが無い | 登録のないメールアドレスで依頼する | `mail_not_configured` を返す。アカウントがある場合と同じ値を返す |
| TC-identity-requestAccessRecovery-09 | A に、管理者復旧が発行した、`channel` が `operator` の期限内の `pending` の回復依頼 R がある | A のメールアドレスで依頼する | `requested` を返す。`channel` が `email` の `pending` の回復依頼を一つ作り、メールを送る。R は `pending` のまま変わらず、R の値で「回復の値の状態を確かめる」が `usable` を返す |
| TC-identity-requestAccessRecovery-10 | A に `pending` の回復依頼が無い。A のメールアドレスでの二つの依頼が並行し、一方が先に確定する | もう一方を確定する | どちらも `requested` を返す。エラーにしない。A の、`channel` が `email` の `pending` の回復依頼は、先に確定した依頼の一つだけ。後の依頼は、メールを送らない |
