---
title: "Kaggleコンペをローカル環境なしで戦う：GitHub Actions + Kaggle APIクラウドワークフロー"
emoji: "☁️"
type: "tech"
topics: [kaggle,githubactions,python,automation]
published: true
---

## はじめに

KaggleのNotebookをブラウザ上のエディタで直接編集していませんか？

ブラウザエディタでは、Gitによるバージョン管理ができない、差分の確認がしづらい、使い慣れたエディタ（VSCode等）が使えない、といった不便があります。

この記事では、**Notebookのコード管理をGitHubに集約し、`git push`するだけでKaggleに自動デプロイ**するワークフローを紹介します。GitHubのエコシステム（Actions, Secrets, バージョン管理）をそのままKaggleコンペに活用できます。

実際にDeep Past Challenge（Akkadian→English翻訳コンペ）で運用した内容をベースにしています。

## 完成したワークフロー

```
ノートブック編集 → git push → GitHub Actions → Kaggleにアップロード → ブラウザでSubmit
```

1. ローカルまたは任意のデバイスで `.ipynb` を編集
2. `git add && git commit && git push`
3. `gh workflow run kaggle-push.yml -f notebook_dir=deep-past`
4. GitHub ActionsがKaggle APIでノートブックをアップロード
5. **ブラウザでKaggleのNotebook画面を開き「Submit to Competition」をクリック**

## なぜ完全自動化できないのか

当初は提出まで完全自動化する予定でしたが、**Kaggle APIの制限により、コードコンペのNotebook提出（`CreateCodeSubmission`）はAPI経由では実行できません**（2026年2月時点）。

```
requests.exceptions.HTTPError: 403 Client Error: Forbidden
```

エラーの詳細：`Permission 'kernelSessions.get' was denied`

### 技術的な理由

- **権限スコープの制限**: `kernelSessions.get`はNotebookの実行セッション状態を監視するための権限で、公開APIトークン（Legacy / 新形式とも）にはこのスコープが含まれていない
- **コード提出型コンペの特殊性**: 単なるCSVファイルのアップロードとは異なり、Notebook提出はプラットフォーム上での再実行と進捗監視を伴うため、より高度な制御権限が必要
- **ファイル提出（`competitions submit`）も不可**: コードコンペではCSVの直接アップロードは受け付けられない（400 Bad Request）

### 検証した組み合わせ

| 認証方式 | CLI Version | API | 結果 |
|---|---|---|---|
| Legacy API Key | 1.8.4 | `competition_submit_code` | 403 |
| 新 API Token（KGAT_...） | 1.8.4 | `competition_submit_code` | 403 |
| 新 API Token（KGAT_...） | 2.0.0 | `competition_submit_code` | 403 |
| 新 API Token（KGAT_...） | 2.0.0 | `competitions submit`（ファイル） | 400 |

結論として、**APIの仕様が更新されるまでは「コードのデプロイまでを自動化し、最終提出のみブラウザで行う」というハイブリッド運用が最も現実的**です。

## ディレクトリ構成

```
kaggle-competitions/
├── .github/workflows/
│   └── kaggle-push.yml        # Kaggleへのpush自動化
├── deep-past/                  # コンペごとにディレクトリを切る
│   ├── kernel-metadata.json
│   └── deep-past-baseline.ipynb
└── KAGGLE_WORKFLOW.md
```

## kernel-metadata.json

```json
{
  "id": "your-username/your-kernel-slug",
  "title": "Your Kernel Title",
  "code_file": "your-notebook.ipynb",
  "language": "python",
  "kernel_type": "notebook",
  "is_private": "true",
  "enable_gpu": "false",
  "enable_tpu": "false",
  "enable_internet": "false",
  "dataset_sources": [],
  "competition_sources": ["competition-slug"],
  "kernel_sources": [],
  "model_sources": []
}
```

### 重要なポイント

- **`enable_internet: "false"`**: コードコンペでは必須。Internet ONだとNotebookが提出対象にならない
- **`competition_sources`**: コンペのデータがマウントされる。ただしパスに注意（後述）
- **`is_private: "true"`**: 提出用。メダル狙いの公開用は別Notebookを作る

## GitHub Actionsワークフロー

```yaml
name: Kaggle Kernels Push

on:
  workflow_dispatch:
    inputs:
      notebook_dir:
        description: 'Notebook directory (e.g., deep-past)'
        required: true
        type: string

jobs:
  push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install kaggle CLI
        run: pip install kaggle

      - name: Push notebook to Kaggle
        env:
          KAGGLE_API_TOKEN: ${{ secrets.KAGGLE_API_TOKEN }}
        run: kaggle kernels push -p "${{ inputs.notebook_dir }}"
```

### GitHub Secretsの設定

| Secret | 内容 |
|---|---|
| `KAGGLE_API_TOKEN` | Kaggle Settings → API Tokens → Generate で取得（`KGAT_...`形式） |

## ハマりどころ

### 1. データパスの罠

`competition_sources`のマウント先は：

```
/kaggle/input/competitions/<competition-slug>/
```

**`/kaggle/input/<competition-slug>/`ではありません。** `competitions/`サブディレクトリが入ります。

デバッグ用コード：

```python
from pathlib import Path
for item in sorted(Path('/kaggle/input').iterdir()):
    print(f'  {item.name}/')
    for sub in sorted(item.iterdir()):
        print(f'    {sub.name} ({sub.stat().st_size:,} bytes)')
```

### 2. `kernels output`のWindowsエンコーディング問題

非ASCII文字（アッカド語等）を含むログは`kaggle kernels output`でcp932エラーになります。API直叩きで回避：

```python
import base64, json, urllib.request, pathlib

creds = json.loads(pathlib.Path.home().joinpath('.kaggle/kaggle.json').read_text())
username = creds['username']
key = creds['key']
auth = base64.b64encode(f'{username}:{key}'.encode()).decode()

url = f'https://www.kaggle.com/api/v1/kernels/output?userName={username}&kernelSlug=your-kernel-slug'
req = urllib.request.Request(url)
req.add_header('Authorization', f'Basic {auth}')

resp = urllib.request.urlopen(req)
outer = json.loads(resp.read().decode('utf-8'))
log = json.loads(outer['logNullable'])

for e in log:
    if e.get('stream_name') == 'stdout':
        print(e['data'], end='')
```

### 3. Internet ONだと提出できない

コードコンペで`enable_internet: "true"`にすると、Notebookは正常に実行されますが**提出対象として認識されません**。必ず`"false"`にしてください。

## 運用の流れ（実例）

Deep Past Challenge（Akkadian→English翻訳コンペ）での実際の流れ：

1. ローカルでベースラインNotebookを作成（TF-IDF nearest neighbor）
2. `git push` → GitHub Actions → Kaggleにアップロード
3. ブラウザでSubmit
4. Public Score: 5.6（1795位）でベースライン提出完了

スコア改善のイテレーションも同じ流れで回せます。

## まとめ

| 工程 | 自動化 | 手段 |
|---|---|---|
| ノートブック編集 | - | ローカル / 任意のデバイス |
| Kaggleへのアップロード | **自動** | GitHub Actions + `kaggle kernels push` |
| 提出 | **手動** | ブラウザで「Submit to Competition」 |
| スコア確認 | **自動** | `kaggle competitions submissions` API |

完全自動化には至りませんでしたが、コードの管理・デプロイまでをGitHub経由で行えるため、**どのデバイスからでもgit pushするだけでKaggleにコードが反映**されます。

提出ボタンを押す1クリックだけが手動ですが、それ以外の工程はすべてクラウドで完結します。
