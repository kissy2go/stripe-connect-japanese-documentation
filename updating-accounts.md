# アカウントの更新

あなたは API を通して Stripe の作成や管理を行う場合、Stripe に提供しなければならない情報が何なのかを常に留意しておく必要があります。

-----

マネージド・アカウントを実行すると、動作とともに膨大な量の権限をあなたに与えます: Stripe をスタンドアロン・アカウントのために利用できるほぼすべての設定は、この API を通して可能です。
この権限により、あなたは様々なリスクやコンプライアンス処理を通してアカウントの面倒を見ると予想され、主に Stripe の要求する情報を収集する形をとることになるでしょう。

アカウントオブジェクトの設定可能なプロパティの全リストは、[API リファレンス](https://stripe.com/docs/api#update_account)で利用可能となっています: このセクションではマネージド・アカウントにもっとも関連性の高いものをまとめており、より微細な詳細に関して述べています。

## アカウントの検証

マネージド・アカウントは `verification` プロパティを持っており、これはアカウント保有者に関する Stripe が必要とする情報を表す読み取り専用のハッシュです（[身元確認](https://stripe.com/docs/connect/identity-verification)も参照してください）。
これはマネージド・アカウントをトラッキングするためのとても重要なセクションで、それを無視するとあなたが制御しているアカウントへの送金が無効になる結果をまねきます。

```json
  "verification": {
    "fields_needed": [
      "legal_entity.type"
    ],
    "due_by": null,
    "disabled_reason": null
  },
```

この `fields_needed` プロパティは、アカウントの検証に必要なフィールドが文字列の配列として表現されています。
これらの文字列は通常、`Account` オブジェクトのプロパティと直接結びついています。
たとえば、`legal_entity.type` という文字列は、`legal_entity` というハッシュの `type` フィールドに対応しています。
`fields_needed` が空でない、追加情報が必要な場合、その情報は追加機能（もし、たとえば `transfers_enabled` が false）を有効にする必要もあるもので、または（もし `due_by` が設定されていれば）、機能がいずれは無効にされてしまうことから防ぐために必要です。

`fields_needed` が空でない場合は、`due_by` が設定されているはずです。
これは本人確認のため情報がいつまでに必要なのかの UNIX Timestamp です。
通常、私たちが期日までに必要な情報を受け取らない場合、アカウントへの銀行振込を無効にするでしょう。
しかしながら、まれな状況では他の結果が存在します。
例えば、送金がすでに無効になっており、私たちの問い合わせが合理的な期間内に回答いただけない場合、Stripe は課金処理の能力を無効にするでしょう。
不正やその他のまれな状況が発生することを除き、アカウントの機能を制限する前に、Stripe は常に少なくとも 3 日間の猶予を与えます。

これとは別に `disabled_reason` も設定されています。
これは、なぜこのアカウントが送金や課金ができないのかの理由が説明された文字列です。
この理由はいくつかのカテゴリに分類することができます:

1. `rejected.fraud` - このアカウントは不正や違法行動の疑い原因で拒否されました。
2. `rejected.terms_of_service` - このアカウントは利用規約違反の疑いで拒否されました。
3. `rejected.listed` - このアカウントは（金融サービスプロバイダや政府などの）サードパーティで禁じられている個人もしくは企業のリスト内のものと一致したため拒否されました。
4. `rejected.other` - このアカウントはその他のいくつかの理由で拒否されました。
5. `fields_needed` - このアカウントの送金や課金機能を有効にするためには追加の検証情報が必要です。
6. `listed` - このアカウントは禁じられている個人または企業のリストに一致している可能性があります。Stripe がアカウントを拒否もしくは元に戻すかを適切に調査します。
7. `other` - このアカウントは拒否されてはいませんが、その他の理由で無効になっています。

あなたはアカウントの現在の能力を知るために `charges_enabled` と `transfers_enabled` プロパティを調べることができます。
あなたがより詳細な情報をご希望なら、お気軽に私たちにお聞きください。

あなたが `fields_needed` に最初に触れるのは、アカウントを作成するときです: 送金が可能になる前までは、私たちは一定の情報セットを必要とします。
具体的に必要なものは、国や法人の種類によって異なります。
むしろ、これらのオプションのすべての組み合わせの必要条件（およびそれらの要件の変更）をトラッキングするよりも、発見するために `fields_needed` を使い、これらが発生したら住所の確認が必要です。

これらが言っていることは、この最終的に要求される情報を知ることは、プランニングに有効だということです。
私たちは、あなたが収集する情報を決定するのに役立ついくつかの[アドバイスやガイドライン](https://support.stripe.com/questions/what-information-do-i-need-to-provide-to-verify-managed-accounts)を提供します。

私たちは、新しいアカウントを作成する際に、開発者がこのフローを利用することを期待しています:

1. 前もってどの情報、通常はあなたがサポートしたい国での `fields_needed` のサブセットを要求するのかを決める。
2. ユーザーにこの情報をリクエストし、彼らのアカウントを作成する際にこれらを利用する。
3. アカウントを作成した後は、任意の追加要件の `verification[fields_needed]` を確認する。追加の情報が必要な場合、ユーザーからこれを入手し、連携アカウントを更新する。
4. 必要に応じて、`verification[fields_needed]` の変更を Webhook で監視し、必要とされている追加の情報をユーザーに伝える。

最後に、`fields_needed` にはいくつか特殊な値がある:

- `legal_entity.additional_owners`: EU のオーナーの要件については、以下の [`additional_owners` セクション](https://stripe.com/docs/connect/updating-accounts#additional-owners)を参照してください。
- `legal_entity.verification.document`: [本人確認のセクション](https://stripe.com/docs/connect/identity-verification)を参照し、個人の身元を確認してください。これはまた、`legal_entity.additional_owners.#.verification.document` に適用されることに注意してください（`#`には 0, 1, 2 または 3 が指定可能）。
- `external_account`: これは単に Stripe アカウントの検証を完了するために、Stripe が銀行口座を求めているということを示しています。詳細は[銀行送金ガイド](https://stripe.com/docs/connect/bank-transfers)にあるので参照してください。

## 追加の所有者

`legal_entity[additional_owners]` パラメータは、個人情報のハッシュの配列です。
[単一ユーロ決済地域](https://en.wikipedia.org/wiki/Single_Euro_Payments_Area)の加盟国にいる場合、Stripe は代表に加えてその会社の少なくとも 25% を誰が保有しているのかの情報を収集し検証することを必須にしています。
それぞれのオーナーごとに、可能はフィールドは `first_name` と `last_name`, `dob`, `address` です。
住所と生年月日フィールドは、トップレベルの `legal_entity[address]` と `legal_entity[dob]` の書式と同じにしてください、住所はアカウントと同じ国である必要はありません。
Stripe はそれぞれのオーナーの検証を試み、もし検証が失敗した場合、私たちは追加で情報のリクエストをするでしょう。
このように、それぞれのオーナーはまさに `legal_entity[verification]` ハッシュ（[本人確認のセクション](https://stripe.com/docs/connect/identity-verification)を参照）と同じように動作する `verification` プロパティを持っています。

`legal_entity.additional_owners` が `verification[fields_needed]` 配列内に記載されていた場合、任意の追加の所有者を提供するために、ユーザーに依頼する必要があります。
あなたは、追加の所有者についての情報を持つ配列または整数インデックスのハッシュを渡すことができます。
それ以外の、追加の所有者の所有者が存在しない場合、以下の例に示すように `legal_entity[additional_owners]` を更新する必要があります。

```ruby
Stripe.api_key = PLATFORM_SECRET_KEY
account = Stripe::Account.retrieve({CONNECTED_STRIPE_ACCOUNT_ID})

# Create additional owners
account.legal_entity.additional_owners = [
  {:first_name => 'Bob', :last_name => 'Smith'},
  {:first_name => 'Jane', :last_name => 'Doe'}
]

# Add an additional owner
length = account.legal_entity.additional_owners.length
account.legal_entity.additional_owners[length] = {
  :first_name => 'Andrew',
  :last_name => 'Jackson'
}

# Update additional owners
account.legal_entity.additional_owners[0].first_name = 'Robert'

# Indicate that there are no additional owners
account.legal_entity.additional_owners = nil

account.save
```

## 利用規約の同意

Stripe はすべてのマネージド・アカウントに [Stripe 連携アカウント規約](https://stripe.com/connect/account-terms)に同意を必須としている。
あなたは、ユーザーが本契約に同意したことを確認し、彼らの銀行口座に支払いを行う前に、Stripe に必要なパラメータを提供しなければならない:

```ruby
Stripe.api_key = PLATFORM_SECRET_KEY
account = Stripe::Account.retrieve({CONNECTED_STRIPE_ACCOUNT_ID})
account.tos_acceptance.date = Time.now.to_i
account.tos_acceptance.ip = request.remote_ip # Assumes you're not using a proxy
account.save
```

あなたのプラットフォーム上で Stripe を通して決済を受ける前に、あなたのユーザーが [Stripe 連携アカウント規約](https://stripe.com/connect/account-terms)に同意することを保証する責任がある。
彼らのアカウントをアクティベートする時になど、あなたのユーザーには少なくとも規約のリンクを提示し、Stripe を使う前に明示的に規約に同意してもらう必要があります。

|アカウントを登録する|
|:--|
アカウントを登録すると、[利用規約](https://stripe.com/docs/connect/updating-accounts#tos-acceptance)と[Stripe 連携アカウント規約](https://stripe.com/connect/account-terms)に同意したことになります。

私たちはまた、あなたの規約の中で、決済の受け入れは [Stripe 連携アカウント規約](https://stripe.com/connect/account-terms)に依存して提供されているという条件を、あなたのユーザーに明示して組み込むことをおすすめしています。
[Stripe 連携アカウント規約](https://stripe.com/connect/account-terms)への明示的な参照とリンクを契約内容の中に含めることは、これを達成する 1 つの方法です。
例えば、規約を以下のようにします:

```
Payment processing services for [account holder term, e.g. drivers or sellers] on [platform name] are provided by Stripe and are subject to the Stripe Connected Account Agreement, which includes the Stripe Terms of Service (collectively, the “Stripe Services Agreement”). By agreeing to [this agreement / these terms / etc.] or continuing to operate as a [account holder term] on [platform name], you agree to be bound by the Stripe Services Agreement, as the same may be modified by Stripe from time to time. As a condition of [platform name] enabling payment processing services through Stripe, you agree to provide [platform name] accurate and complete information about you and your business, and you authorize [platform name] to share it and transaction information related to your use of the payment processing services provided by Stripe.
```

## Webhook

マネージド・アカウントに関する重要な `verification` ハッシュの変更を含むすべての変更は、`account.updated` イベントを通して Webhook を通じて送信されます。
これらを監視するには、あなたの[アカウントの設定](https://dashboard.stripe.com/account/webhooks)でプラットフォームの Webhook URL を設定してください。

-----

## 次に

ユーザーの検証や決済フローをカスタマイズに関して詳しく知りたい場合は以下を読んでください。

- [本人確認](https://stripe.com/docs/connect/identity-verification)
- [銀行への送金](https://stripe.com/docs/connect/bank-transfers)
- [すべての API リファレンス](https://stripe.com/docs/api)
