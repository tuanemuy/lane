# 移行定義を適用済みとして記録する（recordMigrationsAsApplied）

断りが無い限り、実行者は、メンバー A が発行した、スキーマ反映の許可（`schema_apply`）を持つトークン T。ワークスペースは `idle`、現行スキーマの版は 3、適用済みの移行定義は無く、現行スキーマの構造は直接反映で作られている。要求は前提とする版 3 を添える。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-schema-recordMigrationsAsApplied-01 | 手元の列は M1、M2。M1、M2 を順に適用した構造は、現行スキーマのコンテンツの構造と一致する。M2 は補完値と値の複写を持つ | 範囲の終端に M2 を指定して記録する | 記録した移行定義 M1、M2、記録後の版 4、対象の `WorkspaceId` を返す。適用済みの列は M1、M2（どちらも `recorded`）になり、版は 4 になる。現行スキーマ、モデルとフォームの ID は変わらず、ワークスペースは `idle` のまま。版 4 の記録（`recorded`、M1、M2、実行者 T と発行者 A、記録の時点の T の表示用の名称と A の名前）を追記する。操作履歴に `schema.migrations_marked_applied` を一件残す（実行者は T と A の名称、対象はスキーマ、名称は版 4）。公開内容、下書き、ごみ箱のコンテンツ、履歴の版は変わらず、補完値と値の複写は適用されない。ジョブを作らない |
| TC-schema-recordMigrationsAsApplied-02 | 手元の列は M1、M2、M3。M1、M2 を適用した構造が、現行の構造と一致する。M3 は項目を加える | M2 までを記録する | M1、M2 だけを記録し、版は 4 になる。M3 は未適用のまま |
| TC-schema-recordMigrationsAsApplied-03 | 上の記録の後。スキーマの構造は、M3 を適用した結果と一致する | 版 4 を前提に、M1、M2、M3 の列で反映を要求する | `SCHEMA_DRIFT` で拒否されず、受け付ける。受け付けたジョブの `plan.migrations` は M3 だけを持つ |
| TC-schema-recordMigrationsAsApplied-04 | 手元の列は M1、M2。M1 だけを適用した構造は、現行の構造と一致しない。M1、M2 を適用した構造は一致する | M1 までを記録する | `BusinessRuleError`（`SCHEMA_RECORD_MISMATCH`）。食い違う定義と、記録できる移行定義（M2）を添える。適用済みの列、版、版の記録、操作履歴は変わらない |
| TC-schema-recordMigrationsAsApplied-05 | 適用済みの移行定義は M1。手元の列は M1、M2 | 範囲の終端に M1 を指定して記録する | `BusinessRuleError`（`MIGRATION_NOT_PENDING`） |
| TC-schema-recordMigrationsAsApplied-07 | 手元の M2 が、その時点の構造に無いキーを指す | M2 までを記録する | `BusinessRuleError`（`MIGRATIONS_INVALID`）。該当する操作と理由を添える。何も記録しない |
| TC-schema-recordMigrationsAsApplied-08 | 適用済みの移行定義は M1。手元の M1 の操作の並びが、適用済みの M1 と異なる | M2 までを記録する | `BusinessRuleError`（`MIGRATION_SEQUENCE_DIVERGED`）。食い違う移行定義を添える。何も記録しない |
| TC-schema-recordMigrationsAsApplied-09 | ワークスペースは、反映で `applying` | M2 までを記録する | `UnavailableError`（`SCHEMA_APPLYING`）。何も記録しない |
| TC-schema-recordMigrationsAsApplied-10 | 要求が前提とする版は 2 | M2 までを記録する | `ConflictError`（`SCHEMA_VERSION_CHANGED`）。適用済みの列と版は変わらない |
| TC-schema-recordMigrationsAsApplied-11 | 記録の確定の前に、別の反映の要求が確定した | M2 までを記録する | `ConflictError`（`STALE_VERSION`）。適用済みの列、版、版の記録、操作履歴は変わらない |
| TC-schema-recordMigrationsAsApplied-12 | 実行者のトークンは `schema_apply` を持たない | M2 までを記録する | `ForbiddenError`（`GRANT_MISSING`）。必要な許可を示す。何も記録しない |
