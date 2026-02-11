---
title: "低スペックPCのFlutterビルドをGitHub Actionsに移行した話"
emoji: "🔨"
type: "tech"
topics: ["flutter", "android", "githubactions", "cicd", "個人開発"]
published: true
---

## 背景

Androidアプリ（Flutter製）を個人開発しています。

開発環境はCeleron N4500 / RAM 4GB のノートPC。Flutter + Android SDKをローカルに入れていましたが、ビルドのたびにファンが唸り、完了まで10分以上かかっていました。

しかもディスク容量も逼迫気味。`flutter build apk`を走らせると、一時ファイルがどんどん積み上がっていきます。

「毎回これは辛い」と思い、GitHub Actionsでクラウドビルドに切り替えました。

---

## 構成

```
ローカル: コード修正 → git push
GitHub Actions: flutter build apk --release → APKをArtifactに保存
ローカル: gh run download でAPK取得 → Google Play Consoleにアップロード
```

ローカルにAndroid SDKは不要です。

---

## 手順

### 1. keystoreをbase64エンコード

Google Playへの署名に使うkeystoreファイルをGitHub Secretsに登録します。

```python
import base64
with open('android/app/upload-keystore.jks', 'rb') as f:
    print(base64.b64encode(f.read()).decode('utf-8'))
```

出力された文字列をコピーしておきます。

### 2. GitHub Secretsに登録

リポジトリの `Settings → Secrets and variables → Actions` で以下の4つを登録します。

| Secret名 | 値 |
|---|---|
| `KEYSTORE_BASE64` | 上記のbase64文字列 |
| `STORE_PASSWORD` | keystoreのパスワード |
| `KEY_PASSWORD` | keyのパスワード |
| `KEY_ALIAS` | keyのエイリアス（例: `upload`） |

### 3. workflow.ymlを作成

`.github/workflows/android-build.yml` を作成します。

```yaml
name: Android Build

on:
  push:
    branches:
      - master
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Java 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Set up Flutter
        uses: subosito/flutter-action@v2
        with:
          channel: 'stable'
          cache: true

      - name: Install dependencies
        run: flutter pub get

      - name: Decode keystore
        run: |
          echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 --decode > android/app/upload-keystore.jks

      - name: Create key.properties
        run: |
          cat > android/key.properties << EOF
          storePassword=${{ secrets.STORE_PASSWORD }}
          keyPassword=${{ secrets.KEY_PASSWORD }}
          keyAlias=${{ secrets.KEY_ALIAS }}
          storeFile=upload-keystore.jks
          EOF

      - name: Build release APK
        run: flutter build apk --release

      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: release-apk
          path: build/app/outputs/flutter-apk/app-release.apk
          retention-days: 7
```

### 4. APKの取得

ビルド完了後、GitHub CLIでダウンロードできます。

```bash
# 最新のRunID確認
gh run list --repo your-username/your-repo --limit 3

# APKダウンロード
gh run download <RUN_ID> --repo your-username/your-repo --dir ./apk-output
```

またはGitHubのActionsページ → 該当ビルド → Artifactsセクションから直接ダウンロードも可能です。

---

## 注意点

**ブランチ名に注意**

`on: push: branches:` に指定するブランチ名はリポジトリの実際のデフォルトブランチ名に合わせます。`main` の場合もあれば `master` の場合もあります。

**keystoreはgitignoreに入れておく**

```
# .gitignore
*.jks
*.keystore
key.properties
```

keystoreは絶対にリポジトリにコミットしないようにします。GitHub Secretsを使うことで、クラウドビルドでも安全に署名できます。

**初回は `flutter pub get` キャッシュが効かない**

初回ビルドは依存パッケージのダウンロードも含むため15分程度かかります。2回目以降は `subosito/flutter-action` の `cache: true` が効いて速くなります。

---

## まとめ

| | ローカルビルド | GitHub Actions |
|---|---|---|
| ビルド時間 | 10分以上（低スペックPC） | 10〜15分（初回）、以降は速くなる |
| PC負荷 | ファン爆音、他作業できない | なし |
| ディスク消費 | build/フォルダが肥大化 | なし |
| Android SDK | 必要 | 不要 |

低スペックPCでFlutter開発をしている方や、ローカルビルド環境を持ちたくない方にはおすすめの構成です。

---

## リンク

- [subosito/flutter-action](https://github.com/subosito/flutter-action)
- [actions/upload-artifact](https://github.com/actions/upload-artifact)
