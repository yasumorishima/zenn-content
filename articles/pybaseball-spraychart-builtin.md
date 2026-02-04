---
title: "pybaseballのspraychart()で打球分布を1行で可視化する【Google Colab対応】"
emoji: "⚾"
type: "tech"
topics: ["python", "pybaseball", "baseball", "データ分析", "googlecolab"]
published: true
---

## はじめに

pybaseballでStatcastデータを可視化するとき、ファールラインを`ax.plot()`で手動で引いていませんか？

私も以前はこんなコードを書いていました：

```python
# 以前の自分のコード（手動でファールライン描画）
ax.plot([0, -330*0.707], [0, 330*0.707], 'k-', lw=2)  # レフト側
ax.plot([0, 330*0.707], [0, 330*0.707], 'k-', lw=2)    # ライト側
# ... 外野フェンス、ダイヤモンド、ベース...延々と続く
```

座標変換も自己流で、毎回微調整が必要でした。

実は、pybaseballには`spraychart()`という組み込み関数があり、**1行でスプレーチャートが完成**します。この記事ではその使い方を紹介します。

:::message
この記事は「Statcastデータで打球分布を可視化する」シリーズの1本目です。
- **方法1: pybaseball spraychart()** ← この記事
- 方法2: matplotlib手動描画（ヒートマップ対応）
:::

## 環境構築（Google Colab）

Google Colabで以下を実行するだけでOKです。

```python
!pip install pybaseball duckdb -q
```

## データ取得

```python
from pybaseball import statcast, spraychart
import duckdb

# ====== 設定 ======
BATTER_ID = 660271      # 大谷翔平 MLBAM ID
SEASON_YEAR = 2025
GAME_TYPE = "R"         # "R"=レギュラーシーズン, "P"=ポストシーズン, None=全試合
# ==================

# Statcastデータ取得（シーズン全体）
df_raw = statcast(start_dt=f'{SEASON_YEAR}-03-01', end_dt=f'{SEASON_YEAR}-12-31')
print(f"Total records (raw): {len(df_raw):,}")

# DuckDBでフィルター
con = duckdb.connect()

if GAME_TYPE:
    df = con.execute(f"""
        SELECT * FROM df_raw WHERE game_type = '{GAME_TYPE}'
    """).df()
    print(f"Filtered (game_type='{GAME_TYPE}'): {len(df):,}")
else:
    df = df_raw.copy()
```

:::message alert
`GAME_TYPE`を指定しないとオープン戦のデータも混入します。レギュラーシーズンのみ分析したい場合は必ず`"R"`を設定してください。
:::

## DuckDBで打球データを抽出

```python
# ヒット（安打）のみ
df_hits = con.execute("""
    SELECT * FROM df
    WHERE batter = 660271
      AND events IN ('home_run', 'double', 'triple', 'single')
      AND hc_x IS NOT NULL AND hc_y IS NOT NULL
""").df()

# 全打球（アウト含む）
df_all = con.execute("""
    SELECT * FROM df
    WHERE batter = 660271
      AND hc_x IS NOT NULL AND hc_y IS NOT NULL
""").df()

# ホームランのみ
df_hr = con.execute("""
    SELECT * FROM df
    WHERE batter = 660271
      AND events = 'home_run'
      AND hc_x IS NOT NULL AND hc_y IS NOT NULL
""").df()

print(f"Hits: {len(df_hits)}, All: {len(df_all)}, HR: {len(df_hr)}")
```

## spraychart() の基本

```python
spraychart(data, team_stadium, title='', colorby='events')
```

| パラメータ | 説明 | 例 |
|---|---|---|
| `data` | `hc_x`, `hc_y`, `events`列を含むDataFrame | `df_hits` |
| `team_stadium` | 球場名（30球団 or `'generic'`） | `'dodgers'`, `'generic'` |
| `colorby` | 色分けの基準 | `'events'` |

**必要な列は`hc_x`、`hc_y`、`events`の3つだけ。** 座標変換も球場描画も全部自動です。

### 基本例

```python
# 汎用スタジアム + イベント別色分け
spraychart(df_hits, 'generic', title='Ohtani 2025 Hits', colorby='events')
```

```python
# 全打球
spraychart(df_all, 'generic', title='Ohtani 2025 All Batted Balls', colorby='events')
```

## 球場別スプレーチャート（重要）

### よくある間違い

```python
# ❌ 間違い - 球場の形だけ変わるが、データは全球場のまま
spraychart(df_all, 'dodgers', title='@ Dodger Stadium')
```

球場名を変えても**表示するデータまでフィルターされるわけではない**ので、全球場のデータがドジャースタジアムの形で表示されてしまいます。

### 正しい方法：`home_team`でフィルター

```python
# ✅ 正しい - その球場でのデータのみ使う
df_dodgers = df_all[df_all['home_team'] == 'LAD']
spraychart(df_dodgers, 'dodgers', title='Ohtani @ Dodger Stadium', colorby='events')
```

### 球場別に一括表示する関数

```python
STADIUM_TEAMS = {
    'dodgers': 'LAD', 'angels': 'LAA', 'yankees': 'NYY',
    'red_sox': 'BOS', 'astros': 'HOU', 'padres': 'SD',
    'giants': 'SF', 'mariners': 'SEA',
}

def spraychart_by_stadium(df_data, stadium_name):
    team_code = STADIUM_TEAMS.get(stadium_name)
    df_stadium = df_data[df_data['home_team'] == team_code]

    if len(df_stadium) == 0:
        print(f"No data at {stadium_name}")
        return

    print(f"{stadium_name.upper()}: {len(df_stadium)} batted balls")
    spraychart(df_stadium, stadium_name,
               title=f'Ohtani 2025 @ {stadium_name.title()}',
               colorby='events')

# ドジャースタジアム（ホーム）
spraychart_by_stadium(df_all, 'dodgers')

# パドレス（アウェイ）
spraychart_by_stadium(df_all, 'padres')
```

## ホームラン専用表示

```python
df_hr_dodgers = df_hr[df_hr['home_team'] == 'LAD']

if len(df_hr_dodgers) > 0:
    spraychart(df_hr_dodgers, 'dodgers',
               title=f'Ohtani 2025 HRs @ Dodger Stadium ({len(df_hr_dodgers)})')
```

## spraychart()のメリット・デメリット

| メリット | デメリット |
|---|---|
| コード最小限（1行で描画） | ヒートマップとの組み合わせが難しい |
| 30球団の球場形状を内蔵 | カスタマイズに制限がある |
| 座標変換が不要 | 散布図のみ（密度表示不可） |

**ヒートマップを重ねたい場合**は、方法2（matplotlib手動描画）を使ってください。

## まとめ

手動でファールラインを描画するコードは、`spraychart()`1行で置き換えられます。

```python
# Before: 手動で何十行もコード書いてた
ax.plot([0, -330*0.707], [0, 330*0.707], 'k-', lw=2)
ax.plot([0, 330*0.707], [0, 330*0.707], 'k-', lw=2)
# ... 省略 ...

# After: 1行
spraychart(df_hits, 'generic', title='Ohtani 2025 Hits', colorby='events')
```

**注意点:**
- `GAME_TYPE`でオープン戦を除外する
- 球場別に表示するときは必ず`home_team`でデータをフィルターする

## Google Colabで試す

この記事のコードはGoogle Colabで実行できます。

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/yasumorishima/mlb-statcast-visualization/blob/main/ohtani_1_spraychart_pybaseball.ipynb)

## 参考

- [pybaseball公式リポジトリ](https://github.com/jldbc/pybaseball)
- [Baseball Savant](https://baseballsavant.mlb.com/)
