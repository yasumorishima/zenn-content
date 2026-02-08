---
title: "日本人MLB選手のStatcastデータセットをKaggleに公開した話"
emoji: "⚾"
type: "tech"
topics: ["kaggle", "python", "baseball", "データ分析", "pybaseball"]
published: true
---

## はじめに

MLB分析ノートブックをKaggleに公開していたのですが、「データセット自体も公開すれば再利用できるのでは」と考え、日本人MLB選手のStatcastデータをまとめてみました。

この記事では、Kaggleデータセットの作成方法と、pybaseballを使ったデータ収集の流れを記録しておきます。

---

## 📊 作成したデータセット

https://www.kaggle.com/datasets/yasunorim/japan-mlb-pitchers-batters-statcast

**Japan MLB Pitchers Batters Statcast (2015-2025)**

- **投球データ**: 25投手、118,226投球（65MB）
- **打撃データ**: 10打者、56,362打撃（31MB）
- **選手メタデータ**: 34選手
- **期間**: 2015年〜2025年（Statcast導入以降）

---

## 🛠️ データ収集の流れ

### 1. pybaseballでの選手ID取得

まず、選手のMLBAM IDを取得します。

```python
from pybaseball import playerid_lookup

# 選手名から検索
player = playerid_lookup("ohtani", "shohei")
print(player)
# key_mlbam列がMLBAM ID（660271）
```

### 2. Statcastデータの取得

MLBAM IDを使ってpitch-by-pitchデータを取得します。

```python
from pybaseball import statcast_pitcher, statcast_batter

# 投手データ
df_pitching = statcast_pitcher(
    start_dt="2015-03-01",
    end_dt="2025-02-08",
    player_id=660271  # 大谷翔平
)

# 打者データ
df_batting = statcast_batter(
    start_dt="2015-03-01",
    end_dt="2025-02-08",
    player_id=660271  # 大谷翔平
)
```

### 3. 複数選手のデータ結合

全選手分を取得してconcatします。

```python
import pandas as pd

pitcher_ids = [660271, 506433, 808967, ...]  # 全投手のMLBAM ID
batting_ids = [660271, 673548, 807799, ...]  # 全打者のMLBAM ID

# 投手データ
pitching_dfs = []
for pid in pitcher_ids:
    df = statcast_pitcher("2015-03-01", "2025-02-08", pid)
    pitching_dfs.append(df)
df_all_pitching = pd.concat(pitching_dfs, ignore_index=True)

# 打者データ
batting_dfs = []
for bid in batting_ids:
    df = statcast_batter("2015-03-01", "2025-02-08", bid)
    batting_dfs.append(df)
df_all_batting = pd.concat(batting_dfs, ignore_index=True)
```

### 4. CSV保存

```python
df_all_pitching.to_csv("japanese_mlb_pitching.csv", index=False)
df_all_batting.to_csv("japanese_mlb_batting.csv", index=False)
```

---

## 📦 Kaggleデータセットの作成

### 1. dataset-metadata.json作成

データセットフォルダに `dataset-metadata.json` を配置します。

```json
{
  "title": "Japan MLB Pitchers Batters Statcast",
  "id": "yasunorim/japan-mlb-pitchers-batters-statcast",
  "licenses": [{"name": "CC0-1.0"}],
  "keywords": ["baseball", "sports", "japan"]
}
```

### 2. Kaggle CLIでアップロード

```bash
kaggle datasets create -p japanese-mlb-players-statcast/
```

初回アップロード後、更新する場合は：

```bash
kaggle datasets version -p japanese-mlb-players-statcast/ -m "Add new player data"
```

### 3. Description・カバー画像の設定

Kaggle UIでSubtitle（80文字以内）、Description（Markdown）、カバー画像を設定します。

---

## 📁 データの中身

### `japanese_mlb_pitching.csv`（投球データ）

投球ごとの詳細データ。主なカラム：

