+++ 
Categories = ["AWS"] 
Tags = ["AWS", "ツール", "aws-cli", "通知", "費用"] 
date = "2026-03-28T15:00:00+09:00" 
# lastmod = "2025-10-03T17:00:00+09:00"
title = "AWS費用監視ツール（前編：AWSマネジメントコンソール & AWS CLI）" 
archives = ["2026", "2026-03", "2026-03-22"]
+++

> この記事は[2021年版](https://qiita.com/t-taku/items/ef0e7edc79f89929d466)をベースに、2026年現在のベストプラクティスに合わせてリバイズしたものです。

## 1. はじめに
2021年に「AWS費用監視ツール」の記事を書いてから、もう5年が経ちました。当時はIAMユーザーのアクセスキーを使ってCLIをセットアップしていましたが、いまではAWSの公式ガイドラインで長期的なアクセスキー（Long-term credentials）の使用は非推奨になっています。

実際、筆者自身も日頃のAWS運用ではIAM Identity Center（旧AWS SSO）に移行しており、「そろそろ記事もアップデートしないとな…」と思いつつ、なかなか手を付けられていませんでした。

そこで今回、記事の内容を現在のベストプラクティスに沿った形で書き直すことにしました。

- 「毎日マネコンにログインしてCost Explorerで費用を確認するのは面倒…」
- 「通知ツールを自作したいけど、Lambdaの実装はハードルが高そう…」

こうした悩みをお持ちの方に向けて、マネコンでの確認方法からCLIでの取得方法まで、順を追って紹介していきます。

### 1.1 2021年版からの主な変更点
| 項目 | 2021年版 | 2026年版（本記事） |
|------|----------|----------|
| IAM認証 | IAMユーザー＋アクセスキー（長期認証情報） | IAM Identity Center（SSO＋一時認証情報） |
| AWS CLI バージョン | v1/v2（混在） | AWS CLI v2のみ |
| 認証情報の設定 | `aws configure` | `aws configure sso` |

### 1.2 記事の構成（前編・後編）
記事は2部構成になっています。

- 前編（本記事）：マネコンとAWS CLIでのコスト確認
- 後編：Lambdaを使ったデイリー通知の自動化

なお、IAM Identity Centerの許可セットとAWS CLI（SSO）のセットアップは、本編の流れを遮らないよう[付録A]、[付録B]にまとめました。CLIの例を試す前に、先にこちらを実施してください。参考リンクは[付録C]にあります。

## 2. 全体構成
[![前編ではマネコンとCLIでのコスト確認、後編ではLambdaによる自動通知を扱う構成図](https://imgur.com/NcMjnaT.png)](https://imgur.com/NcMjnaT.png)

全体の流れとしては、まずマネコンのCost Explorerで「どんなデータが見えるのか」を確認し、次にAWS CLIで同じデータをコマンドで取得できるようにします。CLIで取得できるようになれば、後編のLambdaによる自動通知への道がぐっと近くなります。

## 3. AWSマネジメントコンソールでのコスト確認
### 3.1 Cost Explorer
まずはマネコンから。Cost Management → Cost Explorerでコスト確認ができます。

[![img2](https://imgur.com/CqvjWfA.png)](https://imgur.com/CqvjWfA.png)

### 3.2 主な確認ポイント
- 期間指定：日次・月次・年次を選べます。デフォルトは当月です。
- グループ化条件：サービス別、リージョン別、タグ別などにまとめられます。
- フィルター：料金タイプ（クレジット・使用料・税金）で絞り込めます。

### 3.3 クレジット適用前後のコスト確認
2021年版でも触れましたが、2026年現在も確認方法は同じです。
- クレジット適用後（デフォルト）：フィルターなしで表示されるのがクレジット適用済みの料金です。
- クレジット適用前：「料金タイプ」フィルターで「クレジット」を除外すると確認できます。

クレジット適用前の金額を把握しておくと、「実際にはいくら使っているのか」が明確になるので、個人的にはこちらもチェックするようにしています。

## 4. AWS CLIでのコスト確認

さて、ここからが本題です。マネコンで確認していた情報を、CLIでも取得してみましょう。

ここからの手順は、以下の準備が完了していることを前提にしています。まだの方は先に実施してください。

- IAM Identity Center の有効化・ユーザー作成・許可セット（コスト閲覧用ポリシー）の割り当て → [付録A]
- AWS CLI v2 のインストール・環境変数の設定・`aws sso login` によるログイン → [付録B] 

また、[付録B] B.2のとおり `export AWS_PROFILE=billing-user` を設定するか、各コマンドに `--profile billing-user` を付与してください。

### 4.1 基本コマンド：get-cost-and-usage

> 補足：Cost Explorer API の料金について
> `aws ce get-cost-and-usage` などの Cost Explorer API は、1リクエストあたり $0.01 の料金が発生します。1日1回程度の呼び出しなら問題ありませんが、Lambdaなどで短い間隔に何度も呼び出すと、料金とスロットリングの両面で負荷が大きくなるので注意です。

それでは、3章でマネコン（GUI）から確認したのと同じ期間のデータを、CLIで取得してみましょう。

まず、期間をシェル変数に設定します。

```bash
START=2026-02-01
END=2026-03-01   # Cost Explorer は End 日を含まないため「翌月1日」にする
```

基本のコマンドは `aws ce get-cost-and-usage` です。

```bash
aws ce get-cost-and-usage \
  --time-period Start=${START},End=${END} \
  --granularity MONTHLY \
  --metrics "AmortizedCost"
```

> `AmortizedCost` とは？
> リザーブドインスタンスや Savings Plans の前払い費用を利用期間で按分（償却）した金額です。オンデマンド利用のみの場合は `UnblendedCost`（非ブレンドコスト）とほぼ同じ値になります。

JSON形式で、期間のAWS費用合計が返ってきます。

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

### 4.2 必要な値のみ抽出する

JSON丸ごとだと見づらいので、`--query`と`--output text`を組み合わせてコスト金額だけを取り出してみます。

```bash
aws ce get-cost-and-usage \
  --time-period Start=${START},End=${END} \
  --granularity MONTHLY \
  --metrics "AmortizedCost" \
  --query "ResultsByTime[].Total.AmortizedCost.Amount" \
  --output text
# 209.1272342834
```

### 4.3 クレジット適用前のコストを取得する

3章でマネコンから確認したように、クレジットを除いた使用料を取得してみましょう。`--filter`オプションを使います。まず、`filter.json`というファイルを作成します。

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

フィルターを適用してコマンドを実行します。`file://filter.json` はカレントディレクトリの相対パスです。上記 JSON を `filter.json` として保存したディレクトリに `cd` してから実行するか、`file:///絶対パス/filter.json` を指定してください。ファイルが無いと `No such file or directory: 'filter.json'` というエラーになります。

```bash
aws ce get-cost-and-usage \
  --time-period Start=${START},End=${END} \
  --granularity MONTHLY \
  --metrics "AmortizedCost" \
  --filter file://filter.json \
  --query "ResultsByTime[].Total.AmortizedCost.Amount" \
  --output text
# 440.2072342834
```

これで、クレジット適用前のAWS料金を取得できました！

### 4.4 【おまけ】サービス別コストをjqで整形する

ここからはおまけです。[`jq`](https://jqlang.github.io/jq/)はJSON を整形・加工できるコマンドラインツールで、AWS CLIの出力を加工するのに非常に便利です。未インストールの場合は`brew install jq`（macOS）や`sudo apt install jq`（Ubuntu）などで導入できます。

`jq`を使って、サービス別のコスト内訳を見やすく整形してみましょう。

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

## 5. まとめ
本記事では、2021年版の記事をベースに、2026年のAWSベストプラクティス（IAM Identity Center + 一時認証情報）に沿った形でコスト監視の基礎を紹介しました。

CLIでコストを取得できるようになれば、あとはこれをLambdaに載せて自動通知する仕組みを作るだけです。後編では、実際にLambdaを使ったデイリー通知の仕組みを構築していきますので、ぜひそちらもご覧ください。

## [付録A] IAM Identity Center と許可セット（閲覧専用）
コスト・請求の閲覧専用にIdentity Centerのユーザーを用意する手順です。

### A.1 アクセスキーを使わない方式へ更新する背景
2021年版では`aws_access_key_id`と`aws_secret_access_key`を使うIAMユーザーを作成していました。しかし、これらは長期的に有効な認証情報で、以下のようなリスクがあります。
- 誤ってGitHubなどにコミットすると、第三者に悪用されるおそれがある
- 定期的なローテーションが必要だが、実運用では忘れがち
- AWSの公式ベストプラクティスでも、Long-term credentialsの使用は非推奨

実際、アクセスキー漏洩の事故はよく耳にしますし、筆者も一時期はキーの管理に気を遣っていました。IAM Identity Centerに移行してからは、こうした心配から解放されて快適です。

### A.2 IAM Identity Centerの設定
IAM Identity Centerを使うと、一時的な認証情報（最長12時間）が自動的に発行・更新されるため、アクセスキーの管理が不要になります。セットアップも意外と簡単です。

#### A.2.1 IAM Identity Centerの有効化
1. AWSマネジメントコンソール → IAM Identity Centerを開く
2. 「有効にする」をクリック（初回のみ）
3. アイデンティティソースを設定します（例：今回は AWS 管理の Identity Center ディレクトリを使います）。

#### A.2.2 ユーザーとグループの作成
IAM Identity Center > ユーザー > ユーザーを追加
- ユーザー名例：`billing-user`
- メールアドレスを登録すると招待メールが届きます
- グループは、複数人に同じ権限を一括付与したい場合に使います。今回は個人利用のため作成せず、ユーザーを直接割り当てます。

#### A.2.3 許可セット（Permission Set）の作成
コスト確認に必要な最小権限を持つ許可セットを作成します。

- 概要: ここで作成するインラインポリシーは、AWS の請求情報およびコスト分析機能を閲覧専用（ReadOnly）で利用するための権限セットです。開発者や管理者がコスト状況を可視化することを目的としています。リソースの変更や課金設定の変更はできません。
- 対象機能: Billing（請求・明細）、Cost Explorer（コスト分析）、Budgets（予算確認）
- 操作: IAM Identity Center > 許可セット > 許可セットを作成   

 「カスタム許可セット」を選び、インラインポリシーに次の JSON を追記します。
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

> 補足：`aws-portal:ViewBilling` / `aws-portal:ViewUsage` はレガシーアクションです。AWS は新しい細粒度アクション（`billing:*`、`cur:*` 等）への移行を進めています。既存環境との互換性のため本記事では併記していますが、新規構築の場合は [AWS Billing のアクション一覧](https://docs.aws.amazon.com/service-authorization/latest/reference/list_awsbilling.html)を参照し、新アクションのみに絞ることも検討してください。

ポイントは、必要最小限の権限（最小権限の原則）を付与することです。`AdministratorAccess` を付与するのはアンチパターンですので避けましょう。ワイルドカード（`ce:Get*` など）は読み取り範囲が広いため、用途に応じて [Cost Explorer の API 一覧](https://docs.aws.amazon.com/aws-cost-management/latest/APIReference/API_Operations_AWS_Cost_Explorer_Service.html)を参照しながら、個別アクションへ絞ることも検討できます。

#### A.2.4 AWSアカウントへのアクセス割り当て
- IAM Identity Center > AWSアカウント > 対象アカウントを選択

  「ユーザーまたはグループを割り当て」で、作成したユーザー（またはグループ）と許可セットを紐付けます。

以上で、SSOユーザーの設定は完了です。ここまでできれば、あとはCLI側の設定だけです。

## [付録B] AWS CLI のセットアップ（v2 + SSO）
### B.1 AWS CLI v2 のインストール
macOS、Linux、Windowsそれぞれの公式インストーラーを使います。筆者はmacOS環境なので、Homebrewでサクッとインストールしました。
```bash
# macOS（Homebrew）
brew install awscli

# バージョン確認
aws --version
aws-cli/2.27.31 Python/3.13.3 Darwin/24.6.0 exe/x86_64
```

公式ドキュメント：[AWS CLI v2 インストールガイド](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

### B.2 環境変数
以降、`aws` の例では SSO プロファイル名を `billing-user` に統一します。

プロファイル定義は初回だけ B.3 の `aws configure sso --profile billing-user` で `~/.aws/config` に書き込みます。既に SSO プロファイルがある場合は B.3 は不要なので、そのまま B.4 へ進んでください。

先にシェル環境変数を設定しておくと、`--profile` の繰り返しやページャで止まる挙動を避けられます。

#### B.2.1 AWS_PROFILE
`--profile` を省略するための環境変数です。
```bash
export AWS_PROFILE=billing-user
```

設定後は `aws sts get-caller-identity` のように、引数にプロファイル名を付けなくても本記事で使う `billing-user` が使われます。

#### B.2.2 AWS_PAGER
AWS CLI v2は、出力が一定以上あると既定でページャ（多くの環境では`less`）に渡します。画面下に`(END)`と出て`q`で抜ける挙動になるのですが、これが地味に煩わしいんですよね。以下の設定で、ページャを無効にしてそのまま標準出力に流すことができます。
```bash
export AWS_PAGER=cat
```
※`AWS_PAGER=""` でも構いません(ページャは使われません)。

### B.3 SSOプロファイルの設定
> この節の対話設定は「初回のみ」です。`~/.aws/config`に、使いたいSSOプロファイル（本記事では`billing-user`）がまだ存在しないときだけ実施してください。

> すでに別マシンなどで`aws configure sso`を済ませている場合は、この設定はスキップしてB.4節の`aws sso login`から始めてください。

アクセスキーの代わりに`aws configure sso`を使います。`--profile`を付けておくとプロファイル名を先に決められるので便利です。SSO session nameはプロファイル名とは別のラベルにするのがおすすめです。
```bash
aws configure sso --profile billing-user
```

対話形式で次のように入力します。

```
SSO session name (Recommended): my-sso
SSO start URL [None]: https://<your-domain>.awsapps.com/start
SSO region [None]: ap-northeast-1
SSO registration scopes [sso:account:access]: （Enterでデフォルト）
```

> SSO session name は `~/.aws/config` 内の `[sso-session …]` のラベルです。プロファイル名と同じにする必要はありません。

ブラウザが自動的に開き、IAM Identity Center の AWS アクセスポータルで認証します。認証後、ターミナルに戻ると続きが表示されます。

```
Default client Region [ap-northeast-1]: （利用リージョン。Enterで既定でも可）
CLI default output format (json if not specified) [None]: json
```
#### B.3.1 `~/.aws/config` に追記される内容(サンプル)
`aws configure sso --profile billing-user` を完了すると、概ね次の2つのブロックが書かれます。

次の例は説明用です。`sso_account_id`、`sso_role_name`、`sso_start_url` などは、ご自身の環境の値に読み替えてください（`sso_account_id` は `<account_id>`、`sso_role_name` は許可セットに対応する `<permission set>` を想定）。

```ini
[profile billing-user]
sso_session = my-sso
sso_account_id = <account_id>
sso_role_name = <permission set>
region = ap-northeast-1
output = json

[sso-session my-sso]
sso_start_url = https://d-xxxxxxxxxx.awsapps.com/start/
sso_region = ap-northeast-1
sso_registration_scopes = sso:account:access
```

### B.4 SSOログインと接続確認
```bash
# SSOログイン（ブラウザが開く）
aws sso login
Attempting to automatically open the SSO authorization page in your default browser.
If the browser does not open or you wish to use a different device to authorize this request, open the following URL:

https://oidc.ap-northeast-1.amazonaws.com/authorize?response_type=code&client_id=<dummy>&redirect_uri=...&state=<uuid>&code_challenge_method=S256&scopes=sso%3Aaccount%3Aaccess&code_challenge=<hash>
Successfully logged into Start URL: https://d-xxxxxxxxxx.awsapps.com/start/
```
途中、ブラウザによるユーザ認証が出るのでログインします。
[![img3](https://imgur.com/rteWJ6m.png)](https://imgur.com/rteWJ6m.png)

```bash
# 接続確認（認証が通り、どのロールでいるかを表示）
aws sts get-caller-identity
```

```json
{
    "UserId": "AROAEXAMPLE00000000000:billing-user",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/AWSReservedSSO_example-permission-set_a1b2c3d4e5f67890/billing-user"
}
```

無事アクセスできました！

一時認証情報の有効期間は、許可セットのセッション持続時間に依存します（デフォルト1時間、最大12時間まで延長可能）。期限切れ後は再度`aws sso login`を実行してください。これで、本編の4章のCLI操作を試す準備が整いました。

## [付録C] 参考リンク
- [AWS CLI v2 インストールガイド](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [IAM Identity Center で AWS CLI を設定する](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html)
- [Security best practices in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [Best practices for the AWS Cost Explorer API](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-api-best-practices.html)
- [get-cost-and-usage — AWS CLI リファレンス](https://docs.aws.amazon.com/cli/latest/reference/ce/get-cost-and-usage.html)

*Amazon Web Services、およびその他のAWS商標は、米国およびその他の諸国におけるAmazon.com, Inc.またはその関連会社の商標です。*
