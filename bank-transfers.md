# 銀行送金

あなたが管理する任意の Stripe アカウントにたいして、送金スケジュールの設定やどのような送金を送信するのかのすべての制御を変更することができます。

-----

マネージドもしくはスタンドアロンの任意の連携アカウントにたいして送金や課金をするとデフォルトで、連携アカウントの Stripe の残高が蓄積され、毎日の繰り返し基準にもとづいて支払いが行われる。
しかしながら、マネージド・アカウントのために、Stripe はきめ細かな動作を制御する機能を提供しています。

マネージド・アカウントにたいして、Stripe はあなたに以下のことを可能にしています:

- 支払いを行う銀行口座もしくはデビットカード内容のアカウント設定
- 資金が払い出される頻度の制御
- Stripe がマイナス残高のアカウントのハンドルをどのように行うかの決定

デビットカードへの送金は 3,000 USD が上限であり、受取人のカードがプリペイド式でない US Visa もしくは MasterCard のデビットカードでなければならないことに注意してください。

## 銀行口座

マネージド・アカウントは、資金の宛先となる可能性のある銀行口座と Stripe アカウントに関連付けられたデビットカードのすべてのリストを `external_accounts` プロパティとして保持しています。

```json
  "external_accounts": {
    "object": "list",
    "total_count": 0,
    "has_more": false,
    "url": "/v1/accounts/acct_14qyt6Alijdnw0EA/external_accounts",
    "data": []
  },
```

