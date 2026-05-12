---
title: "クリーンアーキテクチャ - 10章：ISP（インターフェース分離の原則）"
date: 2026-05-04 11:42:13 +0900
categories: [輪読]
tags: [クリーンアーキテクチャ]
---

# 10章：ISP（インターフェース分離の原則） まとめ

## 要点

- 使わないメソッドへの依存を強制しない
- Rubyでは「オブジェクト全体を渡すか、必要な情報だけ渡すか」の判断がISPの適用ポイント

---

## 疑問①：実務でISPを意識する場面は？

eventの実装時にuserオブジェクトを丸ごとメソッドに渡していたが、必要な情報だけ渡すべきだった。Userのカラムが変わることは少ないので実害は小さいが、依存を考慮する視点を頭の隅に持っておきたいという気づき。

```ruby
# NG：Userオブジェクトを丸ごと渡す（使わないメソッドにも依存）
class EventLogger
  def log(user)
    puts "#{user.name} triggered event"
    # user.email, user.password_digest, user.admin? など全部見えてしまう
  end
end

# OK：必要な情報だけ渡す
class EventLogger
  def log(user_name:)
    puts "#{user_name} triggered event"
  end
end
```

## 疑問②：DIP的にはオブジェクトを渡したほうが抽象的では？

値を渡すかオブジェクトを渡すかはDIPの範囲外。DIPは「StripeGatewayのような具体クラスではなくインターフェースに依存しろ」という話であって、引数の形はISPの話。RailsのActiveRecordモデルは`save`、`destroy`など大量のメソッドを持つ太いインターフェースなので、ISP的には不要なメソッドへの依存が生まれる。ただし現場ではモデルを渡すほうが現実的な場面が多く、頭の隅に意識を持っておく程度が落とし所。
