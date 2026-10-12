# 競合の扱い

[../../domains/index.md](../../domains/index.md) の「競合の扱い」の、入口の種類ごとの代表ケース。値の引き換えの結果は [../token/redeemPreviewExchange.md](../token/redeemPreviewExchange.md)（`TC-token-redeemPreviewExchange-09`）が、イベントの消費者と定期ジョブの結果は [eventConsumersAndScheduledJobs.md](eventConsumersAndScheduledJobs.md) が持つ。

- 断りが無ければ、ワークスペース W は利用でき、W に未公開のコンテンツ A がある
- 「`fn` を一回だけ呼ぶ」は、確定しなかった UnitOfWork をやり直さないことを、UnitOfWork の `fn` の呼び出しの回数で確かめることを指す

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-common-conflictHandling-01 | 実行者は、W の `content:edit` を持つメンバー。保存が認可を終えた後、確定の前に、W の利用終了の開始が確定する | A の下書きを保存する（Content の `saveDraft`） | `NotFoundError`。A は変わらない。`fn` を一回だけ呼ぶ |
| TC-common-conflictHandling-02 | 実行者は、W の `content:edit` を持つメンバー。保存が A と現行スキーマを読んだ後、確定の前に、別の保存が A を確定する場合と、スキーマの反映が確定してスキーマの版が進む場合 | A の下書きを保存する（Content の `saveDraft`） | 前者は `ConflictError`（`STALE_VERSION`）、後者は `ConflictError`（`SCHEMA_CHANGED`）を、そのまま返す。A は、別の保存と反映が確定した内容のまま。`fn` を一回だけ呼ぶ |
| TC-common-conflictHandling-03 | アカウント A に `pending` の回復依頼がある | A のメールアドレスで、アクセス回復の依頼を二つ並行して行う（Identity の `requestAccessRecovery`）。一方の確定が `ConflictError` で拒否される | どちらも `requested` を返す。エラーにしない |
| TC-common-conflictHandling-04 | `operator`。初期所有者が未参加のワークスペース W がある。発行し直しが W を確かめた後、招待の発行の確定の前に、W の利用終了の開始が確定する | W の初期所有者の招待を発行し直す（Installation の `reissueInitialOwnerInvitation`） | `ConflictError`（`WORKSPACE_NOT_ACTIVE`）を、そのまま返す。招待を作らない |
