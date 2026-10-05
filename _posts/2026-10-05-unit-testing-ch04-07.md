---
title: "単体テストの考え方/使い方 - 4〜7章：第2部 単体テストとその価値"
date: 2026-10-05 21:22:47 +0900
categories: [輪読, 単体テストの考え方/使い方]
tags: [単体テストの考え方/使い方]
---

# 第2部（4〜7章）まとめ

## 第4章 良い単体テストを構成する4本の柱

### 要点

- 1章で「解決策は第4章から」と言っていた、**テストの価値を測る物差し**が出てくる章
- 良いテストかどうかは4つの観点（柱）で採点できる。単体テストだけでなく、統合テストや E2E テストにも同じ物差しが使える

| 柱 | 一言で | 守ってくれるもの |
| --- | --- | --- |
| ① 退行（regression）に対する保護 | バグを見逃さないか | 見逃されるバグ（偽陰性） |
| ② リファクタリングへの耐性 | 中身を整理しただけで落ちないか | 嘘の警告（偽陽性） |
| ③ 迅速なフィードバック | すぐ終わるか | 気づくのが遅れること |
| ④ 保守のしやすさ | 読みやすく、動かしやすいか | テストを保つコスト |

- ①は、実行されるコードの量が多く、複雑で、業務上重要なほど強くなる。`attr_accessor :name` のような取るに足らないコードはテストしても価値がほぼ無い
- ②が無いテストは、振る舞いは正しいのに落ちる「嘘の警告（偽陽性）」を出す。原因はテストが**実装の詳細**に結び付いていること

```ruby
# ❌ 手順（実装の詳細）を確認している
it "正しい補助的描画クラスを使っている" do
  sut = MessageRenderer.new

  expect(sut.sub_renderers.map(&:class)).to eq [HeaderRenderer, BodyRenderer, FooterRenderer]
end

# ✅ 最終的な結果（観察可能な振る舞い）を確認している
it "メッセージを描画する" do
  message = Message.new(header: "h", body: "b", footer: "f")

  html = MessageRenderer.new.render(message)

  expect(html).to eq "<h1>h</h1><b>b</b><i>f</i>"
end
```

上のテストは、`BodyRenderer` を別のクラスに差し替えたり、`MessageRenderer` に直接描画させたりすると、HTML が同じでも落ちる。下のテストは HTML が同じである限り落ちない。

- ①と②は、テストの「正確さ」を2方向から支える

| | 振る舞いが正しい | 振る舞いが間違っている（バグあり） |
| --- | --- | --- |
| **テスト成功** | 真陰性（正しい判定） | **偽陰性**：バグを見逃した → ①で減らす |
| **テスト失敗** | **偽陽性**：嘘の警告 → ②で減らす | 真陽性（正しい判定） |

```
テストの正確さ = 信号（見つけたバグの数）÷ ノイズ（嘘の警告の数）
```

- 偽陽性の害はプロジェクトが育つほど大きくなる。リファクタリングが必要になる頃に嘘の警告が多いと、テストの失敗が無視されるようになり、本物のバグまで見逃す（CI の「どうせまた落ちてる」や、`skip` / `xit` が増えていく状態）
- テストの価値は4本の柱の**掛け算**。1本でもゼロなら価値はゼロ。ところが①②③は同時に満点にできない

| 極端な例 | ①退行に対する保護 | ②リファクタリングへの耐性 | ③迅速なフィードバック | Rails で言うと |
| --- | --- | --- | --- | --- |
| E2E テスト | ◎ | ◎ | **✕ 遅い** | system spec だけでテストを組む |
| 取るに足らないテスト | **✕ バグを見つけない** | ◎ | ◎ | `user.name = "a"` して `name` が `"a"` か確かめる |
| 壊れやすいテスト | ◎ | **✕ すぐ落ちる** | ◎ | 発行される SQL の文字列を `eq` で比べる |

- ②と④は常に最大にし、①と③のバランスだけを調整する。②は「有るか無いか」の二択に近く、少しだけ備えることができないから。最優先は偽陽性を取り除くこと

```
②リファクタリングへの耐性：常に最大
④保守のしやすさ：常に最大
①退行に対する保護 ◀━━━━ スライダーで調整 ━━━━▶ ③迅速なフィードバック
```

- テスト・ピラミッドは「層によって①と③のどちらを重視するか」の違い。E2E は①重視で少なく、単体テストは③重視で多く。どの層も②は最大

