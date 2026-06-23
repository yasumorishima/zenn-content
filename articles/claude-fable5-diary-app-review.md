---
title: "個人開発の日記アプリ「Daily Diary」を Claude Fable 5 でレビューしてもらった記録 — 見つかったバグと所感"
emoji: "📔"
type: "tech"
topics: ["claude", "claudecode", "flutter", "個人開発", "ai"]
published: true
---

## はじめに

個人開発で作っている日記アプリ **「Daily Diary」**（Flutter 製、Google Play 公開中）を、リリース前にひととおり品質チェックしたくなり、Claude Fable 5 にコードレビューと修正を任せてみました。その記録です。

Fable 5 は公開からわずか 3 日（2026 年 6 月 9 日 → 6 月 12 日）で、米商務省の輸出管理指令により全世界で利用停止になったモデルですが、止まる直前にこの作業を通せました。ここでは「他モデルが見逃したバグを見つけた」式の一般論ではなく、自分のリポジトリのコミットに残っている **実際の差分** を載せていきます。

## アプリ: Daily Diary

シンプルなオフライン日記アプリです。

- 📱 **オフラインファースト**: データは端末内のみに保存、クラウド不要
- 🌐 **5 言語対応**: 日本語 / 英語 / 中国語 / 韓国語 / スペイン語
- 🌙 ダークモード（システム設定追従）/ 📅 カレンダー（気分インジケータ）/ 📊 統計（連続記録・気分トレンド）/ 🔍 全文検索 / 🎲 ランダム表示 / 💾 JSON エクスポート・インポート
- Google Play: <https://play.google.com/store/apps/details?id=com.diary.daily>

技術スタックは Flutter 3.x / Dart、状態管理 Provider、ローカル保存 Hive（NoSQL）、ローカライズ Flutter intl です。

## Fable に頼んだこと

「全部直して」ではなく **「品質的に気になるところを洗い出して直して」** くらいの粒度。具体的には `flutter analyze` の警告解消、テスト整備、CI 追加、依存ライブラリの更新です。analyzer は 47 件の警告が出ている状態でした。

## 1. async gap をまたいだ BuildContext の利用

`await` の後で `BuildContext`（`Navigator` やローカライズ）を触る箇所が複数あり、await 中に画面が破棄されると例外になる潜在バグでした。Fable は `await` の直後に `if (!mounted) return;` を挿入しています。

```diff
   final db = DatabaseService();
   final existingEntry = await db.getEntryByDate(_selectedDate);
+  if (!mounted) return;

   final result = await Navigator.push(
     context,
```

通常操作ではめったに踏まないので、自分でテストしていても気づきにくいタイプです。

## 2. await をまたぐ l10n 参照の巻き上げ

インポート処理 `_importData` では、`AppLocalizations.of(context)`（多言語文言）を await の後の複数箇所でその都度引いていました。これを最初の await の前に 1 回だけ引く形へ巻き上げ、確認ダイアログを出す前に `mounted` ガードを追加しています。

```diff
       _isImporting = true;
     });
+
+    final l10n = AppLocalizations.of(context)!;

     try {
       final result = await FilePicker...
       ...
       if (file.bytes == null) {
-        final l10n = AppLocalizations.of(context)!;   // await 後に都度参照
         throw Exception(l10n.fileReadError);
       }
       ...
-      final l10n = AppLocalizations.of(context)!;
+      if (!mounted) return;                            // ダイアログ表示前にガード
```

## 3. テストが何もテストしていなかった

テンプレートのまま残っていた `widget_test.dart` が、存在しない `MyApp` を参照していてコンパイルすら通らない状態でした（＝実質テストゼロ）。これを削除し、モデルと Hive 永続化の **実テスト 12 本** に置き換えています。

```
removed test/widget_test.dart            （壊れたテンプレ）
added  test/diary_entry_test.dart   +49  （モデルのシリアライズ等）
added  test/database_service_test.dart +92（Hive ベースの保存・取得）
```

「テストがある」つもりで実は無い、という一番こわい状態だったので、ここが直ったのは大きいです。

## 4. analyzer 警告 47 件 → 0

代表例は `Color.withOpacity()` の非推奨（色精度対応で `withValues` 推奨に）。27 箇所を置換しています。

```diff
-  color: Colors.white.withOpacity(0.1),
+  color: Colors.white.withValues(alpha: 0.1),
```

ほかに非推奨の `ColorScheme.background / onBackground` の除去、`RadioListTile` の `groupValue / onChanged` 廃止に伴う `RadioGroup` ancestor API への移行などを実施。`dart fix --apply` で機械的に直せるものは一括、API 移行が要る箇所は手で直す、という分担でした。

## 5. 依存ライブラリのメジャー更新

主要依存を最新メジャーへ。破壊的 API 変更が 2 件ありました。

```diff
-  google_mobile_ads: ^4.0.0
+  google_mobile_ads: ^9.0.0
-  share_plus: ^7.2.1
-  file_picker: ^8.1.4
+  share_plus: ^12.0.2
+  file_picker: ^11.0.2
```

**share_plus**: `Share.shareXFiles(...)` が `SharePlus.instance.share(ShareParams(...))` に変更。

```diff
-  await Share.shareXFiles(
-    [XFile(file.path)],
-    subject: fileName,
+  await SharePlus.instance.share(
+    ShareParams(
+      files: [XFile(file.path)],
+      subject: fileName,
+    ),
   );
```

**file_picker**: インスタンス経由の `FilePicker.platform.pickFiles(...)` が静的メソッド `FilePicker.pickFiles(...)` に。

```diff
-  final result = await FilePicker.platform.pickFiles(
+  final result = await FilePicker.pickFiles(
```

バージョン固定の理由もメモしておくと、share_plus 13 系は file_picker 11 と衝突、file_picker 12 は beta のみ、という事情で **share_plus 12 / file_picker 11** に落ち着けています。

## 6. CI を追加

push / PR で `flutter analyze` とテストを回す `flutter-ci.yml` を新設。これで「壊れたテスト」状態に逆戻りしないようにしています。

## 個人開発者としての所感

率直に言うと、Fable が見つけたのは派手な新機能ではなく、**自分ひとりだと体系的にチェックしない種類のもの** ばかりでした。

- async gap の mounted → タイミング次第でしか落ちない
- 壊れたテスト → 「テストがある」つもりで実は無い
- 非推奨 API の山 → 動いてはいるが、いつか壊れる
- 依存の塩漬け → メジャー更新は破壊変更が怖くて後回しになりがち

機能を書くところまでは自分でもできますが、こういう「条件が揃ったときだけ出る」「テストの実効性」「非推奨 API と依存の棚卸し」あたりは、レビュー専任がいないと抜けがちです。そこを一度しっかり通してくれたのは、個人開発の品質チェックとして十分有用でした。

一方で限界もありました。広告表示やエクスポート／インポート共有のような **実機 UI 依存の挙動** は CI では確認できず、最後は自分で実機チェックが必要です。アプリの企画・設計・実機テスト・ストア申請・最終的なコードレビューは引き続き自分の仕事、という分担感です。

公開 3 日で止まってしまったのは残念ですが、止まる前にこの一回を通せたのは記録として残しておく価値があると思っています。

---

日記アプリ「Daily Diary」は Google Play で公開中です。よかったら触ってみてください。

📱 <https://play.google.com/store/apps/details?id=com.diary.daily>
