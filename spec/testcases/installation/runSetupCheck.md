# 導入時の動作確認を行う（`runSetupCheck`）

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-installation-runSetupCheck-01 | `operator`。保存先、メディアの保管先、設置全体のメール送信が動く設置。資源は足りている | 確認メールの宛先を指定して動作確認を行う | 保存先、メディアの保管先、メール送信がどれも `passed`、資源はどれも `sufficient` の報告を返す。指定した宛先に確認メールを送る。確認用の値と実体は残らず、設置のデータは変わらない |
| TC-installation-runSetupCheck-02 | `operator`。メディアの保管先への保存、または設置全体のメール送信が失敗する設置 | 動作確認を行う | エラーにせず、失敗した項目を `failed` と理由で報告する。理由は認証情報を含まない |
| TC-installation-runSetupCheck-04 | `operator`。設置全体のメール送信を宣言していない設置。ほかの項目は動く | 動作確認を行う | メール送信を `not_configured` と報告する。ほかの項目は `passed` |
| TC-installation-runSetupCheck-05 | `operator`。データの保存容量の使用量が上限に達している設置 | 動作確認を行う | その資源を `short` と報告する |
| TC-installation-runSetupCheck-06 | `operator`。資源の一つの状態を取得できない設置 | 動作確認を行う | その資源を `unknown` と報告し、不足なしと判定しない |
