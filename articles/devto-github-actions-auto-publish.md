---
title: "DEV.toにGitHub Actionsで記事を自動投稿する方法"
emoji: "🚀"
type: "tech"
topics: ["devto", "githubactions", "ci", "自動化", "ブログ"]
published: true
---

## はじめに

英語圏の技術ブログプラットフォーム「DEV.to」に、GitHub Actionsを使って記事を自動投稿する方法を紹介します。

GitHub にMarkdownファイルをpushするだけで、DEV.toに記事が自動公開される仕組みを作ります（Zenn/Qiitaと同じような使い心地）。

---

## 完成形

```
GitHubリポジトリにMarkdown push
    ↓ (GitHub Actions)
DEV.toに自動投稿（published: true）
```

**メリット:**
- Markdownファイルで記事管理
- Git履歴で変更追跡
- 1回のpushで複数サイト配信可能（Quarto Blog + DEV.to等）

---

## 前提条件

- GitHubアカウント
- DEV.toアカウント
- GitHubリポジトリ（既存のブログリポジトリでOK）

---

## Step 1: DEV.to APIキーを取得

### 1. DEV.to Settingsにアクセス

https://dev.to/settings/extensions

### 2. APIキーを生成

1. **Description**欄に名前を入力（例: `GitHub Actions Auto Publish`）
2. **Generate API Key**をクリック
3. 生成されたAPIキー（長い文字列）を**コピー**

:::message alert
APIキーは一度しか表示されません。必ずコピーしてください。
:::

---

## Step 2: GitHub Secretsに保存

### ローカルで実行（推奨）

```bash
cd your-repo
gh secret set DEV_TO_API_KEY
# APIキーを貼り付け（Enterキー2回）
```

### Web UIで設定する場合

1. GitHubリポジトリ → **Settings** → **Secrets and variables** → **Actions**
2. **New repository secret**
3. Name: `DEV_TO_API_KEY`、Value: コピーしたAPIキー
4. **Add secret**

---

## Step 3: GitHub Actionsワークフロー作成

### ディレクトリ構造

```
your-repo/
  .github/
    workflows/
      publish-to-devto.yml  ← 作成
  devto/
    article1.md  ← DEV.to用記事
    article2.md
```

### `.github/workflows/publish-to-devto.yml`

```yaml
name: Publish to DEV.to

on:
  push:
    branches: [main]
    paths:
      - 'devto/**/*.md'
      - '.github/workflows/publish-to-devto.yml'

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Publish articles to DEV.to
        env:
          DEVTO_API_KEY: ${{ secrets.DEV_TO_API_KEY }}
        run: |
          for file in devto/*.md; do
            echo "Processing $file"

            # Extract frontmatter and content
            title=$(sed -n 's/^title: //p' "$file")
            description=$(sed -n 's/^description: //p' "$file")
            tags=$(sed -n 's/^tags: //p' "$file")
            canonical_url=$(sed -n 's/^canonical_url: //p' "$file")

            # Remove frontmatter to get body
            body=$(sed '1,/^---$/d; /^---$/,$d' "$file")

            # Create JSON payload
            json_payload=$(jq -n \
              --arg title "$title" \
              --arg body "$body" \
              --arg tags "$tags" \
              --arg canonical "$canonical_url" \
              '{article: {title: $title, body_markdown: $body, tags: ($tags | split(", ")), canonical_url: $canonical, published: true}}')

            # Post to DEV.to API
            response=$(curl -X POST https://dev.to/api/articles \
              -H "Content-Type: application/json" \
              -H "api-key: $DEVTO_API_KEY" \
              -d "$json_payload")

            echo "Response: $response"

            # Wait 10 seconds between posts to avoid rate limit
            sleep 10
          done
```

---

## Step 4: 記事ファイルを作成

### `devto/my-first-article.md`

```markdown
---
title: My First Article
published: true
description: This is my first article on DEV.to
tags: javascript, webdev, tutorial
canonical_url: https://yourblog.com/my-first-article
cover_image: https://example.com/image.png
---

## Introduction

This is my first article!

## Code Example

\`\`\`javascript
console.log("Hello, DEV.to!");
\`\`\`

## Conclusion

Thank you for reading!
```

### Frontmatterの説明

| フィールド | 説明 |
|---|---|
| `title` | 記事タイトル |
| `published` | `true`で即公開、`false`で下書き |
| `description` | 記事の説明（SNSシェア時に表示） |
| `tags` | カンマ区切りのタグ（最大4つ） |
| `canonical_url` | 元記事のURL（SEO対策） |
| `cover_image` | カバー画像URL（オプション） |

---

## Step 5: GitHubにpush

```bash
git add .
git commit -m "Add DEV.to article"
git push
```

GitHub Actionsが自動実行され、DEV.toに記事が投稿されます。

---

## 動作確認

### GitHub Actionsログ確認

```bash
gh run list --limit 1
gh run view <run-id> --log
```

### DEV.toで確認

https://dev.to/dashboard

記事が公開されていることを確認してください。

---

## レート制限対策

DEV.to APIは短時間に連続投稿すると制限がかかります（429エラー）。

**対策:** ワークフローに `sleep 10` を追加（記事間に10秒の遅延）

```bash
# Wait 10 seconds between posts to avoid rate limit
sleep 10
```

---

## 応用: 複数サイトへの自動配信

同じMarkdownファイルを、複数のプラットフォームに配信できます：

```
GitHubにpush
    ↓
├→ Quarto Blog (GitHub Pages)
├→ DEV.to (GitHub Actions)
└→ Hashnode (GitHub Actions)
```

**メリット:**
- 1回の執筆で複数サイトに配信
- Markdownファイル1つで一元管理
- canonical_url設定でSEO問題なし

---

## トラブルシューティング

### 記事が投稿されない

- APIキーが正しいか確認（`gh secret list`）
- DEV.to Dashboardで記事が下書き状態になっていないか確認
- GitHub Actionsログでエラーを確認

### レート制限エラー（429）

- `sleep 10` の遅延を追加
- または、手動で5分待ってから再実行

### タグが反映されない

- タグはカンマ区切り（`tags: javascript, webdev`）
- 最大4つまで
- スペースの有無に注意

---

## 参考リンク

- [DEV.to API Documentation](https://developers.forem.com/api)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Quarto Publishing](https://quarto.org/docs/publishing/)

---

## おわりに

GitHub ActionsでDEV.toへの自動投稿を実現しました。

Markdownファイルをpushするだけで、英語圏の読者にもリーチできる仕組みが完成しました。

Hashnode等の他のプラットフォームにも同じ方法で対応できるので、ぜひ試してみてください！
