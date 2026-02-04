---
title: "pybaseballに1日でPR 7件出した話（未マージ・メンテ低頻度リポジトリへのOSS貢献）"
emoji: "⚾"
type: "tech"
topics: ["python", "oss", "pybaseball", "baseball", "github"]
published: true
---

## はじめに

[pybaseball](https://github.com/jldbc/pybaseball)というPythonライブラリにOSS貢献として1日でPR 7件 + Issueコメント 3件を出しました。

**重要な前提**: 2026年2月4日時点で**PRはすべて未マージ**です。pybaseballはメンテナンスの頻度が低く（数ヶ月に1回程度のマージペース）、マージされるかどうかも分かりません。この記事は「マージされた成功体験」ではなく、「どうやってissueを選んで修正したか」の記録です。

## pybaseballとは

pybaseballは、メジャーリーグの統計データをPythonで簡単に取得できるライブラリです。GitHub Stars 1,500以上。

主なデータソース:
- **Statcast**: MLB公式のピッチングデータ（球速、回転数、arm angleなど）
- **Baseball Reference**: チーム成績、選手成績
- **FanGraphs**: 高度な分析指標（WAR、wRC+など）

```python
from pybaseball import statcast, batting_stats, pitching_stats

# Statcastデータ取得
df = statcast('2024-07-01', '2024-07-01')

# シーズン打撃成績
batting = batting_stats(2024)

# チーム別の成績
from pybaseball import schedule_and_record
schedule = schedule_and_record(2024, 'NYY')
```

野球データ分析をしている人にとっては必須のライブラリですが、メンテナーの対応頻度が低いため、未修正のissueが溜まっている状態でした。

## メンテナンス頻度について

pybaseballは**メンテナンスの頻度が非常に低い**です。

- PRがマージされるまで数ヶ月かかることがある
- issueへのメンテナーの返信も少ない
- 最後のリリースからかなり時間が経っている

この点を理解した上で、「マージされなくても自分の学習になる」「issueを調査すること自体がコミュニティへの貢献」というスタンスで取り組みました。

## 出したPR一覧

| # | PR | Issue | 内容 | 難易度 |
|---|---|---|---|---|
| 1 | [#498](https://github.com/jldbc/pybaseball/pull/498) | #490 | ドキュメントtypo修正 | 簡単 |
| 2 | [#499](https://github.com/jldbc/pybaseball/pull/499) | #467 | `errors='ignore'`非推奨警告修正 | 簡単 |
| 3 | [#500](https://github.com/jldbc/pybaseball/pull/500) | #459 | `team_results.py` FutureWarning修正 | 簡単 |
| 4 | [#501](https://github.com/jldbc/pybaseball/pull/501) | #455 | `retrosheet.py`非推奨GitHub認証修正 | 中程度 |
| 5 | [#502](https://github.com/jldbc/pybaseball/pull/502) | #462 | `team_fielding_bref`入力バリデーション追加 | 中程度 |
| 6 | [#503](https://github.com/jldbc/pybaseball/pull/503) | #461 | `team_batting_bref`/`team_pitching_bref` HTML構造変更対応 | 重い |
| 7 | [#504](https://github.com/jldbc/pybaseball/pull/504) | #486 | `team_ids` 2022以降データ欠損修正 | 中程度 |

加えて、動作確認コメント3件（#470, #477, #481）も投稿しました。

## issueの選び方

20件程度のオープンissueを眺めて、以下の基準で選びました。

### 取り組んだもの
- **非推奨警告の修正**（#459, #467, #455）: 修正方法がWarningメッセージに書いてある。確実に直せる
- **入力バリデーション追加**（#462）: issue作成者が修正方法を提案済み
- **スクレイピング破損**（#461）: エラーメッセージから原因の仮説が立てやすい
- **データ欠損**（#486）: コメントに回避策が書いてあり、それをコードに落とすだけ

### 見送ったもの
- **外部サービス起因**（#492, #479）: FanGraphsが5xxや403を返す。コード側では修正不可
- **FanGraphs APIの挙動**（#469）: サーバー側でパラメータが無視される。コード修正不可
- **機能要望**（#476）: 仕様変更に近い。メンテナーの方針判断が必要
- **不明瞭なissue**（#477）: 情報不足だが、テストしたら現バージョンで動いた

### コツ: 簡単なものから始める

最初にtypo修正（#498）と非推奨警告修正（#499）を出しました。これには2つの理由があります。

1. **リポジトリの慣習を把握できる**（PRの書き方、ブランチ戦略、テストの有無）
2. **難しいissueの調査中に得た知識が活きる**（コードベースに慣れる）

実際、簡単なPRを出す過程でBaseball Referenceのスクレイピング周りのコード構造を把握でき、後の#461（HTML構造変更対応）の調査がスムーズに進みました。

## 一番重かったPR: #503（HTML構造変更対応）

`team_batting_bref()`と`team_pitching_bref()`が`IndexError`で完全に壊れているissue（#461）の修正です。

### 調査プロセス

1. **エラーの場所を特定**: `soup.find_all('table', {'class': 'sortable stats_table'})[0]`で空リストが返る
2. **仮説**: Baseball ReferenceがHTMLの構造を変えた
3. **検証**: 実際にHTMLを取得して調べたところ、テーブルのIDが変わっていた

```
旧: id="team_batting",     class="sortable stats_table"
新: id="players_standard_batting", class="stats_table sortable soc"
```

4. **修正**: テーブルIDを新しいものに更新 + ヘッダー取得を動的化（ハードコードされた`[1:28]`を`thead`からの動的取得に変更）
5. **動作確認**: ローカルで`team_batting_bref('NYY', 2023)`を実行して正常にデータ取得を確認

### ポイント: 同種の関数を比較する

同じリポジトリ内の`team_fielding_bref()`は正常に動いていました。この関数はHTMLコメント内からテーブルを抽出する方式を使っていて、Baseball Referenceの変更に影響されていませんでした。

「動いている関数」と「壊れている関数」を比較することで、原因の切り分けが早くできました。

## Issueコメントも貢献になる

PRだけがOSS貢献ではありません。以下の3件のissueに「現バージョンで動作確認済み」というコメントを残しました。

- **#470**: `schedule_and_record`がエラーになる → `curl_cffi`移行で解決済みと報告
- **#477**: `stats range`が壊れている → 同上
- **#481**: GH_TOKEN未設定でエラー → PR #501で解決済みと報告

issueが放置されていると「まだ壊れているのか」と不安になります。「最新版で確認したら動いた」という一言だけでも、他のユーザーにとっては有益な情報です。

## まとめ

- **pybaseball**: MLBデータ取得のPythonライブラリ。便利だがメンテ低頻度
- **1日でPR 7件 + コメント 3件**を出した。すべて未マージ（2026年2月時点）
- **issueの選び方**: 非推奨警告修正→バリデーション追加→スクレイピング修正の順に難易度を上げた
- **メンテ低頻度リポジトリ**への貢献は、マージされなくてもissue調査やコメント自体がコミュニティへの貢献になる
- **コメントだけでも貢献**: 「現バージョンで動いた」の一言が他のユーザーの助けになる

マージされたら追記します。
