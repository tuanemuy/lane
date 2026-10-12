# 止まった派生形式の生成をやり直す（`restartStalledDerivatives`）

| ID | 前提条件 | 操作 | 期待結果 |
|---|---|---|---|
| TC-media-restartStalledDerivatives-01 | ワークスペースに、派生形式 k が `generating` で、生成を始めてから `DerivativePolicy.stalledAfterMs` 以上、始め直しの回数が 0 の画像がある | そのワークスペースを指定して実行する | k は `generating` のまま、生成を始めた日時が現在時刻になり、始め直しの回数が 1 になる。生成の対象（定義と世代）と変更前の派生形式は変わらない。k を指す `media.derivative_regeneration_requested` を 1 件記録する。始め直した件数 1、失敗にした件数 0 を返す |
| TC-media-restartStalledDerivatives-02 | ワークスペースに、派生形式 k が `generating` で、生成を始めてから `DerivativePolicy.stalledAfterMs` 以上、始め直しの回数が `DerivativePolicy.maxRestarts` の画像がある | そのワークスペースを指定して実行する | k が `failed`（理由 `stalled`）になる。生成の対象と変更前の派生形式を引き継ぐ。イベントを記録しない。始め直した件数 0、失敗にした件数 1 を返す |
| TC-media-restartStalledDerivatives-03 | ワークスペースに、派生形式 k が `generating` で、生成を始めてから `DerivativePolicy.stalledAfterMs` 未満の画像がある。直前の起動が始め直した派生形式を含む | そのワークスペースを指定して実行する | 画像は変わらない。始め直しの回数は変わらず、イベントを記録しない。件数は 0 と 0 を返す |
| TC-media-restartStalledDerivatives-07 | ワークスペースに、止まった生成を持つ画像が 2 件ある。1 件の保存が、同じ画像の派生形式の記録へのほかの保存と競合する | そのワークスペースを指定して実行する | 競合しない画像の生成を始め直し、イベントを記録する。競合した画像は、読み直さずに飛ばす。止まったまま残り、次の起動が対象にする |
