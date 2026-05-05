# zenn-articles

zono819 の Zenn 記事リポジトリ。GitHub push で自動デプロイ。

## 記事の作成

```bash
npx zenn new:article --slug my-article-slug
```

`articles/` 配下に `.md` ファイルが生成される。

## 下書き / 公開の切り替え

frontmatter の `published` フィールドで制御する。

```yaml
---
title: "記事タイトル"
emoji: "🗾"
type: "tech"          # tech or idea
topics: ["topic1"]    # 最大5つ
published: false      # false = 下書き / true = 公開
---
```

- `published: false` で push → Zenn の下書きに保存される
- `published: true` に変えて push → 公開される
- ブランチや別ディレクトリによる管理は不要

## プレビュー

```bash
npx zenn preview
# → http://localhost:8000
```

## 参考

- [Zenn CLI ガイド](https://zenn.dev/zenn/articles/zenn-cli-guide)
- [GitHub 連携ガイド](https://zenn.dev/zenn/articles/connect-to-github)
