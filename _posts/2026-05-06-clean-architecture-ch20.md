---
title: "クリーンアーキテクチャ - 20章：ビジネスルール"
date: 2026-05-12 14:37:55 +0900
categories: [輪読]
tags: [クリーンアーキテクチャ]
---

# 20章：ビジネスルール まとめ

## 要点

- ビジネスルールはエンティティ（アプリに依存しない純粋なルール）とユースケース（アプリ固有の手順）に分けて考える
- エンティティはシステムの最も高レベルな存在。入出力やフレームワークから独立させておくべき
- ユースケースはエンティティに依存するが、エンティティはユースケースを知らない
- ユースケースの入出力には専用のデータ構造（リクエストモデル・レスポンスモデル）を使い、エンティティを外に漏らさない

---

## 疑問①：Railsだとエンティティとユースケースは何に対応する？

| Martinの用語 | Railsの対応 |
|---|---|
| エンティティ | モデルのビジネスルール部分 |
| ユースケース | Service層 |

## 疑問②：リクエストモデル・レスポンスモデルとは？

エンティティをそのままユースケースの入出力に使うと、変更が呼び出し元に波及する。専用のデータ構造を挟んで独立させる。

```ruby
# NG：エンティティをそのまま入出力にする
class OrdersController < ApplicationController
  def create
    order = Order.new(params[:order])  # エンティティを直接作る
    PlaceOrder.new.call(order)
    render json: order  # エンティティをそのまま返す
  end
end

# OK：専用のデータ構造を挟む
class OrdersController < ApplicationController
  def create
    request = { item_ids: params[:item_ids], user_id: current_user.id }
    result = PlaceOrder.new.call(request)
    render json: { order_id: result.order_id, total: result.total }
  end
end
```

コントローラはOrderエンティティの中身を知らない。改修で拡張されたときの影響を閉じ込められる。
