---
title: "HTTP 200でもアプリは寝ていた — Streamlit CloudのZzzをPlaywrightで防ぐ"
emoji: "🛏️"
type: "tech"
topics: ["streamlit", "playwright", "python", "raspberrypi", "githubactions"]
published: true
---

## TL;DR

- Streamlit Community Cloud のアプリは一定期間アクセスがないとスリープする（Zzz）
- `urllib` や `requests` で HTTP GET しても **200 が返るのにアプリは起きない**
- 原因: レスポンスは静的 HTML シェル（4,271 bytes）で、Python アプリは起動していない
- **Playwright（ヘッドレス Chromium）** で実際にブラウザ訪問すれば起こせる
- Raspberry Pi の Docker + GitHub Actions で 6 時間ごとに自動実行する構成にした

---

## 問題: 30 個のアプリが全部 Zzz

WBC 2026 のスカウティングダッシュボードを Streamlit Community Cloud にデプロイしています（全 20 チーム × 打者・投手 = 30 アプリ）。

しばらく使わないとアプリがスリープし、開くたびに「Zzzz — This app has gone to sleep」が表示されます。

「定期的に HTTP リクエストを送ればいいだろう」と考え、Raspberry Pi で keepalive スクリプトを動かしていました。

---

## 試行 1: urllib + CookieJar → 失敗

Streamlit Cloud は認証リダイレクト（303）があるため、最初は `urllib.request.urlopen` では 303 エラーになりました。`CookieJar` を使ってリダイレクトを追跡するようにしたところ、HTTP 200 が返るようになりました。

```python
import http.cookiejar
import urllib.request

cj = http.cookiejar.CookieJar()
opener = urllib.request.build_opener(urllib.request.HTTPCookieProcessor(cj))
req = urllib.request.Request(url, headers={"User-Agent": "Mozilla/5.0 ..."})

with opener.open(req, timeout=90) as resp:
    body = resp.read()
    print(f"OK {resp.status} ({len(body)} bytes)")
    # → OK 200 (4271 bytes)
```

全 35 URL が `OK 200 (4271 bytes)` になり、「これで解決」と思いました。

**しかし、アプリはまだ Zzz のままでした。**

---

## 調査: 全部同じ 4,271 bytes

おかしいと思い、レスポンスの中身を確認しました。

```python
text = body.decode("utf-8")
print(text[:300])
```

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    ...
    <script type="module" crossorigin src="/-/build/assets/index-BCOs3Liy.js"></script>
  </head>
  <body>
    <noscript>You need to enable JavaScript to run this app.</noscript>
    <div id="root"></div>
  </body>
</html>
```

**全 URL が同じ静的 HTML を返していました。** これは Streamlit の「シェルページ」で、JavaScript が実行されて初めてアプリの中身がロードされます。

さらに `/_stcore/health` エンドポイントも試しましたが、303 リダイレクトになり、CookieJar を使っても同じ HTML シェルが返るだけでした。

---

## 原因: SPA は HTTP GET では起きない

Streamlit Cloud のアーキテクチャを整理すると、以下のようになっています。

```
ブラウザが GET → HTML シェル（静的ファイル）を返す
  → JS が実行される
    → WebSocket で /_stcore/stream に接続
      → ここで初めて Python アプリが起動する
```

つまり:

- **HTTP GET** → 静的 HTML シェルだけ返す（Python は動かない）
- **ブラウザ訪問** → JS → WebSocket → Python 起動

`urllib` や `requests` は JavaScript を実行しないため、どんなにリクエストを送っても**アプリの Python プロセスは起動しません**。HTTP 200 は「静的ファイルの配信に成功した」という意味でしかなく、アプリが動いているかどうかとは無関係です。

:::message
これは Streamlit に限らず、**SPA（Single Page Application）全般**に当てはまります。React や Vue のアプリも、HTTP GET では HTML シェルしか返しません。ヘルスチェックで「200 が返るから大丈夫」と思っていると、実はアプリが落ちていた、ということが起こり得ます。
:::

---

## 解決: Playwright でブラウザ訪問

ヘッドレスブラウザ（Playwright + Chromium）を使えば、JavaScript を実行して WebSocket 接続を確立でき、アプリを実際に起動できます。

さらに、アプリが寝ている場合は「Yes, get this app back up!」ボタンが表示されるので、これを自動クリックします。

```python
from playwright.async_api import async_playwright

