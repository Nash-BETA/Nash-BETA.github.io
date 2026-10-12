---
title: "単体テストの考え方/使い方 - 8〜10章：第3部 統合（integration）テスト"
date: 2026-10-12 22:21:38 +0900
categories: [輪読, 単体テストの考え方/使い方]
tags: [単体テストの考え方/使い方]
---

# 第3部（8〜10章）まとめ

## ポイント

### 単体テストと統合テストは「何を確かめるか」で役割を分ける（8.1）

| | 何を確かめるか | 本数 | Rails で言うと |
| --- | --- | --- | --- |
| 単体テスト | ビジネス・シナリオの異常ケースをできるだけ多く | 多い | `build` で作る model / PORO の spec |
| 統合テスト | **最長のハッピー・パス**1本（すべてのプロセス外依存を通る正常系）＋ 単体テストで拾えない異常ケース | ビジネス・シナリオごとに1〜2本 | request spec |

- 単体テストはドメイン（ドメイン・モデル／アルゴリズム）を、統合テストはコントローラを確かめる
- 統合テストは遅くて保守コストが高い代わりに、実行するコードが多く（退行に対する保護が強い）、実装の詳細から遠い（リファクタリングへの耐性が高い）
- すぐに落ちる異常ケース（早期失敗、Fail Fast）は統合テストで確かめなくてよい。価値の無いテストを書くくらいなら、書かないほうがまし

### 統合テストでは、トランザクションをフェーズごとに分ける（10.2）

本の TIP は「統合テストでは少なくとも3つのトランザクションを使う」。テスト中にどこで BEGIN / COMMIT が走るかを SQL ログで並べると違いが見える。

```ruby
it "従業員のメールアドレスを、顧客のものに変える" do
  user = create(:user, email: "user@mycorp.com", user_type: "employee")   # 準備
  patch "/users/#{user.id}/email", params: { email: "new@gmail.com" }     # 実行
  expect(User.find(user.id).email).to eq "new@gmail.com"                   # 確認
end
```

本が言う形（transactional tests を切り、後始末は DatabaseCleaner などで行う）：

```
-- 準備：create(:user)
BEGIN; INSERT INTO users ...; COMMIT;            ← ① 準備のトランザクション。本当に DB に確定する

-- 実行：patch（コントローラの中の transaction do ... end）
BEGIN; UPDATE companies ...; UPDATE users ...; COMMIT;   ← ② 本番と同じ1本のトランザクション

-- 確認：User.find
SELECT * FROM users WHERE id = ...;              ← ③ 確定済みのデータを、別の読み込みで読む
```

Rails の標準（`use_transactional_fixtures = true`）：

```
BEGIN;                                           ← テストの最初に RSpec が張る（テスト全体を包む）
  INSERT INTO users ...;                         ← 準備：まだ確定していない
  SAVEPOINT active_record_1;                     ← 実行：コントローラの transaction は入れ子の SAVEPOINT になる
    UPDATE companies ...; UPDATE users ...;
  RELEASE SAVEPOINT active_record_1;             ← 本番なら COMMIT のところ。実際には確定していない
  SELECT * FROM users WHERE id = ...;            ← 確認：確定していないデータを、同じトランザクションの中から読む
ROLLBACK;                                        ← テストの最後に全部取り消す（後始末が不要になる）
```

準備・実行・確認がすべて1つのトランザクションの中で走り、一度も COMMIT されない。普通の CRUD ならほぼ問題は出ないが、コミットや接続の違いに依存する処理では本番と差が出る。

| 場面 | 何が起きるか |
| --- | --- |
| 別スレッドや別の DB 接続から読む（`Thread.new` の中の処理、読み取り専用レプリカへの接続など） | まだコミットされていないので、データが見えない |
| 同時実行の制御（`lock`、`SELECT ... FOR UPDATE`、一意制約の取り合い） | ロックの取り合いが起きず、本番の競合を再現できない |
| 確定のタイミングに依存する処理 | COMMIT が起きないので、「コミットされた後に外部に通知する」などの順序を確かめられない |

落とし所：普通の request spec は Rails の標準のままでよい。上の表に当たる処理をテストするときだけ、その spec で transactional tests を切る。

```ruby
RSpec.describe "在庫の引き当て（同時実行）", type: :request do
  self.use_transactional_tests = false   # この spec だけ本番と同じトランザクションの張り方にする
  before { DatabaseCleaner.clean_with(:deletion) }   # 後始末はテストの前に行う（10.3 の推奨）
  # ...
end
```


