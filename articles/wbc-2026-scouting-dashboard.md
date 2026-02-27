---
title: "WBC 2026 スカウティングダッシュボードとStatcastデータセットを公開した話"
emoji: "⚾"
type: "tech"
topics: ["baseball", "kaggle", "streamlit", "python", "データ分析"]
published: true
---

## 作ったもの

WBC 2026（ワールドベースボールクラシック）に向けて、MLBのStatcastデータを使ったスカウティングツールを3つ作りました。

1. **Streamlit スカウティングダッシュボード** — 全20チーム・30アプリ（打者17 + 投手13）
2. **Kaggle データセット** — 20カ国・54万投球以上のStatcastデータ
3. **Kaggle EDA ノートブック** — 国別ファストボール球速などの分析

## 🌐 ランディングページ

**[https://wbc-2026-scouting-dashboard-zvg.caffeine.xyz/](https://wbc-2026-scouting-dashboard-zvg.caffeine.xyz/)**

全チームのダッシュボードリンクをプール別にまとめたランディングページ（ICP / Caffeine 上でホスト、英語・日本語対応）。

---

## 1. スカウティングダッシュボード

**GitHub**: https://github.com/yasumorishima/wbc-scouting

WBC 2026ロスターに選ばれた選手のうち、MLB登録のある選手のStatcastデータを可視化しています。

- **打者ダッシュボード**: 17カ国・105選手
- **投手ダッシュボード**: 13カ国・86選手

各チームごとに独立したStreamlitアプリとして公開しています（計30アプリ）。

### 打者ダッシュボード

ページを開いた瞬間に**OPS上位3選手のハイライトカード**と**チーム打線のレーダーチャート**（打率・出塁率・長打率・三振率・四球率の5軸）が表示されます。選手を選ばなくても、チームの打線の特徴が一目で分かります。

スプレーチャート・打球速度・打球角度・カウント別成績などを表示します。

![スプレーチャート](/images/wbc-batter-spray-chart.png)

ストライクゾーンを9分割し、コース別の打撃成績をヒートマップで確認できます。各セクションには用語のキャプションが付いており、野球に詳しくなくても読めるようになっています。

![ゾーンヒートマップ](/images/wbc-batter-zone-heatmap.png)

### 投手ダッシュボード

ページを開いた瞬間に**K%上位3選手のハイライトカード**と**チーム投手陣のレーダーチャート**（K%・Whiff%・BB%・被打率・xwOBA・球速の6軸）が表示されます。

球種ごとのロケーション分布、L/Rスプリット、カウント別の傾向を表示します。

![ピッチロケーション](/images/wbc-pitcher-location.png)

球種ごとの変化量（水平・垂直方向）をプロットします。球速との関係も確認できます。

![変化量チャート](/images/wbc-pitcher-movement.png)

### ダッシュボード URL一覧

| チーム | 打者 | 投手 |
|---|---|---|
| USA | [wbc-usa-batters](https://wbc-usa-batters.streamlit.app/) | [wbc-usa-pitchers](https://wbc-usa-pitchers.streamlit.app/) |
| Japan | [wbc-japan-batters](https://wbc-japan-batters.streamlit.app/) | [wbc-japan-pitchers](https://wbc-japan-pitchers.streamlit.app/) |
| Dominican Republic | [wbc-dr-batters](https://wbc-dr-batters.streamlit.app/) | [wbc-dr-pitchers](https://wbc-dr-pitchers.streamlit.app/) |
| Mexico | [wbc-mex-batters](https://wbc-mex-batters.streamlit.app/) | [wbc-mex-pitchers](https://wbc-mex-pitchers.streamlit.app/) |
| Puerto Rico | [wbc-pr-batters](https://wbc-pr-batters.streamlit.app/) | [wbc-pr-pitchers](https://wbc-pr-pitchers.streamlit.app/) |
| Korea | [wbc-kor-batters](https://wbc-kor-batters.streamlit.app/) | [wbc-kor-pitchers](https://wbc-kor-pitchers.streamlit.app/) |
| Netherlands | [wbc-ned-batters](https://wbc-ned-batters.streamlit.app/) | [wbc-ned-pitchers](https://wbc-ned-pitchers.streamlit.app/) |
| Canada | [wbc-can-batters](https://wbc-can-batters.streamlit.app/) | [wbc-can-pitchers](https://wbc-can-pitchers.streamlit.app/) |
| Italy | [wbc-ita-batters](https://wbc-ita-batters.streamlit.app/) | [wbc-ita-pitchers](https://wbc-ita-pitchers.streamlit.app/) |
| Israel | [wbc-isr-batters](https://wbc-isr-batters.streamlit.app/) | [wbc-isr-pitchers](https://wbc-isr-pitchers.streamlit.app/) |
| Great Britain | [wbc-gb-batters](https://wbc-gb-batters.streamlit.app/) | [wbc-gb-pitchers](https://wbc-gb-pitchers.streamlit.app/) |
| Panama | [wbc-pan-batters](https://wbc-pan-batters.streamlit.app/) | [wbc-pan-pitchers](https://wbc-pan-pitchers.streamlit.app/) |
| Colombia | [wbc-col-batters](https://wbc-col-batters.streamlit.app/) | [wbc-col-pitchers](https://wbc-col-pitchers.streamlit.app/) |
| Cuba | [wbc-cuba-batters](https://wbc-cuba-batters.streamlit.app/) | — |
| Chinese Taipei | [wbc-twn-batters](https://wbc-twn-batters.streamlit.app/) | — |
| Nicaragua | [wbc-nic-batters](https://wbc-nic-batters.streamlit.app/) | — |
| Australia | [wbc-aus-batters](https://wbc-aus-batters.streamlit.app/) | — |

:::message
**Streamlitのスリープについて**

しばらくアクセスがないとアプリがスリープします（「Zzzz」「Your app is in the oven」という画面）。その場合は数秒待つか、ページをリロードすると復帰します。
:::

---

## 2. Kaggle データセット

https://www.kaggle.com/datasets/yasunorim/wbc-2026-scouting

![データセットページ](/images/wbc-kaggle-dataset.png)

WBC 2026ロスター選手（MLB登録あり）のStatcastデータをまとめたデータセットです。データは [Baseball Savant](https://baseballsavant.mlb.com/) から [pybaseball](https://github.com/jldbc/pybaseball) で取得しました。

### 収録データ

| ファイル | 内容 |
|---|---|
| `statcast_batters.csv`（36MB） | 打者：324,099投球、18カ国 |
| `statcast_pitchers.csv`（29MB） | 投手：217,139投球、14カ国 |
| `batter_summary.csv` | 打者サマリー：105選手・19カ国 |
| `pitcher_summary.csv` | 投手サマリー：86選手・14カ国 |
| `rosters.csv` | 全ロスター：309選手・20カ国 |
| `stadiums.csv` | MLB球場座標（スプレーチャート描画用） |

---

## 3. Kaggle EDA ノートブック

https://www.kaggle.com/code/yasunorim/wbc-2026-scouting-eda-statcast-analysis

カ国別のファストボール平均球速や打球速度など、データセットの概観を可視化しています。

![ノートブックのグラフ出力](/images/wbc-notebook-graph.png)

---

## おわりに

ロスターの情報は Baseball America（2026年2月）の発表をもとにしています。選手の漏れや誤りがある可能性もあります。何かお気づきの点があればコメントいただけると助かります。

データは MLB レギュラーシーズンのみで、WBC での実際のパフォーマンスとは異なる場合があります。参考のひとつとして使っていただければ幸いです。

---

## リンク

- **ダッシュボード（GitHub）**: https://github.com/yasumorishima/wbc-scouting
- **データセット（Kaggle）**: https://www.kaggle.com/datasets/yasunorim/wbc-2026-scouting
- **EDA ノートブック（Kaggle）**: https://www.kaggle.com/code/yasunorim/wbc-2026-scouting-eda-statcast-analysis
