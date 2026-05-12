---
title: "クリーンアーキテクチャ - 7章：SRP（単一責任の原則）"
date: 2026-05-12 10:23:13 +0900
categories: [輪読]
tags: [クリーンアーキテクチャ]
---

# 7章：SRP（単一責任の原則） まとめ

## 要点

- **SRPの核心は「誰のために変更するか」でクラスを分けること。アクターが違えば変更理由も変更タイミングも違うので、同じクラスに同居させると意図しない影響やマージ競合が起きる**
- ドメイン概念ではなくアクター（変更を要求する人・グループ）が基準
- RailsでService層を設けても、その中でアクターごとにクラスを分けていなければSRP違反

---

## 疑問①：Service層があればSRP守れているのでは？

Service層を設けていても、1つのServiceに複数アクターの責務が混在していたら意味がない。

```ruby
# NG：1つのServiceに複数アクターの責務が混在
class EmployeeService
  def calculate_pay(employee)
    # 経理部が使う：給与計算
    hours = employee.regular_hours
    hours * employee.pay_rate
  end

  def report_hours(employee)
    # 人事部が使う：勤怠レポート
    hours = employee.regular_hours  # calculate_payと同じメソッドを使っている
    generate_report(hours)
  end
end

# OK：アクターごとにクラスを分ける
class PayCalculator
  def call(employee)
    employee.regular_hours * employee.pay_rate
  end
end

class HourReporter
  def call(employee)
    hours = employee.tracked_hours  # 独自のロジックを持つ
    generate_report(hours)
  end
end
```

アクターごとにクラスが分かれていれば、経理部の変更が人事部に波及しない。