### モックは、外部へ出ていく経路の一番端をモックにする（9.1）

cam なら `UsersCompanyTraitUpdatedEvent.send!` ではなく `SnsPublisher.publish` をモックにして、実際に SNS に出ていく payload を確かめる。

```ruby
# ❌ 途中（イベント・クラス）をモックにしている：payload の組み立てを確かめていない
allow(UsersCompanyTraitUpdatedEvent).to receive(:send!)
# ...
expect(UsersCompanyTraitUpdatedEvent).to have_received(:send!).with(user, users_company_trait, old_trait)

# ✅ 境界（SnsPublisher）をモックにして、実際に SNS に出ていく中身を確かめる
allow(SnsPublisher).to receive(:publish)
# ...
expect(SnsPublisher).to have_received(:publish).with(
  anything,                                          # トピック名（設定値）
  "company_account_manager.UsersCompanyTraitUpdated",
  hash_including(cam_user_id: user.id, new_company_trait_id: new_trait.id),
).once
```

- 途中のクラスをモックにすると、payload の組み立てを通らない（①退行に対する保護が弱い）うえ、メソッド名を変えただけで落ちる（②リファクタリングへの耐性が弱い）
- 外部のシステムが期待しているのは「この中身が届くこと」で、どのメソッドが呼ばれたかではない。外から見える副作用は、出ていく中身そのものだけ
- 期待値のイベント名は、プロダクション・コードの定数ではなくリテラルで書く（定数が間違っていても通ってしまう「同義語反復」のテストを避けるため）

---

## 第8章 なぜ、統合（integration）テストを行うのか？

### 要点

- 統合テストは**コントローラ**を、単体テストは**ドメイン**を確かめる。役割分担（何を確かめるか・本数）は冒頭の「ポイント」参照
- すぐに落ちる異常ケースは統合テストで確かめなくてよい（**早期失敗、Fail Fast**）。問題が起きたらその場で止めれば、気づくのが速く、壊れたデータも保存されない

```ruby
# ❌ 設定が無くても nil のまま進み、どこかで想定外の動きになる
api_key = ENV["PAYMENT_API_KEY"]

# ✅ 設定が無ければ、その場で KeyError になる（起動時に読めば、そもそも立ち上がらない）
api_key = ENV.fetch("PAYMENT_API_KEY")
```

- プロセス外依存は「**管理下にある依存**」と「**管理下にない依存**」に分かれ、前者は本物を使い、後者だけモックにする（5.4 の「アプリからしか触らない DB はモックにしない」に名前が付いた）

| | 定義 | 例 | やり取りの性質 | 統合テストでの扱い |
| --- | --- | --- | --- | --- |
| **管理下にある依存**（managed） | このアプリだけが自由に扱える。外部はこのアプリの API を通してしか触れない | このアプリ専用の DB | 外から見えない＝実装の詳細 | **本物を使う**。最後の状態を確かめる |
| **管理下にない依存**（unmanaged） | このアプリが自由に扱えない。他のシステムにも副作用が見える | メール配信、メッセージ・バス、外部 API | 外から見える＝外部との契約 | **モックにする**。やり取りを確かめる |

| 依存 | 分類 |
| --- | --- |
| そのサービス専用の MySQL | 管理下にある → 本物（Rails のテスト DB） |
| メール配信、Slack 通知、決済 API | 管理下にない → モック |
| 別サービスの API（cam から fam や requl を呼ぶ、ws から cam の API で氏名を引く、など） | 管理下にない → モック |

- 他のアプリと共有しているテーブルは、テーブル単位で「管理下にない」扱いにする（1行を1件のメッセージとみなし、モックでやり取りを確かめる）。DMS で Redshift に複製されて分析側が読んでいる cam のテーブルはこれに近い。そもそも DB を共有してアプリ同士を繋ぐのは避け、API かメッセージ・バスで繋ぐ
- 本物の DB を使えないなら、DB をモックにした統合テストは**書かない**。確かめられるのは「どのメソッドを呼んだか」くらいで、単体テスト以上の守りにならない。ドメインの単体テストに集中する
- 7章のユーザ管理システムに統合テストを書くと、何を書くかは機械的に決まる

