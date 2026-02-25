---
title: "Marcel法がLightGBMを上回った話 — NPB選手成績予測システムを作った"
emoji: "⚾"
type: "tech"
topics: [python, baseball, machinelearning, データ分析, fastapi]
published: true
---

## はじめに

「来年、この選手はどのくらいの成績を残せるか？」

この問いに答えるため、NPB（日本プロ野球）の選手成績予測システムを作りました。

→ **GitHub**: https://github.com/yasumorishima/npb-prediction

予測手法は2種類用意しました。

- **Marcel法**: 1980年代に考案されたシンプルな統計手法
- **LightGBM / XGBoost**: 現代的な機械学習

結果として、**Marcel法がMLを上回りました**。これは先行研究で知られた事実ですが、NPBデータで自分でも実証できた点が面白い発見でした。

---

## データ取得

データソースは2つです。

- [プロ野球データFreak（baseball-data.com）](https://baseball-data.com) — 打者・投手の年度別成績（2015-2025）
- [NPB公式サイト（npb.jp）](https://npb.jp) — 詳細打撃成績（2塁打・3塁打・犠飛、wOBA算出用）

`pandas.read_html()` でHTMLテーブルを取得しています。取得したのは以下のデータです。

| ファイル | 内容 |
|---|---|
| 打者成績 | 3,780行（2015-2025） |
| 投手成績 | 3,773行（2015-2025） |
| 順位表 | 132行（12球団×11年） |
| 生年月日 | 2,479人 |
| 詳細打撃成績 | 4,538行（npb.jp） |

データの提供元：[プロ野球データFreak](https://baseball-data.com)、[NPB公式サイト](https://npb.jp)

---

## Marcel法とは

Tom Tangoが考案した成績予測手法です。実装はシンプルです。

### ステップ1: 過去3年の加重平均

直近の成績ほど重要という考えのもと、**5/4/3** の比率で加重平均します。

```python
weight_map = {0: 5, 1: 4, 2: 3}  # 0=直近、1=2年前、2=3年前
```

### ステップ2: リーグ平均への回帰

サンプルが少ない選手（出場機会が少ない）は、リーグ平均に引き戻します。

```python
# 打者OPS: リーグ平均への回帰係数
regression = 1200 / (pa + 1200)
predicted = (1 - regression) * weighted + regression * league_avg
```

### ステップ3: 年齢調整

ピーク年齢（27歳）からの距離に応じて成績を増減させます。

```python
age_factor = 1.0 + (27 - age) * 0.003  # 27歳でピーク
```

---

## wOBA / wRC+ を自前算出

打者の総合得点貢献を測る指標 **wOBA** と、リーグ平均を100とした **wRC+** を、NPBの公式データから自前で算出しました。

MLB（Baseball Savant等）と違い、NPBには公式のwOBA公開値がないため、各イベントに重みをつけて計算しています。

```python
# wOBA計算（NPBリーグ環境に合わせた係数）
woba = (
    0.69 * BB + 0.72 * HBP +
    0.89 * H1B + 1.27 * H2B +
    1.62 * H3B + 2.10 * HR
) / (AB + BB + HBP + SF)
```

### 2024年 wRC+ Top3（自前算出）

| 選手 | チーム | wOBA | wRC+ |
|---|---|---|---|
| 近藤健介 | ソフトバンク | .479 | 249 |
| オースティン | DeNA | .478 | 248 |
| サンタナ | 楽天 | .441 | 220 |

---

## 機械学習（LightGBM / XGBoost）

特徴量には年齢・過去成績に加え、先ほどのwOBA/wRC+も追加しました。

```python
features = [
    'age', 'OPS_prev1', 'OPS_prev2', 'OPS_prev3',
    'woba', 'wrc_plus',  # セイバー指標追加
    'PA_prev1', 'PA_prev2'
]
```

LightGBMとXGBoostの両方を試しましたが、精度はほぼ同等でした。

---

## 精度比較（2024年バックテスト）

2015-2023年のデータで学習し、2024年実績と比較しました。

### 打者OPS

| 手法 | MAE | 対象 |
|---|---|---|
| **Marcel法** | **.055** | 規定打席以上 |
| LightGBM | .077 | 規定打席以上 |

### 投手ERA

| 手法 | MAE | 対象 |
|---|---|---|
| **Marcel法** | **0.62** | 100投球回以上 |
| LightGBM | 0.95 | 100投球回以上 |

**Marcel法がMLを大きく上回りました。**

wOBA/wRC+を特徴量に追加した後も傾向は変わらず。先行研究が指摘する「Marcel法は驚くほど強いベースライン」という評価を、NPBデータでも確認できました。

なぜMarcel法が強いのか、考えられる理由は以下です。

- 選手の真の実力は1〜2年程度しか変化しない
- MLは過学習しやすく、サンプル数が限られると精度が落ちる
- シンプルな加重平均が、実際の成績変化の分布にフィットしている

---

## ピタゴラス勝率

チームの強さを「得点と失点の比率」で予測する手法です。

```
勝率 ≈ 得点^k / (得点^k + 失点^k)
```

指数 `k` はMLBでは `1.83` が標準ですが、NPBでの最適値を探したところ **`k=1.72`** でした。

| 指数 | MAE | 対象 |
|---|---|---|
| NPB最適（k=1.72） | **3.20勝** | 全12球団（2015-2025） |
| MLB標準（k=1.83） | 3.32勝 | 同上 |

---

## FastAPIで推論APIを構築

予測をAPIとして提供できるようにしました。`pip install -r requirements.txt` → `uvicorn api:app --reload` で起動し、`http://localhost:8000/docs` からSwagger UIで試せます。

### エンドポイント一覧

| パス | 内容 |
|---|---|
| `GET /predict/hitter/{name}` | 打者の翌年予測（Marcel + ML） |
| `GET /predict/pitcher/{name}` | 投手の翌年予測（Marcel + ML） |
| `GET /predict/team/{name}` | チームのピタゴラス勝率 |
| `GET /sabermetrics/{name}` | wOBA / wRC+ / wRAA |
| `GET /rankings/hitters` | 打者ランキング |
| `GET /rankings/pitchers` | 投手ランキング |
| `GET /pythagorean` | 全チームのピタゴラス勝率 |

### レスポンス例（牧秀悟 2025年予測）

```json
{
  "player": "牧 秀悟",
  "team": "DeNA",
  "marcel": { "OPS": 0.834, "AVG": 0.295, "HR": 22.9, "RBI": 81.4 },
  "ml": { "pred_OPS": 0.874 }
}
```

Docker対応もしており、`docker compose up --build` で一発起動できます。

---

## まとめ

| 項目 | 内容 |
|---|---|
| データ | baseball-data.com + npb.jp（2015-2025、5種類） |
| Marcel法精度 | 打者OPS MAE=.055 / 投手ERA MAE=0.62 |
| ML精度 | 打者OPS MAE=.077 / 投手ERA MAE=0.95 |
| ピタゴラス勝率 | NPB最適 k=1.72、MAE=3.20勝 |
| API | FastAPI 7エンドポイント、Docker対応 |

「最新のAIが必ずしも勝つわけではない」という事実をNPBデータで実証できたのが一番の収穫でした。

→ **GitHub**: https://github.com/yasumorishima/npb-prediction

**データ出典**: [プロ野球データFreak](https://baseball-data.com) / [NPB公式サイト](https://npb.jp)
