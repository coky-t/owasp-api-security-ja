API4:2023 制限のないリソース消費 (Unrestricted Resource Consumption)
====================================================================

| 脅威エージェント/攻撃手法 | セキュリティ上の弱点 | 影響 |
| - | - | - |
| API 依存 : 悪用難易度 **2** | 普及度 **3** : 検出難易度 **3** | 技術的影響 **2** : ビジネス依存 |
| エクスプロイトはシンプルな API リクエストを必要とします。複数の同時リクエストは単一のローカルコンピュータから、またはクラウドコンピューティングリソースを使用することで実行できます。 | クライアントとのインタラクションやリソース消費を制限しない API を見かけることはよくあります。ほとんどの場合、インタラクションはログに記録されますが、監視の欠如や不適切な監視によって、悪意のあるアクティビティが気づかれずに見過ごされてしまいます。 | エクスプロイトはリソース枯渇による DoS につながる可能性がありますが、サービスプロバイダの請求にも影響を与える可能性があります。 |

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

攻撃者は /api/v1/images に POST リクエストを発行して大きな画像をアップロードします。
アップロードが完了すると、API は異なるサイズの複数のサムネイルを作成します。
アップロードされた画像のサイズが原因で、サムネイルの作成中に利用可能なメモリが枯渇し、この API は応答しなくなります。


### シナリオ #2

クレジットカードをアクティベートするにはクレジットカードに印字されている最後の四桁の数字を指定して、以下の API リクエストを発行する必要があります。 (そのカードに物理的にアクセスできるユーザーのみがこのような操作を実行できるはずです)



```
POST /graphql

{
  "query": "mutation {
    validateOTP(token: \"abcdef\", card: \"123456\") {
      authToken
    }
  }"
}
```

悪意のある人物は以下のようなリクエストを作成して、クレジットカードに物理的にアクセスすることなくクレジットカードのアクティベーションを実行できるでしょう。


```
POST /graphql

[
  {"query": "mutation {activateCard(token: \"abcdef\", card: \"0000\") {authToken}}"},
  {"query": "mutation {activateCard(token: \"abcdef\", card: \"0001\") {authToken}}"},
  ...
  {"query": "mutation {activateCard(token: \"abcdef\", card: \"9999\") {authToken}}"}
}
```

この API は activateCard オペレーションの試行回数を制限していないため、いずれか一つが成功するでしょう。


### シナリオ #3

サービスプロバイダはクライアントが API を使用して任意の大きさのファイルをダウンロードできるようにします。
これらのファイルはクラウドオブジェクトストレージに保存され、それほど頻繁に変更されることはありません。
このサービスプロバイダはサービスレートを向上させ、帯域幅の消費を低く抑えるために、キャッシュサービスに依存しています。
このキャッシュサービスは 15GB までのファイルしかキャッシュしません。


ファイルの一つが更新されると、そのサイズは 18GB に増加します。
すべてのサービスクライアントはすぐに新しいバージョンの取得を開始します。
このクラウドサービスには消費コストアラートも最大コスト許容値もなかったため、翌月の請求額は平均で 13 米ドルから 8000 米ドルに増加しました。


## 防止方法

* [メモリ][1]、[CPU][2]、[再起動回数][3]、[ファイルディスクリプタ、プロセス][4] を簡単に制限できるコンテナベースのソリューションを使用します。

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
