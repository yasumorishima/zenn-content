---
title: "Claude Code on Windowsで PowerShell ツールを有効にする"
emoji: "💻"
type: "tech"
topics: ["claudecode", "powershell", "windows"]
published: false
---

## 結論

Claude Code v2.1.84（2026年3月）から、Windows環境でネイティブPowerShellツールが使えるようになりました。

`C:\Users\<ユーザー名>\.claude\settings.json` に1行追加するだけです。

```json
{
  "env": {
    "CLAUDE_CODE_USE_POWERSHELL_TOOL": "1"
  }
}
```

次回起動時から有効になります。

---

## なぜ必要か

Claude CodeはWindowsでもGit Bash経由でコマンドを実行します。普段の `git` や `gh` では問題になりませんが、以下の場面でズレが出ます。

- **パス変換**: `C:\Users\...` が `/c/Users/...` に変換される。ツール間でパスを受け渡すとき齟齬が起きることがある
- **Windowsネイティブ操作**: レジストリ（`HKLM:\...`）やサービス管理などはBashから直接扱えない
- **PowerShell cmdlet**: `Get-ChildItem`, `Get-Process` 等をそのまま使いたい場面

PowerShellツールを有効にすると、Git Bashを経由せずにPowerShellで直接コマンドが実行されます。

---

## 仕組み

### シェルの自動検出

PowerShellツール有効時、Claude Codeは以下の順で検出します。

1. `pwsh.exe`（PowerShell 7+）
2. `powershell.exe`（Windows PowerShell 5.1）

PowerShell 7がインストールされていればそちらが優先されます。5.1しかなくても動きます。

### BashツールとPowerShellツールの違い

| 項目 | Bash | PowerShell |
|------|------|------------|
| パス | POSIX形式（`/c/Users/...`） | Windows形式（`C:\Users\...`） |
| パイプ | テキストベース | オブジェクトベース |
| 対応OS | macOS / Linux / Windows(Git Bash) | Windows のみ |
| プロファイル読み込み | あり | **なし** |
| サンドボックス | WSL 2で対応 | **未対応** |
| autoモード | 対応 | **未対応** |

有効にしても**Bashツールは残ります**。両方が登録された状態になり、コマンドに応じてClaude Codeが選択します。

---

## 注意点

### 現時点のステータス: opt-in プレビュー

2026年4月時点でまだプレビューです。以下の制限があります。

- **autoモード未対応**: PowerShellコマンドは自動承認されません。実行時に確認プロンプトが出ます
- **プロファイル未読み込み**: `$PROFILE` で設定しているエイリアスや関数は使えません
- **サンドボックス未対応**: Bashツール（WSL 2環境）にあるサンドボックス機能はPowerShellツールにはありません

### Git Bashは引き続き必要

PowerShellツールは **Bashの代替ではなく補完** です。Claude Code自体の起動にはGit Bashが必要なので、アンインストールしないでください。

Git Bashの自動検出に失敗する場合は、明示的にパスを指定できます。

```json
{
  "env": {
    "CLAUDE_CODE_GIT_BASH_PATH": "C:\\Program Files\\Git\\bin\\bash.exe"
  }
}
```

### セキュリティ関連の修正（v2.1.89）

v2.1.89（2026年4月）で以下のセキュリティ修正が入っています。

- 末尾 `&` によるバックグラウンドジョブバイパスの防止
- `-ErrorAction Break` によるデバッガハングの防止
- PowerShell 5.1でのクォート+空白を含む引数の扱い改善

PowerShellツールを使う場合は、最新版に更新しておくのがおすすめです。

---

## hooksでPowerShellを使う

`settings.json` のhooksでもPowerShellを指定できます。

```json
{
  "hooks": {
    "PreToolUse": [{
      "hooks": [{
        "shell": "powershell",
        "type": "command",
        "command": "Get-Process | Where-Object {$_.Name -like 'node*'}"
      }]
    }]
  }
}
```

---

## まとめ

- `CLAUDE_CODE_USE_POWERSHELL_TOOL=1` を `settings.json` に追加するだけ
- Windowsパスの変換問題が減り、cmdletも直接使える
- まだプレビュー（autoモード・サンドボックス未対応）
- Git Bashは引き続き必要

Windows環境でClaude Codeを使っている方は試してみてください。
