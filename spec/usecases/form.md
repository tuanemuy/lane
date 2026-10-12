# Form のユースケース

フォームの受付、回答の保存、メール通知とその送信結果を扱うユースケースを定める。型、振る舞い、ポートは [domains/form.md](../domains/form.md)、ドメインをまたぐ規則は [domains/index.md](../domains/index.md)、共通の型とポートは [domains/common.md](../domains/common.md) が定める。各ユースケースの振る舞いの結果（結果の種類ごとの条件、エラー、途中で止まったときに残る状態）は、`spec/testcases/form/` の、見出しの識別子と同じ名前のファイルが定める。

スキーマの反映の確定に伴う `FormSettings` の作成・紐付け・削除済みへの遷移は Schema のユースケース、初期構成の読み込み、復旧後の利用の再開、ワークスペースの利用終了の消去、状態確認は Installation のユースケースが定める。どれも、[domains/form.md](../domains/form.md) が公開する振る舞い・手順・項目を使う。

## 共通

各ユースケースは、ここに加わる分だけを書く。

### 認可

- メンバーの認可の手続きとトークンの認可の手続きは、[domains/index.md](../domains/index.md) の「認可」が定める。各ユースケースの「認可」は、受け付ける実行者の種類と、必要な権限だけを書く
- 受け付ける実行者が `appToken` と `memberToken` のユースケースは、「フォームの送信を受け付ける」だけ
- 権限を複数挙げる読み取りのユースケースの出し分けは、共通の「権限による出し分け」が定める

### 書き込みの UnitOfWork

- Form の集約とポートは、すべてワークスペースの区画にある。書き込みを持つユースケースは、`UnitOfWorkProvider.run(scope, fn)` に、`{ kind: "workspace", workspaceId, condition }` の `WriteScope` を、UnitOfWork ごとに一つ渡す。条件（`WorkspaceCondition`）は、各ユースケースの「トランザクション境界」だけが書く
- `fn` は、`WorkspaceContext` のリポジトリを使う。UnitOfWork の外の読み取りは、`PartitionReader.workspace(workspaceId)` のリポジトリを使う。各ユースケースの「トランザクション境界」が挙げるリポジトリは、この区画のもの
- 件数を区切る置き換え（`replaceRetentionByForm`）は、件数を区切る削除と同じに、一つの UnitOfWork で一度だけ呼ぶ
- 件数に比例して書くユースケースは、「トランザクション境界」に書き込みの回数を書く

### フォームの定義

現行のフォームの定義は、`WorkspaceSchemaRepository.find` で読んだ `WorkspaceSchema` の `formIds` から `FormId` に対応するキーを求め、`Schema.form` で得る。`FormSettings` と `WorkspaceSchema` の両方を読むユースケースは、`WorkspaceSchema` を先に読む。読んだ設定と現行のフォームの対応は、[domains/form.md](../domains/form.md) の「現行のフォームとの対応」が定める。

### 操作履歴

操作履歴を残すユースケースは、[../domains/audit.md](../domains/audit.md) の「トランザクション境界」に従って追記する。実行者の名称は認可で読んだ `Account` から求める。`FormTarget` の名称は、[domains/form.md](../domains/form.md) の「削除済みフォームの表示」に従う。

`changes`（`AuditChange` の列）の項目は、操作の種類ごとに次の表のとおり。値は変更前と変更後の `ValueChange`。

| 操作の種類 | `field` | 値 |
| --- | --- | --- |
| `form.notification_definition_changed` | `subject`、`body` | 件名、本文。`TemplateText.format` の記法の文字列 |
| | `senderName` | 送信者の表示名。無ければ `null` |
| | `replyTo` | 返信先の項目のキー。無ければ `null` |
| `form.recipients_changed` | `recipients` | すべての通知先のメールアドレスの列。通知先の並びの順 |
| | `enabledRecipients` | 有効な通知先のメールアドレスの列。通知先の並びの順 |
| `form.retention_policy_changed` | `retentionDays` | 保持する日数 |
| `mail_connection.changed` | `destination` | 接続先。登録では変更前が `null` |
| | `conditions` | 接続条件の `名前=値` の文字列の列。名前の、文字列のコードポイントの昇順。登録では変更前が `null` |
| | `credentials` | 認証情報（`identity` と秘密）。登録と、認証情報を入力した更新で、変えた事実だけを `SecretChange` で渡す |

### 権限による出し分け

必要な権限を「または」で複数挙げる読み取りのユースケースは、出力の値を、次の表の権限を持つ実行者にだけ返す。持たない権限の値は、出力に含めない。値で絞り込む条件と検索は、その値の権限を必要とする。返す値は、この表だけが定める。各ユースケースの「出力」は、返すものの単位だけを書く。