この属性は、`external_accounts` を通してアカウントの作成もしくは更新の際に追加することができる。
値は [Stripe.js](https://stripe.com/docs/stripe.js#collecting-bank-account-details) から返却されたデビットカードもしくは銀行口座のトークンになります。
代わりに、銀行口座の詳細情報を直接与えることができます（機密データがあなたのサーバーに送信されるように、あまり理想的ではありません）。

## 複数の口座情報

デフォルトでは、マネージド・アカウントの更新とともに `external_accounts` に新しい値が与えられると、元の口座情報は新しいものに置き換えられます。
連携アカウントに追加のデビットカードや銀行口座を追加するには、[銀行口座](https://stripe.com/docs/api#account_create_bank_account)と[カード登録](https://stripe.com/docs/api#account_create_card) の API エンドポイントを利用してください。

特定の通過で料金が支払われたときにはデフォルトで、Stripe は銀行口座かデビットカードにその通貨で支払いを行い、それによって手数料の両替を回避しています。
その通貨で複数の口座が登録されている場合、`default_for_currency` が設定されているいずれかを選択します。

Stripe は
あなたの参考になる、あなたのユーザーがオプションを選択しやすくするための手助けになる[利用可能な国/通過の組み合わせ](https://stripe.com/docs/connect/required-verification-information)のリストをメンテナンスしています。

## 支払い情報

アカウントの `transfer_schedule` プロパティは、このアカウントの Stripe 残高が彼らの銀行口座に支払われる頻度を示しています:

```json
  "transfer_schedule": {
    "delay_days": 7,
    "interval": "daily"
  },
```

`delay_days` は、このアカウントで作成された課金が支払い可能になるのはいつかという情報になります。
このフィールドは通常、自動支払の制御のためのものです。
例えば、マネージド・アカウントが課金が作成されてから 2 週間後に市kんを受け取れるようにしたいのなら、`interval` に `daily` をセットし、このフィールドには `14` を設定してください。
デフォルトは、彼らの国によって定められている、口座の許容される最低値になっています。
許容される最低値を選択したい場合、このフィールドを設定もしくは更新する際に `minimum` という文字列を渡してください。

`interval` というサブフィールドは 4 つの設定が可能になっています:

- `manual` は任意の自動支払を止めます。あなたは[送金 API](https://stripe.com/docs/api#create_transfer)（連携アカウントとして振る舞う）を使ってアカウントの残高から手動で支払いをする必要があります。
- `daily` は課金されてから `delay_days` 後に、自動的にその課金（もしくはこの口座に関連付けられた送金）の支払いを行います。この `delay_days` の値は、あなた自身の送金スケジュール以下、もしくはアカウントの国でデフォルトの送金スケジュール以下にすることはできません。
- `weekly` は週に一度 `weekly_anchor` パラメータ（`monday` のような小文字の曜日）で指定されている曜日に、自動的に残高から支払いを行います。
- `monthly` は月に一度 `monthly_anchor` パラメータ（1-31 の数字）で指定された日に、自動的に残高から支払いを行います。

## 手動送金を使う

`transfer_schedule[interval]` に `manual` を設定した場合、銀行口座に残高を支払うように指示されるまで（もしくは最大で 30 日経過するまで）、アカウント保有者の残高内に資金を保持しています。
これらの資金の支払いがトリガするためには、[送金 API](https://stripe.com/docs/api#create_transfer)を利用してください。
例えば、マネージド・アカウントの残高から彼らの銀行口座に 10 USD を送るためには:

```ruby
Stripe.api_key = PLATFORM_SECRET_KEY
Stripe::Transfer.create(
  {
    :amount => 1000,
    :currency => "usd",
    :destination => "default_for_currency"
  },
  {:stripe_account => CONNECTED_STRIPE_ACCOUNT_ID}
)
```

`destination=default_for_currency` を設定している理由は、Stripe が与えられた通過でアカウントのデフォルトの口座もしくはデビットカードに送金を行うようにするためです。

ユーザーの有効な残高（その時のその時の送金可能な上限）を取得するためには、彼らの代わりに[残高照会](https://stripe.com/docs/api#retrieve_balance)呼び出しを実行してくだい。

Stripe はそれぞれの残高の中で、異なる支払い元からの残高の積立を監視しています。
[残高照会](https://stripe.com/docs/api#retrieve_balance)レスポンスは、ソースの種別によって、それぞれの残高の構成要素に分類します。
非クレジットカード残高の API を介して送金をしたい場合、リクエスト内でどこから転送するかを指定する必要があります。
任意の送金元の残高の一部は（返金やチャージバックを介すと）マイナスになることに注意し、送金は有効な残高の総計より大きい値ではできないことにも注意してください。

```ruby
Stripe.api_key = PLATFORM_SECRET_KEY
Stripe::Transfer.create(
  :amount => 24784,
  :currency => 'usd',
  :destination => 'default_for_currency',
  :source_type => 'bank_account',
)
```

この API 呼び出しは、連携アカウントの Stripe 残高から彼らの銀行口座に資金が移動するだけです。
Stripe アカウント間で資金を移動させたい場合は、[特殊な送金ケース](https://stripe.com/docs/connect/special-case-transfers)や[プラットフォームを介した課金](https://stripe.com/docs/connect/payments-fees#charging-through-the-platform)を参照してください。

## マイナス残高

マネージド・アカウントは時々、通常は払い戻しが原因でマイナス残高に陥ることがあります。
このようなケースでは、Stripe はマイナス残高の監視を行い、アカウントに入ってくる新しい資金と相殺させることで調整します。
残高がしばらくマイナスだと、あなたはアカウントへの口座送金ができなくなります。
アカウントの Stripe 残高が再びプラスになると、Stripe はマネージド・アカウントへの口座送金を再開させます。

最終的には、あなた（プラットフォーム）はマネージド・アカウント（マネージド・アカウントのみが対象でスタンドアロン・アカウントは別です）のマイナス残高に対する責任を負っています。
代わりの資金を確保する目的で、Stripe はあなたのマネージド・アカウントを横断してマイナス残高をカバーするため、あなたのプラットフォームアカウントの残高を予備としておさえています。

そこで言われるのが、マイナス残高から資金を回収することの補助を提供しています。
支払うべき資金の代わりをマネージド・アカウントの銀行口座から引き落とすか否かを示すためには、マネージド・アカウントの `debit_negative_balances` フラグを設定してください。
これは常に可能ではなく、国や銀行口座の詳細に依存しています。
また、自動的にデビットカードから引き落とすことはできません。

-----

## 参考文献

あなたの実装を手助けする残りのガイドをチェックしてください。

- [マネージド・アカウント](https://stripe.com/docs/connect/managed-accounts)
- [アカウントの更新](https://stripe.com/docs/connect/updating-accounts)
