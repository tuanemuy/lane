# 派生形式の定義を登録済みの画像に適用する（`applyDerivativeDefinitions`）

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-media-applyDerivativeDefinitions-01 | 派生形式 a が `ready` の画像が 2 件ある。派生形式 b を追加した反映が確定している | その反映の `schema.applied` を処理する | 2 件とも、b が新しい世代の `generating`（変更前の派生形式なし）になる。a は変わらない。画像ごとに、b を指す `media.derivative_regeneration_requested` を 1 件記録する。b の実体は無い |
| TC-media-applyDerivativeDefinitions-02 | 派生形式 a が `ready` の画像がある。a の定義を変更した反映が確定している | `schema.applied` を処理する | a が新しい世代の `generating` になり、変更前の派生形式として生成済みの a を持つ。a を指すイベントを 1 件記録する。取得結果と配信は、変更前の a を返し続ける |
| TC-media-applyDerivativeDefinitions-03 | 派生形式 a が `generating` または `failed` で、変更前の派生形式を持つ画像がある。a の定義を変更した反映が確定している | `schema.applied` を処理する | a が新しい世代の `generating` になり、変更前の派生形式を引き継ぐ。a を指すイベントを 1 件記録する |
| TC-media-applyDerivativeDefinitions-04 | 派生形式 a、b が `ready` の画像がある。b を取り除いた反映が確定している | `schema.applied` を処理する | b の状態が無くなる。b の生成済みの実体は消去待ちになり、メディアの保管先から無くなる。a は変わらない。イベントを記録しない |
| TC-media-applyDerivativeDefinitions-05 | 派生形式 a を取り除いた反映が画像に適用され、a の状態が無い。その後に、同じキー a を再び定義した反映が確定している | `schema.applied` を処理する | a は、取り除く前の世代と異なる新しい世代の `generating` になる |
| TC-media-applyDerivativeDefinitions-06 | 派生形式 a が `ready` の画像がある。a を取り除いた反映と、同じ定義で a を再び定義した反映が、どちらも画像に適用される前に確定している | `schema.applied` を処理する | a は変わらない。イベントを記録しない |
| TC-media-applyDerivativeDefinitions-07 | 派生形式 a が `ready` の画像がある。派生形式の定義を変えない反映が確定している | `schema.applied` を処理する | 画像は変わらない。イベントを記録しない |
| TC-media-applyDerivativeDefinitions-08 | ワークスペースに文書のメディアがある。派生形式を追加した反映が確定している | `schema.applied` を処理する | 文書は変わらない |
| TC-media-applyDerivativeDefinitions-10 | イベントの反映の後に、派生形式 c を追加した次の反映が確定している | 先の反映の `schema.applied` を処理する | 処理の時点の現行スキーマの定義を適用する。c が `generating` になる |
| TC-media-applyDerivativeDefinitions-11 | 前の配送が一部の画像に適用した後に中断している。または、`media.registered` の処理が画像に定義を当て終えている | 同じイベントの再配送を処理する | 残りの画像に適用し、画像ごとにイベントを記録する。適用済みの画像は変わらず、イベントを記録しない |
| TC-media-applyDerivativeDefinitions-12 | 画像が 3 件ある。走査の順で 2 件目の保存が、同じ画像の派生形式の記録へのほかの保存と競合する | `schema.applied` を処理し、続けて、その再配送を処理する | 1 件目は適用済みになる。2 件目と 3 件目は変わらない。走査を続けずに `ConflictError`（理由 `STALE_VERSION`）を投げて終える。イベントは配送の対象に残る。再配送が 2 件目と 3 件目に適用する。1 件目は変わらず、イベントを記録しない |
| TC-media-applyDerivativeDefinitions-13 | 画像が 2 件ある。走査の途中で、派生形式の定義を変更した次の反映が確定する | 先の反映の `schema.applied` を処理し、続けて、次の反映の `schema.applied` と、先の反映の `schema.applied` の再配送を処理する | 先の反映の処理は、次の反映の確定の後の UnitOfWork が確定せず、走査を続けずに `ConflictError`（理由 `SCHEMA_CHANGED`）を投げて終える。イベントは配送の対象に残る。次の反映の処理が、2 件とも新しい定義にする。再配送は画像を変えず、イベントを記録しない。古い定義が、新しい定義の後に残らない |
| TC-media-applyDerivativeDefinitions-14 | 現行スキーマが派生形式 a、b を定義している。`states` が空の派生形式の記録を持つ画像が登録された | その登録の `media.registered` を処理する | a、b が、それぞれ新しい世代の `generating`（変更前の派生形式なし）になる。a、b を指す `media.derivative_regeneration_requested` を 1 件ずつ記録する。ほかの画像は変わらない |
| TC-media-applyDerivativeDefinitions-15 | 現行スキーマが派生形式を定義していない。画像が登録された | `media.registered` を処理する | 派生形式の記録は `states` が空のまま。イベントを記録しない |
| TC-media-applyDerivativeDefinitions-16 | 画像が登録された。`media.registered` の処理が、スキーマを読んだ後、確定の前に、派生形式 c を追加した反映が確定する | `media.registered` を処理し、続けて、その反映の `schema.applied` と、`media.registered` の再配送を処理する | `media.registered` の処理は確定せず、`ConflictError`（理由 `SCHEMA_CHANGED`）を投げて終える。イベントは配送の対象に残る。`schema.applied` の処理が、その画像に c を含む現行の定義を当てる。再配送は画像を変えず、イベントを記録しない |
