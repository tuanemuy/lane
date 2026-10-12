# 下書きを保存する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-saveDraft-01 | A は未公開で、通常の管理対象にある。`version` は 1、下書きの値は V1 | 編集権限を持つメンバーが、A、`version` 1、入力エラーの無い値 V2 を指定して保存する | A の下書きの値は V2 になり、下書きの更新日時は保存の時刻になる。`version` 2 と、空の入力エラーを返す。V1 は、下書きだった版として履歴に残る。版にした操作は下書きの保存、実行者は保存したメンバー。公開の変更の記録を追記せず、`content.publication_changed` を記録しない |
| TC-content-saveDraft-02 | A は公開中で、下書きを持たない | 編集権限を持つメンバーが、A と値を指定して保存する | A に下書きができる。A は公開中のまま。公開内容と、公開内容の更新日時は変わらない。履歴の版は作られない。公開取得の結果は変わらない |
| TC-content-saveDraft-03 | A は公開中で、下書きを持つ | 編集権限を持つメンバーが、A と値を指定して保存する | 下書きを置き換える。置き換え前の下書きは、下書きだった版として履歴に残る。公開内容と、公開内容の更新日時は変わらない |
| TC-content-saveDraft-04 | A のモデルに、必須の項目と、文字数の制約を持つ項目がある | 編集権限を持つメンバーが、必須の項目を欠き、文字数の制約に違反する値で保存する | エラーにしない。保存し、必須項目の欠落と制約違反を、下書きの入力エラーとして返す |
| TC-content-saveDraft-05 | A のモデルに一意な項目 `slug` がある。同じモデルの B の公開内容の `slug` は「x」 | 編集権限を持つメンバーが、`slug` が「x」の値で A を保存する | エラーにしない。保存する |
| TC-content-saveDraft-06 | T は未公開のコンテンツ | 編集権限を持つメンバーが、T を必須の参照で参照する値で A を保存する | エラーにしない。保存する |
| TC-content-saveDraft-07 | A のモデルに数値の項目がある | 編集権限を持つメンバーが、数値の項目に数値でない文字列を指定して保存する | `ValidationError` を返す。項目ごとの理由を添える。A の下書きと `version` は変わらない |
| TC-content-saveDraft-08 | メンバーが A を読み取った後に、スキーマの反映で A のモデルから項目 `old` が削除された。A の保存済みの内容は `old` の値を持たず、A の `version` は変わっていない | 編集権限を持つメンバーが、読み取ったときの `version` と、`old` の値を含む入力で保存する | `ValidationError` を返す。現行スキーマに合わない項目と理由を添える。A の下書きは変わらない |
| TC-content-saveDraft-09 | メンバーが A を読み取った後に、スキーマの反映で A のモデルから項目 `old` が削除された。A の下書きは `old` の値を持っていて、反映がその値を取り除き、A の `version` が進んだ | 編集権限を持つメンバーが、読み取ったときの `version` と、`old` の値を含む入力で保存する | `ConflictError`（`STALE_VERSION`）を返す。`ValidationError` にしない。A の下書きは変わらない |
| TC-content-saveDraft-10 | A のモデルの項目 `legacy` は非推奨。A の最新の内容の `legacy` の値は「a」 | 編集権限を持つメンバーが、`legacy` を「b」にした値で保存する | `ValidationError` を返す。該当する項目を示す。A の下書きは変わらない |
| TC-content-saveDraft-11 | A のモデルの項目 `legacy` は非推奨。A の最新の内容の `legacy` の値は「a」 | 編集権限を持つメンバーが、`legacy` を「a」のままにして、非推奨でない項目を変えた値で保存する | 保存する |
| TC-content-saveDraft-12 | A のモデルは非推奨 | 編集権限を持つメンバーが、A と値を指定して保存する | `BusinessRuleError`（`ModelDeprecated`）を返す。A の下書きは変わらない |
| TC-content-saveDraft-13 | メンバー甲が A を `version` 1 で読み取った後、メンバー乙が A の下書きを保存し、A の `version` は 2 | 甲が、A、`version` 1、乙の下書きと違う値を指定して保存する | `ConflictError`（`STALE_VERSION`）を返す。A の下書きは乙が保存した値のまま |
| TC-content-saveDraft-15 | A の `version` は 2、下書きの値は V2 | 編集権限を持つメンバーが、A、`version` 1、V2 と等しい値を指定して保存する | 何も書かずに、保存済みとして `version` 2 を返す。履歴の版は作られない |
| TC-content-saveDraft-16 | メンバーが A を読み取った後、A がごみ箱へ移動され、`version` が進んだ | 編集権限を持つメンバーが、A、読み取ったときの `version`、現在の下書きと違う値を指定して保存する | `ConflictError`（`STALE_VERSION`）を返す。A は変わらない |
| TC-content-saveDraft-17 | C はごみ箱にある | `content_manage` の許可と編集権限を持つトークンで、C、現在の `version`、値を指定して保存する | `BusinessRuleError`（`ContentInTrash`）を返す。C は変わらない |
| TC-content-saveDraft-18 | A は、メンバーが読み取った後に完全削除されている | 編集権限を持つメンバーが、A と値を指定して保存する | `NotFoundError` を返す |
| TC-content-saveDraft-19 | 指定するメディアは、ワークスペースに存在しない | 編集権限を持つメンバーが、そのメディアを使う値で A を保存する | `ConflictError`（`MEDIA_NOT_FOUND`）を返す。保存せず、A の下書きは変わらない |
| TC-content-saveDraft-20 | 必須の参照の項目に選んだ T と、本文のリンク先に選んだ U が、選んだ後に完全削除されている | 編集権限を持つメンバーが、T を参照し U へリンクする値で A を保存する | 保存する。T と U への参照は参照切れとして保存する |
| TC-content-saveDraft-23 | メンバーは、公開権限を持たず、編集権限を持つ。A は公開中 | メンバーが、A と値を指定して保存する | 下書きとして保存する |
| TC-content-saveDraft-24 | メンバーは編集権限を持たない | A と値を指定して保存する | `ForbiddenError`（`PERMISSION_MISSING`）を返す。不足する権限（`content:edit`）を添える。A は変わらない |
| TC-content-saveDraft-25 | トークンは `content_manage` の許可と編集権限を持ち、範囲に A のモデルを含む | トークンで、A、読み取ったときの `version`、値を指定して保存する | 下書きを保存する。下書きを作った実行者として、トークンと、トークンを指す値（名称または発行の経路）と、発行者または紐付くメンバーの名前を記録する |
| TC-content-saveDraft-28 | 単一モデル S のコンテンツ S1 があり、`version` は 2、下書きの値は V1 | 編集権限を持つメンバーが、S1、`version` 2、値 V2 を指定して保存する | 下書きの値は V2 になる。`version` 3 を返す。V1 は、下書きだった版として履歴に残る |
| TC-content-saveDraft-29 | メディア Md は、共有する説明を持つ。A の下書きは Md を使っていない | 編集権限を持つメンバーが、メディアの項目に Md を指定し、使用箇所ごとの説明（`alt` と `caption`）に、共有する説明と違う値を入れて A を保存する | A の下書きは、Md への参照と、入力した `alt` と `caption` を持つ。Md の共有する説明は変わらない。Md の使用箇所に、A の下書きでの使用（公開内容での使用でない）が加わる |
| TC-content-saveDraft-30 | メンバー甲が、編集画面で A の入力を始めた。セッションの期限切れの後、編集権限を持つ別のメンバー乙が再ログインした | 乙が、甲の入力を、入力したメンバーを甲として保存する | `ForbiddenError`（`EDITOR_MISMATCH`）を返す。A の下書きと `version` は変わらない |
| TC-content-saveDraft-31 | メンバー甲が、編集画面で A の入力を始めた。セッションの期限切れの後、甲が再ログインした | 甲が、入力したメンバーを甲として、読み取ったときの `version` で保存する | 下書きを保存する |
| TC-content-saveDraft-32 | A のモデルにリッチテキストの項目 `body` がある | 編集権限を持つメンバーが、`body` に、扱う要素に無い種類のブロックを含む本文を指定して保存する | `ValidationError` を返す。`body` を示す。A の下書きは変わらない |
| TC-content-saveDraft-33 | A のモデルにリッチテキストの項目 `body` がある。メディア Md は、画像でないファイル | 編集権限を持つメンバーが、`body` の本文の画像に Md を使う値で保存する | `ValidationError` を返す。該当する位置と理由（`imageRequired`）を添える。A の下書きは変わらない |
| TC-content-saveDraft-34 | A のモデルの単一選択の項目 `tone` は非推奨でなく、その選択肢 `old` は非推奨。A の最新の内容の `tone` は `old` | 編集権限を持つメンバーが、`tone` を非推奨でない選択肢 `calm` に変えた値で保存する | 保存する。下書きの `tone` は `calm` |
| TC-content-saveDraft-35 | A のモデルのブロックの項目 `sections` は非推奨でなく、ブロックの種類 `banner` は非推奨。A の最新の内容の `sections` は `banner` のブロックを一つ持つ | 編集権限を持つメンバーが、`sections` から `banner` のブロックを削除した値で保存する | 保存する。下書きの `sections` は `banner` のブロックを持たない |
| TC-content-saveDraft-36 | A のモデルの単一選択の項目 `tone` は非推奨でなく、その選択肢 `old` は非推奨。A の最新の内容の `tone` は `calm` | 編集権限を持つメンバーが、`tone` を `old` にした値で保存する | `ValidationError` を返す。`tone` を示す。A の下書きは変わらない |