| 値 | 返す実行者 |
| --- | --- |
| フォームの `FormId`、キー、表示名称、状態（有効か削除済みか）。紐付け待ちの設定のフォームのキー。回答の `SubmissionId`、受付日時、`FormId`、受付時のフォームのキーと表示名称 | ユースケースの認可を通った実行者 |
| 通知先（メールアドレスと有効・無効）、通知定義、不整合のある参照と理由、現行のフォームの項目ごとのキー・表示名称・参照に使える用途 | `notification:manage` |
| 通知定義の参照の不整合の有無 | `notification:manage` または `intake:manage` |
| 受付状態、回答の保持方針、有効な通知先の件数、受付を有効にするのに足りないもの | `intake:manage` |
| 通知の状態、通知の回、受付時の通知対象、宛先ごとの送信の状態と失敗の理由、通知の送信結果、未送信の内訳、送信失敗の通知の対処の状態。通知の条件での絞り込み | `notification:manage` |
| 受付時のフォーム（`Submission.form`）、入力内容、問い合わせ対象の送信時の情報と、そのコンテンツの現在の置き場所、一覧表示用の項目の値。検索 | `submission:read` |

### 通知の送信結果

出力に通知の送信結果を含めるユースケースは `SubmissionNotification.result` を、未送信の内訳を含めるユースケースは通知の `state.kind` と `attempt.kind` を返す。二軸の意味は、[domains/form.md](../domains/form.md) の「通知の送信結果」が定める。

### 通知の送信

「回答の通知を送る」と「止まった通知を検出する」が使う手続き。入力は `WorkspaceId`、`SubmissionId`、始める回（`NotificationAttempt`）。出力は `send` / `record` / `ignore` のどれか。判定は、[domains/form.md](../domains/form.md) の「通知の送信」（`SubmissionNotification.start`、`complete`）の結果の名前で指す。

- **使用するドメインの振る舞い・ポート**: `WorkspaceDirectory.findById`、`WorkspaceRepository.find`、`WorkspaceAvailability.of`（Workspace）、`SubmissionNotificationRepository.findById`、`SubmissionRepository.findById`、`FormSettingsRepository.findById`、`Clock.now`、`SubmissionNotification.start`、`SubmissionNotificationRepository.save`、`MailConnectionRepository.find`、`Submission.outgoingMail`、`MailSender.send`、`SubmissionNotification.complete`
- **トランザクション境界**: 二つの UnitOfWork と、その間の送信に分ける。どちらの UnitOfWork も、条件は `active` で、`submissionNotificationRepository` を書く。回答を書かない
  - ワークスペースが利用できるかどうかは、UnitOfWork を開く前に読んだ名簿の項目（`PartitionReader.installation()` の `WorkspaceDirectory.findById`）と、一つ目の UnitOfWork で読んだ `Workspace` から、`WorkspaceAvailability.of` で求める
  - 一つ目の UnitOfWork は、`Workspace`、通知、回答、回答のフォームの運用設定を読み、`start` を呼んで、`record` と `send` の結果の通知を保存する
  - UnitOfWork の外の送信は、`send` の結果の通知の確定の後に行う。その時点の `MailConnection` を読み、`Submission.outgoingMail` で作ったメールを、`MailSender.send` で、`start` が返した `recipients` へ送る
  - 二つ目の UnitOfWork は、通知を読み直し、始めた回と `MailSender.send` の結果を `complete` に渡して、`recorded` が真の結果の通知を保存する
  - 途中で止まったときに残る状態とその後の扱いは、[../testcases/form/sendSubmissionNotification.md](../testcases/form/sendSubmissionNotification.md) と [../testcases/form/detectStalledNotifications.md](../testcases/form/detectStalledNotifications.md) が定める

## 送信接続設定

### 送信接続設定を取得する（`getMailConnection`）

登録済みの送信接続設定を、認証情報を含めずに返す。

- **シナリオ**: FORM-01
- **認可**: 実行者は `member`。`notification:manage`
- **入力**: なし
- **出力**: `MailConnectionView`、または未登録
- **使用するドメインの振る舞い・ポート**: `MailConnectionRepository.find`、`MailConnection.view`
- **トランザクション境界**: UnitOfWork 不要。書き込みがない

### 送信接続設定を登録する（`registerMailConnection`）

ワークスペースの送信接続設定を登録する。

- **シナリオ**: FORM-01
- **認可**: 実行者は `member`。`notification:manage`
- **操作履歴**: `mail_connection.changed`
- **入力**: `MailConnectionInput`
- **出力**: `MailConnectionView`
- **使用するドメインの振る舞い・ポート**: `MailSender.settingRule`、`MailConnectionInput.validate`（`{ kind: "register" }`。規則を渡す）、`Confidential.reveal`、`SecretCipher.protect`、`ProtectedMailSecret.create`、`MailConnectionRepository.find`、`MailConnection.register`（読んだ登録済みの設定を `current` に渡す）、`MailConnectionRepository.insert`、`MailConnection.view`
- **トランザクション境界**: UnitOfWork 必要。条件は `active`。`mailConnectionRepository` と `auditLogRepository` を使う。`insert` と操作履歴の追記を一つの UnitOfWork で確定する。入力の検証と認証情報の保護は、UnitOfWork の前に行う

### 送信接続設定を更新する（`updateMailConnection`）

登録済みの送信接続設定を置き換える。

