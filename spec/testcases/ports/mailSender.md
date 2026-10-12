# MailSender

`spec/domains/form.md` の `MailSender` の契約を、すべてのアダプターが通すケースで定める。送信の方式に依らないケースだけを持つ。

- 「送信基盤」は、`connection.endpoint` が指す試験用の相手（`spec/domains/common.md`）。応答を指定でき、受けた接続とメールを観測できる
- 「使える設定」は、アダプターの送信の方式で使える接続先、接続条件、認証情報（`identity` は、方式が使わなければ `null`）。アダプターごとの試験が用意する。「送信者のアドレスの接続条件」は、使える設定の接続条件のうち、アダプターが送信者のアドレスとして解釈するもの。アダプターごとの試験が、その名前を示す。ケースは、使える設定から決まった手順で作った設定だけを使い、送信の方式ごとに行うケースと行わないケースを分けない
- 一本の接続で順に送るアダプターと、宛先ごとに別の接続で送るアダプターのどちらも通るケースだけを持つ
- 接続の後に切れた失敗の理由（`connection` か `unconfirmed`）は、アダプターごとの検証の対象にする
- `connection` は、使える設定を `MailConnectionInput.validate`（このアダプターの規則を渡す）で通した `MailSetting` で作った `MailConnection.register` の結果。`mail` は `Confidential.of` で包んだ `OutgoingMail`
- R1、R2、R3 は別の宛先

## settingRule

「`settingRule().check(candidate)`」は、`settingRule` が返した規則の `check` に、設定の接続先、接続条件、認証情報の `identity` の有無を `MailSettingCandidate` として渡すことを指す。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mailSender-01 | 使える設定 | `settingRule().check(candidate)` を呼ぶ | 空の列 |
| TC-ports-mailSender-02 | 使える設定。送信基盤へ接続できる場合と、接続できない場合 | `settingRule()` を呼び、`check(candidate)` を呼ぶ | どちらも空の列を返す。送信基盤は接続を受けない |
| TC-ports-mailSender-04 | 使える設定と、問題のある設定 | それぞれ `settingRule().check(candidate)` を二度呼ぶ | どちらも、二度とも同じ結果を返す。例外を投げない |
| TC-ports-mailSender-05 | 使える設定に、アダプターが解釈しない名前の接続条件を一つ加えた設定 | `settingRule().check(candidate)` を呼ぶ | `target` が `condition`（`name` は加えた名前）、`reason` が `unknown` の問題を含む |
| TC-ports-mailSender-06 | 使える設定の接続先を、アダプターの受け付けない形にした設定 | `settingRule().check(candidate)` を呼ぶ | `target` が `destination`、`reason` が `invalid` の問題を含む |
| TC-ports-mailSender-07 | アダプターの受け付けない形の接続先と、解釈しない名前の接続条件を持つ設定 | `settingRule().check(candidate)` を呼ぶ | 二つの問題を両方とも返す |
| TC-ports-mailSender-09 | 使える設定と、問題のある設定 | 二つの `check` を並行して呼ぶ | それぞれが、単独で呼んだ場合と同じ結果を返す |
| TC-ports-mailSender-10 | 使える設定。その接続条件の名前ごとに、その一つを除いた設定を作る（除くと接続条件が無くなる設定は、`MailConnectionInput.validate` が規則に渡さないので除く） | それぞれの設定で `settingRule().check(candidate)` を呼ぶ | どの結果も、空の列か、`target` が `condition`（`name` は除いた名前）、`reason` が `required` の問題だけを持つ列。ほかの名前と、ほかの `target` の問題を含まない |
| TC-ports-mailSender-11 | 使える設定から、送信者のアドレスの接続条件を除いた設定（ほかの接続条件が無い使える設定では、解釈しない名前の接続条件を一つ加える） | `settingRule().check(candidate)` を呼ぶ | `target` が `condition`（`name` は送信者のアドレスの接続条件の名前）、`reason` が `required` の問題を含む |
| TC-ports-mailSender-12 | 使える設定の、送信者のアドレスの接続条件の値を、メールアドレスの形式でない文字列にした設定 | `settingRule().check(candidate)` を呼ぶ | `target` が `condition`（`name` は送信者のアドレスの接続条件の名前）、`reason` が `invalid` の問題を含む |
| TC-ports-mailSender-13 | 使える設定の `hasIdentity` を逆にした設定 | `settingRule().check(candidate)` を呼ぶ | `target` が `identity` の問題を一件だけ返す。`reason` は、偽にした設定では `required`、真にした設定では `unknown` |

