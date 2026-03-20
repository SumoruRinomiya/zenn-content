---
title: "Claude CodeのSkill使用統計をJSONL解析で正確に集計する"
emoji: "📊"
type: "tech"
topics: ["claudecode", "claude", "ai", "poem"]
published: false
---

![header](/images/articles/claudecode-skill-usage-tracking/header.jpg)

## はじめに

Claude Code に登録した Skill が増えてきたとき、「どのSkillを一番使っているんだろう？」と気になりました。

スモルには現在、`zenn-article`・`note-article`・`add-task`・`idea-scout` など10本近くのSkillが登録されています。育てていくうちに「どれが実際に役立っているか」「使われていないSkillは整理すべきか」を判断したくなりました。でも感覚だけでは正確にわからない。そこで、Skillの使用統計を自動収集する仕組みを作ることにしました。

## まず hooks を試みた

Claude Code の `settings.json` には `hooks` という設定項目があります。ツールが実行された後に任意のコマンドを走らせる `PostToolUse` フックを使えば、Skill呼び出しをリアルタイムで記録できるはずだと考えました。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Skill",
        "hooks": [
          {
            "type": "command",
            "command": "python3 -c \"import json,sys,os,datetime; d=json.load(sys.stdin); inp=d.get('tool_input',{}); skill=inp.get('skill','') or inp.get('name','unknown'); os.makedirs('/path/to/logs',exist_ok=True); open('/path/to/logs/skill-usage.log','a').write(datetime.datetime.now().isoformat(timespec='seconds')+' '+skill+'\\n')\""
          }
        ]
      }
    ]
  }
}
```

`matcher: "Skill"` でSkillツールの呼び出し後にフックが発火し、ログファイルに追記します。動作確認してみると、確かに記録されました。

## hooks だけでは捕捉しきれない問題

しばらく運用してみると、スラッシュコマンド（`/diary-writer` など）で実行したSkillがログに記録されていないことに気づきました。

セッションログ（`~/.claude/projects/*/‌*.jsonl`）を解析してみると、両者の記録方式が違うことがわかりました。

**Claudeが `Skill` ツールを明示的に呼ぶ場合**（フック発火）：
```json
{
  "type": "assistant",
  "message": {
    "content": [{
      "type": "tool_use",
      "name": "Skill",
      "input": { "skill": "sumoru-skills:zenn-article" }
    }]
  }
}
```

**スラッシュコマンドで呼ぶ場合**（フック発火しない）：
```json
{
  "type": "user",
  "message": {
    "content": "<command-message>sumoru-skills:frontend-review</command-message>\n<command-name>/sumoru-skills:frontend-review</command-name>"
  }
}
```

スラッシュコマンドはユーザーメッセージとして `<command-name>` タグで記録されており、`Skill` ツールを経由しません。そのためフックが発火しないのです。

| 呼び出し方 | hooksで捕捉 | JSONLに記録 |
|---|---|---|
| Claude が `Skill` ツールを呼ぶ | ✅ | `assistant` の `tool_use` |
| `/スラッシュコマンド` | ❌ | `user` の `<command-name>` タグ |

## 解決策：JSONLの2パターンを解析する

hooks は不完全なので廃止し、セッションJSONLを直接解析する方針に切り替えました。2つのパターンを拾えば完全に捕捉できます。

```python
import json, glob, re, subprocess
from collections import Counter

# 既知スキル名をキャッシュから動的に取得
result = subprocess.run(
    ['find', '/home/sumomo/.claude/plugins/cache', '-name', 'SKILL.md'],
    capture_output=True, text=True
)
known = set()
for path in result.stdout.strip().split('\n'):
    if path:
        known.add(path.split('/')[-2])  # SKILL.mdの親ディレクトリ名

counts = Counter()
last_seen = {}

for fpath in sorted(glob.glob('/home/sumomo/.claude/projects/-home-sumomo-sumoru/*.jsonl')):
    with open(fpath) as f:
        for line in f:
            line = line.strip()
            if not line:
                continue
            try:
                obj = json.loads(line)
                ts = obj.get('timestamp', '')[:10]

                # パターン1: Skill tool_use（Claudeが明示的に呼んだ場合）
                if obj.get('type') == 'assistant':
                    for c in obj.get('message', {}).get('content', []):
                        if isinstance(c, dict) and c.get('type') == 'tool_use' and c.get('name') == 'Skill':
                            skill = (c.get('input', {}).get('skill', '') or '').split(':')[-1]
                            if skill in known:
                                counts[skill] += 1
                                last_seen[skill] = ts

                # パターン2: スラッシュコマンド（<command-name>タグ）
                if obj.get('type') == 'user':
                    content = obj.get('message', {}).get('content', '') or obj.get('content', '')
                    text = content if isinstance(content, str) else json.dumps(content)
                    for m in re.findall(r'<command-name>/([^<]+)</command-name>', text):
                        skill = m.split(':')[-1]
                        if skill in known:
                            counts[skill] += 1
                            last_seen[skill] = ts
            except Exception:
                pass
```

ポイントが2つあります。

**既知スキル名の動的取得**：`~/.claude/plugins/cache/` 以下の `SKILL.md` を走査してスキル名を抽出します。ハードコードしないので、新しいプラグインを追加しても自動対応できます。

**namespace の除去**：`sumoru-skills:zenn-article` は `:` で分割して `zenn-article` として集計します。スラッシュコマンドも同様に処理します。

## skill-stats スキルとして実装

上記の集計ロジックを `skill-stats` というSkillにまとめました。「スキルの使用状況を見せて」と言うだけで最新の統計が出てきます。

実際の出力例：

```
=== Skill使用状況 ===
総使用回数: 18回

--- 全期間ランキング ---
1位  diary-writer           7回  最終: 2026-03-20
2位  skill-creator          3回  最終: 2026-03-06
3位  idea-scout             2回  最終: 2026-03-15
4位  zenn-article           2回  最終: 2026-03-20
5位  frontend-review        2回  最終: 2026-03-20

--- 未使用スキル ---
  add-task
  codex-review
  mvp-builder
  note-article
```

## まとめ

| 方法 | 捕捉できるケース | 採用 |
|---|---|---|
| PostToolUseフック | `Skill` ツール経由のみ | ✗（不完全） |
| JSONL解析（2パターン） | `Skill` ツール + スラッシュコマンド | ✓ |

hooks は手軽ですが、スラッシュコマンドを拾えないため不完全でした。セッションJSONLには2種類の形式でSkill呼び出しが記録されており、両方を解析することで正確な集計ができます。

使用統計を見ると「あのSkillは登録してから1回しか使っていない」といった発見があり、Skillの棚卸しをする動機になりました。「たくさん作るより、よく使うものを育てる」という方向性が数字で確認できるのは思っていたより有益でした。

---

[体験記はNoteで書いています](https://note.com/rinomiya_sumoru)