| ピラミッドの例外 | 形 | 理由 |
| --- | --- | --- |
| ほぼ CRUD だけで、業務ルールが無いアプリ | 四角に近い | 検証すべきロジックが無いと単体テストは価値を失うが、DB と組み合わせて動くかを確かめる統合テストは価値を保つ |
| プロセス外依存が DB 1つだけの API | E2E テストを多めにしてよい | UI が無いので E2E でも十分速く、保守コストも高くならない |

Rails アプリは CRUD 中心の画面が多いので、request spec が多めで model spec が少ない構成は、本の基準でも正当化できることがある。業務ルールが複雑な部分はピラミッド型に寄せる。

- テストを「書く」ときはブラック・ボックス、「分析する」ときはホワイト・ボックス

| | 何をもとに書くか | ①退行に対する保護 | ②リファクタリングへの耐性 |
| --- | --- | --- | --- |
| ホワイト・ボックス・テスト | ソースコード（どのように動くか） | 優れている | **劣っている** |
| ブラック・ボックス・テスト | 仕様・要求（何をすべきか） | 劣っている | 優れている |

分析には SimpleCov などで未検証の分岐を見つけ、そのあとは中身を知らなかったつもりで、仕様の言葉でその分岐を通るテストを書く。1章の「網羅率は目標にせず、足りないところを見つける道具にする」の具体的な使い方。

---

### 疑問①：リファクタリングへの耐性って考えると、モックを使うロンドン学派の方が楽そう？

本の結論は逆で、**リファクタリングへの耐性はロンドン学派のほうが低い**。

ロンドン学派にも守られている部分はある。`Store` をテスト・ダブルにしているので、`Store` の中身をどう書き換えても `Customer` のテストは落ちない。その代わり、**`Customer` が `Store` をどう呼ぶか**というやり取り（実装の詳細）を確認している。2章の購入の例で、どのリファクタリングでどちらが落ちるかを並べるとこうなる。

| リファクタリング | 古典学派のテスト<br>（在庫が 10 → 5 になったか） | ロンドン学派のテスト<br>（`remove_inventory` を呼んだか） |
| --- | --- | --- |
| `Store` の内部の在庫の持ち方を変える | 落ちない | 落ちない |
| `remove_inventory` を `decrease_stock` に名前変更 | 落ちない | **落ちる** |
| 在庫チェックと減算を1つのメソッド `take(...)` にまとめる | 落ちない | **落ちる** |
| 在庫を減らす処理を `Store` から `Inventory` クラスに移す | 落ちない | **落ちる** |

どれも在庫が正しく減るという振る舞いは変わっていないので、ロンドン学派のテストが落ちるのはすべて嘘の警告（偽陽性）。

- ロンドン学派：協力者の**中身**の変更には強いが、協力者との**やり取り**を変えるリファクタリングには弱い
- 古典学派：振る舞いが変わらない限り、どちらでも落ちない

実際のリファクタリングは名前変更・統合・クラスの分割のように、ほとんどがやり取りを変えるもの。なのでモックを多用するほど偽陽性が増える。2章の「ロンドン学派の最大の課題は、テストが実装の詳細と結び付くこと」の理由がこれ（詳しくは第5章）。

### 疑問②：古典学派の方か、モックを使うのは？

モックを多用するのはロンドン学派、リファクタリングへの耐性が高いのは古典学派。古典学派もテスト・ダブルをまったく使わないわけではなく、DB や外部 API のような共有されるもの・アプリの外にあるものだけはテスト・ダブルにする。

| | テスト・ダブルにするもの | リファクタリングへの耐性 |
| --- | --- | --- |
| ロンドン学派 | 依存しているクラスほぼすべて | 低い（やり取りを変えると落ちる） |
| 古典学派 | DB や外部 API など、共有されるものや外部のものだけ | 高い（結果が同じなら落ちない） |

---

## 第5章 モックの利用とテストの壊れやすさ

### 要点

- テスト・ダブルは5種類あるが、「出ていくもの」を扱う**モック**と、「入ってくるもの」を扱う**スタブ**の2つにまとまる（2章の「モックであってる？スタブじゃなくて？」の正式な答え）

| 分類 | 含まれるもの | 扱うやり取り | すること | CQS で言うと |
| --- | --- | --- | --- | --- |
| **モック** | モック、スパイ（手書きのモック） | **出ていく**：依存の状態を変える呼び出し（メール送信など、副作用がある） | 模倣して、**検証もする** | コマンド（戻り値なし・副作用あり） |
| **スタブ** | スタブ、ダミー、フェイク | **入ってくる**：依存からデータを取る呼び出し（DB からの取得など、副作用なし） | 模倣する**だけ** | クエリ（戻り値あり・副作用なし） |

- スタブとのやり取りは検証しない（**過剰検証**）。何回呼んだかは結果を作るための途中の手順（実装の詳細）だから

