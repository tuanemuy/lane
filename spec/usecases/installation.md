# Installation のユースケース

設置の起動と初期構成の読み込み、設置の更新、動作確認と状態確認、ワークスペースの作成と利用終了、バックアップと復旧を行う。型、振る舞い、ポートは [../domains/installation.md](../domains/installation.md) が、区画とドメインをまたぐ規則は [../domains/index.md](../domains/index.md) が、共通のポートとエラーの理由は [../domains/common.md](../domains/common.md) が定める。各ユースケースの振る舞いの結果（結果の種類ごとの条件、エラー、途中で止まったときに残る状態とその後の扱い）は、`spec/testcases/installation/` の、見出しの識別子と同じ名前のファイルが定める。

各ドメインが公開する手順と項目を呼ぶ分担は、[../domains/index.md](../domains/index.md) の「依存方向」が定める。手順が使う振る舞いとポート、UnitOfWork の区切り、やり直しは各ドメインの節が定め、このファイルは、公開された手順の振る舞いとポートの名前を並べない。このファイルは、どの手順をどの区画の UnitOfWork で呼ぶかを書く。各ドメインの節の手順に当たらない読み取り（「紐付けを待つ運用設定を一覧する」）だけは、そのユースケースが、読むポートのメソッドを書く。

コードは `packages/core/src/application/installation/` に置く。見出しの識別子は、`spec/testcases/installation/` のファイル名と同じ。

管理者復旧の回復の値の発行（HOST-11）は Identity のユースケースが、メンバーが管理画面で行う紐付けを待つ運用設定の確認（HOST-03）は Form と Token のユースケースが、初期所有者の招待の受諾（HOST-04）は Workspace のユースケースが行う。バックアップの定期的な取得と、定期ジョブを名簿のワークスペースごとに呼ぶ規則は、[../domains/index.md](../domains/index.md) の「定期ジョブ」が定める。

## 共通

### 認可

- 運用コマンドのユースケース（「設置を起動する」を含む）は、`operator` だけを受け付け、運用コマンドの入口が呼ぶ。「ワークスペースの利用終了を完了させる」は、`system` だけを受け付ける。「設置の状態を確かめる」は、実行者の種類によらず、入口が使う（使う入口は、そのユースケースの「認可」）。判定と、受け付けない実行者の結果は、[../domains/index.md](../domains/index.md) の「認可」が定める

### 設置の状態の確認

運用コマンドのユースケースが初めに行う確認。受け付けるものと、受け付けないときの結果は、[../domains/installation.md](../domains/installation.md) の「設置の状態」だけが、入口とユースケースのどちらが確かめるかは、[../domains/index.md](../domains/index.md) の「設置の状態」が定める。

- 設置の設定の確認: `InstallationEnvironment.inspectSettings` の結果を `BootReadiness.settingIssues` に渡し、その結果と、自分のコマンド（各ユースケースの「コマンド」）を `InstallationState.blockingIssues` に渡す
- 状態の確認: 設定の問題が返らず、自分のコマンドが `StateCheckedAccess` に当たれば、`InstallationRepository.find`（`PartitionReader.installation()`）、`RecoveryJournal.find`、`InstallationEnvironment.dataVersion`、`Clock.now` の結果から `InstallationState.of` で状態を求め、`InstallationState.admit` に、状態と自分のコマンドを渡す
- 各ユースケースの「設置の状態の確認」は、この二つの確認を指す。各ユースケースは、この確認と、使うポートを書き直さない
- 「バックアップを一覧する」（`list_backups`）は、設置の設定の確認だけを行い、`Installation` と復旧の記録を読まない
- 「設置を起動する」は、設置の設定の確認の代わりに、`BootReadiness.assess` で設置の設定と資源を確かめる

### 出力

- 出力が含まない値は、[../domains/installation.md](../domains/installation.md) の「ユースケース（概要）」が、URL の組み立てと、初期所有者の招待の値を出力に含める設置は、[../domains/index.md](../domains/index.md) の「URL の組み立て」が定める

### 区画と UnitOfWork

- 区画と UnitOfWork は、[../domains/common.md](../domains/common.md) の「WriteScope」「UnitOfWorkProvider」に従う。「設置の区画」は `{ kind: "installation" }`、「ワークスペースの区画（`none`）」は `{ kind: "workspace", workspaceId, condition: { kind: "none" } }` を指す
- 二つの区画を書くユースケースは、区画ごとの UnitOfWork を、[../domains/index.md](../domains/index.md) の「区画をまたぐ手順」の順に確定し、同じ節の「手順ごとの規則」に従う
- 一つの UnitOfWork の書き込みは、区画の回数（[../domains/common.md](../domains/common.md) の `UnitOfWorkCapacity`）に収める
- UnitOfWork の外の読み取りは、`PartitionReader.installation()` と `PartitionReader.workspace(workspaceId)` のリポジトリで行う
- 「名簿のワークスペースごとに読む」は、`WorkspaceDirectory.findPageByStatus` で、指定した状態の項目を最後のページまで読み、項目ごとに、その区画を `PartitionReader.workspace` で読むことを指す
- `Installation` を保存する UnitOfWork は、中で `InstallationRepository.find` を呼び、そこで得た `expectedVersion` で `save` する。中で読んだ `Installation` が、その実行が前提にした `Installation` と違う場合は、[../domains/installation.md](../domains/installation.md) の「InstallationRepository」が定める

