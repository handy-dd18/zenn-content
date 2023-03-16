# Zenn CLI

* [📘 How to use](https://zenn.dev/zenn/articles/zenn-cli-guide)

# command list
## Zenn CLI Update

$ npm install zenn-cli@latest

## article create
$ npx zenn new:article  

$ npx zenn new:article --slug 記事のスラッグ --title タイトル --type idea

$ npx zenn preview # プレビュー開始  

## article delete
Remove from Dashboard

# memo

## 必要情報
---

title: "" # 記事のタイトル  
emoji: "😸" # アイキャッチとして使われる絵文字（1文字だけ）  
type: "tech" # tech: 技術記事 / idea: アイデア記事  
topics: [] # タグ。["markdown", "rust", "aws"]のように指定する  
published: true # 公開設定（falseにすると下書き）  

---

<br>
  
## 記事の書き方  
* [Zenn Markdown Guide](https://zenn.dev/zenn/articles/markdown-guide)

<br>

## 画像アップロード方法
* [GitHubリポジトリ連携で画像をアップロードする方法](https://zenn.dev/zenn/articles/deploy-github-images)