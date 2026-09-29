# CLAUDE.md

## このプロジェクト

技術書を輪読しながら、理解を深めるためのプロジェクト。

## 対象の本

| 状態 | 本 | 目次 | `<book-slug>` | tags |
| --- | --- | --- | --- | --- |
| **進行中** | 単体テストの考え方／使い方（Vladimir Khorikov 著・須田智之 訳） | `unit-testing-toc.md` | `unit-testing` | `[単体テストの考え方/使い方]` |
| ペンディング | クリーンアーキテクチャ（Robert C. Martin 著） | `clean-architecture-toc.md` | `clean-architecture` | `[クリーンアーキテクチャ]` |

- 本の指定が無い依頼は「進行中」の本として扱う
- まとめの `title` は `本の名前 - ◯章：タイトル`（例：`単体テストの考え方/使い方 - 2章：単体テストとは何か？`）
- 章・節の表記は各目次ファイル（日本語版）に合わせる

## 読者の前提

- Ruby/Railsエンジニア
- コード例はRuby/Railsで具体化する

## 使い方（モード）

用途に応じて `.claude/rules/` 配下のルールを使い分ける。

- **壁打ち・まとめ** → `.claude/rules/sparring.md` を参照
- **進捗管理** → `.claude/rules/progress.md` を参照
- **振り返り・自己改善** → `.claude/rules/retrospective.md` を参照（常時バックグラウンドで動作）

進捗は `progress.md`、まとめは `_posts/` に格納。対話の気づきは `observations.md` に蓄積される。

## 本ごとの注意点

### 単体テストの考え方／使い方

- 本のコード例は C#（xUnit / Moq）。Ruby/Rails では RSpec（`let` / `instance_double` / `allow` / `expect(...).to have_received` 等）に置き換えて示す
- 古典学派 vs ロンドン学派、モック vs スタブ、観察可能な振る舞い vs 実装の詳細など、対になる概念の区別が本書の軸。用語の境界を曖昧にしない
- Rails 特有の事情（ActiveRecord でドメインと DB が密結合、`spec/requests` と `spec/models` の役割分担、FactoryBot、`travel_to` 等）と本の主張のトレードオフを明確にする
- クリーンアーキテクチャで議論した内容（Humble Object、境界、DIP など）と繋がる箇所では、その繋がりにも触れる

### クリーンアーキテクチャ

- SOLID原則とコンポーネントの原則は対応関係がある（CCPはSRPのコンポーネント版、CRPはISPのコンポーネント版など）。議論時は過去の原則との繋がりにも触れる
- 本の主張と現実のRails開発のトレードオフを明確にする
