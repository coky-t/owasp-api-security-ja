# API7:2023 サーバーサイドリクエストフォージェリ (Server Side Request Forgery)

| 脅威エージェント/攻撃手法 | セキュリティ上の弱点 | 影響 |
| - | - | - |
| API 依存 : 悪用難易度 **容易** | 普及度 **普通** : 検出難易度 **容易** | 技術的影響 **中程度** : ビジネス依存 |
| 悪用にはクライアントが提供する URI にアクセスする API エンドポイントを攻撃者が見つける必要があります。一般的に、基本的な SSRF (レスポンスが攻撃者に返される場合) は、攻撃が成功したかどうかについて攻撃者にフィードバックがないブラインド SSRF よりも悪用しやすくなります。 | アプリケーション開発における最近のコンセプトでは開発者はクライアントが提供する URI にアクセスすることを推奨しています。そのような URI の検証が欠如していたり不適切であることがよくある問題です。問題を検出するには、定期的な API リクエストとレスポンスの解析が必要です。レスポンスが返されない場合 (ブラインド SSRF)、脆弱性を検出するにはより多くの労力と創造性が必要になります。 | 悪用に成功すると、内部サービスの列挙 (ポートスキャンなど) 、情報漏洩、ファイアウォールやその他のセキュリティメカニズムのバイパスにつながる可能性があります。場合によっては、DoS が発生したり、サーバーが悪意のあるアクティビティを隠すためのプロキシとして使用される可能性もあります。 |

## その API は脆弱か？

サーバーサイドリクエストフォージェリ (SSRF) の欠陥はユーザーが提供する URL を検証せずにリモートリソースを API が取得する際に発生します。
これによりファイアウォールや VPN で保護されている場合でも、攻撃者はアプリケーションを強制して細工したリクエストを予期しない宛先に送信できます。



アプリケーション開発における最近のコンセプトにより SSRF はより一般的でより危険なものになっています。


より一般的に - Webhook、URL からのファイルフェッチ、カスタム SSO、URL プレビューなどのコンセプトは開発者がユーザーの入力に基づいて外部リソースにアクセスすることを推奨します。



より危険に - クラウドプロバイダ、Kubernetes、Docker などの最近のテクノロジは予測可能でよく知られたパスで HTTP を介して管理チャンネルと制御チャンネルを公開します。
これらのチャンネルは SSRF 攻撃の格好のターゲットになります。


また、最近のアプリケーションは接続されていることが自然であるため、アプリケーションのアウトバウンドトラフィックを制限することはより困難です。


SSRF のリスクは完全に排除できるとは限りません。
保護メカニズムを選択する際には、ビジネスリスクとニーズを考慮することが重要です。

## 攻撃シナリオの例

### シナリオ #1

あるソーシャルネットワークではユーザーがプロフィール画像をアップロードできます。
ユーザーは自分のマシンから画像ファイルをアップロードするか、画像の URL を提供するかのいずれかを選択できます。
二つ目を選択すると、以下の API コールをトリガーします。

```
POST /api/profile/upload_picture

{
  "picture_url": "http://example.com/profile_pic.jpg"
}
```

攻撃者は悪意のある URL を送信し、API エンドポイント使用して内部ネットワーク内でポートスキャンを開始できます。


```
{
  "picture_url": "localhost:8080"
}
```

レスポンスタイムから攻撃者はポートが開いているかどうかを把握できます。


### シナリオ #2

あるセキュリティ製品ではネットワークで異常を検出するとイベントを生成します。
チームによっては、SIEM (Security Information and Event Management) などのより広範で一般的な監視システムでイベントを確認することを好みます。
この目的のために、この製品は Webhook を使用して他のシステムとの統合を提供します。


新しい Webhook の作成の一環として、GraphQL 情報が SIEM API の URL とともに送信されます。


```
POST /graphql

[
  {
    "variables": {},
    "query": "mutation {
      createNotificationChannel(input: {
        channelName: \"ch_piney\",
        notificationChannelConfig: {
          customWebhookChannelConfigs: [
            {
              url: \"http://www.siem-system.com/create_new_event\",
              send_test_req: true
            }
          ]
    	  }
  	  }){
    	channelId
  	}
	}"
  }
]

```

この作成プロセスでは、API バックエンドが提供された Webhook URL にテストリクエストを送信し、ユーザーにレスポンスを提示します。


攻撃者はこのフローを活用し、認証情報を公開する内部クラウドメタデータなどの機密リソースを API にリクエストできます。


```
POST /graphql

[
  {
    "variables": {},
    "query": "mutation {
      createNotificationChannel(input: {
        channelName: \"ch_piney\",
        notificationChannelConfig: {
          customWebhookChannelConfigs: [
            {
              url: \"http://169.254.169.254/latest/meta-data/iam/security-credentials/ec2-default-ssm\",
              send_test_req: true
            }
          ]
        }
      }) {
        channelId
      }
    }
  }
]
```

アプリケーションはテストリクエストからのレスポンスを表示するので、攻撃者はクラウド環境の認証情報を閲覧できます。


## 防止方法

* ネットワーク内のリソース取得メカニズムを分離します。通常、この機能はリモートリソースを取得するためのものであり、内部リソースではありません。

* 可能な限り、以下の許可リストを使用します。
  * ユーザーがリソースをダウンロードすることが想定されるリモートオリジン (Google ドライブ、Gravatar など)

  * URL スキーマとポート
  * 特定の機能で受け入れられるメディアタイプ
* HTTP リダイレクトを無効にします。
* URL パースの不一致によって引き起こされる問題を避けるために、十分にテストされ保守されている URL パーサーを使用します。

* クライアントから提供されるすべての入力データを検証してサニタイズします。
* クライアントに未加工のレスポンスを送信してはいけません。

## 参考資料

### OWASP

* [Server Side Request Forgery][1]
* [Server-Side Request Forgery Prevention Cheat Sheet][2]

### その他

* [CWE-918: Server-Side Request Forgery (SSRF)][3]
* [URL confusion vulnerabilities in the wild: Exploring parser inconsistencies,
   Snyk][4]

[1]: https://owasp.org/www-community/attacks/Server_Side_Request_Forgery
[2]: https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
[3]: https://cwe.mitre.org/data/definitions/918.html
[4]: https://snyk.io/blog/url-confusion-vulnerabilities/