## send の基本動作

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mailSender-14 | 送信基盤がすべての宛先を受理する | `recipients` が `[R1]` の `send` を呼ぶ | 結果は 1 件。`address` は R1、`outcome` は `accepted`。送信基盤は 1 通を受け取り、その宛先は R1 だけ |
| TC-ports-mailSender-15 | 送信基盤がすべての宛先を受理する | `recipients` が `[R1, R2, R3]` の `send` と、`[R3, R1, R2]` の `send` を呼ぶ | 結果は 3 件ずつで、`address` は、それぞれ `recipients` の順。すべて `accepted`。送信基盤は呼び出しごとに 3 通を受け取り、どのメールも宛先は一つだけ |
| TC-ports-mailSender-17 | 送信基盤がすべての宛先を受理する | `recipients` が `[R2, R1, R2, R3, R1]` の `send` を呼ぶ | 結果は 3 件で、`address` は R2、R1、R3 の順。送信基盤は、宛先ごとに 1 通の 3 通を受け取る |
| TC-ports-mailSender-19 | 送信基盤がすべての宛先を受理する | 送信者の表示名と返信先を持つ `mail` で `send` を呼ぶ | 送信基盤が受け取ったメールの、送信者のアドレスは、`connection` の送信者のアドレスの接続条件の値に等しい。送信者の表示名、返信先、件名、本文は `mail` の値に等しい |
| TC-ports-mailSender-20 | 送信基盤がすべての宛先を受理する | `senderName` と `replyTo` が `null` の `mail` で `send` を呼ぶ | 送信基盤が受け取ったメールは、送信者の表示名と返信先を持たない |
| TC-ports-mailSender-21 | 送信基盤がすべての宛先を受理する | 本文が複数行で、ASCII でない文字を含む `mail` で `send` を呼ぶ | 送信基盤が受け取ったメールの件名と本文は、`mail` の値に等しい |
| TC-ports-mailSender-22 | 送信基盤がすべての宛先を受理する | 件名が空文字列の `mail` で `send` を呼ぶ | 結果は `accepted`。送信基盤が受け取ったメールの件名は空 |
| TC-ports-mailSender-23 | 送信基盤が、`connection` の認証情報での認証を受け付ける | `send` を呼ぶ | 送信基盤は、`connection.credentials.secret` を `SecretCipher.reveal` で戻した値での認証を受ける。`connection.credentials.identity` が `null` でなければ、`reveal` で戻したその値も認証に使う |

