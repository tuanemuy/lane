# 引き継ぎ

この spec は、独立した読みの予算に達して止めた（17 周目の読みの反例 46 件が、16 周目の 44 件から減らなかった）。17 周目の読みの反例のうち、実装の挙動を分ける 16 件は、読み直さずに代案で直した。残りを「残る反例」に、申し送りを「申し送り」に置く。引用の行番号は、この文書を置いたときの spec のもの。

## 残る反例

「残る反例」があるあいだ、spec は範囲ごとにしか Implementation Ready にならない。範囲に取りかかる前に、その範囲の残る反例を直し、範囲のファイルと範囲が参照する上流の節を新しい読み手に読ませる。読み手が反例ゼロを返した範囲を、下の「Implementation Ready になった範囲」に記録する。全範囲が Implementation Ready になったら、この節を消す。

出どころは、引用された記述が 17 周目の読みの直前の直し（16 周目の反例の直し）で足された・変えられたものを「直しが生んだもの」、技術設計を規則・例・配線の層へ移したときに生まれたものを「層の移行が生んだもの」、それより前からあるものを「前からあったもの」とする。

| 範囲 | 残る反例 | 直しが生んだもの | 層の移行が生んだもの | 前からあったもの |
| --- | --- | --- | --- | --- |
| 体験設計の層 | 6 | 0 | 0 | 6 |
| flows と共通の型 | 1 | 0 | 1 | 0 |
| Content | 2 | 0 | 0 | 2 |
| Schema | 3 | 0 | 0 | 3 |
| Form | 6 | 1 | 1 | 4 |
| Webhook と入口 | 1 | 0 | 0 | 1 |
| Identity と Workspace | 3 | 0 | 1 | 2 |
| Token と Audit | 4 | 0 | 0 | 4 |
| Media | 0 | 0 | 0 | 0 |
| Installation | 4 | 0 | 0 | 4 |
| 計 | 30 | 1 | 3 | 26 |

### Implementation Ready になった範囲

なし

### 体験設計の層

#### 残る反例 01

- 条件: 1
- 効く範囲: scenario の content-editing（EDIT-13）、pages の P10
- 出どころ: 前からあったもの
- 読み手の引用:

  > **C7.1-01 の「公開」などの単位の実行者・日時を、現在の内容については確かめられない**
  > - 契約 `spec/review/contract.md:254`「下書きの保存・公開・下書きの破棄・公開取り消し・復元・移行・値の書き換えなど、利用者が理解できる単位で履歴を確認できる。公開内容だった版と下書きだった版を区別でき、実行者、日時、変更内容を確認し」
  > - `spec/scenario/content-editing.md:321`「現在の下書きと公開内容は版ではない。」、同 324 行「CMS は、版の内容とあわせて、その内容を作った操作の種類、実行者、日時を示す。」。`spec/pages/index.md:326` の P10 も版だけを一覧する
  > - 落ちている場所: EDIT-13 と P10。最初の公開（置き換える公開内容が無い）や最後の下書きの保存は、版を作らないので、その操作の実行者と日時は、後の操作で版になるまでどの画面にも現れない（下流の `domains/content.md:118` は `produced` を持つが、それを示すユースケースの出力も無い）
  > - 代案: EDIT-13 の手順 2 に「一覧の先頭に、現在の下書きと公開内容を、それを作った操作の種類・実行者・日時とともに示す」を加え、P10 の機能に同じ一行を加える

#### 残る反例 02

- 条件: 1
- 効く範囲: scenario の schema-migration
- 出どころ: 前からあったもの
- 読み手の引用:

  > **C6.1-05 前半と C6.1-07 の「同じデータからは同じ結果」が scenario に無い**
  > - 契約 `spec/review/contract.md:198`「同じ移行定義からは環境や時刻によらず同じ操作の並びを得る」、同 200 行「同じ移行定義の列を複数のワークスペースへ適用でき、同じデータからは同じ結果になる」
  > - scenario で対応するのは `spec/scenario/schema-migration.md:18`「同じスキーマと移行定義の列を、トークンを切り替えて複数のワークスペースへ反映できる。」だけで、結果が環境・時刻・ワークスペースによらないことはどのシナリオにも書かれていない（下流の `domains/schema.md:1750`「同じデータに同じ列を適用した結果は、ワークスペースによらない」は契約から直接引いている）
  > - 代案: SMIG の「共通の前提」に「同じ移行定義からは、環境と時刻によらず同じ操作の並びを得る。同じデータに同じ列を適用すると、ワークスペースによらず同じ結果になる」を一行加える

#### 残る反例 03

- 条件: 1
- 効く範囲: scenario の form-operation
- 出どころ: 前からあったもの
- 読み手の引用:

  > **C8.2-09 の「エラー情報」が scenario に無い**
  > - 契約 `spec/review/contract.md:295`「通常のログ、操作履歴、エラー情報へ回答本文や連絡先を記録しない」
  > - scenario で対応するのは `spec/scenario/hosting.md:13`「運用コマンドの出力と通常のログは、…回答の入力内容を含まない。」と `spec/scenario/workspace-admin.md:324`「操作履歴は、…フォーム回答の入力内容を含まない。」だけで、エラー情報（受付失敗の応答、送信失敗の理由、反映の失敗の理由など）に回答の入力内容を含めないことは書かれていない（下流の `domains/form.md:35` は扱っている）
  > - 代案: `form-operation.md` の「共通の規則」に「CMS が示すエラーと失敗の理由は、回答の入力内容と連絡先を含まない」を加える

#### 残る反例 04

- 条件: 1
- 効く範囲: scenario の content-editing（EDIT-17）、pages の P10・P08・P11・P13
- 出どころ: 前からあったもの
- 読み手の引用:

  > **P10 の版の削除の確認から、EDIT-17 の確認の項目が落ちている**
  > - 上流 `spec/scenario/content-editing.md:425`「削除した版は、確認、比較、復元ができなくなること」。同 427 行「対象の版が使用しているメディアと、必須の参照または本文のリンクで参照しているコンテンツ」
  > - 下流 `spec/pages/index.md:331`「対象の版ごとの操作の種類、実行者、日時、…、対象の版が使用するメディアと参照するコンテンツ、削除の後に残る使用・参照を確認で示す」。「確認、比較、復元ができなくなること」が無く、参照の範囲も「必須の参照または本文のリンク」から「参照するコンテンツ」に変わっている。P08・P11・P13 は版を対象に加えたときに「P10 の版の削除の確認と同じ内容」を示すので、同じ欠落を引き継ぐ
  > - 代案: P10 の当該の機能を「確認の内容は EDIT-17 の手順 3 に従う」に置き換える（条件 7 の 1 と同じ直し方）

#### 残る反例 05

- 条件: 4
- 効く範囲: scenario の app-integration・hosting・workspace-admin と共通の規則（index）
- 出どころ: 前からあったもの
- 読み手の引用:

  > **表示アプリへの通知の秘密情報に語が三つあり、「通知設定」が二つのものを指す**
  > - `spec/scenario/app-integration.md:307`「表示アプリは、通知設定の秘密情報で、通知が CMS から届いたものであることを確かめる」
  > - `spec/scenario/hosting.md:97`「表示アプリへの通知の宛先の秘密情報」
  > - `spec/scenario/index.md:62`「秘密情報（認証情報、通知設定の秘密情報）」
  > - 一方で `spec/scenario/index.md:20`「通知設定の権限 | 通知先、通知定義、送信接続設定の変更、…」は、フォームの通知の設定を「通知設定」と呼ぶ。「通知設定の秘密情報」が表示アプリへの通知の秘密情報なのか、通知設定の権限が扱う送信接続設定の認証情報なのかが、語から決まらない
  > - 代案: 表示アプリへの通知の秘密情報を「表示アプリへの通知の秘密情報」の一語にそろえ、app-integration.md 307 行、index.md 62 行、workspace-admin.md 324 行を置き換える

#### 残る反例 08

- 条件: 7
- 効く範囲: scenario の content-editing・publishing・form-operation、pages の P09・P10・P17
- 出どころ: 前からあったもの
- 読み手の引用:

  > **確認の内容が scenario と pages の 2 か所に別々に書かれている**
  > - 例: EDIT-17 の確認（`spec/scenario/content-editing.md:423-428`）と P10 の版の削除の確認（`spec/pages/index.md:331`）。PUB-03 の確認（`spec/scenario/publishing.md:96-101`）と P09 の公開取り消し（`spec/pages/index.md:297`）。FORM-08 の確認（`spec/scenario/form-operation.md:226-230`）と P17 の削除（`spec/pages/index.md:488`）。一方で P19 は「メンバーの利用を解除する。確認の内容は WS-03 に従う」（`spec/pages/index.md:533`）と参照で書く
  > - 既に食い違いが出ている（条件 1 の 4）。scenario の確認の項目を一つ足すと、pages の書き写しも直す必要があり、改訂が 2 層に広がる
  > - 代案: pages の確認の内容は、P19 と同じく「確認の内容は <シナリオ> の手順 N に従う」とし、scenario だけが列挙する。条件 4 の 2（版を残す規則を EDIT-13 と各操作の 2 か所に書いている）も同じ構造で、同じ直し方で消える

### flows と共通の型

#### 残る反例 09

