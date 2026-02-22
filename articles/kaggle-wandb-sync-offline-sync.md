---
title: "KaggleのW&B実験管理をオフラインでも動かす：kaggle-wandb-sync を作った"
emoji: "🧬"
type: "tech"
topics: [kaggle, python, githubactions, wandb, pypi]
published: true
---

## はじめに

KaggleのコンペティションNotebookは、提出対象にするためにインターネット接続を**無効**にする必要があります。

これが意味するのは、**W&B（Weights & Biases）にリアルタイムでメトリクスを送れない**ということです。`wandb.log()` を呼んでも何も起きない。

この問題を解決するCLIツールを作り、PyPIに公開しました。

```bash
pip install kaggle-wandb-sync
```

→ **PyPI**: https://pypi.org/project/kaggle-wandb-sync/
→ **GitHub**: https://github.com/yasumorishima/kaggle-wandb-sync

## 仕組み

W&Bには `WANDB_MODE=offline` というオプションがあります。これを使うと、メトリクスをW&Bクラウドに送らず、ローカルに保存します。

Kaggle Notebookの場合、「ローカル」＝Kaggleの出力ディレクトリに保存されます。

あとは：
1. `kaggle kernels output` でファイルをダウンロード
2. `wandb sync` でW&Bクラウドに送る

**kaggle-wandb-sync** はこの一連の流れをCLIで自動化します。

```
Notebookがオフライン実行 → 出力ダウンロード → wandb sync → W&Bクラウド
```

## Notebookの設定

Notebookに2行追加するだけです。

```python
import os
os.environ['WANDB_MODE'] = 'offline'   # import wandb の前に設定する
os.environ['WANDB_PROJECT'] = 'my-project'

import wandb
wandb.init(name="my-run")
# ... 学習コード ...
wandb.log({"loss": 0.1, "accuracy": 0.95})
wandb.finish()
```

> **注意**: `WANDB_MODE=offline` は `import wandb` より**前**に設定してください。

## 使い方

Notebookの実行が終わったら、1コマンドでsyncできます。

```bash
export WANDB_API_KEY=your_api_key

# push → 完了待ち → ダウンロード → sync を一括実行
kaggle-wandb-sync run my-competition/
```

ステップごとに実行する場合：

```bash
kaggle-wandb-sync push   my-competition/           # Notebookをpush
kaggle-wandb-sync poll   username/my-competition   # COMPLETEになるまで待つ
kaggle-wandb-sync output username/my-competition   # 出力をダウンロード
kaggle-wandb-sync sync   ./kaggle_output           # W&Bにsync
```

Notebook実行後すぐにsyncしたい場合（pushはスキップ）：

```bash
kaggle-wandb-sync run my-competition/ --skip-push
```

## スコアの記録

Kaggleのリーダーボードでスコアが出たら、W&B runに後から記録できます。

```bash
kaggle-wandb-sync score https://wandb.ai/me/my-project/runs/abc123 \
  --tm-score 0.17 \
  --rank 779
```

v0.1.4から、`score` コマンド実行時に `submitted: True` も自動でセットされます（スコアが記録できた＝Submit済みのため）。

その他のメトリクスは `--metric` で追加できます：

```bash
kaggle-wandb-sync score https://wandb.ai/.../runs/abc123 \
  --metric auc=0.95 \
  --metric logloss=0.32 \
  --rank 42
```

## GitHub Actionsで自動化する

ローカルで実行するだけでなく、GitHub Actionsで自動化することもできます。

```yaml
name: Kaggle W&B Sync

on:
  workflow_dispatch:
    inputs:
      notebook_dir:
        description: "Notebookのディレクトリ名"
        required: true

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install
        run: pip install kaggle-wandb-sync

      - name: Kaggle認証設定
        run: |
          mkdir -p ~/.kaggle
          echo '${{ secrets.KAGGLE_API_TOKEN }}' > ~/.kaggle/kaggle.json
          chmod 600 ~/.kaggle/kaggle.json

      - name: パイプライン実行
        env:
          WANDB_API_KEY: ${{ secrets.WANDB_API_KEY }}
        run: kaggle-wandb-sync run ${{ inputs.notebook_dir }}
```

Notebookが完了したら、GitHub ActionsのUIからワークフローを手動実行するだけです。

**必要なSecrets**: `KAGGLE_API_TOKEN`（`~/.kaggle/kaggle.json`の中身）と `WANDB_API_KEY`

## 実例：Stanford RNA 3D Folding 2

[Stanford RNA 3D Folding 2](https://www.kaggle.com/competitions/stanford-rna-3d-folding-2) でこのツールを使って実験管理しました。

訓練データの3D座標をテンプレートとして使うアプローチ（テンプレートマッチング）を試し、スコアをW&Bに記録：

```bash
kaggle-wandb-sync score https://wandb.ai/.../runs/hahu4ygj \
  --tm-score 0.17 \
  --rank 779
```

baseline・improved・template matchingの3バージョンをW&Bで比較管理できています。

→ Notebook: https://www.kaggle.com/code/yasunorim/template-matching-w-b-via-kaggle-wandb-sync

## コマンド一覧

| コマンド | 内容 |
|---|---|
| `run` | 全工程一括（push → poll → output → sync） |
| `push` | NotebookをKaggleにpush |
| `poll` | Notebookの完了を待つ |
| `output` | 出力ファイルをダウンロード |
| `sync` | オフラインrunをW&Bにsync |
| `score` | 提出スコアをW&B runに記録 |

## まとめ

```bash
pip install kaggle-wandb-sync
kaggle-wandb-sync run my-competition/
```

Kaggleのインターネット制限があっても、W&Bで実験管理できるようになります。

スコアの記録も `score` コマンド1つで完結するので、複数バージョンの比較が楽になりました。
