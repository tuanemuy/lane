# spec

型安全なヘッドレス CMS・フォーム受付基盤の設計。要件は [requirements.md](requirements.md) が定める。

独立した読みの予算で止めたときに残った反例と、実装へ渡す申し送りは [handoff.md](handoff.md) が持つ。

## シナリオ

利用者の操作とフロー、異常系。

- [シナリオ一覧](scenario/index.md)（権限、共通の規則、未対処の失敗の規則を含む）
- [設置の運用](scenario/hosting.md)
- [スキーマの開発](scenario/schema-development.md)
- [スキーマの変更と移行](scenario/schema-migration.md)
- [アプリの連携](scenario/app-integration.md)
- [コンテンツの編集](scenario/content-editing.md)
- [公開と削除](scenario/publishing.md)
- [メディア](scenario/media.md)
- [フォームの運用](scenario/form-operation.md)
- [ワークスペースの管理](scenario/workspace-admin.md)
- [認証](scenario/authentication.md)

## ページ

- [画面構成](pages/index.md)

## ドメイン

- [ドメイン一覧](domains/index.md)（境界、依存方向、集約、ドメインをまたぐ規則、イベント、定期ジョブ）
- [共通の型とポート](domains/common.md)
- [Identity](domains/identity.md)
- [Workspace](domains/workspace.md)
- [Audit](domains/audit.md)
- [Token](domains/token.md)
- [Schema](domains/schema.md)
- [Media](domains/media.md)
- [Content](domains/content.md)
- [Form](domains/form.md)
- [Webhook](domains/webhook.md)
- [Installation](domains/installation.md)

## ユースケース

- [Identity](usecases/identity.md)
- [Workspace](usecases/workspace.md)
- [Audit](usecases/audit.md)
- [Token](usecases/token.md)
- [Schema](usecases/schema.md)
- [Media](usecases/media.md)
- [Content](usecases/content.md)
- [Form](usecases/form.md)
- [Webhook](usecases/webhook.md)
- [Installation](usecases/installation.md)
- [アプリ向けの HTTP API と `packages/client`](usecases/app-api.md)

## 動的フロー

- [動的フロー台帳](flows/index.md)

## テストケース

規則はドメイン、例はテストケース、配線はユースケースが持つ。ユースケースの振る舞いの結果（結果の種類ごとの条件、要求が振る舞いを定めるエラー、途中で止まったときに残る状態とその後の扱い）は、ユースケーステストの行だけが定める。表は `| ID | 前提条件 | 操作 | 期待結果 |` で、ID は `TC-{ディレクトリ名}-{ファイル名}-{連番}`。実装したテストは、名前に ID を含める。

ユースケーステストは、ユースケースごとに 1 ファイル。ユースケースを通らない振る舞いのテストは、下の括弧書きのとおりに置く。

- [共通の規則](testcases/common/)（[domains/index.md](domains/index.md) と [domains/common.md](domains/common.md) が定める、ドメインをまたぐ規則の代表の例。規則ごとに 1 ファイル。各ユースケーステストは、この展開を持たない）
- [Identity](testcases/identity/)
- [Workspace](testcases/workspace/)
- [Audit](testcases/audit/)
- [Token](testcases/token/)
- [Schema](testcases/schema/)（ユースケースごとのファイルに加えて、[domains/schema.md](domains/schema.md) の「WriteScope とスキーマ」の例を [stableSchemaWhileApplying](testcases/schema/stableSchemaWhileApplying.md) に置く）
- [Media](testcases/media/)
- [Content](testcases/content/)（ユースケースごとのファイルに加えて、ユースケースを通らないドメインの関数のテストを、関数の名前（`ContentMigration.accumulate` の形）のファイルに置く）
- [Form](testcases/form/)
- [Webhook](testcases/webhook/)
- [Installation](testcases/installation/)
- [`packages/client`](testcases/client/)（ユースケースを通らない、クライアントの振る舞いと型の導出。関数ごとに 1 ファイル）
- [アプリ向けの HTTP API の入口](testcases/app-api/)（ユースケースを通らない、入口の規則。エラーの形、利用できないワークスペースの応答、フォームの受付結果、メディアの配信の応答、`apps/cli` がスキーマの反映の要求に添える値）

ポート適合テストは、ポートごとに 1 ファイル。ポートの契約の全項目の例を持ち、すべてのアダプター実装が共通で通す。

- [ポート適合テスト](testcases/ports/)
