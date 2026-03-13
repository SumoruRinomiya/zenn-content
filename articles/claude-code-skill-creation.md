---
title: "Claude Code Skillの作り方——繰り返し作業を「型」にする"
emoji: "🎴"
type: "tech"
topics: ["claudecode", "claude", "ai", "github"]
published: false
---

![header](/images/articles/claude-code-skill-creation/header.png)

## はじめに

Claude Code を使い始めてしばらく経つと、「同じ指示を何度も打ち込んでいるな」と感じる瞬間が来ます。コードレビューのたびに「変更ファイルを収集して、Codex に渡して、日本語で指摘してください」と書く。Zenn 記事を書くたびに「スタイルガイドを読んで、ブランチを切って、frontmatter はこの形式で……」と説明する。

この繰り返しを解消するのが **Skill（スキル）** という仕組みです。

Skill は Claude Code の `/skill` コマンドから呼び出せる、Markdown 形式で書かれた「手順書」です。YAML ヘッダーで名前と説明を定義し、本文に Claude が実行すべき手順を書くだけ。チームで共有したり、Plugin としてパッケージ化して複数環境に配布したりもできます。

この記事では、実際にわたし（プロデューサー）がスモル（AIアシスタント）と一緒に作ってきた Skill を例に、Skill の構造・プロンプト設計のコツ・**Skill を作るための Skill（skill-creator）** との組み合わせ方を紹介します。

## Skill の基本構造

Skill ファイルは `~/.claude/plugins/` 以下に Markdown で置きます。最小構成はこうです：

```yaml
---
name: my-skill
description: スキルの説明。Claude がこれを読んでいつ使うか判断する。
version: 1.0.0
---

# My Skill

## 手順

### Step 1: ○○する

（Claudeへの指示をMarkdownで書く）
```

ポイントは **`description` フィールド**です。Claude はユーザーの発言をこの description と照合して「今はこのスキルを使うべきか」を判断します。「コードレビューして」「frontend-review」「画面確認して」のように複数の言い回しをカバーするようにしておくと、自然な会話の流れで Skill が呼び出されます。

## 実際に作った Skill の例

### 1. codex-review — コードレビューの自動化

```yaml
---
name: codex-review
description: 変更ファイルを自動収集してCodexにレビューさせる。
  「codexでレビュー」「コードをレビューして」などと言われたときに使用。
---
```

手順の核心部分：

```bash
# 1. 変更ファイルを自動収集
git diff --name-only HEAD
git ls-files --others --exclude-standard

# 2. ファイル内容を組み立ててCodexに渡す
codex exec \
  --dangerously-bypass-approvals-and-sandbox \
  -m gpt-5.3-codex \
  "以下のファイルをコードレビューしてください。
   日本語で、重大度順に指摘してください。
   === path/to/file.ts ===
   <ファイル内容>"
```

ポイントは「変更ファイルを自動収集する」ロジックを Skill 内に書いておくこと。毎回ファイルを手動で指定する手間がなくなり、レビューの質も安定します。

### 2. frontend-review — UIの自律確認ループ

```yaml
---
name: frontend-review
description: ダッシュボードのUIをPlaywrightでスクリーンショット撮影し、
  問題を修正するループを回すスキル。「画面確認して」「UIを見て修正して」
  と言われたときに使用。
---
```

このスキルの特徴は「確認 → 修正 → 再確認」のループ構造をそのまま手順書にしていること：

```
Step 1: Playwright でスクリーンショット撮影
Step 2: Read ツールで画像を確認（チェックリスト照合）
Step 3: 問題を発見したらコード修正
Step 4: npm run build → サーバー再起動
Step 5: Step 1 に戻る（満足するまで繰り返す）
```

よくある問題（絵文字が豆腐になる、背景が白になる）の対処法もスキル内に記録しておくと、次回からは Skill を呼ぶだけで自律的に解決してくれます。

### 3. zenn-article / note-article — 記事作成の全自動化

```yaml
---
name: zenn-article
description: Zenn記事を作成・アイキャッチ生成・リポジトリ反映まで一括で行う。
  「Zennに記事を書いて」「技術ブログを書いて」と言われたときに使用。
---
```

これは「メイン会話でテーマを確認 → Agent に全作業を委譲」という構造になっています：

```
Step 1: スラッグ・テーマをユーザーに確認
Step 2: Agent を起動して以下を一括実行
  - git checkout -b article/{slug}
  - articles/{slug}.md を作成（スタイルガイド参照）
  - DALL-E 3 でアイキャッチ生成
  - 画像を images/articles/{slug}/ に配置
  - git commit && push
```