- **シナリオ**: FORM-01
- **認可**: 実行者は `member`。`notification:manage`
- **操作履歴**: `mail_connection.changed`
- **入力**: `MailConnectionInput`。`credentials` の `null` は、入力しなかったことを表す
- **出力**: `MailConnectionView`
- **使用するドメインの振る舞い・ポート**: `MailConnectionRepository.find`、`MailSender.settingRule`、`MailConnectionInput.validate`（UnitOfWork の前に読んだ設定を `current` にした `update`。規則を渡す）、`Confidential.reveal`、`SecretCipher.protect`、`ProtectedMailSecret.create`、`MailConnection.update`、`MailConnectionRepository.save`、`MailConnection.view`
- **トランザクション境界**: UnitOfWork 必要。条件は `active`。`mailConnectionRepository` と `auditLogRepository` を使う。`save` と操作履歴の追記を一つの UnitOfWork で確定する。入力の検証と認証情報の保護は、UnitOfWork の前に、`PartitionReader` で読んだ設定で行う。UnitOfWork の中で設定を読み直して `update` に渡す

### 送信接続設定の動作を確認する（`testMailConnection`）

入力中の設定で、指定した宛先へ試験メールを一通送る。試験メールの件名と本文は、このユースケースが持つ固定の文面で、`MailConnection.testMail` に渡す。

- **シナリオ**: FORM-01
- **認可**: 実行者は `member`。`notification:manage`
- **入力**: `MailConnectionInput`、試験メールの宛先。`credentials` が `null` の入力は、登録済みの認証情報を使う
- **出力**: `RecipientResult`（`accepted`、または `failed` と `DeliveryFailureReason`）
- **使用するドメインの振る舞い・ポート**: `MailConnectionRepository.find`、`MailSender.settingRule`、`MailConnectionInput.validate`（未登録なら `{ kind: "register" }`、登録済みなら、読んだ設定を `current` にした `update`。規則を渡す）、`RecipientAddressInput.validate`、`Confidential.reveal`、`SecretCipher.protect`、`ProtectedMailSecret.create`、`MailConnection.register`（`current` は `null`）または `MailConnection.update`（どちらも結果を保存しない）、`MailConnection.testMail`、`MailSender.send`
- **トランザクション境界**: UnitOfWork 不要。書き込みがない

### 送信接続設定の更新が影響するフォームを一覧する（`listFormsAffectedByMailConnection`）

送信接続設定の更新が影響するフォーム（`FormInventory.affectedByMailConnection`）を返す。

- **シナリオ**: FORM-01
- **認可**: 実行者は `member`。`notification:manage`
- **入力**: なし
- **出力**: 影響するフォームの一覧（`FormInventory.affectedByMailConnection` の結果と、各フォームのキーと表示名称）
- **使用するドメインの振る舞い・ポート**: `FormSettingsRepository.findAll`、`SubmissionNotificationRepository.summarizeByForm`、`FormInventory.affectedByMailConnection`、`FormSettings.isAccepting`、`WorkspaceSchemaRepository.find`
- **トランザクション境界**: UnitOfWork 不要。書き込みがない

## フォームの運用設定

### フォームの運用設定を一覧する（`listFormSettings`）

現行のフォーム、回答が残る削除済みフォーム（`FormInventory.remaining`）、紐付け待ちの設定の概況を返す。

- **シナリオ**: FORM-02, FORM-03, FORM-04, FORM-05, FORM-06, FORM-11, SMIG-13, HOST-03
- **認可**: 実行者は `member`。`notification:manage` または `intake:manage`
- **入力**: なし
- **出力**: フォームごとの概況と、紐付け待ちの設定ごとの宣言の内容
- **使用するドメインの振る舞い・ポート**: `FormSettingsRepository.findAll`、`PendingFormSettingsRepository.findAll`、`SubmissionNotificationRepository.summarizeByForm`、`FormInventory.remaining`、`WorkspaceSchemaRepository.find`、`FormSettings.referenceIssues`、`TemplateText.format`
- **トランザクション境界**: UnitOfWork 不要。書き込みがない

### フォームの運用設定を取得する（`getFormSettings`）

フォーム一つの運用設定を返す。

- **シナリオ**: FORM-02, FORM-03, FORM-04, FORM-05, FORM-06, FORM-11, SMIG-13
- **認可**: 実行者は `member`。`notification:manage` または `intake:manage`
- **入力**: `FormId`
- **出力**: フォーム一つの運用設定
- **使用するドメインの振る舞い・ポート**: `WorkspaceSchemaRepository.find`、`FormSettingsRepository.findById`、`MailConnectionRepository.find`、`FormSettings.enabledRecipients`、`FormSettings.missingForReception`、`FormSettings.referenceIssues`、`ReceivedForm.of`、`NotificationTemplate.usagesOf`、`TemplateText.format`
- **トランザクション境界**: UnitOfWork 不要。書き込みがない

### 通知先を追加する（`addRecipient`）

フォームに、有効な通知先を加える。

- **シナリオ**: FORM-02, FORM-11, SMIG-13
- **認可**: 実行者は `member`。`notification:manage`
- **操作履歴**: `form.recipients_changed`
- **入力**: `FormId`、メールアドレス
- **出力**: 変更後の通知先
- **使用するドメインの振る舞い・ポート**: `RecipientAddressInput.validate`、`FormSettingsRepository.findById`、`WorkspaceSchemaRepository.find`、`FormSettings.addRecipient`、`FormSettingsRepository.save`
- **トランザクション境界**: UnitOfWork 必要。条件は `active`。`formSettingsRepository` と `auditLogRepository` を使う。`save` と操作履歴の追記を一つの UnitOfWork で確定する

