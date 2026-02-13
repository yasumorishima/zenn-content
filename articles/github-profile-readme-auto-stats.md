---
title: "GitHub Profile READMEの統計を自動更新する（GitHub Actions + Python）"
emoji: "🤖"
type: "tech"
topics: [github,githubactions,python,automation]
published: true
---

## はじめに

GitHub Profile README（`username/username` リポジトリの `README.md`）に、OSS貢献のPR数やKaggleの統計を載せている方は多いと思います。

ただ、PRが増えるたびに手動で数字を更新するのは面倒です。

この記事では、**GitHub Actions + Python スクリプトで週次自動更新する仕組み**を紹介します。

## 完成イメージ

毎週月曜に GitHub Actions が自動実行され、以下の数字が最新に更新されます：

- OSS全体のPR数 / Merged数
- 特定Organization（team-mirai等）のPR統計
- Kaggle Dataset数 / Notebook数
- リポジトリ内のノートブック数

## 仕組み

### 1. README.md にHTMLマーカーを埋め込む

更新したい数字をHTMLコメントで囲みます。

```markdown
<!-- OSS_STATS_START -->(35 PRs / 14 Merged)<!-- OSS_STATS_END -->
```

マーカーはHTMLコメントなのでGitHub上では表示されません。Python スクリプトがこのマーカー間のテキストを正規表現で置換します。

### 2. Python スクリプト（`scripts/update_readme.py`）

#### PR統計の取得

`gh search prs` コマンドでリポジトリごとにPRを検索し、状態（merged/open/closed）を集計します。

```python
def search_prs(repos: list[str]) -> dict[str, int]:
    """Get PR counts using gh search prs, one repo at a time."""
    all_prs = []
    for repo in repos:
        output = run([
            "gh", "search", "prs",
            "--author", AUTHOR,
            "--repo", repo,
            "--limit", "200",
            "--json", "state",
        ])
        if not output:
            continue
        all_prs.extend(json.loads(output))

    merged = sum(1 for p in all_prs if p["state"].upper() == "MERGED")
    open_count = sum(1 for p in all_prs if p["state"].upper() == "OPEN")
    closed = sum(1 for p in all_prs if p["state"].upper() == "CLOSED")
    return {
        "total": len(all_prs),
        "merged": merged,
        "open": open_count,
        "closed": closed,
    }
```

**ポイント**:
- `gh search prs` は GitHub Search API を使うため、デフォルトの `GITHUB_TOKEN` で他のpublicリポも検索可能
- `gh pr list` だと `GITHUB_TOKEN` のスコープ制限で他リポにアクセスできない場合がある
- `state` の値は小文字（`"merged"` 等）で返ってくるため、`.upper()` で正規化

#### Kaggle統計の取得

```python
def get_kaggle_dataset_count() -> int | None:
    output = run([*kaggle_cmd(), "datasets", "list", "--user", "yasunorim", "--csv"])
    if not output:
        return None
    lines = output.strip().split("\n")
    count = len(lines) - 1  # CSVヘッダーを除く
    return count if count > 0 else None
```

**ポイント**: `--csv` オプションで出力をCSV形式にし、行数からカウント。API失敗時は `None` を返して既存値を維持します。

#### マーカー置換

```python
def replace_marker(text: str, marker: str, replacement: str) -> str:
    pattern = rf"(<!-- {marker}_START -->).*?(<!-- {marker}_END -->)"
    return re.sub(pattern, rf"\1{replacement}\2", text, flags=re.DOTALL)
```

### 3. 手動管理の数字は `config.json` で

Kaggleのメダル数やタイトルなど、APIで自動取得できない数字は設定ファイルで管理します。

```json
{
  "kaggle_bronze_medals": 8,
  "kaggle_title": "Notebooks Expert"
}
```

### 4. GitHub Actions ワークフロー

```yaml
name: Update Profile README

on:
  schedule:
    - cron: '0 0 * * 1'  # 毎週月曜 00:00 UTC
  workflow_dispatch:       # 手動実行も可能

permissions:
  contents: write

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Configure Kaggle credentials
        env:
          KAGGLE_USERNAME: ${{ secrets.KAGGLE_USERNAME }}
          KAGGLE_KEY: ${{ secrets.KAGGLE_KEY }}
        run: |
          mkdir -p ~/.kaggle
          echo '{"username":"'"$KAGGLE_USERNAME"'","key":"'"$KAGGLE_KEY"'"}' > ~/.kaggle/kaggle.json
          chmod 600 ~/.kaggle/kaggle.json

      - name: Update README stats
        env:
          GH_TOKEN: ${{ github.token }}
        run: python scripts/update_readme.py

      - name: Commit and push if changed
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add README.md
          if git diff --cached --quiet; then
            echo "No changes to commit"
          else
            git commit -m "docs: update profile stats $(date -u +%Y-%m-%d)"
            git push
          fi
```

## セットアップ手順

### 1. リポジトリ構成

```
username/username/
├── README.md              # HTMLマーカー入り
├── config.json            # 手動管理の数字
├── requirements.txt       # kaggle
├── scripts/
│   └── update_readme.py   # 更新スクリプト
└── .github/workflows/
    └── update-profile.yml # 週次ワークフロー
```

### 2. GitHub Secrets の設定

Kaggle APIを使う場合、リポジトリの Settings → Secrets に以下を追加：

- `KAGGLE_USERNAME`: Kaggleユーザー名
- `KAGGLE_KEY`: `~/.kaggle/kaggle.json` の `key` 値

### 3. README.md にマーカーを挿入

更新したい数字をマーカーで囲みます：

```markdown
<!-- OSS_STATS_START -->(35 PRs / 14 Merged)<!-- OSS_STATS_END -->
```

### 4. 手動テスト

```bash
# ローカルで実行（gh CLIログイン済みの場合）
python scripts/update_readme.py

# GitHub Actionsで手動実行
gh workflow run update-profile.yml
```

## ハマりポイント

### `gh pr list` vs `gh search prs`

最初は `gh pr list --repo <repo>` を使っていましたが、GitHub Actions の `GITHUB_TOKEN` では**自リポ以外へのアクセスが制限**されて全リポ0件になりました。

`gh search prs` は GitHub Search API を使うため、publicリポであればデフォルトトークンで検索できます。

### `state` の大文字・小文字

- `gh pr list --json state` → 大文字（`"MERGED"`, `"OPEN"`, `"CLOSED"`）
- `gh search prs --json state` → 小文字（`"merged"`, `"open"`, `"closed"`）

`.upper()` で正規化しておくと安全です。

### API失敗時の安全設計

Kaggle CLIが環境によって動かない場合があるため、API失敗時は `None` を返して**既存の値を維持**する設計にしています。0で上書きしてしまうと、README上の数字が一時的におかしくなります。

## まとめ

- HTMLマーカー + 正規表現置換で、README の好きな位置の数字を更新可能
- `gh search prs` でpublicリポのPR統計を追加トークンなしで取得
- API失敗時は既存値維持の安全設計
- `config.json` でAPI取得できない数字も管理
- 週次 + 手動実行のハイブリッド運用

手動更新の手間が完全になくなるので、Profile README に統計を載せている方にはおすすめです。
