---
title: "GitHub Actionsで年次自動再学習 — NPB予測システムをMLOps的に育てた"
emoji: "⚙️"
type: "tech"
topics: [python, mlops, githubactions, fastapi, baseball]
published: true
---

## はじめに

NPB（日本プロ野球）の選手成績予測プロジェクトを作っていました。

→ 前回記事: [Marcel法がLightGBMを上回った話](https://zenn.dev/shogaku/articles/npb-prediction-marcel-vs-ml)

ひととおり動くようになって気づいたのですが、毎年3月（開幕前）に手動で `python ml_projection.py` を実行する運用になっていました。

これだと以下が全部手動です：

- データ取得（スクレイピング）
- モデルの再学習
- 予測CSVの更新
- 「今年は精度が上がったのか下がったのか」の確認

GitHub Actionsで自動化して、モデル保存と精度記録も追加しました。

→ **GitHub**: https://github.com/yasumorishima/npb-prediction

## 追加した3つの仕組み

| 仕組み | やったこと |
|---|---|
| **モデル保存** | `joblib` で `.pkl` に保存 → `data/models/` に年度ごとに置く |
| **精度記録** | Marcel vs ML の MAE を JSON に保存 + FastAPI で `/metrics` エンドポイント |
| **自動実行** | GitHub Actions で毎年3月1日に自動実行（FA・移籍確定後） |

順番に説明します。

## ① モデルをファイルに保存する（joblib）

学習後にモデルを `.pkl` ファイルに保存します。

```python
import joblib
from pathlib import Path

MODELS_DIR = Path("data/models")
MODELS_DIR.mkdir(parents=True, exist_ok=True)

# 打者モデル保存（lgb と xgb、両方保存する）
for model_name, res in h_results.items():
    if "model" in res:
        pkl_path = MODELS_DIR / f"{model_name}_hitters_{TARGET_YEAR}.pkl"
        joblib.dump(res["model"], pkl_path)
        print(f"Saved: {pkl_path}")
```

ファイル名に年度を入れているので、毎年実行しても上書きされません。

```
data/models/
├── lgb_hitters_2026.pkl    # LightGBM 打者モデル
├── xgb_hitters_2026.pkl    # XGBoost 打者モデル
├── lgb_pitchers_2026.pkl
└── xgb_pitchers_2026.pkl
```

## ② 精度をJSONに記録して、APIで確認できるようにする

### JSONに保存

```python
import json
from datetime import datetime

# Marcel法との比較も含めて記録
metrics = {
    "year": TARGET_YEAR,
    "data_end_year": DATA_END_YEAR,
    "generated_at": datetime.utcnow().isoformat(),
    "hitter": {k: round(v["mae"], 4) for k, v in h_results.items() if "mae" in v},
    "pitcher": {k: round(v["mae"], 4) for k, v in p_results.items() if "mae" in v},
}
metrics["hitter"]["marcel"] = round(marcel_mae, 4)
metrics["pitcher"]["marcel"] = round(marcel_mae_p, 4)

path = Path("data/metrics") / f"metrics_{TARGET_YEAR}.json"
with open(path, "w") as f:
    json.dump(metrics, f, indent=2)
```

出力されるJSON（`metrics_2026.json`）の構造例：

```json
{
  "year": 2026,
  "data_end_year": 2025,
  "generated_at": "2026-11-01T09:30:00",
  "hitter": {
    "lgb": 0.031,
    "xgb": 0.033,
    "ensemble": 0.030,
    "marcel": 0.048
  },
  "pitcher": {
    "lgb": 0.58,
    "xgb": 0.61,
    "ensemble": 0.57,
    "marcel": 0.63
  }
}
```

`hitter.lgb < hitter.marcel` ならMLがMarcelを上回っている。そうでなければMarcelのほうが精度が高い、という判断ができます。

### FastAPI で `/metrics` エンドポイントを追加

`data/metrics/` 以下の JSON を全部読み込んで返すエンドポイントです。

```python
def _load_all_metrics() -> list[dict]:
    if not METRICS_DIR.exists():
        return []
    result = []
    for p in sorted(METRICS_DIR.glob("metrics_*.json")):
        with open(p, encoding="utf-8") as f:
            result.append(json.load(f))
    return sorted(result, key=lambda x: x.get("year", 0))

all_metrics = _load_all_metrics()

@app.get("/metrics")
def get_metrics():
    if not all_metrics:
        raise HTTPException(503, "メトリクスデータがありません")
    return {"件数": len(all_metrics), "メトリクス": all_metrics}
```

年が増えるたびに自動で追記されます。複数年分が溜まれば精度の推移をグラフにできます。

## ③ GitHub Actions で全部自動化する

`annual_update.yml` の全体構成：

```yaml
name: Annual NPB Update

on:
  schedule:
    - cron: '0 9 1 3 *'   # 毎年3月1日 9:00 UTC（FA・移籍確定後、開幕前）
  workflow_dispatch:         # 手動実行も可
    inputs:
      data_end_year:
        description: 'Last season year (例: 2025)'
        default: ''

permissions:
  contents: write  # git push するために必要

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install pandas numpy beautifulsoup4 requests lxml scikit-learn lightgbm xgboost joblib

      - name: 1. Fetch hitter/pitcher stats
        run: python fetch_npb_data.py

      - name: 2. Fetch detailed batting stats
        run: python fetch_npb_detailed.py

      - name: 3. Fetch standings + Pythagorean
        run: python pythagorean.py

      - name: 4. Calculate wOBA/wRC+
        run: python sabermetrics.py

      - name: 5. Marcel projections
        run: python marcel_projection.py

      - name: 6. ML projections (LightGBM/XGBoost)
        run: python ml_projection.py

      - name: Commit and push updated data
        run: |
          git config user.name 'github-actions[bot]'
          git config user.email 'github-actions[bot]@users.noreply.github.com'
          git add data/
          if git diff --staged --quiet; then
            echo "No data changes to commit"
          else
            git commit -m "auto: update NPB data to ${NPB_DATA_END_YEAR}"
            git push
          fi
```

`git add data/` だけで `data/models/*.pkl` と `data/metrics/*.json` も一緒にコミットされます。

## 実際に動かしたら踏んだバグ

ローカルで動いていたコードをCIに乗せると、今まで見えなかったデータ品質の問題が出てきます。今回は4連続でバグが出ました。

### Bug 1: StringDtype のまま数値演算

```
TypeError: can't multiply sequence by non-int of type 'str'
```

スクレイピングしたデータの `AVG` / `OBP` 等が文字列型のまま掛け算に使われていました。

```python
# Before: RC27/XR27 しか変換していなかった
for col in ["RC27", "XR27"]:
    df[col] = pd.to_numeric(df[col], errors="coerce")

# After: 使う列を全部変換する
for col in ["AVG", "OBP", "SLG", "OPS", "PA", "HR", ..., "RC27", "XR27"]:
    df[col] = pd.to_numeric(df[col], errors="coerce")
```

### Bug 2: NaN を `== 0` でスキップできない

```
ValueError: cannot convert float NaN to integer
```

Bug 1 の修正で無効な値が NaN になりましたが、`if pa == 0:` では NaN をスキップできません。

```python
float('nan') == 0  # → False（スキップされない）
```

```python
# Before
if pa == 0:
    continue

# After
if pd.isna(pa) or pa == 0:
    continue
```

### Bug 3: テストデータが空で predict がクラッシュ

```
ValueError: Input data must be 2 dimensional and non empty.
```

NaN が連鎖して、ホールドアウト用のテストデータが 0 件になっていました。

```python
# テストデータが空でも落ちないようにガードを追加
if len(X_test) > 0:
    pred = model.predict(X_test)
    mae = mean_absolute_error(y_test, pred)
    results[name] = {"model": model, "pred": pred, "mae": mae}
else:
    print("WARNING: テストデータが空。学習のみ実施。")
    results[name] = {"model": model}  # MAEなし、モデルのみ保存
```

### Bug 4: github-actions[bot] の書き込み権限がない

```
remote: Permission to ... denied to github-actions[bot].
fatal: unable to access ...: The requested URL returned error: 403
```

GitHub Actions のデフォルトトークンは読み取り専用です。ワークフローに権限を明示する必要があります。

```yaml
# ジョブの外（ワークフローレベル）に書く
permissions:
  contents: write
```

---

4つとも「ローカルでは動いていた」パターンです。CI に乗せることで初めて発覚しました。

## まとめ

追加した変更をまとめると：

| ファイル | 変更内容 |
|---|---|
| `ml_projection.py` | `joblib` でモデル保存、精度を `metrics_*.json` に出力 |
| `api.py` | `/metrics` エンドポイント追加 |
| `requirements.txt` | `joblib>=1.3` 追加 |
| `.github/workflows/annual_update.yml` | 新規作成（8ステップ、毎年3月1日実行）|
| `fetch_rosters.py` | NPB支配下登録選手一覧を取得（Marcel予測から退団・MLB移籍選手を除外するため） |

実行すると `data/` 以下に以下のファイルが生成され、そのままGitにコミットされます：

```
data/models/lgb_hitters_2026.pkl
data/models/xgb_hitters_2026.pkl
data/models/lgb_pitchers_2026.pkl
data/models/xgb_pitchers_2026.pkl
data/metrics/metrics_2026.json
```

複数年分溜まれば精度の推移を追えるようになります。「MLOps」と呼べるかはまだ怪しいですが、少なくとも「スクリプトを手動実行する運用」からは脱却できました。

→ **GitHub**: https://github.com/yasumorishima/npb-prediction

---

## 関連記事

- [Marcel法の限界を超えたい — NPB予測にベイズ回帰を導入した15ステップの記録](https://zenn.dev/shogaku/articles/npb-bayes-projection-story)
