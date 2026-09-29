# セットアップ手順

CloudFront + Keycloak SSO で既存S3バケットを保護する構成([cloudformation/template.yaml](cloudformation/template.yaml))を、
ゼロからデプロイするための手順。ユーザー作成・グループ運用については別途扱う。

## 1. CFn実行前に準備しておくリソース

| # | リソース | 備考 |
|---|---|---|
| 1 | 保護対象のS3バケット(既存) | どのリージョンでもよい(このテンプレートはリージョンをパラメータで指定する)。バケットポリシーはこのスタックが上書きするので、既存のポリシーがあれば事前に内容を確認しておく |
| 2 | 公開したいコンテンツ | 最低限`index.html`をバケットにアップロードしておく(空のままだとCloudFrontアクセス時にS3が`AccessDenied`を返す) |
| 3 | Keycloakのリアルム | 既に稼働していること(discoveryエンドポイント `/.well-known/openid-configuration` が到達可能であること) |
| 4 | Keycloakのconfidential client(OIDC) | Client type: OpenID Connect / Client authentication: ON / Standard flow: 有効。Valid Redirect URIsは**デプロイ後に**追加するのでこの時点では空でよい |
| 4a | Keycloakのグループ + Group Membershipプロトコルマッパー | アクセスを許可したいユーザーが所属するグループ(例:`NRKKUsers`)を作成。④のclientのGroup Membershipプロトコルマッパーを追加し、`groups`クレームをID token/Access tokenに含める設定をする(下記2章参照) |
| 5 | CloudFrontに割り当てるカスタムドメイン | 1〜5件。**そのドメインが既存の別CloudFront配信(ワイルドカードDNS等)と衝突しないか事前に確認**(下記4章参照) |
| 6 | ACM証明書 | 対象ドメインをカバーし、**必ず us-east-1 リージョンで発行**されたもの(CloudFrontの制約。Lambda@Edgeの制約でスタック自体もus-east-1固定) |
| 7 | Route53 Hosted Zone(DNSをAWSで管理する場合のみ) | 対象ドメインを含むゾーン。手動でDNSを管理する場合は不要 |
| 8 | AWS CLIの認証情報 | 対象AWSアカウント・us-east-1で、CloudFormation / Cognito / CloudFront / S3 / IAM / Lambda / Route53 / ACM / ServerlessRepo に対する権限があること |
| 9 | Cognito Hosted UIドメインプレフィックスの候補 | 英数字と`-`のみ、グローバルに一意な文字列を決めておく(実際のドメインは `<prefix>.auth.us-east-1.amazoncognito.com`) |

## 2. CFn実行に必要な情報の収集場所

テンプレートのパラメータごとに、値の調べ方・取得場所をまとめる。

### `ExistingS3BucketName` / `ExistingS3BucketRegion`

- バケット名: 1章の②で用意したバケット名そのもの
- バケットのリージョン確認:
  ```bash
  aws s3api get-bucket-location --bucket <バケット名>
  ```
  出力の`LocationConstraint`が空(null)の場合は`us-east-1`を意味する

### `AppDomainName1`〜`AppDomainName5`

- 使いたいドメイン名を、既存の別CloudFront配信と衝突していないか事前確認する:
  ```bash
  # 現在そのドメインがどこを指しているか確認
  dig +short A <ドメイン名>
  dig +short CNAME <ドメイン名>
  ```
  何らかの`*.cloudfront.net`が返ってきた場合、既存の別配信(ワイルドカードDNS経由の場合もある)と衝突する可能性が高い。
  Route53のワイルドカードレコードの有無も確認する:
  ```bash
  aws route53 list-resource-record-sets --hosted-zone-id <Hosted Zone ID> \
    --query "ResourceRecordSets[?Type=='CNAME' || Type=='A']"
  ```
  **注意**: `*.example.com`形式のワイルドカードは、`sub.example.com`だけでなく`a.b.example.com`のような
  多階層のサブドメインにも(そこに個別レコードが無い限り)適用される。「サブドメインを1段掘れば逃げられる」
  という想定は誤り。安全に確認するには、実際に対象ドメイン名で`dig`を打ってみるのが最も確実。

### `AcmCertificateArn`

- **us-east-1**で対象ドメインをカバーする証明書があるか確認:
  ```bash
  aws acm list-certificates --region us-east-1 \
    --query "CertificateSummaryList[?DomainName=='<ドメイン名>']"
  ```
  ワイルドカード証明書(`*.example.com`)で代用できる場合もあるので、一覧を目で見て確認するのが早い:
  ```bash
  aws acm list-certificates --region us-east-1 --output table
  ```