### 通知先のメールアドレスを修正する（`changeRecipientAddress`）

通知先のメールアドレスを置き換える。

- **シナリオ**: FORM-02, FORM-11
- **認可**: 実行者は `member`。`notification:manage`
- **操作履歴**: `form.recipients_changed`
- **入力**: `FormId`、修正前のメールアドレス、修正後のメールアドレス
- **出力**: 変更後の通知先
- **使用するドメインの振る舞い・ポート**: `RecipientAddressInput.validate`、`FormSettingsRepository.findById`、`WorkspaceSchemaRepository.find`、`FormSettings.changeRecipientAddress`、`FormSettingsRepository.save`
- **トランザクション境界**: 「通知先を追加する」と同じ

### 通知先を有効・無効にする（`setRecipientEnabled`）

通知先の有効・無効を変える。

- **シナリオ**: FORM-02, FORM-11
- **認可**: 実行者は `member`。`notification:manage`
- **操作履歴**: `form.recipients_changed`
- **入力**: `FormId`、メールアドレス、有効か無効か
- **出力**: 変更後の通知先
- **使用するドメインの振る舞い・ポート**: `RecipientAddressInput.validate`、`FormSettingsRepository.findById`、`WorkspaceSchemaRepository.find`、`FormSettings.setRecipientEnabled`、`FormSettingsRepository.save`
- **トランザクション境界**: 「通知先を追加する」と同じ

### 通知先を削除する（`removeRecipient`）

通知先を取り除く。

- **シナリオ**: FORM-02, FORM-11
- **認可**: 実行者は `member`。`notification:manage`
- **操作履歴**: `form.recipients_changed`
- **入力**: `FormId`、メールアドレス
- **出力**: 変更後の通知先
- **使用するドメインの振る舞い・ポート**: `RecipientAddressInput.validate`、`FormSettingsRepository.findById`、`WorkspaceSchemaRepository.find`、`FormSettings.removeRecipient`、`FormSettingsRepository.save`
- **トランザクション境界**: 「通知先を追加する」と同じ

### 通知定義を変更する（`changeNotificationTemplate`）

フォームの通知定義を、現行のフォームの定義で参照を確かめて置き換える。

- **シナリオ**: FORM-03, FORM-04, SMIG-13
- **認可**: 実行者は `member`。`notification:manage`
- **操作履歴**: `form.notification_definition_changed`
- **入力**: `FormId`、`NotificationTemplateInput`
- **出力**: 変更後の通知定義
- **使用するドメインの振る舞い・ポート**: `NotificationTemplateInput.validate`、`FormSettingsRepository.findById`、`WorkspaceSchemaRepository.find`、`FormSettings.requireActive`、`FormSettings.changeNotification`、`FormSettingsRepository.save`
- **トランザクション境界**: UnitOfWork 必要。条件は `schemaVersion`（参照の検証に使った `WorkspaceSchema` のスキーマの版）。`formSettingsRepository` と `auditLogRepository` を使う。`save` と操作履歴の追記を一つの UnitOfWork で確定する

### 通知定義を入力例で描画する（`previewNotificationTemplate`）

編集中の通知定義を、現行のフォームの入力例で描画した件名、本文、返信先を返す。

- **シナリオ**: FORM-03, FORM-04
- **認可**: 実行者は `member`。`notification:manage`
- **入力**: `FormId`、`NotificationTemplateInput`
- **出力**: `RenderedNotification`
- **使用するドメインの振る舞い・ポート**: `NotificationTemplateInput.validate`、`FormSettingsRepository.findById`、`WorkspaceSchemaRepository.find`、`FormSettings.requireActive`、`Clock.now`、`NotificationRendering.sampleSource`、`NotificationRendering.render`
- **トランザクション境界**: UnitOfWork 不要。書き込みがない

### 参照の不整合が残る通知定義を一覧する（`listBrokenNotificationTemplates`）

通知定義に参照の不整合が残る現行のフォームを返す。

- **シナリオ**: FORM-04
- **認可**: 実行者は `member`。`notification:manage`
- **入力**: なし
- **出力**: 不整合が残るフォームごとの、`FormId`、キー、表示名称
- **使用するドメインの振る舞い・ポート**: `FormSettingsRepository.findAll`、`WorkspaceSchemaRepository.find`、`FormSettings.referenceIssues`
- **トランザクション境界**: UnitOfWork 不要。書き込みがない

### フォームの受付を有効にする（`openReception`）

受付停止のフォームを受付中にする。

- **シナリオ**: FORM-05, SMIG-13
- **認可**: 実行者は `member`。`intake:manage`
- **入力**: `FormId`
- **出力**: 現在の受付状態
- **使用するドメインの振る舞い・ポート**: `FormSettingsRepository.findById`、`MailConnectionRepository.find`、`FormSettings.requireActive`、`FormSettings.open`、`FormSettingsRepository.save`
- **トランザクション境界**: UnitOfWork 必要。条件は `active`。`formSettingsRepository` と `mailConnectionRepository`（読み取り）を使う。`FormSettings.open` が `changed` を返した設定の `save` を確定する

