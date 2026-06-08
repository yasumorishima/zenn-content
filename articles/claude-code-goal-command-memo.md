---
title: "Claude Code の /goal を使ってみる前に調べたことメモ"
emoji: "🎯"
type: "tech"
topics: ["claudecode", "claude", "ai", "codex"]
published: false
---

> 2026年6月時点の自分用メモ。Claude Code はほぼ毎日アップデートが入るので、実際に使うときは `claude --version` と `/release-notes` で最新を確認すること。

## 結論から

`/goal` は**公式 Claude Code に実在する組み込みコマンド**だった。2026年5月11日の **v2.1.139** で入ったやつ。やることはシンプルで、

> 「**完了条件を1行で宣言すると、その条件が満たされるまで Claude がターンをまたいで勝手に作業し続けてくれる**」

これだけ。普段なら1ターンごとに「続けて」と言わないといけないところを、終了条件を先に決めておけば放置できる、という話。

仕組みが面白くて、**作業するモデルと「終わったか判定するモデル」が別**になっている。各ターンが終わるたびに、別の小型で速いモデル（デフォルトは Haiku）が会話ログを読んで「条件を満たした? yes / no」を返す。`no` なら理由が次ターンの指示として注入されてもう一周、`yes` になったら goal が自動でクリアされる。

VentureBeat の記事で Sprinklr の人が言ってた一言が的を射てた（意訳）。「自分の宿題を自分で採点させちゃダメ。作業してるモデルこそ、終わったかどうかの判定者として最悪」。だから builder と judge を分けてる、と。

## 基本的な使い方

```
/goal <条件>          # ゴールを設定。即座に走り出す（「やって」は不要）
/goal                 # 引数なし。今の状態（条件・経過時間・ターン数・トークン・直近の判定理由）を表示
/goal clear           # 解除。stop / off / reset / none / cancel でも同じ
```

- 1セッションにつきゴールは1つだけ。新しい `/goal` は古いのを上書きする。
- 条件は最大4,000文字。
- `/clear` で会話をリセットすると goal も一緒に消える。
- インタラクティブ／`-p`（ヘッドレス）／デスクトップアプリ／リモートコントロールで使える。

### ヘッドレスで一発実行

```bash
claude -p "/goal CHANGELOG.md has an entry for every PR merged this week"
```

これだけで完走するまで走る。止めたければ Ctrl+C。cron や CI から叩くならこの形。

## 条件の書き方が9割

ここが一番大事そう。**「検証できる終了状態があるタスク」にだけ使う**。

- 良い例: `all tests in test/auth pass (npm test exits 0)`、`the module compiles and every call site is migrated`
- ダメな例: `make it production-ready`、`make the UI better` ← 主観的な条件はトークンを焼き続けて永遠に終わらない

自分用テンプレとして、こういう4要素で書くのが良さそう（公式が明言してるのは最初の3つ。4番目は「無人実行なら入れとけ」くらいのトーン）:

```
/goal implement dark mode matching the Figma spec,    # 1. 測定可能な終了状態
all tests pass (npm test exits 0),                     # 2. 検証方法
do not modify existing API contracts,                  # 3. 守るべき制約
or stop after 30 turns                                 # 4. ターン/時間の上限
```

## ハマりそうなポイント（先に知っておく）

- **トークン予算は内蔵されていない。** 曖昧な条件で一晩放置するとトークンを焼き尽くす事故報告あり。条件に `or stop after N turns` を必ず入れる。
- **判定モデルはログしか読まない。** ツールを自分で実行したりファイルを読んだりはしない。だから Claude が「テスト通ったと思う」とだけ書いて実際は走らせてなくても、judge は誤って `yes` を返しうる。→ 条件に「`npm test` を実行して exit 0 を確認」と書いて証拠を強制する。
- **pause / resume は無い。** OpenAI の Codex CLI 版 `/goal` には create/pause/resume/clear があるけど、Claude Code 版は「active → 達成で自動クリア or 手動 clear」だけ。混同しないこと。
- **複合目標に弱い。** 「認証作り直し＋OAuth追加＋テスト＋ドキュメント更新」みたいな盛り合わせは judge が処理しきれない。検証可能な end state ごとに分割する。
- **trust dialog 承認済みのワークスペースが必要。** `disableAllHooks` が設定されてると使えない（judge が hooks の仕組みを使ってるから）。