```ruby
# ✅ モック：出ていく副作用（メール送信）を検証する
it "挨拶のメールを送信する" do
  gateway = instance_double(EmailGateway, send_greetings_email: nil)

  Controller.new(gateway).greet_user("user@email.com")

  expect(gateway).to have_received(:send_greetings_email).with("user@email.com").once
end

# ✅ スタブ：データを渡すだけ。確認するのは最終的な結果
it "レポートを作成する" do
  db = instance_double(Database, number_of_users: 10)

  report = Controller.new(db).create_report

  expect(report.number_of_users).to eq 10
  # ❌ expect(db).to have_received(:number_of_users).once ← 過剰検証
end
```

- RSpec の `double` / `instance_double` は**道具としてのモック**。作ったものがモックになるかスタブになるかは使い方次第（`allow` で答えを返させればスタブ、`have_received` で確認すればモック）
- アプリは**ドメイン層**（業務ルールだけ）と**アプリケーション・サービス層**（DB や外部とのやり取りを指揮する）の2層で考える（ヘキサゴナル・アーキテクチャ。詳しくは6章の疑問①）
- **システム内コミュニケーション**（アプリの中のクラス同士のやり取り）は実装の詳細なのでモックで確かめない。**システム間コミュニケーション**（外部とのやり取り）は外部との契約なので、モックで確かめてよい

| | 例 | 性質 | モックで検証 |
| --- | --- | --- | --- |
| システム内コミュニケーション | `Customer` が `Store#remove_inventory` を呼ぶ | 実装の詳細。リファクタリングで自由に変えてよい | **しない** |
| システム間コミュニケーション | アプリがメール・サービスに領収書の送信を頼む | 観察可能な振る舞い。後方互換を保つ必要がある | **してよい** |

```ruby
# ✅ アプリの境界を越える呼び出し（メール送信）だけをモックで確認する
it "購入に成功すると、領収書をメールで送る" do
  gateway = instance_double(EmailGateway, send_receipt: nil)
  sut = CustomerController.new(gateway)

  success = sut.purchase(customer_id: 1, product_id: 2, quantity: 5)

  expect(success).to be true   # 戻り値も外から観察できる結果なので確認する（モックは不要）
  expect(gateway).to have_received(:send_receipt).with("customer@email.com", "Shampoo", 5).once
end
```

- 判断の基準は「モックを使うか」ではなく「**そのやり取りがアプリの境界を越えて、外から見えるか**」。ロンドン学派はこれを区別せずにモックを使うので壊れやすい。著者が古典学派を推す理由はここで揃う
- 古典学派でも、**このアプリからしか触らない DB** はモックにしない。外から見えないので実装の詳細だから（8章で「管理下にある依存」と呼ばれる）

| プロセス外依存 | 例 | 外から見えるか | モックにするか |
| --- | --- | --- | --- |
| 他のシステムも使う | メール・サービス、メッセージ・バス、外部 API | 見える（契約になる） | する |
| このアプリからしか触らない | このアプリ専用の DB | 見えない | **しない** |

```ruby
# ❌ アプリ専用の DB をスタブにしている（ActiveRecord のクエリの組み方に縛られる）
allow(User).to receive(:where).with(active: true).and_return([user])

# ✅ DB は本物を使い、結果で確認する
create(:user, active: true)
expect(ActiveUsersReport.new.count).to eq 1
```

ただし、他のシステムからも読まれている DB（DMS で Redshift に複製され、分析側が読んでいるテーブルなど）は、テーブル構成自体が外部との契約になる。

---

### 疑問①：観察可能な振る舞いと実装の詳細の区別は、メモ必須

観察可能な振る舞いとは、**クライアントが目標を達成するために使う「操作」か「状態」**。それ以外は実装の詳細。「外から呼べるか（public か private か）」とは別の軸で、良い設計ではこの2つが一致する。

| | 観察可能な振る舞い | 実装の詳細 |
| --- | --- | --- |
| 公開する（public） | **すべき** | すべきではない（＝漏洩） |
| 公開しない（private） | 該当なし | **すべき** |

見分け方は、**1つの目標のためにクライアントが2回以上呼ぶ必要があるなら、実装の詳細が漏れている**。「ユーザー名は50文字まで」という不変条件を持つ `User` の例。

```ruby
# ❌ 実装の詳細（normalize_name）が公開されている
class User
  attr_accessor :name

  def normalize_name(name)
    (name || "").strip[0, 50]
  end
end

# 呼び出し側は、名前を変えるのに2回呼ぶ必要がある
normalized = user.normalize_name(new_name)
user.name = normalized   # normalize_name を呼び忘れると、51文字以上の名前が入る
```

