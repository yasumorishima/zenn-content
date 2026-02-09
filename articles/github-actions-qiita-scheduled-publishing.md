---
title: "GitHub ActionsでQiita記事の予約投稿を自動化する"
emoji: "🤖"
type: "tech"
topics: ["githubactions", "qiita", "automation", "ci", "devops"]
published: false
published_at: 2026-02-09 12:00
---

# はじめに

Qiita CLIを使ってGitHub経由で記事を管理していますが、**Qiitaには予約投稿機能がありません**。

そこで、GitHub Actionsを使って予約投稿を実現する方法を紹介します。

## 完成イメージ

```markdown
---
title: 記事タイトル
published_at: 2026-02-09 12:00  ← この時刻に自動公開
---
```

記事に`published_at`を書いてpushするだけで、指定時刻に自動公開されます🎉

## 仕組み

1. 記事を`published_at`付きでGitHubにpush
2. GitHub Actionsが毎日定時（例: 12:00）に起動
3. `published_at`が現在時刻以前の記事を検索
4. `published_at`を削除してpush
5. Qiita CLIが変更を検知して自動公開

## 実装

### 1. ワークフローファイルを作成

`.github/workflows/scheduled-publish.yml`:

```yaml
name: Scheduled Article Publishing

on:
  schedule:
    # JST 12:00と12:30 = UTC 03:00と03:30 (日本時間 - 9時間)
    - cron: '0,30 3 * * *'
  workflow_dispatch: # 手動実行も可能

permissions:
  contents: write

jobs:
  publish-scheduled-articles:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Qiita CLI
        run: npm install -g @qiita/qiita-cli

      - name: Check for scheduled articles
        id: check
        run: |
          # 現在時刻（JST）を取得
          CURRENT_TIME=$(TZ=Asia/Tokyo date +"%Y-%m-%d %H:%M")
          echo "Current time (JST): $CURRENT_TIME"

          # published_at が現在時刻以前の記事を探す
          ARTICLES_TO_PUBLISH=""

          for file in public/*.md; do
            if [ -f "$file" ]; then
              # published_at を抽出
              PUBLISH_TIME=$(grep "^published_at:" "$file" | sed 's/published_at: //' | tr -d '\r')

              if [ -n "$PUBLISH_TIME" ]; then
                echo "Found scheduled article: $file (publish at: $PUBLISH_TIME)"

                # 時刻比較（YYYY-MM-DD HH:MM形式）
                if [[ "$PUBLISH_TIME" < "$CURRENT_TIME" ]] || [[ "$PUBLISH_TIME" == "$CURRENT_TIME" ]]; then
                  echo "  → Ready to publish!"
                  ARTICLES_TO_PUBLISH="$ARTICLES_TO_PUBLISH $file"
                else
                  echo "  → Not yet (scheduled for $PUBLISH_TIME)"
                fi
              fi
            fi
          done

          echo "articles=$ARTICLES_TO_PUBLISH" >> $GITHUB_OUTPUT

      - name: Publish articles
        if: steps.check.outputs.articles != ''
        env:
          QIITA_TOKEN: ${{ secrets.QIITA_TOKEN }}
        run: |
          for file in ${{ steps.check.outputs.articles }}; do
            echo "Publishing: $file"

            # published_at を削除（実際に公開）
            sed -i '/^published_at:/d' "$file"

            # Git設定
            git config user.name "GitHub Actions"
            git config user.email "actions@github.com"

            # コミット
            git add "$file"
            git commit -m "Publish scheduled article: $(basename $file)"
          done

          # プッシュ（Qiita CLIが自動的に記事を公開）
          git push

      - name: Summary
        run: |
          if [ -n "${{ steps.check.outputs.articles }}" ]; then
            echo "✅ Published articles: ${{ steps.check.outputs.articles }}"
          else
            echo "ℹ️ No articles scheduled for this time"
          fi
```

### 2. QIITA_TOKENをSecretsに設定

GitHub リポジトリ → Settings → Secrets and variables → Actions → New repository secret

- Name: `QIITA_TOKEN`
- Secret: Qiitaのアクセストークン

### 3. 記事を書く

`public/my-article.md`:

```markdown
---
title: 予約投稿のテスト
tags:
  - Test
private: false
published_at: 2026-02-09 12:00  ← 公開時刻を指定
---

この記事は12:00に自動公開されます！
```

