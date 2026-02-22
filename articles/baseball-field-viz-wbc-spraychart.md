---
title: "WBCデータで野球場スプレーチャートを描く：baseball-field-viz を作った"
emoji: "⚾"
type: "tech"
topics: [python, baseball, matplotlib, pypi, statcast]
published: true
---

## はじめに

pybaseballの組み込み`spraychart()`は便利ですが、**ヒートマップを重ねられない**という制約があります。

seabornの`kdeplot`や`histplot`でゾーン別の打球密度を表示したい場合、matplotlibで野球場を手動描画する必要があります。

座標変換と野球場描画を毎回書くのが面倒なので、ライブラリにまとめました。

```bash
pip install baseball-field-viz
```

> **PyPI**: https://pypi.org/project/baseball-field-viz/
> **GitHub**: https://github.com/yasumorishima/baseball-field-viz

## baseball-field-viz とは

Statcastデータから野球場スプレーチャートを描くPythonライブラリです。

3つの関数を提供します。

```python
from baseball_field_viz import transform_coords, draw_field, spraychart
```

| 関数 | 説明 |
|---|---|
| `transform_coords(df)` | Statcast `hc_x`/`hc_y` → フィート座標変換（ホームプレート原点） |
| `draw_field(ax)` | matplotlib Axesに野球場を描画 |
| `spraychart(ax, df, color_by='events')` | 上2つを組み合わせた一発生成 |

## 使い方

### 一番シンプルな例

```python
import matplotlib.pyplot as plt
from baseball_field_viz import spraychart

fig, ax = plt.subplots(figsize=(10, 10))
spraychart(ax, df, color_by='events', title='選手名 - 打球分布')
plt.show()
```

`color_by='events'` でホームラン=赤、トリプル=橙、ダブル=青、シングル=緑に自動色分けされます。

### ヒートマップを重ねる（この方法の最大のメリット）

`draw_field(ax)` で野球場を描いたあと、seabornの`kdeplot`をそのまま重ねられます。

```python
import seaborn as sns
from baseball_field_viz import draw_field, transform_coords

df_t = transform_coords(df[df['hc_x'].notna()])

fig, axs = plt.subplots(1, 2, figsize=(16, 8))

draw_field(axs[0])
sns.kdeplot(data=df_t[hits_mask], x='x', y='y',
            ax=axs[0], cmap='Reds', fill=True, alpha=0.6)
axs[0].set_xlim(-350, 350); axs[0].set_ylim(-50, 400)

draw_field(axs[1])
sns.kdeplot(data=df_t[outs_mask], x='x', y='y',
            ax=axs[1], cmap='Blues', fill=True, alpha=0.6)
axs[1].set_xlim(-350, 350); axs[1].set_ylim(-50, 400)

plt.tight_layout()
plt.show()
```

## WBC 2026 データで試す

WBC 2026スカウティング用に作成したKaggleデータセット（全18カ国・MLB Statcastデータ）でこのライブラリを使ったノートブックを公開しています。

> **Kaggle Notebook**: https://www.kaggle.com/code/yasunorim/wbc-2026-spray-charts-with-baseball-field-viz

### 全18カ国のスプレーチャート

`draw_field(ax)` で野球場を描いたあと、国別にカラーリングして散布図を重ねると、全カ国の打球分布を一枚で確認できます。

### 上位4カ国の比較（spraychart）

`spraychart()` を使えば数行で各国のスプレーチャートを生成できます。

```python
top_countries = ['USA', 'Dominican Republic', 'Venezuela', 'Japan']
fig, axs = plt.subplots(2, 2, figsize=(16, 14))

for ax, country in zip(axs.flat, top_countries):
    df_c = df[df['country_name'] == country]
    spraychart(ax, df_c, color_by='events', title=country)

plt.tight_layout()
plt.show()
```

### 日本のヒットvsアウト ヒートマップ

打球がどのゾーンに集まりやすいかが、kdeplotで視覚的にわかります。ヒットとアウトを左右で比較することで、どのゾーンで生産的な打球が生まれているかを把握できます。

## インストール

```bash
pip install baseball-field-viz
```

依存パッケージ：`matplotlib>=3.5` / `numpy>=1.21` / `pandas>=1.3`

## savant-extrasとの組み合わせ

同じく自作の[savant-extras](https://github.com/yasumorishima/savant-extras)（バットトラッキングデータ取得ライブラリ）と組み合わせると、データ取得から可視化まで一貫したMLB分析パイプラインが構築できます。

## まとめ

```python
pip install baseball-field-viz
```

- pybaseballの`spraychart()`ではできないヒートマップ重ね表示が可能
- matplotlibの`Axes`を直接操作できるので、`kdeplot`・`histplot`など任意のプロットを重ねられる
- WBC 2026データで実際に動作確認済み（Kaggle Notebookで公開中）

> **PyPI**: https://pypi.org/project/baseball-field-viz/
> **GitHub**: https://github.com/yasumorishima/baseball-field-viz
