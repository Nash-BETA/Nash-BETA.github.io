---
title: "クリーンアーキテクチャ - 25章：レイヤーと境界"
date: 2026-05-17 20:14:37 +0900
categories: [輪読, クリーンアーキテクチャ]
tags: [クリーンアーキテクチャ]
---

# 25章：レイヤーと境界 まとめ

## 要点

- 一見シンプルなシステムでも、境界はいくらでも引ける
- アーキテクトの仕事は「境界がどこにあり得るか」を見抜くこと、そして「どれを実装し、どれを引かないか」を判断すること
- 境界を引かなさすぎるとリジッド、引きすぎるとオーバーエンジニアリング
- 判断軸は「**変化しそうなところに境界を引く**」

---

## 疑問①：「英語UI」って具体的に何を指している？

最初の切り分けでの「英語UI」は、英語の入出力を扱う層ぜんぶ。コマンドのパースもテキスト生成もまだ分かれていない状態。

```ruby
class EnglishUI
  # 入力：英語テキスト → ゲームコマンド
  def parse(input_text)
    # "go east" → MoveCommand.new(:east)
  end

  # 出力：ゲーム状態 → 英語テキスト
  def render(state)
    # state → "You are in a room. There's a passage to the east"
  end
end
```

次の節（流れを分割する）で、これを Language と TextDelivery に分解していく。

## 疑問②：GameRules って何？

ゲーム本体のロジックだけを持ったクラス。入出力もテキストも保存も知らない、純粋にゲームルールだけ。

```ruby
class GameRules
  attr_reader :player_room, :arrows, :wumpus_room

  def initialize
    @player_room = 1
    @arrows = 3
    @wumpus_room = 7
  end

  # 抽象コマンドを受け取る（テキストは知らない）
  def move(direction)
    @player_room = cave_map[@player_room][direction]
    check_hazards
  end

  def shoot(direction)
    @arrows -= 1
    target = cave_map[@player_room][direction]
    @killed_wumpus = (target == @wumpus_room)
  end

  # 抽象状態を返す（テキストではなく構造化データ）
  def state
    {
      player_room: @player_room,
      arrows: @arrows,
      adjacent_rooms: cave_map[@player_room],
      smells_wumpus: adjacent_to_wumpus?,
      killed_wumpus: @killed_wumpus
    }
  end
end
```

ポイント：

- `move(:east)` を受け取るのであって、`"go east"` という文字列は受け取らない
- 返すのは状態のハッシュ。「あなたは部屋にいる」という英語テキストではない
- 「英語」も「ディスク保存」も知らない

Railsでいうと、「Modelの中でも特にビジネスロジックだけを抜き出した部分」のような立ち位置（ActiveRecordの永続化機能などは持たない）。

## 疑問③：ストリームが複数ある／さらに分割できるって？

Hunt the Wumpus でもデータの流れは複数本ある：

1. **入力ストリーム**：プレイヤー入力 → ゲームコマンド
2. **出力ストリーム**：ゲーム状態 → プレイヤーへのテキスト
3. **保存ストリーム**：ゲーム状態 → ディスク/DB

それぞれが独立して変わる理由を持つ（出力だけ多言語化、保存だけクラウド化、など）。

さらに「出力ストリーム」1本も分割できる：

```
ゲーム状態
  ↓
Language（英語 / スペイン語 / 日本語） ← 言葉に翻訳
  ↓
TextDelivery（コンソール / SMS / ファイル） ← 配信媒体
  ↓
プレイヤー
```

Language（言葉）と TextDelivery（媒体）は直交する関心事。英語×コンソール、スペイン語×SMS、自由に組み合わせ可能。

つまり「英語UI」と一塊にしていた部分は、実はもう一段境界が引ける。
