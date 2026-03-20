---
title: "Claude CodeのSkill使用統計をhooksで自動収集する"
emoji: "📊"
type: "tech"
topics: ["claudecode", "claude", "ai", "poem"]
published: false
---

![header](/images/articles/claudecode-skill-usage-tracking/header.jpg)

## はじめに

Claude Code に登録した Skill が増えてきたとき、「どのSkillを一番使っているんだろう？」と気になりました。

スモルには現在、`zenn-article`・`note-article`・`add-task`・`idea-scout` など10本近くのSkillが登録されています。育てていくうちに「どれが実際に役立っているか」「使われていないSkillは整理すべきか」を判断したくなりました。でも感覚だけでは正確にわからない。そこで、Skillの使用統計を自動収集する仕組みを作ることにしました。

## まずセッションログを調べてみた

Claude Code は `~/.claude/projects/` 以下にセッションログを JSONL 形式で保存しています。

```
~/.claude/projects/-home-sumomo-sumoru/
  ├── abc123.jsonl
  ├── def456.jsonl
  └── ...
```

各行には `tool_use` タイプのエントリが含まれており、どのツールが呼ばれたかが記録されています。Skillの呼び出しは `mcp__` プレフィックスが付くので、これを集計すればいけるかと思いました。

```bash
# jsonlからtool_use行を抽出してカウントする試み
cat ~/.claude/projects/-home-sumomo-sumoru/*.jsonl \
  | jq -r 'select(.type == "tool_use") | .name' \
  | grep 'mcp__' \
  | sort | uniq -c | sort -rn
```

一見うまくいくように見えましたが、問題がありました。

## ログから集計するのは精度が低い問題

セッションログには**すべてのやりとり**が入っています。つまり：

- 同じSkillを1回の会話で複数回呼ぶと、ログには全呼び出し分が記録される
- 逆に、古いセッションファイルは定期的にローテーションされて消える
- ログファイルの構造がAPIバージョンによって微妙に変わる可能性がある

「たぶん合ってるっぽい数字」は出せますが、信頼できる統計とは言えません。ログの事後集計より、**呼ばれた瞬間に記録する**方が正確です。

## 解決策：PostToolUseフックをsettings.jsonに追加

Claude Code の `settings.json` には `hooks` という設定項目があります。ツールが実行された後に任意のコマンドを走らせる `PostToolUse` フックを使えば、Skill呼び出しをリアルタイムで記録できます。

`~/.claude/settings.json` に以下を追加しました：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "mcp__",
        "hooks": [
          {
            "type": "command",
            "command": "python3 /home/sumomo/sumoru/scripts/log_skill_usage.py"
          }
        ]
      }
    ]
  }
}
```

`matcher` に `mcp__` を指定することで、MCP経由のツール（＝Skill）が呼ばれたときだけフックが起動します。

## フックの実装コード

フックは stdin からツール実行の情報を JSON で受け取ります。`log_skill_usage.py` の中身はこちらです：

```python
#!/usr/bin/env python3
import json
import sys
import os
from datetime import datetime

LOG_PATH = "/home/sumomo/sumoru/logs/skill_usage.jsonl"

def main():
    try:
        data = json.load(sys.stdin)
    except Exception:
        sys.exit(0)

    tool_name = data.get("tool_name", "")
    if not tool_name.startswith("mcp__"):
        sys.exit(0)

    # mcp__plugin_name__skill_name → skill_name を抽出
    parts = tool_name.split("__")
    skill_name = parts[-1] if len(parts) >= 3 else tool_name

    record = {
        "timestamp": datetime.now().isoformat(),
        "tool_name": tool_name,
        "skill_name": skill_name,
        "session_id": data.get("session_id", ""),
    }

    os.makedirs(os.path.dirname(LOG_PATH), exist_ok=True)
    with open(LOG_PATH, "a") as f:
        f.write(json.dumps(record, ensure_ascii=False) + "\n")

if __name__ == "__main__":
    main()
```

ポイントは2点です：

1. **フックが失敗してもClaude Codeの動作を止めない**よう、例外は握りつぶして `sys.exit(0)` で終了する
2. **スキル名だけ抽出**：`mcp__sumoru-skills__zenn-article` なら `zenn-article` を記録する

## skill-statsスキルの作成

ログを溜めても見られなければ意味がないので、`skill-stats` というSkillも作りました。スモルに「スキルの使用統計を見せて」と言うと、ログを集計してランキング表示します。

```python
#!/usr/bin/env python3
# skills/skill-stats/run.py
import json
from collections import Counter
from pathlib import Path

LOG_PATH = Path("/home/sumomo/sumoru/logs/skill_usage.jsonl")

def main():
    if not LOG_PATH.exists():
        print("ログファイルがまだありません。")
        return

    records = []
    with open(LOG_PATH) as f:
        for line in f:
            try:
                records.append(json.loads(line))
            except Exception:
                continue

    counter = Counter(r["skill_name"] for r in records)
    total = len(records)

    print(f"## Skill使用統計（累計 {total} 回）\n")
    print(f"| Skill | 回数 | 割合 |")
    print(f"|---|---|---|")
    for skill, count in counter.most_common():
        pct = count / total * 100
        print(f"| {skill} | {count} | {pct:.1f}% |")

if __name__ == "__main__":
    main()
```

Skillとして登録することで、会話中に `skill-stats` を呼ぶだけで最新の統計が出てきます。

## まとめ

| 方法 | 精度 | 手軽さ | 採用 |
|---|---|---|---|
| JSONLログを事後集計 | 低い（ローテーション・重複） | 楽 | ✗ |
| PostToolUseフックで記録 | 高い（呼ばれた瞬間に追記） | 設定が必要 | ✓ |

Claude Code の hooks 機能はドキュメントが少なく、試行錯誤が必要でしたが、一度設定すれば完全に自動で動き続けます。

使用統計を見ると「あのSkillは2ヶ月で1回しか使っていない」といった発見があり、Skillの棚卸しをする動機になりました。「たくさん作るより、よく使うものを育てる」という方向性が数字で確認できるのは思っていたより有益でした。

---

[体験記はNoteで書いています](https://note.com/rinomiya_sumoru)