## 送信の失敗

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mailSender-24 | 送信基盤へ接続できない | `recipients` が `[R1, R2]` の `send` を呼ぶ | 例外を投げない。結果は 2 件で、どちらも `failed`（`connection`） |
| TC-ports-mailSender-25 | 送信基盤が認証を拒否する | `recipients` が `[R1, R2]` の `send` を呼ぶ | 例外を投げない。結果は 2 件で、どちらも `failed`（`authentication`）。送信基盤はメールを受け取らない |
| TC-ports-mailSender-26 | 送信基盤が R2 だけを宛先として拒否する | `recipients` が `[R1, R2, R3]` の `send` を呼ぶ | 例外を投げない。結果は R1 が `accepted`、R2 が `failed`（`recipientRejected`）、R3 が `accepted` |
| TC-ports-mailSender-29 | 送信基盤が、宛先の拒否でも認証の拒否でもない理由で、メールを受理しない | `recipients` が `[R1]` の `send` を呼ぶ | 例外を投げない。結果は `failed`（`rejected`） |
| TC-ports-mailSender-30 | 送信基盤が、R1 へのメールを最後まで受け取った後、受理の応答を返す前に接続を切る | `recipients` が `[R1]` の `send` を呼ぶ | 例外を投げない。結果は `failed`（`unconfirmed`） |
| TC-ports-mailSender-32 | 送信基盤が、R1 への送信を受けた後、応答を返さない | `recipients` が `[R1]` の `send` を呼ぶ | 例外を投げない。`MailSendPolicy.recipientTimeoutMs` までに、`failed`（`unconfirmed`）を返す |
| TC-ports-mailSender-33 | 送信基盤が、R1 を受理し、R2 への応答を返さない | `recipients` が `[R1, R2]` の `send` を呼ぶ | R1 は `accepted`、R2 は `failed`（`unconfirmed`）。`send` は、宛先の数と `MailSendPolicy.recipientTimeoutMs` の積までに結果を返す |
| TC-ports-mailSender-34 | `SecretCipher.reveal` が `SystemError`（理由 `SECRET_UNAVAILABLE`）を返す | `recipients` が `[R1, R2]` の `send` を呼ぶ | 例外を投げない。結果は 2 件で、どちらも `failed`（`authentication`）。送信基盤は、接続もメールも受け取らない |
| TC-ports-mailSender-35 | 送信基盤がすべての宛先を受理する | `recipients` が空の `send` を呼ぶ | `SystemError`（理由 `INVALID_QUERY`）。送信基盤はメールを受け取らない |

## 重複と再試行

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mailSender-36 | 送信基盤がすべての宛先を受理する | 同じ `connection`、`mail`、`recipients`（`[R1]`）で `send` を二度呼ぶ | どちらも `accepted`。送信基盤は R1 宛ての 2 通を受け取る |
| TC-ports-mailSender-37 | 送信基盤が、R1 への最初の試みを、宛先の拒否、またはほかの理由の不受理で拒否し、二度目以降を受理する | `recipients` が `[R1]` の `send` を一度呼ぶ | 結果は `failed`。送信基盤が R1 への送信を受けるのは 1 回だけで、メールを受理しない |

## 並行性

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mailSender-40 | 二つの送信基盤 A、B がすべての宛先を受理する | A を指す `connection` と B を指す `connection` で、`send` を並行して呼ぶ | A は A への呼び出しのメールだけを、B は B への呼び出しのメールだけを受け取る。どちらも `accepted` |
| TC-ports-mailSender-41 | 送信基盤がすべての宛先を受理する。`MailConnectionRepository` に、別の送信基盤を指す設定が保存してある | 保存していない `connection` で `send` を呼ぶ | 渡した `connection` の送信基盤だけがメールを受け取る。保存済みの設定は変わらない |
| TC-ports-mailSender-42 | 送信基盤がすべての宛先を受理する | 同じ `connection` で、別の `mail` と宛先の `send` を並行して呼ぶ | それぞれの結果は、その呼び出しの `recipients` と同じ順で、件数も一致する。メールの内容が呼び出しの間で混ざらない |

## 可視性

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-mailSender-43 | 送信基盤がすべての宛先を受理する | `UnitOfWorkProvider.run` の外で `send` を呼ぶ | UnitOfWork を開かずに `accepted` を返す |
| TC-ports-mailSender-44 | 送信基盤がすべての宛先を受理する | `UnitOfWorkProvider.run` の `fn` の中で `send` を呼び、その後に `fn` が例外を投げる | 送信基盤はメールを受け取っている。送信は取り消されない |
| TC-ports-mailSender-45 | 送信基盤が認証を拒否する | `send` を呼ぶ | 結果の各要素は `address` と `outcome` だけを持つ。認証情報の値を含まない |