### 手順を呼ぶ順と段の順

- 各ドメインの手順を呼ぶ順（初期構成の宣言の検証と生成、利用再開前の処理、利用終了の区画の消去）は、[../domains/index.md](../domains/index.md) の「手順を呼ぶ順」の表が定める。区画をまたぐ段の順は、同じ節の「区画をまたぐ手順」の表が定める。このファイルは、順を持たず、二つの表を参照する
- 各ドメインが公開する手順の段の `WriteScope`、UnitOfWork の区切り、書き込みの回数は、手順を公開する節が定め、どの段も `UnitOfWorkCapacity` に収まる。このファイルが段ごとの `WriteScope` と回数を書くのは、`Installation` を書く段を持つユースケースだけで、段は [../domains/index.md](../domains/index.md) の段の名前で書く

### 利用終了の手順

「ワークスペースの利用を終了する」と「ワークスペースの利用終了を完了させる」は、同じ手順を、[../domains/index.md](../domains/index.md) の「区画をまたぐ手順」の「ワークスペースの利用終了」の行の段の順で呼ぶ。名簿の状態の変更と開始は、Workspace の「利用終了の消去」が公開し、各段の `WriteScope` と UnitOfWork の区切りは、その節が定める。

区画の消去は、次のとおり。

- [../domains/index.md](../domains/index.md) の「手順を呼ぶ順」の順に、各ドメインの「利用終了の消去」が公開する、区画の消去を呼ぶ。区画の消去のほかに公開された消去（区画の外の実体の消去、設置の区画の索引の除外）は、ここで呼ばず、「ワークスペースの利用終了を完了させる」が呼ぶ
- UnitOfWork の外で行うと公開された消去は、UnitOfWork を開かずに呼ぶ
- 区画の消去の終わり方と、確定済みの手順と消去済みの手順の通過は、Workspace の「利用終了の消去」が定める

### 各ドメインの項目

状態確認、終了前の確認、利用再開前の確認は、各ドメインの節が公開する型と関数を呼んで、項目を集める。項目の作り方は [../domains/common.md](../domains/common.md) の「各ドメインが公開する項目」に従う。このファイルは、公開された `kind`、型、関数の名前を並べない。

| 集めるもの | 公開する節 |
| --- | --- |
| 状態確認の、ワークスペースごとの項目と、設置の全体の項目。項目のワークスペースごとに添える値 | 各ドメインの「状態確認の項目」。項目の単位（設置、ワークスペース）、読むメソッド、添える値は、各節が定める |
| 終了前の確認の、件数の項目、失われるデータの種類、消去する資格情報、CMS の通常データの外に残る情報 | 各ドメインの「利用終了の消去」 |
| 利用再開前の確認の項目と、残件数 | 各ドメインの「利用再開前の処理」。項目は `ResumeItemDefinition` の形（[../domains/common.md](../domains/common.md) の「各ドメインが公開する項目」）で、単位と残件数への算入を持つ |

- その節に項目を公開しないドメインは、集める対象にならない
- 件数の読み取りは、各節が公開するポートのメソッドを、`PartitionReader` のリポジトリで呼ぶ

## 設置を起動する（`bootInstallation`）

- **シナリオ**: HOST-01、HOST-02、HOST-05、HOST-08、SDEV-04、SDEV-06
- **認可**: `operator`
- **コマンド**: `boot`
- **操作履歴**: `bootstrap.loaded`、`token.issued`（実行者はどちらも `operator`）

### 概要

起動のたびに、設置の設定と資源から起動できるかどうかを確かめる。起動できる設置では、`Installation.decideBoot` が決めたこと（初期構成の読み込み、読み込みの続き、宣言なしの開始、読み込まない起動、起動しない）を行う。決まり方と、起動が復旧の記録を変える場面は、[../domains/installation.md](../domains/installation.md) の「設置の状態」と「起動と初期構成」が定める。

### 入出力

- **入力**: 初期構成の宣言（宣言が無ければ `null`）。形と、形に合わない記述の扱いは [../domains/installation.md](../domains/installation.md) の「初期構成の読み込み」の表が定め、各位置の宣言の型は、受けるドメインの「初期構成の宣言」の節が定める
- **出力**: 起動の結果。読み込んだ、宣言なしで始めた、読み込まずに起動した、起動しない、のいずれか。各結果が示す内容は、[../domains/installation.md](../domains/installation.md) の「起動と初期構成」「初期構成の読み込み」「BootReadiness」が定める

