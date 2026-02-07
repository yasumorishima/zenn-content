---
title: "Kaggle S6E2参加記：GitHub連携 + W&B + GPU 3モデルアンサンブルのワークフロー"
emoji: "🏥"
type: "tech"
topics: ["kaggle", "python", "機械学習", "wandb", "lightgbm"]
published: true
---

## はじめに

Kaggle Playground Series S6E2「Predicting Heart Disease」に参加しました。

https://www.kaggle.com/competitions/playground-series-s6e2

この記事では、コンペの内容よりも**ワークフロー全体**に焦点を当てます。

- **GitHub → Kaggle CLI** でノートブックを管理・実行
- **Weights & Biases（W&B）** で実験管理
- **GPU** で LightGBM / XGBoost / CatBoost を高速化

Kaggleコンペに初めて本格参加する方や、GitHub連携・W&B・GPUの設定で迷っている方の参考になれば幸いです。

### コンペ概要

| 項目 | 内容 |
|---|---|
| タスク | 心臓病予測（二値分類: Presence / Absence） |
| 評価指標 | AUC-ROC |
| データ | Train 630,000行 / Test 270,000行 |
| 締切 | 2026-02-28 |

### 結果

| 指標 | 値 |
|---|---|
| CV AUC（5-fold） | 0.95528 |
| LB Score | 0.95337 |
| 順位 | 616位 |

ノートブックはこちら:

https://www.kaggle.com/code/yasunorim/s6e2-heart-disease-eda-ensemble-wandb

---

## 1. GitHub → Kaggle CLI ワークフロー

### なぜGitHub管理？

Kaggle Web UIでもノートブックは編集できますが、以下の理由からGitHubで管理しています。

- **バージョン管理**: gitで差分が追える
- **ローカルエディタ**: VS Code等で快適に編集
- **CLI一発デプロイ**: `kaggle kernels push` でKaggleに反映

### セットアップ

#### 1. Kaggle CLIインストール

```bash
pip install kaggle
```

#### 2. API認証設定

Kaggleの「Settings」→「API」→「Create New Token」で `kaggle.json` をダウンロードし、`~/.kaggle/kaggle.json` に配置します。

#### 3. kernel-metadata.json

ノートブックと同じディレクトリに `kernel-metadata.json` を作成します。

```json
{
  "id": "your-username/your-notebook-slug",
  "title": "Your Notebook Title",
  "code_file": "your-notebook.ipynb",
  "language": "python",
  "kernel_type": "notebook",
  "is_private": "false",
  "enable_gpu": "true",
  "enable_tpu": "false",
  "enable_internet": "true",
  "dataset_sources": [],
  "competition_sources": ["playground-series-s6e2"],
  "kernel_sources": [],
  "model_sources": []
}
```

**ポイント**:
- `id`: `ユーザー名/slug` 形式。slugとtitleを一致させる
- `is_private`: `"false"` でpublic公開（upvote狙いなら必須）
- `enable_gpu`: `"true"` でGPU有効化（無料枠あり）
- `enable_internet`: `"true"` でW&B等の外部サービスに接続可能
- `competition_sources`: コンペのslugを指定するとデータが自動マウントされる
- boolean値は**文字列**（`"true"` / `"false"`）で指定する

#### 4. push & 確認

```bash
# Kaggleにpush（Windowsの場合UTF-8設定が必要）
PYTHONUTF8=1 kaggle kernels push -p .

# 実行状況確認
kaggle kernels status ユーザー名/slug

# ログ取得
kaggle kernels output ユーザー名/slug -p ./output

# 提出
kaggle competitions submit -c playground-series-s6e2 \
  -f ./output/submission.csv \
  -m "v4: ensemble CV AUC 0.955"
```

### 運用フロー

```
ローカルでipynb編集
  ↓
kaggle kernels push
  ↓
Kaggle上で実行（GPU使用）
  ↓
kaggle kernels status で完了確認
  ↓
kaggle kernels output でsubmission.csv取得
  ↓
kaggle competitions submit で提出
  ↓
kaggle competitions submissions で LB スコア確認
```

ローカルではコードを書くだけで、**実行は全てKaggle上で行います**。データのダウンロードやローカルでの処理は不要です。

---

## 2. Weights & Biases（W&B）連携

