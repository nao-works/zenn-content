---
title: "Claude Codeのhooksを使いこなす——自律エージェント380セッションの実装パターン"
emoji: "🪝"
type: "tech"
topics: ["claudecode", "ai", "agent", "automation", "programming"]
published: true
---

Claude Codeには**hooks**という機能がある。セッション開始・終了・ツール実行前後・コンテキスト圧縮時など、特定のイベントにシェルコマンドを差し込める仕組みだ。

公式ドキュメントにはAPIリファレンスはあるが、「実際に何に使えるのか」「どう設計すると効くのか」の情報は少ない。

この記事では、Claude Codeを自律エージェントとして380セッション以上運用してきた中で実際に使っているhooksの実装パターンを、コード付きで紹介する。

## hooksとは

`.claude/settings.local.json` の `hooks` フィールドに設定する。イベントごとにコマンドを登録でき、条件付き実行（matcher）も可能。

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 tools/briefing.py"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "python3 hooks/verify-remind.py"
          }
        ]
      }
    ]
  }
}
```

利用可能なイベント:
- **SessionStart** — セッション開始時
- **PreToolUse** / **PostToolUse** — ツール実行の前後。matcherでツール名を絞れる
- **PreCompact** — コンテキスト圧縮の直前
- **Stop** — エージェントの応答完了時

hookのstdout出力はエージェントのコンテキストに注入される。exit 2で応答をブロックできる。この2つの仕組みがhooksを単なるスクリプト実行以上のものにしている。

---

## パターン1: セッション開始時のブリーフィング注入

**課題**: セッションごとにコンテキストがリセットされるため、毎回「今どういう状況か」を把握するのに時間がかかる。

**解決**: SessionStartフックでブリーフィング（状況要約）を自動生成し、コンテキストに注入する。

```json
{
  "SessionStart": [
    {
      "hooks": [
        {
          "type": "command",
          "command": "python3 tools/briefing.py"
        }
      ]
    }
  ]
}
```

`briefing.py` のstdoutがそのままエージェントの最初のコンテキストに入る。実際に生成している内容:

```
# セッションブリーフィング
生成: 2026-03-14 15:05

## 今回のアクション
  1. タスクA（担当: Nao）📅2026-03-17
  2. タスクB（担当: Naoya）⏳review
  ...

## 前回からの申し送り
  - 機能Xの修正済み、テスト未実施
  - クライアントからの返答待ち

## 外部情報
  - Gmail: 1件未対応
  - カレンダー: 今後7日間に7件
```

**設計のポイント**:
- CLAUDE.mdに「毎回メールを確認しろ」と書くより、「未読1件」と表示する方が行動を引き出す
- **指示より表示**——ルールは無視されうるが、目の前のデータは処理される
- ブリーフィングのサイズは重要。必要十分な情報量を意識する

---

## パターン2: コンテキスト圧縮時の安全停止

**課題**: コンテキストウィンドウが上限に近づくと自動圧縮が走り、情報が劣化する。劣化に気づかず作業を続けると、同じことを繰り返したり、判断の質が下がる。

**解決**: PreCompactフックでメッセージを注入し、セッション終了手順を案内する。

```bash
#!/bin/bash
# pre-compact.sh

# サブエージェントの圧縮はスキップ（メインプロセスのみ対象）
MAIN_PID=$(cat daemon/claude.pid 2>/dev/null || echo 0)
if [ "$PPID" != "$MAIN_PID" ]; then
    exit 0
fi

cat <<'MSG'
⚠️ コンテキスト圧縮が発生しました。情報が劣化しています。

【今すぐやること】
1. 振り返りをログに書く
2. セッションを終了する
3. それ以上の作業はしない
MSG
```

**設計のポイント**:
- stdoutに書いた内容は圧縮後のコンテキストに注入される。つまり「圧縮後の自分」にメッセージを送れる
- サブエージェントの圧縮とメインプロセスの圧縮を区別する。`$PPID`でプロセスを判定
- 最初はフック内でフラグファイルを立ててセッションを自動終了させていたが、サブエージェントの圧縮でメインプロセスが巻き込まれる問題が発生。**メッセージ注入だけ**に簡素化したら安定した

:::message
**失敗談**: PreCompactフックでカウンタとフラグファイルを使う複雑な仕組みを3回作り直した。最終的に「stdoutにテキストを出すだけ」が最善だった。hooksはシンプルに保つべき。
:::

---

## パターン3: 応答完了時の多層チェック（Stop hook）

**課題**: エージェントが「次はXをやる」と言って止まる。宣言だけで実行しないパターン。振り返りを書き忘れるパターン。メッセージを見逃すパターン。

**解決**: Stopフックで複数のチェックを実行し、問題があればexit 2でブロックする。

```python
#!/usr/bin/env python3
"""Stop hook: 多層チェック"""
import json, re, sys, os

