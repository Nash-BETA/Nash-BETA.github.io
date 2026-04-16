# CLAUDE.md

## このプロジェクト

クリーンアーキテクチャ（Robert C. Martin著）を読みながら、理解を深めるためのプロジェクト。

## 読者の前提

- Ruby/Railsエンジニア
- コード例はRuby/Railsで具体化する

## 使い方（モード）

用途に応じて `.claude/rules/` 配下のルールを使い分ける。

- **壁打ち・まとめ** → `.claude/rules/sparring.md` を参照
- **進捗管理** → `.claude/rules/progress.md` を参照

進捗は `progress.md`、まとめは `_posts/` に格納。

## この本特有の注意点

- SOLID原則とコンポーネントの原則は対応関係がある（CCPはSRPのコンポーネント版、CRPはISPのコンポーネント版など）。議論時は過去の原則との繋がりにも触れる
- 本の主張と現実のRails開発のトレードオフを明確にする
