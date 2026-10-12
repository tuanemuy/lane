# 引き換えの値をトークンに引き換える

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-token-redeemPreviewExchange-01 | ワークスペース W のメンバー M がプレビューを開いて作った、期限内の `pending` の引き換え。M は在籍する | 引き換えの値を指定する。ほかのトークンを示さない | トークンの値と有効期限を返す。値は `TokenValue.prefixes.server` で始まり、W の ID を含む。トークンは `active`、サーバー用、M に紐付き、発行の経路は `preview`、許可はプレビュー取得だけ、範囲はワークスペース全体、期限は引き換えの時刻の `TokenLifetimePolicy.previewTokenMs` 後。引き換えは `redeemed` になり、発行したトークンを記録する。`token.issued` の操作履歴を、実行者を M、対象を発行の経路 `preview` のトークンとして残す。操作履歴はトークンの値と引き換えの値を含まない |
| TC-token-redeemPreviewExchange-03 | `redeemed` の引き換え（一度使った引き換えと、応答が表示アプリに届かなかった引き換え） | 同じ引き換えの値をもう一度指定する | `BusinessRuleError`（`PreviewExchangeRedeemed`）。二つ目のトークンを発行しない。先に発行したトークンは `active` のまま残る。操作履歴を追記しない |
| TC-token-redeemPreviewExchange-04 | 期限を過ぎた `pending` の引き換え | 引き換えの値を指定する | `BusinessRuleError`（`PreviewExchangeExpired`）。トークンも操作履歴も残らない。引き換えは変わらない |
| TC-token-redeemPreviewExchange-05 | 名簿で開設済みで、区画の `Workspace` が `active` のワークスペース W | W の ID を含み W のどの引き換えにも該当しない値と、ワークスペースの ID を取り出せない値を、それぞれ指定する | どちらも `NotFoundError` |
| TC-token-redeemPreviewExchange-07 | 期限内の `pending` の引き換え。プレビューを開いた後に、メンバーが解除された、または、ワークスペースが多要素認証を必須にし、メンバーは多要素認証を設定していない | 引き換えの値を指定する | `BusinessRuleError`（`TokenMemberUnavailable`）。トークンも操作履歴も残らない。引き換えは `pending` のまま |
| TC-token-redeemPreviewExchange-08 | 期限内の `pending` の引き換え。プレビューを開いた後に、ワークスペースが利用終了の途中になった | 引き換えの値を指定する | `BusinessRuleError`（`TokenMemberUnavailable`）。トークンも操作履歴も残らない。引き換えは `pending` のまま |
| TC-token-redeemPreviewExchange-09 | 期限内の `pending` の引き換え。引き換えがメンバーの認可を確かめた後、確定の前に、区画の `Workspace` が `closing` になった | 引き換えの値を指定する | `BusinessRuleError`（`TokenMemberUnavailable`）。`NotFoundError` にしない。トークンも操作履歴も残らない。引き換えは `pending` のまま |
| TC-token-redeemPreviewExchange-10 | 期限内の `pending` の引き換え。二つの引き換えが、どちらも、UnitOfWork の中で `pending` の引き換えを読んだ | 二つの引き換えが、それぞれ確定を試みる | 一方だけが成功する。成功しなかった側は `ConflictError`（`STALE_VERSION`）で、トークンも操作履歴も残さない。引き換えから発行されたトークンは一つ |
