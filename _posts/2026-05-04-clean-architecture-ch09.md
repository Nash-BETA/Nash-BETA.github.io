---
title: "クリーンアーキテクチャ - 9章：LSP（リスコフの置換原則）"
date: 2026-05-12 11:13:13 +0900
categories: [輪読]
tags: [クリーンアーキテクチャ]
---

# 9章：LSP（リスコフの置換原則） まとめ

## 要点

- サブタイプは親タイプと置換可能でなければならない
- テンプレートメソッドパターンはLSPを守りやすくするための手段であり、LSP自体は概念

---

## 疑問①：テンプレートメソッドパターンとLSPの違いは？

LSPは「置換可能であるべき」という概念・原則。テンプレートメソッドパターンはそれを実現するためのデザインパターン（手段）。LSPを守る方法はテンプレートメソッドパターンだけではない。

```ruby
# テンプレートメソッドパターン：LSPを守りやすくする手段
class BaseExporter
  def call(data)
    formatted = format(data)  # サブクラスが実装
    output(formatted)          # サブクラスが実装
  end
end

class CsvExporter < BaseExporter
  def format(data) = data.map(&:to_csv)
  def output(formatted) = File.write("out.csv", formatted.join)
end

class JsonExporter < BaseExporter
  def format(data) = data.map(&:to_json)
  def output(formatted) = File.write("out.json", formatted.join)
end
```

CsvExporterもJsonExporterもBaseExporterと同じように使える（置換可能）。これがLSP。テンプレートメソッドパターンで「`format`と`output`を実装してね」という暗黙のインターフェースを作っている形。Javaなら`interface Exporter`を明示的に定義するところを、Rubyでは基底クラスやダックタイピングで実現している。