### 4. GitHubにpush

```bash
git add public/my-article.md
git commit -m "Add scheduled article"
git push
```

あとは待つだけ！12:00になると自動的に公開されます🎉

## ポイント解説

### cron式とタイムゾーン

```yaml
schedule:
  - cron: '0,30 3 * * *'  # UTC 03:00と03:30 = JST 12:00と12:30
```

GitHub ActionsはUTCで動作するため、JSTから9時間引く必要があります。

カンマ区切りで複数の分を指定できます（`0,30` = 0分と30分）。

### workflow_dispatch

```yaml
on:
  schedule:
    - cron: '0 3 * * *'
  workflow_dispatch:  # ←これで手動実行も可能
```

テスト時や緊急時に手動実行できるようにしておくと便利です。

### 時刻比較

```bash
if [[ "$PUBLISH_TIME" < "$CURRENT_TIME" ]]; then
  # 公開時刻が過去 → 公開
fi
```

`YYYY-MM-DD HH:MM`形式なら、文字列比較で時刻の前後を判定できます。

## ⚠️ 重要：通常のpushでエラーを防ぐ

`published_at`付きの記事をpushすると、通常の`publish.yml`ワークフローが**即座に公開しようとしてエラーになります**。

これを防ぐため、`publish.yml`に以下の処理を追加します：

```yaml
# .github/workflows/publish.yml
jobs:
  publish_articles:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      # ← ここから追加
      - name: Skip scheduled articles temporarily
        run: |
          # published_at がある記事は一時的にスキップ
          for file in public/*.md; do
            if [ -f "$file" ] && grep -q "^published_at:" "$file"; then
              echo "Temporarily ignoring scheduled article: $file"
              sed -i 's/^ignorePublish: false/ignorePublish: true/' "$file"
            fi
          done
      # ← ここまで追加

      - uses: increments/qiita-cli/actions/publish@v1
        with:
          qiita-token: ${{ secrets.QIITA_TOKEN }}
          root: "."

      # ← ここから追加
      - name: Restore ignorePublish setting
        if: always()
        run: |
          # ignorePublish を元に戻す（scheduled-publish用）
          for file in public/*.md; do
            if [ -f "$file" ] && grep -q "^published_at:" "$file"; then
              sed -i 's/^ignorePublish: true/ignorePublish: false/' "$file"
            fi
          done
      # ← ここまで追加
```

### 仕組み

1. **push時**: `published_at`がある記事を`ignorePublish: true`に変更 → Qiita CLIがスキップ → 元に戻す
2. **scheduled-publish時**: `published_at`を削除してpush → 通常の`publish.yml`が動いて公開

これで、予約投稿の記事をpushしてもエラーが出なくなります✅

## 応用編

### 複数の公開時刻に対応

**方法1: カンマ区切り**（同じ時の複数の分）

```yaml
schedule:
  - cron: '0,30 3 * * *'  # JST 12:00と12:30
```

**方法2: 複数のcron**（異なる時刻）

```yaml
schedule:
  - cron: '0 3 * * *'   # JST 12:00
  - cron: '0 9 * * *'   # JST 18:00
  - cron: '0 12 * * *'  # JST 21:00
```

**方法3: 30分ごと**（きめ細かくチェック）

```yaml
schedule:
  - cron: '*/30 * * * *'  # 30分ごと
```

### Slack通知

公開時にSlackに通知を送ることもできます：

```yaml
- name: Notify Slack
  uses: slackapi/slack-github-action@v1
  with:
    payload: |
      {
        "text": "記事を公開しました: ${{ steps.check.outputs.articles }}"
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

## Zennとの比較

Zennには元々予約投稿機能があります：

```markdown
---
published_at: 2026-02-09 12:00
---
```

これだけで予約投稿できます。GitHub Actionsは不要です。

## おわりに

GitHub Actionsを使えば、Qiitaでも予約投稿が実現できます。

業務時間中に記事を書いて、お昼休みや退勤後に自動公開する...といった使い方ができて便利です！

## 参考

- [GitHub Actions公式ドキュメント](https://docs.github.com/ja/actions)
- [Qiita CLI](https://github.com/increments/qiita-cli)
- [cron式の書き方](https://crontab.guru/)
