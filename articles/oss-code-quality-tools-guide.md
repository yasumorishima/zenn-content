---
title: "OSSにPRを出す前に自分のフォークでコードチェックする方法"
emoji: "🔍"
type: "tech"
topics: ["oss", "github", "coderabbit", "codeql"]
published: true
---

## この記事の対象読者

OSSにPRを出したいが、メンテナーに粗いコードを見せたくない人。

大きなOSSプロジェクトにはCIやレビューbotが整備されていることが多い。しかし、全てのプロジェクトがそうとは限らない。自分のフォークに品質チェックツールを入れておけば、どのOSSに出すときでも事前にチェックできる。

## 結論：CodeRabbit + CodeQL の2つで十分

| ツール | 役割 | 料金（public repo） |
| --- | --- | --- |
| CodeRabbit | AIコードレビュー（ロジック・品質・命名） | 無料 |
| CodeQL | セキュリティ脆弱性の静的解析 | 無料（GitHub標準） |

他にもDependabotやSonarCloudがあるが、「自分のコード変更をPR前にチェックする」目的ではこの2つで事足りる。

- **Dependabot**: upstreamの依存パッケージ管理はメンテナーの仕事。フォーク側で入れる意味が薄い
- **SonarCloud**: CodeRabbitと役割が被る上に、セットアップが手間

## ワークフロー

```
upstream/main ← PR（チェック済みのコードを提出）
    ↑
自分のフォーク/feature-branch
    ↑ PRを作る（フォーク内で feature → main）
    ↑ → CodeRabbit が自動レビュー
    ↑ → CodeQL がセキュリティスキャン
    ↑ → 指摘を修正
    ↑
ローカルで開発
```

ポイント：**upstreamに出す前に、フォーク内でPRを作ってbotにチェックさせる**。指摘を潰してからupstreamにPRを出す。

## 導入手順

### 1. CodeRabbit（5分）

1. [GitHub Marketplace の CodeRabbit ページ](https://github.com/marketplace/coderabbitai)にアクセス
2. 「Open Source」プラン（$0/月）を選択
3. 自分のアカウントを選択
4. 「Only select repositories」で対象のフォークを選択
5. 「Install & Authorize」をクリック
6. CodeRabbitのサイトにリダイレクトされたら「GitHub Cloud」でログイン

これだけで、フォーク上のPRにCodeRabbitが自動でレビューコメントを付けるようになる。

#### 便利なコマンド

- `@coderabbitai review` — PRコメントに書くと再レビュー
- `@coderabbitai resolve` — 指摘を解決済みにする
- `.coderabbit.yaml` をリポジトリルートに置くとカスタマイズ可能

### 2. CodeQL（3分）

1. フォークのGitHubページを開く
2. 「Settings」→ 左メニュー「Code security and analysis」
3. 「Code scanning」セクションの CodeQL で「Set up」→「Default」を選択
4. 「Enable CodeQL」をクリック

デフォルト設定で、PRの作成時とデフォルトブランチへのpush時に自動で解析が実行される。

## 実際に使ってみて

チームみらいのOSSにPRを出した際、upstreamにCodeRabbitが入っていたため、PR提出後に以下のような指摘をもらった。フォーク側でbot チェックをしておけば、upstreamに出す前に自分で潰せていた例だ。

**catchブロックのfail-open問題**

```typescript
// 指摘されたコード（fail-open：エラーでも処理続行）
try {
  await deleteShape(shapeId);
} catch (error) {
  console.error(error);
}
// この後に削除成功前提の処理が続いていた
```

```typescript
// 修正後（fail-closed：エラー時は安全側に倒す）
try {
  await deleteShape(shapeId);
} catch (error) {
  console.error(error);
  return; // ここで処理を止める
}
```

これはセキュリティに関わるパターンで、人間のレビューでも見落としやすい。botに機械的にチェックしてもらう価値がある。

## upstreamにbotが入っているプロジェクトでも意味はある？

ある。理由は2つ。

1. **upstreamのCIはPR提出後に動く**。指摘されてから直すより、事前に潰しておく方がメンテナーへの印象が良い
2. **フォーク内で試行錯誤できる**。upstreamのPRで「修正→re-review→修正→...」を繰り返すとコミット履歴が汚れる

## まとめ

| やること | 内容 |
| --- | --- |
| CodeRabbit導入 | GitHub Marketplaceで数クリック |
| CodeQL有効化 | フォークのSettingsで数クリック |

2つとも無料、クラウド実行（PCへの負荷なし）、設定ファイルの追加も不要。

自分のフォークに入れておけば、どのOSSプロジェクトに参加してもPR前にチェックできる体制が整う。
