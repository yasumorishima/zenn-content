---
title: "球場係数・守備指標・Stuff+をPythonで取得する（savant-extras v0.3.2〜v0.4.1）"
emoji: "⚾"
type: "tech"
topics: ["python", "pypi", "baseball", "mlb", "savant"]
published: true
---

## はじめに

[pybaseball](https://github.com/jldbc/pybaseball) は Baseball Savant のデータを Python で扱う定番ライブラリですが、対応していないリーダーボードが多数あります。これらを補完するために作成した **[savant-extras](https://github.com/yasumorishima/savant-extras)** に、v0.3.2〜v0.4.1 で次の4つを追加しました。

| バージョン | 追加内容 |
|---|---|
| v0.3.2 | **park_factors** — FanGraphs 球場補正係数（30球団、2015+） |
| v0.3.3 | lxml 依存追加・StringIO fix（pandas 2.0+ 対応） |
| v0.4.0 | **outs_above_average** / **outfield_jump** — Baseball Savant 守備指標 |
| v0.4.1 | **pitcher_quality** — Stuff+ / Location+ / Pitching+（FanGraphs、2020+） |

いずれも pybaseball では取得できないデータです。

```bash
pip install savant-extras
```

---

## 1. park_factors — 球場補正係数（v0.3.2）

### 何が取れるか

FanGraphs の Park Factors テーブルから、MLB 全30球団の球場補正係数を取得します。100が中立、100超がヒッター有利、100未満がピッチャー有利です。

```python
from savant_extras import park_factors, park_factors_range

# 1シーズン分（30チーム）
df = park_factors(2024)
print(df.columns.tolist())
# ['season', 'team', 'pf_5yr', 'pf_3yr', 'pf_1yr', 'pf_hr', 'pf_1b', 'pf_2b', 'pf_3b', 'pf_so', 'pf_bb', 'pf_fip']

print(df[df["team"] == "COL"][["team", "pf_5yr", "pf_hr"]])
#    team  pf_5yr  pf_hr
# 5   COL     113    131

# 複数シーズン（機械学習の特徴量として使うなど）
df_multi = park_factors_range(2020, 2025)
print(df_multi.shape)  # (180, 12)  ← 6年 × 30チーム
```

### 主な列

| 列名 | 説明 |
|---|---|
| `pf_5yr` | 5年加重平均（最も安定、推奨） |
| `pf_3yr` | 3年加重平均 |
| `pf_1yr` | 当年のみ |
| `pf_hr` | 本塁打補正 |
| `pf_fip` | FIP計算用補正 |

### 活用例：予測モデルへの組み込み

前年の実績をそのまま使う場合、プレーヤーがコアーズフィールドでプレーしていたかどうかで数値が大きく変わります。`park_factors_range` で取得した辞書を使えば動的に補正できます。

```python
pf_lookup = {
    (int(r.season), str(r.team)): float(r.pf_5yr)
    for _, r in park_factors_range(2020, 2025).iterrows()
}
# 使い方: pf_lookup.get((2024, "COL"), 100)
```

---

## 2. outs_above_average — アウト貢献値（v0.4.0）

### 何が取れるか

Baseball Savant の **Outs Above Average (OAA)** は、守備機会ごとの「捕球期待値」を元に実際のアウト数との差分を集計した守備指標です。方向別（前後左右）の内訳も取得できます。

```python
from savant_extras import outs_above_average, outs_above_average_range

df = outs_above_average(2024)
print(df[["last_name", "pos", "outs_above_average", "fielding_runs_prevented"]].head(10))
```

### 主な列

| 列名 | 説明 |
|---|---|
| `outs_above_average` | 総合OAA |
| `outs_above_average_infront` | 前方向 |
| `outs_above_average_behind` | 後方向 |
| `outs_above_average_lateral_toward3bline` | 三塁方向 |
| `outs_above_average_lateral_toward1bline` | 一塁方向 |
| `fielding_runs_prevented` | 失点阻止推計（得点換算） |
| `actual_success_rate_formatted` | 実際の捕球成功率 |

```python
# 複数シーズン比較（内野手のみ）
df = outs_above_average_range(2022, 2024, pos="Infielder")
```

---

## 3. outfield_jump — 外野手の初動指標（v0.4.0）

### 何が取れるか

Baseball Savant の **Outfield Jump** は、打球に対する外野手の反応・加速・ルーティングを数値化した指標です。2016年以降のデータが利用可能です。

```python
from savant_extras import outfield_jump, outfield_jump_range

df = outfield_jump(2024)
print(df[["last_name", "rel_league_reaction_distance",
          "rel_league_burst_distance", "rel_league_bootup_distance"]].head(10))
```

### 主な列

| 列名 | 説明 |
|---|---|
| `rel_league_reaction_distance` | 最初の1歩（反応速度） |
| `rel_league_burst_distance` | 加速フェーズ（爆発力） |
| `rel_league_routing_distance` | ルーティング効率 |
| `rel_league_bootup_distance` | Jump 総合スコア |

値はリーグ平均からの差（フィート単位）。プラスが優秀です。

```python
# 2016〜2024 の経年変化
df = outfield_jump_range(2016, 2024)
```

---

## 4. pitcher_quality — Stuff+ / Location+ / Pitching+（v0.4.1）

### 何が取れるか

FanGraphs の **Stuff+** / **Location+** / **Pitching+** は、投球品質を 100 = MLB平均 で指数化した指標です。**pybaseball では取得不可**で、FanGraphs も CSV 書き出しが JavaScript 必須のため、通常のスクレイピングでは取得できません。

savant-extras では、FanGraphs のサーバーサイドレンダリング JSON（`__NEXT_DATA__`）を正規表現で抽出することで対応しました。

```python
from savant_extras import pitcher_quality, pitcher_quality_range

df = pitcher_quality(2024)
print(df.columns.tolist())
# ['name', 'team', 'season', 'age', 'ip', 'mlbam_id',
#  'stuff_plus', 'location_plus', 'pitching_plus',
#  'stuff_fa', 'stuff_si', 'stuff_fc', 'stuff_sl', 'stuff_cu', 'stuff_ch', ...]

# Stuff+ トップ10
print(df.sort_values("stuff_plus", ascending=False).head(10)[
    ["name", "team", "stuff_plus", "location_plus", "pitching_plus"]
])
```

### 主な列

| 列名 | 説明 |
|---|---|
| `stuff_plus` | 総合 Stuff+（球速・変化・回転） |
| `location_plus` | 総合 Location+（配球・コース） |
| `pitching_plus` | 総合 Pitching+（Stuff+Location+） |
| `stuff_fa` / `loc_fa` / `pit_fa` | 四シーム別の各指標 |
| `stuff_sl` / `loc_sl` / `pit_sl` | スライダー別の各指標 |
| `mlbam_id` | Statcast との結合キー |

球種別（FA/SI/FC/SL/CU/CH/ST/FS）のStuff+/Location+/Pitching+もすべて含まれます。

```python
# 3シーズンの経年比較
df_multi = pitcher_quality_range(2022, 2024)
df_multi.groupby("season")["stuff_plus"].mean()
# season
# 2022    99.8
# 2023   100.1
# 2024   100.0
```

### 活用例：機械学習の特徴量として

MLBAM IDを結合キーにして、Statcastデータと組み合わせるのが自然な使い方です。

```python
import pandas as pd
from savant_extras import pitcher_quality_range

pq = pitcher_quality_range(2020, 2024)

# Statcast データと mlbam_id で結合
# merged = statcast_df.merge(pq[["mlbam_id","season","stuff_plus","location_plus"]],
#                            on=["mlbam_id","season"], how="left")
```

---

## まとめ

```python
from savant_extras import (
    park_factors, park_factors_range,
    outs_above_average, outs_above_average_range,
    outfield_jump, outfield_jump_range,
    pitcher_quality, pitcher_quality_range,
)
```

| 指標 | 取得元 | 特徴 |
|---|---|---|
| park_factors | FanGraphs guts | 30球団・複数年・11指標 |
| outs_above_average | Baseball Savant | 方向別OAA・得点換算 |
| outfield_jump | Baseball Savant | 反応/加速/ルーティング |
| pitcher_quality | FanGraphs leaders | Stuff+/Location+・球種別 |

いずれも pybaseball では取得できないデータです。ML 予測モデルの特徴量として活用できます。

- **GitHub**: https://github.com/yasumorishima/savant-extras
- **PyPI**: https://pypi.org/project/savant-extras/
