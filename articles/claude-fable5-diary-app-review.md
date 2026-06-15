---
title: "Claude Fable 5 に個人開発の日記アプリをレビューしてもらった記録 — 見つかったバグと所感"
emoji: "📔"
type: "tech"
topics: ["claude", "claudecode", "flutter", "gas", "ai"]
published: false
---

## はじめに

個人開発で続けている日記アプリ（Flutter のモバイル版と、Google Apps Script で作った連用日記の Web 版）を、Play ストアへ出す前にひととおり品質チェックしたくなりました。そこで Claude Fable 5 にコードレビューと修正を任せてみた、という記録です。

Fable 5 は 2026 年 6 月 9 日に公開され、その 3 日後の 6 月 12 日に米政府の輸出管理指令で全ユーザー向けに利用停止になりました（執筆時点で再開未定）。短い期間でしたが、その間に実際にやってもらった内容と、個人開発者としての所感を残しておきます。

なお「他モデルが見逃したバグを見つけた」といった一般論ではなく、自分のリポジトリのコミットに残っている **実際の修正** だけを書きます。

## 頼んだこと

- Flutter 版: `flutter analyze` の警告解消、テスト整備、CI 追加
- GAS 版: 検索まわりとセキュリティの見直し

「全部直して」ではなく「品質的に気になるところを洗い出して直して」くらいの粒度で投げました。

## 見つかったバグ（モバイル / Flutter）

### 1. async の後に BuildContext を mounted チェックなしで使っていた

`await` をまたいで `setState` や `BuildContext`（ローカライズ参照や `Navigator`）を触る箇所が複数あり、画面が破棄された後に呼ばれるとクラッシュする潜在バグでした。

```dart
// 修正前（概念）: await の後、ウィジェットが破棄されていても触ってしまう
await databaseService.importData(file);
setState(() { /* ... */ });                 // 破棄後だと例外
final l10n = AppLocalizations.of(context)!; // await をまたいで context 参照

// 修正後: await の前に取っておく / ガードを入れる
final l10n = AppLocalizations.of(context)!; // await の前に確定
await databaseService.importData(file);
if (!mounted) return;
setState(() { /* ... */ });
```

通常操作ではめったに踏まないので、自分でテストしていても気づきにくいタイプでした。

### 2. テストがそもそも何もテストしていなかった

テンプレートのまま残っていた `widget_test.dart` が、存在しない `MyApp` を参照していてコンパイルすら通らない状態でした（＝実質テストゼロ）。これをモデルと Hive 永続化の実テスト 12 本に置き換えてもらいました。

ほかに `flutter analyze` の警告 47 件（`withOpacity` 非推奨 → `withValues` への置換 27 箇所、非推奨 ColorScheme API、`RadioGroup` への移行など）を 0 にし、push/PR で analyze とテストを回す CI も追加しました。

## 見つかったバグ（GAS / 連用日記 Web）

### 3. 検索ハイライトでユーザー入力をそのまま正規表現にしていた

検索語を `RegExp` にそのまま渡していたため、`(` や `+`、`*` など正規表現のメタ文字を含む語で検索すると壊れる（マッチが意図どおりにならない／例外）バグでした。

```javascript
// 修正前: ユーザー入力をそのまま RegExp に
const regex = new RegExp('(' + escapedQuery + ')', 'gi');

// 修正後: メタ文字をエスケープしてから
function escapeRegex(text) {
  return text.replace(/[.*+?^${}()|[\]\]/g, '\$&');
}
const regex = new RegExp('(' + escapeRegex(escapedQuery) + ')', 'gi');
```

### 4. エラーページの XSS

エラーページがタイトルやメッセージを生のまま HTML に埋め込んでいました。

```javascript
// 修正前
<h1>⚠️ ${title}</h1>
<div class="error-message">${message}</div>

// 修正後
<h1>⚠️ ${escapeHtml(String(title))}</h1>
<div class="error-message">${escapeHtml(String(message))}</div>
```

### 5. 日記本文がログに出ていた

デバッグ用に残っていた `Logger.log('... ' + JSON.stringify(entries))` 系で、日記の本文が丸ごとログに出力されていました。プライバシー的に良くないので削除しました。

## 個人開発者としての所感

率直に言うと、Fable が見つけたのは「派手な新機能」ではなく、**自分ひとりだと体系的にチェックしない種類のバグ** でした。

- XSS・ログへの本文漏れ → 普通に使う分には見えない
- 正規表現エスケープ → 特定の文字で検索したときだけ顕在化
- async gap の mounted → タイミング次第でしか落ちない
- 壊れたテスト → 「テストがある」つもりで実は無い

機能を書くところまでは自分でもできますが、こういう「条件が揃ったときだけ出る」「セキュリティ／プライバシー」「テストの実効性」あたりは、レビュー専任がいないと抜けがちです。そこを一度しっかり通してくれたという意味で、個人開発の品質チェックには十分有用でした。

一方で限界もありました。

- 直してもらった `escapeRegex` のエスケープ表現が一度欠けていて、追いの修正が必要でした（完璧に一発ではない）。
- 広告表示やエクスポート／インポート共有のような **実機 UI 依存の挙動** は CI では確認できず、最後は自分で実機チェックが必要です。

結論としては、「コードレビュー担当を雇うほどではないけれど、個人開発を一通り見てほしい」という用途にちょうど良い、という体感でした。公開 3 日で止まってしまったのは残念ですが、止まる前にこの一回を通せたのは記録として残しておく価値があると思っています。
