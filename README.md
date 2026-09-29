# CloudFront + Keycloak SSO で既存S3バケットを保護する

[cloudfront-authorization-at-edge](https://github.com/aws-samples/cloudfront-authorization-at-edge) (AWS SAR, v2.3.2) の
Lambda@Edge関数を利用し、Keycloak(OIDC)をIdPとしたSSOで既存のS3バケットをCloudFront経由のみ閲覧可能にする。

[![Launch Stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/quickcreate?templateURL=https%3A%2F%2Fnrkk-learn.s3.ap-northeast-1.amazonaws.com%2Fpublic%2Fcfn%2Faws-cloud-front-edge-keycloak%2Ftemplate.yaml&stackName=keycloak-protected-s3)

> **注意**: CloudFormationの「クイック作成」の`templateURL`は**S3にホストされたオブジェクトのURLのみ**を
> サポートしており、GitHubのraw URL等は使えない。このボタンはS3バケット`nrkk-learn`
> (`public/cfn/aws-cloud-front-edge-keycloak/template.yaml`)にアップロード済みのテンプレートを指している。
> テンプレートを更新した場合は、このS3オブジェクトも同期して更新すること。
> また、Lambda@Edgeの制約上デプロイ先リージョンは常に `us-east-1` 固定にしている
> (テンプレート自体はS3上のどのリージョンにあっても取得可能)。

## アーキテクチャ

- Cognito User Pool を新規作成し、Keycloak を OIDC の User Pool Identity Provider として登録
- cloudfront-authorization-at-edge はネストスタック(`AWS::Serverless::Application`)として組み込むが、
  `CreateCloudFrontDistribution: false` を指定し、Lambda@Edge関数(check-auth / parse-auth / refresh-auth /
  sign-out / http-headers)だけを流用する
- CloudFront Distribution・S3バケットポリシー(OAI)はテンプレート側で既存バケットに対して定義する
- カスタムドメイン(**最大5件**) + ACM証明書、Route53 Aliasレコード(任意)に対応
- Pre Token Generation Lambdaで、Keycloak側のグループ membership を `cognito:groups` クレームに
  同期し、指定グループに所属しているユーザーのみアクセスを許可する

## 前提条件

- **us-east-1リージョンにのみデプロイ可能**(Lambda@Edgeの制約)
- ACM証明書は **us-east-1** で発行されたもの
- 保護対象の既存S3バケットは同じAWSアカウント内に存在すること
- Keycloak側に confidential client (Client Authentication ON) が作成済みであること
- Keycloak側で、アクセスを許可したいユーザーが所属するグループ(例: `NRKKUsers`)を作成済みであること。
  さらに、そのclientに **Group Membership** プロトコルマッパーを追加し、`groups` クレームを
  ID token / Access token に含める設定が必要(Client scopes → 対象scope → Mappers →
  Add mapper → By configuration → Group Membership。Token Claim Name は `groups`)

## デプロイ手順

```bash
aws cloudformation deploy \
  --region us-east-1 \
  --template-file cloudformation/template.yaml \
  --stack-name my-keycloak-protected-s3 \
  --capabilities CAPABILITY_IAM CAPABILITY_AUTO_EXPAND \
  --parameter-overrides \
    ExistingS3BucketName=my-existing-bucket \
    ExistingS3BucketRegion=ap-northeast-1 \
    AppDomainName1=files.example.com \
    AppDomainName2=files2.example.com \
    AcmCertificateArn=arn:aws:acm:us-east-1:123456789012:certificate/xxxxxxxx \
    HostedZoneId=Z0123456789ABCDEFGHIJ \
    KeycloakIssuerUrl=https://keycloak.example.com/realms/myrealm \
    KeycloakClientId=my-client-id \
    KeycloakClientSecret=my-client-secret \
    CognitoDomainPrefix=my-app-auth \
    UserPoolGroupName=NRKKUsers
```

`AppDomainName1`(必須)〜`AppDomainName5`(任意)で最大5件のドメインを指定できる。
使わないスロットは省略(デフォルト空文字)してよい。指定した全ドメインは同一のACM証明書(SAN)で
カバーする必要がある。

`HostedZoneId` を省略(空文字)する場合はDNSレコードを手動で設定すること。指定した場合、
`AppDomainName1`〜`5`に含む全ドメインが同一のHosted Zoneに属している前提でAliasレコードを
まとめて作成する。

## デプロイ後に必要な手動作業

### 1. Keycloak側のリダイレクトURI登録

デプロイ後のスタックOutput `KeycloakRedirectUriToRegister` の値
(`https://<CognitoDomainPrefix>.auth.us-east-1.amazoncognito.com/oauth2/idpresponse`)を、
Keycloak管理コンソールの対象クライアントの **Valid Redirect URIs** に追加する。

### 2. アクセス制御について

このテンプレートは、Keycloak側のグループ membership を Cognitoの`cognito:groups`クレームに
同期する(Pre Token Generation Lambda)ことでアクセス制御している。具体的には:

1. Keycloakでログインすると、そのユーザーが所属するグループ一覧が `groups` クレームとして
   IDトークンに含まれる(前提条件で設定したプロトコルマッパーによる)
2. CognitoのAttributeMapping経由で、この値がログインごとに `custom:groups` 属性に反映される
3. Pre Token Generation Lambda が `custom:groups` を読み取り、そのままJWTの`cognito:groups`
   クレームに上書き注入する
4. `cloudfront-authorization-at-edge` が `UserPoolGroupName`(デプロイ時に指定したグループ名、
   既定値`NRKKUsers`)がその`cognito:groups`に含まれているかを判定し、アクセス可否を決める

つまり**Cognito側でのグループ管理は不要で、Keycloak側でユーザーをグループに追加/削除するだけで
アクセス許可が同期される**。前提条件のプロトコルマッパー設定を忘れると`groups`クレームが
送られず、常にアクセス拒否になるので注意。

### 3. 動作確認

`AppUrl` Outputの値 (`AppDomainName1`のURL) にアクセスし、Keycloakのログイン画面に
リダイレクトされることを確認する。ログイン後、`UserPoolGroupName`に指定したグループに
Keycloak側で所属しているユーザーはコンテンツが表示され、所属していないユーザーは
アクセスが拒否されることを確認する。

## 既知の注意点

- `ExistingBucketPolicy` リソースはバケットポリシーを**上書き**する。既存バケットに他の
  バケットポリシーが設定済みの場合は、デプロイ前に内容を確認し、必要な文を
  `cloudformation/template.yaml` の `ExistingBucketPolicy` に統合すること。
- `KeycloakIssuerUrl` は Keycloak の `/.well-known/openid-configuration` が実際に応答する
  Issuer と一致させること(末尾スラッシュの有無に注意)。
- 本構成はOrigin Access Identity(OAI)を使用している。これはaws-samples公式の
  持ち込みCloudFront配信サンプル(`example-serverless-app-reuse/reuse-auth-only.yaml`)に
  準拠した構成。OACへの切り替えも可能だが未検証。
