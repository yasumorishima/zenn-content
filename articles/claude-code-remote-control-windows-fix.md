---
title: "Claude Code remote-control が Windows で Workspace not trusted になる対処法"
emoji: "📱"
type: "tech"
topics: ["claudecode", "claude", "windows", "ai"]
published: false
---

## はじめに

2026年2月25日にリリースされた Claude Code の **Remote Control** 機能。スマホやブラウザからローカルの Claude Code セッションを操作できる便利な機能です。

早速試してみたところ、Windows 環境で以下のエラーが出ました。

```
Error: Workspace not trusted. Please run `claude` in C:\Users\<username> first to review and accept the workspace trust dialog.
```

「普通に `claude` は使えているのに…」という状態。原因と解決法をまとめます。

## 前提条件

Remote Control は **Max プランが必要**です（Pro プランは近日対応予定）。また、`claude auth status` で `claude.ai` 経由でのログインが確認できることも確認してください。

## 原因

`~/.claude.json` の中に、プロジェクトごとの信頼フラグが記録されています。

```json
"C:/Users/<username>": {
  ...
  "hasTrustDialogAccepted": false,
  ...
}
```

`hasTrustDialogAccepted` が `false` のまま。通常の `claude` コマンドはこのフラグをチェックしませんが、`claude remote-control` は**起動前にこのフラグを厳密に確認**します。何百回も `claude` を使っていても、このフラグが `false` だとエラーになります。

## 対処法

`~/.claude.json` を直接編集します。

### 注意：パスが2種類ある

Windows 環境では `.claude.json` 内に同じディレクトリでもパスが2種類記録されている場合があります。

- バックスラッシュ版: `"C:\\Users\\<username>"`
- スラッシュ版: `"C:/Users/<username>"`

`claude remote-control` が実際に参照するのは**スラッシュ版**です。両方を直しておくのが安全です。

### 手順

1. `C:\Users\<username>\.claude.json` をテキストエディタで開く
2. `"C:\\Users\\<username>"` と `"C:/Users/<username>"` の2エントリを探す
3. それぞれの `"hasTrustDialogAccepted": false` を `"hasTrustDialogAccepted": true` に変更
4. 保存

**変更前:**

```json
"C:/Users/<username>": {
  "allowedTools": [],
  "hasTrustDialogAccepted": false,
  ...
}
```

**変更後:**

```json
"C:/Users/<username>": {
  "allowedTools": [],
  "hasTrustDialogAccepted": true,
  ...
}
```

## 確認

保存後、ターミナルで再実行します。

```bash
claude remote-control
```

以下のように表示されれば成功です。

```
·✔︎· Connected · <username> · HEAD

Continue coding in the Claude app or https://claude.ai/code/session_...
space to show QR code
```

スペースキーでQRコードが表示されますが、Windows CMD では効かない場合があります。その場合は表示されているURLをスマホのブラウザで直接開いてください。

## まとめ

| 項目 | 内容 |
|------|------|
| エラー原因 | `.claude.json` の `hasTrustDialogAccepted` が `false` |
| 確認すべきパス | `C:/Users/<username>`（スラッシュ版）|
| 修正方法 | 直接 `true` に書き換え |

Remote Control は PC のターミナルを起動したままスマホから Claude Code を操作できる機能です。Windows ユーザーでこのエラーに遭遇した方の参考になれば幸いです。