### 使用するドメインの振る舞い・ポート

- `InstallationEnvironment.inspectSettings`、`InstallationEnvironment.readResource`（`ResourceKind` の三つ）、`BootReadiness.assess`
- 設置の状態の確認、`Installation.decideBoot`
- `DeclarationDigest.source`（各ドメインが「宣言の照合値から除く値」として公開するプロパティを除いた宣言を渡す）、`SecretTokenIssuer.digest`、`DeclarationDigest.create`
- [../domains/index.md](../domains/index.md) の「手順を呼ぶ順」の順に、各ドメインの「初期構成の宣言」が公開する検証と生成の手順。宣言の位置（`path`）は、[../domains/installation.md](../domains/installation.md) の「初期構成の読み込み」の表から渡す。先のドメインの生成の結果は、後のドメインの生成へ渡す
- `RecoveryJournal.clear`（初期化の前の設置に残った、完了した復旧の記録の消去）
- `Installation.initialize`、`InstallationRepository.insert`、`InstallationRepository.find`、`Installation.completeInitialConfiguration`、`InstallationRepository.save`
- `Clock.now`、`IdGenerator.generate`
- `AuditActor.from`、`AuditLogEntry.record`、`AuditLogRepository.appendMany`（ワークスペースごとの `bootstrap.loaded` と、そのワークスペースで宣言したトークンごとの `token.issued`）

### トランザクション境界

- **UnitOfWork の要否**: 必要
- `BootReadiness.assess`、設置の状態の確認、宣言の照合値の計算、`decideBoot`、宣言の検証は、UnitOfWork の外で行う。宣言の検証は、`load_initial_configuration` と `resume_initial_configuration` の起動だけが行う
- 各ドメインの手順が UnitOfWork の外で行うと定めた処理（秘密情報の保護、照合値の計算、ID の生成）は、その段の UnitOfWork を開く前に済ませる
- 読み込みは、[../domains/index.md](../domains/index.md) の「区画をまたぐ手順」の初期構成の読み込みの段の順に、段ごとの UnitOfWork で確定する
- 段ごとの `WriteScope` と、書き込みの回数
  - 設置の区画の最初の段: 設置の区画。`loading` の `Installation` の `insert`（1 回）と、各ドメインの生成の、設置の区画の部分。宣言のすべてのワークスペースの分を、一つの UnitOfWork で書く
  - ワークスペースごとの区画の段: ワークスペースの区画（`none`）。各ドメインの生成の、その区画の部分と、`bootstrap.loaded` と、そのワークスペースで宣言したトークンごとの `token.issued` の `AuditLogEntry` の追記（`appendMany` の一回の呼び出し）
  - ワークスペースごとの設置の区画の段: 設置の区画。Workspace の生成の、名簿と所属の索引の部分
  - `Installation` を読み込み済みにする段: 設置の区画。UnitOfWork の中で `find` で読んだ `Installation` の、`completeInitialConfiguration` の結果の `save`。1 回。前提にする `Installation` は、最初の段で保存したもの（続きを進める起動では、状態の確認で読んだもの）
- ワークスペースごとの二つの段は、宣言の順に、一つのワークスペースの二つの段を終えてから、次のワークスペースへ進む
- 続きを進める起動は、最初の段の UnitOfWork を開かない。宣言のワークスペースの段を順に進め、すべてのワークスペースの後に、`Installation` を読み込み済みにする段を行う
- 宣言なしで始める起動は、設置の区画の一つの UnitOfWork で、`initialConfiguration` が `settled` の `Installation` の `insert` だけを確定する（1 回）
- 状態の `leftover` が `null` でない設置の `load_initial_configuration` と `start_without_configuration` の起動は、宣言の検証の後、最初の UnitOfWork を開く前に、UnitOfWork の外で `RecoveryJournal.clear` で、その記録を消す。`expectedVersion` は、状態の確認で読んだ記録の値

## 設置の状態を確かめる（`checkInstallationAvailability`）

- **シナリオ**: HOST-01、HOST-02、HOST-05、HOST-07、HOST-08、HOST-11
- **認可**: 実行者の種類によらない。呼ぶ入口は [../domains/index.md](../domains/index.md) の「設置の状態」が定める

### 概要

設置の設定と状態から、入口が始めようとする処理（利用者の要求、ワーカーの処理、ほかのドメインの運用コマンド）を受け付けるかどうかを返す。受け付けない結果を、入口が `access` ごとにどう返すかは、[../domains/installation.md](../domains/installation.md) の「設置の状態」が定める。

### 入出力

