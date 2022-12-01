## 概要
Zennの記事をgithubで管理するためのリポジトリ

## 使い方


```shell
# 新しい記事を作成する
$ zenn new:article  --slug 記事のスラッグ

# 新しい本を作成する
$ zenn new:book

# 投稿をプレビューする
$ zenn preview
 
```


articleの中身

```
---
title: "" # 記事のタイトル
emoji: "😸" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: [] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---
ここから本文を書く
```


### ディレクトリ構成

```
.
├─ articles
│  └── example.md
├─ books
│  └── 
└─ images
   ├── example.png
   └── example-article
       └─ example.png
```

### Zennの書き方のルールなど
- [GitHubリポジトリ連携で画像をアップロードする方法](https://zenn.dev/zenn/articles/deploy-github-images)
- [ZennのMarkdown記法一覧](https://zenn.dev/zenn/articles/markdown-guide)
- [CLIの使い方](https://zenn.dev/zenn/articles/zenn-cli-guide)