| 決めること | 判断 | 理由 |
| --- | --- | --- |
| どのシナリオを書くか | **従業員のメールアドレスを、顧客のものに変える**1本だけ | 副作用が一番多い＝最長のハッピー・パス |
| 「確定済みなので変更できない」異常ケースは？ | 書かない | 確認を呼び忘れても `change_email` が例外を投げて止まる（早期失敗） |
| DB は？ | **本物を使う** | 管理下にある依存 |
| メッセージ・バスは？ | **モックにする** | 管理下にない依存 |

```ruby
RSpec.describe "メールアドレスの変更", type: :request do
  it "従業員のメールアドレスを、顧客のものに変える" do
    # 準備：管理下にある DB には本物のデータを入れる
    company = create(:company, domain_name: "mycorp.com", number_of_employees: 1)
    user = create(:user, email: "user@mycorp.com", user_type: "employee")
    # 管理下にないメッセージ・バスだけをモックにする
    message_bus = instance_double(MessageBus, send_email_changed_message: nil)
    allow(MessageBus).to receive(:new).and_return(message_bus)

    # 実行：外部クライアントと同じ入口から呼ぶ
    patch "/users/#{user.id}/email", params: { email: "new@gmail.com" }

    # 確認：DB から読み直して、最後の状態を確かめる
    expect(response).to have_http_status(:ok)
    expect(User.find(user.id)).to have_attributes(email: "new@gmail.com", user_type: "customer")
    expect(Company.find(company.id).number_of_employees).to eq 0
    # 確認：外部との約束（メッセージの送信）を確かめる
    expect(message_bus).to have_received(:send_email_changed_message).with(user.id, "new@gmail.com").once
  end
end
```

確認フェーズで `user.reload` ではなく `User.find` で**読み直す**のがポイント。書き込みだけでなく、DB から読んで model に戻す経路まで一緒に確かめられる。

| | アプリの置き場所 | 管理下にある依存（DB） | 管理下にない依存（メッセージ・バス） |
| --- | --- | --- | --- |
| 統合テスト | テストと同じプロセス | 本物 | モック |
| E2E テスト | テスト環境にデプロイした別プロセス | 本物（確認はアプリの API 経由） | 本物 |

管理下にない依存だけをモックにした統合テストがあれば、E2E テストは必須ではない。デプロイ後の動作確認としていくつか足すのは問題ない。

- 実装クラスが1つしかないインターフェイスは**抽象ではない**

| よくある理由 | 著者の反論 |
| --- | --- |
| プロセス外依存を抽象化でき、疎結合になる | 実装が1つなら具象クラスと結合度は変わらない。抽象化は**発見する**もので、作り出すものではない。本当の抽象には実装が2つ以上要る |
| 将来の機能追加に備えられる（OCP） | 今要らない機能に時間を使うのは **YAGNI** に反する。コードは少ないほど良い |

| 依存 | インターフェイス |
| --- | --- |
| 管理下にない依存（メッセージ・バス、外部 API） | 作る（統合テストでモックにするため） |
| 管理下にある依存（アプリ専用の DB） | 作らない。具象クラスを渡す |
| ドメイン・クラス（`IUser` など） | **絶対に作らない**（ドメイン同士のやり取りをモックで確かめると壊れやすい） |
| 実装が本当に2つ以上ある | モックにするかに関係なく作ってよい |

Ruby にはインターフェイスが無く、`instance_double` で具象クラスからそのままテスト・ダブルを作れる。外部 API を包むクラス（`PaymentGateway` など）はモックにする境界として作る価値があるが、ActiveRecord を `UserRepository` で包んだり、`NotImplementedError` を投げるだけの抽象基底クラスを置いたりするのは、モックのためなら不要。

| | インターフェイスを置く理由 | 実装が1つのときに置くか |
| --- | --- | --- |
| クリーンアーキテクチャ（11章 DIP） | 上位の方針が下位の詳細に依存しないよう、依存の向きを内側に向けるため | 置く |
| この本 | モックを作るため | 管理下にない依存以外は置かない |

目的が違うので正面からの矛盾ではない（Martin は依存の向きとコンポーネントの独立性、Khorikov はテストの価値と YAGNI を優先）。Ruby はダックタイピングなので、この対立は C# や Java ほど表に出ない。

- 統合テストのベスト・プラクティス。ドメインの境界をはっきりさせ、層は**3つまで**にする

