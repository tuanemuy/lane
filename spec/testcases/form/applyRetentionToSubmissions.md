# 保持方針を既存の回答へ適用する

断りが無い限り、「F へ適用する」は、F の現在の保持方針（「保持方針の既存の回答への適用を確認する」が返した値）を、確認した保持方針として渡すことを指す。

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-applyRetentionToSubmissions-01 | F の保持方針は 30 日。F の回答は三件。S1 は保持方針 365 日で 10 日前に受付、S2 は保持方針 365 日で 40 日前に受付、S3 は保持方針 30 日で 5 日前に受付。別のフォーム G に、保持方針 365 日の回答 S4 がある | `intake:manage` を持つメンバーが、確認した保持方針を 30 日として F へ適用する | `applied` と、置き換えた件数 2 を返す。S1、S2、S3 の `retention` は 30 日になる。S2 を含め、どの回答も削除しない。S4 の `retention` は 365 日のまま |
| TC-form-applyRetentionToSubmissions-02 | F の保持方針は 30 日。保持方針 365 日の回答 S の通知は送信中（`sending`） | `intake:manage` を持つメンバーが、F へ適用する | S の `retention` は 30 日になる。S の通知は変わらない |
| TC-form-applyRetentionToSubmissions-03 | F の保持方針は 30 日。保持方針 365 日の回答がある。前の適用が途中で失敗または中断した（最初の UnitOfWork の確定が `SystemError` で失敗し、どの回答も 365 日のままの場合と、一部の回答だけが 30 日になった場合） | `intake:manage` を持つメンバーが、同じ保持方針（30 日）で F へ再び適用する | `applied` を返す。残りの回答の `retention` が 30 日になり、置き換えた件数を返す。F の保持方針は 30 日のまま |
| TC-form-applyRetentionToSubmissions-04 | F の回答が無い | `intake:manage` を持つメンバーが、F へ適用する | `applied` と、置き換えた件数 0 を返す |
| TC-form-applyRetentionToSubmissions-06 | F の保持方針は 30 日。保持方針 365 日の回答がある。メンバーが 30 日での適用を確認した後、ほかのメンバーが F の保持方針を 60 日に変更した | `intake:manage` を持つメンバーが、確認した保持方針を 30 日として F へ適用する | エラーにしない。`policyChanged` と、現在の保持方針（60 日）、置き換えた件数 0 を返す。どの回答の `retention` も変わらない。30 日も 60 日も適用しない |
| TC-form-applyRetentionToSubmissions-07 | 削除済みフォーム F の `removed` の設定。F の回答が残る | `intake:manage` を持つメンバーが、F へ適用する | `BusinessRuleError`（`FormRemoved`）。F の回答の `retention` は変わらない |
| TC-form-applyRetentionToSubmissions-09 | F の保持方針は 30 日。保持方針 365 日の回答がある | `intake:manage` を持たないメンバーが、F へ適用する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `intake:manage`。どの回答の `retention` も変わらない |
| TC-form-applyRetentionToSubmissions-11 | F の保持方針は 30 日。保持方針 365 日の回答が `FormSweep.maxLimit` 件より多くある。適用が一部の回答を置き換えた後、残りを置き換える前に、スキーマの反映で F が削除される | `intake:manage` を持つメンバーが、F へ適用する。続けて、同じ操作をやり直す | どちらも `BusinessRuleError`（`FormRemoved`）。置き換えた回答は 30 日のまま残り、残りの回答の `retention` は 365 日のまま変わらない。残りの回答は、365 日の保持期限で自動整理の対象になる |
| TC-form-applyRetentionToSubmissions-12 | F の保持方針は 30 日。保持方針 365 日の回答が `FormSweep.maxLimit` 件より多くある。適用が一部の回答を置き換えた後、残りを置き換える前に、ほかのメンバーが F の保持方針を 60 日に変更する | `intake:manage` を持つメンバーが、確認した保持方針を 30 日として F へ適用する | `policyChanged` と、現在の保持方針（60 日）、置き換えた件数を返す。置き換えた回答は 30 日のまま残り、残りの回答の `retention` は 365 日のまま。確認していない 60 日を、どの回答にも適用しない |
| TC-form-applyRetentionToSubmissions-13 | 前の行の適用が `policyChanged` で止まった | `intake:manage` を持つメンバーが、「保持方針の既存の回答への適用を確認する」で 60 日を確認し、確認した保持方針を 60 日として F へ適用する | `applied` を返す。F のすべての回答の `retention` は 60 日になる |
