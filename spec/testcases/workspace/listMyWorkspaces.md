# 所属するワークスペースを一覧する

ユースケースは [../../usecases/workspace.md](../../usecases/workspace.md) が定める。

- A は、セッションのアカウント。断りが無ければ、A の最後に利用したワークスペース（`MembershipIndex.findLastUsed`）は無い
- 「メンバー」は、ワークスペースの区画に `Member` があり、所属の索引に、A のそのワークスペースの `joined` の項目があることを指す。「A の項目」は、所属の索引の、A の項目を指す。断りが無ければ、ワークスペースは名簿で開設済み
- 名称は、W1 が「Alpha」、W2 が「Beta」

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-workspace-listMyWorkspaces-01 | A は、`active` のワークスペース W1 と W2 のメンバー。どちらも多要素認証を必須にしていない。A が所属しない `active` のワークスペース V がある | 一覧する | W1、W2 の順に、それぞれの ID と名称を、どちらも `granted` で返す。V を含めない。ログインの後に開くワークスペースは W1 |
| TC-workspace-listMyWorkspaces-02 | A は W1 と W2 のメンバー。A の最後に利用したワークスペースは W2 | 一覧する | ログインの後に開くワークスペースは W2 |
| TC-workspace-listMyWorkspaces-03 | A は W1 と W2 のメンバーだった。A の最後に利用したワークスペースは W2。「メンバーの利用を解除する」が、W2 の A のメンバーを解除した | 一覧する | W1 だけを返す。ログインの後に開くワークスペースは W1 |
| TC-workspace-listMyWorkspaces-06 | A は W1 と W2 のメンバー。W2 は多要素認証を必須にしている。A の多要素認証は `disabled` | 一覧する | W1 を `granted`、W2 を `mfa_setup_required` で返す。W2 に、多要素認証を必須にしていることを返す |
| TC-workspace-listMyWorkspaces-09 | A は、どのワークスペースのメンバーでもない | 一覧する | 空の列と、ログインの後に開くワークスペースが無いことを返す。エラーにしない |
| TC-workspace-listMyWorkspaces-10 | A は W1 のメンバー。A の W2 の `joined` の項目があり、W2 の区画に A の `Member` が無い（解除が後段の前に止まった） | 一覧する | W1 だけを返す。A の W2 の項目は変わらない |
| TC-workspace-listMyWorkspaces-11 | A は W1 のメンバー。A の W2 の `joining` の項目があり、W2 の区画に A の `Member` がある（参加が後段の後、確かめ直しの前に止まった） | 一覧する | W1 と W2 を返す。A の W2 の項目は `joining` のまま変わらない |
