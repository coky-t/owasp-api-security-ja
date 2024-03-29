# API4:2023 制限のないリソース消費 (Unrestricted Resource Consumption)

| 脅威エージェント/攻撃手法 | セキュリティ上の弱点 | 影響 |
| - | - | - |
| API 依存 : 悪用難易度 **平均的** | 普及度 **広範** : 検出難易度 **容易** | 技術的影響 **重度** : ビジネス依存 |
| 悪用にはシンプルな API リクエストを必要とします。複数の同時リクエストは単一のローカルコンピュータから、またはクラウドコンピューティングリソースを使用することで実行できます。利用可能な自動化ツールのほとんどは高負荷のトラフィックによって DoS を引き起こし、API のサービスレートに影響するように設計されています。 | クライアントとのインタラクションやリソース消費を制限しない API を見かけることはよくあります。返されるリソース数を制御するパラメータを含む API リクエストや、レスポンスのステータス、タイム、長さの解析実行など、細工された API リクエストにより、問題を特定できるはずです。同じことがバッチ操作にも当てはまります。脅威エージェントはコストへの影響を可視化できませんが、これはサービスプロバイダ (クラウドプロバイダなど) のビジネスモデルや価格モデルに基づいて推測できます。 | 悪用することでリソース枯渇による DoS につながる可能性があるだけでなく、CPU 需要の上昇やクラウドストレージのニーズの増加などにより、インフラストラクチャ関連などの運用コストの増加につながる可能性もあります。 |

## その API は脆弱か？

API リクエストを満たすにはネットワーク帯域幅、CPU、メモリ、ストレージなどのリソースが必要です。必要なリソースはサービスプロバイダが API 統合によって利用可能になり、電子メール、SMS、電話コールの送信や生体認証での検証など、リクエストごとに料金を支払うこともあります。




以下の制限のうち一つでも欠落していたり不適切に設定されている (低すぎる、高すぎるなど) 場合、API は脆弱となります。


* 実行タイムアウト
* 最大割り当てメモリ
* ファイルディスクリプタの最大数
* プロセスの最大数
* 最大アップロードファイルサイズ
* 一回の API クライアントリクエストで実行するオペレーション数 (GraphQL バッチ処理など)

* 一回のリクエスト・レスポンスで返せるページごとのレコード数
* サードパーティサービスプロバイダの利用限度額

## 攻撃シナリオの例

### シナリオ #1

あるソーシャルネットワークは SMS 認証を利用した「パスワードを忘れた場合」フローを実装し、ユーザーがパスワードをリセットするために SMS 経由でワンタイムトークンを受信できるようにしました。



ユーザーが「パスワードを忘れた場合」をクリックすると、API コールがユーザーのブラウザからバックエンド API に送信されます。


```
POST /initiate_forgot_password

{
  "step": 1,
  "user_number": "6501113434"
}
```

そして、裏側では、API コールがバックエンドから SMS のデリバリを担当するサードパーティ API に送信されます。


```
POST /sms/send_reset_pass_code

Host: willyo.net

{
  "phone_number": "6501113434"
}
```

サードパーティプロバイダ Willyo はこのタイプのコールごとに 0.05 ドルを請求します。

攻撃者は最初の API コールを何万回も送信するスクリプトを作成します。
バックエンドはこれに続き Willyo に数万のテキストメッセージの送信を要求し、同社は数分で数千ドルを失うことになります。



### シナリオ #2

ある GraphQL API エンドポイントではユーザーがプロフィール画像をアップロードできます。

```
POST /graphql

{
  "query": "mutation {
    uploadPic(name: \"pic1\", base64_pic: \"R0FOIEFOR0xJVA…\") {
      url
    }
  }"
}
```

アップロードが完了すると、API はアップロードされた画像をもとにさまざまなサイズの複数のサムネールを生成します。
この画像操作にはサーバーから多くのメモリを取得します。


API は従来のレート制限保護を実装しています。ユーザーは短期間に何度も GraphQL エンドポイントにアクセスできません。
また API はサムネールを生成する前にアップロードされた画像のサイズをチェックし、大きすぎる画像の処理を避けます。



攻撃者は GraphQL の柔軟な性質を利用して、これらのメカニズムを簡単にバイパスできます。