- 条件: 2
- 効く範囲: Common（testcases/common）、Form、Webhook
- 出どころ: 層の移行が生んだもの
- 読み手の引用:

  > 共通の規則「確認した回」を、ユースケーステストで展開している（層の表の testcases の「書かない」の「共通の規則を展開した行」に当たる）。
  > - 規則の定義元（共通の規則）: `spec/domains/common.md` 153 行「確認した回。手動再試行と対処の記録は、メンバーが対象を選んだときに確認した送信の回（確認した回）を添える。……その送信は、手動再試行では、先に始まった一件の再試行として扱い、二通目を送らず、受け付けない対象に数えない。対処の記録では、送信失敗でない送信として受け付けない」
  > - 共通の規則の代表を置くファイルが、展開先を名指ししている: `spec/testcases/common/FailureHandling.md` 3 行「確認した回の扱いは、手動再試行と対処の記録のユースケースのテストが持つ」
  > - 展開した行: `spec/testcases/form/retryNotifications.md` 11 行（`TC-form-retryNotifications-07`）、`spec/testcases/webhook/requestDeliveryRetry.md` 11 行（`TC-webhook-requestDeliveryRetry-06`）、`spec/testcases/webhook/markDeliveryFailureHandled.md` 10 行（`TC-webhook-markDeliveryFailureHandled-05`）。`spec/index.md` 64 行は「各ユースケーステストは、この展開を持たない」と定める
  > - 代案: 代表の 1 行を `testcases/common/FailureHandling.md` に置き、ユースケーステストからは外す。回の型（Form の `attempt`、Webhook の `retryRound`）ごとに例が要るなら、`common.md` の規則を「各ドメインの `requestRetry`・`markHandled` が定める」に移し、規則を共通から外す

### Content

#### 残る反例 13

- 条件: 1
- 効く範囲: Content
- 出どころ: 前からあったもの
- 読み手の引用:

  > **参照先のモデルの不適合（`referenceMismatch`）の規則に、代表のテストケースが無い。**
  > - 上流: `spec/domains/content.md:1079`「コンテンツ参照 | 参照先が `targets` にあり、その `modelId` が、項目の `models` のどのモデルの ID でもない | 型を満たさない入力エラー（`referenceMismatch`）」。シナリオの異常系 `spec/scenario/content-editing.md:180`「**許可していないモデル**: スキーマの反映で参照を許可するモデルが縮小され、保存済みの参照先が許可の外になった場合、編集画面は入力エラーとして示す。」。契約 C5.3-06「型を満たさない入力エラー（必須項目の欠落、値の種類・選択肢・参照先の不適合）のある下書きは返さず」
  > - 落としている下流: `spec/testcases/content/` のどのファイルにも、`referenceMismatch` が生じる例（許可の外のモデルを参照する値の保存、許可するモデルを縮小する移行の後の下書きの入力エラー、その下書きの公開の拒否）が無い。`spec/testcases/schema/commitSchemaApply.md:30`（TC-schema-commitSchemaApply-24）は「下書きの参照の値は変わらない」だけを確かめ、入力エラーを確かめない。`referenceMismatch` が現れるのは `spec/testcases/ports/contentRepository.md` の前提条件だけ
  > - 代案: `saveDraft.md` か `ContentMigration.migrateContent.md` に、許可の外のモデルのコンテンツを指す参照が `referenceMismatch` の入力エラーとして下書きに残る行を一つ置く

#### 残る反例 14

- 条件: 7
- 効く範囲: Content
- 出どころ: 前からあったもの
- 読み手の引用:

  > **単一モデルと公開内容の参照の条件が、不変条件とポートの拒否の表に別々に書かれている。**
  > - `spec/domains/content.md:239`「単一モデルは、通常の管理対象にコンテンツを一つまで持つ。ごみ箱のコンテンツは数えない」と、`spec/domains/content.md:1974`「`SINGLE_EXISTS` | `cardinality` が `single` のコンテンツが、同じ `modelId` の通常の管理対象に二つ以上ある。ごみ箱のコンテンツは数えない」
  > - `spec/domains/content.md:234`「公開内容の保護する参照は、同じワークスペースの、通常の管理対象にある公開中のコンテンツを指す」と、`spec/domains/content.md:1976`「`REFERENCE_NOT_PUBLISHED` | 公開内容が、保護する参照の記録で、通常の管理対象の公開中のコンテンツでないものを参照する」
  > - 同じ表の `UNIQUE_VALUE_TAKEN`（`spec/domains/content.md:1975`「一意な値が重複する（「Content」の「不変条件」）」）は不変条件を参照するだけなので、三つの行で書き方が割れている。次に「ごみ箱のコンテンツも数える」「ごみ箱の参照先を許す」のような要求が来ると、二か所を直す必要があり、片方だけを直すと事前の確認と確定の時点の拒否が食い違う
  > - 代案: 二つの行も「「Content」の「不変条件」の単一モデルの条件に反する」「「Content」の「不変条件」の公開内容の保護する参照の条件に反する」と、不変条件を参照する形にする

### Schema

#### 残る反例 16

- 条件: 2
- 効く範囲: Schema
- 出どころ: 前からあったもの
- 読み手の引用:

  > ドメインが定める事柄をユースケースが書き写している（層の表の usecases の「書かない」にある「ドメインの規則の書き写し」に当たる）。
  > - `spec/usecases/schema.md:377`「新しい ID は、`WorkspaceSchema.requiredIds` が返した個数の `ModelId` と `FormId` と、確定の操作の ID 一つを `IdGenerator.generate` で作り、`NewIds` として `commit` に渡す（…）。`SchemaVersionRecord.ofCommit` には、`commit` に渡した準備済みのジョブと、`commit` が返した `WorkspaceSchema` を渡す。」
  > - 同じ内容の定義元は `spec/domains/schema.md:392`「`NewIds` は、呼び出し側が、`requiredIds` が返した個数の `ModelId` と `FormId` と、確定の操作の ID（`operationId`）を一つ作って `commit` に渡す。」と `spec/domains/schema.md:540`「`ofCommit` は、`commit` に渡した準備済みのジョブ（`job`）と、`commit` が返した `WorkspaceSchema`（`committed`）から記録を作る。」
  > - 同じ形がもう一か所ある。`spec/usecases/schema.md:489`「適用中の反映は、読んだ `WorkspaceSchema` の `state.jobId` のジョブを `findById` で読み、`WorkspaceSchema.inProgressApply` に渡して求める。」は、`spec/domains/schema.md:145`「`job` は、`ws.state.jobId` のジョブを読んだ結果（無ければ `null`）。」と同じことを述べている。
  > - 代案: 呼び出し側の手順は usecases にだけ残す。ドメインには「ID は振る舞いの外で作る」と引数の意味だけを書く。または逆に、usecases の文を削ってドメインの節を参照する。

#### 残る反例 17

- 条件: 7
- 効く範囲: Schema
- 出どころ: 前からあったもの
- 読み手の引用:

  > 確定でほかのドメインへ渡す値を、usecases の二か所に別々に書いている。この節が唯一の定義元だと宣言しているのに、確定の節がもう一度定義している。
  > - `spec/usecases/schema.md:116`「受け渡しは、このファイルのユースケースが行い、対応は、この節だけが定める。」
  > - `spec/usecases/schema.md:127`「| Schema → Content | ジョブの ID | `ContentStagingArea.promote` の `jobId` |」と `:128`「| Schema → Content | ジョブの ID、確定の操作の ID（`NewIds.operationId`）、ジョブの `actor`、反映後の版、反映後の `WorkspaceSchema.modelIds` | `PublicationLedgerRepository.appendStaged` の `jobId`、`MigrationRecordHeader`（操作の ID、実行者、スキーマの版）、`modelIds`。…」
  > - `spec/usecases/schema.md:388`「`ContentStagingArea.promote` には、ジョブの ID を渡す。`PublicationLedgerRepository.appendStaged` には、ジョブの ID と、`MigrationRecordHeader`（`NewIds.operationId`、ジョブの `actor`、反映後の版）と、`commit` が返した `WorkspaceSchema` の `modelIds` を渡す。」
  > - 次に来そうな要求として、記録のヘッダーに項目が一つ増える場合を考える。この改訂では二か所を直す必要があり、片方を直し忘れると食い違う。
  > - 代案: 388 行目の引数の列挙を削り、「渡す値は『ほかのドメインとの対応』の表に従う」とだけ書く。

#### 残る反例 18

- 条件: 7
- 効く範囲: Schema
- 出どころ: 前からあったもの
- 読み手の引用:

  > 止まった反映を検出するときの読み方の引数を、同じファイルの二か所に書いている。
  > - `spec/usecases/schema.md:96`「| 止まった反映・事前確認を検出して失敗させる | `ApplyStall.before(now)` | `active` |」と `:99`「対象は、`SchemaApplyJobRepository.findInProgress` を、`limit` を `SchemaLimits.inProgressLimit` にして、…読む。」
  > - `spec/usecases/schema.md:457`「`SchemaApplyJobRepository.findInProgress`（`progressedBefore` は `ApplyStall.before(now)`、`limit` は `SchemaLimits.inProgressLimit`）…」
  > - 代案: 457 行目は「『進行中のジョブの中断』のポート」とだけ書いて引数を省く。

### Form

#### 残る反例 20

