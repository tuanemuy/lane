# 止まった反映・事前確認を検出して失敗させる（failStalledSchemaApplies）

断りが無い限り、対象のワークスペース W は利用中。「W を対象に呼ぶ」は、定期ジョブが、W を入力に呼ぶことを指す。閾値は `ApplyStall.thresholdMinutes`。「閾値より前」は、閾値より 1 分長い時間の前、「閾値より後」は、閾値より 1 分短い時間の前を指す。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-schema-failStalledSchemaApplies-01 | W は、`preparing` の反映 J で `applying`。進行を最後に記録したのは閾値より前 | W を対象に呼ぶ | 1 を返す。ジョブ J は `failed`（`interrupted`）になり、`preparation` は準備の途中のままで、失敗の理由は `effects` を持たない。W は `idle` に戻り、J を未解消の失敗した反映（`failedApply`）として持つ。準備領域は残らない。現行スキーマ、版、データ、運用設定は変わらない |
| TC-schema-failStalledSchemaApplies-02 | W は、準備済み（`prepared`）の反映 J で `applying`。進行を最後に記録したのは閾値より前 | W を対象に呼ぶ | 1 を返す。ジョブ J は `failed`（`interrupted`）になり、`preparation` は準備を終えたもので、失敗の理由は準備済みの反映の `effects` を引き継ぐ。W は `idle` に戻り、準備領域は残らない |
| TC-schema-failStalledSchemaApplies-03 | 事前確認のジョブは `preparing`。進行を最後に記録したのは閾値より前。W は `idle` | W を対象に呼ぶ | 1 を返す。ジョブは `failed`（`interrupted`）になり、準備領域は残らない。`WorkspaceSchema` は変わらない |
| TC-schema-failStalledSchemaApplies-04 | W は、`preparing` の反映 J で `applying`。進行を最後に記録したのは閾値より後。ほかに、`checked` と `failed` の事前確認のジョブと、`succeeded` の反映のジョブがある | W を対象に呼ぶ | 0 を返す。何も変わらず、W は `applying` のまま |
| TC-schema-failStalledSchemaApplies-05 | W の反映と、止まった事前確認のジョブ一件が、どちらも閾値より前から進んでいない | W を対象に呼ぶ | 2 を返す。二つのジョブは、どちらも `failed`（`interrupted`）になる。W は `idle` に戻る |
| TC-schema-failStalledSchemaApplies-06 | W と、別のワークスペース W2 が、どちらも閾値より前から進んでいない反映で `applying` | W を対象に呼ぶ | 1 を返す。W は `idle` に戻る。W2 の `WorkspaceSchema` は変わらない |
| TC-schema-failStalledSchemaApplies-07 | 止まった事前確認のジョブが二件ある。一件の失敗の UnitOfWork が、並行した保存と競合する（`ConflictError`、`STALE_VERSION`） | W を対象に呼ぶ | 1 を返す。エラーを投げない。競合した一件は変わらず、もう一件は `failed`（`interrupted`）になる。競合した一件は、次の起動でも止まっていれば、再び対象になる |
| TC-schema-failStalledSchemaApplies-09 | 止まった事前確認のジョブを読んだ後、失敗の UnitOfWork の前に、そのジョブの準備が一区切り進んだ | W を対象に呼ぶ | 0 を返す。そのジョブは `preparing` のまま |