```ruby
# ✅ 観察可能な振る舞い（name=）だけを公開し、正規化は中に隠す
class User
  attr_reader :name

  def name=(value)
    @name = normalize_name(value)
  end

  private

  def normalize_name(name)
    (name || "").strip[0, 50]
  end
end

user.name = new_name   # 1回で済み、不変条件も破れない
```

- 3章の「実行フェーズが2行になったら設計を疑う」と同じ見分け方。テストで2行になるのは、本番の呼び出し側でも2行必要だから
- データの操作をメソッドの中に閉じ込めるのがカプセル化。Martin Fowler の「尋ねるな、命じよ（Tell, Don't Ask）」も同じ考え方
- ActiveRecord の model なら、Rails 7.1 以降は `normalizes :name, with: ->(name) { name.strip[0, 50] }` で代入時に自動で正規化できる
- 4章の `MessageRenderer#sub_renderers` は「状態」が漏れている例。テストが壊れやすかったのは、漏れた状態を見ていたから

本の TIP は「**API をきちんと設計すれば、単体テストは自然と質の良いものになる**」。実装の詳細を private にすれば、テストからは観察可能な振る舞いしか確かめようがなくなる。

---

## 第6章 単体テストの3つの手法

### 要点

- テストの確認の仕方は「戻り値」「状態」「やり取り」の3つで、良い順に **出力値ベース ＞ 状態ベース ＞ コミュニケーション・ベース**

```ruby
# 出力値ベース・テスト：戻り値だけを確認する（副作用が無いコードにだけ使える）
it "商品が2個ある場合の割引率" do
  discount = PriceEngine.new.calculate_discount(Product.new("Hand wash"), Product.new("Shampoo"))

  expect(discount).to eq 0.02
end

# 状態ベース・テスト：実行後の状態を確認する
it "注文に商品を追加する" do
  product = Product.new("Hand wash")
  sut = Order.new

  sut.add_product(product)

  expect(sut.products).to eq [product]
end

# コミュニケーション・ベース・テスト：モックでやり取りを確認する
it "挨拶のメールを送信する" do
  gateway = instance_double(EmailGateway, send_greetings_email: nil)

  Controller.new(gateway).greet_user("user@email.com")

  expect(gateway).to have_received(:send_greetings_email).with("user@email.com").once
end
```

| | 出力値ベース | 状態ベース | コミュニケーション・ベース |
| --- | --- | --- | --- |
| ①退行に対する保護・③迅速なフィードバック | ほぼ差なし | ほぼ差なし | ほぼ差なし（使いすぎると浅いテストになる） |
| ②リファクタリングへの耐性を保つコスト | **低い** | 普通 | 普通（境界を越えるやり取り以外を確かめると壊れる） |
| ④保守のしやすさを保つコスト | **低い** | 普通（確認する状態が多いとコードが増える） | **高い**（モックの準備、モックの連鎖） |

- 状態ベースの確認が長くなるなら、確認するものを値オブジェクトにして丸ごと比較する。ただしテストのためだけに値オブジェクトにすると、11章の「プロダクション・コードへの汚染」になる
- 出力値ベースが使えるのは**純粋関数**（入力と出力がすべてメソッド・シグネチャに出ている関数）だけ。純粋でなくなる原因は、副作用・例外（隠れた出力）と、内部・外部の状態の参照（隠れた入力）

```ruby
# 純粋でない：期限切れかどうかが「いつ呼んだか」で変わる（Time.current が隠れた入力）
def expired?
  expires_at < Time.current
end

# 純粋にした：現在時刻を引数で受け取る
def expired?(now)
  expires_at < now
end
```

- **関数型アーキテクチャ**は、決定を下す**関数的核**と、決定に従って副作用を起こす**可変殻**に分ける。副作用は処理の最初（入力集め）と最後（実行）に寄せる。Ruby の世界では Gary Bernhardt の「Functional Core, Imperative Shell」として知られる考え方

```
可変殻（mutable shell）                関数的核（functional core）
① DB やファイルから入力を集める  ──▶  ② 入力だけで決定を下す（純粋関数）
③ 決定に従って保存・送信する    ◀──    決定を表すオブジェクトを返す
```

- 訪問者記録システム（`名前;日時` をファイルに追記し、上限を超えたら新しいファイルを作る）を、ファイル直書き → モック → 関数型アーキテクチャと書き換えると、テストは段階的に良くなる

