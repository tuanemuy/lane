# コンテンツを作成する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-createContent-01 | 現行スキーマに、複数件を持つモデル M がある。生成したコンテンツID X のコンテンツは無い | 編集権限を持つメンバーが、M、X、入力エラーの無い値を指定して作成する | 「作成した」と、コンテンツID X、`version` 0、空の入力エラーを返す。X のコンテンツは、未公開で、通常の管理対象にあり、入力の値を下書きに持つ。作成日時と下書きの更新日時は作成の時刻。最後の変更は、下書きの保存と、作成したメンバーの名称。`content.publication_changed` を記録しない |
| TC-content-createContent-02 | M に必須の項目と、文字数の制約を持つ項目がある | 編集権限を持つメンバーが、必須の項目を欠き、文字数の制約に違反する値で作成する | エラーにしない。コンテンツを作り、必須項目の欠落と制約違反を、下書きの入力エラーとして返す |
| TC-content-createContent-03 | M のコンテンツ参照の項目が、未公開の T を参照できる | 編集権限を持つメンバーが、T を参照する値で作成する | コンテンツを作る |
| TC-content-createContent-04 | M に数値の項目がある | 編集権限を持つメンバーが、数値の項目に数値でない文字列を指定して作成する | `ValidationError` を返す。項目ごとの理由を添える。コンテンツは作られない |
| TC-content-createContent-05 | M は非推奨のモデル | 編集権限を持つメンバーが、M のコンテンツを作成する | `BusinessRuleError`（`ModelDeprecated`）を返す。コンテンツは作られない |
| TC-content-createContent-06 | M に非推奨の項目がある | 編集権限を持つメンバーが、非推奨の項目に値を指定して作成する | `ValidationError` を返す。該当する項目を示す。コンテンツは作られない |
| TC-content-createContent-07 | X のコンテンツが M に既にあり、最新の内容の値は V。`version` は 0 | 編集権限を持つメンバーが、M、X、V と等しい値を指定して作成する | 何も書かずに、「同じ内容で既にある」と、X のコンテンツを返す。`version` は 0 のまま。M のコンテンツは増えない |
| TC-content-createContent-08 | X のコンテンツが既にあり、最新の内容の値は V | 編集権限を持つメンバーが、X と、V と違う値を指定して作成する | `ConflictError`（`ALREADY_EXISTS`）を返す。X のコンテンツは変わらない |
| TC-content-createContent-09 | トークンは `content_manage` の許可と編集権限を持ち、範囲に M を含む。コンテンツID X のコンテンツは無い | トークンで、M、X、値を指定して作成する | 「作成した」を返す。X のコンテンツは、未公開で、指定した値を下書きに持つ。`version` は 0。下書きを作った実行者として、トークンと、トークンを指す値（名称または発行の経路）と、発行者または紐付くメンバーの名前を記録する |
| TC-content-createContent-10 | M に、初期値を持つ必須の項目 `tone` がある | `content_manage` の許可と必要な権限を持つトークンで、`tone` を欠く値を指定して作成する | エラーにしない。コンテンツを作る。下書きの `tone` は未設定で、`tone` の欠落を下書きの入力エラーとして返す |
| TC-content-createContent-12 | X のコンテンツは、完全削除されている | `content_manage` の許可と必要な権限を持つトークンで、M、X、値を指定して作成する | 「作成した」を返す。X のコンテンツができる |
| TC-content-createContent-13 | 単一モデル S の通常の管理対象に、コンテンツが無い | 編集権限を持つメンバーが、S、生成したコンテンツID、値を指定して作成する（最初の保存） | 「作成した」を返す。S のコンテンツは、未公開で、入力の値を下書きに持つ。`version` は 0 |
| TC-content-createContent-14 | 単一モデル S の通常の管理対象に、コンテンツが既にある | 編集権限を持つメンバーが、S と、別のコンテンツID を指定して作成する | `BusinessRuleError`（`SingleContentExists`）を返す。二件目は作られない |
| TC-content-createContent-15 | 単一モデル S のコンテンツは、ごみ箱にある一件だけ | 編集権限を持つメンバーが、S と新しいコンテンツID、値を指定して作成する | 「作成した」を返す。S の通常の管理対象に、新しいコンテンツができる。ごみ箱のコンテンツは変わらない |
| TC-content-createContent-16 | 単一モデル S の通常の管理対象に、コンテンツが無い | 二人のメンバーが、S の最初の保存を、別々のコンテンツID で並行して行う | 先に確定した保存がコンテンツを作る。確定の時点で検出された他方は `ConflictError`（`SINGLE_EXISTS`）を返す。S の通常の管理対象のコンテンツは一件 |
| TC-content-createContent-17 | 指定するメディアは、ワークスペースに存在しない | 編集権限を持つメンバーが、そのメディアを使う値で作成する | `ConflictError`（`MEDIA_NOT_FOUND`）を返す。コンテンツは作られない |
| TC-content-createContent-18 | 必須の参照の項目に選んだ T が、選んだ後に完全削除されている | 編集権限を持つメンバーが、T を参照する値で作成する | エラーにしない。コンテンツを作る。T への参照は参照切れとして保存する |
| TC-content-createContent-19 | 編集中に、モデル M がスキーマの反映で削除されている | 編集権限を持つメンバーが、M のコンテンツを作成する | `NotFoundError` を返す。M のキーを示す。コンテンツは作られない |
| TC-content-createContent-22 | メンバーは編集権限を持たない | M のコンテンツを作成する | `ForbiddenError`（`PERMISSION_MISSING`）を返す。不足する権限（`content:edit`）を添える。コンテンツは作られない |
| TC-content-createContent-25 | M にメディアの項目がある。非公開のメディア Md がある | 編集権限を持つメンバーが、Md と、使用箇所ごとの説明（`alt` と `caption`）を指定した値で作成する | コンテンツを作る。下書きの値は、指定した `alt` と `caption` を持つ。Md の使用箇所に、作成したコンテンツの下書きが現れる。Md は非公開のまま |
| TC-content-createContent-26 | メンバー甲が、編集画面で M の新規作成の入力を始めた。セッションの期限切れの後、編集権限を持つ別のメンバー乙が再ログインした | 乙が、甲の入力を、入力したメンバーを甲として作成する | `ForbiddenError`（`EDITOR_MISMATCH`）を返す。コンテンツは作られない |
| TC-content-createContent-27 | M にリッチテキストの項目 `body` がある | 編集権限を持つメンバーが、`body` に、扱う要素に無い種類のブロックを含む本文を指定して作成する | `ValidationError` を返す。`body` を示す。コンテンツは作られない |
| TC-content-createContent-28 | M に、初期値を持つ非推奨でない項目 `tone` と、初期値を持つ非推奨の項目 `legacy` がある | 編集権限を持つメンバーが、`ContentDefaults.initial` の結果を値にして作成する | 「作成した」を返す。下書きは `tone` の初期値を持ち、`legacy` の値を持たない |