## 似てるけど違うやつとの整理

| 機能 | 何をする | `/goal` との関係 |
|---|---|---|
| `/plan`（plan mode） | 読み取り専用で計画を作るだけ。実装はしない | `/goal` は計画の後の「実行ループ」。代替ではなく相補 |
| `/loop` | 時間間隔で同じプロンプトを再実行（デプロイ監視など） | `/goal` は条件ベース。終わったら止まる用途はこっち |
| auto mode | ターン内のツール承認を自動許可するだけ | **auto mode + `/goal` が完全自走の定番セット** |
| Stop hook | 設定ファイルで永続定義する仕組み | `/goal` はそのセッション限定版（中身は prompt-based Stop hook） |
| Agent View（`claude agents`） | 複数バックグラウンドセッションの管理ダッシュボード | v2.1.139 で同時実装。`/bg` と組めば並列監視できる |

公式（Boris Cherny）の推奨ワークフローは「**Plan mode で計画を詰める → auto-accept に切り替えて実行**」の二段構え。`/goal` はこの実行段階に差し込むイメージ。

## 自分が試すときの段取り

1. **環境確認**: `claude --version` で v2.1.139 以上か。古ければ `claude update`。対象ディレクトリで一度起動して trust を承認。
2. **小さく試す**: 30分以内に終わる検証タスクで。先に `git status` クリーン＆現状のテスト失敗数を控える。
   ```
   /goal all tests in test/auth pass (npm test exits 0),
   do not modify files outside test/auth or src/auth,
   do not delete or skip tests,
   or stop after 15 turns
   ```
3. **auto mode と組む**: `Shift+Tab` で permission mode を auto まで回す。これで承認クリックも「続けて」も不要になる。
4. **監視運用**: `/bg` でバックグラウンド化 → `claude agents` で一覧監視。進捗は引数なしの `/goal` で judge の直近理由を読む。暴走したら `/goal clear`、ダメなら Ctrl+C。
5. **夜間バッチ**: `claude -p "/goal <ターン上限付き条件>"` を cron / GitHub Actions から。

**やめ時の目安**:
- judge の理由が3ターン連続で同じ → ループってる。手動介入。
- トークンが想定の2倍超え → 条件が曖昧。clear して書き直す。
- 条件に「production-ready」「looks good」みたいな主観語が入ってる → そもそも `/goal` 向きじゃない。

## 業界の文脈（メモ）

実は **OpenAI の Codex CLI のほうが11日早い**。Codex CLI v0.128.0（2026年4月30日）で `/goal` が入ってて、Claude Code の v2.1.139 が5月11日。両社とも「コーディングエージェントの長時間自走」を標準機能にしてきた、という流れ。Anthropic の差別化点は機能の先行ではなく、builder/judge 分離の実装の洗練度、という評価だった。

---

### 一次ソース
- 公式コマンド解説: `https://code.claude.com/docs/en/goal`（"Keep Claude working toward a goal"）
- コマンド一覧: `https://code.claude.com/docs/en/commands`
- CHANGELOG: `https://code.claude.com/docs/en/changelog`（v2.1.139 のエントリ）

### 二次ソース（運用Tips系はここ由来）
- VentureBeat "Claude Code's '/goals' separates the agent that works from the one that decides it's done"
- Codex CLI 側: `https://github.com/openai/codex/releases/tag/rust-v0.128.0`

> ⚠️ メモの注意書き: 4要素テンプレ・「3ターン連続同一理由」の閾値・トークン焼き切り事例などはコミュニティのブログ／Medium 由来の二次情報。公式が明言してるのは「測定可能な終了状態 / 検証方法 / 守るべき制約」の3要素まで。
