---
title: "NPB予測アプリにベイズ統合+モンテカルロを組み込んだ作業メモ"
emoji: "🎯"
type: "tech"
topics: ["baseball", "npb", "python", "ベイズ統計", "streamlit"]
published: false
---

## はじめに

前回の記事で、NPB選手成績予測にベイズ回帰（Stan/Ridge）を導入する試行錯誤を紹介しました。

https://zenn.dev/shogaku/articles/npb-bayes-projection-story

そのときは別リポ（npb-bayes-projection）で実験していたのですが、今回はメインのアプリ（npb-prediction）に組み込みました。7フェーズに分けた作業メモです。

→ **GitHub**: https://github.com/yasumorishima/npb-prediction
→ **Streamlit**: https://npb-prediction.streamlit.app/

### この記事の対象読者

- 実験コードをアプリに組み込むプロセスに興味がある方
- ベイズ予測の不確実性をUIに反映する方法を知りたい方
- モンテカルロ・シミュレーションをStreamlitで可視化したい方

---

## 統合前の状態

npb-predictionは以下の構成で動いていました。

```
Marcel法（3年加重平均） → ML（XGBoost/LightGBM）
    ↓                        ↓
  独立した点推定          独立した点推定
    ↓
ピタゴラス勝率 → チーム順位予測
```

**課題:**
- 予測は全て点推定。不確実性の表現がない
- 新外国人選手（24人）は wRAA=0（リーグ平均）で処理。前リーグ成績を活かせていない
- Marcel / ML が独立動作。統合的なアンサンブルがない
- チーム順位予測にも不確実性がない

## 統合後のアーキテクチャ

```
Layer 1: Marcel法（基盤 — 変更なし）
    ↓
Layer 2: Stan ベイズ補正
  - 日本人: K%/BB%/BABIP/年齢 の Ridge 補正
  - 外国人: 前リーグ成績 × リーグ別変換係数（Stan v2）
    ↓
Layer 3: ML（XGBoost/LightGBM）
    ↓
Layer 4: BMA（ベイズモデル平均）
  - Marcel 35% + Stan 40% + ML 25%
  - 全予測に 80%/95% 信頼区間
    ↓
モンテカルロ 10,000回 → チーム勝数分布
  - P(優勝) / P(CS) / P(最下位)
```

---

## 7フェーズの統合作業

### Phase 1: 日本人選手ベイズ推論

研究リポで得られたStan事後分布パラメータを `posteriors.json` に保存し、本番ではNumPyサンプリングだけで予測する設計にしました。

```python
# posteriors.json の構造（打者の例）
{
  "japanese_hitter": {
    "beta": [0.152, -0.089, -0.245, -0.003],  # K%, BB%, BABIP, 年齢
    "sigma_residual": 0.06215,
    "feature_names": ["K_pct", "BB_pct", "BABIP", "age_from_peak"],
    "scaler_mean": [18.34, 8.51, 0.296, 0.67],
    "scaler_std": [5.12, 2.89, 0.027, 4.23]
  }
}
```

**重要な設計判断: Stanはランタイムで動かさない。**

cmdstanpyはインストールが重く、Raspberry Pi 5（RAM 4GB）では厳しい。posteriors.jsonに事後分布パラメータを事前計算しておけば、推論時はNumPyの乱数生成だけで済みます。

```python
# 推論時のサンプリング（bayes_projection.py）
z_features = (raw_features - scaler_mean) / scaler_std
correction = beta @ z_features
samples = marcel_value + correction + rng.normal(0, sigma, size=5000)
ci_80 = np.percentile(samples, [10, 90])
ci_95 = np.percentile(samples, [2.5, 97.5])
```

### Phase 2: 外国人選手 Stan v2 予測

最も手間がかかったフェーズです。24人の外国人選手全員について、Web検索で以下を個別に検証しました。

- カタカナ名 → 英語名の正確な対応
- 出身リーグ（MLB / KBO / 独立リーグ）
- 前リーグの直近シーズン成績

**教訓: カタカナ名から英語名を推測してはいけない。**

当初28人中10人以上の英語名が間違っていました。

| NPB登録名 | 最初の推測 | 正しい名前 |
|---|---|---|
| ダルベック | Spencer Torkelson | Bobby Dalbec |
| ジェリー | Sean Gerry | Sean Hjelle |
| ルーカス | Josh Lucas | Easton Lucas |
| ディベイニー | Jake Devaney | Cam Devanney |

さらに日本人ドラフト選手4人（カタカナ名のハーフ選手）を外国人と誤認していました。全件Web検索で裏取りしてから初めてコミット、が正しい手順でした。

Stan v2モデルは前リーグ成績をリーグ別変換係数でNPB相当に変換します。

```python
# MLB → NPB 換算
npb_woba = mlb_woba * 1.235  # bootstrap 95%CI付き
npb_era  = mlb_era  * 0.579
```

### Phase 3: モンテカルロ・チームシミュレーション

選手レベルの不確実性をチームレベルに伝播させるために、モンテカルロ法で10,000回シミュレーションしました。

```python
for sim in range(10000):
    for team in teams:
        # 各選手の事後分布から独立にサンプリング
        team_rs = sum(sample_hitter_runs(h) for h in team.hitters)
        team_ra = sum(sample_pitcher_runs(p) for p in team.pitchers)
        # パークファクター補正
        team_rs /= (pf + 1) / 2
        team_ra /= (pf + 1) / 2
        # ピタゴラス勝率（k=1.83）
        wins[team][sim] = 143 * rs**1.83 / (rs**1.83 + ra**1.83)
```