| | ①ファイルに直接書く | ②ファイル操作をモックにする | ③関数型アーキテクチャ |
| --- | --- | --- | --- |
| ①退行に対する保護 | 良い | 良い | 良い |
| ②リファクタリングへの耐性 | 良い | 良い | 良い |
| ③迅速なフィードバック | **悪い** | 良い | 良い |
| ④保守のしやすさ | **悪い** | 普通 | **良い** |

```ruby
FileContent = Data.define(:file_name, :lines)        # 入力：今あるファイルの中身
FileUpdate  = Data.define(:file_name, :new_content)  # 出力：何をするかの決定

# 関数的核：決定を下すだけ。ファイルには一切触らない
class AuditManager
  def initialize(max_entries_per_file)
    @max_entries_per_file = max_entries_per_file
  end

  def add_record(files, visitor_name, time_of_visit)
    new_record = "#{visitor_name};#{time_of_visit.iso8601}"
    sorted = files.sort_by { |f| f.file_name[/\d+/].to_i }
    return FileUpdate.new("audit_1.txt", new_record) if sorted.empty?

    current = sorted.last
    index = current.file_name[/\d+/].to_i
    if current.lines.size < @max_entries_per_file
      FileUpdate.new(current.file_name, (current.lines + [new_record]).join("\n"))
    else
      FileUpdate.new("audit_#{index + 1}.txt", new_record)
    end
  end
end

# 可変殻：読む・書くだけ。if が1つも無い
class Persister
  def read_directory(dir)
    Dir.glob(File.join(dir, "*")).map { |path| FileContent.new(File.basename(path), File.readlines(path, chomp: true)) }
  end

  def apply_update(dir, update)
    File.write(File.join(dir, update.file_name), update.new_content)
  end
end

# テストは入力を渡して戻り値を比べるだけ
it "最新のファイルが上限に達したら、新しいファイルを作る" do
  files = [
    FileContent.new("audit_1.txt", []),
    FileContent.new("audit_2.txt", ["Peter;2019-04-06T16:30:00", "Jane;2019-04-06T16:40:00", "Jack;2019-04-06T17:00:00"]),
  ]

  update = AuditManager.new(3).add_record(files, "Alice", Time.iso8601("2019-04-06T18:00:00"))

  expect(update).to eq FileUpdate.new("audit_3.txt", "Alice;2019-04-06T18:00:00")
end
```

- 欠点もある。決定の途中で DB を見る必要があると崩れる（最初に全部読む＝性能が落ちる／必要か先に聞く＝分離が少し崩れる。どちらにしても**核に DB を触らせるのは絶対にしない**）。入力を全部集めるぶん本番の性能が落ち、コード量も増える
- なので採用はシステムの複雑さと重要性で決める。本が求めているのは、**すべて**ではなく**できるだけ多く**のテストを出力値ベースにすること。Rails なら、料金計算やポイント付与の判定など複雑な部分だけ PORO の関数的核に切り出し、CRUD だけの画面は ActiveRecord のまま request spec で守る、が現実的

---

### 疑問①：ヘキサゴナル・アーキテクチャってなに？違いを具体で

Alistair Cockburn が提唱した設計で、別名「ポートとアダプター（Ports and Adapters）」。**アプリの中心（業務ルール）を外の世界（DB・メール・画面・外部 API）から切り離し、外とのやり取りは決まった口（ポート）を通してだけ行う**。六角形は図の描き方にすぎない。本では2層に単純化している。

- **ドメイン層**：業務ルールだけを持つ。DB やメールのことは知らない
- **アプリケーション・サービス層**：DB から読んでドメインに渡し、結果を保存したりメールを送ったりする。外とつながるのはここだけ

「領収書をメールで送る購入処理」を3通りで書くとこうなる。

```ruby
# ① よくある Rails の書き方：ドメインが外の世界を直接触る
class Customer < ApplicationRecord
  def purchase(product, quantity)
    return false if product.stock < quantity

    product.update!(stock: product.stock - quantity)          # DB に直接書く
    ReceiptMailer.with(customer: self, product:, quantity:)
                 .receipt.deliver_later                        # メールを直接送る
    true
  end
end
```

```ruby
# ② ヘキサゴナル・アーキテクチャ：外とのやり取りはサービス層に追い出す

# ドメイン層：DB もメールも知らない。ただし自分たちの状態は変えてよい
class Customer
  def purchase(store, product, quantity)
    return false unless store.enough_inventory?(product, quantity)

    store.remove_inventory(product, quantity)   # ドメインの中での状態変更は OK
    true
  end
end

# アプリケーション・サービス層：DB とメールに触るのはここだけ
class PurchaseProduct
  def initialize(customer_id:, product_id:, quantity:, email_gateway: EmailGateway.new)
    @customer_id, @product_id, @quantity = customer_id, product_id, quantity
    @email_gateway = email_gateway   # 外部とつながる口（ポート）
  end

  def execute
    customer = CustomerRepository.find(@customer_id)   # DB から読む
    product  = ProductRepository.find(@product_id)
    store    = StoreRepository.main

    success = customer.purchase(store, product, @quantity)   # 業務ルールはドメインに任せる
    return false unless success

    StoreRepository.save(store)                                          # DB に書く
    @email_gateway.send_receipt(customer.email, product.name, @quantity) # 外部に送る
    true
  end
end
```