### フォームの受付を停止する（`closeReception`）

受付中のフォームを受付停止にする。

- **シナリオ**: FORM-05
- **認可**: 実行者は `member`。`intake:manage`
- **入力**: `FormId`
- **出力**: 現在の受付状態
- **使用するドメインの振る舞い・ポート**: `FormSettingsRepository.findById`、`FormSettings.requireActive`、`FormSettings.close`、`FormSettingsRepository.save`
- **トランザクション境界**: UnitOfWork 必要。条件は `active`。`formSettingsRepository` を使う。`FormSettings.close` が `changed` を返した設定の `save` を確定する

### 回答の保持方針を変更する（`changeRetentionPolicy`）

フォームの回答の保持方針を置き換える。既存の回答への適用は「保持方針を既存の回答へ適用する」が、別の確定で行う。

- **シナリオ**: FORM-06, SMIG-13
- **認可**: 実行者は `member`。`intake:manage`
- **操作履歴**: `form.retention_policy_changed`
- **入力**: `FormId`、保持する日数
- **出力**: 変更後の保持方針
- **使用するドメインの振る舞い・ポート**: `RetentionInput.validate`、`FormSettingsRepository.findById`、`WorkspaceSchemaRepository.find`、`FormSettings.requireActive`、`FormSettings.changeRetention`、`FormSettingsRepository.save`
- **トランザクション境界**: UnitOfWork 必要。条件は `active`。`formSettingsRepository` と `auditLogRepository` を使う。`save` と操作履歴の追記を一つの UnitOfWork で確定する

### 保持方針の既存の回答への適用を確認する（`previewRetentionApplication`）

現在の保持方針、または保存の前の保持方針を、そのフォームの既存の回答すべてへ適用した場合の影響を返す。

- **シナリオ**: FORM-06
- **認可**: 実行者は `member`。`intake:manage`
- **入力**: `FormId`、保持する日数（保存の前の保持方針で確認する場合。無ければ現在の保持方針）
- **出力**: 確認した保持方針と `RetentionPreview`。メンバーは、ここで示した保持方針を、「保持方針を既存の回答へ適用する」に、確認した保持方針として渡す
- **使用するドメインの振る舞い・ポート**: `RetentionInput.validate`、`FormSettingsRepository.findById`、`FormSettings.requireActive`、`Clock.now`、`SubmissionRepository.previewRetention`
- **トランザクション境界**: UnitOfWork 不要。書き込みがない

### 保持方針を既存の回答へ適用する（`applyRetentionToSubmissions`）

そのフォームの既存の回答の保持方針を、メンバーが適用範囲と削除予定を確認した保持方針に置き換える。適用するかどうかは `FormSettings.retentionToApply` が決める。

- **シナリオ**: FORM-06
- **認可**: 実行者は `member`。`intake:manage`
- **入力**: `FormId`、確認した保持方針（保持する日数。「保持方針の既存の回答への適用を確認する」が返した値）
- **出力**: 適用を終えた（`applied`）、または保持方針が変わった（`policyChanged`。現在の保持方針を添える）。どちらも、置き換えた回答の件数を持つ
- **使用するドメインの振る舞い・ポート**: `RetentionInput.validate`、`FormSettingsRepository.findById`、`FormSettings.requireActive`、`FormSettings.retentionToApply`、`SubmissionRepository.replaceRetentionByForm`
- **トランザクション境界**: UnitOfWork 必要。条件は `active`。`formSettingsRepository`（読み取り）と `submissionRepository` を使う。件数を区切り、複数の UnitOfWork に分ける
  - どの UnitOfWork も、認可を行い、`FormSettings` を読み直して `requireActive` を呼び、`retentionToApply` に確認した保持方針を渡す。`apply` が真なら、その保持方針で `replaceRetentionByForm` を、`limit` に `FormSweep.maxLimit` を渡して、共通の「書き込みの UnitOfWork」のとおり呼ぶ。返る件数が `limit` より少なくなるまで、別の UnitOfWork で繰り返す
  - `apply` が偽の UnitOfWork は、何も書かずに終え、以後の UnitOfWork を開かない

## 受付と通知

### フォームの送信を受け付ける（`acceptSubmission`）

アプリが送ったフォームのキーと入力内容を、受付時点の現行の定義で検証し、回答と、その通知を保存して、ドメインの判定を受付結果として返す。

