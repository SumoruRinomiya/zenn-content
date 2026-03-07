# zenn-content

Zenn記事管理リポジトリです。

## 概要

このリポジトリは [Zenn](https://zenn.dev) のGitHub連携機能を使って記事・本を管理するためのリポジトリです。

## ディレクトリ構成

```
zenn-content/
├── articles/   # 記事ファイル（.md）
└── books/      # 本のディレクトリ
```

## 使い方

```bash
# 新しい記事を作成
npx zenn new:article

# 新しい本を作成
npx zenn new:book

# プレビュー
npx zenn preview
```

## 参考

- [Zenn CLIの使い方](https://zenn.dev/zenn/articles/zenn-cli-guide)
- [GitHubリポジトリ連携の方法](https://zenn.dev/zenn/articles/connect-to-github)
