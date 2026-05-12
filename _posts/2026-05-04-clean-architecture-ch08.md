---
title: "クリーンアーキテクチャ - 8章：OCP（開放閉鎖の原則）"
date: 2026-05-04 10:48:13 +0900
categories: [輪読]
tags: [クリーンアーキテクチャ]
---

# 8章：OCP（開放閉鎖の原則） まとめ

## 要点

- case文やif分岐が増えていく箇所がOCPの適用ポイント
- ポリモーフィズムで各クラスに切り出し、既存コードを修正せずに拡張できるようにする

---

## 疑問①：case文を増やしていくなら結局影響するのでは？

影響を0にするのではなく、影響を少なくするための話。完全に変更ゼロは無理だが、変更箇所を局所化するのがOCPの狙い。

## 疑問②：Rubyならcase文に頼らない書き方ができる

```ruby
# NG：typeが増えるたびにcase文を修正
class NotificationSender
  def call(type, message)
    case type
    when :email
      send_email(message)
    when :slack
      send_slack(message)
    end
  end
end

# OK：ポリモーフィズムで拡張に対して開く
class EmailNotification
  def call(message)
    # メール送信
  end
end

class SlackNotification
  def call(message)
    # Slack送信
  end
end

# Rubyならconstantizeで動的に解決
klass = "#{type.to_s.camelize}Notification".constantize
klass.new.call(message)
```

Rubyの動的な特性を活かせばcase文そのものを排除できる。新しい通知手段はクラスを追加するだけ。
