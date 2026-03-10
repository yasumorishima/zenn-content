---
title: "GitHub Actionsで複数リポのREADME統計を自動同期する"
emoji: "🔄"
type: "tech"
topics: ["githubactions", "github", "自動化", "python", "oss"]
published: false
---

## はじめに

GitHubで複数リポジトリを運用していると、プロフィールREADMEに書いたOSS貢献数やPRステータスが古くなりがちです。

この記事では、**oss-contributions リポのREADMEをSingle Source of Truthとして、プロフィールREADMEへ自動同期する仕組み**を紹介します。

実際にハマったPAT権限の問題と、それを構造的に解決した方法も共有します。

## 課題：数字がずれる

筆者のGitHubプロフィールREADMEには、OSS貢献の合計PR数やMerged数を表示しています。

```markdown
<!-- OSS_STATS_START -->(40 PRs / 17 Merged)<!-- OSS_STATS_END --> across 12 repositories
```

一方、[oss-contributions](https://github.com/your-username/oss-contributions) リポには全PRの詳細が管理されており、毎週月曜に自動リフレッシュされています。

問題は、プロフィール側の更新スクリプトが**個別にリポごとのPR数をAPIで取得**していたこと。リポのリストが不完全だと、数字がずれます。

実際に起きていたズレ：

| 項目 | プロフィール | oss-contributions |
|---|---|---|
| 合計PR | 40 | **43** |
| Merged | 17 | **21** |
| リポ数 | 12 | **14** |

## 解決：Single Source of Truth パターン

個別APIコールをやめて、**oss-contributions READMEのサマリーテーブルをparseする方式**に変更しました。

```
oss-contributions README（月曜09:00 UTC自動更新）
        ↓ GitHub API で取得
プロフィール README（月曜09:30 UTC自動更新）
```

### oss-contributions のサマリーテーブル

```markdown
| Project | Language | PRs | Merged | Open | Closed |
|---|---|---|---|---|---|
| [action-board](...) | TypeScript | 13 | 11 | 0 | 2 |
| [optuna](...) | Python | 5 | 5 | 0 | 0 |
| ...
| **Total** | | **43** | **21** | **13** | **9** |
```

### パーサー

```python
def fetch_oss_readme() -> str | None:
    """oss-contributions README.md を GitHub API で取得"""
    output = run([
        "gh", "api",
        "repos/your-username/oss-contributions/contents/README.md",
        "--jq", ".content",
    ])
    if not output:
        return None
    return base64.b64decode(output).decode("utf-8")


def parse_oss_summary(text: str) -> list[dict]:
    """サマリーテーブルをパース"""
    entries = []
    for m in re.finditer(
        r"\| \[(\w[\w-]*)\]\(https://github\.com/([^)]+)\) \|"
        r" ([^|]+) \| (\d+) \| (\d+) \| (\d+) \| (\d+) \|",
        text,
    ):
        entries.append({
            "repo": m.group(2),
            "total": int(m.group(4)),
            "merged": int(m.group(5)),
            "open": int(m.group(6)),
            "closed": int(m.group(7)),
        })
    return entries
```

これにより、oss-contributionsにリポを追加すれば、プロフィール側は自動で反映されます。リスト漏れが構造的に起きません。

## ハマりポイント：PAT vs GITHUB_TOKEN

### 最初の設計（失敗）

最初は oss-contributions のワークフローから、PATを使ってプロフィールリポにpushする設計でした。

```yaml
# oss-contributions/.github/workflows/refresh-contributions.yml
- name: Sync stats to profile README
  env:
    GH_TOKEN: ${{ secrets.MY_PAT }}
  run: |
    git clone https://x-access-token:${GH_TOKEN}@github.com/user/user.git /tmp/profile
    # ... 更新 ...
    git push  # ← 403 Permission denied
```

Fine-grained PATで「All repositories」「Contents: Read and write」に設定しても、`git push` で403エラーが発生しました。

```
remote: Permission to user/user.git denied to user.
fatal: unable to access '...': The requested URL returned error: 403
```

GitHub Contents API (`gh api ... -X PUT`) でも同じ403。

### 解決策：pull方式

対象リポ（プロフィール）側にワークフローを置いて、`GITHUB_TOKEN` で自分自身のREADMEを更新する方式に変更しました。

```
【Before: push方式】
oss-contributions → (PAT) → プロフィールリポ  ❌ 403

【After: pull方式】
プロフィールリポ → (GITHUB_TOKEN + gh api) → oss-contributions README を読む
              → (GITHUB_TOKEN) → 自分のREADMEを更新  ✅
```

`GITHUB_TOKEN` は自リポへのwrite権限を常に持っているため、PAT管理が不要になりました。

### ワークフロー構成

```yaml
# your-username/.github/workflows/update-profile.yml
on:
  schedule:
    - cron: '30 9 * * 1'  # 月曜09:30 UTC（oss-contributions更新の30分後）
  workflow_dispatch:

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - name: Update README stats
        env:
          GH_TOKEN: ${{ github.token }}
        run: python scripts/update_readme.py
      - name: Commit and push if changed
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add README.md
          if ! git diff --cached --quiet; then
            git commit -m "docs: update profile stats $(date -u +%Y-%m-%d)"
            git push
          fi
```

## 同期される内容

スクリプトは以下を自動同期します：

1. **合計統計**（`<!-- OSS_STATS_START -->` マーカー内）
2. **リポジトリ数**（`across N repositories` テキスト）
3. **サブグループ統計**（team-mirai等のマーカー）
4. **PRステータス更新**（Open → Merged等の変更反映）
5. **新規PR追加**（oss-contributionsに追加されたPRの自動反映）

HTMLコメントマーカーを使うことで、READMEの他の部分に影響を与えずに更新できます。

```markdown
<!-- OSS_STATS_START -->(43 PRs / 21 Merged)<!-- OSS_STATS_END -->
```

## まとめ

| 方針 | 説明 |
|---|---|
| Single Source of Truth | oss-contributions READMEを唯一のデータソースにする |
| pull方式 | 対象リポ側でGITHUB_TOKENを使い、PATを不要にする |
| cronずらし | データソースの更新後に同期を実行する |
| HTMLコメントマーカー | 更新箇所を明示し、他のコンテンツへの影響を防ぐ |

「複数リポで同じ数字を表示したいけど手動更新は面倒」という場面で参考になれば幸いです。
