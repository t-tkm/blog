+++ 
Categories = ["AWS"] 
Tags = ["AWS", "ツール", "aws-cli", "通知", "費用"] 
date = "2026-03-22T15:00:00+09:00" 
# lastmod = "2025-10-03T17:00:00+09:00"
title = "AWS費用監視ツール（前編：AWSマネジメントコンソール & AWS CLI）" 
archives = ["2026", "2026-03", "2026-03-22"]
+++

> **この記事は [2021年版](https://qiita.com/t-taku/items/ef0e7edc79f89929d466) を最新のAWSベストプラクティスに沿ってアップデートしたものです。**

## 1. はじめに
AWSを使い始めると「気づいたら思わぬ費用が発生していた」という経験をしたことがある方も多いと思います。この背景のもと、2021年当時、AWSの経験が浅いエンジニア向けに、AWS CLIを使ったコスト確認、Lambdaを使ったデイリー通知の方法を紹介しました。

当時の記事では、IAMユーザーのアクセスキーを使ったAWS CLIのセットアップ方法を紹介しましたが、現在では、長期的なアクセスキー（Long-term credentials）の使用はAWSの公式セキュリティガイドラインで非推奨となっています。

本記事では、アクセスキーを使わず、**IAM Identity Center（旧 AWS SSO）** を使った認証へ移行する方式で、記事を更新しようと思います。

## 2. 主な変更点(2021年版から)
| 項目 | 2021年版 | 2025年版 |
|------|----------|----------|
| IAM認証 | IAMユーザー+アクセスキー | IAM Identity Center（SSO+一時認証情報 |
| AWS CLI バージョン | v1/v2（混在） | AWS CLI v2のみ |
| 認証情報の設定 | `aws configure` | `aws configure sso` |

## 3. IAM設定更新
### 3.1 なぜアクセスキーを使わない方式へ更新するのか
2021年版では`aws_access_key_id`と`aws_secret_access_key`を使うIAMユーザーを作成していました。しかし、これらは長期的に有効な認証情報であり、以下のリスクがあります。
- 誤ってGitHubなどにコミットすると、第三者に悪用される
- 定期的なローテーションが必要だが、実運用では忘れがち
- AWSの公式Best PracticesでもLong-term credentialsの使用は非推奨

### 3.2 IAM Identity Centerの設定
IAM Identity Centerを使うと、一時的な認証情報(最長12時間)が自動的に発行・更新されるため、アクセスキーの管理が不要になります。

#### 3.2.1 IAM Identity Centerの有効化
1. AWSマネジメントコンソール → IAM Identity Centerを開く
2. 「有効にする」をクリック（初回のみ）
3. アイデンティティソースを設定（ex. 今回はAWS管理のIdentity Centerディレクトリを活用）

#### 3.2.2 ユーザーとグループの作成
IAM Identity Center > ユーザー > ユーザーを追加
- ユーザー名例：`billing-user`
- メールアドレスを登録すると招待メールが届きます
- グループ(複数人に同じ権限を付けたい場合。今回は個人で使うだけなのでグループは作らず、ユーザーを直接割り当て)

#### 3.2.3 許可セット（Permission Set）の作成
コスト確認に必要な最小権限を持つ許可セットを作成します。

- 概要
  - ここで作成する(インライン)ポリシーは、AWSの請求情報およびコスト分析機能を閲覧専用(ReadOnly)で利用するための権限セットです。開発者や管理者がコスト状況を可視化することを目的とし、リソースの変更や課金設定の変更はできません。

- 対象機能
  - Billing（請求・明細）
  - Cost Explorer（コスト分析）
  - Budgets（予算確認）

- 操作
  - IAM Identity Center > 許可セット > 許可セットを作成
    
    「カスタム許可セット」を選択し、インラインポリシーに以下を追記

- ポリシー内容
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "BillingAndCostExplorerReadOnly",
        "Effect": "Allow",
        "Action": [
          "aws-portal:ViewBilling",
          "aws-portal:ViewUsage",
          "billing:Get*",
          "ce:Get*",
          "ce:Describe*",
          "budgets:ViewBudget"
        ],
        "Resource": "*"
      }
    ]
  }
  ```

ポイントは、必要最小限の権限（最小権限の原則）を付与することです。`AdministratorAccess` を付与するのはアンチパターンですので避けましょう。ワイルドカード（`ce:Get*` 等）は読み取り範囲が広いので、用途に応じて [Cost Explorer の API 一覧](https://docs.aws.amazon.com/aws-cost-management/latest/APIReference/API_Operations_AWS_Cost_Explorer_Service.html)を参照しながら個別アクションへ絞る検討もできます。

#### 3.2.4 AWSアカウントへのアクセス割り当て
- IAM Identity Center > AWSアカウント > 対象アカウントを選択

  「ユーザーまたはグループを割り当て」で、 作成したユーザー（またはグループ）+ 許可セットを紐付ける

## 4. AWS CLIのセットアップ（v2 + SSO）
### 4.1 AWS CLI v2 のインストール
macOS、Linux、Windowsそれぞれの公式インストーラーを使います。
```bash
# macOS（Homebrew）
brew install awscli