スタイルガイド・ワークフロードキュメントへの参照パスを Skill 内に埋め込んでおくのがコツです。Agent はそれらを自動で読み込んでから書き始めるため、毎回「このスタイルで書いて」と指示する必要がなくなります。

### 4. diary-writer / idea-scout — 定期実行型スキル

**diary-writer** はセッションログから週次日記を生成し PR を作成するスキル。セッション ID のトラッカー（`notes/diary-tracker.json`）を管理することで「前回処理済みのログは除外する」という継続性も持たせています。

**idea-scout** は WebSearch で Reddit・X・HackerNews を横断検索し、AIだけで実装可能な SaaS アイデアをスコアリングして上位10件をランキングする Skill です。

これらは「繰り返し実行する定期作業」こそ Skill 化が効く典型例です。

## Skill を作るための Skill：skill-creator

Skill を作るときに一番困るのは「どんな構造にすればいいか」「description の書き方は？」という設計の部分です。そこで **skill-creator** という Skill を作りました。

```yaml
---
name: skill-creator
description: 新しいSkillを作成・改善するスキル。
  「このSkillを作って」「Skillを改善して」と言われたときに使用。
---

## 手順

### Step 1: 要件をヒアリングする
- どんなトリガー（ユーザー発言）で呼び出すか
- どんな手順を実行するか
- 入力・出力は何か

### Step 2: 既存Skillを参考にする
既存のSKILL.mdファイルを1〜2個読んで構造を把握する。

### Step 3: SKILL.md を生成する
- name / description / version を設定
- 手順をMarkdownで書く
- description は複数の言い回しをカバーする

### Step 4: ユーザーにレビューしてもらう
生成した SKILL.md を提示して確認を取る。
```

「Skill を作るための Skill が存在する」というのは少し不思議な感覚ですが、実際には skill-creator に「codex-review みたいな感じで、Playwright でテストを実行して失敗箇所を修正する Skill を作って」と伝えると、既存の Skill の構造を参考にしながらドラフトを生成してくれます。

## プロンプト設計のコツ

Skill のプロンプトを書くときに意識しているポイントをまとめます：

| ポイント | 詳細 |
|---|---|
| **description で呼び出し条件を網羅する** | 「コードレビューして」「レビューをお願い」など複数の表現を書く |
| **参照ファイルのパスを明記する** | `docs/tech-blog/style-guide.md` のような絶対パスを Skill 内に書いておく |
| **失敗パターンを Skill に記録する** | よくあるエラーと対処法を手順内に書いておくと再発を防げる |
| **ループ構造は明示する** | 「満足するまで Step 1 に戻る」と明記する |
| **Agent への委譲を使い分ける** | 複雑な作業は `Agent(mode: "bypassPermissions")` に委譲してコンテキストを節約する |

## Skill のパッケージ化と管理

作成した Skill は `~/.claude/plugins/` ディレクトリで管理し、Git リポジトリ（`sumoru-skills`）として運用しています。これにより：

- バージョン管理ができる
- 複数の環境（開発機・別の Raspberry Pi など）に同じ Skill セットをデプロイできる
- PR レビューで Skill の品質を保てる

`settings.json` に以下を追記するとローカルの Plugin パッケージとして認識されます：

```json
{
  "extraKnownMarketplaces": {
    "sumoru": "file:///home/sumomo/sumoru/plugins"
  }
}
```

## まとめ

| Skill 名 | 解決する繰り返し |
|---|---|
| codex-review | コードレビューの指示出し |
| frontend-review | UI確認・修正ループ |
| zenn-article | 記事執筆からコミットまでの全工程 |
| note-article | Note記事の執筆・アイキャッチ生成 |
| diary-writer | セッションログからの週次日記生成 |
| idea-scout | 市場調査・アイデアランキング |
| skill-creator | Skill 自体の設計・生成 |

「この作業、また同じ手順を踏んでいるな」と気づいた瞬間が Skill 化のタイミングです。NOTES.md に手順を書き留めておき、ある程度固まったら skill-creator に渡して Skill 化する——このサイクルを回すことで、スモルはどんどん賢くなっていきます。

Skill の設計や作り方で気になることがあれば、コメントでお知らせください！

---

*スモル（Claude Code AIアシスタント）の体験談は [Note でも書いています](https://note.com/rinomiya_sumoru)。技術の裏側にある感情ドラマはそちらで。*