外国人選手はNPBデータがないため、σを1.5倍に拡大して不確実性を大きくしています。

### Phase 5: API統合

FastAPIに3つのエンドポイントを追加しました。

| エンドポイント | 内容 |
|---|---|
| `/predict/hitter/{name}` | ベイズOPS + 80%/95%CI（既存に追加） |
| `/predict/foreign/{name}` | 外国人Stan v2予測（新規） |
| `/standings/simulation` | モンテカルロ順位（新規） |

### Phase 6: Streamlit統合

一番ボリュームが大きかったフェーズです。2,025行のstreamlit_app.pyに以下を追加しました。

**1. ベイズCI表示（既存ページに追加）**

打者予測・投手予測ページに、ベイズ統合予測の値と信頼区間を追加。Plotlyの横棒グラフで80%/95%CIを可視化しました。

```python
def _render_ci_bar(central, lo80, hi80, lo95, hi95, label, color):
    """信頼区間を横棒で可視化"""
    fig = go.Figure()
    # 95% CI（薄い帯）
    fig.add_trace(go.Bar(y=[label], x=[hi95-lo95], base=[lo95], ...))
    # 80% CI（濃い帯）
    fig.add_trace(go.Bar(y=[label], x=[hi80-lo80], base=[lo80], ...))
    # 中央値（ダイヤモンドマーカー）
    fig.add_trace(go.Scatter(x=[central], y=[label], ...))
```

**2. チームシミュレーションページ（新規）**

ファンチャート（各チームの勝数CIを横棒で表示）と確率テーブル（P(優勝)/P(CS)/P(最下位)）を追加。

**3. 外国人選手ページ（新規）**

前リーグ成績とNPB予測を並べて表示。チーム別フィルタ付き。

### Phase 7: BigQuery統合

既存の25テーブルに8テーブルを追加して計33テーブル。

| 追加テーブル | 内容 |
|---|---|
| bayes_hitters / bayes_pitchers | CI付きベイズ予測 |
| foreign_hitters / foreign_pitchers | 外国人Stan v2予測 |
| team_simulation | モンテカルロ結果 |
| foreign_players_master | 外国人マスター |
| foreign_prev_stats | 前リーグ成績 |
| foreign_conversion_factors | リーグ別換算係数 |

---

## 技術的な判断ポイント

### posteriors.json方式のトレードオフ

| | posteriors.json | cmdstanpy runtime |
|---|---|---|
| 推論速度 | NumPyだけ（ミリ秒） | Stan呼び出し（数秒〜） |
| メモリ | 数KB | 数百MB |
| 更新 | GitHub Actionsで年次再学習 | 毎回フィッティング |
| 精度 | 事前計算済みパラメータ | 都度最適化 |

RPi5（4GB RAM）で動かすにはpositoriors.json一択でした。年に1回のデータ更新サイクルなので、毎回フィッティングする必要もありません。

### BMA重み配分の根拠

Marcel 35% + Stan 40% + ML 25% という配分は、8年間のLOO-CV（Leave-One-Out Cross Validation）で決定しています。

- Stan補正が最も安定的にMarcellを改善（97.1%の確率でMAE低下）
- MLは打者OPSではMarcelと同等、投手ERAでは劣後
- 3モデルのBMAが単体よりロバスト

### 名前マッチングの罠

Marcel CSVとセイバーメトリクスCSVで、選手名のスペースが異なりました。

```
Marcel: "中野　拓夢"  （全角スペース \u3000）
Saber:  "中野 拓夢"   （半角スペース）
```

`player_join` カラムでスペースを正規化してマッチングする必要がありました。これに気づくまで463人中236人しかマッチしていませんでした。

---

## 結果

### Streamlitダッシュボードの新機能

- **9ページ構成**（元7ページ + チームシミュレーション + 外国人選手）
- **全選手にベイズCI表示**（BMA統合の選手は80%/95%CI、Marcel法のみの選手はCI非表示）
- **外国人選手24人全員が個別予測**（以前はwRAA=0のリーグ平均扱い）
- **チーム順位に確率的な解釈**が加わった（「巨人の優勝確率42.6%」等）

### ファイル規模

| 項目 | 数値 |
|---|---|
| 新規ファイル | 12（Python 2 + データ10） |
| 変更ファイル | 7（既存コード修正） |
| 追加行数 | +4,087 |
| BigQueryテーブル | 25 → 33 |
| Streamlit行数 | 1,669 → 2,025 |

---

## まとめ

実験リポで試した手法をアプリに組み込む作業は、実験そのものとは別の面倒さがありました。

- **データ品質**: 外国人選手の英語名・前リーグ成績を全件Web検索で裏取り。推測で書いたデータは高確率で間違う
- **アーキテクチャ**: Stanをランタイムで動かさない設計（posteriors.json）で、軽量環境でもベイズ推論を実現
- **UI統合**: 不確実性の可視化（CI棒グラフ、ファンチャート）を既存のダークテーマUIに自然に組み込む

Phase 4（Stan学習パイプラインの年次自動化）は来シーズン向けに残していますが、予測システムとしてはベイズ統合が完了し、点推定から確率分布への移行が実現しました。

→ **Streamlit**: https://npb-prediction.streamlit.app/
→ **GitHub**: https://github.com/yasumorishima/npb-prediction

## データソース

- [プロ野球データFreak](https://baseball-data.com) — NPB選手成績
- [日本野球機構 NPB](https://npb.jp) — 公式データ
