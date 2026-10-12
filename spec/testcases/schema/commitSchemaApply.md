# 反映を確定する（commitSchemaApply）

断りが無い限り、ワークスペースは利用中で、ジョブ J は、準備済み（`prepared`）の反映（`mode` は `commit`）。`WorkspaceSchema` は J で `applying`、現行スキーマの版は 3。ジョブの `actor` は、メンバー A が発行したトークン T。前提条件は、反映前のデータと、反映が適用する変更を書く。「確定する」は、J を入力に呼ぶことを指す。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-schema-commitSchemaApply-01 | 適用済みの移行定義は M1。反映は M2、M3 を適用する | 確定する | `succeeded` のジョブ J を返す。`success` は、版 4、`IdGenerator` が作った確定の操作の ID、反映後のスキーマの識別子、適用した移行定義 M2、M3、準備済みの反映の `effects` を持つ。ジョブは、実行者 T と発行者 A、受け付けた時点の T の表示用の名称と A の名前を保つ。ワークスペースは `idle` になり、現行スキーマは要求のスキーマ、版は 4、適用済みの列は M1、M2、M3（M2、M3 は `applied`）。版 4 の記録（`applied`、M2、M3 の名前と要約、データを失い得る操作、実行者 T と発行者 A、名称、反映後のスキーマ全体の識別子）を追記する。コンテンツ、履歴の版、メディアの使用箇所は変換結果に切り替わり、準備領域は残らない。`schema.applied`（版 4）を記録する。操作履歴に `schema.migrations_applied` を一件残す（実行者は T と A の名称、対象はスキーマ、名称は版 4）。`schema.reflected` の操作履歴は残さない |
| TC-schema-commitSchemaApply-02 | ワークスペースは、J の前に終えた反映 J0 を、未解消の失敗した反映（`failedApply`）として持つ | 確定する | ジョブ J は `succeeded`。ワークスペースは `idle` になり、未解消の失敗した反映を持たなくなる。`SchemaStatus.of` の `unresolved_failed_apply` の件数は 0 になる。ジョブ J0 は `failed` のまま変わらない |
| TC-schema-commitSchemaApply-03 | 反映は移行定義を適用せず、文字数の制約と表示名称だけを置き換える | 確定する | ジョブ J は `succeeded`、版は 4。版 4 の記録は `reflected` で、移行定義を持たない。適用済みの列は変わらない。操作履歴に `schema.reflected` を一件残す |
| TC-schema-commitSchemaApply-04 | 反映は直接反映（`plan.source` は `direct`）で、項目を削除する | 確定する | ジョブ J は `succeeded`、`plan.source` は `direct`、版は 4。`success.appliedMigrations` は空。版 4 の記録は `direct` で、失った項目を表すデータを失い得る操作を持つ。適用済みの列と、移行定義による構造は変わらない。操作履歴に `schema.pushed` を一件残す |
| TC-schema-commitSchemaApply-05 | 反映は画像の派生形式を一つ加え、一つの大きさを変える | 確定する | `schema.applied` のペイロードは、版 4 だけを持つ。派生形式のキーを持たない |
| TC-schema-commitSchemaApply-06 | 反映は、公開内容に値のある項目を削除する | 確定する | 公開の変更の記録を一件追記する。記録は、ジョブ J の `success.operationId` と同じ操作の ID、操作の種類 `migrated`、反映後の版 4、実行者 T、公開内容が変わったコンテンツごとの変更を持つ。`content.publication_changed` を一つ記録する。イベントは、ワークスペースの ID と、その操作の ID だけを持つ。ユースケースの出力は通し番号を持たない。表示アプリへの通知は、このユースケースでは作らない |
| TC-schema-commitSchemaApply-07 | 反映はモデルを削除する。そのモデルに公開中のコンテンツがある | 確定する | そのモデルのコンテンツは、公開中のものを含めて、ごみ箱のものと履歴の版とともに無くなる。公開の変更の記録の `changes` は、公開中だったコンテンツを公開取り消しとして持つ |
| TC-schema-commitSchemaApply-08 | 反映は、モデル article の公開内容に値のある項目を削除する。別のモデルの公開内容が、article のコンテンツ C を参照する。参照元は、この反映で変わらない | 確定する | 公開の変更の記録の `referrers` は、参照元のコンテンツの ID を持つ。C を含まない |
| TC-schema-commitSchemaApply-09 | 反映は、公開内容を変えない変更（表示名称の変更）だけを行う | 確定する | 公開の変更の記録を追記しない。`content.publication_changed` を記録しない。`schema.applied` は記録する |
| TC-schema-commitSchemaApply-10 | 反映はフォーム contact を加える。contact の紐付け待ちの運用設定は無い | 確定する | contact に新しい `FormId` を割り当てる。運用設定を、受付停止で作る |
| TC-schema-commitSchemaApply-11 | 初期構成が、キー contact の運用設定を受付中で宣言し、紐付け待ちになっている。反映は contact を加える | 確定する | その運用設定を、新しく割り当てた contact の `FormId` に紐付ける。運用設定を新しく作らない。紐付け待ちの設定は残らない。操作履歴に、対象がそのフォームの `bootstrap.setting_bound` を一件残す |
| TC-schema-commitSchemaApply-12 | トークンの範囲が、キー news のモデルを紐付け待ちで持つ。反映はモデル news を加える | 確定する | トークンの範囲は、news の `ModelId` に紐付け済みになる。操作履歴に、対象がそのトークンの `bootstrap.setting_bound` を一件残す |
| TC-schema-commitSchemaApply-13 | 受付中のフォーム contact に、保存済みの回答と未送信の通知がある。反映は contact を削除する | 確定する | contact の `FormId` を対応から外し、運用設定を削除済みにする。保存済みの回答と未送信の通知は変わらない |
| TC-schema-commitSchemaApply-14 | 反映は、フォームのキーを contact から inquiry に変える | 確定する | contact の運用設定は削除済みになる。inquiry は新しい `FormId` を持ち、運用設定は受付停止で新しく作られる。contact の運用設定を引き継がない |
| TC-schema-commitSchemaApply-15 | 反映は、通知定義が参照するフォームの項目を削除する | 確定する | ジョブ J は `succeeded`。運用設定の通知定義は変わらない。`success.effects.brokenNotificationReferences` はその参照を持つ |
| TC-schema-commitSchemaApply-16 | モデル post に、公開内容、下書き、履歴の版を持つコンテンツがある。トークンの範囲が post を指す。反映はモデルのキーを post から article に変える（`renameModel`） | 確定する | article は post の `ModelId` を保つ。コンテンツID、公開内容、下書き、公開状態、履歴の版を引き継ぐ。トークンの範囲は同じ `ModelId` を指し続ける |
| TC-schema-commitSchemaApply-17 | 反映は、同じ移行定義でモデル post を削除し、同じキー post のモデルを加える | 確定する | post は、`IdGenerator` が新しく作った `ModelId` を持つ。削除前の post のコンテンツは無くなる。削除前の `ModelId` を指すトークンの範囲の要素は、新しい post を指さない |
| TC-schema-commitSchemaApply-18 | 項目 title に値を持つ、公開内容、下書き、ごみ箱のコンテンツ、履歴の版がある。反映は項目のキーを title から heading に変える | 確定する | 公開内容、下書き、ごみ箱のコンテンツ、履歴の版のすべてで、値は heading にあり、title には無い。履歴の版は、heading の値を持つ現行スキーマの形になる |
| TC-schema-commitSchemaApply-19 | 項目 note に値を持つ、公開内容、下書き、ごみ箱のコンテンツ、履歴の版がある。反映は note を削除する | 確定する | 公開内容、下書き、ごみ箱のコンテンツ、履歴の版のすべてから、note の値が無くなる |
| TC-schema-commitSchemaApply-20 | ブロックの項目に、種類 quote のブロックを持つ内容がある。反映は quote を削除する | 確定する | 公開内容、下書き、ごみ箱のコンテンツ、履歴の版のすべてから、quote のブロックが無くなる |
| TC-schema-commitSchemaApply-21 | テキストの項目 price に、公開内容は「12」、下書きと履歴の版は「未定」を持つ。反映は price を数値に変える | 確定する | 公開内容の price は数値 12 になる。下書きと履歴の版の price は未設定になる |
| TC-schema-commitSchemaApply-22 | 単一選択の項目に、選択肢 old を選んだ内容がある。反映は old を削除し、置き換え先に new を指定する | 確定する | 公開内容、下書き、ごみ箱のコンテンツ、履歴の版のすべてで、old を選んだ値は new になる |
| TC-schema-commitSchemaApply-23 | 反映は、任意の項目 lead を補完値「未入力」で必須にする。lead が未設定の内容と、設定済みの内容がある | 確定する | 公開内容、下書き、ごみ箱のコンテンツ、履歴の版のすべてで、未設定の lead に「未入力」が入る。設定済みの lead は変わらない |
| TC-schema-commitSchemaApply-24 | 反映は、コンテンツ参照の項目に許可するモデルを減らす。外したモデルのコンテンツを指す値を持つ下書きがある | 確定する | 下書きの参照の値は変わらない |
| TC-schema-commitSchemaApply-25 | 項目 summary に値を持つ、公開内容、下書き、ごみ箱のコンテンツ、履歴の版がある。反映は、項目 excerpt を加え、summary から excerpt へ値を複写し、summary を削除する | 確定する | 公開内容、下書き、ごみ箱のコンテンツ、履歴の版のそれぞれで、excerpt は、その内容の summary の値を持つ。summary の値は無くなる |
| TC-schema-commitSchemaApply-27 | 項目 b が設定済みの内容と、未設定の内容がある。反映は、項目 a から b へ値を複写する | 確定する | 未設定だった b は a の値を持つ。設定済みだった b は変わらない。a の値は変わらない |
| TC-schema-commitSchemaApply-28 | 反映は、必須の項目 c を補完値「x」で加え、項目 a から c へ値を複写する。a に値を持つ内容と、a が未設定の内容がある | 確定する | a に値を持つ内容の c は a の値、a が未設定の内容の c は「x」になる |
| TC-schema-commitSchemaApply-29 | 数値の項目 a に 5 を持つ下書きと、テキストの項目 a2 に「abc」を持つ下書きがある。複写先は未設定。反映は、a からテキストの項目 t へ、a2 から数値の項目 n へ値を複写する | 確定する | t は「5」になる。n は未設定のまま |
| TC-schema-commitSchemaApply-30 | 二つのワークスペースが同じデータを持つ。それぞれに、同じ移行定義の列を適用する準備済みの反映がある | それぞれを確定する | 二つのワークスペースの、反映後の公開内容、下書き、ごみ箱のコンテンツ、履歴の版の値は等しい |
| TC-schema-commitSchemaApply-31 | 公開内容と、それと異なる下書きを持つコンテンツ、未公開のコンテンツ、ごみ箱のコンテンツがある | 確定する | 下書きは公開されない。公開内容と下書きの対応、公開状態、置き場所は変わらない |
| TC-schema-commitSchemaApply-32 | 反映は、テキストの項目の最大文字数を縮める。超える値を持つ公開内容がある | 確定する | ジョブ J は `succeeded`。その公開内容は公開中のまま、値は変わらない。`success.effects.constraintViolations` はそのコンテンツと項目の位置を持つ |
| TC-schema-commitSchemaApply-33 | 反映は、範囲を選択したトークンの範囲にあるモデルを削除する。確定と並行して、そのトークンの失効が確定した | 確定する | 確定は `ConflictError`（`STALE_VERSION`）で何も反映されない。反映を失敗させない。ワークスペースは準備済みの反映 J で `applying` のまま、準備領域は残る。現行スキーマとデータは変わらず、ジョブ J は `prepared` のまま |
| TC-schema-commitSchemaApply-34 | 上の競合で、準備済みのまま残った反映 | 再び確定する | ジョブ J は `succeeded`、版は 4 になる |
| TC-schema-commitSchemaApply-35 | 準備領域が、すべてのコンテンツと履歴の版の結果を過不足なく持たない | 確定する | 何も切り替えない。反映は失敗で終わり、ジョブ J は `failed`（`commitRejected`、`stagingMismatch`）で、準備済みの反映の `effects` を持つ。ワークスペースは `idle` に戻り、Content と Media の準備領域は残らない。現行スキーマ、版、データ、運用設定は変わらない。版の記録と操作履歴を作らず、`schema.applied` と `content.publication_changed` を記録しない |
| TC-schema-commitSchemaApply-36 | 切り替えが、公開内容の整合性の違反（`UNIQUE_VALUE_TAKEN`）で拒否される | 確定する | ジョブ J は `failed`（`commitRejected`、`publicationIntegrity`）。ワークスペースは `idle` に戻り、現行スキーマとデータは変わらない |
| TC-schema-commitSchemaApply-37 | メディアの使用箇所の切り替えが、切り替えた後の使用箇所の指すメディアが無いこと（`MEDIA_NOT_FOUND`）で拒否される | 確定する | ジョブ J は `failed`（`commitRejected`、`mediaMissing`）。ワークスペースは `idle` に戻り、準備領域は残らない。現行スキーマとデータは変わらない |
| TC-schema-commitSchemaApply-38 | メディアの使用箇所の切り替えが、公開内容の使うメディアが公開でないこと（`MEDIA_NOT_PUBLIC`）で拒否される | 確定する | ジョブ J は `failed`（`commitRejected`、`publicationIntegrity`）。ワークスペースは `idle` に戻り、準備領域は残らない。現行スキーマとデータは変わらない |
| TC-schema-commitSchemaApply-39 | 確定の UnitOfWork の `WorkspaceSchemaRepository.save` が、`SystemError` を投げる | 確定する | その `SystemError` を投げる。何も反映されず、反映を失敗させない。ワークスペースは準備済みの反映 J で `applying` のまま、準備領域は残る |
| TC-schema-commitSchemaApply-40 | 切り替えが、処理能力の上限を超える | 確定する | ジョブ J は `failed`（`capacityExceeded`）で、失敗の理由は準備済みの反映の `effects` を持つ。ワークスペースは `idle` に戻り、準備領域は残らない。現行スキーマとデータは変わらない |
| TC-schema-commitSchemaApply-41 | ジョブ J は準備済みの反映でない。`succeeded` の反映（版は 4）、`failed` の反映、`preparing` の反映、`checked` の事前確認のどれでも同じ | 確定する | 何も書かず、引いたジョブをそのまま返す。`WorkspaceSchema`、版、版の記録、操作履歴は変わらず、イベントを記録しない |
| TC-schema-commitSchemaApply-46 | フォーム contact は受付中で、保存済みの回答がある。反映は、contact の項目のキーを mail から email に変える | 確定する | contact の `FormId` と運用設定は変わらず、受付中のまま。保存済みの回答は変わらない。`success.effects.forms.changed` は、contact の `removedFields` に mail、`addedFields` に email を持つ |
| TC-schema-commitSchemaApply-47 | 反映は、範囲を選択したトークンの範囲を変えない変更だけを行う | 確定する | そのトークンを保存せず、`bootstrap.setting_bound` を残さない |
| TC-schema-commitSchemaApply-48 | 単一選択の項目と複数選択の項目に、選択肢 old を選んだ内容がある。反映は、それぞれの項目の old のキーを new に変える（`renameOption`） | 確定する | 公開内容、下書き、ごみ箱のコンテンツ、履歴の版のすべてで、old を選んだ値は new を選んだ値になる。`success.effects.losses` は空 |
| TC-schema-commitSchemaApply-49 | ブロックの項目に、種類 quote のブロックを持つ内容がある。反映は quote のキーを citation に変える（`renameBlockType`） | 確定する | 公開内容、下書き、ごみ箱のコンテンツ、履歴の版のすべてで、quote のブロックは、中の値を保ったまま citation のブロックになる。`success.effects.losses` は空 |
| TC-schema-commitSchemaApply-50 | 任意の単一選択の項目 s と任意の複数選択の項目 m は、どちらも選択肢 old を持つ。s で old を選んだ内容、m で old と a を選んだ内容、m で old だけを選んだ内容がある。反映は、s と m のそれぞれから old を削除し、置き換え先を未設定にする | 確定する | s で old を選んだ値は未設定になる。m で old と a を選んだ値は a だけを選んだ値に、old だけを選んだ値は未設定になる。公開内容、下書き、ごみ箱のコンテンツ、履歴の版のすべてで同じ。`success.effects.losses` は、二つの操作（`loss` は `optionValues`）を持つ |
