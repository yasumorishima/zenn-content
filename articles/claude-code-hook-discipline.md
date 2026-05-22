---
title: "Claude Code hook で AI coding assistant の規律を補強する — 個人運用での設計パターン参考"
emoji: "🪝"
type: "tech"
topics: ["claude", "claudecode", "ai", "workflow", "shell"]
published: false
---

## はじめに

LLM coding assistant (Claude Code 等) に dev workflow を任せていると、 memory file の「注意書き」 だけだと同じハマりを繰り返すと感じることがあります。

Claude Code には **Stop / PreToolUse / PostToolUse / PostToolUseFailure** の hook 機構があり、 これを使うと「LLM が忘れても hook が exit 2 で block + stderr message を流し込む」 という形で規律を補強できます。

本稿は私が個人で運用している hook の **設計パターン + 一般的に有用そうな例** をまとめたものです。 環境や好みで合う合わないがあるので、 **あくまで参考レベル**として読んでください。

## 私が memory rule だけだと不十分だと感じた経緯

LLM assistant に build / deploy / git push 等を任せると、 こういう anti-pattern が起きました:

- `git push` した直後に end-turn して deploy fail に気付かない
- Tool が internal error で fail しても silent retry してしまう
- Bash command を `&&` で連結すると permission prompt 暴発で silent-stall するのに、 つい連結してしまう
- secret token を含む値を cmd に書き込んでしまう

memory file の「注意書き」 を入れても、 別 turn / 別 session で普通に忘れる感覚がありました。 LLM は context window と注意のばらつきで規律を維持しにくいので、 強制可能なものは hook で補強した方が私は楽だと感じています。

## Claude Code hook の四層

| event | timing | 用途 |
|---|---|---|
| **PreToolUse** | tool 実行前 | 危険 cmd の予防 |
| **PostToolUse** | tool 実行後 (成功) | 出力の後処理・notification |
| **PostToolUseFailure** | tool 実行後 (失敗) | tool error の silent retry 防止 |
| **Stop** | turn 終了前 | end-turn 規律の最終 check |

私が便宜的に分けている役割:

1. **PreToolUse = 火事を起こす前に止める** (予防)
2. **PostToolUseFailure = 火事の煙を user から隠さない** (transparency)
3. **Stop = 火事の鎮火を確認してから帰る** (closure)

## 実例 (一般的に有用そうな 7 個)

### PreToolUse 系

**1. Bash 連結 cmd 警告** — `&&` / `;` / `||` の連結を block。 連結すると permission prompt 暴発で silent-stall が起きやすいので私は分けるようにしました。

```bash
#!/usr/bin/env bash
input=$(cat)
cmd=$(printf '%s' "$input" | jq -r '.tool_input.command // ""')

if echo "$cmd" | grep -qE ' (&&|;|\|\|) '; then
    echo "[hook:no-bash-compound] cmd 連結は分けて実行を推奨" >&2
    exit 2
fi
exit 0
```

**2. 長時間 cmd の timeout 明示** — `npm run build` / `git clone` 等で timeout 指定が無い場合 block。 silent-stall 予防。

```bash
cmd=$(printf '%s' "$input" | jq -r '.tool_input.command // ""')
timeout_ms=$(printf '%s' "$input" | jq -r '.tool_input.timeout // 0')

if echo "$cmd" | grep -qE 'npm (run|install|ci|build)|git clone|pip install'; then
    if [ "$timeout_ms" = "0" ] || [ "$timeout_ms" = "null" ]; then
        echo "[hook:require-timeout] 長時間 cmd は timeout 明示推奨 (例: 600000ms = 10 分)" >&2
        exit 2
    fi
fi
exit 0
```

**3. secret token literal の cmd 書き込み** — `sk-`, `gho_`, `AKIA`, JWT などのパターンを cmd 内で検出 → block。 env variable 経由を推奨。

```bash
cmd=$(printf '%s' "$input" | jq -r '.tool_input.command // ""')

if echo "$cmd" | grep -qE '\b(sk-[a-zA-Z0-9]{20,}|gho_[a-zA-Z0-9]{20,}|AKIA[A-Z0-9]{16}|eyJ[a-zA-Z0-9_-]+\.[a-zA-Z0-9_-]+\.[a-zA-Z0-9_-]+)\b'; then
    echo "[hook:no-token-literal] secret literal を cmd に直書きは避ける、 env variable 推奨" >&2
    exit 2
fi
exit 0
```

**4. 巨大 file の offset 無し Read** — 1MB 超 file を `offset` / `limit` 無しで Read tool に渡すと context が bloat するため block。

```bash
file_path=$(printf '%s' "$input" | jq -r '.tool_input.file_path // ""')
offset=$(printf '%s' "$input" | jq -r '.tool_input.offset // 0')

if [ -f "$file_path" ]; then
    size=$(stat -c%s "$file_path" 2>/dev/null || echo 0)
    if [ "$size" -gt 1048576 ] && [ "$offset" = "0" ]; then
        echo "[hook:no-huge-read] 1MB 超 file は offset+limit 指定推奨" >&2
        exit 2
    fi
fi
exit 0
```

### PostToolUseFailure 系

**5. Tool failure を surface** — tool が runtime error で fail した直後、 silent retry を block。