### W&Bとは

[Weights & Biases](https://wandb.ai) は機械学習の実験管理プラットフォームです。

**できること**:
- ハイパーパラメータの自動記録
- 学習曲線・メトリクスのリアルタイム可視化
- 実験間の比較（テーブル・グラフ）
- モデルのアーティファクト管理
- チームでの共有

### Kaggleでの設定方法

#### Step 1: W&Bアカウント作成 & APIキー取得

1. https://wandb.ai でアカウント作成
2. https://wandb.ai/authorize でAPIキーをコピー

#### Step 2: Kaggle Secretsに登録

ここが最初迷いやすいポイントです。Kaggleの「API Tokens」ページではなく、**ノートブックのSecrets設定**に登録します。

1. Kaggleでノートブックを開く
2. 右サイドバーの **「Add-ons」→「Secrets」**
3. **Label**: `WANDB_API_KEY`、**Value**: コピーしたAPIキー

*Kaggle Web UIの「Add-ons > Secrets」からAPIキーを設定します*

#### Step 3: ノートブックでのログインコード

```python
USE_WANDB = False
try:
    from kaggle_secrets import UserSecretsClient
    secrets = UserSecretsClient()
    wandb_api_key = secrets.get_secret('WANDB_API_KEY')
    if wandb_api_key:
        wandb.login(key=wandb_api_key)
        USE_WANDB = True
        print('W&B logged in successfully!')
    else:
        print('W&B API key not set. Running without experiment tracking.')
except Exception as e:
    print(f'W&B not available ({e}). Running without experiment tracking.')
```

**フォールバック設計**にしておくと、Secretsを設定していなくても（W&Bなしで）ノートブックは正常に動作します。公開ノートブックでは、閲覧者がそのまま実行できるのでこの設計が重要です。

#### Step 4: 実験ログの記録

```python
def wandb_init(name, tags, config):
    if USE_WANDB:
        return wandb.init(
            project='kaggle-s6e2-heart-disease',
            name=name, tags=tags, config=config, reinit=True
        )
    return None

def wandb_log(data):
    if USE_WANDB:
        wandb.log(data)

def wandb_end():
    if USE_WANDB:
        wandb.finish()

# 使用例
wandb_init('lgb-baseline', ['lightgbm', 'gpu'], {'model': 'LightGBM', **lgb_params})
# ... 学習ループ内 ...
wandb_log({'fold': fold, 'fold_auc': fold_auc})
wandb_log({'cv_auc': lgb_auc})
wandb_end()
```

### 注意点

- `enable_internet: "true"` が必須（W&Bはインターネット接続が必要）
- Kaggle Secretsは**ノートブックごと**に設定が必要
- **`kaggle kernels push`（CLI push）するとSecretsの紐付けがリセットされる**。push後はWeb UIで再度Secret ONにしてからRunする必要がある
- Kaggle標準のwandbは古いため、**`!pip install -q --upgrade wandb`** をノートブック冒頭に追加しておく（新形式APIキー `wandb_v1_...` に対応するため）
- W&Bの無料プランで個人利用は十分

---

## 3. GPU設定（LightGBM / XGBoost / CatBoost）

Kaggleでは `"enable_gpu": "true"` で無料のGPUが使えます。各GBDTライブラリのGPU設定をまとめます。

### LightGBM

```python
lgb_params = {
    'objective': 'binary',
    'metric': 'auc',
    'device': 'gpu',  # これだけでOK
    'n_estimators': 1000,
    'learning_rate': 0.05,
    # ...
}
```

### XGBoost（2.0+の注意点）

```python
xgb_params = {
    'objective': 'binary:logistic',
    'eval_metric': 'auc',
    'tree_method': 'hist',   # 'gpu_hist'は廃止
    'device': 'cuda',        # GPU指定はこちら
    'n_estimators': 1000,
    'learning_rate': 0.05,
    # ...
}
```

:::message alert
**XGBoost 2.0以降で `gpu_hist` は廃止されました。** `tree_method: 'hist'` + `device: 'cuda'` を使ってください。`gpu_hist` を指定すると以下のエラーになります:
```
XGBoostError: Invalid Input: 'gpu_hist', valid values are: {'approx', 'auto', 'exact', 'hist'}
```
実際にv3で遭遇し、v4で修正しました。
:::

### CatBoost

```python
cat_params = {
    'iterations': 1000,
    'learning_rate': 0.05,
    'depth': 6,
    'eval_metric': 'AUC',
    'task_type': 'GPU',  # これだけでOK
    # ...
}
```

CatBoostではGPU上でAUC計算ができないため、以下の警告が出ますが動作に問題はありません:
```
Default metric period is 5 because AUC is/are not implemented for GPU
```

### GPU設定まとめ

| ライブラリ | GPU設定 | 注意点 |
|---|---|---|
| LightGBM | `device: 'gpu'` | シンプル |
| XGBoost | `tree_method: 'hist'` + `device: 'cuda'` | 2.0+で`gpu_hist`廃止 |
| CatBoost | `task_type: 'GPU'` | AUC警告は無視してOK |

---

## 4. 3モデルアンサンブル

### アプローチ

LightGBM、XGBoost、CatBoostの3モデルをそれぞれ5-fold Stratified CVで学習し、予測確率の**単純平均**でアンサンブルしました。

```python
ens_preds = (lgb_preds + xgb_preds + cat_preds) / 3
```

### 特徴量

元の13特徴量に加え、6つの交互作用特徴量を追加しました（計19特徴量）。

| 特徴量 | 意味 |
|---|---|
| Age × Max HR | 年齢と最大心拍数の交互作用 |
| Age × ST depression | 年齢とST低下の交互作用 |
| ST depression × Slope of ST | ST関連指標の組み合わせ |
| BP × Cholesterol | 血圧とコレステロールの交互作用 |
| Max HR / Age | 年齢補正済み最大心拍数 |
| Number of vessels fluro × Thallium | 血管数とタリウム検査の組み合わせ |

### 結果

| Model | CV AUC |
|---|---|
| **Ensemble (avg)** | **0.95528** |
| CatBoost | 0.95524 |
| LightGBM | 0.95515 |
| XGBoost | 0.95513 |

- 3モデルの性能差はわずか（0.95513〜0.95524）
- アンサンブルで +0.0001〜0.0002 程度の改善
- CV→LB の差は約0.002で、大きなオーバーフィットなし

---

## 5. トラブルシューティング

開発中に遭遇したエラーと解決策をまとめます。

### v1: カラム名不一致

```python
# NG: UCI Heart Failure Dataset風の名前を想定していた
train['HeartDisease']

# OK: 実際のカラム名はスペース入り
train['Heart Disease']
```

**教訓**: コンペごとにカラム名は異なる。`train.columns` で実際の名前を確認してからコードを書く。

### v3: XGBoost gpu_hist 廃止

```
XGBoostError: Invalid Input: 'gpu_hist'
```

XGBoost 2.0+では `tree_method: 'gpu_hist'` が削除されています。`'hist'` + `device: 'cuda'` に変更して解決。

### W&B 接続エラー

```
W&B not available (Connection error trying to communicate with service.)
```

Kaggle Secretsに `WANDB_API_KEY` を設定していない場合に発生します。フォールバック設計にしておけばノートブック自体は動作するため、デバッグ段階ではW&B接続を後回しにできます。

---

## まとめ

| 項目 | 内容 |
|---|---|
| ワークフロー | GitHub → `kaggle kernels push` → Kaggle実行 → CLI提出 |
| 実験管理 | W&B（Kaggle Secretsで認証、フォールバック設計） |
| GPU設定 | LGB/XGB/CatBoost それぞれ設定方法が異なる |
| XGBoost注意 | 2.0+で`gpu_hist`廃止 → `hist` + `device:'cuda'` |
| 結果 | CV 0.95528 → LB 0.95337（616位） |

初回提出としてはベースラインが出せたので、ここから改善を重ねていきます。

### 改善候補

- Optuna でハイパーパラメータチューニング（W&Bに自動記録）
- Multi-seed averaging で安定化
- Rank-based ensemble（単純平均→順位ベース）
- 元データ（UCI Heart Disease Dataset）を追加学習データとして利用
- Target encoding for low-cardinality features

---

**ノートブック**: https://www.kaggle.com/code/yasunorim/s6e2-heart-disease-eda-ensemble-wandb

参考になれば幸いです。