- **入力**: `InstallationAccess`（`request`、`worker`、または運用コマンドの種類を持つ `operations`）
- **出力**: `Admission`。受け付ける（`admitted: true`）、状態が受け付けない（`StateRejection`）、設置の設定が受け付けない（`SettingsRejection`）のいずれか

### 使用するドメインの振る舞い・ポート

- `InstallationEnvironment.inspectSettings`、`BootReadiness.settingIssues`、`InstallationState.blockingIssues`
- `InstallationRepository.find`（`PartitionReader.installation()`）、`RecoveryJournal.find`、`InstallationEnvironment.dataVersion`
- `InstallationState.of`、`InstallationState.admit`、`Clock.now`

### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みが無い。`BackupStore` を読まない
- 受けた `access` で、共通の「設置の状態の確認」と同じ読み取りと関数を呼ぶ

## 設置を更新する（`updateInstallation`）

- **シナリオ**: HOST-05
- **認可**: `operator`
- **コマンド**: `update`

### 概要

保存済みのデータを、実行中の CMS が扱うデータの版へ更新し、`Installation` が持つ保存済みのデータの版を進める。更新の進め方、更新の前の判定と、更新中の設置の扱いは、[../domains/installation.md](../domains/installation.md) の「UpdateReadiness」が、更新の前に必要な作業と影響、停止時間と復旧方法を示す文書は「更新手順」が定める。

### 入出力

- **入力**: 無し
- **出力**: 更新が完了した（更新後の版）、または更新が不要

### 使用するドメインの振る舞い・ポート

- 設置の状態の確認（状態の `installation` と `running` を使う）
- `InstallationEnvironment.supportsUpdateFrom`（`installation.dataVersion` を渡す）
- `InstallationEnvironment.readResource`（`ResourceKind` の三つ）
- `UpdateReadiness.assess`
- `InstallationRepository.find`、`Installation.beginUpdate`、`InstallationRepository.save`
- `InstallationEnvironment.updateData`
- `InstallationRepository.find`、`Installation.completeUpdate`、`InstallationRepository.save`
- `Clock.now`

### トランザクション境界

- **UnitOfWork の要否**: 必要
- 読み取りと `UpdateReadiness.assess` は、UnitOfWork の外で行う
- 更新の開始: 設置の区画の一つの UnitOfWork で、その中で `find` で読んだ `Installation` の `beginUpdate` の結果を `save` する。書き込みは 1 回。前提にする `Installation` は、状態の確認で読んだもの。更新中の設置（状態が `updating`）では、この UnitOfWork を開かず、`updateData` から始める
- `InstallationEnvironment.updateData` は、更新の開始の確定の後に、UnitOfWork を開かずに呼ぶ
- 更新の完了: `updateData` が `ok: true` を返した場合に、設置の区画の一つの UnitOfWork で、その中で `find` で読んだ `Installation` の `completeUpdate` の結果を `save` する。書き込みは 1 回。前提にする `Installation` は、更新の開始で保存したもの（更新中の設置では、状態の確認で読んだもの）

## 導入時の動作確認を行う（`runSetupCheck`）

- **シナリオ**: HOST-01、HOST-05
- **認可**: `operator`
- **コマンド**: `setup_check`

### 概要

保存先への書き込みと読み出し、メディアの保管先への保存と取り出し、指定した宛先への設置全体のメール送信を試し、項目ごとの結果と、不足している資源を返す。

### 入出力

- **入力**: 確認メールの宛先
- **出力**: `SetupCheckReport`

### 使用するドメインの振る舞い・ポート

- 設置の状態の確認
- `EmailAddress.create`
- `InstallationEnvironment.probe`（`data_store`、`media_store`）
- `SystemMailer.send`
- `InstallationEnvironment.readResource`（`ResourceKind` の三つ）
- `StatusAssessment.setupCheck`

### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みが無い

## ワークスペースを作成し、初期所有者の招待を発行する（`createWorkspace`）

- **シナリオ**: HOST-04
- **認可**: `operator`
- **コマンド**: `create_workspace`
- **操作履歴**: `invitation.created`（実行者は `operator`）

### 概要

ワークスペースを名簿に登録し、ワークスペースと、作成が作るワークスペースに一つの集約をそろえて作り、名簿を開設済みにし、指定した初期所有者に宛てた招待を発行して、設置全体のメール送信で送る。段の中身、作成の終わり、途中で止まった作成の続き、作成を終えた ID と違う指定の実行は、Workspace の「初期構成の宣言」のワークスペースの作成が、作成と並行した利用終了は、Workspace の「作成と利用終了の並行」が定める。

### 入出力

- **入力**: `WorkspaceId`（初回は入口が `IdGenerator.generate` で作る。実行し直しでは運営者が示された値を渡し、入口が `parse` に通す）、ワークスペースの名称、初期所有者のメールアドレス
- **出力**: 作成した（作成したワークスペースと、発行した招待の届け方の結果）、または作成済み

