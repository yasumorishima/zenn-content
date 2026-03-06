---
title: "Statcastデータで選手成績予測の精度は上がるか — Marcel法との比較"
emoji: "📊"
type: "tech"
topics: ["baseball", "python", "機械学習", "mlops", "データ分析"]
published: true
---

## はじめに

このシリーズではNPBベイズ順位予測を積み重ねてきました。前回の記事：

https://zenn.dev/shogaku/articles/npb-bayes-pf-validation

シリーズの途中で「**Statcastのような打球品質データがなければ、次の壁は越えられない**」という結論を出しました。

https://zenn.dev/shogaku/articles/npb-bayes-projection-story

今回は舞台をMLBに移して、それを検証します。MLB Statcastのトラッキングデータ（打球速度・バレル率・Whiff率等）を使い、Marcel法を上回る成績予測モデルを作れるかを試しました。

> **GitHub**: https://github.com/yasumorishima/baseball-mlops
> **Streamlit**: https://baseball-mlops.streamlit.app/

---

## はじめての方へ：この記事で登場する用語

| 用語 | 意味 |
|---|---|
| **Marcel法** | 過去3年の成績を加重平均して翌年を予測するシンプルな手法。重みは直近ほど高い（5:4:3） |
| **Statcast** | MLBが全球場に設置したトラッキングシステム。打球速度・回転数・スプリント速度など多数の指標を計測 |
| **wOBA** | 重み付き出塁率。四球・単打・二塁打・本塁打などに異なる重みをつけた打撃総合指標 |
| **xFIP** | 守備と運の影響を除いた投手の実力指標。K%/BB%/被HR率から計算 |
| **xwOBA** | 実際の結果ではなく打球の質（速度・角度）から計算したwOBA |
| **バレル率** | 最も安打になりやすい打球速度・角度の組み合わせに入る打球の割合 |
| **Stuff+** | 球速・変化・回転数を総合評価した球質指標。100が平均 |
| **Whiff%** | スイングした時の空振り率 |
| **MAE** | 平均絶対誤差。小さいほど精度が高い |
| **Strict Holdout** | ハイパーパラメータ調整に一切使っていないデータでの評価 |

---

## NPBシリーズで見えたこと

NPBシリーズでは、Marcel法にベイズ回帰（Stan/Ridge）を加えて精度向上を試みました。

結果は「選手レベルでは一貫した改善（打者 p=0.06）、チームレベルでは改善しない」というものでした。原因はPA加重の構造的問題です。高PA（レギュラー）選手に対しては、Marcel法が3年加重平均で既に十分な精度を出しており、K%/BB%/BABIPを追加してもマージンが残っていませんでした。

NPBの公開データには限界があります。「どれだけ良い打球を打てるか」「球質はどうか」という実力指標が集計統計では取れないのです。一方MLBには**Statcast**があります。

---

## 手法

### データ
- **ソース**: pybaseball（FanGraphs + Baseball Savant）
- **対象**: MLB打者（PA≥100）/ 投手（IP≥30）
- **期間**: 2015-2024（学習）、2025（評価）

### 打者の特徴量（38個）

| カテゴリ | 特徴量 |
|---|---|
| Statcast | EV、バレル率、xwOBA、スプリント速度、打球角度、ev95%等 |
| FanGraphs | HardHit%、Contact%、O-Swing%、SwStr% |
| 1年トレンド | wOBA変化量、xwOBA変化量、K%変化量、BB%変化量、バレル率変化量 |
| **2年トレンド（v7）** | **wOBA 2年分の方向性（上昇/下降傾向）** |
| 工学特徴量（v7） | **age_from_peak**（ピーク年齢29歳からの距離）、**park_factor**、**team_changed**（移籍フラグ）、pa_rate |
| 交互作用 | age × xwOBA-wOBA乖離（運の年齢感度） |
| スタッキング | lgb_delta（LightGBM OOFの残差） |

### 投手の特徴量（35個）

| カテゴリ | 特徴量 |
|---|---|
| Statcast | K%、BB%、Whiff%、CSW%、SwStr%、バレル率、EV等 |
| 球質 | Stuff+、Location+、Pitching+、球速、回転数 |
| 1年トレンド | xFIP変化量、K%変化量、BB%変化量、K-BB%変化量 |
| **2年トレンド（v7）** | **xFIP 2年分の方向性** |
| 工学特徴量（v7） | **age_from_peak**、**park_factor**、**team_changed**、ip_rate、FIP-ERA乖離 |
| 交互作用 | age × K-BB% |
| スタッキング | lgb_delta |

NPBシリーズで取り組んだ球場補正（park_factor）の知見は、baseball-mlopsにも `park_factor` 特徴量として組み込まれています。

### モデル構成

3つのモデルを組み合わせます。

1. **Marcel法**（ベースライン）: 過去3年加重平均 + 平均回帰 + 年齢調整
2. **LightGBM**: Optuna 1000トライアルで最適化（時系列 expanding-window CV）
3. **Bayes補正（ElasticNet）**: Marcelの残差をStatcast特徴量で予測、80% CI付与
   - Recency Decay: 近年サンプルを0.85/年で重み付け
   - LGBのOOF予測を追加特徴量（スタッキング）として使用
4. **アンサンブル**: Marcel×31% + LightGBM×33% + Bayes×36%（逆MAE比で自動算出）

### バックテスト設計

2025年はStrict Holdout — Optuna/CVに一切使わず、最後に1回だけ評価します。

```
2015-2019: 初期学習データ
2020-2024: 時系列 expanding-window CV（Optuna最適化）
2025:      Strict Holdout（ハイパーパラメータ調整に未使用）
```

---

## 結果

