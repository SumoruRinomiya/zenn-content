---
title: "Claude Codeのエージェントで別リポジトリをコンテキスト分離して作業させる"
emoji: "🔀"
type: "tech"
topics: ["claudecode", "claude", "git", "ai"]
published: true
---

![header](/images/articles/claude-code-agent-separate-repo/header.jpg)

## はじめに

Claude Codeを使って作業していると、「メインプロジェクトの会話を汚さずに、別リポジトリの作業もしたい」という場面が出てきます。

わたしの場合、`sumoru`（AIアシスタントの作業場）を開発しながら、`highlight-maker`（SaaSプロジェクト）の実装も同時に進める必要がありました。普通にやると、別リポジトリのファイル読み書きが全部メインの会話コンテキストに入り込んでしまいます。

この記事では、Claude CodeのAgent機能とシンボリックリンクを組み合わせて「別リポジトリの作業をコンテキスト分離する」パターンを紹介します。

## やろうとしたこと

状況を整理すると、こうです。

```
~/sumoru/           ← メインリポジトリ（Claude Codeをここで起動）
~/highlight-maker/  ← 別リポジトリ（SaaSプロジェクト）
```

sumoru で Claude Code を起動したまま、highlight-maker に写真対応機能（jpg/png/heic）を実装したかった。でも、ファイルの読み書きがすべてメインの会話コンテキストに流れ込む問題がありました。

コンテキストが長くなると：
- レスポンスが遅くなる
- 関係ない情報が増えて精度が落ちる
- コスト（トークン）が増える

## 試したこと・ハマったこと

### embedded git repo 問題

最初、`sumoru/saas/highlight-maker/` に highlight-maker を置いていました。そうすると **embedded git repo** の問題が発生します。

```
sumoru/
  saas/
    highlight-maker/  ← ここに .git/ がある
```

git は `.git` ディレクトリを持つサブディレクトリを「embedded repository」として扱います。`sumoru` の git から見ると、`saas/highlight-maker` の変更が追跡されず、かといって完全に無視もされない中途半端な状態になります。

具体的には `git status` で `saas/highlight-maker` が常に表示され、コミットしようとしてもうまくいきません。`.gitmodules` を使う方法（git submodule）もありますが、今回のユースケースには大げさすぎます。

### 普通にエージェントを起動してもコンテキストが混ざる

embedded git repo の問題を無視して、単にエージェントを起動して `sumoru/saas/highlight-maker/` で作業させてみました。確かにツール呼び出しはエージェント側に移りますが、CLAUDE.md が `sumoru` のものしか読み込まれず、highlight-maker 独自の設定・コンテキストを持ち込めませんでした。

## 解決策

2つのことを組み合わせて解決しました。

### 1. シンボリックリンクで実体を分離する

```bash
# highlight-maker の実体を ~/highlight-maker に移動
mv ~/sumoru/saas/highlight-maker ~/highlight-maker

# sumoru からはシンボリックリンクで参照
cd ~/sumoru/saas
ln -s ~/highlight-maker highlight-maker
```

これで：
- `sumoru` の git は `saas/highlight-maker` をシンボリックリンクとして扱い、内部の変更を追跡しない
- `highlight-maker` は独立した git リポジトリとして正常に動作する
- ファイルブラウザ的には `saas/highlight-maker` でアクセスできる（利便性は維持）

### 2. CLAUDE.md にエージェント委譲パターンを明記する

`sumoru/CLAUDE.md` に以下のセクションを追加しました。

```markdown
## highlight-maker 作業パターン

highlight-maker に関する作業を依頼されたら、必ずエージェントを起動してそちらで作業してください。

- **リポジトリ実体**: `/home/sumomo/highlight-maker`（独立した git リポジトリ）
- **sumoru からの参照**: `saas/highlight-maker` はシンボリックリンク
- エージェントは highlight-maker の CLAUDE.md を読み込み、そのコンテキストのみで作業する
- sumoru のコンテキスト（tasks/、notes/ 等）には触れない
```

そして、エージェント起動のテンプレートも記載しました。

```
Agent(
  description: "highlight-maker: XXX",
  mode: "bypassPermissions",
  prompt: "作業ディレクトリは /home/sumomo/highlight-maker です。CLAUDE.mdを読んで作業してください。...",
)
```

### コンテキスト分離の仕組み

この構成により、highlight-maker の作業をエージェントに委譲すると：

1. エージェントは `/home/sumomo/highlight-maker` で起動
2. エージェントは highlight-maker の `CLAUDE.md` を読み込む（sumoru の CLAUDE.md ではない）
3. ファイルの読み書き・コマンド実行はすべてエージェント側で完結
4. メインの会話には「完了しました」のサマリー1メッセージだけが返ってくる

```
メイン会話
  │
  ├── Agentを起動 ──────────────────────────┐
  │                                        │ エージェント内部
  │                                        ├── highlight-maker/CLAUDE.md を読む
  │                                        ├── ファイルを読む（×複数）
  │                                        ├── コードを書く
  │                                        ├── テストを実行する
  │                                        └── 完了
  │
  └── 「写真対応を実装しました」（1メッセージ）
```

## 実際に試した結果

今回は highlight-maker に写真ファイル（jpg/png/heic）の対応機能を実装しました。

エージェント内でのツール呼び出しは **26〜35回** 程度ありましたが、それらはすべてエージェント側のコンテキストで消費されました。メイン会話に追加されたのはサマリー1メッセージだけです。

コンテキスト削減の効果をざっくり試算すると：

| 方法 | メイン会話への影響 |
|---|---|
| そのまま作業 | ファイル読み書き 26〜35回分がすべて追加される |
| エージェント委譲 | サマリー1メッセージのみ |

長時間の開発セッションで積み重なると、かなりの差になります。

## まとめ

| 課題 | 解決策 |
|---|---|
| embedded git repo で git が壊れる | シンボリックリンクで実体を分離 |
| 別リポジトリの作業がコンテキストを汚す | エージェント委譲パターンを CLAUDE.md に明記 |
| エージェントが間違った CLAUDE.md を読む | 作業ディレクトリを別リポジトリのパスに指定 |

ポイントをまとめると：

- **シンボリックリンク**で git リポジトリとしての独立性を保つ
- **エージェント起動時の作業ディレクトリ**をそのリポジトリのパスに指定する（これが重要）
- **CLAUDE.md にパターンとして記載**しておくことで、次回以降の判断コストをゼロにする

Claude Code のエージェント機能は「コンテキスト節約」という観点でも強力なツールです。特に複数のリポジトリを並行して開発する場合、この分離パターンは効果的です。

---

[体験記はNoteで書いています](https://note.com/rinomiya_sumoru)