| カラム | 説明 |
|---|---|
| `pitcher` | 投手のMLBAM ID |
| `game_date` | 試合日 |
| `pitch_type` | 球種（FF, SL, CU, CH, SI等） |
| `release_speed` | リリース時球速（mph） |
| `release_pos_x`, `release_pos_y`, `release_pos_z` | リリースポイント座標 |
| `pfx_x`, `pfx_z` | 変化量（フィート） |
| `release_spin_rate` | 回転数（rpm） |
| `events` | 打撃結果（single, strikeout等） |
| `game_type` | R=レギュラーシーズン、S=春季、F/D/L/W=ポストシーズン |

### `japanese_mlb_batting.csv`（打撃データ）

打撃ごとの詳細データ。主なカラム：

| カラム | 説明 |
|---|---|
| `batter` | 打者のMLBAM ID |
| `launch_speed` | 打球速度（mph） |
| `launch_angle` | 打球角度（度） |
| `hit_distance_sc` | 飛距離（フィート） |
| `estimated_ba_using_speedangle` | xBA（期待打率） |
| `estimated_woba_using_speedangle` | xwOBA（期待wOBA） |
| `events` | 打撃結果 |
| `game_type` | 試合タイプ |

### `players.csv`（選手メタデータ）

| カラム | 説明 |
|---|---|
| `mlbam_id` | MLBAM ID |
| `player_name` | 選手名 |
| `position` | pitcher / batter |

---

## 💡 使用例

### DuckDBでSQLクエリ

```python
import pandas as pd
import duckdb

df = pd.read_csv("japanese_mlb_pitching.csv")
con = duckdb.connect()

# ダルビッシュ有（MLBAM ID: 506433）のレギュラーシーズンデータ
df_darvish = con.execute("""
    SELECT game_year, pitch_type, COUNT(*) as count,
           AVG(release_speed) as avg_speed,
           AVG(release_spin_rate) as avg_spin
    FROM df
    WHERE pitcher = 506433 AND game_type = 'R'
    GROUP BY game_year, pitch_type
    ORDER BY game_year, count DESC
""").df()
```

### 打球分布の可視化

```python
import matplotlib.pyplot as plt

df_batting = pd.read_csv("japanese_mlb_batting.csv")
df_ohtani = df_batting[df_batting["batter"] == 660271]

plt.scatter(df_ohtani["launch_angle"], df_ohtani["launch_speed"], alpha=0.5)
plt.xlabel("Launch Angle (degrees)")
plt.ylabel("Launch Speed (mph)")
plt.title("Batted Ball Profile")
plt.show()
```

---

## 🔗 このデータセットを使った分析ノートブック

以下のノートブックで本データセットを活用しています。

1. [ダルビッシュ有の投球変化（2021-2025）](https://www.kaggle.com/code/yasunorim/darvish-pitching-evolution)
2. [今永昇太の2年目の変化（2024-2025）](https://www.kaggle.com/code/yasunorim/imanaga-rookie-to-sophomore-pitching)
3. [千賀滉大の「お化けフォーク」分析（2023-2025）](https://www.kaggle.com/code/yasunorim/senga-ghost-fork-analysis-2023-2025)
4. [菊池雄星の「スライダー革命」（2019-2025）](https://www.kaggle.com/code/yasunorim/kikuchi-slider-revolution-2019-2025)

---

## 📝 学んだこと

### Kaggleデータセット更新の注意点

- `kaggle datasets version` は前バージョンのファイルを引き継ぐ（個別ファイル削除不可）
- 不要ファイルを入れた場合は削除→再作成が必要
- 削除したデータセットのタイトル/slugは再利用不可（Kaggle側で予約される）

### CSVのバックアップ

- Kaggleだけでなく、GitHubにもCSVをバックアップしておく（`.gitignore`から除外）
- データセット削除時のデータロスト防止

### `dataset-metadata.json` vs Kaggle APIのメタデータ

- ローカルの `dataset-metadata.json` は `{"title":...}` 形式
- `kaggle datasets metadata` でダウンロードすると `{"info": {...}}` 形式
- 更新時は後者の形式で `--update` オプションを使う

---

## 🔗 リンク

- **データセット**: https://www.kaggle.com/datasets/yasunorim/japan-mlb-pitchers-batters-statcast
- **GitHubリポジトリ**: https://github.com/yasumorishima/kaggle-datasets
- **分析ノートブック（GitHub）**: https://github.com/yasumorishima/mlb-statcast-visualization