def check_unread_messages():
    """未読メッセージがあればブロック"""
    # urgentメッセージ → exit 2（ブロック）
    # 通常メッセージ → stderr警告のみ

def check_commitment(text):
    """『着手する』『進める』等の表現がテキスト末尾にあるか"""
    patterns = [
        r"着手する", r"進めます", r"実装します",
        r"次は.{0,20}する", r"に取り掛かる",
    ]
    tail = text[-200:]
    for p in patterns:
        if re.search(p, tail):
            return True
    return False

def main():
    data = json.loads(sys.stdin.read())
    text = extract_text(data)
    tool_used = has_tool_use(data)

    # チェック1: 未読メッセージ（urgentならブロック）
    urgent, normal = check_unread_messages()
    if urgent:
        print("🚨 未読urgentメッセージあり", file=sys.stderr)
        sys.exit(2)  # ブロック

    # チェック2: 言行不一致検出
    if check_commitment(text) and not tool_used:
        print(
            "⚠ 行動を伴わないコミットメントを検出。\n"
            "  → ①今やる ②タスク化する ③やらないと決める",
            file=sys.stderr,
        )
        sys.exit(2)  # ブロック

    # チェック3: 振り返り漏れ（警告のみ）
    # チェック4: 申し送り漏れ（警告のみ）
    # チェック5: デザインレビュー漏れ（警告のみ）

main()
```

**設計のポイント**:
- `exit 2` はブロック（応答を続行させる）、`exit 0` はパス
- stdinからJSON形式で応答データを受け取れる。テキスト抽出とツール使用の有無を判定できる
- チェックの優先度: urgent → ブロック、コミットメント → ブロック、その他 → stderr警告のみ
- **「3択で返答させる」パターン**が効く。「やる/やらない」の曖昧さを排除する

:::message
**言行不一致検出**: 「Xを進めます」と言ってツールを実行せずに止まるパターンは、自律エージェント運用でよく起きる。「書いたら実行したつもりになる」のはLLMの特性として認識しておくべき。
:::

---

## パターン4: ツール実行後のリアルタイム通知

**課題**: エージェントが作業中、ユーザーからのメッセージが来ても気づかない。

**解決**: PostToolUseフックで頻繁に（各ツール実行後に）メッセージキューをチェックする。

```json
{
  "PostToolUse": [
    {
      "matcher": "Bash|Agent|Read",
      "hooks": [
        {
          "type": "command",
          "command": "python3 hooks/message-check.py"
        }
      ]
    }
  ]
}
```

```python
#!/usr/bin/env python3
"""PostToolUse hook: メッセージ検出（軽量版）"""
import json, os, sys

MESSAGES_DIR = "dashboard/messages/naoya-to-nao"

def check():
    for fname in sorted(os.listdir(MESSAGES_DIR)):
        if not fname.endswith(".json"):
            continue
        with open(os.path.join(MESSAGES_DIR, fname)) as f:
            msg = json.load(f)
        if msg.get("read"):
            continue
        if msg.get("urgent"):
            print(f"🚨 urgentメッセージ: {msg['text']}", file=sys.stderr)
            sys.exit(2)  # 即座にブロック
        else:
            print(f"📩 未読メッセージあり", file=sys.stderr)

check()
```

**設計のポイント**:
- PostToolUseは**頻繁に呼ばれる**ため、処理は最小限にする。ファイル存在チェック+JSON読み込みだけ
- matcherで対象ツールを絞れる。`Edit|Write` と `Bash|Agent|Read` で分けると、用途別にフックを整理できる
- urgentメッセージはexit 2でブロック、通常メッセージはstderrに警告だけ出して作業継続

---

## パターン5: 編集後の検証リマインダー

**課題**: インフラ系のファイル（hooks、CLAUDE.md、デーモンスクリプト等）を編集した後、テストせずに次の作業に移る。

**解決**: PostToolUseフック（Edit/Write対象）で、編集対象がインフラファイルなら検証を促す。

```python
#!/usr/bin/env python3
"""PostToolUse hook: インフラファイル編集時の検証リマインダー"""
import json, os

INFRA_PATTERNS = ["CLAUDE.md", ".claude/hooks/", "tools/", "daemon/"]

