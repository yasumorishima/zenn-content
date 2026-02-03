---
title: "Flutterアプリのアイコンが透明になる問題の原因と解決法（Android adaptive icon）"
emoji: "📱"
type: "tech"
topics: ["flutter", "android", "adaptiveicon", "googleplay"]
published: true
---

## この記事の対象読者

Flutterで作ったAndroidアプリのアイコンが、ホーム画面で透明・グレー・白背景になってしまい困っている人。

## 症状

Google Play Storeではアイコンが正常に表示されるのに、**ホーム画面のアイコンが透明（またはグレー）になる**。

具体的には：
- Play Storeのアプリページ → 紫背景に本のアイコン（正常）
- ホーム画面 → 白い円の中に小さい紫の四角（異常）

キャッシュクリアやアプリの再インストールをしても直らない。

## 原因

Android 8.0（API 26）以降の**adaptive icon**の仕組みに問題があった。

### adaptive iconの構造

```
android/app/src/main/res/
├── mipmap-anydpi-v26/
│   ├── ic_launcher.xml        ← ここがadaptive iconの定義
│   └── ic_launcher_round.xml
├── mipmap-xxxhdpi/
│   └── ic_launcher.png        ← 従来のPNGアイコン（フォールバック用）
└── drawable-xxxhdpi/
    └── ic_launcher_foreground.png  ← foregroundレイヤー
```

adaptive icon XMLは`foreground`と`background`の2レイヤーを合成してアイコンを生成する：

```xml
<!-- mipmap-anydpi-v26/ic_launcher.xml -->
<adaptive-icon xmlns:android="http://schemas.android.com/apk/res/android">
    <background android:drawable="@drawable/ic_launcher_background"/>
    <foreground android:drawable="@drawable/ic_launcher_foreground"/>
</adaptive-icon>
```

### 根本原因：foregroundファイルの不在

XMLが`@drawable/ic_launcher_foreground`を参照しているのに、**`drawable-*`フォルダに`ic_launcher_foreground.png`が存在しなかった**。

Flutterプロジェクトのデフォルト構成では、`flutter create`時にadaptive icon XMLが生成されるが、foreground画像は自動生成されない。結果として：

1. XMLが存在しない画像を参照
2. Androidがforegroundのレンダリングに失敗
3. アイコンが透明/グレーにフォールバック

**Play Storeでは正常に見える理由**：Play Storeはアップロード時の512x512アイコン画像を表示しており、端末のadaptive icon仕組みとは無関係。

## 試したが効果がなかった対処法

| バージョン | 対処 | 結果 |
|---|---|---|
| v1.0.9 | `ic_launcher_round.png`を全密度フォルダに追加 | 変わらず |
| v1.0.10 | background XMLを`@color/`から`@drawable/`(gradient shape)に変更 | 変わらず |
| v1.0.11 | `mipmap-anydpi-v26`フォルダごと削除（PNGフォールバック） | アイコンは出るが白背景+小さい四角 |

v1.0.11でPNGフォールバックにしたことでアイコン自体は表示されるようになったが、Androidのランチャーが丸いマスクを適用するため、四角いPNGアイコンが白い円の中に小さく表示される見栄えの悪い結果になった。

## 解決法

### 1. foreground画像を用意する

`drawable-*`の各密度フォルダに`ic_launcher_foreground.png`を配置する。

| 密度 | サイズ |
|---|---|
| mdpi | 108x108 |
| hdpi | 162x162 |
| xhdpi | 216x216 |
| xxhdpi | 324x324 |
| xxxhdpi | 432x432 |

foreground画像は**透明背景**に**アイコン本体のみ**を配置する。アイコンはキャンバスの中央に大きめに配置すること。adaptive iconのsafe zoneは全体の約66%（内側72dp / 全体108dp）なので、重要な部分はキャンバスの中央66%に収める。

### 2. background画像/XMLを用意する

背景色やグラデーションの場合はXMLで定義できる：

```xml
<!-- drawable/ic_launcher_background.xml -->
<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android"
    android:shape="rectangle">
    <gradient
        android:startColor="#667eea"
        android:endColor="#764ba2"
        android:angle="315"
        android:type="linear"/>
</shape>
```

### 3. adaptive icon XMLを配置する

```xml
<!-- mipmap-anydpi-v26/ic_launcher.xml -->
<?xml version="1.0" encoding="utf-8"?>
<adaptive-icon xmlns:android="http://schemas.android.com/apk/res/android">
    <background android:drawable="@drawable/ic_launcher_background"/>
    <foreground android:drawable="@drawable/ic_launcher_foreground"/>
</adaptive-icon>
```

`ic_launcher_round.xml`も同じ内容で作成する。

### 最終的なファイル構成

```
android/app/src/main/res/
├── mipmap-anydpi-v26/
│   ├── ic_launcher.xml          ← adaptive icon定義
│   └── ic_launcher_round.xml    ← 同上
├── drawable/
│   └── ic_launcher_background.xml  ← 紫グラデーション背景
├── drawable-mdpi/
│   └── ic_launcher_foreground.png  ← 108x108
├── drawable-hdpi/
│   └── ic_launcher_foreground.png  ← 162x162
├── drawable-xhdpi/
│   └── ic_launcher_foreground.png  ← 216x216
├── drawable-xxhdpi/
│   └── ic_launcher_foreground.png  ← 324x324
├── drawable-xxxhdpi/
│   └── ic_launcher_foreground.png  ← 432x432
├── mipmap-mdpi/
│   └── ic_launcher.png          ← フォールバック用PNG
│   ...
```

## チェックリスト

adaptive iconのトラブルに遭遇したら、以下を確認する：

- [ ] `mipmap-anydpi-v26/ic_launcher.xml`が存在するか
- [ ] XMLの`@drawable/ic_launcher_foreground`が参照するPNGが**全密度の`drawable-*`フォルダ**に存在するか
- [ ] foreground画像のアイコン部分がキャンバスの中央66%に収まっているか
- [ ] foreground画像のアイコンが十分に大きいか（小さいと丸いマスク内で縮小表示される）

## まとめ

Flutterアプリのアイコンが透明になる原因は、**adaptive icon XMLが参照するforeground画像が存在しない**ことだった。Play Storeでは正常に見えるため発見が遅れやすい。`mipmap-anydpi-v26`のXMLと`drawable-*`のforeground PNGの対応関係を確認することが重要。
