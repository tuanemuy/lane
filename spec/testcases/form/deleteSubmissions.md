# 回答を削除する

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-form-deleteSubmissions-01 | 回答 S1、S2、S3 | `submission:delete` を持つメンバーが、S1 と S2 を削除する | S1 と S2 が、その通知とともに無くなり、S3 は残る。S1 と S2 は、「回答を一覧・検索する」と「回答を取得する」に現れない |
| TC-form-deleteSubmissions-02 | 削除済みフォームの回答 S1 | `submission:delete` を持つメンバーが、S1 を削除する | S1 が無くなる |
| TC-form-deleteSubmissions-03 | S1 の通知は送信失敗で、未対処。未対処の送信失敗の件数は 1 | `submission:delete` を持つメンバーが、S1 を削除する | 未対処の送信失敗の件数は 0 になる |
| TC-form-deleteSubmissions-04 | S1 の通知は最初の通知の送信待ち（`attempt` は `initial`、状態は `waiting`） | `submission:delete` を持つメンバーが、S1 を削除する。その後に、S1 の `form.submission_accepted` が消費され、定期ジョブ「止まった通知の検出」が実行される | S1 が無くなる。通知は始まらず、メールを送らない |
| TC-form-deleteSubmissions-05 | 回答 S1 がある。S9 は、ほかのメンバーの削除で既に無い。S7 は、メンバーが対象を確認した後、確定までに、自動整理が削除する | `submission:delete` を持つメンバーが、S1、S7、S9 を削除する | エラーにしない。削除の完了を返す。S7 と S9 を削除済みとして扱い、S1 が、その通知とともに無くなる |
| TC-form-deleteSubmissions-09 | 回答 S1 | `submission:read` だけを持つメンバーが、S1 を削除する | `ForbiddenError`（`PERMISSION_MISSING`）。不足する権限は `submission:delete`。S1 は残る |
