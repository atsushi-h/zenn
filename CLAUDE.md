---
description: zenn-cliとBunを使用したZenn記事・書籍管理
globs: "articles/**.md, books/**/**.md, books/**/config.yaml, package.json"
alwaysApply: true
---

このリポジトリはzenn-cliを使用してZennの記事と書籍を管理します。パッケージマネージャーとしてBunを使用します。

## パッケージマネージャー: Bun

npm/yarn/pnpmの代わりにBunコマンドを使用してください:

- `bun install` - 依存関係のインストール
- `bun run <script>` - package.jsonのスクリプトを実行
- `bunx <package> <command>` - パッケージの実行（npxと同等）

## Zenn CLI コマンド

### 新規コンテンツの作成

```bash
# 新規記事を作成
bun run new:article

# 新規書籍を作成
bun run new:book
```

### ローカルプレビュー

```bash
# プレビューサーバーを起動（http://localhost:8000）
bun run preview
```

### コンテンツ一覧

```bash
# 全記事を一覧表示
bun run list:articles

# 全書籍を一覧表示
bun run list:books
```

## ファイル構造

**記事:** `articles/` ディレクトリ直下にMarkdownファイルを配置
```
articles/
├── my-first-article.md
└── another-article.md
```

**書籍:** 書籍ごとにディレクトリを作成し、config.yamlと章ファイルを配置
```
books/
└── my-book-slug/
    ├── config.yaml
    ├── cover.png
    ├── chapter1.md
    └── chapter2.md
```

## 記事のフロントマター

各記事ファイルの先頭に必要なフィールド:

```yaml
---
title: "記事のタイトル"
emoji: "📝"
type: "tech" # または "idea"
topics: ["zenn", "typescript", "bun"] # 最大5個
published: false # 公開する場合はtrue
---
```

オプションフィールド:
- `published_at: "2026-02-09"` または `"2026-02-09 10:00"` （JST タイムゾーン）

## 書籍の設定

各書籍ディレクトリに `config.yaml` を作成:

```yaml
title: "書籍のタイトル"
summary: "簡潔な説明"
topics: ["topic1", "topic2"] # 最大5個
published: false
price: 0 # 0（無料）または 200-5000（100円単位）
chapters:
  - chapter1 # .md拡張子なしのスラグ
  - chapter2
```

章のフロントマター（各.mdファイル内）:

```yaml
---
title: "章のタイトル"
free: false # 有料書籍の場合のみ意味を持つ
---
```

## スラグの命名規則

- **使用可能文字:** a-z, 0-9, ハイフン (-), アンダースコア (_) のみ
- **文字数:**
  - 記事: 1-50文字
  - 書籍/章: 12-50文字
- **重要:** 公開後はスラグを変更できません（変更すると新しい記事/書籍として扱われます）

## ワークフロー

1. **作成:** `bun run new:article` または `bun run new:book`
2. **編集:** 生成されたMarkdownファイルにコンテンツを記述
3. **プレビュー:** `bun run preview` でlocalhost:8000にアクセス
4. **コミット:** `git add . && git commit -m "記事を追加"`
5. **公開:** GitHubのmainブランチにプッシュ
6. **デプロイ:** Zennが自動的に同期して変更をデプロイ

## 重要な注意事項

- 記事/書籍のトピックは最大5個まで
- `published: true` でZennに公開されます
- プレビューサーバーのデフォルトポートは8000
- `published_at` は日本標準時（JST）を使用
- GitHub連携にはZennダッシュボードでリポジトリ接続が必要