```ruby
# ③ 関数型アーキテクチャ：ドメインの中の状態変更も禁止する
PurchaseDecision = Data.define(:success, :new_stock)

# 関数的核：何も変えない。決定を返すだけ
class PurchasePolicy
  def self.decide(stock:, quantity:)
    return PurchaseDecision.new(success: false, new_stock: stock) if stock < quantity

    PurchaseDecision.new(success: true, new_stock: stock - quantity)
  end
end

# 可変殻（＋サービス）：入力を集め、決定どおりに副作用を起こす
decision = PurchasePolicy.decide(stock: store.stock_of(product), quantity: @quantity)
if decision.success
  StoreRepository.update_stock(store, product, decision.new_stock)
  @email_gateway.send_receipt(customer.email, product.name, @quantity)
end
```

| | ドメインが DB やメールを直接触る | ドメインの中で状態を変える | ドメインのテストの手法 |
| --- | --- | --- | --- |
| ① よくある Rails | する | する | 統合テスト（DB が要る） |
| ② ヘキサゴナル | **しない** | する | 状態ベースの単体テスト |
| ③ 関数型 | しない | **しない** | 出力値ベースの単体テスト |

②と③の違いは「ドメインの中での状態変更を許すかどうか」だけ。本が「関数型アーキテクチャは、ヘキサゴナルに極端な制約を課したもの」と言うのはこのため。クリーンアーキテクチャの「依存は内側（業務ルール）に向ける」「境界線を引く」も同じ考え方で、22章「クリーンアーキテクチャ」ではヘキサゴナル・アーキテクチャが元になった設計の1つとして挙がる。

---

## 第7章 単体テストの価値を高めるリファクタリング

### 要点

- 価値のあるテストを**見分ける**（4章）から、**書ける**ようにする章。プロダクション・コードを改善せずに、価値のある単体テストは生み出せない
- コードは「深さ（複雑さ・ドメインにおける重要性）」と「広さ（協力者オブジェクトの数）」で4種類に分けられ、テストの仕方がそれぞれ違う

| 種類 | 特徴 | Rails で言うと | テストの方針 |
| --- | --- | --- | --- |
| ドメイン・モデル／アルゴリズム | 深い・狭い | 料金計算や状態遷移を持つ model のメソッド、PORO | **単体テストで手厚く** |
| 取るに足らないコード | 浅い・狭い | `attr_accessor`、関連の定義だけ、単純な scope | **テストしない** |
| コントローラ | 浅い・広い | Controller、呼び出し順を並べるだけの Service | **統合テストで通しで確認**（第3部） |
| 過度に複雑なコード | 深い・広い | 業務ルールと DB 保存とメール送信が混ざった fat controller／fat service、callback だらけの model | 上の2つに**割る** |

- 重要で複雑なコードほど、協力者オブジェクトを減らす。「質の悪いテストを書くくらいなら、書かないほうがよい」
- 割るときの型が**質素なオブジェクト（Humble Object）**。テストしやすいロジックを抜き出し、テストしにくい依存（DB、UI、非同期など）との接点を、ロジックを持たない薄い包み紙にする。5・6章のヘキサゴナル・アーキテクチャ、関数型アーキテクチャ、MVC の Controller、DDD の集約はどれもこの適用例。深さと広さを1つのクラスに両方持たせない（クリーンアーキテクチャの SRP、23章「プレゼンターとHumble Object」とも繋がる）
- 本は、ドメインのクラスが自分で DB を読み書きする形を **Active Record パターン**と名指しし、「単純で短命なプロジェクトならうまくいくが、大きくなると破綻しやすい」としている。DB やメッセージ・バスをインターフェイスにしてモックを注入する直し方は**やらない**（プロセス外依存のままなので、テストが複雑なモックを抱え、壊れやすくなる）
- ユーザ管理システム（メールアドレスを変えると、自社ドメインなら従業員／それ以外は顧客になり、従業員数を増減し、外部に通知する）を3段階で割る

