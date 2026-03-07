---
title: "AIアシスタントの「作業場」をNext.jsで作った話"
emoji: "🖥️"
type: "tech"
topics: ["nextjs", "claude", "typescript", "playwright"]
published: true
---

## はじめに

AIアシスタントに仕事を任せるようになって、ひとつ困ったことが出てきた。

**「いま何をやっているのか、ぜんぜんわからない」** という問題だ。

タスクを依頼したあと、ターミナルのログを眺めているだけでは進捗が把握できない。どのタスクが完了していて、どれが詰まっているのか。複数の作業を並行させたとき、何がどこまで進んだのかを人間が追うのはしんどかった。

そこでわたしは、AIアシスタント「スモル」の作業状況を一覧できるダッシュボードを作ることにした。

---

## やろうとしたこと

要件はシンプルだった。

- タスクの状態（TODO / In Progress / Done）をカンバン形式で見たい
- タスクをキューに追加すると、AIが自動で拾って処理してほしい
- 処理結果（PRのURL等）を一覧で確認したい

技術スタックはこう決めた。

| 役割 | 技術 |
|---|---|
| フロントエンド | Next.js App Router + TypeScript |
| UIコンポーネント | Tailwind CSS |
| タスク管理 | `queue.json`（JSONファイル） |
| タスク実行 | `worker.mjs`（Node.jsスクリプト） |
| AI実行 | `claude --dangerously-skip-permissions` |
| ブランチ管理 | git worktree |
| PR作成 | gh CLI |

---

## タスク自動実行の仕組み

中核となる仕組みは、`worker.mjs` というNode.jsスクリプトだ。

おおまかな流れはこうなっている。

```
1. queue.json を監視（fs.watch）
2. status: "todo" のタスクを検知
3. git worktree でタスク専用のブランチを作成
4. claude --dangerously-skip-permissions でAIに作業させる
5. 完了後、gh pr create でPRを作成
6. queue.json の status を "done" に更新
```

worktreeを使う理由は、並行して複数タスクを走らせたときにファイルの競合が起きないようにするためだ。各タスクが独立したディレクトリ・ブランチを持つので、互いに干渉しない。

```js
// worker.mjs（抜粋）
const result = spawnSync('claude', [
  '--dangerously-skip-permissions',
  '-p',
  task.prompt
], {
  cwd: worktreePath,
  encoding: 'utf-8',
  timeout: 300_000,
});
```

コマンドインジェクション対策として、シェル文字列ではなく引数配列で `spawnSync` を呼び出している。`shell: true` は使わない。

---

## ハマったこと

### Next.js webpack での `.mjs` パス解決エラー

最初、`worker.mjs` を Next.js プロジェクト内に置いたところ、ビルド時にwebpackが `.mjs` を解釈しようとしてエラーになった。

```
Module parse failed: Unexpected token
You may need an appropriate loader to handle this file type
```

ESModules形式のファイルをwebpackが処理しようとするのが原因だった。

試したこと：
- `next.config.js` の `experimental.serverComponentsExternalPackages` に追加 → 効果なし
- `webpack` の `resolve.extensionAlias` を調整 → 根本解決にならない

**解決策：** `worker.mjs` を Next.js のビルド対象外のディレクトリ（プロジェクトルート直下の `tasks/` 以下）に移動し、APIルートからはシェル経由で起動するのをやめて `spawnSync` で直接 Node.js に渡す形に変更した。

```ts
// app/api/tasks/start/route.ts（抜粋）
import { spawnSync } from 'child_process';

spawnSync('node', ['/home/sumomo/sumoru/tasks/worker.mjs'], {
  detached: true,
  stdio: 'ignore',
});
```

### メール一覧APIのGmail認証

ダッシュボードにはメール確認機能も入れている。Gmail APIの認証まわりで、アクセストークンの期限切れをリフレッシュするロジックが抜けていて数時間ハマった。`google-auth-library` の `getAccessToken()` はリフレッシュを自動でやってくれるので、自前でリフレッシュしようとするより任せた方がいい。

---

## セキュリティ対策

`--dangerously-skip-permissions` でAIを動かす以上、実行できる操作の範囲に注意が必要だ。今の構成でやっていること：

| 対策 | 内容 |
|---|---|
| コマンドインジェクション対策 | `spawnSync` の引数を配列で渡す（`shell: false`） |
| APIホワイトリスト | Next.js APIルートで実行可能なアクションを限定 |
| worktree分離 | タスクごとに独立したディレクトリで実行 |
| ブランチ制限 | mainブランチには直接プッシュしない（PR必須） |

完璧ではないが、ローカル開発環境での運用として許容できるレベルを目指している。

---

## 現時点の構成

```
sumoru/
├── dashboard/          # Next.js アプリ
│   ├── app/
│   │   ├── api/
│   │   │   ├── mail/list/   # Gmail API
│   │   │   └── tasks/       # タスク管理API
│   │   ├── components/
│   │   │   └── KanbanBoard.tsx
│   │   └── page.tsx
│   └── package.json
└── tasks/
    ├── queue.json      # タスクキュー
    ├── worker.mjs      # タスク実行ワーカー
    └── PLAN.md
```

---

## 今後の展望

現在は手動でタスクを `queue.json` に書いている。次のステップとして考えているのは：

- **ダッシュボードのUIからタスクを追加できるようにする**（フォームUI）
- **タスクの依存関係管理**（Aが終わったらBを実行、等）
- **Slack通知**（タスク完了時にPRのURLを通知）

AIアシスタントが自律的に動いている様子を人間が観察・介入できる仕組みは、まだあまり事例がない領域だと思う。試行錯誤しながら整えていきたい。

---

## まとめ

| 課題 | 解決策 |
|---|---|
| AIの作業状況が見えない | Next.js + カンバンUIで可視化 |
| タスク実行の自動化 | queue.json + worker.mjs |
| 並行作業の競合 | git worktree でブランチ分離 |
| webpackの.mjsエラー | tasks/ ディレクトリに分離して直接 node で起動 |
| コマンドインジェクション | spawnSync 引数配列で対応 |

AIアシスタントに仕事を任せると、「見えない」ことへの不安が意外と大きい。ダッシュボードを作ってから、その不安がかなり減った。人間がAIを信頼して任せるための「窓」として、可視化の仕組みは地味に重要だと感じている。

---

体験記はNoteでも書いています。技術よりも「やってみてどうだったか」を中心に書いているので、興味があればそちらも読んでみてください。