| 層 | 役割 | Rails で言うと |
| --- | --- | --- |
| ドメイン層 | ビジネス・ロジック | 業務ルールを持つ model や PORO |
| アプリケーション・サービス層 | 外部からの入口、ドメインとプロセス外依存の連携を指揮する | Controller、Service |
| インフラ層 | ドメインに属さないアルゴリズム、プロセス外依存へのアクセス | 外部 API のクライアント、メール送信のゲートウェイ、ActiveRecord の DB アクセス |

「コンピュータ科学のどんな問題も、もう1段の間接参照で解決できる。ただし、間接参照が多すぎるという問題は除く」（David J. Wheeler）。層が多いと1つの機能が全部の層に散らばり、テストも「層ごとに周りをモックにして少しだけ動かす」価値の低いものが増える。

- 循環依存は、コールバックをやめて値を返すことで断ち切る

```ruby
# ❌ 循環依存：呼ばれた側が、終わったら呼んだ側を呼び返す
class CheckOut
  def execute(order_id)
    GenerateReport.new.execute(order_id, self)   # 自分を渡す
  end
end

class GenerateReport
  def execute(order_id, check_out)
    # ...レポートを作ったら check_out を呼び返す
  end
end

# ✅ 値を返すだけにする
class CheckOut
  def execute(order_id)
    report = GenerateReport.new.execute(order_id)   # 結果は値（Report）で受け取る
    # ...
  end
end

class GenerateReport
  def execute(order_id)
    # ...Report を返す
  end
end
```

- 1つのテストに実行フェーズを複数置いてよいのは、外部の銀行のテスト環境がとても遅い・呼べる回数に上限がある、のように**プロセス外依存をテストのために自由に扱えない**ときだけ。実際には E2E テストでしか起きない
- ログも「誰が見るか」でテストするかが決まる

| | 誰が見るか | 例 | 性質 | テスト |
| --- | --- | --- | --- | --- |
| **サポート・ログ** | サポート担当、システム管理者など開発者以外 | 「ユーザー 12 の種類が従業員から顧客に変わった」 | 観察可能な振る舞い（業務上の要求） | **する**（モックでやり取りを確かめる） |
| **診断ログ** | 開発者だけ | 「change_email 開始」「change_email 終了」 | 実装の詳細 | しない |

サポート・ログは専用のクラスに業務の言葉で切り出し、ドメインからはドメイン・イベントで伝える（ドメインから直接呼ぶと過度に複雑になる）。

```ruby
UserTypeChangedEvent = Data.define(:user_id, :old_type, :new_type)

# ドメイン：イベントを溜めるだけ
class User < ApplicationRecord
  def change_email(new_email, company)
    # ...
    if user_type != new_type
      company.change_number_of_employees(new_type == "employee" ? 1 : -1)
      domain_events << UserTypeChangedEvent.new(id, user_type, new_type)
    end
    # ...
    domain_events << EmailChangedEvent.new(id, new_email)
  end
end

# サポート・ログ専用のクラス：業務の言葉でメソッドを用意する
class DomainLogger
  def user_type_changed(user_id:, old_type:, new_type:)
    Rails.logger.info({ event: "user_type_changed", user_id:, old_type:, new_type: }.to_json)
  end
end

# イベントを受けて、外部への送信先に振り分ける
class EventDispatcher
  def initialize(message_bus:, domain_logger:)
    @message_bus, @domain_logger = message_bus, domain_logger
  end

  def dispatch(events)
    events.each do |event|
      case event
      when EmailChangedEvent    then @message_bus.send_email_changed_message(event.user_id, event.new_email)
      when UserTypeChangedEvent then @domain_logger.user_type_changed(**event.to_h)
      end
    end
  end
end
```

| テスト | 確かめること |
| --- | --- |
| 単体テスト（`User`） | `UserTypeChangedEvent` が溜まったか |
| 統合テスト（request spec） | `DomainLogger` をモックにして、`user_type_changed` が呼ばれたか |

- 診断ログはドメインの中に書かない（読みにくくなり、ログもノイズだらけになる）。デバッグで一時的に入れたら消す。静的なメソッドからロガーを取ってくるのは「環境コンテキスト（ambient context）」というアンチ・パターンで、依存を明示的に渡すのが本の主張。Rails の `Rails.logger` はまさに環境コンテキストだが、全部を引数で渡すのは現実的ではないので、**テストしたいサポート・ログだけは専用のクラスに切り出す**くらいが落とし所

---

### 疑問①：cam だとどのディレクトリがどの層にあたりますか？

