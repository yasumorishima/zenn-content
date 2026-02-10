---
title: "pybaseballにない「日付範囲指定」を可能にするライブラリを作った"
emoji: "⚾"
type: "tech"
topics: ["python", "pypi", "baseball", "mlb", "pybaseball"]
published: true
---

## はじめに

[pybaseball](https://github.com/jldbc/pybaseball) は、Baseball Savantのデータをpipで簡単に取得できるPythonライブラリです。Statcastのピッチレベルデータからリーダーボードまで、MLB分析の定番ツールとして広く使われています。

ただし、pybaseballのリーダーボード系関数には共通の制限があります。

**年単位（シーズン全体）でしかデータを取得できない**ということです。

```python
# pybaseball: シーズン全体のみ
from pybaseball import statcast_batter_bat_tracking

df = statcast_batter_bat_tracking(year=2024)  # 2024シーズン全体
# → 4月だけ、前半戦だけ、といった指定はできない
```

「4月と9月でバットスピードはどう変わったのか？」「前半戦と後半戦で打者のスイング傾向はどう違うのか？」といった分析をしたい場合、pybaseballだけでは対応できません。

この問題を解決するために、[savant-extras](https://github.com/yasumorishima/savant-extras) というライブラリを作りました。

## savant-extras とは

Baseball Savantのバットトラッキング・リーダーボードを**任意の日付範囲**で取得できるPythonライブラリです。

pybaseballの補完として設計しており、pybaseballが対応していない日付範囲指定に特化しています。

バットトラッキングデータ（Hawk-Eye）は2024シーズンから利用可能です。

## インストール

```bash
pip install savant-extras
```

## 基本的な使い方

### 1. `bat_tracking()` — 日付範囲を指定してデータ取得

```python
from savant_extras import bat_tracking

# 2024年4月のバッターデータ
df = bat_tracking("2024-04-01", "2024-04-30")
print(df[["name", "avg_bat_speed", "attack_angle", "competitive_swings"]].head(5))
```

出力:

```
                 name  avg_bat_speed  attack_angle  competitive_swings
0  Stanton, Giancarlo      80.776081      7.886978                 135
1         Cruz, Oneil      77.538084      9.574513                 127
2    Robert Jr., Luis      77.198599     11.955779                  12
3     Schwarber, Kyle      77.093025     14.716905                 167
4       Chapman, Matt      77.088279      5.304906                 153
```

ピッチャー視点のデータも取得できます:

```python
# 2024年4〜6月のピッチャー被バットトラッキング
df = bat_tracking("2024-04-01", "2024-06-30", player_type="pitcher")
print(df[["name", "avg_bat_speed", "competitive_swings"]].head(5))
```

出力:

```
              name  avg_bat_speed  competitive_swings
0      Buttó, José      72.870642                 254
1        Sears, JP      72.681473                 577
2     Snell, Blake      72.646426                 182
3  Walker, Taijuan      72.646226                 298
4    Pérez, Martín      72.557050                 402
```

### 2. `bat_tracking_monthly()` — 月別スプリットの取得

シーズン中の月別データ（4月〜10月）を一括取得します。返されるDataFrameには `month` カラムが追加されます。

```python
from savant_extras import bat_tracking_monthly

df = bat_tracking_monthly(2024, min_swings=1)

# Stantonの月別バットスピード推移
stanton = df[df["name"] == "Stanton, Giancarlo"]
print(stanton[["month", "avg_bat_speed", "competitive_swings"]])
```

出力:

```
   month  avg_bat_speed  competitive_swings
       4      80.776081                 135
       5      80.548995                 148
       6      81.011443                 137
       7      82.736527                  18
       8      81.483215                 133
       9      82.096844                 143
```

Stantonは7月以降にバットスピードが上昇し、9月には82.1 mph に達していることが分かります。

全打者の月別平均を出すこともできます:

```python
print(df.groupby("month")["avg_bat_speed"].mean())
```

### 3. `bat_tracking_splits()` — 前半戦/後半戦の比較

オールスターブレイク前後（7月13日/14日で分割）のデータを取得します。

```python
from savant_extras import bat_tracking_splits

splits = bat_tracking_splits(2024)
first = splits["first_half"]
second = splits["second_half"]

# Stantonの前半/後半比較
s1 = first[first["name"] == "Stanton, Giancarlo"]
s2 = second[second["name"] == "Stanton, Giancarlo"]

print(f"前半戦: {s1['avg_bat_speed'].values[0]:.1f} mph ({s1['competitive_swings'].values[0]:.0f} swings)")
print(f"後半戦: {s2['avg_bat_speed'].values[0]:.1f} mph ({s2['competitive_swings'].values[0]:.0f} swings)")
```

出力:

```
前半戦: 80.8 mph (420 swings)
後半戦: 81.9 mph (294 swings)
```

Stantonは後半戦でバットスピードが約1.1 mph上昇しています。

## ユースケース

savant-extrasの日付範囲指定は、以下のような分析に活用できます。

### スランプ/ホットストリーク分析

特定の期間（例: 6月の不振期）だけのバットトラッキングデータを取得し、好調時と比較:

```python
# 好調期 vs 不振期
hot = bat_tracking("2024-05-01", "2024-05-31")
cold = bat_tracking("2024-08-01", "2024-08-31")
```

### 移籍前後の比較

トレードデッドライン（7月30日前後）でのパフォーマンス変化:

```python
before_trade = bat_tracking("2024-04-01", "2024-07-29")
after_trade = bat_tracking("2024-07-30", "2024-09-30")
```

### シーズン序盤 vs 終盤

開幕直後の状態と、シーズン終盤のパフォーマンス比較:

```python
early = bat_tracking("2024-04-01", "2024-04-30")
late = bat_tracking("2024-09-01", "2024-09-30")
```

## pybaseball との比較

| | pybaseball | savant-extras |
|---|---|---|
| シーズン全体のデータ | ✅ | ✅ |
| 月別スプリット | ❌ | ✅ |
| 前半戦/後半戦 | ❌ | ✅ |
| 任意の日付範囲 | ❌ | ✅ |
| 移籍前後の比較 | ❌ | ✅ |

savant-extrasはpybaseballの代替ではなく、**補完**です。ピッチレベルのStatcastデータには引き続きpybaseballの `statcast()` を使い、リーダーボードの日付範囲分析にsavant-extrasを使う、という使い分けを想定しています。

## 技術メモ

- Baseball Savantの公開CSVエンドポイント（pybaseballと同じ）を利用
- 2024年以前の日付を指定するとwarningが出ます（Hawk-Eyeデータは2024年以降のみ）
- `bat_tracking_monthly()` はサーバーへの負荷を考慮し、リクエスト間に1秒のsleepを入れています
- HTMLレスポンスや空レスポンスが返った場合は空のDataFrameを返します

## リンク

- **PyPI**: https://pypi.org/project/savant-extras/
- **GitHub**: https://github.com/yasumorishima/savant-extras
- **pybaseball**: https://github.com/jldbc/pybaseball
