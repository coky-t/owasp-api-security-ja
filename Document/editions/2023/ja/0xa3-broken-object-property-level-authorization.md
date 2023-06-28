# API3:2023 オブジェクトプロパティレベル認可の不備 (Broken Object Property Level Authorization)

| 脅威エージェント/攻撃手法 | セキュリティ上の弱点 | 影響 |
| - | - | - |
| API 依存 : 悪用難易度 **容易** | 普及度 **普通** : 検出難易度 **容易** | 技術的影響 **中程度** : ビジネス依存 |
| API はすべてのオブジェクトのプロパティを返すエンドポイントを公開する傾向にあります。これは特に REST API に当てはまります。GraphQL などの他のプロトコルでは、返すべきプロパティを指定するために、リクエストに細工が必要となることもあります。操作可能なこれらの追加プロパティを特定するには、より多くの労力が必要ですが、このタスクを支援する自動化ツールがいくつかあります。 | API レスポンスを検査するだけで、返されたオブジェクトの表現内の機密情報を特定できます。ファジングは追加の (隠された) プロパティを特定するためによく使用されます。それらを変更できるかどうかは、API リクエストを作成し、そのレスポンスを解析することでわかります。ターゲットプロパティが API レスポンスで返されない場合、サイドエフェクト解析が必要になるかもしれません。 | プライベートオブジェクトや機密オブジェクトに不認可アクセスすると、データ開示、データ損失、データ破損を引き起こす可能性があります。特定の条件下では、オブジェクトプロパティへの不認可アクセスにより、権限昇格やアカウントの部分的または完全な乗っ取りにつながる可能性があります。 |

## その API は脆弱か？

ユーザーが API エンドポイントを使用してオブジェクトにアクセスできるようにする場合、ユーザーがアクセスしようとしている特定のオブジェクトプロパティにアクセスする許可を持っているかどうかを検証することが重要です。



API エンドポイントは以下のような場合に脆弱となります。

* API エンドポイントは機密性が高くユーザーが読み取ってはいけないオブジェクトのプロパティを露出している場合。 (以前の名称: "[過剰なデータ露出 (Excessive Data Exposure)][1]")


* API エンドポイントはユーザーがアクセスできないはずの機密性の高いオブジェクトのプロパティの値を変更、追加、削除できる場合。 (以前の名称: "[一括割り当て (Mass Assignment)][2]")



## 攻撃シナリオの例

### シナリオ #1

ある出会い系アプリではユーザーが他のユーザーの不適切な振る舞いを報告できます。
このフローの一環として、ユーザーは「報告」ボタンをクリックし、以下のような API コールがトリガーされます。


```
POST /graphql
{
  "operationName":"reportUser",
  "variables":{
    "userId": 313,
    "reason":["offensive behavior"]
  },
  "query":"mutation reportUser($userId: ID!, $reason: String!) {
    reportUser(userId: $userId, reason: $reason) {
      status
      message
      reportedUser {
        id
        fullName
        recentLocation
      }
    }
  }"
}
```

認証済みユーザーが "fullName" や "recentLocation" などの他のユーザーがアクセスすることを想定していない機密性の高い (レポートされる) ユーザーオブジェクトプロパティにアクセスできるため、この API エンドポイントは脆弱です。



### シナリオ #2

あるタイプのユーザー (「ホスト」) が別のタイプのユーザー (「ゲスト」) に自分のアパートの貸し出しを提供するオンラインマーケットプレイスプラットフォームでは、ゲストに宿泊料金を請求する前に、ホストがゲストによる予約を受け入れることを要求されます。



このフローの一環として、ホストによって API コールが `POST /api/host/approve_booking` に以下の正当なペイロードで送信されます。


```
{
  "approved": true,
  "comment": "Check-in is after 3pm"
}
```

ホストはこの正当なリクエストをリプレイして、以下の悪意のあるペイロードを追加します。


```
{
  "approved": true,
  "comment": "Check-in is after 3pm",
  "total_stay_price": "$1,000,000"
}
```

ホストが内部オブジェクトプロパティ `total_stay_price` にアクセスする許可を持っているか検証していないので、この API エンドポイント脆弱です。そして、ゲストは想定よりも多く請求されることになります。



### シナリオ #3

ショート動画をベースとしたソーシャルネットワークでは制限的なコンテンツフィルタリングと検閲を実施しています。
アップロードされた動画がブロックされても、ユーザーは以下の API リクエストを使用して動画の説明を変更できます。


```
PUT /api/video/update_video

{
  "description": "a funny video about cats"
}
```

イライラしたユーザーは正当なリクエストをリプレイして、以下の悪意のあるペイロードを追加します。


```
{
  "description": "a funny video about cats",
  "blocked": false
}
```

ユーザーが内部オブジェクトプロパティ `blocked` にアクセスする許可を持っているか検証していないので、この API エンドポイント脆弱です。そして、ユーザーはその値を `true` から `false` に変更し、自身のブロックされたコンテンツをアンロックできます。




## 防止方法

* API エンドポイントを使用してオブジェクトを開示する場合、開示するオブジェクトのプロパティにアクセスする許可をユーザーが持っていることを常に確認します。

* `to_json()` や `to_string()` などの汎用的なメソッドの使用は避けます。
  代わりに、具体的に返したい特定のオブジェクトプロパティを厳選します。
* 可能であれば、クライアントの入力をコード変数、内部オブジェクト、オブジェクトプロパティに自動的にバインド ("Mass Assignment") する関数の使用は避けます。


* クライアントが更新すべきオブジェクトのプロパティのみ変更できるようにします。

* スキーマベースのレスポンス検証メカニズムを実装して、セキュリティの層を増やします。このメカニズムの一環として、すべての API メソッドによって返されるデータを定義して適用します。


* エンドポイントのビジネス要件や機能要件に従って、返されるデータ構造を必要最小限に抑えます。


## 参考資料

### OWASP

* [API3:2019 Excessive Data Exposure - OWASP API Security Top 10 2019][1]
* [API6:2019 - Mass Assignment - OWASP API Security Top 10 2019][2]
* [Mass Assignment Cheat Sheet][3]

### External

* [CWE-213: Exposure of Sensitive Information Due to Incompatible Policies][4]
* [CWE-915: Improperly Controlled Modification of Dynamically-Determined Object Attributes][5]

[1]: https://owasp.org/API-Security/editions/2019/en/0xa3-excessive-data-exposure/
[2]: https://owasp.org/API-Security/editions/2019/en/0xa6-mass-assignment/
[3]: https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html
[4]: https://cwe.mitre.org/data/definitions/213.html
[5]: https://cwe.mitre.org/data/definitions/915.html