cam の `app/` 配下を本の3層に当てはめるとこうなる（反応：「わかりやすくていいね」）。

| 本の層 | cam のディレクトリ | 補足 |
| --- | --- | --- |
| **ドメイン層**（ビジネス・ロジック） | `app/models`、`app/validators` | model は ActiveRecord なので DB アクセスも同居。イベント送信や外部 API 呼び出しを直接書いている model が8ファイルあり、本の分類では「過度に複雑なコード」寄り |
| **アプリケーション・サービス層**（入口・連携の指揮） | `app/controllers`、`app/apis`（Grape）、`app/handlers/sqs_handler`（SQS からの入口）、`app/jobs`、`app/services`、`app/forms` | `services` は混在。多くは連携の指揮だが、業務ロジックを持つものもある（ドメイン層に寄せる候補） |
| **インフラ層**（プロセス外依存へのアクセス） | `app/repositories`、`app/resources`、`app/events`、`app/mailers`、`app/uploaders`、`app/queries` | 下の表 |
| 3層の外（表示） | `app/serializers`、`app/view_models`、`app/decorators`、`app/helpers` | アプリケーション・サービス層の出力の形を整えるもの |

| ディレクトリ | 中身 | 依存の種類 | 統合テストでの扱い |
| --- | --- | --- | --- |
| `repositories` | `finc_app_repository`、`wellness_survey_repository`、`mission_server_repository`、`sns_repository`、`redis_connection` など | ほぼ**管理下にない**（他サービスの API、SNS） | モック |
| `resources` | finc_app への HTTP クライアント | 管理下にない | モック |
| `events` | `UsersCompanyTraitUpdatedEvent.send!` のように SNS に直接 publish するクラス | 管理下にない | モック |
| `mailers` | メール送信 | 管理下にない | モック（Rails なら `deliveries` で確認） |
| `queries` | DB の検索オブジェクト | **管理下にある**（cam 専用の MySQL） | 本物の DB |

cam ならではの点が2つある。

- **`repositories` の意味が本と違う**：本の「リポジトリ」は DB アクセスのことだが、cam の `repositories` は他サービスや AWS へのクライアントが中心。本の分類では「管理下にない依存へのゲートウェイ」なので、統合テストでモックにしてよい境界
- **`events` は本のドメイン・イベントとは別物**：本のドメイン・イベントは「起きたことを表す値」で、ドメインが溜めてコントローラが送る。cam の `events` は呼ばれた時点で SNS に送るクラスで、それを model のコールバックから直接呼んでいる箇所がある。`activerecord-import` でイベントを補うサービスが必要になった件は、この構造から来ている

層の数は本の3層におおむね収まっていて、層の重ねすぎは起きていない。手を入れる余地があるのは、ドメイン層に外部への送信が混ざっている model（8ファイル）と、業務ロジックを持っている一部の `services` の2点。

### 疑問②：統合テストではモックを使うべきなの？これは何派の主張？

統合テストでもモックは**原則使わず、管理下にない依存（外部サービス）だけに限って使う**。古典学派をベースに、著者が一歩進めた考え方。

| | 単体テストでテスト・ダブルにするもの | 統合テストでモックにするもの |
| --- | --- | --- |
| ロンドン学派 | 依存しているクラスほぼすべて | （単体テストの段階でほぼ全部モックにしている） |
| 古典学派 | 共有依存（DB・メール・外部 API などのプロセス外依存） | プロセス外依存全般になりがち |
| **この本（著者）** | ドメインのテストではモックを使わない（7章で外部に触れない形に割っているので不要） | **管理下にない依存だけ**。アプリ専用の DB は本物 |

著者は古典学派を支持しつつ、5.4 で「古典学派でもプロセス外依存をすべてモックにするのは使いすぎ」と書き、8章で「管理下にある依存／管理下にない依存」という線を引き直した。

- 外部とのやり取り（管理下にない依存）は、外部との約束なので**モックで確かめるべき**
- それ以外（アプリ専用の DB、アプリ内のクラス同士）は**モックにすべきではない**

### 疑問③：統合テストでの扱いってどう言う意味？

request spec などの統合テストを書くときに、その依存を**本物のまま動かすか、モックに差し替えるか**。どちらにするかで、最後に何を確かめるかも変わる。

