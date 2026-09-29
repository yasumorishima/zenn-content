---
title: "球種の成績は翌年も続くのか：MLB 8,022 組で空振り率と run value を比べた"
emoji: "⚾"
type: "idea"
topics: ["baseball", "mlb", "statcast", "dbt", "duckdb"]
published: true
---

Baseball Savant で球種の成績を見ていて、「去年良かった球種は、今年も良いのか」が気になったので、MLB の公開データで調べてみました。野球の現場の経験者ではなく、公開データを集計しただけです。

## 空振り率は翌年も続き、run value は続きにくい

![空振り率は翌年も続き、run value は続きにくい](https://raw.githubusercontent.com/yasumorishima/mlb-data-pipeline/master/docs/images/pitch_carryover_1_ja.png)

同じ投手・同じ球種の、2 年続けての成績を比べました（2017〜2025 年、8,022 組）。

## 上位の球種が、翌年も上位に残る割合

![上位 20% の球種が、翌年も上位 20% に残る割合](https://raw.githubusercontent.com/yasumorishima/mlb-data-pipeline/master/docs/images/pitch_carryover_2_ja.png)

## run value の内訳

1 球ごとのデータ（約 598 万球）から run value を計算し直し、Savant の公表値と 99% 以上の行で一致することを確かめてから、5 つに分けました。

![run value の内訳](https://raw.githubusercontent.com/yasumorishima/mlb-data-pipeline/master/docs/images/pitch_carryover_3_ja.png)

![打球の運と状況は、翌年にほぼ続かない](https://raw.githubusercontent.com/yasumorishima/mlb-data-pipeline/master/docs/images/pitch_carryover_4_ja.png)

## 運と状況を除くと、翌年がよく当たる

![運と状況を除くと、翌年の run value がよく当たる](https://raw.githubusercontent.com/yasumorishima/mlb-data-pipeline/master/docs/images/pitch_carryover_5_ja.png)

## まとめ

- 空振り率や三振・四死球は、翌年に続きやすい
- 1 年の run value には、翌年に続かない「打球の運」と「走者・アウトの状況」が 3 割ほど入っている

1 年分の run value は、空振り率などと一緒に見ると翌年を考えやすそう、というのが個人的な感想です。

## 付録

**用語**

- 空振り率：スイングした球のうち空振りの割合
- 被 xwOBA：打たれた打球の質（主に打球の速さと角度）から見た、打たれ方の期待値
- run value：1 球ごとに、その球で得点の見込みがどれだけ動いたかを足したもの（プラスほど投手に有利）
- 打球の運：打球の質から見込まれる値と、実際の結果（ヒットかアウトか）の差

**調べ方**

- 球種ごとの平均を引いてから比べています（スライダー同士、カーブ同士で比べるため）
- 帯は、2 シーズンのうち少ないほうの球数で分けています
- 「翌年を当てる」は、2 年目が 2021 年までの組で式を作り、2022 年以降の組で確かめました。0.29 と 0.35 の差の 95% 区間は 0.03〜0.10 です
- 2020 年（短縮シーズン）を含みます。翌年に投げなくなった投手は入っていません

**集計**：https://github.com/yasumorishima/mlb-data-pipeline
