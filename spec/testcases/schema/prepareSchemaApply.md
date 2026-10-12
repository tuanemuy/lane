# 反映・事前確認を準備する（prepareSchemaApply）

断りが無い限り、ワークスペースは利用中。「反映」は、`mode` が `commit` の `preparing` のジョブ J で、`WorkspaceSchema` は J で `applying`。「事前確認」は、`mode` が `dryRun` の `preparing` のジョブ。「準備する」は、`prepareSchemaApply` を、その要求の `schema.apply_requested`（続きとして記録されたものを含む）ごとに、ジョブが `preparing` でなくなるまで順に呼ぶことを指す。消費者が、`prepared` の後に続けて呼ぶ「反映を確定する」を含めない。「一回だけ呼ぶ」は、`prepareSchemaApply` を一度だけ呼ぶことを指す。「`prepared` になる」は、ジョブが `prepared` になり、ワークスペースが `applying` のままであることを、「`blocked` で失敗する」は、ジョブが `failed`（`blocked`）になり、ワークスペースが `idle` に戻って、そのジョブを未解消の失敗した反映（`failedApply`）として持つことを指す。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-schema-prepareSchemaApply-01 | 反映。公開中のコンテンツ、下書きを持つコンテンツ、ごみ箱のコンテンツ、履歴の版がある。反映後の公開内容は、型を満たさない入力エラーも一意性の違反も持たない | 準備する | 反映は `prepared` になり、`effects` を持つ。現行スキーマは変わらず、コンテンツと履歴の版の読み取りは、反映前の内容を返す。メディアの使用箇所は変わらない。`WorkspaceSchema` を保存しない |
| TC-schema-prepareSchemaApply-02 | 事前確認。データは上と同じ | 準備する | ジョブは `checked` になり、`blockers.total` は 0、`effects` を持つ。準備領域は残らない。`WorkspaceSchema`、コンテンツ、履歴の版、メディアの使用箇所、運用設定は変わらない |
| TC-schema-prepareSchemaApply-03 | 同じデータと同じスキーマ・移行定義に対する、事前確認と反映。反映後の公開内容に、必須の項目の欠落がある | それぞれを準備する | 事前確認のジョブは `checked` になり、反映は `blocked` で失敗する。事前確認のジョブの `blockers` と `effects` は、反映のジョブの失敗の理由のものと等しい |
| TC-schema-prepareSchemaApply-04 | 反映。移行定義が、任意の項目を補完値なしで必須にする。その項目が未設定の公開内容がある | 準備する | 反映は `blocked` で失敗する。失敗の理由の `blockers` は、そのコンテンツ、モデル、項目の位置を示す `typeError`（`missingRequired`）を持ち、`effects` を持つ。現行スキーマ、版、コンテンツ、履歴の版、メディアの使用箇所、運用設定は変わらない。Content と Media の準備領域は残らない。版の記録と操作履歴を作らず、`content.publication_changed` を記録しない |
| TC-schema-prepareSchemaApply-05 | 反映。移行定義が、テキストの項目を数値に変える。数値に変換できない文字列を持つ公開内容がある | 準備する | 反映は `blocked` で失敗する。`blockers` は、そのコンテンツと項目の位置、操作を示す `unconvertibleValue` を持つ |
| TC-schema-prepareSchemaApply-06 | 反映。同じ値の種類の変更。変換できない文字列を持つのは、下書き、ごみ箱のコンテンツ、履歴の版だけ | 準備する | 反映は `prepared` になる。`effects.losses` は、その操作で値を失う件数を持つ |
| TC-schema-prepareSchemaApply-07 | 反映。スキーマが、項目 slug に一意性を加える（移行定義は無い）。同じ slug の値を持つ公開内容が二件ある | 準備する | 反映は `blocked` で失敗する。`blockers` は、二件のコンテンツそれぞれの `uniqueViolation`（モデル、項目 slug、もう一方のコンテンツを `duplicates` に持つ）を持ち、`total` は 2 |
| TC-schema-prepareSchemaApply-08 | 反映。同じ一意性の追加。同じ値を持つのは、公開内容一件と、別のコンテンツの下書き | 準備する | 反映は `prepared` になる |
| TC-schema-prepareSchemaApply-09 | 反映。移行定義がモデル author を削除する。別のモデルの公開内容の本文が、author のコンテンツへのリンクを持つ | 準備する | 反映は `blocked` で失敗する。`blockers` は、参照元のコンテンツ、リッチテキストの項目の位置、リンク先を示す `referenceToRemovedContent` を持つ |
| TC-schema-prepareSchemaApply-10 | 反映。同じモデルの削除。別のモデルの公開内容が、任意のコンテンツ参照の項目で author のコンテンツを指す | 準備する | 反映は `prepared` になる。この参照は `blockers` にならない |
| TC-schema-prepareSchemaApply-11 | 反映。同じモデルの削除。別のモデルの公開内容が、必須のコンテンツ参照の項目で author のコンテンツだけを指す | 準備する | 反映は `blocked` で失敗する。`blockers` は、そのコンテンツと項目を示す `typeError`（`missingRequired`）を持つ |
| TC-schema-prepareSchemaApply-12 | 反映。移行定義が、任意のコンテンツ参照の項目を補完値なしで必須にする。公開内容が、その項目で、公開内容を持たないコンテンツを指す | 準備する | 反映は `blocked` で失敗する。`blockers` は、そのコンテンツ、項目の位置、参照先のコンテンツを示す `referenceNotPublished` を持つ。`typeError` を持たない |
| TC-schema-prepareSchemaApply-13 | 反映。移行定義が、モデルのキーを post から article に変え、任意の項目を補完値なしで必須にする。その項目が未設定の公開内容がある | 準備する | 反映は `blocked` で失敗する。`blockers` の `typeError` の `model` は、反映後のキー article |
| TC-schema-prepareSchemaApply-14 | 反映。スキーマが、二つのテキストの項目の最大文字数を縮める。両方の項目で超える値を持つ公開内容 A がある | 準備する | 反映は `prepared` になる。`effects.constraintViolations` は、A と二つの項目の位置を持つ一件だけを持ち、`total` は 1 |
| TC-schema-prepareSchemaApply-15 | 反映。未適用の M2 が項目 a を補完値なしで必須にし、M3 が a を任意に戻す。a が未設定の公開内容がある | 準備する | 反映は `prepared` になる。M2 を適用した時点の必須の欠落を確かめない |
| TC-schema-prepareSchemaApply-16 | 反映。移行定義が必須の項目を補完値なしで加える。公開内容を持たず、下書きだけを持つコンテンツがある | 準備する | 反映は `prepared` になる。`effects.draftsWithErrors` はそのコンテンツを持つ |
| TC-schema-prepareSchemaApply-17 | 反映。移行定義が、公開内容に値のある項目を削除し、別のモデルを削除する。削除するモデルに公開中のコンテンツがある | 準備する | 反映は `prepared` になる。`effects.changedPublished` は、値を失う公開内容のコンテンツを `updated`、削除するモデルの公開中のコンテンツを `unpublished` として持つ。`effects.losses` は、項目の削除では `contents.total` が 0 で、`values` に失う値の件数を持つ。モデルの削除では、`contents` が、完全削除されるコンテンツの件数と、コンテンツID の昇順で先頭のコンテンツを持つ |
| TC-schema-prepareSchemaApply-18 | 反映。スキーマが、フォーム contact の項目のキーを mail から email に変え、別のフォームのキーを inquiry から request に変える。contact の通知定義は mail を参照する | 準備する | 反映は `prepared` になる。`effects.brokenNotificationReferences` は、contact と mail、理由 `fieldRemoved` を持つ。`effects.forms.changed` は、contact の `removedFields` に mail、`addedFields` に email を持つ。`effects.forms` は、inquiry を `removed` に、request を `added` に持つ |
| TC-schema-prepareSchemaApply-20 | 事前確認。スキーマは上と同じ | 準備する | ジョブは `checked` になり、`effects` は、反映と同じ `forms` と `brokenNotificationReferences` を持つ |
| TC-schema-prepareSchemaApply-21 | 事前確認。準備の途中で、別の反映が確定して現行スキーマの版が変わった | 準備を続ける | ジョブは `failed`（`schemaChanged`）になる。準備領域は残らない |
| TC-schema-prepareSchemaApply-22 | 事前確認。一回の呼び出しが `WorkspaceSchema` を読んだ後、その UnitOfWork の確定の前に、別の反映の確定が版を進めた | 一回だけ呼び、続けて同じイベントの再配送で呼ぶ | 一回目は `ConflictError`（`SCHEMA_CHANGED`）で何も反映せず、ジョブの進行は変わらない。再配送の呼び出しで、ジョブは `failed`（`schemaChanged`）になり、準備領域は残らない |
| TC-schema-prepareSchemaApply-23 | 反映。準備領域への書き込みが、容量の上限を超える | 準備する | ジョブは `failed`（`capacityExceeded`）になり、`preparation` は準備の途中のままで、失敗の理由は `effects` を持たない。ワークスペースは `idle` に戻り、準備領域は残らない。現行スキーマとデータは変わらない |
| TC-schema-prepareSchemaApply-24 | ジョブは `preparing` でない。準備済み（`prepared`）の反映、`succeeded` の反映、`failed` のジョブ、`checked` の事前確認のどれでも同じ | 一回だけ呼ぶ | 何も書かず、引いたジョブをそのまま返す。イベントを記録しない |
| TC-schema-prepareSchemaApply-26 | 反映 | 並行して、同じ反映に二度呼ぶ | 一方が、ジョブの `ConflictError`（`STALE_VERSION`）になる。もう一方の進行だけが残る |
| TC-schema-prepareSchemaApply-29 | 反映。コンテンツが、`SchemaLimits.preparationChunk` の 2 倍より多く 3 倍より少ない N 件あり、履歴の版は無い | 一回だけ呼ぶ | 反映は `preparing` のまま。`Preparation.progress` は、`processed` が `preparationChunk`、`total` が N。`schema.apply_requested` を改めて記録する |
| TC-schema-prepareSchemaApply-30 | 反映。コンテンツは上と同じ N 件。コンテンツID の昇順で最初と最後のコンテンツの公開内容に、必須の項目が欠けている | 準備する | 反映は `blocked` で失敗する。`blockers.total` は 2 で、二件とも `items` にある |
| TC-schema-prepareSchemaApply-31 | 反映。コンテンツは同じ N 件で、履歴の版は無い。一回だけ呼んだ後 | 準備する | 反映は `prepared` になる。`Preparation.progress` は、`processed` と `total` がどちらも N。`effects` は、一度に準備した場合と等しい |
| TC-schema-prepareSchemaApply-32 | 反映。二回の呼び出しが確定した後、次の呼び出しの UnitOfWork が確定しなかった | 同じイベントの再配送で、準備する | 確定しなかった時点では、反映は `preparing`、ワークスペースは `applying` のままで、`Preparation.progress` は、確定した二回の後の値。準備を終えた反映の `effects` は、確定しなかった呼び出しが無い場合と等しい |
| TC-schema-prepareSchemaApply-33 | 反映。コンテンツも履歴の版も無い | 準備する | 反映は `prepared` になる。`Preparation.progress` は、`processed` と `total` がどちらも 0 |
| TC-schema-prepareSchemaApply-34 | 事前確認。コンテンツは N 件で、履歴の版は無い。一回だけ呼んだ後に、コンテンツを 1 件作成し、別のコンテンツを 1 件完全削除した | 準備を続ける | エラーにしない。ジョブは `checked` になる。準備領域は残らない |
| TC-schema-prepareSchemaApply-35 | 反映。未適用の M2 がテキストの項目 a を数値に変え、M3 が a を削除する。数値に変換できない文字列を a に持つ公開内容がある | 準備する | 反映は `prepared` になる。変換できない値は `blockers` にならない。`effects.losses` は、その値を失う操作を持つ |
| TC-schema-prepareSchemaApply-36 | 反映。未適用の M2 がテキストの項目 a を数値に変え、M3 が a のキーを b に変える。数値に変換できない文字列を a に持つ公開内容がある | 準備する | 反映は `blocked` で失敗する。`blockers` の `unconvertibleValue` の `path` は、反映後のキー b で、`operation` は M2 の操作 |
| TC-schema-prepareSchemaApply-37 | 反映。失敗する公開内容が、`SchemaLimits.sampleLimit` より多い | 準備する | 反映は `blocked` で失敗する。`blockers.total` は該当する件数、`blockers.items` は、コンテンツID の昇順で先頭の `SchemaLimits.sampleLimit` 件 |
| TC-schema-prepareSchemaApply-38 | 反映。移行定義がモデル author を削除する。author のコンテンツ a がある。別のモデルのコンテンツ P の公開内容と下書きが、任意の複数件のコンテンツ参照の項目で a と、残るモデルのコンテンツ b を指す。別のコンテンツ Q の下書きが、任意の一件のコンテンツ参照の項目で a を指す。履歴の版は無い | 準備する | 反映は `prepared` になる。`effects.losses` の author の `removeModel` は、`contents` に a を持ち、`values` は 3。この参照は `blockers` にならない |
