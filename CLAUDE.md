# CLAUDE.md

このリポジトリは zono819 の Zenn 記事置き場。GitHub push で Zenn に自動デプロイされる。

## 記事作成の手順

1. `npx zenn new:article --slug <slug>` でファイル生成
2. `articles/<slug>.md` を編集
3. frontmatter の `published: false` のまま push → 下書き
4. 公開時は `published: true` に変えて push

## frontmatter の必須項目

```yaml
title: ""        # 記事タイトル
emoji: ""        # アイキャッチ絵文字（1文字）
type: "tech"     # tech（技術記事）or idea（アイデア）
topics: []       # タグ（最大5つ、英小文字）
published: false # false=下書き / true=公開
```

## 運用ルール

- slug はケバブケース（例: `mlit-mcp-x402-a2a`）
- 下書きは `published: false` のまま push してよい
- 公開前に `npx zenn preview` でローカル確認推奨
- `node_modules/` は `.gitignore` 済みなので commit 不要

## 進行中の記事

- Issue #1: MLIT MCP × x402 × A2A AgentCard 記事（下書き作成中）
  https://github.com/zono819/zenn-articles/issues/1
