---
title: "git pushするだけでKaggle Notebookをデプロイ：kaggle-notebook-deploy を作った"
emoji: "🏀"
type: "tech"
topics: [kaggle, python, githubactions, pypi, machinelearning]
published: true
---

## はじめに

KaggleのNotebookをブラウザ上のエディタで直接編集していませんか？

ブラウザエディタでは：
- Gitによるバージョン管理ができない
- 差分の確認がしづらい
- VSCode等の使い慣れたエディタが使えない

この不便を解消するために、**`git push`するだけでKaggle Notebookを自動デプロイするCLIツール**を作り、PyPIに公開しました。

```bash
pip install kaggle-notebook-deploy
```

→ **PyPI**: https://pypi.org/project/kaggle-notebook-deploy/
→ **GitHub**: https://github.com/yasumorishima/kaggle-notebook-deploy

この記事では、ツールの使い方と、実際に [March Machine Learning Mania 2026](https://www.kaggle.com/competitions/march-machine-learning-mania-2026)（NCAAトーナメント予測コンペ）で運用した実例を紹介します。

## ワークフロー

```
ノートブック編集 → git push → GitHub Actions → Kaggleにアップロード → ブラウザでSubmit
```

## セットアップ（5分）

### 1. インストール

```bash
pip install kaggle-notebook-deploy
```

### 2. リポジトリのセットアップ

```bash
mkdir my-kaggle && cd my-kaggle && git init
kaggle-notebook-deploy init-repo
```

以下のファイルが自動生成されます：

```
my-kaggle/
├── .github/workflows/
│   └── kaggle-push.yml     # Kaggle push ワークフロー
├── scripts/
│   └── setup-credentials.sh
└── .gitignore
```

### 3. GitHub Secretsの設定

```bash
gh secret set KAGGLE_USERNAME
gh secret set KAGGLE_KEY
```

### 4. コンペへの参加

```bash
# GPU有効・公開Notebookとして作成
kaggle-notebook-deploy init march-machine-learning-mania-2026 --gpu --public
```

```
march-machine-learning-mania-2026/
├── kernel-metadata.json   # Kaggle設定（自動生成）
└── march-machine-learning-mania-2026-baseline.ipynb
```

### 5. デプロイ

```bash
# ローカルから直接push
kaggle-notebook-deploy push march-machine-learning-mania-2026

# または GitHub Actions 経由
git add . && git commit -m "update notebook" && git push
gh workflow run kaggle-push.yml -f notebook_dir=march-machine-learning-mania-2026
```

## 実例：March Machine Learning Mania 2026

[March Machine Learning Mania 2026](https://www.kaggle.com/competitions/march-machine-learning-mania-2026) はNCAAバスケットボール（男女）のトーナメント結果を予測するコンペです。

実際にこのツールでNotebookを作成・デプロイしました：
→ https://www.kaggle.com/code/yasunorim/march-machine-learning-mania-2026-baseline

ローカルでNotebookを編集し、`kaggle-notebook-deploy push` の1コマンドでKaggleに反映されます。

## ハマりどころ

実際に運用する中でいくつかの罠があったので記録しておきます。

### 1. データパスの罠

`kernel-metadata.json`の`competition_sources`でデータをマウントすると、パスは：

```
/kaggle/input/competitions/<slug>/
```

**`/kaggle/input/<slug>/`ではありません。** `competitions/`サブディレクトリが入ります。

パスをハードコードすると`FileNotFoundError`になります。パス自動検出で対応しましょう：

```python
from pathlib import Path

INPUT_ROOT = Path('/kaggle/input')
DATA_DIR = None
for p in INPUT_ROOT.rglob('your-expected-file.csv'):
    DATA_DIR = p.parent
    break

if DATA_DIR is None:
    for item in sorted(INPUT_ROOT.iterdir()):
        print(f'  {item.name}/')
        for sub in sorted(item.iterdir())[:5]:
            print(f'    {sub.name}')
    raise FileNotFoundError('Data directory not found. See structure above.')
```

### 2. NaN列のfillna罠

特徴量の一部が全てNaN（例：女性データにはMassey Ordinalsがない）の場合、`fillna(median)`は効きません。**medianそのものがNaN**になるからです。

```python
# NG: median が NaN の場合は効かない
X = df[feat_cols].fillna(df[feat_cols].median()).values

# OK: 0でフォールバック
X = df[feat_cols].fillna(df[feat_cols].median()).fillna(0).values
```

### 3. Windowsでのエンコーディング問題

Windowsのcp932環境では`kaggle-notebook-deploy push`が文字化けします。`PYTHONUTF8=1`をつけて実行してください。

```bash
PYTHONUTF8=1 kaggle-notebook-deploy push march-machine-learning-mania-2026
```

## コマンドリファレンス

| コマンド | 説明 |
|---|---|
| `init-repo` | GitHub Actionsワークフローを生成 |
| `init <slug>` | コンペディレクトリを生成 |
| `validate <dir>` | kernel-metadata.json を検証 |
| `push <dir>` | Kaggleにデプロイ |

主なオプション（`init`）：

| オプション | 説明 |
|---|---|
| `--gpu` | GPU有効 |
| `--public` | 公開Notebook |
| `--title` | タイトル指定 |
| `--internet` | インターネット有効（コードコンペでは非推奨） |

## なぜ完全自動化できないのか

`git push`だけで提出まで完結できれば理想ですが、Kaggle側の制限があります：

1. **API制限**: `CreateCodeSubmission` APIは公開トークンでは403
2. **Secretsリセット**: `kaggle kernels push`のたびにKaggle Secretsの紐付けが外れる
3. **ルール同意**: コンペ参加にはブラウザでの同意が必要（初回のみ）

そのため、最終的な「Submit to Competition」ボタンのクリックだけは手動操作が必要です。

コードの管理・バージョン管理・デプロイをGitHub側に集約できるだけでも、開発体験は大きく改善されます。

## まとめ

```bash
pip install kaggle-notebook-deploy           # インストール
kaggle-notebook-deploy init-repo             # リポジトリセットアップ
kaggle-notebook-deploy init titanic          # コンペ参加
# ... ノートブックを編集 ...
kaggle-notebook-deploy push titanic          # デプロイ
```

ブラウザエディタとさよならして、ローカルのGitワークフローでKaggleコンペに取り組みましょう。
