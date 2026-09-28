# CloudFront + Keycloak SSO で既存S3バケットを保護する

[cloudfront-authorization-at-edge](https://github.com/aws-samples/cloudfront-authorization-at-edge) (AWS SAR, v2.3.2) の
Lambda@Edge関数を利用し、Keycloak(OIDC)をIdPとしたSSOで既存のS3バケットをCloudFront経由のみ閲覧可能にする。

[![Launch Stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/quickcreate?templateURL=https%3A%2F%2Fraw.githubusercontent.com%2Fnewrelickk-learn%2Faws-cloud-front-edge-keycloak%2Fmain%2Fcloudformation%2Ftemplate.yaml&stackName=keycloak-protected-s3)

> **注意**: このボタンはリポジトリが **public** になっている前提で動作する。CloudFormationサービスが
> `templateURL` を認証なしでフェッチするため、privateリポジトリのままだとスタック作成時にテンプレート取得エラーになる。
> また、Lambda@Edgeの制約上デプロイ先リージョンは常に `us-east-1` 固定にしている。

## アーキテクチャ

- Cognito User Pool を新規作成し、Keycloak を OIDC の User Pool Identity Provider として登録
- cloudfront-authorization-at-edge はネストスタック(`AWS::Serverless::Application`)として組み込むが、
  `CreateCloudFrontDistribution: false` を指定し、Lambda@Edge関数(check-auth / parse-auth / refresh-auth /
  sign-out / http-headers)だけを流用する
- CloudFront Distribution・S3バケットポリシー(OAI)はテンプレート側で既存バケットに対して定義する
- カスタムドメイン(**最大5件**) + ACM証明書、Route53 Aliasレコード(任意)に対応

## 前提条件

- **us-east-1リージョンにのみデプロイ可能**(Lambda@Edgeの制約)
- ACM証明書は **us-east-1** で発行されたもの
- 保護対象の既存S3バケットは同じAWSアカウント内に存在すること
- Keycloak側に confidential client (Client Authentication ON) が作成済みであること

## デプロイ手順

```bash
aws cloudformation deploy \
  --region us-east-1 \
  --template-file cloudformation/template.yaml \
  --stack-name my-keycloak-protected-s3 \
  --capabilities CAPABILITY_IAM CAPABILITY_AUTO_EXPAND \
  --parameter-overrides \
    ExistingS3BucketName=my-existing-bucket \
    AppDomainNames="files.example.com\,files2.example.com" \
    AcmCertificateArn=arn:aws:acm:us-east-1:123456789012:certificate/xxxxxxxx \
    HostedZoneId=Z0123456789ABCDEFGHIJ \
    KeycloakIssuerUrl=https://keycloak.example.com/realms/myrealm \
    KeycloakClientId=my-client-id \
    KeycloakClientSecret=my-client-secret \
    CognitoDomainPrefix=my-app-auth \
    UserPoolGroupName=NRKKUsers
```

`AppDomainNames` は最大5件までカンマ区切りで指定可能(1件のみでも良い)。AWS CLIの
`--parameter-overrides` にカンマ区切り値を渡す場合、シェルにカンマを解釈させないよう
`\,` でエスケープするか、パラメータファイル(`--parameter-overrides file://params.json`)を
使うこと。指定した全ドメインは同一のACM証明書(SAN)でカバーする必要がある。

`HostedZoneId` を省略(空文字)する場合はDNSレコードを手動で設定すること。指定した場合、
`AppDomainNames` に含む全ドメインが同一のHosted Zoneに属している前提でAliasレコードを
まとめて作成する。

## デプロイ後に必要な手動作業

### 1. Keycloak側のリダイレクトURI登録

デプロイ後のスタックOutput `KeycloakRedirectUriToRegister` の値
(`https://<CognitoDomainPrefix>.auth.us-east-1.amazoncognito.com/oauth2/idpresponse`)を、
Keycloak管理コンソールの対象クライアントの **Valid Redirect URIs** に追加する。

### 2. アクセス許可ユーザーの追加

Keycloak認証を通過しただけでは誰でもアクセス可能になってしまうため、
cloudfront-authorization-at-edge は `UserPoolGroupName` で指定したCognitoグループに
所属するユーザーのみアクセスを許可する仕組みになっている。

Cognito User Pool (`CognitoUserPoolId` Output) の `NRKKUsers`(既定値) グループに、
アクセスを許可したいユーザーを追加する。Keycloak経由でログインしたユーザーは
初回ログイン時にUser Pool上にユーザーが作成されるため、それ以降にグループへ追加する。

### 3. 動作確認

`AppUrl` Outputの値 (`AppDomainNames`の1件目のドメインのURL) にアクセスし、Keycloakのログイン画面に
リダイレクトされることを確認する。ログイン後、グループに所属していないユーザーは
アクセスを拒否されることも確認する。

## 既知の注意点

- `ExistingBucketPolicy` リソースはバケットポリシーを**上書き**する。既存バケットに他の
  バケットポリシーが設定済みの場合は、デプロイ前に内容を確認し、必要な文を
  `cloudformation/template.yaml` の `ExistingBucketPolicy` に統合すること。
- `KeycloakIssuerUrl` は Keycloak の `/.well-known/openid-configuration` が実際に応答する
  Issuer と一致させること(末尾スラッシュの有無に注意)。
- 本構成はOrigin Access Identity(OAI)を使用している。これはaws-samples公式の
  持ち込みCloudFront配信サンプル(`example-serverless-app-reuse/reuse-auth-only.yaml`)に
  準拠した構成。OACへの切り替えも可能だが未検証。
