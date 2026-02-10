---
title: "非エンジニアがClaude Codeを使って1ヶ月でできたこと"
emoji: "📱"
type: "idea"
topics: ["claudecode", "claude", "生成ai", "flutter", "ポエム"]
published: true
---

## はじめに

私はエンジニアではありません。仕事でコードを書くわけでも、情報系の学科出身でもない。いわゆる「趣味でコードを触ってきた」タイプの人間です。

そんな私が、2025年末から2026年初頭にかけてClaude Codeを使い始めた結果、思っていたよりずっと多くのことが動いてしまいました。特別な体験談というより、「こんな人間でもこういうことができた」という記録です。

---

## Claude Codeとは

ターミナルで動くAnthropicのAIエージェントです。コードの読み書きだけでなく、ファイル操作、git操作、コマンド実行まで自律的に行ってくれます。「このバグを直して」「この機能を追加して」と日本語で指示すると、ファイルを読んで、修正して、テストまで走らせます。

---

## できたこと

### 1. Flutterアプリをリリースした

いちばん大きかったのはこれです。

**Daily Diary（連用日記）** というAndroidアプリを、Google Playにリリースしました。

https://play.google.com/store/apps/details?id=com.diary.daily

連用日記というのは、1ページに複数年分の同じ日の日記を並べて表示するフォーマットです。「去年の今日、自分は何を書いていたか」がすぐ確認できる。紙の日記では昔からある形式ですが、スマホアプリとしてシンプルに作りたいと思っていました。

Flutterはほとんど初めてでしたが、Claude Codeに「こういうアプリを作りたい」から始めて、機能ごとに相談しながら進めました。タグ入力、データエクスポート、AdMob広告の組み込み、多言語対応（日英中韓西）、アダプティブアイコンの設定まで。アイコンの表示問題だけでv1.0.7からv1.0.13まで引っ張りましたが、最終的には解決しています。

非エンジニアがFlutterアプリをゼロからGoogle Playに出すというのは、以前は考えていませんでした。Claude Codeがなければ、おそらく途中で詰まって止まっていたと思います。

---

### 2. OSSにプルリクエストを出した

プログラマーが集まるOSSプロジェクトに外から参加することは、ハードルが高いイメージがありました。

ところがClaude Codeと一緒にコードを読むと、「ここのエラーメッセージが不親切だな」「この処理、将来的に非推奨になるやつだ」という箇所が見えてくるようになりました。修正内容をClaude Codeと確認しながらPRを出す、というサイクルを繰り返したところ、以下のような実績になりました。

- **team-mirai-volunteer/action-board**: 11 Merged（バグ修正、リファクタリング、テスト追加など）

「どこを直せばいいか探す」「修正内容が適切か確認する」「PRの文章を書く」、これらをClaude Codeと一緒に行いました。私が判断して、Claude Codeが実装を手伝う、という分担です。

---

### 3. Kaggleにデータセットを4本公開した

野球（MLB）のデータを扱うKaggleデータセットを4本作成・公開しました。

- [Dataset 1: Japanese MLB Players Statcast (2015-2025)](https://www.kaggle.com/datasets/yasunorim/japan-mlb-pitchers-batters-statcast)
- [Dataset 2: MLB Bat Tracking (2024-2025)](https://www.kaggle.com/datasets/yasunorim/mlb-bat-tracking-2024-2025)
- [Dataset 3: MLB Pitcher Arsenal Evolution (2020-2025)](https://www.kaggle.com/datasets/yasunorim/mlb-pitcher-arsenal-2020-2025)
- [Dataset 4: MLB Statcast + Bat Tracking (2024-2025)](https://www.kaggle.com/datasets/yasunorim/mlb-statcast-bat-tracking-2024-2025)（約144万行）

データ取得スクリプト、Column Descriptionの一括入力スクリプト、分析ノートブック、ブログ記事まで、Claude Codeとセットで進めました。

---

### 4. 技術ブログを15本書いた

この1〜2ヶ月で、ZennとQiitaに技術記事を15本公開しています。

分析結果の解説、ライブラリの使い方、OSSへの参加レポート、Flutter開発のハマりどころ、GitHub Actionsの設定など。内容はClaude Codeとの作業の中から生まれたものがほとんどです。

記事を書くこと自体もClaude Codeに手伝ってもらっています。構成の相談、下書きのレビュー、コードブロックの整形など。

---

## 感じたこと

**「分からないまま進める」ができるようになった**

以前は、知らない技術領域に踏み込もうとすると、まず調べることから始めて、そのまま調べるだけで終わることがありました。Claude Codeと使い始めてから、「とりあえず動くものを作りながら学ぶ」というサイクルが回るようになりました。

**任せすぎない方がいい**

Claude Codeが出してきたコードを全部信頼するのは危険です。エラーが出たとき、自分でも何が起きているか把握していないと、同じ間違いを繰り返します。コードを読む習慣は手放さない方がいいと感じています。

**「自分が何をしたいか」を言語化するのは自分の仕事**

これが意外と難しかった。Claude Codeは指示されたことをやりますが、「何をすべきか」を決めるのは自分です。目的が曖昧なままだと、AIが作業を進めても結果がずれます。

---

## 2026年に向けて

Daily DiaryのiOS版、次のデータセット、次のOSSプロジェクト、と積み残しはあります。

「非エンジニアだから」という制限は、以前ほど感じなくなりました。それがClaude Codeを使ってみて、いちばん変わったことかもしれません。

---

アプリはこちら（Android）:
https://play.google.com/store/apps/details?id=com.diary.daily
