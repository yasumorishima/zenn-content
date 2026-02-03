---
title: "GitHub CodespaceでNext.js + Supabaseを動かす手順（Docker不要）"
emoji: "☁️"
type: "tech"
topics: ["github", "codespace", "nextjs", "supabase", "docker"]
published: true
---

## はじめに

OSS開発でNext.js + Supabase構成のプロジェクトに貢献しようとしたら、ローカル環境にDockerが必要だった。自分のPCはRAM 4GB（Celeron N4500）でDocker Desktopが動かないため、GitHub Codespaceで対応した。

この記事では、CodespaceでSupabaseを起動してUI確認するまでの手順と、ハマったポイントをまとめる。

## 前提

- PCスペック不足でDocker Desktopが使えない（RAM 4GB等）
- OSSプロジェクトがDocker前提の開発環境（Supabase CLI等）
- GitHub CLIがインストール済み（`gh`コマンドが使える）

## 手順

### 1. Codespace作成

```bash
gh codespace create \
  --repo <ユーザー名>/<リポジトリ名> \
  --branch <ブランチ名> \
  --machine basicLinux32gb
```

作成後、ブラウザで開く:
```bash
gh codespace code --codespace <codespace名> --web
```

:::message
`--machine basicLinux32gb`で十分。Supabase起動 + Next.js devサーバーが問題なく動く。
:::

### 2. Supabase起動

Codespaceのターミナルで:
```bash
# pnpmの場合、グローバルセットアップ（初回のみ）
pnpm setup && source ~/.bashrc

# Supabase起動（npx経由ならCLIインストール不要）
npx supabase start
```

出力される以下の値をメモしておく:
- **Publishable**（anon key）
- **Secret**（service_role key）

### 3. 環境変数設定

```bash
# .env.local にキーを設定（プロジェクトのテンプレートに合わせて変更）
sed -i "s|your-anon-key|<Publishableの値>|" .env.local
sed -i "s|service-role-key|<Secretの値>|" .env.local
```

### 4. ポート54321をPublicに変更

Codespace画面下部の「PORTS」タブを開く:
1. **Add Port** → `54321` を追加
2. 追加したポートを **右クリック** → **Port Visibility** → **Public**

:::message alert
これをやらないと、ブラウザからSupabaseのAPIに到達できない。
:::

### 5. Supabase URLをフォワードURLに変更

```bash
# codespace名を確認
echo $CODESPACE_NAME

# .env.local内のlocalhostをフォワードURLに置換
sed -i "s|http://localhost:54321|https://<codespace名>-54321.app.github.dev|g" .env.local
cp .env.local .env
```

### 6. DB初期化

```bash
pnpm run db:reset
# または npx supabase db reset
```

### 7. Next.js Server Actionsのオリジン許可

`next.config.ts` の `serverActions` に `allowedOrigins` を追加:

```typescript
// next.config.ts
const nextConfig = {
  experimental: {
    serverActions: {
      bodySizeLimit: "10mb",
      allowedOrigins: [
        "localhost:3000",
        "<codespace名>-3000.app.github.dev",
      ],
    },
  },
};
```

sedで追加する場合:
```bash
sed -i 's|bodySizeLimit: "10mb",|bodySizeLimit: "10mb", allowedOrigins:["localhost:3000","<codespace名>-3000.app.github.dev"],|' next.config.ts
```

### 8. 開発サーバー起動 → ブラウザでアクセス

```bash
pnpm dev
```

ブラウザで以下を開く:
- **アプリ**: `https://<codespace名>-3000.app.github.dev`
- **Supabase API**（初回のみ）: `https://<codespace名>-54321.app.github.dev` を開いて「Continue」を押す

## ハマりポイントまとめ

| 症状 | 原因 | 解決策 |
|------|------|--------|
| `Missing NEXT_PUBLIC_SUPABASE_URL` | `.env.local`が未設定 | 手順3でキーを設定 |
| `Invalid Server Actions request` | Next.jsのオリジン不一致 | 手順7で`allowedOrigins`追加 |
| `ERR_BLOCKED_BY_CLIENT` / 空エラー`{}` | Supabase URLが`localhost`のまま | 手順5でフォワードURLに変更 |
| 54321ポートにアクセス不可 | ポートがPrivateのまま | 手順4でPublicに変更 |
| `supabase: command not found` | Supabase CLI未インストール | `npx supabase`で代用 |

## localhost問題の詳細

手順5のフォワードURL設定について補足する。

Codespace内で`npx supabase start`すると`http://localhost:54321`でDBが起動する。しかしブラウザはユーザーのPCで動いているため、Codespace内のlocalhostには直接アクセスできない。

```
ユーザーのPC (ブラウザ)
  ↓ https://<codespace名>-3000.app.github.dev
Codespace (Next.jsサーバー)
  ↓ localhost:54321 → ブラウザから到達不可
Codespace (Supabase)
```

Next.jsのServer Actionsもブラウザからのリクエスト起点で動くため、`NEXT_PUBLIC_SUPABASE_URL`が`localhost`だとブラウザがSupabaseに到達できずエラーになる。

**解決策**: `.env.local`のSupabase URLをCodespaceのフォワードURL（`https://<codespace名>-54321.app.github.dev`）に書き換える。

## 広告ブロッカーは無関係だった

最初はBraveの広告ブロッカーが原因かと思い、Chromeでも試した。結果は同じ。問題はブラウザの拡張機能ではなく、localhostの到達性だった。原因の切り分けは大事。

## テストデータの操作

Codespaceのターミナルから直接DBを操作できる:

```bash
# テーブル構造確認
psql postgresql://postgres:postgres@127.0.0.1:54322/postgres \
  -c "\d テーブル名"

# データ確認
psql postgresql://postgres:postgres@127.0.0.1:54322/postgres \
  -c "SELECT * FROM テーブル名 LIMIT 5"
```

:::message
DB操作は**Codespace内のターミナル**から行うので、こちらは`127.0.0.1`（localhost）でOK。ブラウザを経由しないため。
:::

## Codespaceの停止

使い終わったら忘れずに停止:
```bash
gh codespace stop --codespace <codespace名>
```

:::message
停止し忘れると無料枠を消費するので注意。GitHubの無料枠はPersonalアカウントで月120コア時間。`basicLinux32gb`（2コア）なら60時間使える。
:::

## まとめ

- CodespaceならローカルにDockerがなくてもSupabase環境を使える
- ポイントはSupabase URLをフォワードURLに変更すること
- Next.js Server Actionsの`allowedOrigins`設定も必要
- エラー時は広告ブロッカー等より先にネットワーク到達性を確認する
