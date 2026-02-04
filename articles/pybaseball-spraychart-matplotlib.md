---
title: "matplotlibで野球場を描画してヒートマップを重ねる【Statcast可視化】"
emoji: "🔥"
type: "tech"
topics: ["python", "matplotlib", "baseball", "データ分析", "googlecolab"]
published: true
---

## はじめに

pybaseballの`spraychart()`は便利ですが、**ヒートマップ（密度表示）を重ねられない**という制約があります。

「大谷の打球はどのゾーンに集中しているのか？」を可視化したいなら、matplotlibで野球場を手動描画してseabornのkdeplotを重ねる方法が適しています。

この記事では、Statcast座標の変換方法と、matplotlibでの野球場描画を解説します。

:::message
この記事は「Statcastデータで打球分布を可視化する」シリーズの2本目です。
- 方法1: pybaseball spraychart()（最もシンプル）
- **方法2: matplotlib手動描画** ← この記事
:::

## 環境構築（Google Colab）

```python
!pip install pybaseball duckdb seaborn -q
```

## Statcast座標系を理解する

matplotlibで描画するには、まずStatcastの座標系を理解する必要があります。

**Statcast座標系の特徴：**
- ホームプレート位置: `(125.42, 198.27)`
- Y軸は**画面座標系**（上がゼロ、下が大きい）→ 通常の数学座標系とは逆

そのままでは扱いにくいので、「ホームプレートを原点に、Y軸を外野方向に」変換します：

```python
# 変換式
x = 2.5 * (hc_x - 125.42)  # ホームプレートを原点に
y = 2.5 * (198.27 - hc_y)  # Y軸を反転（外野方向が+）
```

- `125.42, 198.27` = Statcast座標でのホームプレート位置
- 係数`2.5` = おおよそフィート単位に変換

## データ取得 & 座標変換

```python
from pybaseball import statcast
import duckdb
import matplotlib.pyplot as plt
import numpy as np
import seaborn as sns

# ====== 設定 ======
BATTER_ID = 660271      # 大谷翔平 MLBAM ID
SEASON_YEAR = 2025
GAME_TYPE = "R"         # "R"=レギュラーシーズン, "P"=ポストシーズン, None=全試合
# ==================

# データ取得
df_raw = statcast(start_dt=f'{SEASON_YEAR}-03-01', end_dt=f'{SEASON_YEAR}-12-31')

# game_typeでフィルタ（オープン戦除外）
con = duckdb.connect()
if GAME_TYPE:
    df = con.execute(f"""
        SELECT * FROM df_raw WHERE game_type = '{GAME_TYPE}'
    """).df()
else:
    df = df_raw.copy()
```

:::message alert
`GAME_TYPE`を指定しないとオープン戦のデータも混入します。
:::

```python
# DuckDBでデータ抽出
df_hits = con.execute("""
    SELECT * FROM df
    WHERE batter = 660271
      AND events IN ('home_run', 'double', 'triple', 'single')
      AND hc_x IS NOT NULL AND hc_y IS NOT NULL
""").df()

df_outs = con.execute("""
    SELECT * FROM df
    WHERE batter = 660271
      AND events NOT IN ('home_run', 'double', 'triple', 'single')
      AND hc_x IS NOT NULL AND hc_y IS NOT NULL
""").df()

print(f"Hits: {len(df_hits)}, Outs: {len(df_outs)}")
```

### 座標変換

```python
def transform_statcast_coords(df):
    """Statcast座標を野球場座標に変換（ホームプレート原点、Y外野方向）"""
    df = df.copy()
    df['x'] = 2.5 * (df['hc_x'] - 125.42)
    df['y'] = 2.5 * (198.27 - df['hc_y'])
    return df

df_hits_t = transform_statcast_coords(df_hits)
df_outs_t = transform_statcast_coords(df_outs)
```

## フィールド描画関数

描画する要素：ファールライン、外野フェンス、内野アーク、ダイヤモンド、ベース、ピッチャーマウンド

```python
def draw_baseball_field(ax, foul_distance=330, outfield_distance=340):
    """野球場を描画（ホームプレートが原点）"""

    # ファールライン（45度の角度で外野へ）
    ax.plot([0, -foul_distance * 0.707], [0, foul_distance * 0.707],
            'k-', lw=2, label='Foul Line')
    ax.plot([0, foul_distance * 0.707], [0, foul_distance * 0.707], 'k-', lw=2)

    # 内野アーク（約95フィート）
    theta = np.linspace(-np.pi/4, np.pi/4, 100)
    infield_dist = 95
    ax.plot(infield_dist * np.sin(theta), infield_dist * np.cos(theta),
            'green', lw=2, alpha=0.7)

    # 外野フェンス
    ax.plot(outfield_distance * np.sin(theta), outfield_distance * np.cos(theta),
            'saddlebrown', lw=3, alpha=0.7)

    # ベースパス（90フィート四方のダイヤモンド）
    bases_x = [0, 63.64, 0, -63.64, 0]
    bases_y = [0, 63.64, 127.28, 63.64, 0]
    ax.plot(bases_x, bases_y, 'k-', lw=1.5)

    # ベースのマーカー
    ax.scatter([0], [0], color='white', edgecolors='black', s=150, marker='p', zorder=5)
    ax.scatter([63.64, 0, -63.64], [63.64, 127.28, 63.64],
               color='white', edgecolors='black', s=100, marker='s', zorder=5)

    # ピッチャーマウンド（60.5フィート）
    ax.scatter([0], [60.5], color='brown', s=80, zorder=5)

    ax.set_aspect('equal')
    ax.set_facecolor('lightgreen')
    return ax
```