- 無ければ新規発行(DNS検証):
  ```bash
  aws acm request-certificate \
    --region us-east-1 \
    --domain-name <ドメイン名> \
    --validation-method DNS
  ```
  発行後、検証用CNAMEレコードの情報を取得:
  ```bash
  aws acm describe-certificate --region us-east-1 --certificate-arn <取得したARN> \
    --query "Certificate.DomainValidationOptions[0].ResourceRecord"
  ```
  出力される`Name`/`Value`をRoute53(または利用中のDNS)にCNAMEとして登録し、検証完了を待つ:
  ```bash
  aws acm wait certificate-validated --region us-east-1 --certificate-arn <取得したARN>
  ```

### `HostedZoneId`

- DNSをAWSで管理する場合のみ必要:
  ```bash
  aws route53 list-hosted-zones-by-name --dns-name <ドメインの親ゾーン名> \
    --query "HostedZones[0].Id" --output text
  ```
  出力は`/hostedzone/XXXXXXXXXXXXX`形式なので、`/hostedzone/`以降の部分だけを使う。
  手動でDNSを管理する場合は空文字のままでよい(その場合、デプロイ後に自分でCNAME/Aレコードを設定する)。

### `KeycloakIssuerUrl`

- Keycloak管理コンソール → 対象リアルムの realm settings → General タブ内の
  「OpenID Endpoint Configuration」リンク先URLを開く。そのURLから末尾の
  `/.well-known/openid-configuration` を除いた部分が Issuer URL(通常
  `https://<host>/realms/<realm名>` の形)。
- 到達確認:
  ```bash
  curl -s -o /dev/null -w "HTTP %{http_code}\n" https://<host>/realms/<realm名>/.well-known/openid-configuration
  ```
  `HTTP 200`が返ることを確認する。

### `KeycloakClientId` / `KeycloakClientSecret`

- Keycloak管理コンソール → Clients → 1章④で作成したclientを選択 → **Credentials** タブに
  Client secret が表示される(表示されない場合は「Regenerate」で再生成できるが、既に使っている
  他システムがあれば影響するので注意)。
- Client ID はクライアント一覧のClient IDカラムそのもの。

### `CognitoDomainPrefix`

- 自分で決める文字列(英数字と`-`のみ)。他アカウントを含めグローバルに一意である必要がある。
  既に使われているかどうかは、実際にデプロイを試みて`DomainAlreadyExists`のようなエラーが
  出るかどうかで判断するのが確実(事前チェック用の単体APIは無い)。分かりやすい固有のプレフィックス
  (プロジェクト名+用途など)を選んでおけば衝突しにくい。

### `UserPoolGroupName`

- 自分で決めるグループ名。デフォルトは`NRKKUsers`。**Keycloak側に同名のグループを事前に
  作成しておく**こと(1章④a)。ユーザー作成・Keycloak側のグループ運用は別途相談。

### アクセス制御の仕組み(Keycloak側のグループ設定が必須)

このテンプレートはCognito側でユーザーをグループに手動追加する必要がない。代わりに
Keycloak側のグループ membership が Pre Token Generation Lambda 経由で
`cognito:groups` クレームに同期され、`UserPoolGroupName` に指定したグループ名が
含まれているかどうかでアクセス可否が判定される。

Keycloak側のclient(1章④)に **Group Membership** プロトコルマッパーを追加する必要がある:

1. Keycloak管理コンソール → Clients → 対象client → **Client scopes** タブ
2. `<clientId>-dedicated` scope を選択 → **Mappers** タブ → **Add mapper** → **By configuration**
3. **Group Membership** を選択
4. `Name`: 任意(例: `groups`)、`Token Claim Name`: `groups`、
   `Full group path`: OFF(グループ名を`/`無しでそのまま使う場合)
5. `Add to ID token` / `Add to access token` を有効化して保存

この設定を忘れると`groups`クレームがトークンに含まれず、`cognito:groups`が常に空になって
全ユーザーがアクセス拒否される。

## 3. CFnの実行方法(AWS CLI)

### 3-1. 事前確認

```bash
# 現在のデフォルト認証情報とアカウントを確認
aws sts get-caller-identity

# テンプレートの構文チェック(cfn-lint導入済みの場合)
cfn-lint cloudformation/template.yaml

# SAMテンプレートとしての妥当性チェック
sam validate --template-file cloudformation/template.yaml --lint
```

### 3-2. デプロイコマンド

**必ず `--region us-east-1` を指定する**(Lambda@Edgeの制約でスタック自体がus-east-1固定のため)。

```bash
aws cloudformation deploy \
  --region us-east-1 \
  --template-file cloudformation/template.yaml \
  --stack-name <スタック名> \
  --capabilities CAPABILITY_IAM CAPABILITY_AUTO_EXPAND \
  --parameter-overrides \
    ExistingS3BucketName=<バケット名> \
    ExistingS3BucketRegion=<バケットのリージョン> \
    AppDomainName1=<ドメイン名> \
    AcmCertificateArn=<us-east-1のACM証明書ARN> \
    HostedZoneId=<Hosted Zone ID または空文字> \
    KeycloakIssuerUrl=<Keycloakのissuer URL> \
    KeycloakClientId=<Keycloakクライアントのclient ID> \
    KeycloakClientSecret=<Keycloakクライアントのシークレット> \
    CognitoDomainPrefix=<Cognitoドメインプレフィックス> \
    UserPoolGroupName=<Keycloak側と一致させるグループ名> \
  --no-fail-on-empty-changeset
```

