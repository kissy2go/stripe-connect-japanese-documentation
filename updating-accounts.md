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