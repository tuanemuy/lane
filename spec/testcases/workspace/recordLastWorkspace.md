# 最後に利用したワークスペースを記録する

ユースケースは [../../usecases/workspace.md](../../usecases/workspace.md) が定める。

- W は `active` のワークスペース。断りが無ければ、実行者は、アカウント A の W のメンバー M で、所属の索引は、A の W の `joined` の項目を持つ
- 「最後に利用したワークスペース」は、所属の索引が A について持つ値（`MembershipIndex.findLastUsed`）を指す

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-workspace-recordLastWorkspace-01 | A は別のワークスペース V のメンバーでもあり、A の最後に利用したワークスペースは V | W を記録する | 正常に終わる。A の最後に利用したワークスペースは W になる。A の「所属するワークスペースを一覧する」が、ログインの後に開くワークスペースに W を返す。A の `version` は変わらない |
| TC-workspace-recordLastWorkspace-02 | M の権限の組は空。A の最後に利用したワークスペースは無い | W を記録する | 正常に終わる。A の最後に利用したワークスペースは W になる。権限は問わない |
| TC-workspace-recordLastWorkspace-04 | A の W の項目が `joining`（参加が後段の後、確かめ直しの前に止まった）。A の最後に利用したワークスペースは無い | W を記録する | 正常に終わる。A の最後に利用したワークスペースは W になる。項目は `joining` のまま変わらない |
| TC-workspace-recordLastWorkspace-06 | A の最後に利用したワークスペースは W | W を記録する | 正常に終わる。所属の索引を書かない |
