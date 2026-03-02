---
title: "SIGNATEコンペの提出をCLIで自動化する：signate-deploy を作った"
emoji: "🚀"
type: "tech"
topics: [signate, python, githubactions, pypi, machinelearning]
published: true
---

## はじめに

以前、[SIGNATE×GitHub Actionsでローカル環境なしのクラウドML実践ガイド](https://zenn.dev/shogaku/articles/signate-github-actions-cloud-ml)という記事を書きました。

この記事ではGitHub Actionsのワークフローを手動で構築する方法を紹介しましたが、リポジトリの初期化やSIGNATE CLIの設定など、毎回やるセットアップが面倒でした。

そこで、**この一連の手順をCLIで自動化するツール**を作りました。

```bash
pip install signate-deploy
```

> **PyPI**: https://pypi.org/project/signate-deploy/
> **GitHub**: https://github.com/yasumorishima/signate-deploy

## signate-deploy とは

SIGNATEコンペ用のリポジトリ作成から提出までを、CLIコマンドで管理するツールです。

```
signate-deploy init-repo   ← リポジトリ初期化 + GitHub Actions生成
signate-deploy setup-token ← SIGNATE APIトークン設定
signate-deploy init        ← SIGNATE CLI認証
signate-deploy submit      ← GitHub Actions経由で提出
```

既存記事との違い：

| | 既存記事 | signate-deploy |
|---|---|---|
| ワークフロー | 手動でYAML作成 | `init-repo` で自動生成 |
| SIGNATE認証 | 手動で `signate.json` 設定 | `setup-token` で対話式設定 |
| 提出 | `gh workflow run` を手動実行 | `submit` コマンドで実行 |
| データDL | ワークフロー内で手動記述 | `download` コマンドで実行 |

## セットアップ手順

### 1. リポジトリ初期化

```bash
python -m signate_deploy init-repo my-competition
cd my-competition
```

これだけで以下が自動生成されます：
- `.github/workflows/signate-submit.yml`（提出用ワークフロー）
- `.github/workflows/signate-download.yml`（データDL用ワークフロー）
- `train.py`（テンプレート）
- `.gitignore`

### 2. SIGNATE APIトークン設定

```bash
python -m signate_deploy setup-token
```

対話式でSIGNATEのメールアドレスとAPIトークンを入力し、GitHub Secretsに保存します。

### 3. SIGNATE CLI認証

```bash
python -m signate_deploy init
```

SIGNATE CLIの初期認証を行います。

### 4. コンペ情報の確認

```bash
# コンペ一覧
python -m signate_deploy competition-list

# タスク一覧（competition_keyを指定）
python -m signate_deploy task-list --competition-key <competition_key>

# ファイル一覧（task_keyを指定）
python -m signate_deploy file-list --task-key <task_key>
```

### 5. データダウンロード

```bash
python -m signate_deploy download
```

GitHub Actions経由でデータをダウンロードします。

### 6. 提出

```bash
python -m signate_deploy submit
```

GitHub Actions経由で学習と提出を実行します。

## コマンド一覧

| コマンド | 内容 |
|---|---|
| `init-repo` | リポジトリ初期化 + GitHub Actions生成 |
| `init` | SIGNATE CLI認証 |
| `setup-token` | APIトークンをGitHub Secretsに設定 |
| `submit` | GitHub Actions経由で提出 |
| `download` | GitHub Actions経由でデータDL |
| `competition-list` | コンペティション一覧表示 |
| `task-list` | タスク一覧表示 |
| `file-list` | ファイル一覧表示 |

## 実例

実際にSIGNATEのコンペティションで使い、setup-token → download → submit の全フローがGitHub Actions上で完結することを確認しました。

```bash
python -m signate_deploy setup-token   # トークン設定
python -m signate_deploy download      # データダウンロード
# ... 学習コード作成 ...
python -m signate_deploy submit        # 提出
```

## ハマりポイント

### Windows PATH問題

Windowsでは `signate-deploy` コマンドがPATHに入らないことがあります。その場合は `python -m signate_deploy` を使ってください。

```bash
# PATHに入らない場合
python -m signate_deploy <command>
```

### competition_key と task_key は別物

SIGNATEのURLに含まれる `competition=` の値は `competition_key` です。データダウンロードに必要な `task_key` は別途取得する必要があります。

```bash
# 1. competition_key からタスク一覧を取得
python -m signate_deploy task-list --competition-key <competition_key>

# 2. task_key からファイル一覧を取得
python -m signate_deploy file-list --task-key <task_key>
```

### submit/download は GitHub Actions 経由

`submit` と `download` は `gh workflow run` 経由で実行されます。ローカルで直接SIGNATEに提出するわけではないので、事前にGitHubリポジトリとの連携が必要です。

### gh secret set はリポのディレクトリ内で実行する

`gh secret set` はカレントディレクトリがgitリポジトリでないと失敗します。必ずリポジトリのルートで実行してください。

```bash
cd my-competition   # gitリポのディレクトリ内
gh secret set SIGNATE_EMAIL --body your@email.com
```

## v0.1.7: トークン自動更新に対応

SIGNATEのAPIトークンには有効期限があります。v0.1.7からは `init-repo` が `scripts/refresh_signate_token.py` を自動生成し、GitHub Actions実行のたびにトークンを自動取得します。

### 設定方法（リポのディレクトリ内で実行）

```bash
gh secret set SIGNATE_EMAIL --body your@email.com
gh secret set SIGNATE_PASSWORD   # プロンプトで安全に入力
```

これだけで毎回のActions実行で自動的に新しいトークンが取得されます。`setup-token` を手動で再実行する必要はありません。

### v0.1.6以前からの移行

既存リポジトリは `init-repo --force` で最新ワークフローに更新できます。

```bash
python -m signate_deploy init-repo --force
git add .github/workflows/ scripts/
git commit -m "Update to token auto-refresh workflow"
git push
```

`SIGNATE_EMAIL` と `SIGNATE_PASSWORD` をGitHub Secretsに追加すれば完了です（`SIGNATE_TOKEN_B64` は不要になります）。

## まとめ

```bash
pip install signate-deploy
python -m signate_deploy init-repo my-competition
```

SIGNATEコンペの環境構築と提出をCLIで自動化できます。

手動セットアップの詳細は[こちらの記事](https://zenn.dev/shogaku/articles/signate-github-actions-cloud-ml)を参照してください。

> **PyPI**: https://pypi.org/project/signate-deploy/
> **GitHub**: https://github.com/yasumorishima/signate-deploy
