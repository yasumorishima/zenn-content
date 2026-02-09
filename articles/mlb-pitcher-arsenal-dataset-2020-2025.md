---
title: "MLB投手の球種構成データセット（2020-2025）をKaggleに公開した話"
emoji: "⚾"
type: "tech"
topics: ["kaggle", "baseball", "mlb", "python", "データ分析"]
published: true
published_at: 2026-02-09 12:00
---

# はじめに

MLB投手の球種構成（Arsenal）の年次変化を追跡できるデータセットをKaggleに公開しました。

https://www.kaggle.com/datasets/yasunorim/mlb-pitcher-arsenal-2020-2025

本記事では、このデータセットの内容と使い方を紹介します。

## データセットの概要

このデータセットは、2020年から2025年までの6シーズンにわたるMLB投手の球種構成の変化を記録したものです。

### 基本情報

- **期間**: 2020-2025シーズン（6シーズン）
- **行数**: 4,253行（投手×シーズンの組み合わせ）
- **列数**: 111カラム
- **データ形式**: Wide format（1行 = 1投手×1シーズン）
- **フィルタ条件**: 各シーズンで100球以上投球した投手のみ

### データソース

MLB Advanced Media（Statcast）のデータを[pybaseball](https://github.com/jldbc/pybaseball)ライブラリ経由で取得し、投手×シーズン×球種ごとに集計しています。

## データ構造

### 識別子カラム（3列）

- `player_id`: MLB選手ID
- `player_name`: 選手名
- `season`: シーズン年（2020-2025）

### 球種メトリクス（18球種 × 6メトリクス = 108列）

各球種について、以下の6つのメトリクスを記録：

1. **usage_pct**: 使用率（0-100%）
2. **avg_speed**: 平均球速（mph）
3. **avg_spin**: 平均回転数（rpm）
4. **whiff_rate**: 空振り率（0-1）
5. **avg_pfx_x**: 平均横変化量（インチ、重力補正済み）
6. **avg_pfx_z**: 平均縦変化量（インチ、重力補正済み）

### 対象球種（18種類）

FF（フォーシーム）、SI（シンカー）、FC（カッター）、SL（スライダー）、CU（カーブ）、CH（チェンジアップ）、FS（スプリット）、KC（ナックルカーブ）、FO（フォーク）、EP（イーファス）、KN（ナックルボール）、ST（スイーパー）、SV（スラーブ）など

## 使い方

### データのダウンロード

Kaggleデータセットページから直接CSVをダウンロードできます：

```python
import pandas as pd

# Kaggle APIでダウンロード
!kaggle datasets download -d yasunorim/mlb-pitcher-arsenal-2020-2025

# 読み込み
df = pd.read_csv('pitcher_arsenal_evolution_2020_2025.csv')
```

### 分析ノートブック

データの使い方を示す分析ノートブックも公開しています：

https://www.kaggle.com/code/yasunorim/pitcher-arsenal-analysis

以下の分析例が含まれています：

- 個別投手の球種トレンド分析
- MLB全体の球種トレンド（2020-2025）
- 球速変化の分析
- 相関ヒートマップ

## ユースケース

このデータセットは以下のような分析に活用できます：

### 1. 投手の球種トレンド分析

特定投手の球種構成の年次変化を追跡できます。例えば：

```python
# 菊池雄星の球種使用率推移
kikuchi = df[df['player_name'].str.contains('Kikuchi', case=False)]
kikuchi.plot(x='season', y=['SL_usage_pct', 'FF_usage_pct', 'CH_usage_pct'])
```

菊池投手の場合、スライダー使用率が2019年の約20%から2022-2025年には40%以上に増加している様子が見られます。

### 2. 怪我前後の変化検出

投手が怪我から復帰した際の球種構成や球速の変化を分析：

```python
# 特定投手の2シーズン比較
pitcher_data = df[df['player_id'] == 123456]
pitcher_data[['season', 'FF_avg_speed', 'FF_usage_pct']]
```

### 3. MLB全体のトレンド分析

リーグ全体でどの球種が増加/減少しているかを可視化：

```python
# 年次平均使用率
yearly_avg = df.groupby('season')[['FF_usage_pct', 'SI_usage_pct', 'SL_usage_pct']].mean()
yearly_avg.plot(kind='line', marker='o')
```

近年はスライダー・スイーパーの使用率が増加傾向にあります。

### 4. 機械学習の特徴量

投手パフォーマンス予測モデルの特徴量として利用：

```python
# 球種多様性の計算
df['pitch_diversity'] = (df[[col for col in df.columns if 'usage_pct' in col]] > 5).sum(axis=1)
```

## データ取得方法

データ取得用のノートブックも公開しています。同じ形式のデータを最新シーズンで再生成したい場合に利用できます：

https://www.kaggle.com/datasets/yasunorim/mlb-pitcher-arsenal-2020-2025?select=pitcher_arsenal_evolution_2020_2025.ipynb

## おわりに

このデータセットが、野球データ分析や機械学習の研究に役立てば幸いです。

質問やフィードバックがあれば、Kaggleのコメント欄やDiscussionでお気軽にお寄せください！

## 関連データセット

MLB関連のデータセットを他にも公開しています：

- [日本人MLB選手のStatcastデータ（2015-2025）](https://www.kaggle.com/datasets/yasunorim/japan-mlb-pitchers-batters-statcast)
- [MLBバットトラッキングデータ（2024-2025）](https://www.kaggle.com/datasets/yasunorim/mlb-bat-tracking-2024-2025)
