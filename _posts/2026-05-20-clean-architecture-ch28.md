---
title: "クリーンアーキテクチャ - 28章：テスト境界"
date: 2026-05-20 19:47:23 +0900
categories: [輪読, クリーンアーキテクチャ]
tags: [クリーンアーキテクチャ]
---

# 28章：テスト境界 まとめ

## 要点

- テストはシステムの一部であり、設計の対象である
- テストはクリーンアーキテクチャの**最外周**にいる。すべてを知っているが、誰もテストに依存しない
- テスト容易性を意識しない設計は、テストを脆くする（**脆いテスト問題**）
- 脆さを避けるには、ビジネスルールを外側の詳細から独立させ、**テスト専用 API** を経由してテストを書く

---

## 疑問①：テストが「最外周」ってどういうこと？

クリーンアーキテクチャの依存ルールにテストを当てはめると、テストは最も外側のレイヤー：

```
[抽象] Entities ← UseCases ← Adapters ← Frameworks ← Main / Tests [具象]
```

- テストは**全てを知っている**（プロダクションコードを呼び出す）
- でも**何もテストに依存しない**（プロダクションコードはテストの存在を知らない）

Rails でのアナロジー：

```
app/             ← プロダクションコード（テストを知らない）
spec/            ← テスト（app/ の全てを参照できる）
```

`spec/` は `app/` を import するが、逆は無い。

## 疑問②：「脆いテスト問題」って？

GUI、DB、外部 API などの**外側の詳細**にテストが直接依存していると：

- 画面のレイアウト変更で**ロジックは変わってないのにテストが壊れる**
- DB スキーマ変更で**関係ないテストまで失敗する**
- 外部 API が落ちると**テストが落ちる**

```ruby
# NG：脆いテスト
RSpec.describe OrdersController do
  it "creates an order" do
    visit "/orders/new"
    fill_in "Product", with: "Apple"
    click_button "Submit"
    expect(page).to have_content "Order created"  # UI 変更で壊れる
  end
end

# OK：UseCase を直接テスト
RSpec.describe PlaceOrder do
  it "places an order" do
    gateway = double("payment_gateway", charge: true)
    use_case = PlaceOrder.new(payment_gateway: gateway)
    result = use_case.call(item_id: 1, user_id: 2)
    expect(result.success?).to be true  # UI 変えても壊れない
  end
end
```

ビジネスルールは画面の有無に関係なくテストできるように設計する。「テストできない設計」はそれ自体が設計の悪臭。

## 疑問③：テスト API って何？

テストとプロダクションコードの間に挟む「**翻訳層**」。役割は2つ：

1. テストを**ビジネスの言葉**で書けるようにする
2. プロダクションコードの**構造変更からテストを守る**

```
テスト
  ↓
テストAPI（テスト専用インターフェース）
  ↓
プロダクションコード
```

### ① ビジネスの言葉で書く

ダメな例：実装詳細だらけ

```ruby
it "残高1000円のユーザーが500円の商品を買えること" do
  user = User.create(email: "test@example.com", balance: 1000)
  item = Item.create(name: "Apple", price: 500)
  controller_params = { item_id: item.id }
  session[:user_id] = user.id
  post "/orders", params: controller_params
  expect(Order.last.status).to eq "completed"
end
```

何をテストしているかが、ノイズ（email、session など）に埋もれる。

よい例：テスト API を挟む

```ruby
it "残高1000円のユーザーが500円の商品を買えること" do
  user = given_user_with_balance(1000)
  item = given_item_with_price(500)
  result = place_order(user: user, item: item)
  expect(result).to be_completed
end
```

ビジネスの言葉だけで書けている。

### ② 構造変更から守る

テスト API を挟まないと：

- `User.create` を `UserFactory.build` に変えたい → 全テスト書き直し
- `post "/orders"` を `PlaceOrder.new.call` に変えたい → 全テスト書き直し

テスト API を挟むと、内部実装が変わってもテスト API 1箇所を直すだけで済む。

```ruby
# spec/support/test_api.rb （ここだけ直せばOK）
module TestApi
  def given_user_with_balance(amount)
    User.create(email: "test@example.com", balance: amount)
    # ↑ ここを UserFactory.build に変更しても、テスト本体は影響なし
  end

  def given_item_with_price(price)
    Item.create(name: "Item", price: price)
  end

  def place_order(user:, item:)
    PlaceOrder.new.call(user_id: user.id, item_id: item.id)
  end
end
```

### Rails の感覚との対応

Rails でいう `spec/support/` の中のヘルパーモジュールや FactoryBot のファクトリが、これに近い。`sign_in_as(user)` みたいなヘルパーを書く感覚の体系化版。

「テストヘルパーを場当たり的に書く」のではなく、「**テスト専用の API を意識的に設計する**」のが Martin の主張。
