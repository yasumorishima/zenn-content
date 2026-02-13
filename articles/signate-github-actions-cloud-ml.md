---
title: "SIGNATE×GitHub Actionsでローカル環境なしのクラウドML実践ガイド"
emoji: "☁️"
type: "tech"
topics: ["signate", "githubactions", "machinelearning", "python", "lightgbm"]
published: true
---

## はじめに

SIGNATEのコンペティションに参加したいけど、ローカルPCのスペックが心もとない——そんな方に向けて、**GitHub Actionsだけで完結する**機械学習パイプラインの構築方法を紹介します。

データのダウンロードから前処理、モデル学習、SIGNATEへの提出まで、すべてクラウドで行います。

```
SIGNATE CLI (GitHub Actions)
    ↓ データダウンロード
前処理 + LightGBM学習
    ↓ 推論
提出ファイル生成
    ↓ signate submit
SIGNATEに自動提出
```

**メリット:**
- ローカルPCのスペックに依存しない（RAM 4GBでもOK）
- `git push` → 自動で学習＆提出できる
- 実行環境がubuntu-latestで統一される

---

## 前提条件

- GitHubアカウント
- SIGNATEアカウント（メール＆パスワード認証）
- Python環境（CLIインストール用、ローカルで一度だけ使用）

---

## Step 1: SIGNATE CLIをインストール

```bash
pip install signate
```

---

## Step 2: APIトークンを取得

以下のコマンドで `~/.signate/signate.json` が自動生成されます。

```bash
signate token --email=your-email@example.com --password=your-password
```

成功すると以下のメッセージが表示されます：

```
The API Token has been downloaded successfully.
```

:::message
SIGNATEのWebサイト（アカウント設定）からJSONファイルを直接ダウンロードすることもできます。
:::

---

## Step 3: コンペのtask_keyとfile_keyを確認

SIGNATE CLIでコンペのファイル一覧を確認します。

```bash
signate file-list --task_key <task_key>
```

`task_key` はコンペURLのパラメータから取得できます：

```
https://user.competition.signate.jp/ja/competition/detail/?...&task=ここがtask_key
```

実行結果の例：

```
public_key                        file_name          file_size
--------------------------------  -----------------  -----------
5f0e1ebb35af4963b1720557d31daf35  train.csv          482.43 KB
72f23ebe8f004fa0b35323c4f744e8c5  test.csv           476.92 KB
ad3502af26b94d5aad4785351f36aa7b  sample_submit.csv  84.86 KB
```

`public_key` がダウンロード時に使う `file_key` になります。

---

## Step 4: GitHubリポジトリを作成

```bash
mkdir signate-comp && cd signate-comp
git init
```

`.gitignore` を作成（データファイルはgit管理しない）：

```
data/
__pycache__/
*.pyc
.signate/
```

---

## Step 5: SIGNATEトークンをGitHub Secretsに登録

トークンをBase64エンコードしてSecretに保存します。GitHub Secretsはトークン内の文字列をマスクすることがあるため、Base64エンコードが必要です。

```bash
# Base64エンコード
python -c "
import base64
with open('$HOME/.signate/signate.json', 'r') as f:
    token = f.read().strip()
print(base64.b64encode(token.encode()).decode())
"
```

出力されたBase64文字列をGitHub Secretsに登録します：

```bash
# リポジトリ作成（private推奨）
gh repo create your-username/signate-comp --private --source=. --push

# Secret登録
gh secret set SIGNATE_TOKEN_B64 --body "<Base64文字列>"
```

:::message alert
トークンをそのまま（Base64なしで）Secretに入れると、GitHub Actionsのログマスキングによりトークンが壊れて認証エラーになります。必ずBase64エンコードしてください。
:::

---

## Step 6: GitHub Actionsワークフローを作成

`.github/workflows/submit.yml` を作成します：

```yaml
name: Train & Submit

on:
  workflow_dispatch:
    inputs:
      memo:
        description: "Submission memo"
        required: false
        default: "GitHub Actions submission"

jobs:
  submit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: pip install pandas numpy scikit-learn lightgbm signate

      - name: Setup SIGNATE token
        run: |
          mkdir -p ~/.signate
          echo '${{ secrets.SIGNATE_TOKEN_B64 }}' | base64 -d > ~/.signate/signate.json

      - name: Download data
        run: |
          mkdir -p data
          cd data
          signate download --task_key <your_task_key> --file_key <train_file_key>
          signate download --task_key <your_task_key> --file_key <test_file_key>

      - name: Train and predict
        run: python train.py

      - name: Submit to SIGNATE
        run: |
          signate submit data/submission.csv \
            --task_key <your_task_key> \
            --memo "${{ github.event.inputs.memo }}"

      - name: Upload submission as artifact
        uses: actions/upload-artifact@v4
        with:
          name: submission
          path: data/submission.csv
          retention-days: 90
```