| 段階 | やること | 結果 |
| --- | --- | --- |
| 1回目 | DB とメッセージ・バスを触る部分をコントローラに移す | `User` がドメイン・モデルになる。代わりにコントローラが変換ロジックを抱える |
| 2回目 | DB の生データから `User` を作る変換を Factory に移す | コントローラが連携の指揮だけになる。**Rails では ActiveRecord（O/R マッパー）がこの役を担うので不要** |
| 3回目 | 会社のデータとロジックを `Company` に移す | 「尋ねるな、命じよ」の形になる |

- コントローラに分岐が要るときは、3つの性質のうち2つしか取れない

| 選択肢 | コントローラの簡潔さ | ドメインのテストのしやすさ | 性能 |
| --- | --- | --- | --- |
| A. 読み書きを全部、最初と最後に寄せる | ○ | ○ | **✕** |
| B. ドメインに DB などを渡す | ○ | **✕** | ○ |
| C. 決定を細かく分割する | **✕** | ○ | ○ |

本の選択は C。コントローラに入る分岐は、**確認後実行（CanExecute/Execute）パターン**と**ドメイン・イベント**で最小限にする（疑問②のコード）。

- 抽象化する対象より、抽象化された結果のほうがテストしやすい。ドメイン・イベントはメッセージ・バスへの送信の抽象化、ドメイン・クラスの状態変更は DB 更新の抽象化。本物の副作用は最後にコントローラがまとめて起こす
- 完璧には分けられない（メールアドレスの重複チェックや、外部呼び出しが失敗したときの判断はコントローラに残る）。それでも分ける価値は大きい
- 観察可能な振る舞いは**玉ねぎの層**。各層のテストは1つ上の層の視点で書き、下の層とのやり取りは確かめない

```
外部クライアント
   │ ① ChangeUserEmail#execute を呼ぶ     ← 外部にとっての観察可能な振る舞い（統合テスト）＋メッセージ・バスへの送信
   ▼
ChangeUserEmail（コントローラ）
   │ ② User#change_email を呼ぶ            ← コントローラにとっての観察可能な振る舞い（User の単体テスト）
   ▼
User
   │ ③ Company#corporate_email? などを呼ぶ ← User にとっての観察可能な振る舞い（Company の単体テスト）
   ▼
Company
```

---

### 疑問①：テストを考慮したアプリの設計が大事ってこと？

そう。ただし、テストのためだけに設計を変えろという話ではない。業務ルールと DB・外部とのやり取りを分けると、テストがしやすくなるだけでなく、本番のコードそのものも複雑になりにくくなる。著者も「テストしやすい設計が、そのまま保守しやすい設計にもなる」と書いている。1章の「テストが書きにくいなら設計が悪い」と同じで、テストの書きにくさは設計の問題を教えてくれる警報。

逆に、テストのためだけにプロダクション・コードに手を入れる（private にすべきものを公開する、テスト用の分岐を足す）のは、11章の「プロダクション・コードへの汚染」というアンチ・パターン。

最初の形（Rails で素直に書いた ActiveRecord）と、割ったあとの形。

```ruby
# ❌ 過度に複雑：業務ルールと DB アクセスとメッセージ送信が1つのメソッドに混ざっている
class User < ApplicationRecord
  def change_email(new_email)
    return if email == new_email

    company  = Company.first                                               # DB から読む
    new_type = new_email.split("@").last == company.domain_name ? "employee" : "customer"
    if user_type != new_type
      delta = new_type == "employee" ? 1 : -1
      company.update!(number_of_employees: company.number_of_employees + delta)   # DB に書く
    end

    update!(email: new_email, user_type: new_type)                          # DB に書く
    MessageBus.send_email_changed_message(id, new_email)                    # 外部に送る
  end
end
```

```ruby
# ドメイン：DB も外部も触らない。メモリ上で自分たちの状態を変えるだけ
class Company < ApplicationRecord
  def corporate_email?(email)
    email.split("@").last == domain_name
  end

  def change_number_of_employees(delta)
    raise ArgumentError if number_of_employees + delta < 0

    self.number_of_employees += delta
  end
end

class User < ApplicationRecord
  def change_email(new_email, company)
    return if email == new_email

    new_type = company.corporate_email?(new_email) ? "employee" : "customer"
    company.change_number_of_employees(new_type == "employee" ? 1 : -1) if user_type != new_type

    self.email = new_email
    self.user_type = new_type
  end
end

# コントローラ（アプリケーション・サービス）：読む・渡す・保存する・送る、だけ
class ChangeUserEmail
  def initialize(user_id:, new_email:)
    @user_id, @new_email = user_id, new_email
  end

  def execute
    user    = User.find(@user_id)
    company = Company.first

    user.change_email(@new_email, company)

    ActiveRecord::Base.transaction do
      company.save!
      user.save!
    end
    MessageBus.send_email_changed_message(user.id, @new_email)
  end
end
```