### 使用するドメインの振る舞い・ポート

- 設置の状態の確認
- Workspace の「初期構成の宣言」が公開する、ワークスペースの作成（運用コマンド）の手順
- [../domains/index.md](../domains/index.md) の「手順を呼ぶ順」の、運用コマンドによるワークスペースの作成が呼ぶ、Workspace のほかのドメインの「初期構成の宣言」が公開する生成を、宣言なしで呼ぶ
- Workspace の「初期構成の宣言」が公開する、初期所有者の招待の発行し直し（運用コマンド）の手順。「初期所有者の招待を発行し直す」が使う振る舞いとポートを、同じとおりに使う
- `EmailAddress.create`、`IdGenerator.generate`、`Clock.now`

### トランザクション境界

- **UnitOfWork の要否**: 必要
- 作成は、[../domains/index.md](../domains/index.md) の「区画をまたぐ手順」のワークスペースの作成の行の段の順に、段ごとの UnitOfWork で確定する。各段の `WriteScope` と UnitOfWork の区切りは、Workspace の「初期構成の宣言」のワークスペースの作成が定める。招待の発行の段は、「初期所有者の招待を発行し直す」のトランザクション境界のとおり。招待の値の生成、`invitation.created` の追記、メールの送信は、この段だけが行う

## 初期所有者の招待を発行し直す（`reissueInitialOwnerInvitation`）

- **シナリオ**: HOST-04、HOST-08
- **認可**: `operator`
- **コマンド**: `reissue_initial_owner_invitation`
- **操作履歴**: `invitation.created`（実行者は `operator`）

### 概要

所有者がいないワークスペースの初期所有者の招待を、宛先を指定して発行し直す。手順は Workspace の「初期構成の宣言」の初期所有者の招待の発行し直しが、対象と宛先は `InvitationProcess.issueInitialOwner` が、メンバーの招待との関係は「Invitation」が定める。

### 入出力

- **入力**: `WorkspaceId`、初期所有者のメールアドレス
- **出力**: 発行した招待と、その届け方の結果（メールの送信結果、または外部メールを使えない設置での招待の値）

### 使用するドメインの振る舞い・ポート

- 設置の状態の確認
- Workspace の「初期構成の宣言」が公開する、初期所有者の招待の発行し直し（運用コマンド）の手順（対象の確認、発行、招待を届ける、作成を終える段）
- `EmailAddress.create`、`IdGenerator.generate`、`Clock.now`
- `InstallationEnvironment.publicUrl`（招待のメールの本文の URL。共通の「出力」）

### トランザクション境界

- **UnitOfWork の要否**: 必要
- 段ごとの `WriteScope`、UnitOfWork の区切り、UnitOfWork の前の読み取り、各 UnitOfWork の中身、確定の後の届けは、Workspace の「初期構成の宣言」の発行し直しが定める

## 紐付けを待つ運用設定を一覧する（`listPendingBindings`）

- **シナリオ**: HOST-03
- **認可**: `operator`
- **コマンド**: `list_pending_bindings`

### 概要

初期構成でキーを指定して宣言した運用設定のうち、対象の作成を待つもの（紐付け待ち）を、ワークスペースを指定して、すべての種類について返す。

### 入出力

- **入力**: `WorkspaceId`
- **出力**: 紐付けを待つフォームの運用設定と、範囲の紐付けを待つトークン

### 使用するドメインの振る舞い・ポート

- 設置の状態の確認
- `WorkspaceDirectory.findById`（名簿の項目と状態）
- 指定したワークスペースの区画（`PartitionReader.workspace`）の読み取り
  - `PendingFormSettingsRepository.findAll`
  - `TokenRepository.findPage`（`pendingScopeOnly` を `true` にした `TokenListFilter`。最後のページまで。返る `ListedToken` の `token` を使う）、`TokenScope.pendingTargets`
- `Clock.now`

### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みが無い。名簿の項目と、一つのワークスペースの区画だけを読む

## 状態を確認する（`checkInstallationStatus`）

- **シナリオ**: HOST-05、HOST-06、HOST-09
- **認可**: `operator`
- **コマンド**: `status_check`

### 概要

保存容量と通知用の空き、各ドメインが公開する状態確認の項目（設置の全体の項目と、ワークスペースごとの項目）、隔離されたイベントの件数、最後のバックアップを集め、不足と項目ごとの必要な対処を返す。区画を読むワークスペースと、名簿の項目から示すワークスペースは、[../domains/index.md](../domains/index.md) の「区画をまたぐ手順」の状態確認が定める。

### 入出力

- **入力**: 無し
- **出力**: `StatusReport`

### 使用するドメインの振る舞い・ポート

