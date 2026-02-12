---
title: "Dune AnalyticsのJPYC追跡クエリをバグ修正し、GitHub Actionsで自動化した話"
emoji: "🔗"
type: "tech"
topics: ["DuneAnalytics", "SQL", "GitHubActions", "Python", "blockchain"]
published: true
---

## はじめに

以前、新JPYC（2025年10月27日ローンチ）の発行額・償還額・流通額をDune Analyticsで追跡するSQLクエリとダッシュボードを作りました。

**ダッシュボード**: https://dune.com/shogaku_toushi/jpyc-date

今回はそのクエリを見直したところバグを発見したので修正し、さらにDune APIとGitHub Actionsでデータ取得を自動化しました。この記事ではバグの内容・修正方法・自動化の仕組みをまとめます。

※ あくまで個人が作成したクエリであり、JPYC社の公式な集計方法や定義とは異なる可能性があります。

## 前回のおさらい

前回のクエリ（v1）では以下の仕組みでJPYCのオンチェーンデータを分析していました。

- **対象チェーン**: Ethereum, Polygon, Avalanche
- **コントラクト**: `0xE7C3D8C9a439feDe00D2600032D5dB0Be71C3c29`
- **JPYC社ウォレット判定**: Mintイベント（`from = 0x0`）の受取先アドレスを動的に判定
- **分類ロジック**: 各Transferを「発行（JPYC社→顧客）」「償還（顧客→JPYC社）」「一般送金」に分類

月次クエリ（8 CTE）と日次クエリ（19 CTE）の2本で構成していました。

## 見つかったバグ：Mint/Burnイベントの誤分類

### 問題

v1の`all_transfers` CTEは、ERC20 Transferイベントを**フィルタなしで**取得していました。ここにMintイベント（`from = 0x0`）とBurnイベント（`to = 0x0`）が含まれていたのが問題です。

分類ロジックの`CASE WHEN`で以下のように誤分類されていました。

**Mintイベント**（`from = 0x0` → `to = JPYC社ウォレット`）：

```
1. from(0x0)がJPYC社ウォレット？ → No（0x0はJPYC社ではない）
2. to(JPYC社ウォレット)がJPYC社ウォレット？ → Yes
→ 「償還」に分類される ← 本来はMint（内部処理）
```

**Burnイベント**（`from = JPYC社ウォレット` → `to = 0x0`）：

```
1. from(JPYC社ウォレット)がJPYC社ウォレット？ → Yes
→ 「発行」に分類される ← 本来はBurn（内部処理）
```

### 影響

v1とv2の比較結果です。

| 月 | v1 Polygon累積償還 | v2 Polygon累積償還 |
|---|---|---|
| 2025-12 | 2.85億円 | 0 |
| 2026-01 | 6.3億円 | 0 |

v1で「償還」とされていた**6.3億円は全てMintイベントの誤分類**でした。実際にはローンチから約4ヶ月間、ユーザーからの償還はまだ発生していないと考えられます。

## 修正内容（v1 → v2）

### 修正1：Mint/Burnイベントの除外

`all_transfers` CTEの各チェーンのSELECTに、ゼロアドレスのフィルタを追加しました。

```sql
-- v1（修正前）
SELECT ...
FROM erc20_ethereum.evt_Transfer
WHERE contract_address = 0xE7C3D8C9a439feDe00D2600032D5dB0Be71C3c29
AND evt_block_time >= TIMESTAMP '2025-10-27'

-- v2（修正後）
SELECT ...
FROM erc20_ethereum.evt_Transfer
WHERE contract_address = 0xE7C3D8C9a439feDe00D2600032D5dB0Be71C3c29
AND evt_block_time >= TIMESTAMP '2025-10-27'
AND "from" != 0x0000000000000000000000000000000000000000
AND "to" != 0x0000000000000000000000000000000000000000
```

3チェーン × 2クエリ = 6箇所に同じ修正を適用しました。

### 修正2：JPYC社ウォレット間の内部転送ハンドリング

v1では、from/toの両方がJPYC社ウォレットの場合（社内のウォレット間移動）、`CASE WHEN`の順序で「発行」に分類されていました。

```sql
-- v2（修正後）
CASE
    WHEN EXISTS (... from is jpyc) AND EXISTS (... to is jpyc) THEN 'internal'
    WHEN EXISTS (... from is jpyc) THEN 'issuance'
    WHEN EXISTS (... to is jpyc) THEN 'redemption'
    ELSE 'transfer'
END
```

`'internal'`は集計から除外されるため、社内移動が発行額に混入しなくなります。

### 修正3：月次クエリの活動フィルタ統一

v1では月次と日次で活動フィルタの条件が異なっていました。

- 日次：全タイプ（issuance/redemption/transfer）の活動がある日を表示
- 月次：issuance/redemptionがある月のみ表示

v2では月次も`WHERE type != 'internal'`に統一し、一般送金だけの月も表示されるようにしました。

## JPYC公式発表との比較

修正後のクエリ結果をJPYC社の公式発表と比較してみました。

| 時点 | JPYC公式 | v2クエリ結果 |
|---|---|---|
| 2025/12/15 | 累計発行額 **5億円突破** | 6.54億円（12月末時点） |
| 2026/2/2頃 | 累計発行額 **10億円突破** | 10.59億円（1月末時点） |

