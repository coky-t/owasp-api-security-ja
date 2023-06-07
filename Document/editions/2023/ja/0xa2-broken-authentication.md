# API2:2023 認証の不備 (Broken Authentication)

| 脅威エージェント/攻撃手法 | セキュリティ上の弱点 | 影響 |
| - | - | - |
| API 依存 : 悪用難易度 **容易** | 普及度 **普通** : 検出難易度 **容易** | 技術的影響 **重度** : ビジネス依存 |
| 認証メカニズムはすべての人に公開されているため、攻撃者にとって格好の標的となります。認証の問題によっては悪用するためにより高度な技術スキルが必要になることもありますが、エクスプロイトツールは一般的に入手可能です。 | 認証境界や固有の実装の複雑さに関するソフトウェアおよびセキュリティのエンジニアの誤解が認証の問題を蔓延させています。認証の不備を検出する方法論は利用可能であり、作成も簡単です。 | 攻撃者はシステム内の他のユーザーのアカウントを完全に制御し、ユーザーの個人データを読み取り、ユーザーに代わって機密性の高いアクションを実行できます。システムは攻撃者のアクションと正当なユーザーのアクションを区別できそうにありません。 |

## その API は脆弱か？

認証エンドポイントとフローは保護する必要がある資産です。
さらに、「パスワード忘れ/パスワードリセット」も認証メカニズムと同様に扱う必要があります。


API は以下のような場合に脆弱となります。

* 攻撃者が有効なユーザー名とパスワードのリストでブルートフォースを行うクレデンシャルスタッフィングを許可している場合。

* CAPTCHA やアカウントロックアウトメカニズムを提示することなく、攻撃者が同じユーザーアカウントでブルートフォース攻撃を実行することを許可している場合。

* 弱いパスワードを許可している場合。
* URL 内の認証トークンやパスワードなど、機密性の高い認証情報を送信している場合。

* パスワードの確認を求めることなく、ユーザーが電子メールアドレスや現在のパスワードを変更したり、その他の任意の機密性の高い操作を行うことができる場合。

* トークンの真正性を検証していない場合。
* 未署名や弱い署名の JWT トークン (`{"alg":"none"}`) を受け入れる場合。
* JWT の有効期限を検証していない場合。
* パスワードをプレーンテキストか、暗号化されていないか、弱いハッシュ化で使用している場合。
* 弱い暗号化鍵を使用している場合。

そのうえで、マイクロサービスは以下のような場合に脆弱となります。

* 他のマイクロサービスが認証なしでアクセスできる場合。
* 弱いトークンや予測可能なトークンを使用して認証を実施している場合。

## 攻撃シナリオの例

## シナリオ #1

ユーザー認証を行うために、クライアントはユーザー認証情報とともに以下のような API リクエストを発行する必要があります。


```
POST /graphql
{
  "query":"mutation {
    login (username:\"<username>\",password:\"<password>\") {
      token
    }
   }"
}
```

認証情報が有効な場合、認証トークンが返されます。この認証トークンはユーザーを識別するためにその後のリクエストで提供される必要があります。
ログイン試行は回数制限があり、リクエストは一分間に三回までしかできません。



被害者のアカウントでログインをブルートフォースするために、攻撃者は GraphQL クエリのバッチ処理を利用してリクエストの回数制限を回避し、攻撃を高速化します。


```
POST /graphql
[
  {"query":"mutation{login(username:\"victim\",password:\"password\"){token}}"},
  {"query":"mutation{login(username:\"victim\",password:\"123456\"){token}}"},
  {"query":"mutation{login(username:\"victim\",password:\"qwerty\"){token}}"},
  ...
  {"query":"mutation{login(username:\"victim\",password:\"123\"){token}}"},
]
```

## シナリオ #2

ユーザーのアカウントに関連付けられている電子メールアドレスを更新するために、クライアントは以下のような API リクエストを発行する必要があります。


```
PUT /account
Authorization: Bearer <token>

{ "email": "<new_email_address>" }
```

この API では現在のパスワードを提供して本人であることを確認することをユーザーに要求しないため、攻撃者は認証トークンを盗むことができます。被害者のアカウントの電子メールアドレスを更新した後にパスワードリセットのワークフローを開始することで、被害者のアカウントを乗っ取ることができるかもしれません。





## 防止方法

* API への認証に使用できるすべてのフロー (ワンクリック認証を実装したモバイル/ウェブ/ディープリンクなど) を確認します。あなたがどのフローを見逃したかエンジニアに尋ねてください。


* 認証メカニズムについて読みます。何がどのように使用されているかを理解しましょう。OAuth は認証ではありませんし、API キーも違います。

* 認証、トークン生成、パスワード保存において車輪の再発明をしてはいけません。標準規格を使用しましょう。

* 認証情報リカバリやパスワード忘れのエンドポイントはブルートフォース、回数制限、ロックアウト保護の観点からログインエンドポイントとして扱う必要があります。

* 機密性の高い操作 (アカウント所有者の電子メールアドレスや二要素認証電話番号の変更など) には再認証を要求します。

* [OWASP Authentication Cheatsheet][1] を使用します。
* 可能であれば、多要素認証を実装します。
* 認証エンドポイントでのクレデンシャルスタッフィング、辞書攻撃、ブルートフォース攻撃を軽減するためにアンチブルートフォースメカニズムを実装します。このメカニズムは API の通常の回数制限メカニズムよりも厳しいものにすべきです。



* 特定のユーザーに対するブルートフォース攻撃を防ぐために [アカウントロックアウト][2] や CAPTCHA のメカニズムを実装します。弱いパスワードのチェックを実装します。

* API キーはユーザー認証に使用すべきではありません。 [API クライアント][3] 認証にのみ使用すべきです。


## 参考資料

### OWASP

* [Authentication Cheat Sheet][1]
* [Key Management Cheat Sheet][4]
* [Credential Stuffing][5]

### その他

* [CWE-204: Observable Response Discrepancy][6]
* [CWE-307: Improper Restriction of Excessive Authentication Attempts][7]

[1]: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
[2]: https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/04-Authentication_Testing/03-Testing_for_Weak_Lock_Out_Mechanism(OTG-AUTHN-003)
[3]: https://cloud.google.com/endpoints/docs/openapi/when-why-api-key
[4]: https://cheatsheetseries.owasp.org/cheatsheets/Key_Management_Cheat_Sheet.html
[5]: https://owasp.org/www-community/attacks/Credential_stuffing
[6]: https://cwe.mitre.org/data/definitions/204.html
[7]: https://cwe.mitre.org/data/definitions/307.html