# バージョン確認
aws --version
# aws-cli/2.x.x Python/3.x.x ...
```

公式ドキュメント：[AWS CLI v2 インストールガイド](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

### 4.2 環境変数
以降、`aws`の例ではSSOプロファイル名を`billing-user`に統一する。プロファイル本体は 4.3 節で `aws configure sso --profile billing-user` により作成するが、先にシェル環境を整えておくと、`--profile` の繰り返しやページャで止まる挙動を避けやすいので、このタイミングでシェルに設定しておく。

#### 4.2.1 AWS_PROFILE
`--profile` を省略するための環境変数。
```bash
export AWS_PROFILE=billing-user
```

設定後は `aws sts get-caller-identity` のように、引数にプロファイル名を付けなくても本記事で使う`billing-user`が使われる(SSO ログイン済みであることが前提)。

#### 4.2.2 AWS_PAGER
ページャを無効にする（`q`で終了が面倒なとき）ための環境変数。AWS CLI v2は、出力が一定以上あると既定でページャ(多くの環境では`less`)に渡す。画面下に`(END)`と出て`q`で抜ける挙動。これが煩わしいのでOFFにする設定。
```bash
export AWS_PAGER=cat
```

`cat`は「そのまま標準出力に流す」指定。空文字`AWS_PAGER=""`でもページャは使われません。

### 4.3 SSOプロファイルの設定
アクセスキーの代わりに`aws configure sso`を使います。プロファイル名を先に決められるので、次のように`--profile`を付ける方法を推奨します。以降の例ではCLI プロファイル名を `billing-user`に統一する。

SSO session nameはプロファイル名と別のラベルにするのがよい。
```bash
aws configure sso --profile billing-user
```

対話形式で以下を入力します。

```
SSO session name (Recommended): my-sso
SSO start URL [None]: https://<your-domain>.awsapps.com/start
SSO region [None]: ap-northeast-1
SSO registration scopes [sso:account:access]: （Enterでデフォルト）
```

> **SSO session name** は `~/.aws/config` 内の `[sso-session …]` のラベルです。プロファイル名と同じにする必要はありません。

ブラウザが自動的に開き、IAM Identity CenterのAWSアクセスポータルで認証します。認証後、ターミナルに戻ると続きが表示されます。

```
Default client Region [ap-northeast-1]: （利用リージョン。Enterで既定でも可）
CLI default output format (json if not specified) [None]: json
```
#### 4.3.1 `~/.aws/config` に追記される内容
`aws configure sso --profile billing-user`を完了すると、概ね次の2つのブロックが書かれます。

次の例は説明用で、`sso_account_id`や`sso_start_url`の値はご自身の環境のものに読み替えてください。

```ini
[profile billing-user]
sso_session = my-sso
sso_account_id = 123456789012
sso_role_name = t-tkm-billing-permission-set
region = ap-northeast-1
output = json

[sso-session my-sso]
sso_start_url = https://d-xxxxxxxxxx.awsapps.com/start/
sso_region = ap-northeast-1
sso_registration_scopes = sso:account:access
```

### 4.4 SSOログインと接続確認
```bash
# SSOログイン（ブラウザが開く）
aws sso login

# 接続確認（認証が通り、どのロールでいるかを表示）
aws sts get-caller-identity
```

> 一時認証情報はデフォルトで8時間有効です。期限切れ後は再度`aws sso login`を実行してください。

## 5. AWSマネジメントコンソールでのコスト確認
### 5.1 Cost Explorer
マネジメントコンソール → Cost Management → Cost Explorerでコスト確認ができます。

#### 5.1.1 主な確認ポイント
- **期間指定**：日次・月次・年次を選択可能。デフォルトは当月
- **グループ化条件**：サービス別、リージョン別、タグ別など
- **フィルター**：料金タイプ（クレジット・使用料・税金）でフィルタリング可能

#### 5.1.2 クレジット適用前後のコスト確認
2021年版でも触れていましたが、2025年現在も同様の確認方法です。
- **クレジット適用後（デフォルト）**：フィルターなしで表示されるのがクレジット適用済みの料金
- **クレジット適用前**：「料金タイプ」フィルターで「クレジット」を除外することで確認可能

> **補足**
AWS Budgetsは、設定した予算に対してしきい値を超えそうなときに メール等で通知する機能です。マネコンではCost Management → Budgetsから予算を作成する。過去の内訳は Cost Explorer、将来の「はみ出し予告」に近い使い方ができる。AWS Cost Anomaly Detectionは、コストの異常な増加を検知してアラートするサービス（Cost Management 内）。

## 6. AWS CLIでのコスト確認
4.2 節のとおり、`export AWS_PROFILE=billing-user`を済ませてあるものとする（付いていない場合は各コマンドに`--profile billing-user`を足す）。

### 6.1 基本コマンド：get-cost-and-usage
5章のマネコン（GUI）で確認したのと同じ期間のデータを、AWS CLIで取得してみます。

まず、期間をシェル変数に設定します。

```bash
START=2026-02-01
END=2026-03-01   # Cost Explorer は End 日を含まないため「翌月1日」にする
```

基本的なコマンドは `aws ce get-cost-and-usage` です。

```bash
aws ce get-cost-and-usage \
  --time-period Start=${START},End=${END} \
  --granularity MONTHLY \
  --metrics "AmortizedCost"