各フラグの意味:

- `--capabilities CAPABILITY_IAM CAPABILITY_AUTO_EXPAND`: IAMロールを作成するテンプレートであること(`CAPABILITY_IAM`)、
  かつ`AWS::Serverless::Application`(SAR)によるネストスタック展開を行うこと(`CAPABILITY_AUTO_EXPAND`)を明示的に許可する。
  これらを付けないとデプロイが拒否される
- `--no-fail-on-empty-changeset`: パラメータに変更が無く差分が発生しない場合でもエラー終了させない
  (2回目以降の再デプロイで差分が無いケースへの対応)

### 3-3. 進行状況の確認

`aws cloudformation deploy`はフォアグラウンドで実行すると完了までブロックするが、途中経過は別ターミナルから確認できる:

```bash
aws cloudformation describe-stack-events --region us-east-1 --stack-name <スタック名> \
  --query "StackEvents[0:10].[Timestamp,LogicalResourceId,ResourceStatus]" --output table
```

失敗した場合、失敗したリソースだけを抽出する:

```bash
aws cloudformation describe-stack-events --region us-east-1 --stack-name <スタック名> \
  --query "StackEvents[?contains(ResourceStatus,'FAILED')].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]" \
  --output table
```

### 3-4. デプロイ完了後、Outputsを確認する

```bash
aws cloudformation describe-stacks --region us-east-1 --stack-name <スタック名> \
  --query "Stacks[0].Outputs" --output table
```

特に以下2つは次の手動作業に必要:

- `KeycloakRedirectUriToRegister`: Keycloak管理コンソールのクライアント設定
  (Valid Redirect URIs)に追加する値
- `AppUrl` / `CloudFrontDistributionDomainName`: 動作確認用のURL

### 3-5. 再デプロイ・失敗時のやり直し

- パラメータやテンプレートを変更した場合は、同じ`aws cloudformation deploy`コマンドを
  再実行すれば差分だけが適用される(スタック名を変えなければ更新扱いになる)
- チェンジセット作成やリソース作成に失敗すると、スタックは`ROLLBACK_COMPLETE`(または
  新規作成時は`REVIEW_IN_PROGRESS`)のまま残る。この状態のスタックには**実リソースは残っていない**ため、
  一度削除してから再デプロイする:
  ```bash
  aws cloudformation describe-stacks --region us-east-1 --stack-name <スタック名> \
    --query "Stacks[0].StackStatus" --output text

  aws cloudformation delete-stack --region us-east-1 --stack-name <スタック名>
  aws cloudformation wait stack-delete-complete --region us-east-1 --stack-name <スタック名>
  ```

## 4. よくある詰まりどころ

- **ACM証明書のリージョン間違い**: CloudFront用のACM証明書は必ずus-east-1。他リージョンの証明書を
  指定すると`CloudFrontDistribution`リソースの作成で失敗する
- **S3バケットのリージョン不一致**: `ExistingS3BucketRegion`にバケットの実リージョンと異なる値を
  指定すると、CloudFrontのオリジン接続エラーになる
- **ワイルドカードDNSとの衝突**: 対象ドメインが既存の別CloudFront配信のワイルドカードDNS配下にあると、
  `CloudFrontDistribution`作成時に
  `One or more aliases specified for the distribution includes an incorrectly configured DNS record`
  というエラーになる。2章「AppDomainName」の事前確認手順で回避する
- **バケットが空**: S3にコンテンツが1つも無いと、CloudFront経由のアクセス時にS3が`AccessDenied`を返す
  (権限エラーとオブジェクト未存在を区別しないS3の仕様のため)。最低限`index.html`を置いておく
- **Keycloak側のリダイレクトURI未設定**: デプロイ直後はまだKeycloak側にCognitoのリダイレクトURIを
  登録していないため、ログイン画面で`Invalid parameter: redirect_uri`エラーになる。3-4章の
  `KeycloakRedirectUriToRegister`をKeycloakのValid Redirect URIsに追加すること
- **Group Membershipプロトコルマッパー未設定**: Keycloak側でログインは成功するが、S3コンテンツ
  アクセス時に拒否される場合、`groups`クレームがトークンに含まれていない可能性が高い。
  Keycloakのclient設定でGroup Membershipマッパーが有効か、対象ユーザーが`UserPoolGroupName`と
  同名のグループに所属しているかを確認する