| 扱い | テストの中でやること | 最後に確かめるもの |
| --- | --- | --- |
| **本物** | テスト用の DB にそのまま読み書きさせる | 処理のあとの**状態**（DB を読み直して中身を見る） |
| **モック** | `instance_double` などで差し替えて、本当には送らせない | **やり取り**（正しい引数で送ろうとしたか） |

```ruby
# DB（管理下にある依存）→ 本物：実際に保存させて、読み直して確かめる
user = create(:user, email: "user@mycorp.com", user_type: "employee")
# ...
expect(User.find(user.id)).to have_attributes(email: "new@gmail.com")

# メッセージ・バス（管理下にない依存）→ モック：差し替えて、送ろうとしたかを確かめる
message_bus = instance_double(MessageBus, send_email_changed_message: nil)
allow(MessageBus).to receive(:new).and_return(message_bus)
# ...
expect(message_bus).to have_received(:send_email_changed_message).with(user.id, "new@gmail.com")
```

cam の request spec で「ユーザーの部署を変更する」画面を叩く場合なら、cam の MySQL への保存は本物のまま動かして `UsersCompanyTrait` を読み直し、`UsersCompanyTraitUpdatedEvent`（SNS への送信）や `FincAppRepository`（他サービスの API）はモックに差し替えて「送ろうとしたか」を確かめる。

### 疑問④：ドメイン層＝model だけど、Rails だと model が DB のデータみたいな振る舞いをするから勘違いするよね

同じ「モデル」でも、Rails と本では指しているものがずれている。

| | Rails の「model」 | 本の「ドメイン・モデル」 |
| --- | --- | --- |
| 何を表すか | DB のテーブルの1行（`app/models` の ActiveRecord クラス） | 業務の知識・ルール |
| DB との関係 | DB と一体。読み書きもこのクラスがやる | DB を知らない。保存はコントローラが外でやる |
| 置き場所 | `app/models` | 場所は問わない（model のメソッド、PORO、ロジックを持つ一部の service など） |

本の物差しで見ると、`app/models` の中身は3種類に分かれる。

- カラムと関連を定義しているだけの model → **取るに足らないコード**
- `change_email` のように業務ルールを持ち、DB にも外部にも触れないメソッド → **ドメイン・モデル**
- コールバックで SNS に送ったり、メソッドの中で `save!` や外部 API を呼んだりする model → **過度に複雑なコード**

「`app/models` ＝ ドメイン層」ではなく、「`app/models` の中の、DB にも外部にも触れずに業務ルールを判断しているメソッドがドメイン層」と読む。クリーンアーキテクチャ20章の「エンティティ」もほぼ同じもので、Martin も Khorikov も業務ルールを DB から切り離して考える点は共通している。

---

## 第9章 モックのベスト・プラクティス

### 要点

- 8章の「管理下にない依存だけをモックにする」は道のりの約2/3。残りは「**外部へ出ていく経路の一番端（アプリの境界）のクラスをモックにし、実際に外へ出ていく中身を確かめる**」

```
コントローラ → EventDispatcher → MessageBus（業務の言葉でメッセージを組み立てる）→ Bus（SDK を包むだけ）→ 外部
                                  ↑ 8章ではここをモック                            ↑ 9章ではここをモック
```

| モックにする場所 | 確かめるもの | ①退行に対する保護 | ②リファクタリングへの耐性 |
| --- | --- | --- | --- |
| `MessageBus`（途中） | `send_email_changed_message` が呼ばれたか | 低い | 低い（メソッド名を変えると落ちる） |
| **`Bus`（境界）** | **実際に送られる文字列** | 高い | 高い（送る中身が同じなら落ちない） |

cam に置き換えると、`UsersCompanyTraitUpdatedEvent.send!`（途中）ではなく、最後に SNS に送る `SnsPublisher.publish`（境界）をモックにする（コードは冒頭のポイント参照）。

- 境界のテスト・ダブルは手書きのモック（**スパイ**）にすると、確認用のメソッドを持たせられて読みやすい。RSpec の `spy` はフレームワークが作るので、本の定義ではモック

```ruby
# テスト・コード側に置く手書きのモック（スパイ）
class SnsPublisherSpy
  def initialize = @published = []

  def publish(topic, event_name, data)
    @published << { topic:, event_name:, data: }
  end

  def should_publish_number_of_messages(number)
    expect(@published.size).to eq number
    self
  end

  def with_users_company_trait_updated(user_id:, new_trait_id:)
    expect(@published).to include(hash_including(
      event_name: "company_account_manager.UsersCompanyTraitUpdated",
      data: hash_including(cam_user_id: user_id, new_company_trait_id: new_trait_id),
    ))
    self
  end
end

spy.should_publish_number_of_messages(1)
   .with_users_company_trait_updated(user_id: user.id, new_trait_id: new_trait.id)
```

