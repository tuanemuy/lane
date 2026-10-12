# コンテンツの候補を検索する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-content-searchContentCandidates-01 | モデル M1 に未公開の A、モデル M2 に公開中で下書きを持つ B、M1 にごみ箱の C がある | メンバーが、M1 と M2 を対象に、置き場所を通常の管理対象にして検索する | A と B を返し、C を含めない。総件数は 2。候補ごとに、モデル、コンテンツID、`version`、一覧表示用の項目の値、公開状態、下書きの有無、置き場所を返す。値は最新の内容のもの |
| TC-content-searchContentCandidates-02 | M1 の A の一覧表示用のテキスト項目の値が「春の特集」。M2 に B がある | メンバーが、M1 と M2 を対象に、検索語「特集」で検索する | A だけを返す |
| TC-content-searchContentCandidates-03 | M1 に A、M2 に B がある | メンバーが、M1 と M2 を対象に、B のコンテンツID を検索語にして検索する | B だけを返す |
| TC-content-searchContentCandidates-04 | M1 に、下書きを持つ A と、公開中で下書きを持たない B がある | メンバーが、下書き「あり」で検索する | A だけを返す |
| TC-content-searchContentCandidates-05 | M1 に、公開中の A と、未公開の B がある | メンバーが、公開状態「公開中」で検索する | A だけを返す |
| TC-content-searchContentCandidates-06 | M1 に、通常の管理対象の A と、ごみ箱の C がある | メンバーが、置き場所をごみ箱にして検索する | C だけを返す。置き場所はごみ箱 |
| TC-content-searchContentCandidates-07 | M1 の A の最新の内容の更新日時が、M2 の B のものより古い | メンバーが、M1 と M2 を対象に検索する | B、A の順に返す |
| TC-content-searchContentCandidates-08 | 現行スキーマに、キー `gone` のモデルが無い | メンバーが、`gone` を対象に検索する | `NotFoundError` を返す。キー `gone` を示す |
| TC-content-searchContentCandidates-09 | M1 がある | メンバーが、空の検索語で検索する | `ValidationError` を返す |
| TC-content-searchContentCandidates-10 | M1 に、条件に合うコンテンツが無い | メンバーが、検索語を指定して検索する | エラーにしない。0 件を返す |