async def visit(page, url):
    await page.goto(url, wait_until="domcontentloaded", timeout=120_000)
    await page.wait_for_timeout(5000)

    wake_btn = page.get_by_role("button", name="Yes, get this app back up!")
    if await wake_btn.count() > 0:
        print(f"  WAKE  {url}")
        await wake_btn.click()
        await page.wait_for_timeout(60_000)
    else:
        print(f"  OK    {url}")
```

実行結果:

```
[2026-02-27 14:23] Visiting 35 URLs...
  OK    https://npb-prediction.streamlit.app/
  OK    https://wbc-pr-batters.streamlit.app/
  WAKE  https://wbc-cuba-batters.streamlit.app/    ← 寝てたのを起こした
  WAKE  https://wbc-can-batters.streamlit.app/
  WAKE  https://wbc-pan-batters.streamlit.app/
  ...
```

`WAKE` が表示されたアプリは実際に寝ていて、Playwright がボタンをクリックして起こしたものです。

---

## 実装: Raspberry Pi Docker + GitHub Actions

### Raspberry Pi（メイン）

Dockerfile:

```dockerfile
FROM python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    libnss3 libatk-bridge2.0-0 libdrm2 libxkbcommon0 \
    libgbm1 libpango-1.0-0 libcairo2 libasound2 \
    libatspi2.0-0 libcups2 libxcomposite1 libxdamage1 \
    libxfixes3 libxrandr2 libgtk-3-0 libdbus-glib-1-2 \
    fonts-liberation xdg-utils \
    && rm -rf /var/lib/apt/lists/*

RUN pip install --no-cache-dir playwright \
    && playwright install chromium

WORKDIR /app
COPY keepalive.py .
CMD ["python", "keepalive.py"]
```

`docker-compose.yml`:

```yaml
services:
  keepalive:
    build: .
    restart: unless-stopped
    volumes:
      - ./urls.txt:/app/urls.txt:ro
    environment:
      - PYTHONUNBUFFERED=1
```

Raspberry Pi 5（ARM64）でも `playwright install chromium` で Chromium のダウンロード・実行ができました。

### GitHub Actions（バックアップ）

```yaml
name: Streamlit Keepalive

on:
  schedule:
    - cron: '0 */6 * * *'
  workflow_dispatch:

jobs:
  keepalive:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - name: Install Playwright
        run: |
          pip install playwright
          playwright install chromium --with-deps
      - name: Ping all Streamlit apps
        run: python scripts/keepalive.py
```

---

## 失敗から学んだこと

| やったこと | 結果 | 学び |
|---|---|---|
| `urllib.request.urlopen` | 303 エラー | Streamlit Cloud は認証リダイレクトがある |
| `CookieJar` + `build_opener` | 200 が返るが寝たまま | 静的 HTML シェルが返っているだけ |
| `/_stcore/health` エンドポイント | 303 → HTML シェル | アプリが寝ているとヘルスチェックも機能しない |
| **Playwright（ヘッドレス Chromium）** | **アプリが起きた** | JS 実行 + WebSocket 接続が必要 |

---

## 応用: SPA のヘルスチェックに要注意

この問題は Streamlit 固有ではなく、SPA 全般に当てはまる構造的な話です。

- **Render / Railway / Koyeb の無料枠**: 同様のスリープ機能がある。HTTP GET では起きない可能性がある
- **ポートフォリオ・デモアプリ**: 採用担当が見に来た時に Zzz では印象が悪い。Playwright + cron で常時起動させておく
- **監視・モニタリング**: HTTP 200 = アプリ正常、ではない。SPA はブラウザレンダリング後の状態を確認すべき

**「HTTP 200 が返るから大丈夫」は、SPA の世界では通用しない。**

---

## リポジトリ

- GitHub: https://github.com/yasumorishima/wbc-scouting
- keepalive スクリプト: [`scripts/keepalive.py`](https://github.com/yasumorishima/wbc-scouting/blob/master/scripts/keepalive.py)
- GitHub Actions: [`.github/workflows/keepalive.yml`](https://github.com/yasumorishima/wbc-scouting/blob/master/.github/workflows/keepalive.yml)