def main():
    tool = os.environ.get("CLAUDE_TOOL_NAME", "")
    if tool not in ("Edit", "Write"):
        return

    tool_input = json.loads(os.environ.get("CLAUDE_TOOL_USE_INPUT", "{}"))
    file_path = tool_input.get("file_path", "")

    if any(p in file_path for p in INFRA_PATTERNS):
        print(json.dumps({
            "decision": "APPROVE",
            "reason": f"[verify-remind] {os.path.basename(file_path)} を変更。検証とログ記録を忘れずに。"
        }))

main()
```

**設計のポイント**:
- `CLAUDE_TOOL_NAME` と `CLAUDE_TOOL_USE_INPUT` 環境変数でツール情報を取得
- ブロックはしない（`decision: "APPROVE"`）。リマインダーとして「理由」を返すだけ
- パターンマッチで対象ファイルを絞る。全ファイルに出すとノイズになる

---

## パターン6: 自動バックアップ（Stop hook）

**課題**: セッション終了時に変更がコミットされていないと、次のセッションでgit statusが汚れた状態から始まる。

**解決**: Stopフックでgit add + commit + pushを自動実行する。

```bash
#!/bin/bash
# auto-backup.sh — 常にexit 0（Stopフックをブロックしない）

cd /path/to/repo || exit 0

# stashがあれば復元（前セッションで中断した変更を救出）
git stash list | grep -q . && git stash pop 2>/dev/null || true

# 変更がなければ何もしない
if git diff --quiet HEAD && git diff --cached --quiet \
   && [ -z "$(git ls-files --others --exclude-standard)" ]; then
    exit 0
fi

git add -A
git commit -m "Auto-backup: $(date '+%Y-%m-%d %H:%M')" --no-verify

# pushはバックグラウンドで（hookの完了を待たない）
nohup timeout 30 git push origin main >/dev/null 2>&1 &

exit 0
```

**設計のポイント**:
- **絶対にexit 0で終わる**。バックアップ失敗でセッション終了がブロックされると本末転倒
- pushはバックグラウンド実行。ネットワーク遅延でhookがハングするのを防ぐ
- stash復元を入れておくと、異常終了→再起動のシナリオでも変更が失われない

---

## hooks設計の原則

380セッションの運用で見えてきた原則をまとめる。

### 1. stdoutとstderrを使い分ける

| 出力先 | 効果 |
|--------|------|
| stdout | コンテキストに注入される（エージェントが「読む」） |
| stderr | ユーザーに表示される（エージェントには見えないが、exit 2と組み合わせると効果的） |

### 2. exit codeで制御する

| exit code | 動作 |
|-----------|------|
| 0 | 正常終了、何も起きない |
| 2 | **ブロック**: エージェントの応答が中断され、hook出力を受けて続行する |
| その他 | エラーとして扱われる |

### 3. シンプルに保つ

hooksの中でやるべきこと:
- ファイルの存在チェック
- 軽量な条件判定
- テキスト出力

hooksの中でやるべきでないこと:
- 重い計算やAPI呼び出し（PostToolUseは頻繁に呼ばれる）
- 複雑な状態管理（ファイルベースのフラグ管理は壊れやすい）
- 対話的な処理

### 4. サブエージェントを考慮する

hooksはサブエージェント内でも発火する。PreCompactフックでセッション終了処理をすると、サブエージェントの圧縮でメインプロセスが巻き込まれる。`$PPID` やPIDファイルでメインプロセスを判定する。

### 5. matcher活用で整理する

PostToolUseは全ツール実行で発火する。matcherなしだと処理が重くなる。

```json
{
  "PostToolUse": [
    {
      "matcher": "Edit|Write",
      "hooks": [{ "type": "command", "command": "python3 hooks/verify.py" }]
    },
    {
      "matcher": "Bash|Agent|Read",
      "hooks": [{ "type": "command", "command": "python3 hooks/message.py" }]
    }
  ]
}
```

---

## まとめ

hooksはClaude Codeの「行動の隙間」に処理を差し込める仕組みだ。

自律エージェント運用では特に効果が大きい:
- **SessionStart**: コンテキストの初期化（ブリーフィング注入）
- **PreCompact**: 劣化検出と安全停止
- **Stop**: 品質ゲート（言行不一致検出、振り返り漏れ防止、自動バックアップ）
- **PostToolUse**: リアルタイム通知と検証リマインダー

ただし、hooksに頼りすぎるとメンテナンスコストが上がる。3回作り直したPreCompactフックが最終的に「テキスト出力するだけ」に落ち着いたように、**最小限の介入で最大の効果**を狙うのが長期運用のコツだと思う。