```bash
echo "[hook:tool-failure] Tool failed at runtime." >&2
echo "Report the failure and propose a retry path before continuing." >&2
exit 2
```

### Stop 系

**6. Unhandled tool error で end-turn block** — transcript の tail から直近の tool error envelope を検出、 後続 assistant text に retry/error/失敗 vocabulary が無ければ block。

```bash
#!/usr/bin/env bash
input=$(cat)
transcript=$(printf '%s' "$input" | jq -r '.transcript_path // ""')
[ -z "$transcript" ] && exit 0
[ ! -f "$transcript" ] && exit 0

tail_json=$(tail -100 "$transcript")

err_idx=$(printf '%s\n' "$tail_json" | awk '
  /"type":"assistant"/ { next }
  /\[hook:/ { next }
  /Tool result missing/ ||
  /MCP error/ ||
  /TimeoutError/ ||
  /ECONNRESET/ { last=NR }
  END { if (last) print last }
')

[ -z "$err_idx" ] && exit 0

later=$(printf '%s\n' "$tail_json" | tail -n +"$((err_idx + 1))")

if ! printf '%s' "$later" | grep -qiE 'retry|失敗|エラー|error|timeout|タイムアウト'; then
    echo "[hook:unhandled-tool-error] Tool error 後に ack 無しで end-turn しないこと" >&2
    exit 2
fi
exit 0
```

**7. push 後の deploy polling** — `git push origin main` 検出後、 後続 assistant text に deploy polling vocabulary が無ければ block。 これは私が今回作って一番効いた hook です。

```bash
tail_json=$(tail -200 "$transcript")

push_idx=$(printf '%s\n' "$tail_json" | awk '
  /"type":"assistant"/ { next }
  /\[hook:/ { next }
  /main -> main/ { last=NR }
  END { if (last) print last }
')
[ -z "$push_idx" ] && exit 0

later=$(printf '%s\n' "$tail_json" | tail -n +"$((push_idx + 1))")

if ! printf '%s' "$later" | grep -qiE 'deploy[_ ]?state|polling|=success|=failure|gh api.*deployments'; then
    echo "[hook:deploy-not-polled] push 後 deploy state polling 推奨" >&2
    echo "対処: gh api deployments?sha=<full> --jq .[0].id → deployments/<id>/statuses --jq .[0].state" >&2
    exit 2
fi
exit 0
```

## 設計パターン

### stdin で context 受け取り

Claude Code hook は stdin に JSON を渡してきます。 主要 field:

- `.tool_input.command` (Bash tool の場合)
- `.tool_input.file_path` (Read/Write/Edit の場合)
- `.transcript_path` (Stop hook の場合、 当 session の JSONL transcript path)

```bash
input=$(cat)
cmd=$(printf '%s' "$input" | jq -r '.tool_input.command // ""')
```

### transcript scan で文脈確認

Stop hook では `transcript_path` の JSONL を tail scan して直近の context を確認できます:

```bash
tail_json=$(tail -200 "$transcript")
```

assistant 自身の説明文を scan から除外しないと、 誤検出します (assistant が「git push しました」 と書いた text を「push が行われた」 と読んで block してしまう):

```bash
awk '
  /"type":"assistant"/ { next }
  /\[hook:/ { next }
  /main -> main/ { last=NR }
  END { if (last) print last }
'
```

### exit code 規約

- `exit 0` = passthrough (規律違反なし)
- `exit 2` = block + stderr が LLM の次 turn input に reminder として挿入される

stderr に書く message は LLM が次 turn で読むので、 **「何が違反か + 対処方法」 を 2-3 行で書く** と効きやすい印象です。

### settings.json への登録

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {"type": "command", "command": "bash '~/.claude/hooks/no-bash-compound.sh'"},
          {"type": "command", "command": "bash '~/.claude/hooks/no-token-literal.sh'"}
        ]
      },
      {
        "matcher": "Read",
        "hooks": [
          {"type": "command", "command": "bash '~/.claude/hooks/no-huge-read.sh'"}
        ]
      }
    ],
    "Stop": [
      {
        "matcher": "*",
        "hooks": [
          {"type": "command", "command": "bash '~/.claude/hooks/unhandled-tool-error.sh'"},
          {"type": "command", "command": "bash '~/.claude/hooks/deploy-not-polled.sh'"}
        ]
      }
    ]
  }
}
```

## 私の運用 = 漸進的に拡張

最初から全部揃える必要はないと感じていて、 ハマるたびに 1 個追加しています。 判断の目安:

- **memory rule で済む**: context-dependent な judgement (revert すべきか、 commit を細粒化すべきか等)
- **hook 化したい**: 強制可能な pattern (連結 cmd、 token literal、 push 後 polling 等)

「強制可能な規律は強制する、 memory rule で誤魔化さない」 を私は自分の指針にしていますが、 環境によって判断は変わると思います。

## まとめ

memory rule (注意書き) だけだと私は規律を維持しにくく、 強制可能な規律は Claude Code hook で補強しています:

- **PreToolUse**: 危険 cmd を起こす前に止める
- **PostToolUseFailure**: tool error を user から隠さない
- **Stop**: end-turn 前の規律 violation を最終 check

hook 開発は漸進的でいいと感じています。 memory rule + hook の二段階設計が私には合っていました。

**あくまで個人の運用例**ですので、 参考になりそうな部分だけ拾っていただければ。