- **シナリオ**: APP-08, APP-07, SMIG-13
- **認可**: 実行者は `appToken` または `memberToken`。トークンの許可 `form_submit`。対象は送信先のフォーム（`AccessTarget` の `form`）。権限を必要としない
- **イベント**: `form.submission_accepted`
- **入力**: `Actor`、`WorkspaceId`、フォームのキー、`SubmissionInput`、`conditionKeys`（経路が送信の本文から作る列。[usecases/app-api.md](app-api.md) の「フォームの受付結果」。無ければ空）、前提とするスキーマの識別子（`ClaimedFingerprints`。添えなくてよい）
- **出力**: `ReceptionResponse`（[domains/form.md](../domains/form.md) の「ReceptionResult」）
- **使用するドメインの振る舞い・ポート**:
  - 認可: [domains/index.md](../domains/index.md) の「認可」の、トークンの認可の手続き（`authorizeTokenOperation`）。`AccessRequirement` は、`grant` が `form_submit`、`target` が解決した `FormId`、`permissions` が空
  - 利用量: `SubmissionRatePolicy.key`、`SubmissionRatePolicy.limit`、`UsageLimiter.check`、`UsageLimiter.record`
  - 受付: `WorkspaceSchemaRepository.find`、`Schema.form`、`WorkspaceSchema.matchFingerprints`、`FormSettingsRepository.findById`、`SubmissionValidation.inquiryTargets`、`ContentRepository.findByIds`、`PublishedContentFacts.of`、`IdGenerator.generate`、`SubmissionId.create`、`Clock.now`、`Submission.receive`、`SubmissionRepository.insert`、`SubmissionNotificationRepository.insert`、`collectEvents`
  - 問い合わせ対象の事実（`InquiryFacts`）は、`PublishedContentFacts.of` の結果から作る。`null` は `unavailable`、ほかは `published` にする
- **トランザクション境界**: UnitOfWork 必要。条件は `schemaVersion`（フォームのキーを `FormId` に解決するときに読んだ `WorkspaceSchema` のスキーマの版）。結果の優先（[domains/form.md](../domains/form.md) の「受付」の表）の順 1 から 4 は、UnitOfWork の前に、`PartitionReader` のリポジトリと `UsageLimiter` で確かめ、順 5 から 7 は、受付の UnitOfWork の中で `Submission.receive` が決める。順 2 に当たる送信は、UnitOfWork を開かない
  - 受付の UnitOfWork は、`tokenRepository`、`workspaceRepository`、`memberRepository`、`formSettingsRepository`、`contentRepository`（読み取り）、`submissionRepository`、`submissionNotificationRepository` を使う。`memberToken` の認可が読む `Account` は `PartitionReader.installation()` から読む。書き込みの前にトークンの認可の手続きをもう一度行い、一つの `FormSettings` を読み、`Submission.receive` の結果の回答の `insert`、通知の `insert`、`form.submission_accepted` の記録を確定する。読んだ設定が `active` でない、または無い場合は、`receive` を呼ばずに `fn` を終える
  - `UsageLimiter.record` は、UnitOfWork の前に呼ぶ

### 回答の通知を送る（`sendSubmissionNotification`）

`form.submission_accepted` と `form.notification_retry_requested` を消費し、イベントが指す回の通知メールを送って、宛先ごとの送信の結果を `SubmissionNotification` に記録する。

- **シナリオ**: APP-08, FORM-09, FORM-10, FORM-11, HOST-09
- **認可**: 実行者は `system`
- **入力**: イベントの `workspaceId`（ペイロードと別の項目）と、ペイロード。`form.submission_accepted` のペイロードは `submissionId`、`form.notification_retry_requested` は加えて `round`（始める `retry` の回）
- **出力**: なし
- **使用するドメインの振る舞い・ポート**: 共通の「通知の送信」を、イベントの `workspaceId` の区画で、イベントの `submissionId` と、始める回で行う。始める回は、`form.submission_accepted` では `initial`、`form.notification_retry_requested` ではイベントの `round` の `retry`
- **トランザクション境界**: 共通の「通知の送信」が定める

## 回答と通知の送信結果

回答一件は通知を一つ持つ。回答の一覧と詳細は一組で、入力内容と通知の送信結果を返す。

### 回答の対象のフォームを一覧する（`listSubmissionForms`）

回答の絞り込みに指定できるフォーム（`FormInventory.remaining`）を返す。

- **シナリオ**: FORM-07, FORM-09, FORM-11
- **認可**: 実行者は `member`。`submission:read` または `notification:manage`
- **入力**: なし
- **出力**: フォームの一覧
- **使用するドメインの振る舞い・ポート**: `FormSettingsRepository.findAll`、`SubmissionNotificationRepository.summarizeByForm`、`FormInventory.remaining`、`WorkspaceSchemaRepository.find`
- **トランザクション境界**: UnitOfWork 不要。書き込みがない

### 回答を一覧・検索する（`searchSubmissions`）

保存した回答を、`SubmissionFilter` で絞り込み、検索して返す。

- **シナリオ**: FORM-07, FORM-09, FORM-11
- **認可**: 実行者は `member`。`submission:read` または `notification:manage`
- **入力**: `SubmissionFilter` に対応する条件（`FormId`、受付期間、通知の条件（`NotificationCondition`）、検索の文字列）、ページの指定
- **出力**: 回答ごとの行と総件数。行は、回答一件とその通知の状態
- **使用するドメインの振る舞い・ポート**: `Pagination.create`、`SubmissionFilter.create`、`WorkspaceSchemaRepository.find`（`listFields` の対応を作る）、`Confidential.of`、`SubmissionRepository.findPage`、`SubmissionQuery.listValues`（回答と、現行スキーマのそのフォームの `listFields` のキーを渡す。削除済みフォームの回答には空の列を渡す）、`SubmissionNotificationRepository.findByIds`、`SubmissionNotification.result`、`Confidential.reveal`
- **トランザクション境界**: UnitOfWork 不要。書き込みがない

### 回答を取得する（`getSubmission`）