```
POST /graphql

[
  {"query": "mutation {uploadPic(name: \"pic1\", base64_pic: \"R0FOIEFOR0xJVA…\") {url}}"},
  {"query": "mutation {uploadPic(name: \"pic2\", base64_pic: \"R0FOIEFOR0xJVA…\") {url}}"},
  ...
  {"query": "mutation {uploadPic(name: \"pic999\", base64_pic: \"R0FOIEFOR0xJVA…\") {url}}"},
}
```

API は `uploadPic` オペレーションの試行回数を制限していないため、このコールはサーバーメモリの枯渇とサービス拒否につながります。



### シナリオ #3

サービスプロバイダはクライアントが API を使用して任意の大きさのファイルをダウンロードできるようにします。
これらのファイルはクラウドオブジェクトストレージに保存され、それほど頻繁に変更されることはありません。
このサービスプロバイダはサービスレートを向上させ、帯域幅の消費を低く抑えるために、キャッシュサービスに依存しています。
このキャッシュサービスは 15GB までのファイルしかキャッシュしません。


ファイルの一つが更新されると、そのサイズは 18GB に増加します。
すべてのサービスクライアントはすぐに新しいバージョンの取得を開始します。
このクラウドサービスには消費コストアラートも最大コスト許容値もなかったため、翌月の請求額は平均で 13 米ドルから 8000 米ドルに増加しました。


## 防止方法

* コンテナやサーバーレスコード (Lambdas など) のような [メモリ][1]、[CPU][2]、[再起動回数][3]、[ファイルディスクリプタ、プロセス][4] を簡単に制限できるソリューションを使用します。


* 文字列の最大長、配列の最大要素数、アップロードファイルの最大サイズ (保存がローカルかクラウドストレージかに関係なく) など、受信するすべてのパラメータとペイロードのデータサイズの最大値を定義して適用します。



* 定義された時間枠内でクライアントが API とやり取りできる回数に制限 (レート制限) を実装します。

* レート制限はビジネスニーズに基づいて微調整する必要があります。API エンドポイントによっては、より厳格なポリシーが必要になることもあります。

* 一つの API クライアントやユーザーが一つのオペレーション (OTP の検証や、ワンタイム URL をアクセスせずにパスワードリカバリをリクエストするなど) を実行できる回数や頻度を制限や調整します。


* クエリ文字列とリクエストボディパラメータ、特にレスポンスで返されるレコード数を制御するものに対して、適切なサーバーサイドの検証を追加します。


* すべてのサービスプロバイダや API 統合に対して支出制限を設定します。支出制限を設定できない場合には、代わりに課金アラートを設定すべきです。



## 参考資料

### OWASP

* ["Availability" - Web Service Security Cheat Sheet][5]
* ["DoS Prevention" - GraphQL Cheat Sheet][6]
* ["Mitigating Batching Attacks" - GraphQL Cheat Sheet][7]

### その他

* [CWE-770: Allocation of Resources Without Limits or Throttling][8]
* [CWE-400: Uncontrolled Resource Consumption][9]
* [CWE-799: Improper Control of Interaction Frequency][10]
* "Rate Limiting (Throttling)" - [Security Strategies for Microservices-based Application Systems][11], NIST


[1]: https://docs.docker.com/config/containers/resource_constraints/#memory
[2]: https://docs.docker.com/config/containers/resource_constraints/#cpu
[3]: https://docs.docker.com/engine/reference/commandline/run/#restart
[4]: https://docs.docker.com/engine/reference/commandline/run/#ulimit
[5]: https://cheatsheetseries.owasp.org/cheatsheets/Web_Service_Security_Cheat_Sheet.html#availability
[6]: https://cheatsheetseries.owasp.org/cheatsheets/GraphQL_Cheat_Sheet.html#dos-prevention
[7]: https://cheatsheetseries.owasp.org/cheatsheets/GraphQL_Cheat_Sheet.html#mitigating-batching-attacks
[8]: https://cwe.mitre.org/data/definitions/770.html
[9]: https://cwe.mitre.org/data/definitions/400.html
[10]: https://cwe.mitre.org/data/definitions/799.html
[11]: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-204.pdf