- 設置の状態の確認
- `InstallationEnvironment.readResource`（`ResourceKind` の三つ）
- 「各ドメインの項目」（共通）の、状態確認の項目。項目ごとの件数、必要な対処の判定、該当するワークスペースごとに添える値（`StatusItemWorkspace.detail`）は、公開された読み取りと関数の結果を渡す
- `PartitionReader.workspace(workspaceId).quarantinedEventQuery.count()`。読むワークスペースと足し方は、[../domains/index.md](../domains/index.md) の「区画をまたぐ手順」の状態確認が定める
- `BackupStore.findLatest`
- `StatusAssessment.status`、`Clock.now`

### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みが無い

## バックアップを取得する（`takeBackup`）

- **シナリオ**: HOST-05、HOST-07、HOST-08
- **認可**: `operator`
- **コマンド**: `take_backup`

### 概要

設置全体を、区画ごとの時点の写しとして取得し、整合性を確かめ、バックアップの記録を作る。写しの取り方は [../domains/installation.md](../domains/installation.md) の「BackupStore」が定める。

### 入出力

- **入力**: 無し
- **出力**: `BackupRecord`

### 使用するドメインの振る舞い・ポート

- 設置の状態の確認
- `SecretCipher.currentKeyId`
- `IdGenerator.generate`、`BackupRecordId.create`、`Clock.now`
- `BackupStore.capture`、`BackupStore.verify`
- `BackupRecord.record`
- `BackupStore.complete`

### トランザクション境界

- **UnitOfWork の要否**: 不要。設置のデータに書き込まない。`BackupStore` は UnitOfWork に属さない
- `BackupRecordId` は、`capture` の前に作り、`capture`、`verify`、`BackupRecord.record` に同じ ID を渡す。`verify` の結果を、`BackupRecord.record` の `integrity` に写す

## バックアップを一覧する（`listBackups`）

- **シナリオ**: HOST-07、HOST-08
- **認可**: `operator`
- **コマンド**: `list_backups`

### 概要

取得済みのバックアップの記録を、`BackupStore.findPage` の順に、`BackupRecord.isPastRetention` の結果を添えて返す。復旧に使うバックアップの ID は、この一覧から得る。

### 入出力

- **入力**: `Pagination`
- **出力**: `BackupRecord` の列と総件数

### 使用するドメインの振る舞い・ポート

- 設置の状態の確認のうち、設置の設定の確認
- `Pagination.create`、`BackupStore.findPage`
- `BackupRecord.isPastRetention`、`Clock.now`

### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みが無い

## 復旧前の確認を示す（`previewRestore`）

- **シナリオ**: HOST-08
- **認可**: `operator`
- **コマンド**: `restore`（復旧の運用コマンドの入口が、「バックアップから設置全体を復旧する」の前に呼ぶ）

### 概要

指定したバックアップから復旧した場合の復旧前の確認（`RestorePreview`）を、データを変えずに返す。

### 入出力

- **入力**: `BackupRecordId`
- **出力**: `RestorePreview`

### 使用するドメインの振る舞い・ポート

- 設置の状態の確認
- `BackupStore.findById`
- `InstallationEnvironment.restorableSizeLimit`
- `WorkspaceDirectory.findPageByStatus`（すべての状態。最後のページまで。現在のワークスペースの ID と名称）
- `BackupRecord.previewRestore`、`Clock.now`

### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みが無い。復旧の記録を変えない

## バックアップから設置全体を復旧する（`restoreInstallation`）

- **シナリオ**: HOST-07、HOST-08
- **認可**: `operator`
- **コマンド**: `restore`

### 概要

設置全体のデータを、指定したバックアップの内容に置き換え、設置を利用再開前の状態にする。バックアップの ID は、「バックアップを一覧する」で得る。運用コマンドの入口は、初めに「復旧前の確認を示す」の結果を示して運営者の確認を求め、確認を受けた場合にだけ、このユースケースを呼ぶ。対話できない環境では、確認の指定が添えられた場合にだけ呼ぶ。このユースケースが「復旧した」を返した後に、入口は「利用再開前の確認を示す」を呼び、その結果を示す。復旧の記録の遷移と始め直しは [../domains/installation.md](../domains/installation.md) の「Recovery」が、置き換えの契約と、復旧の前提は「BackupStore」が定める。

### 入出力

- **入力**: `BackupRecordId`
- **出力**: 復旧した（復旧点）

### 使用するドメインの振る舞い・ポート

- 設置の状態の確認
- `BackupStore.findById`
- `InstallationEnvironment.dataVersion`、`SecretCipher.currentKeyId`（`RestoreEnvironment`）
- `Recovery.begin`、`RecoveryJournal.save`
- `BackupStore.restore`（区画の置き換えと、バックアップに無いメディアの実体の取り除きを含む）
- `Recovery.complete`、`Recovery.fail`、`Recovery.withdraw`、`RecoveryJournal.save`、`RecoveryJournal.clear`
- `Clock.now`

