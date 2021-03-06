# マネージド・アカウント

あなたは、ユーザの全体の提供されたエクスペリエンスを制御したい場合は、API を通じて Stripe のマネージド・アカウントを作成して動作させることができます。

-----

Stripe のマネージド・アカウントは、アカウント保有者にはほぼ完全に見えなくなります: あなたはこれのすべてのインタラクションと、Stripe の求める任意の情報を収集する責任がある。

マネージド・アカウントを使用すると、あなたは彼らの銀行口座情報を変更する能力を持ち、彼らの Stripe アカウントのその他の任意の情報をプログラムで修正できる。
これらのユーザは Stripe にログインできなくなるので (そしてほとんどは彼ら自身の Stripe アカウントを知る由がないので)、彼らの基盤やダッシュボード、レポーティング、連絡フローなどの提供はあなた次第です。

あなたがマネージド・アカウントの作成に必要な唯一の情報は、その国です: その他のすべての情報はあとで収集して更新できます。
**国は、アカウント保有者が常駐、もしくは彼らのビジネスが法的に確立されている国でなければなりません。**
例えば、もしあなたが米国におり、かつあなたがカナダで法的に確立されたアカウントでビジネスしている場合、あなたはアカウントを作成するためには国として "CA" を用いてください。


## マネージド・アカウントを作成するために必要なこと

- **最低限の API バージョン**: あなたは少なくとも 2014 年 12 月 17 日より最新の API バージョンを使わなければなりません。
- **利用規約の更新**: マネージド・アカウントの作成には [あなたの利用規約の更新](https://stripe.com/docs/connect/updating-accounts#tos-acceptance) が必要で、Stripe の規約への参照を含んでいなければなりません。
- **情報収集リクエストのハンドリング**: アカウント保有者に直接リクエストする代わりに、Stripe は SSN やパスポートの写しのような情報をあなたにリクエストします。あなたはこの情報をユーザーから収集し、Stripe に提供しなければなりません、そうでないと Stripe はアカウントへの送金を無効にするでしょう。
- **オーストラリア、カナダもしくは米国のプラットフォームの場合**: マネージド・アカウントはパブリック・ベータです。オーストラリア、カナダまたは米国のプラットフォームは、[Stripe がサポートする](https://stripe.com/global)国のマネージド・アカウントを作成できます。マネージド・アカウント プラットフォームがあなたの国で利用可能になったときに通知を希望する場合は、[私たちに知らせてください](mailto:connect+JP@stripe.com)。
- **不正調査**: あなたのプラットフォームは、マネージド・アカウントによって発生したなんらかの損失に対しての最終的な責任がある。これらを最適に保護するためには、不正から保護するに、あなたはあなたのプラットフォームを通して登録されたすべてのアカウントを精査する必要があります。この詳細に関しては私たちの[ベストプラクティス](https://stripe.com/docs/connect/best-practices#fraud)を参照してください。

## マネージド・アカウントの作成

最低限、マネージド・アカウントを作成し連携するためには、作成リクエスト内に単に `managed` に true を設定し、国を提供してください。
国は一度設定すると変更することはできません。

国の値はマネージド・アカウントのための値です: アカウント担当者が住んでいるか、彼らのビジネスが法的に確立されている値です。
プラットフォームの国を使ってはいけません。
国の値はまた、彼らの持てる銀行口座の種類と、彼らの身元を確認するために必要な情報です。

```ruby
Stripe.api_key = PLATFORM_SECRET_KEY
Stripe::Account.create(
  {
    :country => "US",
    :managed => true
  }
)
```

API 呼び出しが成功した結果が、ユーザーのアカウント情報です。

```json
{
  ...
  "id": "acct_12QkqYGSOD4VcegJ",
  "keys": {
    "secret": "sk_live_AxSI9q6ieYWjGIeRbURf6EG0",
    "publishable": "pk_live_h9xguYGf2GcfytemKs5tHrtg"
  },
  "managed": true
  ...
}
```

あなたはアカウント ID を保存する必要があるでしょう、また、もしこれらを[認証](https://stripe.com/docs/connect/authentication)のために利用したい場合、秘密鍵と公開鍵を保存する必要もあるでしょう。
この鍵はその後取得することはできません。

唯一の国で作成されたアカウントはかなり限られます: それは少量の資金しか受け取れません。
あなたが送金を可能にし、良好な状態のアカウントを保持したいと望むなら、あなたはアカウント保持者に関する詳細な情報を提供する必要があります。
マネージド・アカウントを作成する際、あなたがそう設定したい[他の多くのフィールド](https://stripe.com/docs/api#create_account)があります。
私たちは、このセクション内の他のガイドでより微細なことを論じています。

すくなくとも、通常あなたは氏名と誕生日を事前に収集する必要があるでしょう。

## Webhook

一度作成されると、すべてのアカウントの変更は `account.updated` イベントとして Webhook を通じて送信されます。
これらを監視するには、あなたの[アカウントの設定](https://dashboard.stripe.com/account/webhooks)でプラットフォームの Webhook URL を設定してください。

-----

## さらに!

マネージド・アカウントはあなたに柔軟性を与えますが、より多くの連携作業が必要となります。
マネージド・アカウントの動作については、こちらをご覧ください。

- [アカウントの更新](https://stripe.com/docs/connect/updating-accounts)
- [認証](https://stripe.com/docs/connect/authentication)
- [決済と手数料](https://stripe.com/docs/connect/payments-fees)
- [すべての API リファレンス](https://stripe.com/docs/api)