```

JSON形式で期間のAWS費用合計が得られます。

```json
{
    "ResultsByTime": [
        {
            "TimePeriod": {
                "Start": "2026-02-01",
                "End": "2026-03-01"
            },
            "Total": {
                "AmortizedCost": {
                    "Amount": "209.1272342834",
                    "Unit": "USD"
                }
            },
            "Groups": [],
            "Estimated": false
        }
    ],
    "DimensionValueAttributes": []
}
```

### 6.2 必要な値のみ抽出する

`--query` と `--output text` を組み合わせてコスト金額のみ取り出します。

```bash
aws ce get-cost-and-usage \
  --time-period Start=${START},End=${END} \
  --granularity MONTHLY \
  --metrics "AmortizedCost" \
  --query "ResultsByTime[].Total[].AmortizedCost.Amount" \
  --output text
# 209.1272342834
```

### 6.3 クレジット適用前のコストを取得する

クレジットを除いた使用料を取得するには、`--filter` オプションを使います。`filter.json` というファイルを作成します。

```json
{
    "Not": {
        "Dimensions": {
            "Key": "RECORD_TYPE",
            "Values": [
                "Credit"
            ]
        }
    }
}
```

フィルターを適用してコマンドを実行します。**`file://filter.json` はカレントディレクトリの相対パス**です。上記 JSON を **`filter.json` として保存したディレクトリに `cd` してから**実行するか、`file:///絶対パス/filter.json` を指定してください。ファイルが無いと `No such file or directory: 'filter.json'` になります。

```bash
aws ce get-cost-and-usage \
  --time-period Start=${START},End=${END} \
  --granularity MONTHLY \
  --metrics "AmortizedCost" \
  --filter file://filter.json \
  --query "ResultsByTime[].Total[].AmortizedCost.Amount" \
  --output text
# 440.2072342834
```

クレジット適用前のAWS料金が取得できました。

> **補足**
>`aws ce get-cost-and-usage`などのCost Explorer APIは、**1リクエストあたり $0.01 の料金**が発生します。1日1回程度の呼び出しなら問題ありませんが、Lambda 等で短い間隔で何度も叩くと料金とスロットリングの両方で痛くなります。

### 6.4 【おまけ】サービス別コストをjqで整形する

`jq` コマンドを使うと、サービス別のコスト内訳を見やすく整形できます。

```bash
aws ce get-cost-and-usage \
  --time-period Start=${START},End=${END} \
  --granularity MONTHLY \
  --metrics "AmortizedCost" \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query "ResultsByTime[].Groups[].[Keys[0],Metrics.AmortizedCost.Amount]" \
  --output json | jq -r '.[] | "\(.[0]): $\(.[1] | tonumber | . * 100 | round / 100)"' | sort -t$ -k2 -rn | head -10
```

上記と同じ `filter.json`（クレジット除外）を付けた版は次のとおりです。
```bash
aws ce get-cost-and-usage \
  --time-period Start=${START},End=${END} \
  --granularity MONTHLY \
  --metrics "AmortizedCost" \
  --group-by Type=DIMENSION,Key=SERVICE \
  --filter file://filter.json \
  --query "ResultsByTime[].Groups[].[Keys[0],Metrics.AmortizedCost.Amount]" \
  --output json | jq -r '.[] | "\(.[0]): $\(.[1] | tonumber | . * 100 | round / 100)"' | sort -t$ -k2 -rn | head -10
```

出力例（上位10サービス）：

```
Amazon EC2: $120.50
Amazon RDS: $45.20
Amazon S3: $12.80
...
```

## 7. まとめと次のステップ
本記事では、2026年のAWSベストプラクティスに沿った形でコスト監視の基礎を解説しました。後編も、本記事同様に現時点での更新を解説します。

## 8. 参考リンク
- [AWS CLI v2 インストールガイド](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [IAM Identity Center で AWS CLI を設定する](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html)
- [Security best practices in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [Best practices for the AWS Cost Explorer API](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-api-best-practices.html)
- [get-cost-and-usage — AWS CLI リファレンス](https://docs.aws.amazon.com/cli/latest/reference/ce/get-cost-and-usage.html)

*Amazon Web Services、およびその他のAWS商標は、米国およびその他の諸国におけるAmazon.com, Inc.またはその関連会社の商標です。*
