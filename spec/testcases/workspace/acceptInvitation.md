# 招待を受諾する

ユースケースは [../../usecases/workspace.md](../../usecases/workspace.md) が定める。

- I は、「メンバーを招待する」が `active` のワークスペース W に作った、宛先 e、編集者の権限の組の招待。断りが無ければ、`pending` で期限内
- N は、作成したばかりで、初期所有者が未参加のワークスペース（所有者がいない `active`）。K は、N の、宛先 e の、`pending` で期限内の初期所有者の招待
- 断りが無ければ、W は多要素認証を必須にしていない。W と N は、名簿で開設済み
- 「名前とパスワード」は、条件を満たす名前とパスワードを指す

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-workspace-acceptInvitation-01 | e のアカウントが無い | I の値、名前、パスワードを入力する | W の ID、メンバーの ID、利用できるか（`granted`）、セッションの値、現在時刻の `SessionPolicy.activeMs` 後の有効期限を返す。メールアドレスが e で、多要素認証が `disabled` のアカウントを作る。所属の索引は、そのアカウントの項目として、W の `joined` の項目だけを持つ。そのアカウントは、編集者の権限の組を持つ W のメンバーになる。I は `accepted` になる |
| TC-workspace-acceptInvitation-02 | e のアカウントが無い。e へのログインの失敗が、最初の失敗から `LoginAttemptPolicy.limit.windowMs` の間に `LoginAttemptPolicy.limit.max` 回に達している | I の値、名前、パスワードを入力する | W の ID、メンバーの ID、利用できるか、ログインが要ることと、最初の失敗の `LoginAttemptPolicy.limit.windowMs` 後の `retryAt` を返す。セッションの値を返さない。アカウントとメンバーを作り、I は `accepted` になる |
| TC-workspace-acceptInvitation-04 | e のアカウント A がある。A でログインしている | I の値と A を入力する | W の ID、メンバーの ID、利用できるか（`granted`）を返す。セッションの値を返さない。A は、編集者の権限の組を持つ W のメンバーになる。I は `accepted` になる。A の名前、パスワード、多要素認証、`version` は変わらない。所属の索引に、A の W の `joined` の項目が加わる。A の「所属するワークスペースを一覧する」が W を返す |
| TC-workspace-acceptInvitation-05 | e のアカウントが無い | K の値、名前、パスワードを入力する | N の ID、メンバーの ID、利用できるか（`granted`）、セッションの値と有効期限を返す。N は `active` のまま、所有者の指定が、作ったメンバーになり、多要素認証を必須にしていない。作ったメンバーは N の所有者で、認可結果はすべての権限を持つ。K は `accepted` になる。操作履歴に、参加したメンバーを対象にした `member.joined` と `workspace.owner_changed`（変更前は `null`）を一件ずつ残す |
| TC-workspace-acceptInvitation-06 | e のアカウント A がある。A でログインしている。A は別のワークスペース V のメンバー | K の値と A を入力する | N のメンバーを作り、A のメンバーが N の所有者になる。K は `accepted` になる。V の A のメンバーと権限の組は変わらない |
| TC-workspace-acceptInvitation-07 | W は多要素認証を必須にしている。e のアカウントが無い | I の値、名前、パスワードを入力する | 利用できるかに `mfa_setup_required` を返す。メンバーを作り、I は `accepted` になる |
| TC-workspace-acceptInvitation-09 | I は `pending` で、有効期限を過ぎている。e のアカウントが無い | I の値、名前、パスワードを入力する | `BusinessRuleError`（`InvitationExpired`）。アカウント、メンバー、セッションを作らない。I は変わらない |
| TC-workspace-acceptInvitation-10 | I は `accepted`。I で参加した、e のアカウント A でログインしている | I の値と A を入力する | `BusinessRuleError`（`InvitationInvalid`）。使用済みであることを示さない。メンバーを作らない。A のメンバーと I は変わらない |
| TC-workspace-acceptInvitation-13 | I の値での「招待の内容と状態を確かめる」が `acceptable` を返した後に、I の有効期限を過ぎた。e のアカウントが無い | I の値、名前、パスワードを入力する | `BusinessRuleError`（`InvitationExpired`）。アカウント、メンバー、セッションを作らない |
| TC-workspace-acceptInvitation-14 | 入力する値に該当する招待が無い | その値、名前、パスワードを入力する | `BusinessRuleError`（`InvitationInvalid`）。利用終了の途中のワークスペースの招待と同じ応答を返す。アカウントを作らない。エラー情報は、入力した招待の値とパスワードを含まない |
| TC-workspace-acceptInvitation-17 | J は、利用終了の途中のワークスペース Cの、`pending` で期限内の招待 | J の値、名前、パスワードを入力する | `BusinessRuleError`（`InvitationInvalid`）。C の情報を示さない。アカウント、メンバー、セッションを作らない。J は変わらない |
| TC-workspace-acceptInvitation-18 | e のアカウントが無い | I の値、未入力の名前、条件を満たすパスワードを入力する | `ValidationError`（`InvalidAccountName`）。名前の項目に理由を持つ。アカウント、メンバー、セッションを作らない。I は `pending` のまま残り、直して送信し直すと参加できる |
| TC-workspace-acceptInvitation-19 | e のアカウントが無い | I の値、名前、条件を満たさないパスワードを入力する | `ValidationError`（`PasswordPolicyViolation`）。パスワードの項目に理由を持つ。アカウント、メンバー、セッションを作らない。I は `pending` のまま残る |
| TC-workspace-acceptInvitation-20 | e のアカウント A がある。ログインしていない | I の値、名前、パスワードを入力する | `UnauthenticatedError`（理由 `SESSION_INVALID`）。メンバーとセッションを作らない。A の名前とパスワードは変わらない。I は `pending` のまま残る |
| TC-workspace-acceptInvitation-21 | e のアカウント A がある。別のメールアドレスのアカウント B でログインしている | I の値と B を入力する | `BusinessRuleError`（`InvitationEmailMismatch`）。B は W のメンバーにならない。I は `pending` のまま残る |
| TC-workspace-acceptInvitation-22 | e のアカウント A がある。別のメールアドレスのアカウント B は、W のメンバー。B でログインしている | I の値と B を入力する | `BusinessRuleError`（`InvitationEmailMismatch`）。`InvitationInvalid` にしない。B のメンバーと I は変わらない |
| TC-workspace-acceptInvitation-23 | I は `pending` で、有効期限を過ぎている。e のアカウント A がある。ログインしていない | I の値、名前、パスワードを入力する | `BusinessRuleError`（`InvitationExpired`）。`UnauthenticatedError` にしない |
| TC-workspace-acceptInvitation-24 | e のアカウント A がある。A でログインしている。I の受諾の前段（所属の索引に A の W の `joining` の項目を置く）が確定した後に、I の取り消しが確定する | 受諾の後段を確定する | `BusinessRuleError`（`InvitationInvalid`）。メンバーを作らない。I は `revoked` のまま。所属の索引の A の W の項目は、`joining` のまま残る。操作履歴に `member.joined` を残さない |
| TC-workspace-acceptInvitation-25 | e のアカウントが無い。I の受諾の前段（アカウントの作成と、`joining` の項目）が確定した後に、I の取り消しが確定する | 受諾の後段を確定する | `BusinessRuleError`（`InvitationInvalid`）。メンバーとセッションを作らない。作ったアカウントは、W の `joining` の項目を持って残る。I は `revoked` のまま |
| TC-workspace-acceptInvitation-26 | e のアカウント A がある。A でログインしている。I の受諾が、前段の確定の後、後段の前に止まった。I は `pending` で期限内 | I の値と A を入力して、もう一度受諾する | W の ID、メンバーの ID、利用できるか（`granted`）を返す。A は W のメンバーになり、I は `accepted` になる。所属の索引は、A の W の `joined` の項目を一つだけ持つ |
| TC-workspace-acceptInvitation-54 | e のアカウントが無い。I の受諾が、前段（アカウントの作成と、`joining` の項目）の確定の後、後段の前に止まった。I は `pending` で期限内 | e と設定したパスワードで「パスワードでログインする」を行い、I の値と作ったアカウントを入力して受諾する | ログインは成功する。受諾は、W の ID、メンバーの ID、利用できるか（`granted`）を返す。作ったアカウントは W のメンバーになり、I は `accepted`、そのアカウントの W の項目は `joined` になる |
| TC-workspace-acceptInvitation-27 | e のアカウント A がある。A でログインしている。I の受諾の前段の確定の後、後段の確定の前に、参加の期限の前に、「所属の索引を修復する」が行われた | 受諾の後段と確かめ直しを確定する | 修復の後も A の W の項目は残る。参加は成立し、A の W の項目は `joined` になる |
| TC-workspace-acceptInvitation-28 | e のアカウント A がある。I の受諾が、後段の確定の後、確かめ直しの前に止まった。A の W の項目は `joining` | A でログインし、「所属するワークスペースを一覧する」を行う。続けて、「所属の索引を修復する」を行う | 一覧は W を返し、A の W の項目は `joining` のまま。修復が、A の W の項目を `joined` にする |
| TC-workspace-acceptInvitation-29 | e のアカウントが無い。I の受諾で、前段（アカウントの作成）と後段（受諾とメンバーの作成）が確定し、確かめ直し（項目を `joined` にする書き込みと、セッションの作成）が確定しない | I の値、名前、パスワードを入力する | エラーにしない。W の ID、メンバーの ID、ログインが要ることを返す。セッションの値を返さない。作ったアカウントは W のメンバーで、I は `accepted`。アカウントの W の項目は `joining` のまま |
| TC-workspace-acceptInvitation-30 | 上の受諾の後 | e と設定したパスワードで「パスワードでログインする」を行い、「所属するワークスペースを一覧する」を行う | ログインは成功する。一覧は W を返す |
| TC-workspace-acceptInvitation-55 | e のアカウント A がある。A でログインしている。I の受諾で、前段と後段が確定し、確かめ直しが確定しない | I の値と A を入力する | エラーにしない。W の ID、メンバーの ID、利用できるか（`granted`）を返す。A は W のメンバーで、I は `accepted`。A の W の項目は `joining` のまま |
| TC-workspace-acceptInvitation-31 | e のアカウント A がある。A でログインしている。I の受諾の前段が確定した。その後、参加の期限を過ぎてから、「所属の索引を修復する」が、参加を取り下げて、A の W の項目を外した | 受諾の後段を確定する | `BusinessRuleError`（`JoiningLapsed`）。`InvitationInvalid` にしない。メンバーを作らない。I は `pending` のまま。A の W の項目は無いまま。`Member` があって項目が無い組にならない |
| TC-workspace-acceptInvitation-32 | e のアカウント A がある。A でログインしている。I の受諾の前段が確定した。参加の期限を過ぎたが、所属の索引の修復は行われていない | 受諾の後段と確かめ直しを確定する | 参加は成立する（[../../domains/workspace.md](../../domains/workspace.md) の「Membership」の参加の期限）。A は W のメンバーになり、I は `accepted`、A の W の項目は `joined` になる |
| TC-workspace-acceptInvitation-33 | e のアカウント A がある。A でログインしている。I の受諾の前段が確定し、参加の期限を過ぎた。受諾の後段が参加の囲いを読み直した後に、「所属の索引を修復する」の取り下げが確定する | 受諾の後段を確定する | `BusinessRuleError`（`JoiningLapsed`）。メンバーを作らない。I は `pending` のまま |
| TC-workspace-acceptInvitation-34 | 同上。受諾の後段が先に確定する | 「所属の索引を修復する」の取り下げを確定する | 取り下げは確定しない。A の W の項目を外さない。A は W のメンバーで、認可結果は `granted` |
| TC-workspace-acceptInvitation-36 | e のアカウント A がある。A でログインしている。A は W のメンバーでない。W は、所有者がいない。e に宛てた、W の `pending` のメンバーの招待 I と、`pending` の初期所有者の招待 L がある。I の受諾と L の受諾が、どちらも事前の確認と前段を確定した後に、I の受諾の後段が確定した | L の受諾の後段と確かめ直しを確定する | `JoiningLapsed` にしない。メンバーを作らず、I の受諾が作った A のメンバーを W の所有者に指定する。A のメンバーが保持する権限の組は、編集者の組のまま。L は `accepted` になる。操作履歴に `workspace.owner_changed` を一件残す。A の W の項目は `joined` になる |
| TC-workspace-acceptInvitation-37 | 取り下げで `JoiningLapsed` になった受諾の後。I は `pending` で期限内 | I の値と A を入力して、もう一度受諾する | 参加は成立する。事前の確認が、進んだ囲いの世代を読み、前段が `joining` の項目を置く。A の W の項目は `joined` になる。I は `accepted` になる |
| TC-workspace-acceptInvitation-38 | e のアカウント A がある。A でログインしている。A は W のメンバーでない。A の W の `joined` の項目が残っている（解除の後段の前に止まった）。A 宛ての新しい招待 I がある | I の値と A を入力して受諾する | 前段が、項目を、新しい世代の `joining` に置き換える。参加は成立し、確かめ直しの後、項目は、その世代の `joined` になる |
| TC-workspace-acceptInvitation-39 | 同上。受諾の事前の確認の前に、「所属の索引を修復する」が、`joined` の項目と、`Member` が無いことを読んでいる | 受諾の前段、後段、確かめ直しを確定した後に、修復の取り下げを行う | 取り下げの確定は、A の `Member` を読み、招待を変えない。A の W の項目は `joined` のまま残る。A は W のメンバーで、「所属するワークスペースを一覧する」は W を返す |
| TC-workspace-acceptInvitation-40 | 同上。修復の取り下げの確定が、受諾の事前の確認より先に確定する | 受諾の前段、後段、確かめ直しを確定した後に、修復の項目の除外を確定する | 除外は、受諾の前段が置いた世代の項目を外さない。A の W の項目は `joined` のまま残る。A は W のメンバーで、「所属するワークスペースを一覧する」は W を返す |
| TC-workspace-acceptInvitation-42 | A は W のメンバーで、A の W の項目は `joined`。A の最後に利用したワークスペースは W。W は、所有者がいない（`vacant`）。A 宛ての初期所有者の招待 I がある | A が I を受諾する | A を所有者に指定する。前段が A の W の項目を新しい世代の `joining` に置き換え、確かめ直しが、その世代の `joined` にする。A の最後に利用したワークスペースは W のまま |
| TC-workspace-acceptInvitation-43 | e のアカウント A がある。A でログインしている。I の、A による二つの受諾が並行し、一方が先に確定する | もう一方を確定する | `BusinessRuleError`（`InvitationInvalid`）。W の A のメンバーは一人のまま。I は `accepted` のまま。負けた側は、所属の索引の A の W の項目を外さない。勝った側の確かめ直しの後、項目は `joined` |
| TC-workspace-acceptInvitation-45 | e のアカウント A がある。A でログインしている | I の値と A を入力する | 操作履歴に `member.joined` を一件残す。実行者と対象は、参加したメンバーで、名称は A の名前 |
| TC-workspace-acceptInvitation-46 | V は、所有者がいない（`vacant`）ワークスペースで、多要素認証を必須にしている。L は、V の、宛先 e の、`pending` で期限内の初期所有者の招待。e のアカウント A は、編集者の権限の組を持つ V のメンバーで、多要素認証は `enabled`。A でログインしている | L の値と A を入力する | V の ID、A のメンバーの ID、利用できるか（`granted`）を返す。メンバーを作らない。A のメンバーが V の所有者になり、認可結果はすべての権限を持つ。A のメンバーが保持する権限の組は変わらない。V は多要素認証を必須にしたまま。L は `accepted` になる。操作履歴に `workspace.owner_changed` を一件残し、`member.joined` を残さない。所有者の変更前は `null` |
| TC-workspace-acceptInvitation-53 | V は、所有者がいない（`vacant`）ワークスペース。V に、宛先 e の、`pending` で期限内の初期所有者の招待 L と、`pending` で期限内のメンバーの招待 I2 がある。e のアカウント A でログインしている | L の値と A を入力し、続けて I2 の値と A を入力する | L の受諾で、A は、空の権限の組を保持する V のメンバーで、V の所有者になる。I2 は `pending` のまま変わらない。I2 の受諾は `BusinessRuleError`（`InvitationInvalid`）。V の A のメンバーは一人のままで、I2 は `pending` のまま残る |
| TC-workspace-acceptInvitation-56 | N に、e のアカウント A の、編集者の権限の組を持つメンバー M と、`member:manage` を持つメンバー X がいる。A でログインしている。A の K の受諾の事前の確認（M を読む）の後、後段の前に、X による M の利用の解除が確定した | 受諾の後段を確定する | `BusinessRuleError`（`InvitationInvalid`）。`JoiningLapsed` にしない。メンバーを作らず、所有者を指定しない。K は `pending` のまま。続けて、A が K の値で受諾をやり直すと、A のメンバーを空の権限の組で作り、N の所有者に指定する |
| TC-workspace-acceptInvitation-57 | e のアカウント A がある。A でログインしている。A は W のメンバーでない。W は、所有者がいない。e に宛てた、W の `pending` のメンバーの招待 I と、`pending` の初期所有者の招待 L がある。I の受諾と L の受諾が、どちらも事前の確認と前段を確定した後に、L の受諾の後段が確定した | I の受諾の後段を確定する | `BusinessRuleError`（`InvitationInvalid`）。`JoiningLapsed` にしない。メンバーを作らない。W の A のメンバーは、L の受諾が作った一つだけ。I は `pending` のまま |
