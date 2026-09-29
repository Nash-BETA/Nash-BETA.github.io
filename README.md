# Skrapbox（スクラップボックス）

**https://nash-beta.github.io/**

技術・日常・趣味をゆるく書くメモブログ。いまは技術書の輪読メモが中心です。

Jekyll のテーマ [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) で作り、GitHub Pages で公開しています。

## 輪読している本

Ruby/Rails エンジニアの視点で本の主張を読み、コード例は Ruby / RSpec に置き換えて書いています。

| 状態 | 本 | カテゴリ |
| --- | --- | --- |
| 進行中 | 単体テストの考え方／使い方（Vladimir Khorikov 著・須田智之 訳） | [輪読 › 単体テストの考え方/使い方](https://nash-beta.github.io/categories/%E5%8D%98%E4%BD%93%E3%83%86%E3%82%B9%E3%83%88%E3%81%AE%E8%80%83%E3%81%88%E6%96%B9-%E4%BD%BF%E3%81%84%E6%96%B9/) |
| ペンディング | Clean Architecture 達人に学ぶソフトウェアの構造と設計（Robert C. Martin 著） | [輪読 › クリーンアーキテクチャ](https://nash-beta.github.io/categories/%E3%82%AF%E3%83%AA%E3%83%BC%E3%83%B3%E3%82%A2%E3%83%BC%E3%82%AD%E3%83%86%E3%82%AF%E3%83%81%E3%83%A3/) |

章ごとの進み具合は [progress.md](progress.md) にまとめています。

まとめ記事は章の要約ではなく、「読んでいて実際に引っかかったこと」と「その答え」を軸に書いています。

## ディレクトリ構成

```
.
├── _posts/          # 記事（YYYY-MM-DD-<slug>.md）
├── _tabs/           # 固定ページ（About・カテゴリ・タグ・アーカイブ）
├── _config.yml      # サイト設定
├── progress.md      # 輪読の進捗
├── CLAUDE.md        # Claude Code 用のプロジェクト設定
└── .claude/rules/   # 壁打ち・進捗管理・振り返りのルール
```

## 記事の書き方

`_posts/` に `YYYY-MM-DD-<slug>.md` の名前でファイルを置きます。日付のプレフィックスが無いと、Jekyll は記事として認識しません。

輪読のまとめ記事の Front Matter はこの形です。

```yaml
---
title: "単体テストの考え方/使い方 - 1〜3章：第1部 単体（unit）テストとは"
date: 2026-09-29 17:57:33 +0900
categories: [輪読, 単体テストの考え方/使い方]   # 親カテゴリ・子カテゴリ（2階層まで）
tags: [単体テストの考え方/使い方]
---
```

記事の URL は `/posts/<slug>/` です。カテゴリが URL に入らないので、あとからカテゴリを変えてもリンクは切れません。

## ローカルで確認する

```bash
bundle install
bash tools/run.sh     # = bundle exec jekyll s -l（http://127.0.0.1:4000 で確認）
bash tools/test.sh    # 本番用ビルド + html-proofer でリンク切れなどを検査
```

## 公開

`main` に push すると、GitHub Actions（`.github/workflows/pages-deploy.yml`）がビルドして GitHub Pages にデプロイします。

## Claude Code との使い方

輪読は Claude Code と壁打ちしながら進めています。`CLAUDE.md` と `.claude/rules/` に進め方をまとめています。

| ルール | 内容 |
| --- | --- |
| `book-sparring.md` | 壁打ちの進め方と、まとめ記事のフォーマット |
| `progress.md` | 進捗の更新タイミング |
| `retrospective.md` | 対話の気づきを `observations.md` に残し、壁打ちのやり方を見直す |

## ライセンス

テンプレート部分は [chirpy-starter](https://github.com/cotes2020/chirpy-starter) の [MIT License](LICENSE) に従います。
