---
title: "Kaggleはコンペだけじゃない：MLB分析ノートブックを公開してみた"
emoji: "📊"
type: "tech"
topics: ["kaggle", "python", "データ分析", "baseball", "pybaseball"]
published: true
---

## はじめに

Kaggleといえばコンペティション（機械学習コンテスト）のイメージが強いですが、**コンペに関係ないデータ分析ノートブックも公開できる**ことを最近知りました。

自分の場合、MLB（メジャーリーグ）の投球データを分析するノートブックをGoogle Colabで作っていたのですが、「これKaggleに公開したらどうだろう？」と試してみたところ、思った以上にリアクションがありました。

この記事では、分析ノートブックをKaggleで公開するまでの流れと気づきを共有します。

---

## 公開した分析ノートブック

MLB Statcastデータ（投球の球速・変化量・リリースポイントなどの詳細データ）を使って、日本人投手4人の投球傾向を分析しました。

| # | 投手 | 分析内容 |
|---|---|---|
| 1 | ダルビッシュ有 | 2021-2025の投球変化（球種割合・変化量の推移） |
| 2 | 今永昇太 | 2024→2025の2年目の変化 |
| 3 | 千賀滉大 | 「お化けフォーク」の分析（FF vs FO比較） |
| 4 | 菊池雄星 | 2019-2025のスライダー使用率変化（7シーズン） |

いずれもpybaseball + DuckDB + matplotlib/seabornで分析し、Google Colabで実行したものをKaggleに公開しました。

### 結果

4本中2本がNotebookメダル（ブロンズ）を獲得しました。残り2本はメダルなしです。

コンペのスコアで競うメダルとは違い、**Notebookメダルは他のユーザーからのupvote（投票）数で決まります**。内容が参考になると思ってもらえたかどうか、というシンプルな基準です。

---

## 公開の手順

### 1. ノートブックを用意する

普段使っているGoogle ColabやJupyter Notebookの `.ipynb` ファイルがそのまま使えます。特別な書き換えは不要です。

### 2. Kaggleにアップロード

Kaggleの「Code」→「New Notebook」からアップロードできます。

CLIを使う場合は `kernel-metadata.json` を作成して `kaggle kernels push` でも公開できます。

```json
{
  "id": "your-username/your-notebook-slug",
  "title": "Your Notebook Title",
  "code_file": "your-notebook.ipynb",
  "language": "python",
  "kernel_type": "notebook",
  "is_private": "false",
  "enable_gpu": "false",
  "enable_internet": "true",
  "competition_sources": [],
  "dataset_sources": []
}
```

### 3. データソースについて

コンペのデータを使う場合は `competition_sources` にslugを指定しますが、**外部データ（API取得など）を使う場合は空で問題ありません**。

自分のMLB分析ではpybasballでStatcastデータをAPI経由で取得していたため、`enable_internet: "true"` にしてKaggle上でもデータ取得できるようにしました。ただし、**ノートブックの実行に時間がかかる場合はColabで実行済みの出力付きノートブックをそのままアップロードする方が確実**です。

### 4. 公開設定

`is_private: "false"` にすればpublic公開されます。他のユーザーがノートブックを閲覧・実行できるようになります。

---

## 気づいたこと

### コンペ以外のノートブックにも需要がある

Kaggleにはコンペ参加者だけでなく、データ分析の手法やツールの使い方を学びに来ている人も多いようです。特定の分野（スポーツ分析など）に興味がある人の目に留まることもあります。

### 英語圏のユーザーが多い

Kaggleのユーザーは圧倒的に英語圏が多いので、ノートブックのタイトルやmarkdownセルを英語で書くと見てもらいやすい傾向があります。日本語だけだと検索に引っかかりにくいかもしれません。

### メダルが取れるかは分からない

4本公開して2本がブロンズ、2本がメダルなしという結果でした。同じ形式・同じクオリティで作っても結果は違います。タイミングやテーマの注目度によるところが大きいと感じます。

---

## まとめ

- Kaggleはコンペだけでなく、**自分の分析ノートブックを公開する場**としても使える
- Google ColabやJupyter Notebookの `.ipynb` をそのままアップロードできる
- upvoteがもらえるとNotebookメダルにつながることもある
- コンペに参加していなくてもKaggleを活用する方法がある

「データ分析はしているけどKaggleのコンペはハードルが高い」という方にとって、分析ノートブックの公開は始めやすい選択肢だと思います。

---

**公開したノートブック:**
- [Senga Ghost Fork Analysis 2023-2025](https://www.kaggle.com/code/yasunorim/senga-ghost-fork-analysis-2023-2025)
- [Kikuchi Slider Revolution 2019-2025](https://www.kaggle.com/code/yasunorim/kikuchi-slider-revolution-2019-2025)
- [Darvish Pitching Evolution 2021-2025](https://www.kaggle.com/code/yasunorim/darvish-pitching-evolution)
- [Imanaga Rookie to Sophomore Pitching](https://www.kaggle.com/code/yasunorim/imanaga-rookie-to-sophomore-pitching)
