---
title: "MLB Statcast + Bat Tracking データセット（2024-2025）をKaggleに公開した話"
emoji: "⚾"
type: "tech"
topics: ["kaggle", "baseball", "mlb", "python", "データ分析"]
published: true
---

## はじめに

MLBの2024-2025シーズンのStatcastデータにBat Tracking（バットトラッキング）メトリクスを組み合わせたデータセットをKaggleに公開しました。

https://www.kaggle.com/datasets/yasunorim/mlb-statcast-bat-tracking-2024-2025

## Bat Trackingとは

Bat Trackingは、MLBが2024年から本格導入した打撃メカニクスの測定技術です。高速カメラによって以下の指標が計測されます:

- **bat_speed**: バットスピード（mph）- インパクト時のバット速度
- **swing_length**: スイング軌道の長さ（feet）- 短いほど効率的
- **swing_path_tilt**: スイング角度（degrees）- アッパースイング/ダウンスイングの判定
- **attack_angle**: アタックアングル - 投球軌道に対するバットの角度
- **attack_direction**: アタック方向 - 引っ張り/流し打ちの判定

従来のStatcastでは「打球結果（exit velocity, launch angle等）」のみでしたが、Bat Trackingによって**スイングそのもの**を定量化できるようになりました。

## データセット概要

### 基本情報
- **データ期間**: 2024-2025レギュラーシーズン
- **行数**: 1,443,802行（約144万投球）
- **カラム数**: 118カラム
- **サイズ**: 808MB
- **Bat Trackingカバレッジ**: 約46.6%の投球（約67万投球）

### 主要カラム

**Bat Tracking系**
- bat_speed, swing_length, swing_path_tilt
- attack_angle, attack_direction
- intercept_ball_minus_batter_pos_x/y_inches

**Statcast系（従来）**
- release_speed, spin_rate, movement (pfx_x, pfx_z)
- launch_speed, launch_angle, hit_distance_sc
- estimated_woba, estimated_ba_using_speedangle
- pitch_type, zone, description, events

**ゲーム情報**
- game_date, game_type, inning, count (balls/strikes)
- player_name, batter, pitcher, stand, p_throws
- home/away score, win expectancy

全118カラムの詳細な説明はKaggleのColumn Descriptionに記載しています。

## データ取得方法

pybaseballライブラリの`statcast()`関数で取得できます。Bat Trackingメトリクスは自動的に含まれています。

```python
from pybaseball import statcast
import pandas as pd

# 2024シーズン
df_2024 = statcast(start_dt='2024-03-20', end_dt='2024-09-29')

# 2025シーズン
df_2025 = statcast(start_dt='2025-03-27', end_dt='2025-09-28')

# 結合
df = pd.concat([df_2024, df_2025], ignore_index=True)

# Bat Trackingデータの確認
print(f"Bat tracking coverage: {df['bat_speed'].notna().sum() / len(df) * 100:.1f}%")
```

データ生成ノートブックはGoogle Colabで実行可能です:
https://colab.research.google.com/github/yasumorishima/kaggle-datasets/blob/main/dataset4_statcast_bat_tracking/generate.ipynb

## 分析ノートブック

データセットを使った基本的な分析ノートブックも公開しています:

https://www.kaggle.com/code/yasunorim/mlb-bat-tracking-analysis-2024-2025

**分析内容:**
1. Bat Tracking カバレッジ分析
2. Bat Speed / Swing Length / Swing Path Tilt の分布
3. Bat Speed vs Launch Speed の相関
4. Top 10 Bat Speed Leaders（選手別）
5. Pitch Type別のBat Speed
6. Attack Angle/Direction ヒートマップ

## 活用方法

このデータセットで可能な分析例:

### 1. スイングメカニクス分析
- 効率的なスイング軌道（swing_length vs bat_speed）
- アッパースイングの効果（swing_path_tilt vs launch_angle）
- 最速バットスピードの選手ランキング

### 2. 投球との関係
- どの球種に対してbat_speedが上がるか
- カウント別のスイング傾向
- コースとattack_directionの関係

### 3. 打球結果との関係
- bat_speed → launch_speed の変換効率
- swing_lengthと打率の関係
- attack_angleとbarrel率の相関

### 4. 選手育成・スカウティング
- ドラフト候補のスイング効率評価
- 選手のスイング改善トラッキング
- 対戦相手の傾向分析

## 他のデータセットとの比較

これまで公開した野球データセット:

1. [日本人MLB選手 Statcast (2015-2025)](https://www.kaggle.com/datasets/yasunorim/japan-mlb-pitchers-batters-statcast)
   - 投手25名・打者10名の詳細データ

2. [MLB Bat Tracking Leaderboard (2024-2025)](https://www.kaggle.com/datasets/yasunorim/mlb-bat-tracking-2024-2025)
   - 選手別シーズン集計データ（452選手）

3. [MLB投手 Arsenal Evolution (2020-2025)](https://www.kaggle.com/datasets/yasunorim/mlb-pitcher-arsenal-2020-2025)
   - 投手の球種構成変化

**今回のデータセット（4つ目）の特徴:**
- **pitch-by-pitch（投球単位）** データ
- Bat Trackingメトリクスを含む最も詳細なデータ
- 2シーズン分の大規模データセット

## おわりに

Bat Trackingは、野球分析に新しい次元を加える技術です。従来「結果」しか見えなかったスイングを、「プロセス」として定量化できるようになりました。

このデータセットが、野球分析・機械学習・スポーツ科学の研究に役立てば幸いです。

## リンク

- **データセット**: https://www.kaggle.com/datasets/yasunorim/mlb-statcast-bat-tracking-2024-2025
- **分析ノートブック**: https://www.kaggle.com/code/yasunorim/mlb-bat-tracking-analysis-2024-2025
- **GitHubリポジトリ**: https://github.com/yasumorishima/kaggle-datasets
- **データ生成ノートブック**: [Google Colab](https://colab.research.google.com/github/yasumorishima/kaggle-datasets/blob/main/dataset4_statcast_bat_tracking/generate.ipynb)
