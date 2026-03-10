---
title: "GitHub ActionsでリポをまたいだREADME自動更新を実装する"
emoji: "🔄"
type: "tech"
topics: ["githubactions", "github", "自動化", "python"]
published: false
---

## はじめに

GitHubで複数リポジトリを運用していると、**あるリポのデータを別のリポのREADMEに反映したい**場面があります。

- プロフィールREADMEに各リポの統計を表示したい
- モノレポのサマリーを別リポに同期したい
- パッケージのバージョンを横断的に表示したい

手動更新だと忘れるし、数字がずれます。この記事では、GitHub Actionsで**クロスリポのREADME自動同期**を実装する方法と、ハマりやすいポイントを紹介します。

## 設計パターン：push vs pull

クロスリポ同期には2つの方式があります。

### push方式（データ元 → 対象リポへ書き込む）

```
データ元リポ → (PAT) → 対象リポのREADMEを更新
```

- PATが必要（`GITHUB_TOKEN` は自リポにしか書き込めない）
- PATの権限管理が面倒
- Fine-grained PATだと権限不足で403になるケースがある

### pull方式（対象リポがデータ元を読みに行く）

```
対象リポ → (GITHUB_TOKEN) → データ元のREADMEをAPI経由で読む
        → (GITHUB_TOKEN) → 自分のREADMEを更新
```

- PAT不要（`GITHUB_TOKEN` で完結）
- publicリポのデータなら権限の心配なし
- 対象リポ側にワークフローを置くだけ

**結論：pull方式が圧倒的に楽です。** PATの発行・管理・権限トラブルをすべて回避できます。

## 実装

### 1. HTMLコメントマーカーを用意する

更新対象のREADMEに、自動更新箇所をマーカーで囲みます。

```markdown
## Stats

<!-- STATS_START -->(10 PRs / 5 Merged)<!-- STATS_END --> across 3 repositories.
```

マーカーの間だけを書き換えるので、READMEの他の部分に影響しません。

### 2. データ元のREADMEをパースするスクリプト

```python
import base64
import re
import subprocess
import sys
from pathlib import Path

README = Path(__file__).resolve().parent.parent / "README.md"


def run(cmd: list[str]) -> str:
    result = subprocess.run(cmd, capture_output=True, text=True, timeout=120)
    return result.stdout.strip()


def fetch_source_readme(owner: str, repo: str) -> str | None:
    """GitHub API経由でREADMEを取得（publicリポならトークン不要）"""
    output = run([
        "gh", "api",
        f"repos/{owner}/{repo}/contents/README.md",
        "--jq", ".content",
    ])
    if not output:
        return None
    return base64.b64decode(output).decode("utf-8")


def replace_marker(text: str, marker: str, replacement: str) -> str:
    """HTMLコメントマーカー間のテキストを置換"""
    pattern = rf"(<!-- {marker}_START -->).*?(<!-- {marker}_END -->)"
    return re.sub(pattern, rf"\1{replacement}\2", text, flags=re.DOTALL)


def parse_stats(source_text: str) -> dict:
    """データ元READMEから統計を抽出（テーブル形式を想定）"""
    # 例: | **Total** | | **43** | **21** | ... |
    m = re.search(
        r"\| \*\*Total\*\* \|.*?\| \*\*(\d+)\*\* \| \*\*(\d+)\*\*",
        source_text,
    )
    if not m:
        return {}
    return {"total": int(m.group(1)), "merged": int(m.group(2))}


def main():
    source = fetch_source_readme("your-org", "your-source-repo")
    if not source:
        print("Failed to fetch source README", file=sys.stderr)
        sys.exit(1)

    stats = parse_stats(source)
    if not stats:
        print("Failed to parse stats", file=sys.stderr)
        sys.exit(1)

    readme = README.read_text(encoding="utf-8")
    readme = replace_marker(
        readme, "STATS",
        f"({stats['total']} PRs / {stats['merged']} Merged)",
    )
    README.write_text(readme, encoding="utf-8")
    print(f"Updated: {stats['total']} PRs / {stats['merged']} Merged")


if __name__ == "__main__":
    main()
```

### 3. ワークフロー

```yaml
name: Sync README Stats

on:
  schedule:
    # データ元の更新スケジュールより後に実行する
    - cron: '30 9 * * 1'
  workflow_dispatch:

permissions:
  contents: write

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Sync stats from source repo
        env:
          GH_TOKEN: ${{ github.token }}
        run: python scripts/sync_stats.py

      - name: Commit and push if changed
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add README.md
          if ! git diff --cached --quiet; then
            git commit -m "docs: sync stats $(date -u +%Y-%m-%d)"
            git push
          fi
```

## ハマりやすいポイント

### PAT方式の403エラー

push方式で実装すると、Fine-grained PATで「All repositories」「Contents: Read and write」に設定しても403エラーになることがあります。

```
remote: Permission to user/repo.git denied to user.
fatal: unable to access '...': The requested URL returned error: 403
```

GitHub Contents API (`-X PUT`) でも同様に403。原因の特定が難しく、pull方式に切り替えるのが最も確実な解決策です。

### cronのタイミング

データ元リポが月曜09:00に更新される場合、同期ワークフローは**09:30以降**にスケジュールします。

```yaml
# ❌ データ元と同じ時間 → 古いデータを取得する可能性
- cron: '0 9 * * 1'

# ✅ データ元の更新後に実行
- cron: '30 9 * * 1'
```

### マーカーの設計

マーカー名は衝突しないように、用途ごとにユニークにします。

```markdown
<!-- PROJECT_STATS_START -->...<!-- PROJECT_STATS_END -->
<!-- BADGE_COUNT_START -->...<!-- BADGE_COUNT_END -->
```

`replace_marker` 関数はマーカー間のテキストだけを置換するので、READMEの構造を壊しません。

## まとめ

| ポイント | 説明 |
|---|---|
| **pull方式を使う** | 対象リポ側にワークフローを置き、GITHUB_TOKENで完結させる |
| **HTMLコメントマーカー** | 更新箇所を明示し、他の内容への影響を防ぐ |
| **cronをずらす** | データ元の更新完了後に同期を実行する |
| **Single Source of Truth** | データ元を1つに絞り、他のリポはそこからpullする |

同じ数字を複数のREADMEに表示している方は、ぜひ試してみてください。