**ポイント:**
- `workflow_dispatch` で手動トリガー（メモ付き）
- Base64デコードでトークン復元
- `signate download` → `train.py` → `signate submit` の一気通貫
- Artifactにも提出ファイルを保存（後から確認可能）

---

## Step 7: 学習スクリプトを作成

`train.py` の例（LightGBM 5-fold CV）：

```python
import pandas as pd
import numpy as np
from sklearn.model_selection import StratifiedKFold
from sklearn.metrics import roc_auc_score
import lightgbm as lgb

def main():
    train = pd.read_csv("data/train.csv")
    test = pd.read_csv("data/test.csv")

    # --- 前処理（コンペに応じて実装） ---
    # train = preprocess(train)
    # test = preprocess(test)

    target = "ProdTaken"  # ターゲット列名
    features = [c for c in train.columns if c not in ["id", target]]
    X, y = train[features], train[target]
    X_test = test[features]

    # --- LightGBM 5-fold CV ---
    params = {
        "objective": "binary",
        "metric": "auc",
        "verbosity": -1,
        "n_estimators": 1000,
        "learning_rate": 0.05,
        "random_state": 42,
    }

    skf = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
    oof_preds = np.zeros(len(X))
    test_preds = np.zeros(len(X_test))

    for fold, (tr_idx, val_idx) in enumerate(skf.split(X, y)):
        model = lgb.LGBMClassifier(**params)
        model.fit(
            X.iloc[tr_idx], y.iloc[tr_idx],
            eval_set=[(X.iloc[val_idx], y.iloc[val_idx])],
            callbacks=[lgb.early_stopping(50), lgb.log_evaluation(0)],
        )
        oof_preds[val_idx] = model.predict_proba(X.iloc[val_idx])[:, 1]
        test_preds += model.predict_proba(X_test)[:, 1] / 5
        print(f"Fold {fold+1}: AUC = {roc_auc_score(y.iloc[val_idx], oof_preds[val_idx]):.5f}")

    print(f"Overall OOF AUC: {roc_auc_score(y, oof_preds):.5f}")

    # --- 提出ファイル作成（ヘッダなし） ---
    submission = pd.DataFrame({"id": test["id"], "pred": test_preds})
    submission.to_csv("data/submission.csv", index=False, header=False)
    print(f"Saved: data/submission.csv ({len(submission)} rows)")

if __name__ == "__main__":
    main()
```

---

## Step 8: 実行する

GitHub Actions画面から手動実行するか、CLIで実行します：

```bash
gh workflow run submit.yml -f memo="Baseline LightGBM v1"
```

実行状況の確認：

```bash
gh run list --limit 1
```

ログの確認：

```bash
gh run view <run_id> --log
```

---

## SIGNATE CLI コマンド早見表

| コマンド | 説明 |
|---|---|
| `signate token --email=xxx --password=xxx` | APIトークン取得 |
| `signate file-list --task_key <key>` | ファイル一覧表示 |
| `signate download --task_key <key> --file_key <key>` | データダウンロード |
| `signate submit <file> --task_key <key> --memo <memo>` | 提出 |

---

## ハマりポイントと対策

### 1. GitHub Secretsのトークンマスキング

**現象:** `Invalid header value` エラーで認証失敗

**原因:** GitHub SecretsがJWTトークン内の文字列をマスク（`***`に置換）してしまう

**対策:** Base64エンコードしてSecretに保存し、ワークフロー内でデコードする

### 2. `--path` オプションの挙動

**現象:** `[Errno 21] Is a directory` エラー

**原因:** `signate download --path data/` のようにディレクトリを指定すると失敗する

**対策:** `cd data && signate download ...` のように、カレントディレクトリを変更してからダウンロードする

### 3. GitHub Actions無料枠

GitHub Freeプランでは月2,000分の実行時間があります。小〜中規模のデータセットなら十分です。1回の学習＆提出に約40秒程度なので、余裕を持って運用できます。

---

## まとめ

SIGNATE CLI + GitHub Actionsで、ローカル環境に依存しない機械学習パイプラインを構築しました。

```
git push → GitHub Actions → データDL → 学習 → 提出
```

この仕組みがあれば、低スペックPCでも、スマホからでも、コードをpushするだけでコンペに参加できます。

---

## 参考リンク

- [SIGNATE CLI（PyPI）](https://pypi.org/project/signate/)
- [SIGNATE CLI でデータセットをダウンロードする（Qiita）](https://qiita.com/john-rocky/items/dd2b7f4cd8e655a5376b)
- [SIGNATE CLIを使うための準備（Zenn）](https://zenn.dev/kawacdev/articles/ea5fd29441012e)
- [GitHub Actions ドキュメント](https://docs.github.com/ja/actions)