### トランザクション境界

- **UnitOfWork の要否**: 不要。設置のデータへの書き込みは `BackupStore.restore` だけで、UnitOfWork に属さない。復旧の記録は、設置のデータの外にある
- 開始: `Recovery.begin` の結果を `RecoveryJournal.save` で保存する。`expectedVersion` は、状態の確認で読んだ復旧の記録の `expectedVersion`（記録が無ければ `null`。初期化の前の設置に残った完了の記録も、この記録に当たる）
- 開始の保存が確定した後に、UnitOfWork を開かずに `BackupStore.restore` を呼ぶ
- `restore` の結果ごとに保存する記録は、[../domains/installation.md](../domains/installation.md) の「Recovery」の、`BackupStore.restore` の結果ごとの表のとおり。`expectedVersion` は、開始の保存の後の値。取り下げの結果が `null` なら、`RecoveryJournal.clear` で記録を消す

## 利用再開前の確認を示す（`reviewResume`）

- **シナリオ**: HOST-07、HOST-08
- **認可**: `operator`
- **コマンド**: `resume`。利用の再開の運用コマンドの入口が「利用を再開する」の前に呼び、復旧の運用コマンドの入口が「バックアップから設置全体を復旧する」が「復旧した」を返した後に呼ぶ。復旧した設置はデータの版が合う利用再開前（`awaiting_resume.version_matched`）にあり、`resume` を受け付ける

### 概要

利用の再開の前に運営者へ示す利用再開前の確認として、復旧点、失われる可能性のある期間、元に戻る操作の種類、再開の前に済ませる処理の残件数と、各ドメインが公開する確認に示す内容を、データを変えずに返す。確認に示す内容は、各ドメインの「利用再開前の処理」が公開する項目で、設置の全体の項目と、ワークスペースごとの項目に分かれる。

### 入出力

- **入力**: 無し
- **出力**: `ResumeReviewReport`

### 使用するドメインの振る舞い・ポート

- 設置の状態の確認（状態の `recovery` を使う）
- 「各ドメインの項目」（共通）の、利用再開前の確認の項目
- `ResumeReview.compose`

### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みが無い。復旧の記録を変えない
- 各ドメインの関数が返した項目を `CountedResumeItem` にして、`ResumeReview.compose` に渡す

## 利用を再開する（`resumeInstallation`）

- **シナリオ**: HOST-08
- **認可**: `operator`
- **コマンド**: `resume`

### 概要

利用再開前の設置で、各ドメインが公開する利用再開前の処理を済ませ、復旧の記録を消す。運用コマンドの入口は、初めに「利用再開前の確認を示す」の結果を示して運営者の確認を求め、確認を受けた場合にだけ、このユースケースを呼ぶ。対話できない環境では、確認の指定が添えられた場合にだけ呼ぶ。処理の中身と、処理の種類（残件数に数える項目の `kind`）は、各ドメインの「利用再開前の処理」が、再開の後に残るワークスペースの扱いは、Workspace の「利用再開前の処理」が定める。

### 入出力

- **入力**: 無し
- **出力**: 再開した後の設置の状態（`InstallationState` の `kind`）

### 使用するドメインの振る舞い・ポート

- 設置の状態の確認（状態の `recovery` を使う）。処理の前と、処理の後の二回行う
- `ResumeReview.remaining`、`Recovery.assertResumable`、`RecoveryJournal.clear`
- [../domains/index.md](../domains/index.md) の「手順を呼ぶ順」の順に、各ドメインの「利用再開前の処理」が公開する処理と、残件数の求め方
- `Clock.now`

### トランザクション境界

- **UnitOfWork の要否**: 必要
- UnitOfWork の区切り、`WriteScope`、一つの UnitOfWork の書き込みの回数は、各ドメインの節が定める
- ワークスペースの区画の処理は、各ドメインの節が定める状態の項目を名簿のワークスペースごとに読み（「名簿のワークスペースごとに読む」）、ワークスペースごとに、その区画で呼ぶ。一回の実行が読む対象の範囲は、各ドメインの節が定める
- 処理の後の設置の状態の確認が受け付けた設置では、各ドメインの項目の件数を区画の記録から読み、`reviewResume` と同じく区画を添えて `ResumeReview.remaining` に渡し、その結果と読み直した `restored` の記録で `Recovery.assertResumable` を通した後に、UnitOfWork の外で `RecoveryJournal.clear` で記録を消す。`expectedVersion` は、読み直した記録の値

## ワークスペースの終了前の確認を示す（`previewWorkspaceClosure`）

- **シナリオ**: HOST-09
- **認可**: `operator`
- **コマンド**: `preview_closure`

### 概要