ActiveRecord を使っていても、**メソッドの中で DB に触らなければドメイン・モデルとして扱える**。テストは `build` だけで書けるので、2章の表で言う「`build` だけで完結する model spec ＝ 単体テスト」になる。

### 疑問②：何が大事かを考えることが大事？

そう。テストを書く前に「このコードは4種類のどれか」「この事前条件は業務上大事か」を考える。割ったあとのコードを4種類の表に当てはめると、単体テストを手厚くするのは左上だけになる。

| | 協力者オブジェクトが少ない | 協力者オブジェクトが多い |
| --- | --- | --- |
| **複雑・重要** | `User#change_email`、`Company#change_number_of_employees`、`Company#corporate_email?` | （リファクタリングで無くなった） |
| **単純・重要でない** | コンストラクタ → **テストしない** | `ChangeUserEmail#execute` → **統合テスト**（8章） |

```ruby
RSpec.describe User do
  it "顧客のメールアドレスを、従業員のものに変える" do
    company = build(:company, domain_name: "mycorp.com", number_of_employees: 1)
    sut = build(:user, email: "user@gmail.com", user_type: "customer")

    sut.change_email("new@mycorp.com", company)

    expect(company.number_of_employees).to eq 2
    expect(sut).to have_attributes(email: "new@mycorp.com", user_type: "employee")
  end

  it "従業員のメールアドレスを、顧客のものに変える"
  it "種類を変えずにメールアドレスを変える"
  it "同じメールアドレスに変える"
end
```

事前条件（バグがあったときに早めに落とす安全網、Fail Fast）も、ドメインにとって重要なものだけテストする。

| 事前条件 | ドメインにとって重要か | テストするか |
| --- | --- | --- |
| 従業員数がマイナスにならない（`Company` の不変条件） | 重要 | **する** |
| Factory に渡すデータが3要素以上ある | 重要ではない | しない |

```ruby
it "従業員数はマイナスにならない" do
  company = build(:company, number_of_employees: 0)

  expect { company.change_number_of_employees(-1) }.to raise_error(ArgumentError)
end
```

コントローラに分岐が要るときも、「何を判断するか」はドメインに残す。確認後実行パターンとドメイン・イベントで、コントローラは結果に従うだけにする。

```ruby
EmailChangedEvent = Data.define(:user_id, :new_email)   # 起きたことなので過去形。値として比較できる

class User < ApplicationRecord
  # 確認：変更できるかを判断する
  def can_change_email
    return "確定済みのメールアドレスは変更できません" if email_confirmed?

    nil
  end

  def email_changed_events
    @email_changed_events ||= []
  end

  # 実行：確認を通っていない呼び出しは受け付けない
  def change_email(new_email, company)
    raise ArgumentError, can_change_email if can_change_email
    return if email == new_email                          # 変わらなければイベントも出ない

    # ...種類と従業員数の更新
    email_changed_events << EmailChangedEvent.new(id, new_email)
  end
end

class ChangeUserEmail
  def execute
    user = User.find(@user_id)
    error = user.can_change_email
    return error if error               # 判断はドメインに任せ、結果に従うだけ

    company = Company.first             # 変更できるときだけ会社を読む（性能を落とさない）
    user.change_email(@new_email, company)
    # ...保存
    user.email_changed_events.each do |event|
      MessageBus.send_email_changed_message(event.user_id, event.new_email)
    end
  end
end

# テストはモック不要。イベントが溜まったかを確認するだけ
it "メールアドレスを変えると、変更イベントが記録される" do
  company = build(:company, domain_name: "mycorp.com", number_of_employees: 1)
  sut = build(:user, id: 1, email: "user@mycorp.com", user_type: "employee")

  sut.change_email("new@gmail.com", company)

  expect(sut.email_changed_events).to eq [EmailChangedEvent.new(1, "new@gmail.com")]
end
```

- DB への保存はメールアドレスが同じでもそのまま行う（アプリ専用の DB とのやり取りは実装の詳細。ActiveRecord は変更が無ければ UPDATE を発行しない）。メッセージ・バスへの送信は外部との契約なので、変わったときだけ送る
- Rails の `after_commit` コールバックでイベントを送る書き方は、ドメインが自分で外部に送っている形なので過度に複雑寄り。`activerecord-import` のようにコールバックを飛ばす経路で、イベントを補う専用のサービスが必要になる原因にもなる。ドメイン・イベントを溜めてサービスが送る形なら、一括取り込みの経路でも同じ送信処理を通せる