回答一件の、受付時の項目と入力内容、通知の宛先ごとの送信の状態を返す。

- **シナリオ**: FORM-07, FORM-09, FORM-10, FORM-11
- **認可**: 実行者は `member`。`submission:read` または `notification:manage`
- **入力**: `SubmissionId`
- **出力**: 回答一件と、その通知の状態
- **使用するドメインの振る舞い・ポート**: `SubmissionRepository.findById`、`SubmissionNotificationRepository.findById`、`SubmissionNotification.result`、`SubmissionNotification.recipientStatuses`、`Confidential.reveal`、`ContentRepository.findByIds`（問い合わせ対象のコンテンツの現在の置き場所は、読んだコンテンツの `location.kind`。無ければ `null`）
- **トランザクション境界**: UnitOfWork 不要。書き込みがない

### 回答を削除する（`deleteSubmissions`）

選んだ回答を、その通知とともに、一度の削除として削除する。

- **シナリオ**: FORM-08, FORM-11
- **認可**: 実行者は `member`。`submission:delete`
- **入力**: `SubmissionId` の列。`BatchTargets.create` で、`max` を `SubmissionBatch.maxTargets` にして作る
- **出力**: なし
- **使用するドメインの振る舞い・ポート**: `BatchTargets.create`、`SubmissionRepository.deleteMany`
- **トランザクション境界**: UnitOfWork 必要。条件は `active`。`submissionRepository` を使う。認可と、一度の `deleteMany` を一つの UnitOfWork で行う。書き込みは、対象の件数によらず `deleteMany` の一回

### 未対処の送信失敗の件数を取得する（`countUnhandledFailures`）

ワークスペースの、未対処の送信失敗が残る通知の件数を返す。

- **シナリオ**: FORM-09
- **認可**: 実行者は `member`。`notification:manage`
- **入力**: なし
- **出力**: 件数
- **使用するドメインの振る舞い・ポート**: `SubmissionNotificationRepository.countUnhandledFailures`
- **トランザクション境界**: UnitOfWork 不要。書き込みがない

### 通知の送信失敗を対処済みとして記録する（`markFailuresHandled`）

送信失敗の通知を一件以上選び、一度に対処済みにする。対処の状態の規則は、[domains/common.md](../domains/common.md) の「FailureHandling」が定める。

- **シナリオ**: FORM-09
- **認可**: 実行者は `member`。`notification:manage`
- **入力**: `SubmissionId` と、確認した回（`NotificationAttempt`。選んだときに示した通知の回）の組の列。`BatchTargets.createBy` で、`max` を `SubmissionBatch.maxTargets`、`key` を `SubmissionId` にして作る
- **出力**: `BulkHandlingDecision` に対応する結果
- **使用するドメインの振る舞い・ポート**: `BatchTargets.createBy`、`SubmissionNotificationRepository.findByIds`、`Clock.now`、`SubmissionNotification.markHandledAll`（入力の組ごとに、読めた通知は通知と確認した回の `found`、読めなかった `SubmissionId` は `notFound` の組。実行者の `MemberId`）、`SubmissionNotificationRepository.save`
- **トランザクション境界**: UnitOfWork 必要。条件は `active`。`submissionNotificationRepository` を使う。認可、`findByIds`、`markHandledAll` が返した通知の `save` を、一つの UnitOfWork で確定する。`save` は、`markHandledAll` が返した通知ごとに一度だけ行う。書き込みは、多くて `SubmissionBatch.maxTargets` 回

### 手動再試行の宛先を確認する（`previewRetryRecipients`）

選んだ通知ごとに、再試行の宛先と、受付時の通知対象から外れた宛先・加わった宛先を返す。

- **シナリオ**: FORM-10, FORM-11
- **認可**: 実行者は `member`。`notification:manage`
- **入力**: `SubmissionId` の列。`BatchTargets.create` で、`max` を `SubmissionBatch.maxTargets` にして作る
- **出力**: 通知ごとの、確認した回（通知の `attempt`）、`RetryPlan`、再試行を受け付けられるかどうかと、受け付けられない理由（`notFailed`、`noRecipients`）。メンバーは、ここで示した回を、「通知を手動で再試行する」に、確認した回として渡す
- **使用するドメインの振る舞い・ポート**: `BatchTargets.create`、`SubmissionNotificationRepository.findByIds`、`FormSettingsRepository.findById`（通知の `formId` の設定）、`SubmissionNotification.retryPlan`、`Clock.now`、`SubmissionNotification.requestRetry`（`confirmed` に通知の `attempt` を渡す。結果を保存しない）。振る舞いには、読んだ設定を渡す
- **トランザクション境界**: UnitOfWork 不要。書き込みがない

### 通知を手動で再試行する（`retryNotifications`）

選んだ送信失敗の通知の、宛先を確認した回の次の回を始める。送信は「回答の通知を送る」が行う。