12月末の6.54億円は、公式の「12/15時点で5億円突破」と整合的と考えられます（12/15〜月末の約2週間分の差）。1月末の10.59億円も、公式の「2月初旬に10億円突破」とおおむね一致しています。

※ クエリの定義（JPYC社ウォレットからの送信=発行）と公式の定義が同一とは限らないため、完全一致は期待していません。近い値が出ていることから、クエリのロジックはおおむね妥当と考えられます。

参考:
- [JPYC EX累計口座10,000件・発行額5億円突破（PR TIMES）](https://prtimes.jp/main/html/rd/p/000000294.000054018.html)
- [ステーブルコインJPYC、発行額10億円に（日本経済新聞）](https://www.nikkei.com/article/DGXZQOUB044GA0U6A200C2000000/)

## Dune API × GitHub Actionsで自動化

クエリ結果をGitHubリポジトリに自動保存する仕組みを作りました。

### 構成

```
dune-analytics/
├── .github/workflows/fetch-jpyc.yml  # GitHub Actions
├── scripts/fetch_jpyc.py             # データ取得スクリプト
├── data/
│   ├── jpyc_monthly.csv              # 月次データ
│   ├── jpyc_daily.csv                # 日次データ
│   └── last_updated.txt              # 最終更新日時
├── queries/                           # SQLクエリ（v1/v2）
└── requirements.txt                   # dune-client
```

### Pythonスクリプト

`dune-client`ライブラリの`get_latest_result()`を使い、Duneのキャッシュ済み結果を取得してCSVに保存します。クエリの再実行はしないため、APIクレジットの消費を抑えられます。

```python
from dune_client.client import DuneClient

dune = DuneClient(os.environ["DUNE_API_KEY"])
result = dune.get_latest_result(query_id)
# result.result.rows → CSV出力
```

### GitHub Actionsワークフロー

毎週月曜9:00（JST）に自動実行し、変更があればコミット＆プッシュします。

```yaml
on:
  schedule:
    - cron: '0 0 * * 1'  # 月曜 00:00 UTC = 09:00 JST
  workflow_dispatch:       # 手動実行も可能

jobs:
  fetch:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install -r requirements.txt
      - run: python scripts/fetch_jpyc.py
        env:
          DUNE_API_KEY: ${{ secrets.DUNE_API_KEY }}
      - name: Commit and push if changed
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add data/
          if git diff --cached --quiet; then
            echo "No changes to commit"
          else
            git commit -m "data: update JPYC data $(date -u +%Y-%m-%d)"
            git push
          fi
```

### セットアップ手順

1. [Dune Settings](https://dune.com/settings/api) でAPIキーを生成
2. GitHubリポジトリの Settings → Secrets → `DUNE_API_KEY` に登録
3. プッシュすればワークフローが有効に

`gh secret set DUNE_API_KEY --body "your-api-key"` でCLIからも登録できます。

### 無料プランでの運用

Dune無料プランは月10,000クレジットです。`get_latest_result()`はキャッシュ取得なので低コスト。週1回の自動取得であれば余裕を持って運用できると考えられます。

### READMEに最新データを自動表示

GitHubのREADMEにHTMLコメントのマーカーを埋め込み、fetchスクリプトでその間を最新データのテーブルに置き換える仕組みにしました。

READMEに以下のマーカーを入れておきます。

```markdown
## Latest Data (Cumulative, Billion JPY)

<!-- LATEST_DATA_START -->
*Run the script to populate this section.*
<!-- LATEST_DATA_END -->
```

Pythonスクリプト側で、取得したデータからMarkdownテーブルを生成し、マーカー間を置き換えます。

```python
import re

def update_readme(repo_root, section_content):
    readme_path = repo_root / "README.md"
    readme = readme_path.read_text(encoding="utf-8")

    start = "<!-- LATEST_DATA_START -->"
    end = "<!-- LATEST_DATA_END -->"

    pattern = re.compile(re.escape(start) + r".*?" + re.escape(end), re.DOTALL)
    replacement = f"{start}\n{section_content}\n{end}"
    new_readme = pattern.sub(replacement, readme)

    readme_path.write_text(new_readme, encoding="utf-8")
```

GitHub Actionsのコミット対象に`README.md`を追加すれば、毎週自動でREADMEのテーブルが更新されます。

```yaml
      - name: Commit and push if changed
        run: |
          git add data/ README.md
          ...
```

リポジトリのREADMEを開くと、最新のデータテーブルが自動表示されます。

https://github.com/yasumorishima/dune-analytics

## まとめ

| やったこと | 内容 |
|---|---|
| バグ修正 | Mint/Burnイベントの誤分類を修正（発行・償還の精度向上） |
| 内部転送対応 | JPYC社ウォレット間の移動を`internal`として除外 |
| 公式データ照合 | 累計発行額がJPYC公式発表とおおむね一致することを確認 |
| 自動化 | Dune API + GitHub Actionsで週次CSV自動更新 |

リポジトリ: https://github.com/yasumorishima/dune-analytics

※ このクエリは個人が作成したものです。JPYC社の公式な集計方法や定義とは異なる可能性があります。参考程度にご覧ください。
