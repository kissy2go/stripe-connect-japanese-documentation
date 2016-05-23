# マネージド・アカウントの本人認証

あなたの管理する Stripe アカウントの本人認証は、あなたのプラっとフォームのリスクを減らす重要で（そして必須な）ステップです。
これを読んだ後にヘルプが必要な場合は、[一般的な質問](https://support.stripe.com/search?q=connect)に対する答えを調べるか、freenode 上の [#stripe](irc://irc.freenode.net/stripe) で他の開発者とライブチャットをしてください。

-----

すべての国は、個人や法人への資金の払い出しに関して独自のルールを持っています。
これらは一般的に、"[Know Your Customer](http://en.wikipedia.org/wiki/Know_your_customer)" (KYC) の規則として知られています。
これらの規則は、大きく 2 つのことを必要とします: 情報を収集し、正しく情報を検証することです。
プラットフォームから必要な情報が与えられると、Stripe はマネージド・アカウントの検証を実行することができるようになります。

マネージド・アカウントは作成されることができ、アカウントの国で、わずかな決済の受け付けを開始することができるようになります。
Stripe は、マネージド・アカウントが銀行振込の受け付けを開始し、決済プロセスを開始できるよう、いくつかの追加情報をたずねます。
マネージド・アカウントは振込を受け付け始めた後、プラットフォームは必要な任意の残りの情報を提供する必要があります。
検証プロセスが完了したとき、マネージド・アカウントは継続的に動作することができます。

## 検証フローと必須項目

Connect プラットフォームは、Stripe が検証を行うために、マネージド・アカウントのオーナーから必要な情報を収集し、提供することが必要です。
正確に、いつどのように情報を要求するかは、プラットフォームによって決定されます。

送金に必要な最小限の情報はプラットフォームから Stripe に対して 7 日以内、もしくは 1,000 USD（または同等の）課金を処理した後のどちらか最初にきたタイミングで提供される必要があります。
この情報には以下の内容を含みます:

- 事業種別（個人もしくは会社）
- 個人もしくは会社を表すフルネーム
- 個人もしくは会社を表す誕生日（設立日）
- 個人もしくは職場の住所（オーストラリアと米国のマネージド・アカウントはこれは不要です）

これが完了しない場合、情報が提供されるまで Stripe は一時的に課金の作成を停止します。
21 日後、Stripe は任意の作成された課金を払い戻しします。

マネージド・アカウントが合計で 1,000 USD（または同等の）送金を受けると、Stripe は依然として必要な追加の[検証情報](https://stripe.com/docs/connect/required-verification-information)を要求します。
Stripe が他の数千 USD（または同等の）送金をした後に、提供された情報でアカウントの検証ができない場合、この問題が解決されるまでは一時的に送金は停止されます。

### 事前に必要な情報を収集する

いくつかのプラットフォームは、ユーザーに 1 回以上情報を尋ねる必要はなく、そしてそのかわり開始時にできるだけ情報を尋ねるでしょう。
あなたがそれを集めるのと同じように、あなたは必要な情報を私たちに提供することができます。

すべての必要な情報はマネージド・アカウントの作成時に提供されるべきで、検証プロセスは自動的に完了することができます。
マネージド・アカウントを完全に処理するように設定し、課金や送金を継続的に受けるようにはじめから検証したい場合、あなたはこのオプションを選択することができます。

## 個人と企業情報

個人にとって、これはその人物に関しての情報を収集することを含んでいます。
企業にとって、企業についての情報とともに、Stripe アカウントを設定するために認証する 1 人または数人の人物（会社を代表する人物）の情報を収集することも必要になります。

会社の代表は、代わりにマネージド・アカウントを設定するために彼らの会社から許可を得ていれば、どなたでも可能です。
これらの個人は、企業のアカウントで発生するすべての活動の責任を負いません。

## 本人確認が必要なら決めなければいけないこと

`Account` オブジェクトは、そのアカウントを検証するために求められている必要事項を表す[検証属性](https://stripe.com/docs/connect/updating-accounts#account-verification)を持っています。
`verification` は `fields_needed` 配列内のフィールドのリストを含んでいます。
連携アカウントの検証が必要かどうかは、`legal_entity.verification.document` や `legal_entity.additional_owners.#.verification.document`（`#` には 0, 1, 2 もしくは 3 が入る）に含まれている `fields_needed` を参照することで試すことができます。

本人認証に求められる情報をキャッチする最も簡単な方法は、`verification[fields_needed]` リストの変更（と同等の他のアカウント変更）があるたびに送信される `account.updated` の Webhook を監視することです。

## 法人の検証ハッシュ

トップレベルの `verification` 属性が Stripe アカウントの検証に関しての情報を含んでいる一方、`legal_entity` 属性は彼ら独自の検証のサブハッシュを持っています。
このハッシュは、関連する法人の身元を確認する情報を含んでいます。
`legal_entity` は以下のようなハッシュです:

```json
"verification": {
  "status": "pending",
  "document": {
    "id": "file_1234",
    "created": 1414480885,
    "size": 10240
  },
  "details": null,
  "details_code": null
}
```

このハッシュは、あなたが身元確認をハンドルするために必要なすべてを含んでいます:

- `status`: 以下の 3 つの値の中から 1 つ
  - `unverified`: 検証が失敗したか、検証をするのに十分な情報がなかのいずれかの理由により、Stripe は現在、このエンティティを検証できません
  - `pending`: Stripe は現在、このエンティティの検証を行っています
  - `verified`: Stripe はこのエンティティの検証に成功しました
- `document`: もっとも最近アップロードされた身元確認書類への参照
- `details`: 身元に関する任意の情報を含む読み取り専用の文字列
- `details_code`: なぜ検証に失敗したかのコードを含む読み取り専用の文字列。以下の 10 の値の中から 1 つ
  - `scan_corrupt`: アップロードされた身元確認書類が破損している
  - `scan_not_readable`: 提供された画像が読み取れない。とてもぼやけているか、明るさが十分ではない
  - `scan_failed_greyscale`: 提供された画像が白黒のため、読み取れない。代わりにカラーコピーをアップロードしてもらう
  - `scan_not_uploaded`: 身元確認書類のスキャンがアップロードされていない
  - `scan_id_type_not_supported`: 提出された身元確認書類の種別は未サポート
  - `scan_id_country_not_supported`: 身元確認書類の国が未サポート
  - `scan_name_mismatch`: 身元確認書類に記載されている名前がアカウントのものと一致していない
  - `scan_failed_other`: その他の理由でスキャンが失敗している
  - `failed_keyed_identity`: 提供された身元確認書類では検証できない
  - `failed_other`: その他の理由で身元確認が失敗した

`unverified` は緊急な問題ではないが、将来的にそのエンティティに関して追加の情報が Stripe として必要なことを意味するということに注意してください。

`details` に提示されたいずれかの検証状況に関する情報は、アカウント保有者への表示に適しています（なぜ検証が失敗したのか等）。

## 本人確認のハンドリング

本人確認の要求に応答するには、2 つの方法があります。
1 つ目は、法人の `first_name`, `last_name`, `dob` ハッシュか、`personal_id_number`、もしくは `ssn_last_4` 属性のうちの 1 つを収集する方法です。
これらの情報はアカウント所有者が間違っている可能性があるため、Stripe はあなたがこれらの情報を変更することを許可しています。

2 つ目の方法は、パスポートや運転免許証などの本人確認書類のスキャンをアップロードすることです。
これは以下の 2 つのステップになります。

1. Stripe にファイルをアプロードする
2. ファイルをアカウントに紐付ける

## ファイルのアップロード

ファイル（JPEG もしくは PNG のどちらか）をアップロードするには、multipart/form-data として `https://uploads.stripe.com/v1/files` に POST リクエストしてください。
ファイルは `file` パラメータで渡し、常に `identity_document` という文字列を含む `purpose` という名前の別のフィールドも渡してください:

```ruby
Stripe.api_key = PLATFORM_SECRET_KEY
Stripe::FileUpload.create(
  {
    :purpose => 'identity_document',
    :file => File.new('/path/to/a/file.jpg')
  },
  {:stripe_account => CONNECTED_STRIPE_ACCOUNT_ID}
)
```

このリクエストでファイルはアップロードされ、トークンが返却されます:

```json
{
  "id": "file_5dtoJkOhAxrMWb",
  "created": 1403047735,
  "size": 4908
}
```

その後、本人確認のため、連携アカウントにファイルを紐付けるために
本人確認のため、連携アカウントにそのファイルを紐付けるために、トークンの `id` の値を利用してください。

## ファイルの紐付け

ファイルがアップロードされ、トークンが払い出された後は、アカウント更新の呼び出し時に単に`document` フィールドの値として提供してください:

```ruby
Stripe.api_key = PLATFORM_SECRET_KEY
account = Stripe::Account.retrieve({CONNECTED_STRIPE_ACCOUNT_ID})
account.legal_entity.verification.document = 'file_5dtoJkOhAxrMWb'
account.save
```

このステップで、`legal_entity[verification][status]` は `pending` に変更されます。
追加の所有者を検証を必要とする場合、代わりに `legal_entity[additional_owners][#][verification][document]` を使ってください、 # は `legal_entity[additional_owners]` 配列内の所有者のインデックスです。

### 本人確認書類の検証の確認

身元確認書類の画像が Stripe のチェックを通過したとすると、`status` フィールドは `verified` に変更されます。
あなたは認証プロセスの完了したことを `account.updated` Webhook の通知で受け取ります。
この検証は 1 日に数分からどこからでも得ることができ、提供されたアイテムをどのように読みだすかに依存することに注意してください。

Stripe は検証された身元確認書類を、不変なエンティティとして扱います。
検証された個人の情報は変更することができません。
検証済みの企業には、例えばファイル内の名前や EIN を変更するときは、[私たちに連絡](https://support.stripe.com/email)してください。

検証の試みが失敗した場合、そのステータスは `unverified` になり、`details` フィールドにその原因をステータスのメッセージが含まれるようになります。
このメッセージは"提供された画像は読み取ることができません"のように、ユーザーに提示しても安全なものとなっています。
加えて、レスポンスには`details_code` の値には `scan_not_readable` などが含まれています。
失敗すると、`verification[fields_needed]` は新しい身元確認書類のアップロードが必要なことを示します。
検証のための締め切りが近い場合は、`verification[due_by]` に日付が挿入されています。
ここでも、`account.updated` Webhook 通知として送信されます。

あなたは知ることができます。

`legal_entity[verification][document]` があれば、アカウントの検証に失敗したアップロードされた身元確認書類を知ることができますが、`verification[fields_needed]` は依然として本人確認書類を必要としていることを示しています。

-----

# 併せて読みたい

あなたの実装を手助けするため、残りのガイドもチェックしてください。

- [マネージド・アカウント](https://stripe.com/docs/connect/managed-accounts)
- [アカウントのアップデート](https://stripe.com/docs/connect/updating-accounts)
- [検証に必要な情報](https://stripe.com/docs/connect/required-verification-information)