### 2025 Strict Holdout

| | Marcel MAE | ML MAE | 改善率 |
|---|---|---|---|
| 打者 wOBA | 0.0331 | **0.0291** | **+12.1%** |
| 投手 xFIP | 0.5038 | **0.4837** | **+4.0%** |

Strict Holdoutのため、この結果は過学習の心配がありません。CV結果（打者 0.0281 / 投手 0.521）とも乖離がなく、安定した改善と判断できます。

### 年別バックテスト（ML vs Marcel）

| 年 | 打者 ML MAE | Marcel MAE | | 投手 ML MAE | Marcel MAE | |
|---|---|---|---|---|---|---|
| 2020 | 0.0359 | 0.0371 | ✓ +3.2% | 0.595 | 0.618 | ✓ +3.7% |
| 2021 | 0.0293 | 0.0317 | ✓ +7.6% | 0.542 | 0.553 | ✓ +1.9% |
| 2022 | 0.0296 | 0.0330 | ✓ +10.3% | 0.578 | 0.569 | ✗ -1.5% |
| 2023 | 0.0277 | 0.0303 | ✓ +8.7% | 0.535 | 0.559 | ✓ +4.3% |
| 2024 | 0.0280 | 0.0333 | ✓ +16.0% | 0.509 | 0.522 | ✓ +2.5% |
| **2025** | **0.0291** | **0.0331** | **✓ +12.1%** | **0.484** | **0.504** | **✓ +4.0%** |

打者はCV全5年 + holdout で6/6勝。投手の2022年敗北は、学習データがCOVID短縮の2020-2021の2年しかない年で、モデルが不安定だったと考えられます。

---

## なぜStatcastが効くのか

Bayes（ElasticNet）モデルは「Marcelの予測残差」をStatcast特徴量で予測します。係数が大きい特徴量ほど、Marcelが見落としている情報を持っています。

### 打者（Marcel残差に効く特徴量）

| 特徴量 | 係数 | 解釈 |
|---|---|---|
| 最高打球速度（maxEV） | +0.0046 | 最も力強い打球を打てる能力はMarcelが捉えられない |
| コンタクト率 | +0.0040 | 空振りの少なさはK%より細かい実力指標 |
| BB%（四球率） | +0.0038 | 選球眼に関する追加情報 |
| xwOBA | +0.0037 | 運を除いた「本来の打撃力」 |

### 投手（Marcel残差に効く特徴量）

| 特徴量 | 係数 | 解釈 |
|---|---|---|
| Pitching+（球質総合） | -0.0892 | 高いほど翌年xFIPが低い（改善方向） |
| K%（三振率） | -0.0631 | 高K%の投手はMarcel予測より良い結果を出す傾向 |
| SwStr%（空振り率） | -0.0346 | 打者を空振りさせる能力 |
| Stuff+（球質） | -0.0279 | 球速・変化・回転数の総合評価 |

Marcel法が使う「ERA/xFIP」には運の成分が混入します。**Statcastの球質指標（Stuff+/Pitching+）はその運成分を除いた実力を反映しているため、翌年予測に追加情報を持つ**のだと考えられます。

---

## MLOpsパイプライン

精度が上がっても、モデルが陳腐化すれば意味がありません。毎週自動更新される仕組みを作りました。

```
毎週月曜 JST 11:00（GitHub Actions cron）
  ↓
fetch_statcast.py（pybaseball → Statcast CSV取得）
  ↓
train.py（LightGBM + Optuna 1000trial + Bayes補正）
  ↓
W&B Model Registry（MAE比較 → production タグ自動昇格）
  ↓
FastAPI（6時間ごとにW&Bから最新モデルを自動ロード）
```

### W&B Model Registry

W&Bにモデルを保存し、MAEが前回を下回ったときだけ `production` タグが昇格します。FastAPIはこのタグを参照するため、**コンテナ再起動なしで最新モデルが自動反映**されます。

---

## NPB Hawk-Eyeへの展望

NPBは2024年に全12球場へHawk-Eye（トラッキングシステム）を設置しました。2026年以降にデータが公開されれば、このパイプラインをそのまま移植できます。

| baseball-mlops | NPB Hawk-Eye対応版 |
|---|---|
| pybaseball | NPB Hawk-Eye API |
| EV / バレル率 / xwOBA | 同等の指標（公開形式未定） |
| Marcel（MLB版） | Marcel（NPB版）|
| LightGBM + Bayes | 同じアーキテクチャ |

---

## まとめ

| | NPBベイズ予測 | baseball-mlops（MLB） |
|---|---|---|
| データ | K%/BB%/BABIP（公開統計） | Statcast（トラッキング） |
| Marcel比改善（選手レベル） | 微小（p=0.06） | **+12.1%（打者）/ +4.0%（投手）** |
| 年別勝率 | — | 打者 6/6、投手 5/6 |

NPBシリーズで見えた「公開統計の限界」は、MLBのトラッキングデータによって超えることができました。

Statcastが効く理由は、**Marcel法が3年加重平均で捉えられない「打球の質」「球質」をトラッキングデータが直接計測しているから**です。K%/BB%といった集計値では拾えない情報が、EV・バレル率・Stuff+に含まれています。

---

## 関連記事

- [球場補正を加えたらNPB予測は改善したか](https://zenn.dev/shogaku/articles/npb-bayes-pf-validation)（前回）
- [Marcel法の限界を超えたい — NPBベイズ回帰15ステップの記録](https://zenn.dev/shogaku/articles/npb-bayes-projection-story)

データ提供: [Baseball Savant](https://baseballsavant.mlb.com/) / [FanGraphs](https://www.fangraphs.com/)（via pybaseball）
