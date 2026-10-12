# メンバーの利用の解除の影響を確かめる

ユースケースは [../../usecases/workspace.md](../../usecases/workspace.md) が定める。

- W は `active` のワークスペース。実行者 X は、W の `member:manage` を持つメンバー。M は、W のメンバー

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-workspace-previewMemberRemoval-01 | W に次のトークンがある。M が発行者で、`origin` が `manual` のアプリに紐付くトークン T1。M が発行者で、`origin` が `bootstrap` のアプリに紐付くトークン T2。M に紐付くトークン T3。別のメンバーが発行者で、`origin` が `manual` のトークン T4 | M を指定して確かめる | T1 と T2 の名称を返す。T3 と T4 を含めない。トークンの値を含めない。M とトークンは変わらない |
| TC-workspace-previewMemberRemoval-03 | M は、「メンバーの利用を解除する」で解除済み | M を指定して確かめる | `NotFoundError` |
| TC-workspace-previewMemberRemoval-04 | 実行者 Y は、W の、`member:manage` を持たないメンバー | Y が、M を指定して確かめる | `ForbiddenError`（理由 `PERMISSION_MISSING`）。不足する権限に `member:manage` を添える。トークンの名称を返さない |
| TC-workspace-previewMemberRemoval-05 | M が発行者で、`origin` が `manual` のアプリに紐付くトークンが三つある。有効な T1、失効済みの T5、期限切れの T6 | M を指定して確かめる | T1 の名称だけを返す。T5 と T6 を含めない |