- **テストはプロダクション・コードを信用しない**。スパイ（テスト・コード）で送られる中身を自分で確かめる。期待値にプロダクション・コードの定数（`UsersCompanyTraitUpdatedEvent::USERS_COMPANY_TRAIT_UPDATED_EVENT`）を使うと、定数が間違っていても通る「同義語反復（tautology）」のテストになるので、リテラルを複製して書く
- ログだけは例外で、`DomainLogger`（途中）をモックにしてよい。見るのは人なので、文字列の形より「出たこと」と「含まれる情報」が大事
- 残りのベスト・プラクティス

| ベスト・プラクティス | 理由 |
| --- | --- |
| モックは統合テストだけで使う | 管理下にない依存を扱うのはコントローラだけなので、単体テスト（ドメイン）にモックの出番は無い |
| 1つのテストにモックが複数あってよい | 「1テスト1モック」は単体を「1単位のコード」と捉える誤解から来ている。数は呼ばれる管理下にない依存の数で決まる |
| モックの呼び出し回数を必ず確かめる | 後方互換には「期待した呼び出しがあること」と「期待していない呼び出しが無いこと」の両方が要る |
| モックにするのは自分のプロジェクトが持つ型だけ | ライブラリの型は直接モックにせず、自前のアダプタ（腐敗防止層）を挟む。cam の `SnsPublisher` がこれ。管理下にない依存だけで十分で、時刻ライブラリや ActiveRecord まで包む必要は無い |

```ruby
# ❌ 呼ばれたかしか見ていない：2回送っていても、別のメッセージが余計に出ていても通る
expect(SnsPublisher).to have_received(:publish).with(anything, "company_account_manager.UsersCompanyTraitUpdated", anything)

# ✅ 1回だけ、かつ他に何も送っていないことまで確かめる
expect(SnsPublisher).to have_received(:publish).once
expect(SnsPublisher).to have_received(:publish)
  .with(anything, "company_account_manager.UsersCompanyTraitUpdated", anything).once
```

外部の HTTP API なら、WebMock でリクエストそのもの（URL・ボディ）を確かめる方法もある。型をモックにするのではなく、実際に出ていく HTTP を確かめる形で、「境界の一番端で中身を確かめる」をさらに推し進めたものと見ることもできる。

---

## 第10章 データベースに対するテスト

### 要点

- DB を本物として使うための準備は、Rails ではほぼ標準で揃っている

| 本の準備 | Rails では |
| --- | --- |
| スキーマを Git などで管理する | `db/migrate` と `db/schema.rb` を Git で管理 |
| 参照データ（アプリが動くのに必須のマスタ）もスキーマと一緒に管理する | `db/seeds.rb`。ただしテスト DB には自動では入らない |
| 開発者ごとに DB を持つ | Docker の MySQL をそれぞれのマシンで動かす |

- 本番への反映は「状態ベース」ではなく「**移行ベース**」。リリース後は既存データの変換（`name` を `first_name` と `last_name` に分けるなど）が必要で、比較ツールには推測できないから。Rails のマイグレーションがこれ。移行の結果が間違っていても DB を直接直さず、取り消す移行を積んでからやり直す
- Rails には `db/migrate`（移行ベース）と `db/schema.rb`（状態のスナップショット）の両方がある。古いマイグレーションが今のモデルのコードを参照して空の DB から再生できなくなったときは、`db:schema:load` や STG のダンプで今の状態を入れて、差分だけ移行する
- 本番では「1つのビジネス・オペレーション＝1つのトランザクション」にして更新をアトミックにする。「何を変えるか」（リポジトリ。Rails では model の `save!`）と「確定するか」（トランザクション。Rails では `ActiveRecord::Base.transaction`）を分け、確定の判断はコントローラが最後に下す。さらに良いのは変更を溜めて最後にまとめて書く**単位作業（Unit of Work）**だが、ActiveRecord には無い
- テストでは準備・実行・確認で別々のトランザクションにする（冒頭のポイント参照）
- 統合テストは並列にせず1つずつ流す。テストごとにコンテナを立てる並列化はコストが高すぎる。Rails の `parallelize` や parallel_tests はワーカーごとに DB を分ける方式なので、このコストはかからない
- 後始末は「**テストの前に消す**」が一番良い