ワークスペースの利用終了で、未完了の通知、失われるデータ、消去する資格情報、CMS の通常データの外に残る情報を、データを変えずに返す。示す内容は、[../domains/installation.md](../domains/installation.md) の「WorkspaceClosure」が定める。

### 入出力

- **入力**: `WorkspaceId`
- **出力**: `ClosurePreview`

### 使用するドメインの振る舞い・ポート

- 設置の状態の確認
- Workspace の「利用終了の消去」が公開する、終了前の確認の読み取り（名簿の項目）
- 「各ドメインの項目」（共通）の、終了前の確認の件数の項目と、失われるデータの種類、消去する資格情報、CMS の通常データの外に残る情報
- `BackupStore.findPage`（すべての記録）
- `WorkspaceClosure.preview`

### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みが無い
- 設置の区画と、そのワークスペースの区画を読む（区画をまたぐ読み取りは [../domains/index.md](../domains/index.md) の「区画」）

## ワークスペースの利用を終了する（`closeWorkspace`）

- **シナリオ**: HOST-04、HOST-09
- **認可**: `operator`
- **コマンド**: `close_workspace`

### 概要

ワークスペースの配信・受付・通知を終え、そのワークスペースの区画のデータの消去を進める。運営者は一回実行する。実行は、利用終了の完了を待たない。途中で止まったときの扱いは、Workspace の「利用終了の消去」と「作成と利用終了の並行」が定める。

### 入出力

- **入力**: `WorkspaceId`、ワークスペースの名称（対象の確認に使う）
- **出力**: 消去の進行中であること

### 使用するドメインの振る舞い・ポート

- 設置の状態の確認
- Workspace の「利用終了の消去」が公開する、対象の確認
- 「利用終了の手順」（共通）の、名簿の状態の変更、開始、区画の消去
- `Clock.now`

### トランザクション境界

- **UnitOfWork の要否**: 必要
- 対象の確認の後、[../domains/index.md](../domains/index.md) の「区画をまたぐ手順」のワークスペースの利用終了の順に行う。段ごとの `WriteScope`、呼ぶメソッド、UnitOfWork の内か外か、終わりの条件は、共通の「利用終了の手順」と、各ドメインの節が定める

## ワークスペースの利用終了を完了させる（`completeWorkspaceClosures`）

- **シナリオ**: HOST-09
- **認可**: `system`（定期ジョブ「利用終了の完了」。対象は、名簿で終了中のワークスペース）

### 概要

名簿で終了中のワークスペースについて、[../domains/index.md](../domains/index.md) の「区画をまたぐ手順」の利用終了の完了の段を、済んでいない段から進め、ワークスペースを名簿から外す。設置の区画から消す記録と、完了したワークスペースの区画に残るものは、Workspace の「利用終了の消去」が定める。

### 入出力

- **入力**: `WorkspaceId`（定期ジョブが、名簿で終了中のワークスペースごとに渡す）
- **出力**: 利用終了が完了したこと、または消去の進行中であること

### 使用するドメインの振る舞い・ポート

- Workspace の「利用終了の消去」が公開する、名簿の項目の読み取り
- 「利用終了の手順」（共通）の、開始と区画の消去。「ワークスペースの利用を終了する」と同じ手順を、同じ順で呼ぶ
- Media の「利用終了の消去」が公開する、実体の消去（消去を始められるかどうかの確かめと、保管先の実体の消去）。確かめの条件と、条件に要る記録は、Media の「MediaClosure」が定める
- Workspace の「利用終了の消去」が公開する、最後の段。ほかのドメインが、設置の区画の記録について公開する消去を含む
- `Clock.now`

### トランザクション境界

- **UnitOfWork の要否**: 必要
- [../domains/index.md](../domains/index.md) の「区画をまたぐ手順」の利用終了の完了の順に行う。段ごとの `WriteScope`、UnitOfWork の内か外か、書き込みの回数は、共通の「利用終了の手順」と、段を公開する節（Workspace と Media の「利用終了の消去」）が定める

## 設置全体の終了前の確認を示す（`previewInstallationClosure`）

- **シナリオ**: HOST-10
- **認可**: `operator`
- **コマンド**: `preview_closure`

### 概要

名簿のすべてのワークスペースについて、ワークスペースの終了前の確認と同じ内容を、データを変えずに返す。CMS が行わない消去は、[../domains/installation.md](../domains/installation.md) の「WorkspaceClosure」と「終了手順」が定める。

### 入出力

- **入力**: 無し
- **出力**: ワークスペースごとの `ClosurePreview` の列

### 使用するドメインの振る舞い・ポート

- 設置の状態の確認
- `WorkspaceDirectory.findPageByStatus`（すべての状態。最後のページまで）
- 名簿のワークスペースごとに、「ワークスペースの終了前の確認を示す」と同じ読み取りと `WorkspaceClosure.preview`

### トランザクション境界

- **UnitOfWork の要否**: 不要。書き込みが無い
