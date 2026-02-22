---
title: "SIGNATEコンペのスコアをW&Bに記録する：signate-wandb-sync を作った"
emoji: "📊"
type: "tech"
topics: [signate, python, wandb, pypi, machinelearning]
published: true
---

## はじめに

以前、[signate-deploy](https://zenn.dev/shogaku/articles/signate-deploy-cli-introduction) というCLIツールを作りました。SIGNATEコンペの提出をGitHub Actions経由で自動化するものです。

今回はその続きとして、**SIGNATEの提出スコアをWeights & Biases（W&B）に記録するツール**を作りました。

```bash
pip install signate-wandb-sync
```

> **PyPI**: https://pypi.org/project/signate-wandb-sync/
> **GitHub**: https://github.com/yasumorishima/signate-wandb-sync

## signate-wandb-sync とは

SIGNATEの提出スコアを、W&B runのsummaryに記録するCLIツールです。

実験管理をW&Bで行っている場合、KaggleではスコアをW&Bに紐付けるのが難しいことがあります（インターネット制限があるコンペ等）。

SIGNATEはAPIによる提出がGitHub Actions上でできるため、**GitHub ActionsでW&B runを作成 → SIGNATEに提出 → ローカルでスコアをW&Bに記録** という流れが比較的シンプルに実現できます。

## インストール

```bash
pip install signate-wandb-sync
```

W&Bへの認証が必要です（事前に `wandb login` を実行するか、`WANDB_API_KEY` 環境変数を設定）。

## フル自動化パイプライン

`signate-deploy` と組み合わせたフロー全体はこうなります。

```
[GitHub Actions]
  1. SIGNATEからデータDL（signate-deploy）
  2. train.py 実行（W&B run作成・ログ記録）
  3. SIGNATEへ提出（signate-deploy）
  → ログにW&B run URLが表示される

[ローカル]
  4. SIGNATEでスコア確認
  5. signate-wandb-sync score で記録
```

### train.py へのW&B組み込み

`train.py` に数行追加するだけです。

```python
import wandb

run = wandb.init(
    project="my-signate-project",
    config={
        "model": "...",
        "params": "...",
    },
)

# 学習・推論 ...

wandb.log({"oof_score": oof_score})

print(f"W&B run URL: {run.url}")  # GitHub Actionsログに出力される
wandb.finish()
```

GitHub ActionsにはW&B API keyをSecretとして設定します。

```yaml
- name: Train and predict
  env:
    WANDB_API_KEY: ${{ secrets.WANDB_API_KEY }}
  run: python train.py
```

### scoreコマンドでスコアを記録

SIGNATEでスコアを確認したら、`score` コマンドで対応するW&B runに記録します。

```bash
# Windowsの場合
PYTHONUTF8=1 signate-wandb-sync score https://wandb.ai/your-entity/your-project/runs/abc123 \
    --score 0.85 --rank 3
```

W&B run URLはGitHub Actionsのログから取得できます（`print(run.url)` で出力したもの）。

実行すると、W&B runのsummaryに以下が追記されます。

```
Updated run: my-run-name (your-entity/your-project/abc123)
  submitted = True
  signate_score = 0.85
  signate_rank = 3
```

## scoreコマンドのオプション

| オプション | 説明 |
|---|---|
| `--score` | SIGNATEの提出スコア（float） |
| `--rank` | リーダーボード順位（int） |
| `--metric KEY=VALUE` | 追加メトリクス（複数指定可） |
| `--project entity/project` | ベアrun IDを使う場合に指定 |

```bash
# 追加メトリクスも記録できる
signate-wandb-sync score <W&B URL> \
    --score 0.85 --rank 3 \
    -m fbeta=0.85 -m recall=0.91
```

## run_idの指定形式

```bash
# 完全URL（推奨）
signate-wandb-sync score https://wandb.ai/entity/project/runs/abc123 --score 0.85

# パス形式
signate-wandb-sync score entity/project/abc123 --score 0.85

# ベアID（--project指定が必要）
signate-wandb-sync score abc123 --project entity/project --score 0.85
```

## まとめ

```bash
pip install signate-deploy signate-wandb-sync
```

- `signate-deploy`: GitHub Actions上でデータDL・学習・提出を自動化
- `signate-wandb-sync`: 提出スコアをW&Bに記録

この2つを組み合わせることで、SIGNATEコンペの実験管理をW&Bで行う環境が整います。

> **PyPI**: https://pypi.org/project/signate-wandb-sync/
> **GitHub**: https://github.com/yasumorishima/signate-wandb-sync
