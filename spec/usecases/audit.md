# Audit のユースケース

操作履歴の確認を実現する。型・振る舞い・ポートは [../domains/audit.md](../domains/audit.md) が定める。見出しの識別子は、`spec/testcases/audit/` のファイル名と同じ。

操作履歴の追記は、操作履歴に残す操作のユースケースが行う（[../domains/audit.md](../domains/audit.md) の「トランザクション境界」）。

## 共通

- 認可は [../domains/index.md](../domains/index.md) の「認可」に従う。各ユースケースは、必要な権限と、受け付ける実行者の種類だけを書く
- 区画: 操作履歴と、対象が現在もあるかを確かめる集約は、どれも入力のワークスペースの区画にある。その区画のリポジトリを `PartitionReader` で読む

## 操作履歴を一覧する（`listAuditLog`）

ワークスペースの操作履歴を、条件で絞り込んで返す。

- **シナリオ**: WS-11
- **認可**: 実行者は `member` だけ。`workspace:manage`
- **入力**: `Actor`、ワークスペースの ID、絞り込みの条件（`AuditLogFilter`）、ページの指定
- **出力**: 記録の列と、条件に該当する総件数。記録ごとに、実行者（`AuditActor`）を持つ
- **使用するドメインの振る舞い・ポート**: `AuditLogFilter.create`、`AuditLogRepository.findPage`
- **トランザクション境界**: UnitOfWork は不要。書き込みがない

## 操作履歴の一件を確かめる（`getAuditLogEntry`）

記録一件の変更の内容と、対象が現在もあるかを返す。

- **シナリオ**: WS-11
- **認可**: 実行者は `member` だけ。`workspace:manage`
- **入力**: `Actor`、ワークスペースの ID、記録の ID
- **出力**: 変更の内容を含む記録と、対象が現在もあるかどうか
- **使用するドメインの振る舞い・ポート**: `AuditLogRepository.findById`。対象が現在もあるかは、`operation.target` の種類ごとに、対象を所有するドメインの次の読み取りと判定で求める
  - `member`: `MemberRepository.findByIds`（Workspace）
  - `invitation`: `InvitationRepository.findById`、`Clock.now`、`Invitation.standing`（Workspace）
  - `token`: `TokenRepository.findById`、`Clock.now`、`Token.standing`（Token）
  - `form`: `WorkspaceSchemaRepository.find`（Schema）
  - `webhook_endpoint`: `WebhookEndpointsRepository.find`、`WebhookEndpoints.find`（Webhook）
  - `workspace`、`schema`: 何も読まない
- **トランザクション境界**: UnitOfWork は不要。書き込みがない
