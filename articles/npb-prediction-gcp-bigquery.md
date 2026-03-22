---
title: "NPB予測システムをBigQueryに載せた — 無料枠でBQML・Cloud Runまで"
emoji: "☁️"
type: "tech"
topics: [bigquery, gcp, python, mlops, baseball]
published: false
---

## はじめに

NPB選手成績予測プロジェクトを1年以上運用してきました。

→ 過去の記事:
- [Marcel法とMLを比較してみた](https://zenn.dev/shogaku/articles/npb-prediction-marcel-vs-ml)
- [GitHub Actionsで年次自動再学習](https://zenn.dev/shogaku/articles/npb-mlops-cicd)

これまでの構成は「GitHub Actions でデータ取得→学習→CSV保存→Streamlit で表示」で完結していました。データはCSV、APIはRaspberry Pi 5のDockerコンテナ、分析はローカルのPython。

ここにGoogle BigQueryを導入して、データの集約・SQL分析・BQMLでの精度比較・Cloud RunでのAPI公開まで一通りやりました。GCP無料枠内で収まっています。

→ **GitHub**: https://github.com/yasumorishima/npb-prediction

---

## はじめての方へ：この記事で登場する用語

| 用語 | 意味 |
|---|---|
| **BigQuery** | Googleのクラウドデータウェアハウス。SQLでデータ分析ができる |
| **BQML（BigQuery ML）** | BigQuery上でSQLだけで機械学習モデルを作れる機能 |
| **Cloud Run** | Googleのサーバーレスコンテナ実行サービス。リクエストが来たときだけ起動する |
| **Marcel法** | 過去3年の成績を加重平均して翌年を予測する統計手法（5:4:3の重み） |
| **MAE** | 平均絶対誤差。予測と実績のずれの平均。小さいほど精度が高い |
| **パークファクター** | 球場ごとの得点しやすさの補正値。100が中立、105以上は打者有利 |

---

## なぜBigQueryを入れたか

CSVベースの運用で困っていたこと：

1. **毎回ゼロからフェッチ** — GitHub Actionsの年次パイプラインは毎回全データを再取得する。差分追加ができない
2. **クロス分析が面倒** — 打者成績とパークファクターをJOINしたいとき、毎回pandasでマージするコードを書いていた
3. **SQLで分析したい** — 「wRC+ TOP10」「年齢カーブ」のような定型分析をSQLで即座に叩きたい
4. **BQMLを試したい** — Pythonで書いたモデルと同じことがSQLだけでどこまでできるか

---

## 構成

```
GitHub Actions (年次パイプライン)
  ├── データ取得 (baseball-data.com / npb.jp)
  ├── Marcel法予測
  ├── ML予測 (XGBoost / LightGBM)
  ├── load_to_bq.py → BigQuery 25テーブル
  ├── bqml_train.py → BQML 4モデル学習・評価
  └── Cloud Run デプロイ (masterマージ時)

BigQuery (data-platform-490901.npb)
  ├── 生データ 15テーブル
  ├── 予測結果 4テーブル
  ├── サーバー指標 6テーブル
  ├── BQML 4モデル
  └── 分析ビュー 10個

表示層
  ├── Streamlit Cloud (ダッシュボード)
  ├── Cloud Run API (サーバーレス)
  └── Raspberry Pi 5 API (常時稼働)
```

---

## BigQueryへのデータロード

`load_to_bq.py` がCSVファイルをBigQueryに流し込みます。

```python
RAW_TABLE_MAP = {
    "npb_hitters_2015_2025.csv": "raw_hitters",
    "npb_pitchers_2015_2025.csv": "raw_pitchers",
    "npb_batting_detailed_2015_2025.csv": "raw_batting_detailed",
    "npb_sabermetrics_2015_2025.csv": "sabermetrics",
    # ... 25テーブル
}
```

FanGraphs/NPBデータのカラム名には `K%`、`BB%`、`HR/9` のようなBigQuery非互換の文字が含まれるため、ロード時にサニタイズしています。

```python
# 正規表現で英数字・アンダースコア以外を除去
new = new.replace("%", "_pct")
new = new.replace("/", "_per_")
new = re.sub(r"[^a-zA-Z0-9_]", "_", new)
```

全テーブル `WRITE_TRUNCATE`（フルリプレース）で毎回上書きするので、スキーマ変更にも追従できます。

---

## BQMLでSQLだけでモデル学習

BigQuery MLでは、SQLのウインドウ関数で特徴量を構築し、`CREATE MODEL` でモデルを学習します。

### 学習用ビュー（特徴量エンジニアリング）

```sql
CREATE OR REPLACE VIEW `npb.v_batter_train` AS
WITH base AS (
  SELECT player, season, OPS, wOBA, K_pct, BB_pct, Age, PA, ...
  FROM `npb.raw_hitters`
  WHERE PA >= 100
),
lagged AS (
  SELECT
    player, season,
    -- 直前シーズンの成績
    LAG(OPS, 1) OVER w AS OPS_y1,
    LAG(wOBA, 1) OVER w AS wOBA_y1,
    -- 2年前
    LAG(OPS, 2) OVER w AS OPS_y2,
    -- トレンド（前年との差分）
    LAG(OPS, 1) OVER w - LAG(OPS, 2) OVER w AS OPS_delta,
    -- 年齢カーブ
    LAG(Age, 1) OVER w - 27 AS age_from_peak,
    POW(LAG(Age, 1) OVER w - 27, 2) AS age_sq,
    -- ターゲット
    OPS AS target_ops
  FROM base
  WINDOW w AS (PARTITION BY player ORDER BY season)
)
SELECT * FROM lagged WHERE OPS_y1 IS NOT NULL;
```

Pythonで書いていたラグ特徴量・差分・年齢曲線をSQLのウインドウ関数で再現しています。

### モデル学習

```sql
CREATE OR REPLACE MODEL `npb.bqml_batter_ops`
OPTIONS(
  model_type = 'BOOSTED_TREE_REGRESSOR',
  input_label_cols = ['target_ops'],
  max_iterations = 200,
  learn_rate = 0.05,
  early_stop = TRUE
) AS
SELECT OPS_y1, wOBA_y1, K_pct_y1, BB_pct_y1,
       age_from_peak, age_sq, OPS_delta, ...
FROM `npb.v_batter_train`;
```

4モデルを作成しました：

| モデル | ターゲット | タイプ |
|---|---|---|
| `bqml_batter_ops` | 翌年 OPS | Boosted Tree |
| `bqml_batter_ops_linear` | 翌年 OPS | 線形回帰 |
| `bqml_pitcher_era` | 翌年 ERA | Boosted Tree |
| `bqml_pitcher_era_linear` | 翌年 ERA | 線形回帰 |

---

## BQML vs Python ML 精度比較

同じデータ・同じ評価期間でMAEを比較しました。

**打者 OPS MAE（低いほど良い）**

| モデル | MAE |
|---|---|
| BQML Boosted Tree | **.0642** |
| Python (XGBoost) | .063 |
| Python (LightGBM) | .066 |
| Marcel法 | .063 |

**投手 ERA MAE（低いほど良い）**

| モデル | MAE |
|---|---|
| BQML Boosted Tree | **.909** |
| Python (XGBoost) | .93 |
| Python (LightGBM) | .92 |
| Marcel法 | **.78** |

BQMLはPython MLと同程度の精度でした。投手ERAではどちらもMarcel法（0.78）に及ばず、これは[前回の記事](https://zenn.dev/shogaku/articles/npb-prediction-marcel-vs-ml)でも触れた課題です。

BQMLの方が特徴量を多く使っている（パークファクター・DIPS指標・Marcel加重平均等を追加）ため、その分が精度に寄与している可能性はあります。

---

## 分析ビュー

自分の分析用にビューを10個用意しました。

| ビュー | 用途 |
|---|---|
| `v_batter_trend` | 選手年度別OPS/wOBAトレンド（前年比付き） |
| `v_pitcher_trend` | 選手年度別ERA/WHIPトレンド + FIP近似 |
| `v_team_pythagorean` | チーム勝率 vs ピタゴラス期待勝率 |
| `v_sabermetrics_leaders` | wRC+リーダーボード（年度別順位付き） |
| `v_marcel_accuracy` | Marcel法の過去精度検証 |
| `v_age_curve` | NPB全体の年齢カーブ（OPS × 年齢） |
| `v_park_effects` | パークファクター影響分析 |
| `v_data_coverage` | シーズン別データカバレッジ |
| `v_data_quality` | テーブル別NULL/欠損値サマリー |

例えば「2025年のwRC+ TOP10」や「年齢カーブのピーク」を確認するときに、pandasを書かずにSQLで済むようになりました。

```sql
-- 自分の環境で使っているクエリ例
SELECT player, team, season, wRC_plus, wOBA, OPS
FROM `npb.v_sabermetrics_leaders`
WHERE season = 2025
ORDER BY wrc_rank
LIMIT 10;
```

---

## Cloud Run デプロイ

既存のFastAPIをCloud Runにデプロイしました。

```dockerfile
FROM python:3.12-slim
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "api:app", "--host", "0.0.0.0", "--port", "${PORT:-8080}"]
```

GitHub ActionsでmasterブランチにマージするとArtifact Registry経由で自動デプロイされます。

Raspberry Pi 5上のDockerコンテナと同じAPIがCloud Runでも動く状態です。

---

## 無料枠の使用状況

全てGCP無料枠内で運用しています。

| リソース | 無料枠 | 使用量 | 使用率 |
|---|---|---|---|
| ストレージ | 10 GB/月 | 約5 MB | 0.05% |
| クエリ | 1 TB/月 | 約22 GB | 2.2% |
| Cloud Run | 200万リクエスト/月 | ごく少量 | ≈0% |

日次でBigQueryの使用量をモニタリングし、月末予測ペースとともにDiscordに通知する仕組みも入れています。

---

## GitHub Actionsパイプライン

年次パイプライン（`annual_update.yml`）にBigQueryロード・BQML学習・Cloud Runデプロイを統合しました。

```
Step 1: fetch_npb_data.py       → 打者/投手成績スクレイプ
Step 2: fetch_npb_detailed.py   → 詳細打撃成績（wOBA算出用）
Step 3: pythagorean.py          → 順位表・ピタゴラス勝率
Step 4: sabermetrics.py         → wOBA/wRC+/wRAA算出
Step 5: marcel_projection.py    → Marcel法予測
Step 6: ml_projection.py        → ML予測 + モデル保存
Step 7: git commit & push       → data/ を自動コミット
Step 8: load_to_bq.py           → BigQuery に全データロード  ← NEW
Step 9: bqml_train.py           → BQML モデル学習・評価     ← NEW
```

BQMLステップは `continue-on-error: true` にしているので、BigQuery側に障害があってもPython MLパイプライン自体は止まりません。

---

## 所感

- BQMLの精度はPythonと同程度だった。SQLのウインドウ関数で特徴量を書くのは慣れが要るが、ビューとして再利用できるのは楽
- 分析ビューは地味に便利。pandasを書かずにSQLで定型分析が済む
- 40,000行程度のデータなら無料枠を気にする必要はほぼない
- Cloud RunとRPi5の2系統でAPIが動くので、片方が落ちても止まらない

---

## 関連記事

- [Marcel法とMLを比較してみた — NPB選手成績予測システムを作った](https://zenn.dev/shogaku/articles/npb-prediction-marcel-vs-ml)
- [GitHub Actionsで年次自動再学習 — NPB予測システムをMLOps的に育てた](https://zenn.dev/shogaku/articles/npb-mlops-cicd)
