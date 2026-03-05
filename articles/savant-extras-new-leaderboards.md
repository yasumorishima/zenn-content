---
title: "球場係数をPythonで取得する（savant-extras v0.3.2〜v0.4.2）"
emoji: "⚾"
type: "tech"
topics: ["python", "pypi", "baseball", "mlb", "savant"]
published: true
---

## はじめに

[pybaseball](https://github.com/jldbc/pybaseball) は Baseball Savant のデータを Python で扱う定番ライブラリですが、対応していないリーダーボードが多数あります。これらを補完するために作成した **[savant-extras](https://github.com/yasumorishima/savant-extras)** の最近の変更をまとめます。

| バージョン | 内容 |
|---|---|
| v0.3.2 | **park_factors** 追加（FanGraphs 球場補正係数） |
| v0.3.3 | lxml 依存追加・StringIO fix（pandas 2.0+ 対応） |
| v0.4.0 | outs_above_average / outfield_jump 追加 |
| v0.4.1 | pitcher_quality 追加 |
| **v0.4.2** | **outs_above_average / outfield_jump / pitcher_quality 削除**（pybaseball に同等機能あり） |

```bash
pip install savant-extras
```

---

## v0.4.2 の変更：3関数を削除

調査の結果、v0.4.0〜v0.4.1 で追加した3つの関数が **pybaseball に既に実装されていた**ため削除しました。

| 削除した関数 | pybaseball の代替 |
|---|---|
| `outs_above_average(year)` | `statcast_outs_above_average(year, 'all')` |
| `outfield_jump(year)` | `statcast_outfielder_jump(year)` |
| `pitcher_quality(year)` | `fg_pitching_data(year, qual=0)[["Stuff+", "Location+", "Pitching+"]]` |

これらは同じデータソース（Baseball Savant / FanGraphs）を参照しており、返すカラムも同一でした。重複した実装を維持するメリットがないため削除しています。

```python
# pybaseball で代替可能
from pybaseball import statcast_outs_above_average, statcast_outfielder_jump, fg_pitching_data

df_oaa = statcast_outs_above_average(2024, 'all')
df_oj  = statcast_outfielder_jump(2024)
df_pq  = fg_pitching_data(2024, qual=0)[["Name", "Team", "Stuff+", "Location+", "Pitching+"]]
```

---

## park_factors — 球場補正係数（v0.3.2、継続）

### 何が取れるか

FanGraphs の Park Factors テーブルから、MLB 全30球団の球場補正係数を取得します。100が中立、100超がヒッター有利、100未満がピッチャー有利です。**pybaseball では取得できません。**

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

```python
pf_lookup = {
    (int(r.season), str(r.team)): float(r.pf_5yr)
    for _, r in park_factors_range(2020, 2025).iterrows()
}
# 使い方: pf_lookup.get((2024, "COL"), 100)
```

---

## まとめ

savant-extras は「pybaseball で取れないデータ」に特化するという方針のもと、重複機能を整理しました。

現在 savant-extras でのみ取得できる主なリーダーボード：

| 関数 | 説明 |
|---|---|
| `park_factors` / `park_factors_range` | FanGraphs 球場補正係数 |
| `bat_tracking` | バットトラッキング（日付範囲指定可） |
| `pitch_tempo` | 投球テンポ |
| `catcher_blocking` / `catcher_stance` | 捕手守備指標 |
| `timer_infractions` | ピッチクロック違反 |

pybaseball と合わせて使うことで、Baseball Savant の主要リーダーボードをほぼカバーできます。

- **GitHub**: https://github.com/yasumorishima/savant-extras
- **PyPI**: https://pypi.org/project/savant-extras/
