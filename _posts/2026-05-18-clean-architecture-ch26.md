---
title: "クリーンアーキテクチャ - 26章：メインコンポーネント"
date: 2026-05-18 21:32:08 +0900
categories: [輪読]
tags: [クリーンアーキテクチャ]
---

# 26章：メインコンポーネント まとめ

## 要点

- Main は最下層・最も汚いコンポーネント。全てを知っていて、全ての具象を名指しで組み立てる
- Main は「アプリ本体」ではなく「**アプリへのプラグイン**」と見る
- 1つのアプリに対して Main は複数あっていい（本番用、開発用、テスト用、デモ用…）
- 汚さを Main に閉じ込めるのが目的

---

## 疑問①：Rails だと Main はどの辺？

Rails では Main の仕事が分散している。フレームワークが大半をやってくれるため。

| Main の役割 | Rails での場所 |
|---|---|
| 起動・ブート | `config/boot.rb`, `bin/rails` |
| アプリ全体の設定 | `config/application.rb` |
| 環境ごとの分岐 | `config/environments/{dev,prod}.rb` |
| 具象クラスの注入・組み立て | `config/initializers/*.rb` |
| ルーティング | `config/routes.rb` |
| Rack エントリポイント | `config.ru` |

具体例：決済ゲートウェイの差し替え。

```ruby
# config/initializers/payment.rb （ここが「Main」相当）
Rails.application.config.payment_gateway = case Rails.env
  when 'production' then StripeGateway.new(api_key: ENV['STRIPE_KEY'])
  when 'test'       then MockGateway.new
  else                   StripeGateway.new(api_key: ENV['STRIPE_TEST_KEY'])
end
```

- `StripeGateway` という具象を名指ししているのはここだけ
- UseCase 側は `Rails.application.config.payment_gateway` を経由して呼ぶので、Stripe を知らない

Rails 文化的にはサービスクラスの中で直接 `StripeGateway.new` と書くことも多いが、それは「Main の仕事が漏れている」状態。クリーンアーキテクチャ的にはアウト。

## 疑問②：依存の方向が逆では？ Main が消えたら何も動かないんだから、みんな Main に依存しているのでは？

「依存」には2つの意味がある。クリーンアーキテクチャで言う依存は**ソースコード依存**の方。

| 依存の種類 | 意味 | Main の位置 |
|---|---|---|
| 実行時依存（ライフサイクル） | 「A が無いと B が起動しない」 | ✅ Main 無いと何も動かない |
| ソースコード依存 | 「A が B を import / 参照している」 | ❌ UseCases は Main を知らない |

コードで見ると：

```ruby
# UseCases 層
class CreateOrderUseCase
  def initialize(payment_gateway:)
    @payment_gateway = payment_gateway
  end

  def execute(order)
    @payment_gateway.charge(order.total)
  end
end

# Main 層（initializer）
gateway = StripeGateway.new(api_key: ENV['STRIPE_KEY'])
use_case = CreateOrderUseCase.new(payment_gateway: gateway)
```

- `CreateOrderUseCase` のファイルには `StripeGateway` も `Main` も**一切登場しない**
- `Main` のファイルには `CreateOrderUseCase` と `StripeGateway` の**両方が登場する**

つまりソースコード上は Main が UseCase を知っている。逆ではない。

**実行時の流れ**：

```
Main: UseCase インスタンス作って、execute 呼ぶ
  ↓
UseCase: 渡された gateway を呼ぶ（Main には呼び返さない）
  ↓
Gateway: Stripe API を叩く
```

Main は生成と起動だけして、あとは引っ込む。UseCase が Main を呼び返すことは無い。「みんなが Main に依存している」というより、「**Main がみんなを呼び出して回している**」が正確。

**比喩：演劇の演出家**

- 演出家（Main）は役者（UseCase）と舞台（Gateway）を選んで配置する
- 開幕後は役者が芝居をする。演出家は舞台袖に引っ込む
- 役者は演出家を知らない（役者の台本に「演出家」は出てこない）
- でも演出家がいないと公演は始まらない