| 方法 | 良いところ | 問題 | Rails で言うと |
| --- | --- | --- | --- |
| テストのたびにバックアップから復元 | 確実 | 遅い | — |
| テストの後に消す | 速い | 途中で止まると消されない | DatabaseCleaner を `after` で |
| 1つのトランザクションで包んでロールバック | 自動で消える | 本番と違うトランザクションの張り方になる | **Rails の transactional tests（標準）** |
| **テストの前に消す** | 速い・本番と同じ・消し忘れが無い | — | DatabaseCleaner を `before` で `deletion` / `truncation` |

参照データは消さない（DatabaseCleaner なら `except:` で除外する）。

- SQLite などのインメモリ DB で代用しない。ベンダー（MySQL か PostgreSQL か）は本番と揃える。utf8mb4 の照合順序の違いで重複判定が変わるような挙動は、本物でないと確かめられない
- 統合テストのコードは、フェーズごとに技術的なコードをヘルパーに追い出して短くする

| フェーズ | 本の方法 | Rails で言うと |
| --- | --- | --- |
| 準備 | オブジェクト・マザー（既定値付きのファクトリ・メソッド） | FactoryBot。そのテストに効く値だけ指定する |
| 実行 | コントローラの呼び出しを包むメソッドで、実行専用のトランザクションを張る | request spec の `patch` が最初からこの役 |
| 確認 | DB から読み直すヘルパーと、流れるように書ける確認メソッド | `User.find(...)` と `have_attributes` |

```ruby
it "従業員のメールアドレスを、顧客のものに変える" do
  user = create(:user, email: "user@mycorp.com", user_type: "employee")   # 準備：効く値だけ指定
  create(:company, domain_name: "mycorp.com", number_of_employees: 1)

  patch "/users/#{user.id}/email", params: { email: "new@gmail.com" }     # 実行

  expect(User.find(user.id)).to have_attributes(email: "new@gmail.com", user_type: "customer")   # 確認
  expect(Company.first).to have_attributes(number_of_employees: 0)
end
```

ヘルパーはまず同じ spec ファイルの中に置き、重複が目立ったら `spec/support` に移す。基底クラス（`rails_helper` の共通設定）には、全テストで必ず実行するもの（後始末など）だけを置く。ヘルパーに分けるとトランザクションの数が増えて少し遅くなるが、保守のしやすさを優先してよい。

- **書き込みは徹底的に、読み込みは複雑・重要なものだけ**テストする。読み込みはバグの害が小さく、ドメイン・モデルも要らない（カプセル化は変更の整合性を守るためのもの）。テストするなら本物の DB で。cam なら `app/queries/users/search_query.rb` のような検索はテストする価値があり、1件返すだけの `show` はテストしなくてよい
- **リポジトリは単独でテストしない**。複雑でないのにプロセス外依存を扱う「コントローラ」で、確かめる中身は統合テストと重なる。統合テストのシナリオの中で間接的に確かめる。`EventDispatcher` も同じ。Rails で言えば、関連や単純な scope を1つずつ確かめる model spec はこれに近い
- 本物の DB を使った統合テストがあれば、DB とのやり取りのリファクタリング、O/R マッパーへの移行、DB ベンダーの変更でも、テストの数行を直すだけで成功を確信できる

---

### 疑問①：「統合テストでは少なくとも3つのトランザクションを使う」が、Rails だとイメージが湧かない

テスト中にどこで BEGIN / COMMIT が走っているかを SQL ログで並べると分かる（冒頭のポイントに全体を載せている）。

- 本が言う形：`create` の COMMIT、コントローラの `transaction` の COMMIT、確認の `SELECT` がそれぞれ別。コントローラは本番と同じ条件で動き、確認はコミットされたデータを読む
- Rails の標準（transactional tests）：テスト全体を1つの BEGIN 〜 ROLLBACK で包むので、コントローラの `transaction` は SAVEPOINT になり、一度も COMMIT されない

普通の CRUD ならほぼ問題は出ないが、別スレッドや別の DB 接続から読む処理、ロックや一意制約の取り合い、コミット後に外部へ通知する処理では本番と差が出る。普通の request spec は Rails の標準のままにして、そうした処理をテストするときだけ、その spec で transactional tests を切る。
