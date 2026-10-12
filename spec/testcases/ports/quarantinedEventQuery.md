# QuarantinedEventQuery

`QuarantinedEventQuery` の契約（[../../domains/common.md](../../domains/common.md)）を、すべてのアダプターが通すケースで定める。

- イベントは、確定済みのワークスペース W の UnitOfWork（`UnitOfWorkProvider.run` に `{ kind: "workspace", workspaceId: W, condition: { kind: "none" } }` を渡す）の中の `collectEvents` で記録し、確定する。X は、もう一つの確定済みのワークスペース
- `count(W)` は、`PartitionReader.workspace(W)` の `quarantinedEventQuery.count()` を指す。ポートのメソッドは、ワークスペースの ID を引数に取らない
- 試験用の消費者は、イベントごとに処理の成否を指定できる。「常に失敗する消費者」は、届いたイベントの処理を毎回失敗させる試験用の消費者を指す。「成功する消費者」は、届いたイベントの処理を毎回終える試験用の消費者を指す
- 「隔離を待つ」は、配送が、そのイベントを配送の対象から外して隔離するまで待つことを指す。隔離までの配送の回数は、試験の構成が定める。ケースは、回数を前提にしない

## count

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-ports-quarantinedEventQuery-01 | イベントを一件も記録していない | `count(W)` を呼ぶ | 0 を返す |
| TC-ports-quarantinedEventQuery-02 | 種類 x のイベントは常に失敗する消費者に、種類 y のイベントは成功する消費者に届く構成で、W にそれぞれ 1 件を確定し、x の隔離を待ち、y の処理が終わっている | `count(W)` を呼ぶ | 1 を返す |
| TC-ports-quarantinedEventQuery-03 | 常に失敗する消費者に対して、W のイベント 3 件を確定し、隔離を待つ | `count(W)` を呼ぶ | 3 を返す |
| TC-ports-quarantinedEventQuery-06 | 常に失敗する消費者に対して、種類 x と種類 z のイベントを W に 1 件ずつ確定し、隔離を待つ | `count(W)` を呼ぶ | 2 を返す |
| TC-ports-quarantinedEventQuery-07 | 常に失敗する消費者に対して、W のイベント 2 件と X のイベント 1 件を確定し、隔離を待つ | `count(W)` と `count(X)` を呼ぶ | `count(W)` が 2 を、`count(X)` が 1 を返す |
| TC-ports-quarantinedEventQuery-08 | 常に失敗する消費者に対して、W のイベント 1 件を確定し、隔離を待つ | 区画に何も確定していないワークスペースの ID v で `count(v)` を呼ぶ | 0 を返す |
| TC-ports-quarantinedEventQuery-09 | 常に失敗する消費者に対して、W の UnitOfWork でドラフト 1 件の `collectEvents` を呼んだ後、`fn` が例外を投げる | `count(W)` を呼ぶ | 0 を返す |
| TC-ports-quarantinedEventQuery-10 | 常に失敗する消費者に対して、W のイベント 1 件を確定し、隔離を待つ | `count(W)` を二回呼ぶ | 二回とも 1 を返す |
| TC-ports-quarantinedEventQuery-11 | 常に失敗する消費者に対して、W のイベント 2 件と X のイベント 1 件を確定し、隔離を待つ | W の UnitOfWork で `WorkspaceContext.deleteEvents(BoundedDeletion.maxLimit)` を呼んで確定した後、`count(W)` と `count(X)` を呼ぶ | `count(W)` が 0 を、`count(X)` が 1 を返す |