- 条件: 1
- 効く範囲: Form
- 出どころ: 前からあったもの
- 読み手の引用:

  > `TemplateText` のエスケープと `format` の往復の規則に、代表のテストケースが無い。
  > - 上流（domains の規則）: `spec/domains/form.md:963`「文章の中の `\{{` は `{{` を、`\\` は `\` を表す。ほかの `\` はそのままの文字とする」、`spec/domains/form.md:969`「`format` は記法の文字列を返す。文章の `\` を `\\` に、`{{` を `\{{` にする。`format` の結果を解析すると、元の並びと等しい値になる」
  > - 下流: `spec/testcases/form/changeNotificationTemplate.md:11`（TC-09）は空の件名と記法の誤りしか扱わない。`spec/testcases/form/getFormSettings.md:5`（TC-01）は「件名と本文は記法の文字列」と書くだけで、文字どおりの `{{` を含む通知定義の保存・取得・描画を示す行がどこにも無い（`spec/testcases/` 全体を検索して確認した）。
  > - 代案: `changeNotificationTemplate` か `getFormSettings` に、本文に `\{{` を含む通知定義を保存し、取得した記法の文字列と描画結果がどうなるかを示す行を 1 行加える。

#### 残る反例 21

- 条件: 1
- 効く範囲: Form
- 出どころ: 前からあったもの
- 読み手の引用:

  > 件名に描画した値の改行を空白に置き換える規則に、代表のテストケースが無い。
  > - 上流: `spec/domains/form.md:1535`「件名は、描画した値に含まれる改行を空白一つに置き換える」
  > - 下流: `spec/testcases/form/sendSubmissionNotification.md:12`（TC-02）は本文の描画だけを扱う。改行を含むテキストの項目を件名で参照する行は、`sendSubmissionNotification.md` にも `previewNotificationTemplate.md` にも無い。
  > - 代案: `sendSubmissionNotification` に、改行を含むテキストの値を件名で参照する回答の行を 1 行加える。

#### 残る反例 22

- 条件: 2
- 効く範囲: Form
- 出どころ: 前からあったもの
- 読み手の引用:

  > ユースケースの文書が、ドメインの不変条件を書き写している。
  > - 記述: `spec/usecases/form.md:331`「回答一件は通知を一つ持つ。回答の一覧と詳細は一組で、入力内容と通知の送信結果を返す。」
  > - 定義元: `spec/domains/form.md:488`「回答は、同じ `SubmissionId` の `SubmissionNotification` を一つ持つ」
  > - 層の表の usecases の「書かない」のうち「ドメインの規則の書き写し」に当たる。同じルールを 2 か所で定めることにもなる（条件 7 の形）。
  > - 代案: 1 文目を削り、`domains/form.md` の Submission の「不変条件」を参照する。

#### 残る反例 23

- 条件: 4
- 効く範囲: Form
- 出どころ: 層の移行が生んだもの
- 読み手の引用:

  > テストケースが、ドメインの定めていないエラーの添付を期待している。
  > - `spec/testcases/form/addRecipient.md:11`（TC-11）「`BusinessRuleError`（`RecipientLimitReached`）。上限を添える」
  > - `spec/domains/form.md:890`「既に `max` 件あれば `BusinessRuleError`（`RecipientLimitReached`）」と、`spec/domains/form.md:1398`「`RecipientLimitReached` | `BusinessRuleError` | 通知先が `RecipientList.max` 件を超える」は、上限を添えることを定めない。同じ形のエラーでも、Webhook は `spec/domains/webhook.md:663` で「上限を添える」と明記している。上流の `spec/scenario/form-operation.md:65` は「上限を示す」。
  > - 代案: `form.md` のエラーコードの表と `RecipientList.add` に「上限（`RecipientList.max`）を添える」と書く。または TC-11 の「上限を添える」を削る。

#### 残る反例 24

- 条件: 4
- 効く範囲: Form
- 出どころ: 前からあったもの
- 読み手の引用:

  > `Change` の定義が、値オブジェクトでの使い方と食い違う。
  > - `spec/domains/form.md:223`「`changed` の `value` は、変更を反映して `version` を 1 進めた集約で、`updatedAt` を持つ集約では `updatedAt` を `now` にする」
  > - `spec/domains/form.md:880-881`「`changeAddress(list: RecipientList, from: EmailAddress, to: EmailAddress): Change<RecipientList>;` / `setEnabled(...): Change<RecipientList>;`」。`RecipientList` は `version` を持たない値オブジェクト。
  > - 代案: 223 行を「集約では `version` を 1 進めた集約、値オブジェクトでは変更後の値」と書き分ける。

#### 残る反例 25

- 条件: 4
- 効く範囲: Form
- 出どころ: 直しが生んだもの
- 読み手の引用:

  > 「権限による出し分け」の表で、同じ値を 2 つの行が別の実行者に割り当てている。
  > - `spec/usecases/form.md:52`「…回答の `SubmissionId`、受付日時、`FormId`、受付時のフォームのキーと表示名称 | ユースケースの認可を通った実行者」
  > - `spec/usecases/form.md:57`「受付時のフォーム（`Submission.form`）、入力内容、… | `submission:read`」。`Submission.form`（`ReceivedForm`）は `key` と `label` を含む（`spec/domains/form.md:1024-1028`）。
  > - 表は「返す値は、この表だけが定める」（`spec/usecases/form.md:48`）としているのに、キーと表示名称を返す相手が 2 通りに読める。テスト（`spec/testcases/form/getSubmission.md:11`、TC-07）はキーと表示名称を返し、受付時のフォームを返さない読み方を採っている。
  > - 代案: 57 行を「受付時のフォームの項目と選択肢（`Submission.form.fields`）」に絞る。

### Webhook と入口

#### 残る反例 26

- 条件: 2
- 効く範囲: Common（NotFoundTarget）、Webhook
- 出どころ: 前からあったもの
- 読み手の引用:

  > `NotFoundTarget` の `change_notification` は、どのユースケースからも返らない。上流にも下流にも、この値を返す場面が無い。
  > - `spec/domains/common.md:821`「`| "webhook_endpoint" | "change_notification"`」
  > - 通知を引くユースケースは `NotFoundError` を返さず、未作成を結果で返す。`spec/usecases/webhook.md:207`「**出力**: 通知が未作成（`notCreated`）、または、宛先ごとの番号（`WebhookDelivery.endpointOrder`）と送信結果の列」
  > - 組を指定する操作は、読めない組を受け付けない理由として返す。`spec/domains/webhook.md:592`「`refusals: readonly Readonly<{ ref: DeliveryRef; reason: "notFound" | "notFailed" | "endpointDeleted" }>[]`」と `spec/domains/webhook.md:601`（`markHandled` の `reason: "notFound" | "notFailed"`）
  > - `spec/` の全層を `change_notification` で検索すると、ほかに当たるのは `WebhookClosure.lostData` の種類（`spec/domains/webhook.md:916`、`:925`）だけだった。これは `DomainKind` の値で、`NotFoundTarget` ではない
  > - 代案: `NotFoundTarget` から `"change_notification"` を削る。

### Identity と Workspace

#### 残る反例 29

- 条件: 2
- 効く範囲: Identity、Common（testcases/common）
- 出どころ: 層の移行が生んだもの
- 読み手の引用:

  > 同じ規則の代表ケースが、共通のテストとユースケースのテストの両方にある（層の表の testcases の「書かない」の「共通の規則を展開した行」）
  > - `spec/testcases/common/conflictHandling.md:12`: 「TC-common-conflictHandling-03 | アカウント A に `pending` の回復依頼がある | A のメールアドレスで、アクセス回復の依頼を二つ並行して行う（Identity の `requestAccessRecovery`）。一方の確定が `ConflictError` で拒否される | どちらも `requested` を返す。エラーにしない」
  > - `spec/testcases/identity/requestAccessRecovery.md:17`: 「TC-identity-requestAccessRecovery-10 | A に `pending` の回復依頼が無い。A のメールアドレスでの二つの依頼が並行し、一方が先に確定する | もう一方を確定する | どちらも `requested` を返す。エラーにしない。…」
  > - 規則は `spec/domains/index.md:365` の「競合の扱い」の「アカウントの有無を明かさない要求」の一行で、当たるユースケースは `requestAccessRecovery` だけ。同じユースケースの同じ場面を二つのファイルが代表として持つ
  > - 代案: requestAccessRecovery-10 からは「エラーにしない」を除き、`requestAccessRecovery` に固有の結果（`pending` が一つだけ残ること、後の依頼がメールを送らないこと）だけを残す。または common 側を requestAccessRecovery-10 への参照にする

#### 残る反例 32

- 条件: 4
- 効く範囲: Identity、Common（domains/index.md のエラーの種類）
- 出どころ: 前からあったもの
- 読み手の引用:

  > ログイン済みの本人確認の失敗に使う `BusinessRuleError` が、エラーの種類の定義に当たらない
  > - `spec/domains/identity.md:334`: 「ログイン済みの本人確認（`MfaEnrollment`）の照合の失敗は、`BusinessRuleError` を投げる」。`spec/domains/identity.md:580`: 「`BusinessRuleError` | `PasswordVerificationFailed` / `CodeVerificationFailed` | ログイン済みの本人確認の失敗」
  > - `spec/domains/index.md:267`: 「`BusinessRuleError` | 集約の状態に依る規則の違反、利用者の入力に由来しない値の不変条件の違反、値オブジェクトの生成では確かめない、入力の定義の間の整合の違反（…）」
  > - パスワードと確認コードの照合の失敗は、三つのどれにも当たらない。presentation は種類から応答を決める（`spec/domains/index.md:263`）ので、種類の定義と使い方が別の Issue で食い違う
  > - 代案: index.md の `BusinessRuleError` の意味に「ログイン済みの本人確認の照合の失敗（再ログインを求めない）」を加える

#### 残る反例 33

- 条件: 7
- 効く範囲: Workspace
- 出どころ: 前からあったもの
- 読み手の引用:

  > メンバーの招待と初期所有者の招待が互いを妨げない規則を、三か所が別々に書いている
  > - `spec/domains/workspace.md:308`: 「メンバーの招待と初期所有者の招待は、互いを妨げず、互いを失効させない。同じ宛先に、`pending` のメンバーの招待と `pending` の初期所有者の招待が、一つずつあり得る。…」
  > - `spec/domains/workspace.md:931`: `issueInitialOwner` の「宛先は、前の招待と違ってよく、既にメンバーである人でも、メンバーでない人でも、`pending` のメンバーの招待の宛先でもよい」
  > - `spec/domains/workspace.md:1135`: `InvitationRepository.insert` の「同じ宛先の、`pending` のメンバーの招待と `pending` の初期所有者の招待は、どちらの順の `insert` も成功する」
  > - 次の要求で「初期所有者の招待の発行は、同じ宛先のメンバーの招待を失効させる」に変わると、三か所と、それぞれの例のテストケース（`TC-workspace-inviteMember-13`、`TC-ports-invitationRepository-14`〜`16`）を揃えて直す
  > - 代案: 規則は「Invitation」の不変条件だけに置き、`issueInitialOwner` とポートの契約は、その不変条件への参照と、自分が担保する範囲（ポートは「種類ごとの一意性」）だけを書く

### Token と Audit

#### 残る反例 34

- 条件: 1
- 効く範囲: Token
- 出どころ: 前からあったもの
- 読み手の引用:

  > ドメインの規則「最終利用日時を 60 秒以内なら記録しない」に、ユースケーステストの例が無い。
  > - 上流: `spec/domains/token.md:250`「`shouldRecordUse` | 最終利用日時（「TokenRepository」の `recordUse`）が `null`、または `now` が `lastUsedAt` の 60 秒後以降なら真」。配線は `spec/usecases/token.md:228`「`Token.shouldRecordUse`（`findByValueDigest` が添えた `lastUsedAt` を渡す）、`TokenRepository.recordUse`」
  > - 落ちている下流: `spec/testcases/token/authenticateToken.md`。記録する側の例（`:5`「最終利用日時は未記録 … 最終利用日時を要求の時刻で記録する」）だけがあり、記録済みの日時から 60 秒より前の要求で記録しない例が無い。WS-07 の入れ替えの手順 3（最終利用日時が進むことを確かめる）の結果を左右する
  > - 代案: `authenticateToken.md` に「最終利用日時が 30 秒前のトークンを示す → 最終利用日時を変えない」の一行を足す

#### 残る反例 35

- 条件: 1
- 効く範囲: Audit、Token
- 出どころ: 前からあったもの
- 読み手の引用:

  > 操作履歴の振る舞い「変更前後が等しい項目を `changes` から除く」に、どの層のテストケースにも例が無い。
  > - 上流: `spec/domains/audit.md:61`「`record` | … `changes` の項目のうち、`ValueChange` の変更前後の値が等しい項目を除く」
  > - 落ちている下流: 範囲のテストケースで複数の項目を持つ `changes` を確かめる行は `spec/testcases/token/changeBrowserTokenRestrictions.md:5`（三つの項目 `origins`・`usageLimitMax`・`usageLimitWindowMs` がすべて変わる例）だけ。一部の項目だけが変わる操作の例が無い（`spec/usecases/token.md:147` は三つの項目を必ず作る）
  > - 代案: `changeBrowserTokenRestrictions.md` に「利用元だけを変える → `changes` は `origins` だけを持つ」の一行を足す

#### 残る反例 37

- 条件: 1
- 効く範囲: Token
- 出どころ: 前からあったもの
- 読み手の引用:

  > スクリプトの認証の依頼のポートの契約「照合値の種類は二つ」に、ポート適合テストの例が足りない。
  > - 上流: `spec/domains/token.md:1385`「`insert` | 書き込み | 照合値の種類は、承認の値と受け取りの値の二つ」と、`spec/domains/common.md:594`「照合値は、同じ区画の、同じ種類の集約の、同じ種類の照合値の中で一意。違う種類の照合値どうしは重なってよい」
  > - 落ちている下流: `spec/testcases/ports/deviceAuthorizationRepository.md:26-31`。同じ種類の照合値の重複（TC-03、TC-04）と `userCode`（TC-06）の例だけがあり、ある依頼の承認の値の照合値と、別の依頼の受け取りの値の照合値が同じ場合に確定する例が無い。二つの照合値を一つの一意性の制約で持つアダプターは、このファイルのケースをすべて通る
  > - 代案: 「D の承認の値の照合値と同じ値を受け取りの値の照合値に持つ F を `insert` する → 確定する」の一行を足す

#### 残る反例 40

- 条件: 4
- 効く範囲: Audit
- 出どころ: 前からあったもの
- 読み手の引用:

  > Audit の理由コードの列挙が、Audit のユースケースが返すコードを落としている。
  > - `spec/domains/audit.md:272`「Audit が返す理由のコードを、すべて列挙する。共通のコード（`ALREADY_EXISTS`、`WORKSPACE_NOT_ACTIVE`、`PERMISSION_MISSING`、`ACTOR_NOT_ALLOWED`、`InvalidPagination`、`INVALID_QUERY`）は common.md にある」
  > - `spec/domains/index.md:338`「4. 結果が `mfa_setup_required` なら `ForbiddenError`（理由 `MFA_SETUP_REQUIRED`）」。`spec/usecases/audit.md:17` と `:28` の実行者は `member` で、この手続きを通る。Token（`spec/domains/token.md:1220`）と Workspace（`spec/domains/workspace.md:993`）の列挙は `MFA_SETUP_REQUIRED` を含む
  > - 代案: `audit.md:272` の括弧に `MFA_SETUP_REQUIRED` を加える

### Installation

#### 残る反例 42

- 条件: 1
- 効く範囲: Installation、scenario の hosting
- 出どころ: 前からあったもの
- 読み手の引用:

  > 対応しない実行環境を CMS が示すことが、技術設計に降りていない。
  > - 上流: `spec/scenario/hosting.md:62`「**資源の不足**: 対応しない実行環境や、必要な資源を満たさない環境では、CMS は起動時または動作確認で不足する資源を示す。」
  > - 下流: `spec/domains/installation.md:1047`「対応する実行環境は、文書が示す。設定の確かめ方は `InstallationEnvironment.inspectSettings` と `probe`、示し方は `BootReadiness` と `StatusAssessment.setupCheck`」。`BootReadiness`（`spec/domains/installation.md:730-736`）が見るのは、設定の五項目（`inspectSettings`）と `ResourceKind` の三つの資源（`readResource`）だけ。実行環境が対応しているかどうかを判定する振る舞いもポートも無く、`spec/testcases/installation/bootInstallation.md` と `runSetupCheck.md` にも、その行が無い。
  > - 代案: scenario の該当文を「必要な資源を満たさない環境では」に絞り、対応環境は導入手順の文書だけが示すことにする。CMS に判定させるなら、`InstallationEnvironment` に実行環境を確かめるメソッドを持たせ、`BootIssue` に種類を足す。

#### 残る反例 44

- 条件: 4
- 効く範囲: Installation
- 出どころ: 前からあったもの
- 読み手の引用:

  > 「更新の進行」が二つの意味で使われている。
  > - `spec/domains/installation.md:23`「Update in progress | 更新の進行 | `Installation` の `update` の `updating`（「Installation」）」。用語集は、`updating` の変種だけを「更新の進行」と定める。
  > - `spec/domains/installation.md:288`「`update` | 必須。更新の進行。`idle` は、更新を始めていないか、完了したこと。`updating` は、…」。フィールドの表は、`idle` と `updating` の両方を取るフィールドの全体を「更新の進行」と呼ぶ。
  > - テストケースも、フィールドの全体の意味で使う。`spec/testcases/installation/updateInstallation.md:5`「更新の進行は `idle`」、`spec/testcases/installation/takeBackup.md:6`「更新の進行は `idle`」、`spec/testcases/installation/checkInstallationAvailability.md:15`「更新の進行は `idle`」。
  > - 代案: 用語集の行を「更新の進行 | `Installation` の `update`（`idle` / `updating`）」に直す。`updating` の変種には、既にある「更新中の設置」（22 行）を使う。

#### 残る反例 45

- 条件: 6
- 効く範囲: Installation、Content、Common（domains/index.md の依存方向）
- 出どころ: 前からあったもの
- 読み手の引用:

  > `BackupStore` の契約と Content の `EditingPresenceStore` の契約が互いを参照し、`index.md` が宣言する依存方向に無い依存（Content ↔ Installation）を作っている。
  > - 宣言: `spec/domains/index.md:30`「`Installation → Workspace, Media（BackupStore の契約）`」、33 行「Installation は、ポート `BackupStore` の契約だけが、Workspace（写す名簿）と Media（メディアの実体の置き換えの手順）に依る」。Content → Installation の矢印は無い（25-29 行）。依存の定義は 22 行「依存する側のポートの契約が、依存される側の集約の状態とポートの契約に依ること」。
  > - Content → Installation: `spec/domains/content.md:2154`（`EditingPresenceStore` の契約）「バックアップの範囲は、[installation.md](domains/installation.md) の「BackupStore」の契約が定める。復旧の後に残った記録は、失効で消える」。
  > - Installation → Content: `spec/domains/installation.md:1167`「区画に結び付かないポートが保つ記録（利用量の制限の記録、コンテンツの編集中の記録）は含まない。`capture` は、これらの記録を写さない。`restore` は、これらの記録を読まず、変えない。復旧の後も、復旧の前と同じ内容を読める」。ポート適合テストも Content のポートで確かめる（`spec/testcases/ports/backupStore.md:109`、TC-65「`restore(B1)` を呼び、`EditingPresenceStore` で読む」）。
  > - 代案: `BackupStore` の契約は、個々のポートの名前を挙げずに、`index.md` の「集約とポートの置き場所」の「区画に結び付かない」ポートの記録を含まないと定める。`content.md:2154` の参照は消す（範囲の外にあることは置き場所の表から決まる）。

#### 残る反例 46

- 条件: 7
- 効く範囲: Installation、Common
- 出どころ: 前からあったもの
- 読み手の引用:

  > 「利用量の制限の記録はバックアップと復旧の対象の外にある」という規則が、2 か所に別々に定義されている。
  > - `spec/domains/common.md:696`（`UsageLimiter`）「記録は期間の終わりまで保てば足りる。…記録は、バックアップと復旧の対象の外にある」。同 698 行「記録を保つ期間と、バックアップの対象にしないことは、アダプターごとの検証の対象で…」。
  > - `spec/domains/installation.md:1167`「区画に結び付かないポートが保つ記録（利用量の制限の記録、コンテンツの編集中の記録）は含まない。`capture` は、これらの記録を写さない。`restore` は、これらの記録を読まず、変えない」。
  > - 次に来そうな要求（例: 利用量の制限の記録も復旧で戻す）では、両方を直す必要があり、片方だけ直すと食い違う。
  > - 代案: 規則は `BackupStore` の契約だけに置き、`common.md:696` の該当文は消すか `installation.md` の「BackupStore」への参照にする。

## 申し送り

条件 3・4・6・7 の、1 か所にしか効かない問い。受け手は実装する側で、効く箇所を含む Issue・フェーズで決める。決めた項目はここから消し、spec に効く決定なら spec に書く。

| ドメイン | 申し送り |
| --- | --- |
| Content | 6 |
| Schema | 7 |
| Form | 7 |
| Webhook | 2 |
| Identity | 2 |
| Workspace | 4 |
| Token | 2 |
| Audit | 1 |
| Media | 6 |
| Installation | 4 |
| 計 | 41 |

### 体験設計の層

#### 申し送り 01

- 問い: 版の復元で、最新の内容が選んだ非推奨の選択肢の値を、版の値に戻すか、最新の値のまま保つか
- 効く 1 か所: `spec/scenario/content-editing.md` の EDIT-14 の手順 4（`restoreRevisionAsDraft`）
- 読み手の引用:

  > **版の復元で、非推奨の選択肢の値をどう扱うか**（条件 3）
  > - 問い: 最新の内容が非推奨の選択肢を選び、版が非推奨でない選択肢を選んでいる場合、復元は版の値（非推奨でない選択肢）に戻すか、最新の値のまま保つか
  > - 効く場所: EDIT-14 の手順 4（`restoreRevisionAsDraft`）
  > - 引用: `spec/scenario/content-editing.md:347`「非推奨の項目・選択肢・ブロックの値は、最新の内容にある値のまま保ち、版の値で変えない。」。契約の層との対応（`spec/review/contract.md:423`）は、非推奨の選択肢から非推奨でない選択肢へ変えることを「新たに使わない」操作としている

#### 申し送り 02

- 問い: ログインの試行の制限の「15 分」は、固定の区切りか、直近 15 分の移動窓か
- 効く 1 か所: `spec/scenario/app-integration.md` の APP-12 の制限の表の、AUTH-02 の「試行の制限」の行（ログインのユースケース）
- 読み手の引用:

  > **ログインの試行の制限の窓**（条件 3）
  > - 問い: 「15 分あたり 5 回」は、固定の 15 分の区切りか、直近 15 分の移動窓か。「その 15 分」はいつ終わるか
  > - 効く場所: AUTH-02 の「試行の制限」（ログインのユースケース）
  > - 引用: `spec/scenario/app-integration.md:383`「15 分あたり 5 回の失敗で、その 15 分が過ぎるまで受け付けない」

#### 申し送り 03

- 問い: 公開・公開取り消し・ごみ箱への移動・完全削除を、編集の画面（P09）に置き続けるか
- 効く 1 か所: `spec/pages/index.md` の P09
- 読み手の引用:

  > **P09 が編集と公開状態の変更の二つを負う**（条件 7。画面の分け方はドメインの境界を変えない）
  > - 問い: 公開・公開取り消し・ごみ箱への移動・完全削除を、編集の画面に置き続けるか
  > - 効く場所: P09
  > - 引用: `spec/pages/index.md:19`「一件のコンテンツの内容を確認・編集し、公開状態を変える」

#### 申し送り 04

- 問い: 通知設定の権限の内容と受付管理の権限の内容を、一画面（P15）に置き続けるか
- 効く 1 か所: `spec/pages/index.md` の P15
- 読み手の引用:

  > **P15 が通知の設定と受付・保持の設定の二つを、権限で出し分けて負う**（条件 7。画面の分け方はドメインの境界を変えない）
  > - 問い: 通知設定の権限の内容と、受付管理の権限の内容を一画面に置き続けるか
  > - 効く場所: P15
  > - 引用: `spec/pages/index.md:432-437` の権限ごとの表

### flows と共通の型

#### 申し送り 05

- 問い: 止まった派生形式の生成のやり直しの一回の件数を、一覧の 1 ページの件数（`Pagination.maxLimit`）と同じにするか、Media の定数を持つか
- 効く 1 か所: `spec/usecases/media.md` の「止まった派生形式の生成をやり直す（`restartStalledDerivatives`）」のトランザクション境界
- 読み手の引用:

  > 一覧の 1 ページの件数の定数を、定期ジョブの読み取りの件数に使っている（条件 7）。APP-12 の 1 ページの件数を変えると、止まった派生形式の生成のやり直しの一回の件数も変わる。効く 1 か所は `spec/usecases/media.md` の「止まった派生形式の生成をやり直す（`restartStalledDerivatives`）」のトランザクション境界。
  > - `spec/domains/common.md` 178 行「`Pagination.maxLimit` は、一覧の 1 ページの件数の上限。値は……APP-12 の「一覧の 1 ページの件数」が定め、技術設計の層では、この定数だけが持つ」
  > - `spec/usecases/media.md` 363 行「`findStalled` は、UnitOfWork を開く前に、`limit` に `Pagination.maxLimit` を渡して、一度の起動で一度だけ呼ぶ」
  > - 問い: このジョブの件数は、ページの件数と同じでよいか。Form の `FormSweep.maxLimit` のように、Media の定数を持つか

#### 申し送り 06

- 問い: 管理画面からのコンテンツの作成で、送り直しに使う ID をクライアントが作るか、サーバー側の入口が要求ごとに作るか
- 効く 1 か所: 管理画面のコンテンツの作成の入口（作成のフォームと server function）。spec では `spec/usecases/content.md` の「コンテンツを作成する（`createContent`）」の入力
- 読み手の引用:

  > 管理画面からのコンテンツの作成で、ID を誰が作るかが決まっていない（条件 6）。`AGENTS.md` の「Idempotent create」は「The client keeps the id across a failed attempt — a fresh id per submit would defeat it」とする。効く 1 か所は、管理画面の作成の入口（作成のフォームと、その server function）。
  > - `spec/domains/common.md` 546 行「コンテンツID の指定が無い要求では、入口が `generate` で作って渡す」
  > - `spec/usecases/content.md` 187 行「送り直しでは、同じ ID を渡す」
  > - 問い: 管理画面のクライアントが ID を作って送り直しに使うのか、サーバー側の入口が要求ごとに作るのか。後者なら、応答を受け取れなかった作成の送り直しで、コンテンツが二件できる

### Content

#### 申し送り 07

- 問い: 並行した保持方針の変更で、後の要求を拒否するか
- 効く 1 か所: `spec/testcases/content/changeRetentionPolicy.md` の TC-content-changeRetentionPolicy-04
- 読み手の引用:

  > **並行した保持方針の変更で、後の要求を拒否するか。**（効く場所: `spec/testcases/content/changeRetentionPolicy.md` の TC-content-changeRetentionPolicy-04）
  > - 引用: 「二人のメンバーが、保持方針の変更を並行して行う | 先に確定した変更だけが保存される。他方は `ConflictError`（`STALE_VERSION`）を返す」
  > - `changeRetentionPolicy` の入力は `version` を持たない（`spec/usecases/content.md:608`「**入力**: 履歴を保持する日数、ごみ箱のコンテンツを保持する日数」）。後の要求が先の確定の後に保持方針を読めば、両方が確定して記録が二件残る。期待結果は割り込みの順に依り、一つに決まらない。WS-10 は確認の後の変化を挙げていない（`spec/scenario/index.md:59`「各操作が挙げない内容（件数など）は、確認の時点の値として示し、その後の変化は確定を妨げない」）

#### 申し送り 08

- 問い: `PublicationLedger.record` は `changes` を並べ替えるのか、並びを確かめるのか
- 効く 1 か所: `spec/domains/content.md` の「PublicationLedger」
- 読み手の引用:

  > **`PublicationLedger.record` は `changes` を並べ替えるのか、並びを確かめるのか。**（効く場所: `spec/domains/content.md` の「PublicationLedger」）
  > - 引用: `spec/domains/content.md:462`「`changes` | その操作の変更の一覧。コンテンツID の昇順に並ぶ」、`spec/domains/content.md:467`「`changes` は、その操作が返した `PublicationChange` ごとの `PublicationChange.toNotice` の値」
  > - ユースケースの対象の列は入力の順を保つ（`BatchTargets` は「残した要素の順を保つ」）ので、`record` が並べ替えなければ、並びの約束が成り立たない。`record` の実装の中で決めれば、ほかへは広がらない

#### 申し送り 09

- 問い: ごみ箱からの復元が別の復元と競り合ったとき、`ContentNotInTrash` と `STALE_VERSION` のどちらで示すか
- 効く 1 か所: `spec/testcases/content/restoreTrashedContent.md` の TC-content-restoreTrashedContent-14
- 読み手の引用:

  > **ごみ箱からの復元が、別の復元と競り合ったときの示し方。**（効く場所: `spec/testcases/content/restoreTrashedContent.md` の TC-content-restoreTrashedContent-14）
  > - 引用: 「C の復元が C を読んだ後、確定までに、ほかのメンバーによる C の復元が確定した | `ConflictError`（`STALE_VERSION`）を返す」
  > - シナリオは同じ状況を「他のメンバーが先に復元していた場合、CMS は通常の管理対象にあることを示す」（`spec/scenario/publishing.md:208`）と定め、確認を挟まない操作なので「確認を示し直す」流れが無い。読む時点の差で `ContentNotInTrash`（TC-content-restoreTrashedContent-05）と `STALE_VERSION` に分かれる結果を、画面でどう示すかは、この一行の中で決まる

### Schema

#### 申し送り 10

- 問い: TC-schema-prepareSchemaApply-06 の「同じ値の種類の変更」が指す操作は何か
- 効く 1 か所: `spec/testcases/schema/prepareSchemaApply.md` の TC-schema-prepareSchemaApply-06
- 読み手の引用:

  > 条件 4。前の行を指す「同じ」が、言葉どおりに読むと、ドメインの規則では起こらない操作を指してしまう。効く箇所は `spec/testcases/schema/prepareSchemaApply.md` の TC-schema-prepareSchemaApply-06。
  > - 引用: `prepareSchemaApply.md:12`「反映。同じ値の種類の変更。変換できない文字列を持つのは、下書き、ごみ箱のコンテンツ、履歴の版だけ」
  > - ドメインでは、同じ値の種類への変更は移行定義の不整合で、準備まで進まない（`spec/domains/schema.md:1650`「同じ値の種類への値の種類の変更は、移行定義の不整合」）。
  > - 「上と同じ、テキストから数値への値の種類の変更」と書けば曖昧さが消える。

#### 申し送り 11

- 問い: フォームの項目の書式を、定義に無い `email` ではなく `constraints.format` で書くか
- 効く 1 か所: `spec/testcases/schema/requestDirectSchemaApply.md` の TC-schema-requestDirectSchemaApply-18・19
- 読み手の引用:

  > 条件 4。テストケースが、スキーマのフォームの項目に無いフィールド名を使っている。効く箇所は `spec/testcases/schema/requestDirectSchemaApply.md` の TC-18・19。
  > - 引用: `requestDirectSchemaApply.md:24`「`email` が真のテキストの項目 reply」「reply の `email` を偽にしたスキーマ」
  > - `FormFieldType` の `text` は `email` を持たず、書式は `constraints.format` で表す（`spec/domains/schema.md:1019`）。`email` は Form の `ReceivedField` の値で、`spec/domains/form.md` の「ReceivedForm」が `constraints.format` から写す。
  > - 「`constraints.format` が `email` の項目」と書けば、定義に沿う。

#### 申し送り 12

- 問い: 適用中の反映の「進行」は `Preparation.progress` の結果か、ジョブの `phase` か
- 効く 1 か所: `spec/usecases/schema.md` の `getMigrationStatus`
- 読み手の引用:

  > 条件 4。テストケースが期待する出力が、ユースケースの使う振る舞いに無い。効く箇所は `getMigrationStatus`。
  > - 引用: `spec/testcases/schema/getMigrationStatus.md:14`「適用中の反映のジョブ（ジョブの ID J、実行者と受け付けた時点の名称、要求の日時、`status`、進行）を返す」
  > - 一方、`spec/usecases/schema.md:486-487` の「使用するドメインの振る舞い・ポート」は `Preparation.progress` を挙げていない。「進行」が `Preparation.progress` の結果を指すのか、ジョブの `phase` を指すのかが決まらない。

#### 申し送り 13

- 問い: 手元のスキーマに不整合があるとき、`checkMigrations` が食い違いを求めるか
- 効く 1 か所: `spec/usecases/schema.md` の `checkMigrations`
- 読み手の引用:

  > 条件 3。手元のスキーマ自体に不整合があるとき、`checkMigrations` が食い違いを求めるかどうかが決まらない。効く箇所は `checkMigrations`。
  > - 引用: `spec/usecases/schema.md:721-724`。使う振る舞いは `MigrationSequence.issues`、`StructureMigration.fold`、`Schema.structureOf`、`ContentStructure.diff`、`StructureMigration.lossyOperations` だけで、`SchemaValidation.validate` が無い。
  > - これに対し `generateMigration`（`:696`）と `computeSchemaFingerprints`（`:674`）は、先に `SchemaValidation.validate` を呼ぶ。
  > - キーが重複したスキーマを `Schema.structureOf` に渡したときの結果は定めていない。

#### 申し送り 14

- 問い: 移行定義に不整合があるとき、`checkMigrations` がデータを失い得る操作を返すか
- 効く 1 か所: `spec/usecases/schema.md` の `checkMigrations`
- 読み手の引用:

  > 条件 3。移行定義に不整合があるとき、`checkMigrations` がデータを失い得る操作を返すかどうかが決まらない。効く箇所は `checkMigrations`。
  > - 引用: `spec/testcases/schema/checkMigrations.md:12`「食い違う定義は空（求めない）」、`:18` も同じ。
  > - データを失い得る操作の扱いは、どの行にも書いていない。

#### 申し送り 15

- 問い: 確定の UnitOfWork が確定せず「反映・事前確認を失敗させる」を呼んだとき、`commitSchemaApply` が何を返すか
- 効く 1 か所: `spec/usecases/schema.md` の `commitSchemaApply`
- 読み手の引用:

  > 条件 3。確定の UnitOfWork が確定せず、「反映・事前確認を失敗させる」を呼んだ場合に、`commitSchemaApply` が何を返すかが決まらない。効く箇所は `commitSchemaApply`。
  > - 引用: `spec/usecases/schema.md:358`「**出力**: 成功で終えたジョブ（`SucceededJob`）、または引いたジョブ」、`:410`「`CommitObstacle.failureOf` が失敗の内容を返せば、『反映・事前確認を失敗させる』を別の UnitOfWork で呼ぶ」
  > - TC-schema-commitSchemaApply-35 から 38、40 は、ジョブの状態だけを書いている。

#### 申し送り 16

- 問い: `blocking` の除外条件を、取り除く操作の列挙でなく、すべての単位の後の構造に値の位置が残るかで定めるか
- 効く 1 か所: `spec/domains/schema.md` の「ValueMigration」
- 読み手の引用:

  > 条件 7。`blocking` の除外条件を、取り除く操作の種類を列挙して定めている。効く箇所は `ValueMigration`。
  > - 引用: `spec/domains/schema.md:1748`「後続の操作で取り除かれた場合（項目、その項目を持つ項目のまとまり・繰り返しの要素・ブロックの種類、モデルの削除）は、`blocking` に加えない」
  > - 代案は「すべての単位を終えた後の構造に、その値の位置が残る場合だけ加える」の一文にすること。取り除く操作が増えても、この規則は直さずに済む。ドメインの境界と集約の分け方は変わらない。

### Form

#### 申し送り 17

- 問い: 読み直した時点で止まっていない通知を、止まった通知の検出が送るか
- 効く 1 か所: `spec/usecases/form.md` の「止まった通知を検出する」のトランザクション境界
- 読み手の引用:

  > 止まった通知を読み直した時点で止まっていない通知（`findStalled` の後に再試行が始まり、新しい `queuedAt` の `waiting` になった通知）を送るか。
  > - 効く場所: `spec/usecases/form.md` の「止まった通知を検出する」のトランザクション境界
  > - 引用: `spec/usecases/form.md:459`「`findStalled` が返した通知ごとに、通知を読み、共通の「通知の送信」を、その通知の `attempt` で行う」。`start` は `waiting` なら止まっているかどうかによらず `send` を返す（`spec/domains/form.md:715`）。楽観ロックがあるので二重送信にはならないが、止まっていない通知まで定期ジョブが送るかどうかが決まっていない。

#### 申し送り 18

- 問い: 既に無い回答が複数あるとき、`NotFoundError` が添える ID は何件か
- 効く 1 か所: `spec/testcases/form/previewRetryRecipients.md` の TC-form-previewRetryRecipients-09
- 読み手の引用:

  > `previewRetryRecipients` で、既に無い回答が複数あるときに `NotFoundError` が添える ID は何件か。
  > - 効く場所: `spec/testcases/form/previewRetryRecipients.md`
  > - 引用: `spec/testcases/form/previewRetryRecipients.md:11`（TC-09）「`NotFoundError`。S9 の ID を添える」。`retryNotifications` は無い回答をすべて `notFound` として返す（`spec/domains/form.md:794`）。

#### 申し送り 19

- 問い: 通知の送信で `MailConnectionRepository.find` が `null` を返したときの扱い
- 効く 1 か所: `spec/usecases/form.md` の共通の「通知の送信」
- 読み手の引用:

  > 共通の「通知の送信」で `MailConnectionRepository.find` が `null` を返した場合の扱い。
  > - 効く場所: `spec/usecases/form.md` の共通の「通知の送信」
  > - 引用: `spec/usecases/form.md:71`「その時点の `MailConnection` を読み、`Submission.outgoingMail` で作ったメールを、`MailSender.send` で…送る」。不変条件の上では起きないはずだが、`null` を前提にしない理由がどこにも書かれていない。

#### 申し送り 20

- 問い: UnitOfWork の前と中で読んだ送信接続設定が違うとき、更新を拒否するか
- 効く 1 か所: `spec/usecases/form.md` の「送信接続設定を更新する」のトランザクション境界
- 読み手の引用:

  > `updateMailConnection` で、UnitOfWork の前に読んだ設定と UnitOfWork の中で読み直した設定が違う場合、更新を拒否するか。
  > - 効く場所: `spec/usecases/form.md` の「送信接続設定を更新する」のトランザクション境界
  > - 引用: `spec/usecases/form.md:110`「入力の検証と認証情報の保護は、UnitOfWork の前に、`PartitionReader` で読んだ設定で行う。UnitOfWork の中で設定を読み直して `update` に渡す」。読み直した `expectedVersion` で `save` するので、並行した更新を黙って上書きする。

#### 申し送り 21

- 問い: 利用再開前の処理で、読んだ通知が `waiting` / `sending` でない、または `null` のときの扱い
- 効く 1 か所: `spec/domains/form.md` の「利用再開前の処理」
- 読み手の引用:

  > 利用再開前の処理で、`findById` で読んだ通知が `waiting` / `sending` でない場合、または `null` の場合の扱い。
  > - 効く場所: `spec/domains/form.md` の「利用再開前の処理」（Installation のユースケースが呼ぶ）
  > - 引用: `spec/domains/form.md:1990`「返った通知ごとに、…`SubmissionNotification.abandon(notification, settings, now)` の結果を `save` で保存する」。`abandon` は `SubmissionNotification<Waiting | Sending>` だけを受ける（`spec/domains/form.md:596-600`）。

#### 申し送り 22

- 問い: 確認した回が今の回と違う通知の対処の記録を、`notFailed` と区別する理由（`attemptChanged` など）で返すか
- 効く 1 か所: `spec/domains/form.md` の「対処の記録」
- 読み手の引用:

  > 確認した回が今の回と違う通知の対処の記録を、理由 `notFailed` で返していること（条件 7。代案はドメインの境界を変えない）。
  > - 効く場所: `spec/domains/form.md` の「対処の記録」
  > - 引用: `spec/domains/form.md:806`「`notification.attempt` が `confirmed` と等しくない | `refused`。理由は `notFailed`」。`spec/testcases/form/markFailuresHandled.md:10`（TC-09）は、再び送信失敗になった通知を `notFailed` と返す。再試行には `alreadyRequested` があり、対処の記録にだけ区別が無い。「再試行が始まっていた」と示す要求が来ると、`HandlingDecision` と P17 の表示の両方を直すことになる。代案は理由に `attemptChanged` を加えること。

### Webhook と入口

#### 申し送り 23

- 問い: 試行の種類と手動再試行の回数の、あり得ない組み合わせを型で排除するか
- 効く 1 か所: `spec/domains/webhook.md` の「WebhookDelivery」のフィールドの型
- 読み手の引用:

  > 条件 7: 試行の種類と手動再試行の回数の、あり得ない組み合わせを型が排除していない。`first` と `automatic` の試行は `retryRound` が 0 の送信にしか現れず、`manual` は 1 以上の送信にしか現れない。それでも型は、`retryRound: 2` の `first` を表せる。
  > - 効く場所: `spec/domains/webhook.md`「WebhookDelivery」のフィールドの型
  > - 引用: `spec/domains/webhook.md:258-267`「`type WebhookDelivery<S extends DeliveryState = DeliveryState> = Readonly<{ … state: S; retryRound: number; … }>`」。`:275`「`manual` は、一回だけの手動再試行で、何回目の手動再試行かは送信の `retryRound` が持つ」。`:364`「送信を `pending` にし、`retryRound` を 1 増やして…`attempt` は `manual`」
  > - 代案は、`Pending` の変種に回の範囲を結び付けるもの（例: `first` と `automatic` は `round: 0` に限る）。集約の分け方を変えないので、申し送りにした。

#### 申し送り 24

- 問い: P28 の「再試行の結果」の状態を、どのユースケースで読むか
- 効く 1 か所: `spec/pages/index.md` の P28
- 読み手の引用:

  > 条件 3: P28 の「再試行の結果」の状態を、どのユースケースで読むかが決まっていない。`requestDeliveryRetry` は、受け付けた時点の状態（送信待ち）しか返さない。終わった後の組ごとの結果は、`listDeliveries` を読み直すことでしか得られない。選んだ組は、別々のページにあり得る。
  > - 効く場所: P28 の実装（`spec/pages/index.md`「P28 表示アプリへの通知の状況」）
  > - 引用: `spec/pages/index.md:622`「再試行の結果: 再試行が終わったとき。組ごとの結果を示す。再び送信失敗になった組には、新しい理由を示す」。`spec/usecases/webhook.md:334`「受け付けた（組ごとの `requested` または `alreadyRequested` と、その組の今の送信結果）」

### Identity と Workspace

#### 申し送り 25

- 問い: `completeAccessRecovery` で、`AccessRecovery.admit` に渡す回復依頼とアカウントを UnitOfWork の前と中のどちらで読むか
- 効く 1 か所: `spec/usecases/identity.md` の `completeAccessRecovery` のトランザクション境界
- 読み手の引用:

  > `completeAccessRecovery` で、`AccessRecovery.admit` に渡す回復依頼とアカウントを UnitOfWork の前と中のどちらで読むか（条件 3）
  > - 効く場所: `spec/usecases/identity.md` の `completeAccessRecovery` のトランザクション境界
  > - 引用（`spec/usecases/identity.md:135`）: 「一つの UnitOfWork（`recoveryRequestRepository` と `accountRepository` の読み取り、両方の `save`、…）。`PasswordHasher.hash` と `UsageLimiter.check` は、UnitOfWork の前に呼ぶ」。`admit` は「照合値の計算の前に呼ぶ」（`spec/domains/identity.md:519`）ので、`hash` の前に読み取りが要る。読み直す場合に `complete` へ渡す `admission.reset.account` と `save` の `expectedVersion` がどちらの読み取りのものかが決まっていない

#### 申し送り 26

- 問い: 新しいアカウントで受諾する場合に、事前の確認が参加の囲いを読まず `JoiningGuard.ticket(null)` を使うか
- 効く 1 か所: `spec/usecases/workspace.md` の `acceptInvitation` の「事前の確認」
- 読み手の引用:

  > 新しいアカウントで受諾する場合の、事前の確認の参加の囲いの読み取り（条件 3）
  > - 効く場所: `spec/usecases/workspace.md` の `acceptInvitation` の「事前の確認」
  > - 引用（`spec/usecases/workspace.md:132`）: 「`JoiningGuardRepository.findById`（そのアカウントの参加の囲い）、`JoiningGuard.ticket`」。アカウントを作る場合、ID は前段の `IdGenerator.generate` で決まり、事前の確認の時点で読む対象が無い。`JoiningGuard.ticket(null)` の 0 を使うことの明示が無い

#### 申し送り 27

- 問い: 所属の索引の修復で、`withdraw` の除外と取り下げの確定の `confirm` を、ページの UnitOfWork にまとめるか、組ごとにするか
- 効く 1 か所: `spec/usecases/workspace.md` の `repairMembershipIndex`
- 読み手の引用:

  > 定期ジョブの修復で、`withdraw` の項目の除外と、取り下げの確定で `Member` を読んだ組の `confirm` を、ページの `confirm`・`remove` の UnitOfWork にまとめるか、組ごとの UnitOfWork にするか。確定しなかった項目の件数の数え方もこれに依る（条件 3）
  > - 効く場所: `spec/usecases/workspace.md` の `repairMembershipIndex`
  > - 引用（`spec/usecases/workspace.md:256`）: 「`confirm` の組と `remove` の項目は、ページごとに、設置の区画の一つの UnitOfWork（書き込みは 2 回以下）。`withdraw` は「共通」の「所属の索引の修復」」。`spec/domains/workspace.md:732`〜`733` は、取り下げの確定と項目の除外を、それぞれ一つの UnitOfWork とする

#### 申し送り 28

- 問い: `JoiningGuard` の `generation` を持たず、`version` から求めるか
- 効く 1 か所: `spec/domains/workspace.md` の「JoiningGuard」
- 読み手の引用:

  > `JoiningGuard` の `generation` と `version` は、どちらも生成時の値が決まっていて `advance` だけで 1 ずつ進むので、常に `generation = version + 1` になる。成り立たない組を型が許す（条件 7。代案は、`ticket` を `version` から求めて `generation` を持たない形。ドメインの境界と集約の分け方を変えない）
  > - 効く場所: `spec/domains/workspace.md` の「JoiningGuard」
  > - 引用（`spec/domains/workspace.md:338`〜`339`）: 「`generation` | 参加の世代。生成時は 1。`advance` で 1 増える」「`version` | 生成時は 0。`advance` で 1 増える」

#### 申し送り 29

- 問い: 最後に利用したワークスペースを、`MembershipIndex` から別のポートに分けるか
- 効く 1 か所: `spec/domains/workspace.md` の「MembershipIndex」
- 読み手の引用:

  > `MembershipIndex` が、所属の索引の項目と、アカウントごとの最後に利用したワークスペースの二つを保存する（条件 7。代案は、最後に利用したワークスペースを別のポートに分ける形。ドメインの境界と集約の分け方を変えない）
  > - 効く場所: `spec/domains/workspace.md` の「MembershipIndex」
  > - 引用（`spec/domains/workspace.md:1214`）: 「アカウントとワークスペースの組ごとに、項目の状態（`joining` / `joined`）と世代を持ち、アカウントごとに、最後に利用したワークスペースを持つ」

### Token と Audit

#### 申し送り 30

- 問い: 操作履歴の対象が現在もあるかの判定を、Audit のドメインの関数として定めるか
- 効く 1 か所: `spec/usecases/audit.md` の `getAuditLogEntry`
- 読み手の引用:

  > 操作履歴の対象が「現在もある」かの判定を、ドメインが持たない（条件 6、ビジネスロジックの配置）。
  > - 問い: 対象の種類ごとの判定（トークンは `standing` が `active` のときだけ、招待は期限内で受諾も取り消しもされていないときだけ、フォームは現行スキーマに同じ `FormId` があるときだけ「ある」）を、Audit のドメインの関数として一文で定めるか
  > - 効く 1 か所: `spec/usecases/audit.md` の `getAuditLogEntry`
  > - 引用: `spec/domains/audit.md:231`「対象が現在もあるかは、…`getAuditLogEntry` が、対象の種類ごとに、対象を所有するドメインの読み取りで求める。種類ごとの条件は、ユースケーステスト（…getAuditLogEntry.md）が定める」、`spec/usecases/audit.md:31`「対象が現在もあるかは、`operation.target` の種類ごとに、対象を所有するドメインの次の読み取りと判定で求める」

#### 申し送り 31

- 問い: 現行スキーマに無い紐付け済みの要素だけを持つトークンを、一覧で「範囲が空」と示すか
- 効く 1 か所: `spec/usecases/token.md` の `listTokens`
- 読み手の引用:

  > 現行スキーマに無い紐付け済みの要素だけを持つトークンの「範囲が空」の示し方（条件 3）。
  > - 問い: 反映の確定と並行した発行で、削除したモデルの要素を持って残ったトークン（`spec/testcases/token/issueToken.md:16` の後者）は、`TokenScope.isEmpty` が偽で、示す要素も無い。一覧で「範囲が空」と示すか
  > - 効く 1 か所: `spec/usecases/token.md` の `listTokens`
  > - 引用: `spec/domains/token.md:267`「現行スキーマに無い要素は、トークンの削除まで範囲に残り、どの対象にも当たらず、範囲を示すときに含めない」、`:701`「`isEmpty` は、`selection` で要素が無ければ真」、`spec/usecases/token.md:82`「`TokenScope.isEmpty`」、`spec/scenario/workspace-admin.md:219`「範囲に選んだモデル・フォームがすべて削除されたトークンは、どの対象も操作できない。CMS は一覧で範囲が空であることを示す」

#### 申し送り 32

- 問い: 承認済みの依頼の確認が示す「承認の時点のワークスペース」を、何から読み、どの値で示すか
- 効く 1 か所: `spec/usecases/token.md` の `getDeviceAuthorization`
- 読み手の引用:

  > 承認済みの依頼の確認が示す「承認の時点のワークスペース」の求め方（条件 3）。
  > - 問い: 名称を何から読み、どの値で示すかを決めるか（`resolveAccountWorkspaces` の列に `approval.workspaceId` があり、`MemberAccess` が `granted` のときだけ示す、など）。テストケースは結果を決めているが、ユースケースは読むものを挙げていない
  > - 効く 1 か所: `spec/usecases/token.md` の `getDeviceAuthorization`
  > - 引用: `spec/usecases/token.md:297`「…所属ワークスペースの列（…）、承認の時点のワークスペース（ID と名称）」、`spec/testcases/token/getDeviceAuthorization.md:13`「承認の後に、承認したメンバーが W の利用を解除された、または、W が多要素認証を必須にし、承認したアカウントは多要素認証を設定していない | … | 依頼の状態 `approved` を返す。承認の時点のワークスペースを示さない」

### Media

#### 申し送り 33

- 問い: 失敗した派生形式の「失敗の理由」を、どの振る舞いで求めるか
- 効く 1 か所: `spec/usecases/media.md` の「メディアを取得する」（`getMedia`）の「使用するドメインの振る舞い・ポート」
- 読み手の引用:

  > 失敗した派生形式の「失敗の理由」をどの振る舞いで求めるか。`statuses` は生成状況だけを返す。失敗の状態でも、定義が現行と異なれば `generating` として示すので、理由を示すかどうかがこの判定と絡む。効く 1 か所は `getMedia`（`spec/usecases/media.md`「メディアを取得する」の「使用するドメインの振る舞い・ポート」L168）。
  > - `spec/domains/media.md` L387-388「状態の定義…が、その定義から `key` を除いた値と異なる | `generating`」「それ以外 | 状態の `status`」
  > - `spec/testcases/media/getMedia.md` L5「`failed` の派生形式は失敗の理由を返し、生成済みとして返さない」
  > - `spec/pages/index.md` L395「派生形式の生成中・失敗: 派生形式が生成中または失敗のとき。失敗の理由を示す」

#### 申し送り 34

- 問い: `MediaName`・`MediaSearchInfo`・`MediaDescriptionText` の `create` は、前後の空白を除いた値を保持するか
- 効く 1 か所: `spec/domains/media.md` の「値オブジェクト」
- 読み手の引用:

  > `MediaName`、`MediaSearchInfo`、`MediaDescriptionText` の `create` は、前後の空白を除いた値を保持するのか、入力をそのまま保持するのか。検証は「前後の空白を除いて」数えるが、保持する値が書かれていない。`fromFileName` だけは空白を除くと明記している。効く 1 か所は Media の値オブジェクト（`spec/domains/media.md`「値オブジェクト」）で、`MediaProfile.describe` の「現在の値と等しければ」の判定と、`findPage` の部分一致はその結果に従う。
  > - `spec/domains/media.md` L496「`MediaName` | 前後の空白を除いて 1 文字以上 200 文字以下」
  > - 同 L527「`fromFileName` は、ファイル名の前後の空白を除き、先頭から 200 文字までを名称にする」

#### 申し送り 35

- 問い: 一覧の検索で大文字と小文字を区別しないとき、英字の外の文字も同一視するか
- 効く 1 か所: `spec/domains/media.md` の「MediaRepository」の `findPage`（アダプター）
- 読み手の引用:

  > 一覧の検索の「英字の大文字と小文字を区別しない」で、英字の外の文字（`Ä` と `ä` など）を同一視するかどうか。JavaScript の `toLowerCase` で書いた in-memory 実装は同一視するが、SQLite の `LIKE` は ASCII の外を同一視しない。効く 1 か所は `MediaRepository.findPage` のアダプター。
  > - `spec/domains/media.md` L838「前後の空白を除いた文字列が、`profile` の `name` または `searchInfo` に部分一致する。英字の大文字と小文字を区別しない」

#### 申し送り 36

- 問い: `schema.applied` の処理で `scan` に渡す `limit` の値
- 効く 1 か所: `spec/usecases/media.md` の `applyDerivativeDefinitions`
- 読み手の引用:

  > `schema.applied` の処理で、`scan` に渡す `limit` の値。`scan` は上限（`Pagination.maxLimit`）だけを定め、ユースケースは渡す値を書いていない。効く 1 か所は `applyDerivativeDefinitions`。
  > - `spec/usecases/media.md` L336「`scan` を `mediaId` の昇順に、返る件数が `limit` より少なくなるまで繰り返し」
  > - `spec/domains/media.md` L1216「`limit` の上限は `Pagination.maxLimit`」

#### 申し送り 37

- 問い: 一つの UnitOfWork で公開使用の記録と非公開化を両方書いたとき、どちらの理由で拒否するか
- 効く 1 か所: `spec/testcases/ports/mediaRepository.md` の TC-20 と `spec/testcases/ports/mediaUsageRepository.md` の TC-28・29（`MediaRepository` と `MediaUsageRepository` のアダプター）
- 読み手の引用:

  > 一つの UnitOfWork で、公開使用の記録とそのメディアの非公開化を両方書いた場合に、どちらの理由で拒否するか。ポート適合テストはどちらの理由も認めており、期待結果が一つに決まらない。両方を一つの UnitOfWork で書くユースケースは無いので、効く 1 か所は `MediaRepository` と `MediaUsageRepository` のアダプター。
  > - `spec/testcases/ports/mediaRepository.md` L56（TC-20）「`ConflictError` になる。理由は `MEDIA_IN_PUBLIC_USE` または `MEDIA_NOT_PUBLIC`」
  > - `spec/testcases/ports/mediaUsageRepository.md` L55-56（TC-28、TC-29）「理由は `MEDIA_NOT_PUBLIC` または `MEDIA_IN_PUBLIC_USE`」「理由は `MEDIA_NOT_FOUND` または `MEDIA_IN_USE`」

### Installation

#### 申し送り 38

- 問い: 並行した起動で負けた側の `ConflictError` の理由を `INSTALLATION_EXISTS` に決めるか
- 効く 1 か所: `spec/testcases/installation/bootInstallation.md` の TC-installation-bootInstallation-41
- 読み手の引用:

  > 並行した起動で負けた側の `ConflictError` の理由が、一つに決まらない。
  > - 効く 1 か所: `spec/testcases/installation/bootInstallation.md` の TC-41。
  > - 引用: `spec/testcases/installation/bootInstallation.md:37`「二つ目の起動は `ConflictError`（理由は問わない）を返し、何も書かない」。一方、`spec/domains/installation.md:687`「`INSTALLATION_EXISTS` | `ConflictError` | 並行した起動で、`Installation` が既にある」、同 345 行「並行した起動は、`InstallationRepository.insert` の一意性で、一方だけが `Installation` を作る」。最初の段の UnitOfWork はアカウントなどの `insert` も含むので、アダプターがどの書き込みで拒否するかで理由が変わり得る。期待結果を `INSTALLATION_EXISTS` に決めるか、エラー表の意味を「理由によらない」に合わせるかを、実装時に決める。

#### 申し送り 39

- 問い: 復旧の出力に、設置が利用再開前であることを含むか
- 効く 1 か所: `spec/usecases/installation.md` の `restoreInstallation` とそのテストケース
- 読み手の引用:

  > 「バックアップから設置全体を復旧する」の出力に、設置が利用再開前であることを含むかどうか。
  > - 効く 1 か所: `restoreInstallation`（ユースケースとそのテストケース）。
  > - 引用: `spec/usecases/installation.md:405`「**出力**: 復旧した（復旧点）」。`spec/testcases/installation/restoreInstallation.md:5`（TC-01）「復旧点（バックアップの取得日時）と、設置が利用再開前であることを返す」。

#### 申し送り 40

- 問い: `BootReadiness` の設定の規則を、`system_mail` の例外でなく、設定ごとの「宣言しなくてよい」で表すか
- 効く 1 か所: `spec/domains/installation.md` の「BootReadiness」
- 読み手の引用:

  > `BootReadiness` の設定の規則が、`system_mail` だけの例外で成り立っている。
  > - 効く 1 か所: `spec/domains/installation.md` の「BootReadiness」。
  > - 引用: `spec/domains/installation.md:732`「`not_declared` の `data_store`、`media_store`、`secret_key`、`public_url` は `setting_missing`。`not_declared` の `system_mail` は、外部メールを使えない設置として受け付ける」。設定ごとに「宣言しなくてよい」を持たせる構造にすれば、例外が消える。ドメインの境界と集約の分け方は変わらない。

#### 申し送り 41

- 問い: データの更新を、`InstallationEnvironment` から別のポートに分けるか
- 効く 1 か所: `spec/domains/installation.md` の「InstallationEnvironment」
- 読み手の引用:

  > `InstallationEnvironment` が、実行環境の読み取りと、保存済みのデータの更新という 2 つの責務を持つ。
  > - 効く 1 か所: `spec/domains/installation.md` の「InstallationEnvironment」。
  > - 引用: `spec/domains/installation.md:1263`「目的: 設置の実行環境から、容量、データの版、更新を持つ版、設置の公開の URL、復旧できる大きさの上限を取得し、保存先と保管先の動作を確かめ、保存済みのデータを更新する」。`updateData` だけが設置のデータを書き、「取得・置き換え・更新の排他」に加わる。データの更新を別のポートに分けても、ドメインの境界と集約の分け方は変わらない。