- **シナリオ**: FORM-10, FORM-11
- **認可**: 実行者は `member`。`notification:manage`
- **イベント**: `form.notification_retry_requested`（再試行を受け付けた通知ごとに一件）
- **入力**: `SubmissionId` と、確認した回（`NotificationAttempt`。「手動再試行の宛先を確認する」が返した値）の組の列。`BatchTargets.createBy` で、`max` を `SubmissionBatch.maxTargets`、`key` を `SubmissionId` にして作る
- **出力**: `BulkRetryDecision` に対応する結果。再試行を始めた結果は、通知ごとに送信結果（共通の「通知の送信結果」）を持つ
- **使用するドメインの振る舞い・ポート**: `BatchTargets.createBy`、`SubmissionNotificationRepository.findByIds`、`FormSettingsRepository.findById`（通知の `formId` の設定）、`Clock.now`、`SubmissionNotification.requestRetries`（入力の組ごとに、読めた通知は通知、確認した回、読んだ設定の `found`、読めなかった `SubmissionId` は `notFound` の組を渡す）、`SubmissionNotification.result`、`SubmissionNotificationRepository.save`、`collectEvents`
- **トランザクション境界**: UnitOfWork 必要。条件は `active`。`submissionNotificationRepository` と `formSettingsRepository`（読み取り）を使う。認可、`findByIds`、`requestRetries` の結果の `save`、イベントの記録を、一つの UnitOfWork で確定する。`save` は、再試行を受け付けた（`requested` の）通知ごとに一度だけ行う。書き込みは、通知の `save` とイベントが、受け付けた通知ごとに一回ずつで、多くて `SubmissionBatch.maxTargets` の 2 倍

## 定期ジョブ

### 保持期限を過ぎた回答を自動整理する（`purgeExpiredSubmissions`）

対象のワークスペースの、保持期限を過ぎた回答を、その通知とともに削除する。

- **シナリオ**: FORM-06, FORM-11（定期ジョブ「回答の自動整理」）
- **認可**: 実行者は `system`
- **入力**: `WorkspaceId`。呼び出し側が、[domains/index.md](../domains/index.md) の「定期ジョブ」の表の、定期ジョブ「回答の自動整理」の対象のワークスペースごとに呼ぶ
- **出力**: そのワークスペースで削除した回答の件数
- **使用するドメインの振る舞い・ポート**: `Clock.now`、`SubmissionRepository.deleteExpired`
- **トランザクション境界**: UnitOfWork 必要。量が件数に比例するので、複数の UnitOfWork に分ける。どの UnitOfWork も、対象のワークスペースの区画を書き、条件は `none`（対象に、区画が `closing` のワークスペースを含むため）。`submissionRepository` を使う。`deleteExpired` は、[domains/common.md](../domains/common.md) の「TransactionalRepository」の「件数を区切る削除」の進め方で呼ぶ

### 回答が残らない削除済みフォームの運用設定を削除する（`disposeRemovedFormSettings`）

対象のワークスペースの、回答が残らない削除済みフォームの運用設定（`FormInventory.disposable` が返す設定）を削除する。

- **シナリオ**: FORM-11（定期ジョブ「回答の自動整理」）
- **認可**: 実行者は `system`
- **入力**: `WorkspaceId`。呼び出し側が、「保持期限を過ぎた回答を自動整理する」と同じワークスペースごとに、その呼び出しを終えた後に続けて呼ぶ。先の呼び出しがエラーで終えたワークスペースでも呼ぶ
- **出力**: そのワークスペースで削除した設定の件数
- **使用するドメインの振る舞い・ポート**: `FormSettingsRepository.findAll`、`SubmissionNotificationRepository.summarizeByForm`、`FormInventory.disposable`、`FormSettingsRepository.delete`
- **トランザクション境界**: UnitOfWork 必要。設定ごとに別の UnitOfWork に分ける。どの UnitOfWork も、対象のワークスペースの区画を書き、条件は `none`（対象に、区画が `closing` のワークスペースを含むため）。`formSettingsRepository` を使う
  - UnitOfWork を開く前に `findAll` と `summarizeByForm` を読み、`FormInventory.disposable` で対象の設定を求める
  - 設定ごとに、読んだ `expectedVersion` を渡して `delete` を確定する

### 止まった通知を検出する（`detectStalledNotifications`）

対象のワークスペースの、止まった通知（`SubmissionNotification.stalled` が当たる通知）を進める。

- **シナリオ**: APP-08, FORM-09, FORM-10, HOST-09（定期ジョブ「止まった通知の検出」）
- **認可**: 実行者は `system`
- **入力**: `WorkspaceId`。呼び出し側が、[domains/index.md](../domains/index.md) の「定期ジョブ」の表の、定期ジョブ「止まった通知の検出」の対象のワークスペースごとに呼ぶ
- **出力**: そのワークスペースで送信を行った件数（共通の「通知の送信」が `send` を返した件数）と、送信失敗にした件数（`record` を返した件数）
- **使用するドメインの振る舞い・ポート**: `Clock.now`、`SubmissionNotificationRepository.findStalled`、`SubmissionNotificationRepository.findById`、共通の「通知の送信」
- **トランザクション境界**: UnitOfWork 必要。通知ごとに、共通の「通知の送信」の UnitOfWork を別に確定する
  - `findStalled` は、UnitOfWork を開く前に、`limit` に `FormSweep.maxLimit` を渡して一度呼ぶ
  - `findStalled` が返した通知ごとに、通知を読み、共通の「通知の送信」を、その通知の `attempt` で行う
