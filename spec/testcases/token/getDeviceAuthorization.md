# スクリプトの認証の内容を確かめる

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-token-getDeviceAuthorization-01 | プレビュー取得の許可を要求した `pending` の依頼。ワークスペース W と V に所属するアカウント | 承認の値を指定する | 依頼の内容（発行するトークンの有効期限は現在時刻の `TokenLifetimePolicy.deviceTokenMs` 後）、依頼の状態 `pending`、所属ワークスペースの列として W と V を返す。どちらも選べる |
| TC-token-getDeviceAuthorization-02 | スキーマ反映の許可を要求した `pending` の依頼。W と V に所属するアカウント。W でスキーマ変更権限を持ち、V で持たない | 承認の値を指定する | W を選べるワークスペースとして、V を、権限が足りず選べないワークスペースとして、不足する権限（`schema:change`）とともに返す。依頼は `pending` のまま変わらない |
| TC-token-getDeviceAuthorization-03 | `pending` の依頼。W に所属するアカウント。W は多要素認証を必須にし、アカウントは多要素認証を設定していない | 承認の値を指定する | エラーにしない。W を、多要素認証が未設定で選べないワークスペースとして返す。依頼は `pending` のまま変わらない |
| TC-token-getDeviceAuthorization-05 | `pending` の依頼。どのワークスペースにも所属しないアカウント | 承認の値を指定する | 依頼の内容と、空の列を返す |
| TC-token-getDeviceAuthorization-06 | `pending` の依頼。W と V に所属するアカウント | 承認の値を指定する。同じアカウントで、所属するワークスペースを一覧する（`listMyWorkspaces`） | 二つの結果のワークスペースの組と順は一致する |
| TC-token-getDeviceAuthorization-07 | 期限を過ぎた `pending` の依頼 | 承認の値を指定する | エラーにしない。依頼の状態 `expired` を返す |
| TC-token-getDeviceAuthorization-08 | メンバーが承認しなかった拒否の依頼と、承認の時点の権限の不足で拒否になった依頼 | それぞれの承認の値を指定する | どちらも、依頼の状態が拒否であることと、拒否の理由を返す。前者の理由は承認しなかったこと、後者の理由は権限が足りないことで、不足する権限を持つ |
| TC-token-getDeviceAuthorization-10 | W で承認された `approved` の依頼。W を利用できるメンバーであるアカウント | 承認の値を指定する | 依頼の状態 `approved` と、承認の時点のワークスペース W、承認の時刻の `TokenLifetimePolicy.deviceTokenMs` 後の有効期限を返す |
| TC-token-getDeviceAuthorization-11 | W で承認された `approved` の依頼。承認の後に、承認したメンバーが W の利用を解除された、または、W が多要素認証を必須にし、承認したアカウントは多要素認証を設定していない | 承認したアカウントで、承認の値を指定する | 依頼の状態 `approved` を返す。承認の時点のワークスペースを示さない |
| TC-token-getDeviceAuthorization-13 | W で承認され、受け取り済みの依頼。W を利用できるメンバーであるアカウント | 承認の値を指定する | 依頼の状態 `redeemed` と、承認の時点のワークスペース W を返す。トークンの値を含まない |
| TC-token-getDeviceAuthorization-14 | ログインしたアカウント | どの依頼にも該当しない承認の値を指定する | `NotFoundError` |
