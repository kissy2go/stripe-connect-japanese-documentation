# マネージド・アカウント認証のテスト

Connect のマネージド・アカウントの異なる検証状況のテストチュートリアルでは、あなたのテスト用の API キーを利用します。

-----

このドキュメントは、あなたが[アカウントの更新](https://stripe.com/docs/connect/updating-accounts)や[本人認証](https://stripe.com/docs/connect/identity-verification)と共に、全般的なマネージド・アカウントに精通していることを前提としています。

あなたのプラットフォームはマネージド・アカウントの本人認証プロセスに関する責任を持っている。
マネージド・アカウントのインクリメンタルに検証できるようにするため、Connect はこのプロセスを効率化しています。
あらかじめ全ての情報が必要になる代わりに、Connect は最低限の初期情報しか必要としないが、アカウントが利用できることは限られます（通常、一定の限度を超えての支払いを遮断します）。
アカウントがより多くの情報をストライプに提供することで、我々はこれらの制限を緩和します。

この検証のインクリメンタルモデルは、ステージに分割されています。
マネージド・アカウントは、一定の売買高をこなすと、ステージは "triggered" となり、Stripe がこのステージにおける必須情報を必要としていることを意味します。
一度トリガーされると、あなたは限られた時間内と追加料金高内に、要求された情報を提供しなければなりません。
この情報を提供すると、ステージは完了となります。
私たちはあなたの不便を最小限に抑え、ステージが鳥がされる前に必要な情報を提供することを可能にしています。

![connect_verification_diagram.png](https://stripe.com/img/connect/testing-verification/connect_verification_diagram.png)

上図は、米国におけるマネージド・アカウントのインクリメンタル検証に関わるステージを示しています。
それぞれの国で必要な情報は異なることに注意してください。

テストモードでは、ことなるステージ間への遷移は、[特別なクレジットカード番号](https://stripe.com/docs/connect/testing#trigger-cards)を利用してください。

それでは、手動で新しいマネージド・アカウントの検証プロセスを経ていきましょう。
一連のコマンドを順次実行することをお勧めします。
複数のステップを同時に完了させることに注意してください: 例えば、名前、生年月日、銀行口座、規約の同意と共にアカウントを作成することができます。
これにより、ただちに最初の検証ステージをバイパスし、このフローのステップ 5 まで飛ぶことができます。

## 1. マネージド・アカウントの作成

新しいテストモードのマネージド・アカウントを作成し、銀行口座の追加、マネージド・アカウント所有者が Stripe の利用規約（ToS）に同意したことを示すところからはじめましょう。
明示的に ToS の同意は、[送金をするために必要](https://stripe.com/docs/connect/updating-accounts#tos-acceptance)です。

```ruby
# Set your secret key: remember to change this to your live secret key in production
# See your keys here https://dashboard.stripe.com/account/apikeys
Stripe.api_key = "sk_test_BQokikJOvBiI2HlWgH4olfQ2"

acct = Stripe::Account.create(
  {
    :country => 'US',
    :managed => true
  }
)

acct.external_accounts.object = 'bank_account'
acct.external_accounts.country = 'US'
acct.external_accounts.currency = 'usd'
acct.external_accounts.routing_number = '110000000'
acct.external_accounts.account_number = '000123456789'
acct.tos_acceptance.date = 1464063447
acct.tos_acceptance.ip = 126.17.124.231
acct.save

acct_id = acct.id
puts acct_id
```

## 2. 最初の検証ステージ

![nostate.png](https://stripe.com/img/connect/testing-verification/nostate.png)

最初は、どのステージもアクティブではありません。
この時点で課金を作成することはできますが、資金を送金することはできません。
これはすべてのマネージド・アカウントのはじめの状態です。

さて、テストカード番号（`4000 0000 0000 4202`）を使って、アカウントの最初のステージをトリガしましょう。
この魔法のクレジットカードは、ステージのトリガの閾値を超えることをシミュレートします。
一般的に、これは一定の全課金ドル額を超えたあとに発生します。

```ruby
Stripe::Charge.create(
  :amount => 1000,
  :currency => "usd",
  :source => {
    :object => "card",
    :number => 4000000000004202,
    :exp_month => 2,
    :exp_year => 2017
  },
  :destination => acct_id
)

# Re-fetch the account to see what its status is.
Stripe::Account.retrieve(acct_id)
```

`verification[fields_needed]` で要求されている追加情報に加え、`verification[due_by]` フィールドにこの情報を提供する締め切りが格納されていることを確認してください。

![stage1a_triggered.png](https://stripe.com/img/connect/testing-verification/stage1a_triggered.png)

これは最初のステージがトリガされたことに起因します。
このステージでは、単に初期状態と同じように課金は有効になっていますが、送金は無効になっています。
アカウントの `charges_enabled` がまだ `true` であることを確認してください: Stripe は情報を提供する期限を通知しましたが、まだこれをパスしていません。

## 3. 課金上限のトリガ

このステージでは最初の課金を可能にしているが、このステージでは作成できる課金の取扱高には上限があります。
その上限を超えることをシミュレートするには、他の[トリガするカード番号](https://stripe.com/docs/connect/testing#trigger-cards)（`4000 0000 0000 4210`）で課金を作成してください。

```ruby
Stripe::Charge.create(
  :amount => 1000,
  :currency => "usd",
  :source => {
    :object => "card",
    :number => 4000000000004210,
    :exp_month => 2,
    :exp_year => 2017
  },
  :destination => acct_id
)
```

![stage1b_triggered.png](https://stripe.com/img/connect/testing-verification/stage1b_triggered.png)

これにより、新しく課金を作成することができなくなり、アカウントの `charges_enabled` が `false` で `verification[disabled_reason]` が `fields_needed` の両方を表示するように更新されました。

## 4. 最初のステップを満たす

このステージでどのような情報が提供する必要があるのかを知るには、最初に必要なフィールドのリストを取得する必要があります。

```ruby
puts(Stripe::Account.retrieve(acct_id))
```

米国では、これらの `verification[fields_needed]` は `first_name` と `last_name`、`dob` です。

Stripe は、当社の [OFAC](https://www.treasury.gov/about/organizational-structure/offices/Pages/Office-of-Foreign-Assets-Control.aspx) 要件を満たすために、このフィールドの初期セットを収集する必要があります。
これらのチェックの特性上、これらが提供されないと Stripe はアカウントへの課金をブロックしなければならない。
名前と生年月日をを収集する前でも一定の金額額まで課金できるが、課金のブロックはビジネスにとってとても破壊的であるので、事前にこれらの潜在的な問題を回避するため、私たちはこれらのフィールドの情報収集を強く推奨します。

```ruby
account = Stripe::Account.retrieve(acct_id)
account.legal_entity.dob.day = 10
account.legal_entity.dob.month = 01
account.legal_entity.dob.year = 1986
account.legal_entity.first_name = "Jonathan"
account.legal_entity.last_name = "Goode"
account.legal_entity.type = "individual"
account.save
```

![stage1_resolved.png](https://stripe.com/img/connect/testing-verification/stage1_resolved.png)

これらの情報を提供すると、Stripe はただちに課金を再度有効にするとともに、送金も有効にします。
また、`verification[fields_needed]` はまだセットされており（違う値で）、`verification[due_by]` はセットされてないことに気づくでしょう。
これは、Stripe はただちに必要ではないが将来的にさらなる情報が必要なことを意味しており、アカウントがさらに課金するまで、それらの情報の提供を延期することができることも意味しています。

## 5. 第 2 ステージ

先に進んで、閾値をトリガさせる魔法のカードを使って、次のステージをトリガしてみましょう。

```ruby
Stripe::Charge.create(
  :amount => 1000,
  :currency => "usd",
  :source => {
    :object => "card",
    :number => 4000000000004202,
    :exp_month => 2,
    :exp_year => 2017
  },
  :destination => acct_id
)

# Re-fetch the account to see what its status is.
puts(Stripe::Account.retrieve(acct_id))
```

![stage2a_triggered.png](https://stripe.com/img/connect/testing-verification/stage2a_triggered.png)

以前と同じように課金と送金はできますが、アカウントの `verification[due_by]` がセットされており、近い将来追加の情報の提供が必要になることを意味しています。

このステージで起こる最悪のことは、Stripe が送金を停止することです（最初のステージに戻り、Stripe は検証されるまで再び課金を無効にするだけです）。

## 6. 送金上限のトリガ

このステージでは課金が無効にされる上限がない一方、送金が無効にされる上限があります。
トリガさせるカード番号（`4000 0000 0000 4236`）を使って、送金上限をトリガさせることができます。
これは課金上限と同様ですが、送金をブロックします。

```ruby
Stripe::Charge.create(
  :amount => 1000,
  :currency => "usd",
  :source => {
    :object => "card",
    :number => 4000000000004236,
    :exp_month => 2,
    :exp_year => 2017
  },
  :destination => acct_id
)
```

![stage2b_triggered.png](https://stripe.com/img/connect/testing-verification/stage2b_triggered.png)

このトリガの後は、すべての送金は失敗します。

```ruby
Stripe::Transfer.create(
  {
    :amount => 400,
    :currency => "usd",
    :recipient => "self",
  },
  {
    :stripe_account => acct_id
  }
)
```

## 7. 第 2 ステージを満たす

```ruby
account = Stripe::Account.retrieve(acct_id)
account.legal_entity.address.line1 = "3180 18th St"
account.legal_entity.address.postal_code = 94110
account.legal_entity.address.city = "San Francisco"
account.legal_entity.address.state = "CA"
account.legal_entity.ssn_last_4 = 1234
account.save
```

![stage2_resolved.png](https://stripe.com/img/connect/testing-verification/stage2_resolved.png)

このステージを満たした後、再び送金が昨日するようになります。

```ruby
Stripe::Transfer.create(
  {
    :amount => 400,
    :currency => "usd",
    :recipient => "self",
  },
  {
    :stripe_account => acct_id
  }
)
```

## 8. 複数のステージを満たす

先に述べたように、正しい情報を与えることで、複数のステージを満たすことができます。
1 度でのこりの 2 つのステージを完了させてみましょう。

まず、`wget https://stripe.com/img/documentation/guides/testing/success.png` を実行して[テストが成功する画像](https://stripe.com/docs/connect/testing#identity-verification)を入手します。

ここから、[テストが成功する画像をアップロード](https://stripe.com/docs/connect/identity-verification#uploading-a-file)することができます。

```ruby
file_obj = Stripe::FileUpload.create(
  {
    :purpose => 'identity_document',
    :file => File.new('success.png')
  },
  {
    :stripe_account => acct_id
  }
)
file = file_obj.id
```

画像がアップロードされると、アカウントの検証情報の両方のピースをアップデートできます。

```ruby
account = Stripe::Account.retrieve(acct_id)
account.legal_entity.personal_id_number = 123456789
account.legal_entity.verification.document = file
account.save
```

![stage4_resolved.png](https://stripe.com/img/connect/testing-verification/stage4_resolved.png)

第 2 ステージを満たした状態から、実際に 1 回で 2 つのステージを飛び越し、完了させることができました。
アカウントの GET 呼び出しで、検証状況を確認できます。

```ruby
puts(Stripe::Account.retrieve(acct_id))
```

`verification[fields_needed]` が空になっていることがわかり、重大な例外（例えば、アカウントが詐欺目的で利用していたことが発覚した等）が発生しない限り、Stripe がこのアカウントに関してもう追加の情報を必要としていないことを意味します。
