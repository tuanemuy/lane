# メディアを登録する（`registerMedia`）

受け付ける種類を書かないケースは、`any` で登録する。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-media-registerMedia-01 | `content:edit` を持つメンバー。現行スキーマが派生形式を 2 つ定義している | 対応形式で容量の上限以下の画像を、元のファイル名を添えて登録する | `private` のメディアを返す。管理用の名称は元のファイル名、登録時の記録は形式・容量・幅・高さ、検索情報は空、共有する説明は未設定。その画像の、`states` が空の派生形式の記録ができ、`media.registered` を 1 件記録する。元のファイルの実体がメディアの保管先にあり、その登録の保管の予約は残らない。メディアは一覧に現れる |
| TC-media-registerMedia-02 | `content:edit` を持つメンバー。現行スキーマが派生形式を定義している | PDF を登録する | `private` の文書のメディアを返す。登録時の記録は形式と容量。派生形式の記録を作らない。イベントを記録しない |
| TC-media-registerMedia-04 | `content:edit` を持つメンバー。同じファイルを登録済み | 同じファイルをもう一度登録する | ID の異なる新しいメディアを返す。メディアは 2 件になる |
| TC-media-registerMedia-05 | `content:edit` を持つメンバー。ワークスペースがスキーマの反映の適用中 | 画像を登録する | `private` のメディアを返す |
| TC-media-registerMedia-06 | `content:edit` を持つメンバー | 対応形式でないファイルを登録する | `ValidationError`（理由 `MediaRejected`）。拒否の理由 `unsupported_format` と、対応形式・容量の上限（`MediaLimits`）を添える。メディア、実体、保管の予約のいずれも残らない |
| TC-media-registerMedia-07 | `content:edit` を持つメンバー | 容量の上限を超えるファイルを登録する | `ValidationError`（理由 `MediaRejected`）。拒否の理由 `too_large` と `MediaLimits` を添える。メディア、実体、保管の予約のいずれも残らない |
| TC-media-registerMedia-08 | `content:edit` を持つメンバー | 受け付ける種類を `image` にして、PDF を登録する | `ValidationError`（理由 `MediaRejected`）。拒否の理由 `not_image` と `MediaLimits` を添える。メディア、実体、保管の予約のいずれも残らない |
| TC-media-registerMedia-09 | `content:edit` を持つメンバー | 先頭の内容が画像の形式で、画像として読み取れないファイルを登録する | `ValidationError`（理由 `MediaRejected`）。拒否の理由 `unreadable_image` と `MediaLimits` を添える。メディアを登録しない |
| TC-media-registerMedia-10 | `content:edit` を持つメンバー。メディアの保管先の容量が足りない | 対応形式の画像を登録する | `UnavailableError`（理由 `STORAGE_CAPACITY_EXCEEDED`）。メディアを登録せず、イベントを記録しない。保管の予約と実体は残らない |
| TC-media-registerMedia-11 | `content:edit` を持たないメンバー | 画像を登録する | `ForbiddenError`（理由 `PERMISSION_MISSING`）。不足する権限 `content:edit` を示す。メディア、実体、保管の予約のいずれも残らない |
| TC-media-registerMedia-14 | `content:edit` を持つメンバー。実体の確定の後、登録の確定までに、ワークスペースの利用終了が始まる | 画像を登録する | `NotFoundError`。メディアは登録されず、イベントも記録されない。保管の予約は残らず、確定した実体は消去待ちになり、メディアの保管先から無くなる |
| TC-media-registerMedia-15 | `content:edit` を持つメンバー。現行スキーマが派生形式を定義している。登録のファイルの確認の後、登録の確定までに、派生形式の定義を変えた反映が確定する | 画像を登録する | `private` のメディアを返す。`media.registered` を記録する |
| TC-media-registerMedia-16 | `content:edit` を持つメンバー。実体の確定の後、登録の確定が失敗する | 画像を登録する | 登録の確定の失敗のエラーを返す。メディアは登録されず、イベントも記録されない。保管の予約は残らず、確定した実体は消去待ちになり、メディアの保管先から無くなる |
| TC-media-registerMedia-17 | `content:edit` を持つメンバー。実体の確定の後、登録の確定と、その後始末がともに失敗する | 画像を登録する | メディアは登録されない。保管の予約が残り、確定した実体はその予約が指す。その実体の配信の要求は `NotFoundError` になる。予約が回収できるようになった後に、定期ジョブ「メディアの実体の消去」が、予約を回収し、その実体を消去する |
| TC-media-registerMedia-18 | `content:edit` を持つメンバー。試験用の相手を、`put` の途中で失敗するよう、または `commit` に応答せず打ち切りの後に確定を届けるよう指定している | 画像を登録する | `MediaStorage` が返したエラーを返す。メディアは登録されない。保管の予約は残らない。そのメディアの ID の消去待ちは、消去を終えて残らない。`commit` の確定が後から届いても、メディアの保管先にその登録の実体は無い |
| TC-media-registerMedia-19 | `content:edit` を持つメンバー。`put` が試みを返した後、予約の確認の前に、その予約が回収されている | 画像を登録する | `ConflictError`（理由 `RESERVATION_NOT_FOUND`）。試みを確定せずに取り除く。メディアは登録されず、イベントも記録されない。メディアの保管先に、その登録の実体も試みの内容も無い |
| TC-media-registerMedia-20 | `content:edit` を持つメンバー。予約の確認の後、`commit` の前に、その予約が回収されている。回収が記録した消去待ちの消去は、まだ行われていない | 画像を登録し、その後に「消去待ちの実体を消去する」を実行する | 登録は `ConflictError`（理由 `RESERVATION_NOT_FOUND`）。実体を確定するが、メディアは登録されず、後始末は何も書かない。確定した実体は、回収が記録した消去待ちが指し、その消去でメディアの保管先から無くなる |
| TC-media-registerMedia-21 | `content:edit` を持つメンバー。ファイルの確認の後、保管の予約の確定までに、ワークスペースの利用終了が始まる | 画像を登録する | `NotFoundError`。保管の予約は確定せず、`MediaStorage.put` を呼ばない。メディア、実体、試みの内容のいずれも残らない |
