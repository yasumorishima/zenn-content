---
title: "OSSにPRを出す前に入れておきたい無料コード品質ツール4選"
emoji: "🔍"
type: "tech"
topics: ["oss", "github", "coderabbit", "sonarcloud", "claudecode"]
published: true
---

## はじめに

OSSにPRを出すとき、機械的に検出できる問題は事前につぶしておきたい。メンテナーのレビュー負担を減らせるし、CIで落ちてからやり直す手間も省ける。

この記事では、OSSリポジトリに無料で導入できるコード品質ツールを4つ紹介する。全てクラウドで動くので、PCへの負荷はない。

## ツール比較表

| ツール | 用途 | 料金（OSS） | 処理場所 | 導入難易度 |
|---|---|---|---|---|
| **CodeRabbit** | AIコードレビュー | 無料 | クラウド | 簡単 |
| **Dependabot** | 依存パッケージの脆弱性検出・自動更新 | 無料（GitHub標準） | クラウド | 簡単 |
| **CodeQL** | セキュリティ脆弱性の静的解析 | 無料（GitHub標準） | クラウド | 簡単 |
| **SonarCloud** | コード品質・バグ・脆弱性の総合解析 | 無料（OSS） | クラウド | 普通 |

## 1. CodeRabbit ― AIがPRをレビューしてくれる

### 何ができるか

PRを作成すると、AIが自動でコードレビューを行い、改善提案をコメントする。ロジックの問題点、命名の改善、パフォーマンスの懸念など、人間のレビュアーに近いフィードバックが得られる。

### 料金

OSSリポジトリは無料。プライベートリポジトリも月間の無料枠がある。

### 導入手順

1. [GitHub Marketplace の CodeRabbit ページ](https://github.com/marketplace/coderabbit)にアクセス
2. 「Install it for free」をクリック
3. 対象のリポジトリ（またはOrganization全体）を選択
4. 「Install」をクリック

次にPRを作成すると、CodeRabbitが自動でレビューコメントを付ける。

### 実際の体験

自分のPRでは、catchブロックでエラー時に処理が続行される（fail-open）パターンを指摘された。エラーハンドリングではfail-closed（状態不明なら安全側に倒す）を意識する必要がある、という学びになった。

### 便利な使い方

- PRのコメントで `@coderabbitai review` と書くと再レビューしてくれる
- `@coderabbitai resolve` で指摘を解決済みにできる
- `.coderabbit.yaml` をリポジトリルートに置くとレビューのカスタマイズが可能

## 2. Dependabot ― 依存パッケージを自動で最新・安全に保つ

### 何ができるか

依存パッケージに脆弱性が見つかると、自動で修正PRを作成する。定期的なバージョン更新PRの作成もできる。

### 料金

無料。GitHubの標準機能。

### 導入手順

1. GitHubリポジトリの「Settings」タブを開く
2. 左メニューの「Code security」をクリック
3. 「Dependabot alerts」をEnableにする
4. 「Dependabot security updates」をEnableにする
5. （任意）「Dependabot version updates」も有効にすると、定期的な更新PRも作成される

### 設定ファイルでカスタマイズ

`.github/dependabot.yml` をリポジトリに追加すると、更新頻度やパッケージマネージャーを指定できる。

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
```

## 3. CodeQL ― GitHubが提供するセキュリティ解析

### 何ができるか

コード内のセキュリティ脆弱性（SQLインジェクション、XSS、パストラバーサルなど）を静的解析で検出する。GitHubが開発・保守しているツールで、GitHub Actionsで自動実行される。

### 料金

パブリックリポジトリは無料。GitHubの標準機能。

### 導入手順

1. GitHubリポジトリの「Settings」タブを開く
2. 左メニューの「Code security」をクリック
3. 「Code scanning」セクションを見つける
4. CodeQLの「Set up」→「Default」を選択
5. 解析対象の言語を確認して「Enable CodeQL」をクリック

デフォルト設定では、PRの作成時とデフォルトブランチへのpush時に自動で解析が実行される。

### カスタム設定（Advanced）

より細かい制御が必要な場合は、GitHub Actionsワークフローとして設定できる。

```yaml
# .github/workflows/codeql.yml
name: "CodeQL"
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: javascript-typescript
      - uses: github/codeql-action/analyze@v3
```

## 4. SonarCloud ― コード品質を総合的にチェック

### 何ができるか

バグ、脆弱性、コードの匂い（Code Smell）、テストカバレッジ、コードの重複などを分析する。PRにステータスチェックとして結果を表示し、品質ゲートの合否も判定する。

### 料金

OSSリポジトリは無料。プライベートリポジトリは有料。

### 導入手順

1. [sonarcloud.io](https://sonarcloud.io) にアクセスし、GitHubアカウントでログイン
2. 「Import an organization from GitHub」をクリック
3. 対象のOrganization（またはユーザー）を選択
4. 解析対象のリポジトリを選択
5. 「Set Up」をクリック
6. 解析方法で「With GitHub Actions」を選択
7. SonarCloudが表示するトークン（`SONAR_TOKEN`）をコピー
8. GitHubリポジトリの Settings → Secrets and variables → Actions で `SONAR_TOKEN` を追加
9. ワークフローファイルを作成:

```yaml
# .github/workflows/sonarcloud.yml
name: SonarCloud
on:
  push:
    branches: [main]
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  sonarcloud:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: SonarCloud Scan
        uses: SonarSource/sonarcloud-github-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

10. `sonar-project.properties` をリポジトリルートに追加:

```properties
sonar.organization=あなたのOrganization名
sonar.projectKey=あなたのプロジェクトキー
```

SonarCloudの管理画面に表示される値をそのまま使う。

## Claude Codeに頼む時のプロンプト集

これらのツールの導入は、Claude Code（`claude`コマンド）でも設定できる。

### CodeRabbit

```
このリポジトリにCodeRabbitの設定ファイルを追加して。
レビュー言語は日本語、テストファイルはレビュー対象外にして。
```

CodeRabbitの設定ファイル（`.coderabbit.yaml`）の例:

```yaml
language: "ja"
reviews:
  path_filters:
    - "!**/*.test.ts"
    - "!**/*.spec.ts"
```

### Dependabot

```
このリポジトリにDependabotの設定ファイルを追加して。
npmパッケージを週次で更新チェック、GitHub Actionsも月次でチェックするようにして。
```

### CodeQL

```
このリポジトリにCodeQLのGitHub Actionsワークフローを追加して。
言語はTypeScriptで、PRとmainブランチへのpush時に実行するようにして。
```

### SonarCloud

```
このリポジトリにSonarCloudのGitHub Actionsワークフローを追加して。
sonar-project.propertiesも作成して。Organization名は〇〇、プロジェクトキーは△△で。
```

### 全部まとめて

```
このリポジトリにコード品質ツールを導入して。
Dependabot（npm週次）、CodeQL（TypeScript）、SonarCloudのワークフローを追加して。
CodeRabbitはGitHub Marketplaceからインストール済み。
```

## まとめ

| やること | 所要時間の目安 |
|---|---|
| CodeRabbit導入 | GitHub Marketplaceで数クリック |
| Dependabot有効化 | リポジトリSettingsで数クリック |
| CodeQL有効化 | リポジトリSettingsで数クリック |
| SonarCloud導入 | アカウント連携 + ワークフロー追加 |

4つ全て導入しても、PCへの負荷はない（全てクラウド実行）。OSSなら費用もかからない。

PRを出す前に機械的なチェックを通しておくと、メンテナーのレビュー負担が減り、マージまでの流れもスムーズになる。