ファールラインが45度なのは、野球場のダイヤモンドが正方形で一塁線・三塁線がそれぞれ45度の角度で外野に伸びるためです。`0.707`は`cos(45°)`です。

## 散布図

```python
fig, ax = plt.subplots(figsize=(10, 10))
draw_baseball_field(ax)

ax.scatter(df_outs_t['x'], df_outs_t['y'], c='blue', alpha=0.5, s=30, label='Outs')
ax.scatter(df_hits_t['x'], df_hits_t['y'], c='red', alpha=0.7, s=50, label='Hits')

ax.set_xlim(-350, 350)
ax.set_ylim(-50, 420)
ax.set_xlabel('X (feet)')
ax.set_ylabel('Y (feet)')
ax.set_title('Ohtani 2025 - Batted Ball Distribution')
ax.legend(loc='upper right')
plt.tight_layout()
plt.show()
```

## ヒートマップ（kdeplot）

**この方法の最大のメリット**がこれです。seabornの`kdeplot`を重ねて、打球の密度分布を可視化できます。

```python
fig, axs = plt.subplots(1, 2, figsize=(16, 8))

# アウト ヒートマップ
draw_baseball_field(axs[0])
sns.kdeplot(data=df_outs_t, x='x', y='y', ax=axs[0],
            cmap='Blues', fill=True, alpha=0.6, levels=10)
axs[0].set_xlim(-350, 350)
axs[0].set_ylim(-50, 400)
axs[0].set_title('Ohtani 2025 - Outs Heatmap')

# ヒット ヒートマップ
draw_baseball_field(axs[1])
sns.kdeplot(data=df_hits_t, x='x', y='y', ax=axs[1],
            cmap='Reds', fill=True, alpha=0.6, levels=10)
axs[1].set_xlim(-350, 350)
axs[1].set_ylim(-50, 400)
axs[1].set_title('Ohtani 2025 - Hits Heatmap')

plt.tight_layout()
plt.show()
```

## ヒストグラム（histplot）

格子状のビンで密度を表示する方法もあります。

```python
fig, axs = plt.subplots(1, 2, figsize=(16, 8))

draw_baseball_field(axs[0])
sns.histplot(data=df_outs_t, x='x', y='y', ax=axs[0],
             cmap='Blues', cbar=True, binwidth=20)
axs[0].set_xlim(-350, 350)
axs[0].set_ylim(-50, 400)
axs[0].set_title('Ohtani 2025 - Outs (Histogram)')

draw_baseball_field(axs[1])
sns.histplot(data=df_hits_t, x='x', y='y', ax=axs[1],
             cmap='Reds', cbar=True, binwidth=20)
axs[1].set_xlim(-350, 350)
axs[1].set_ylim(-50, 400)
axs[1].set_title('Ohtani 2025 - Hits (Histogram)')

plt.tight_layout()
plt.show()
```

## イベント別カラーマップ

シングル・ダブル・トリプル・ホームランを色分けして表示：

```python
colors = {
    'single': 'green',
    'double': 'blue',
    'triple': 'orange',
    'home_run': 'red'
}

fig, ax = plt.subplots(figsize=(12, 12))
draw_baseball_field(ax)

for event, color in colors.items():
    subset = df_hits_t[df_hits_t['events'] == event]
    ax.scatter(subset['x'], subset['y'], c=color, alpha=0.7,
               s=80, label=f"{event} ({len(subset)})")

ax.set_xlim(-350, 350)
ax.set_ylim(-50, 420)
ax.set_title('Ohtani 2025 - Hits by Type')
ax.legend(loc='upper right', fontsize=12)
plt.tight_layout()
plt.show()
```

## メリット・デメリット

| メリット | デメリット |
|---|---|
| ヒートマップ（kdeplot/histplot）を自由に重ねられる | コードが多い |
| 色・スタイル・レイアウトを完全にカスタマイズできる | 座標変換の理解が必要 |
| 複数グラフの横並び比較が容易 | 球場の形は自分で描く |

## まとめ

matplotlibで野球場を手動描画する場合のポイント：

1. **座標変換**: `x = 2.5 * (hc_x - 125.42)`, `y = 2.5 * (198.27 - hc_y)`
2. **フィールド描画**: ファールライン(45度直線) + 外野フェンス(arc) + ダイヤモンド
3. **ヒートマップ**: `draw_baseball_field(ax)` → `sns.kdeplot(...)` の順で重ねる

`spraychart()`では難しいヒートマップ表示が必要なときに最適な方法です。

## Google Colabで試す

この記事のコードはGoogle Colabで実行できます。

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/yasumorishima/mlb-statcast-visualization/blob/main/ohtani_2_matplotlib_manual.ipynb)

## 参考

- [matplotlib公式](https://matplotlib.org/)
- [seaborn公式](https://seaborn.pydata.org/)
- [Baseball Savant](https://baseballsavant.mlb.com/)
