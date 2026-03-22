# AWS費用監視ツール（前編：AWSマネジメントコンソール & AWS CLI）

> **この記事は [2021年版](https://qiita.com/t-taku/items/ef0e7edc79f89929d466) を最新のAWSベストプラクティスに沿ってアップデートしたものです。**

## 1. はじめに

「AWS費用のデイリー通知、やってますか？」

AWSを使い始めると、気づいたら思わぬ費用が発生していた——という経験をしたことがある方も多いと思います。2021年当時の記事では、IAMユーザーのアクセスキーを使ったAWS CLIのセットアップ方法を紹介しました。しかし**2025年現在、長期的なアクセスキー（Long-term credentials）の使用はAWSの公式セキュリティガイドラインで非推奨**となっています。

本記事では、以下の2点を意識してアップデートします。

- **セキュリティ面**：アクセスキーを使わず、**IAM Identity Center（旧 AWS SSO）** を使った認証へ移行する
- **機能面**：2021年以降に追加・強化されたコスト管理の動向にも軽く触れる（異常検知まわりの公式リンクは**付録**にまとめた）

> **記事の構成（全2部）**
> - **前編（本記事）**：AWSマネジメントコンソール & AWS CLI によるコスト確認
> - **後編**：Lambda活用によるコスト通知の自動化

---

## 2. 2021年からの主な変更点

元記事（2021年）との差分を最初に整理しておきます。

| 項目 | 2021年版 | 2025年版 |
|------|----------|----------|
| IAM認証 | IAMユーザー + アクセスキー | **IAM Identity Center（SSO）+ 一時認証情報** |
| AWS CLI バージョン | v1 / v2（混在） | **AWS CLI v2 のみ** |
| 認証情報の設定 | `aws configure` | **`aws configure sso`** |

---

## 3. IAM設定（2025年ベストプラクティス）

### 3.1 なぜアクセスキーを使わないのか

2021年版では `aws_access_key_id` と `aws_secret_access_key` を使うIAMユーザーを作成していました。しかし、これらは**長期的に有効な認証情報**であり、以下のリスクがあります。

- 誤ってGitHubなどにコミットすると、第三者に悪用される
- 定期的なローテーションが必要だが、実運用では忘れがち
- AWSの公式Best PracticesでもLong-term credentialsの使用は**非推奨**

### 3.2 IAM Identity Centerの設定

**IAM Identity Center** を使うと、一時的な認証情報（最長12時間）が自動的に発行・更新されるため、アクセスキーの管理が不要になります。

#### Step 1: IAM Identity Centerの有効化

1. AWSマネジメントコンソール → **IAM Identity Center** を開く
2. 「有効にする」をクリック（初回のみ）
3. **アイデンティティソース** を設定（初期設定は Identity Center ディレクトリ）

#### Step 2: ユーザーとグループの作成

```
IAM Identity Center > ユーザー > ユーザーを追加
```

- ユーザー名例：`billing-user`
- メールアドレスを登録すると招待メールが届きます

**グループ**（複数人に同じ権限を付けたい場合）：

```
IAM Identity Center > グループ > グループを作成
```

- 表示名を入力してグループを作成し、グループ詳細の **「ユーザーを追加」** でメンバーを登録する
- 個人だけで使う場合はグループを作らず、ユーザーを直接割り当ててもよい

#### Step 3: 許可セット（Permission Set）の作成

コスト確認に必要な最小権限を持つ許可セットを作成します。

```
IAM Identity Center > 許可セット > 許可セットを作成
> 「カスタム許可セット」を選択
> インラインポリシーに以下を追記
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ce:GetCostAndUsage",
        "ce:GetCostForecast",
        "ce:GetAnomalies",
        "budgets:ViewBudget",
        "billing:GetBillingData"
      ],
      "Resource": "*"
    }
  ]
}
```

> **ポイント**：必要最小限の権限（最小権限の原則）を付与します。`AdministratorAccess` を付与するのはアンチパターンです。

#### Step 4: AWSアカウントへのアクセス割り当て

```
IAM Identity Center > AWSアカウント > 対象アカウントを選択
> 「ユーザーまたはグループを割り当て」
> 作成したユーザー（またはグループ）+ 許可セットを紐付ける
```

---

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

### 4.2 本記事の前提：プロファイルとページャ

以降、`aws` の例では SSO プロファイル名を **`billing-user`** に統一する。プロファイル本体は **次節**で `aws configure sso --profile billing-user` により作成するが、**先にシェル環境を整えておく**と、`--profile` の繰り返しやページャで止まる挙動を避けやすい。

**環境変数で `--profile` を省略**

毎回 `--profile billing-user` を付ける代わりに、`~/.zshrc` などに次を書いておくとよい。

```bash
export AWS_PROFILE=billing-user
```

設定後は `aws sts get-caller-identity` のように、**引数にプロファイル名を付けなくても** `billing-user` が使われる（SSO ログイン済みであることが前提）。

**ページャを無効にする（`q` で終了が面倒なとき）**

AWS CLI v2 は、出力が一定以上あると **既定でページャ**（多くの環境では `less`）に渡す。画面下に **`(END)`** と出て **`q` で抜ける**挙動はこれが原因で、Oh My Zsh 特有の問題ではない。

同じくシェル設定に例えば次を足す。

```bash
export AWS_PAGER=cat
```

`cat` は「そのまま標準出力に流す」指定です。空文字 **`AWS_PAGER=""`** でもページャは使われません。チームや端末ごとに **`~/.aws/config`** の `[default]` などへ **`cli_pager =`**（空）と書く方法もあります。

> **注意**：ページャを切ると **長い JSON が一気にターミナルに流れる**ため、スクロールで見づらいときは、そのコマンドだけ `--no-cli-pager` を付けるか、一時的にページャ有効のままにするのも手です。

> **補足**：**4.3** の `aws configure sso --profile billing-user` を除き、以降の `aws` 例では **`--profile` を付けない**（**4.2** の `AWS_PROFILE=billing-user` が効いている前提）。別プロファイルを使うときだけ `--profile` を付ければよい。

### 4.3 SSOプロファイルの設定

アクセスキーの代わりに `aws configure sso` を使います。**プロファイル名を先に決められる**ので、次のように **`--profile`** を付ける方法を推奨します。以降の例では **CLI プロファイル名を `billing-user`** に統一する。**SSO session name** は **プロファイル名と別のラベル**にするのがよい（下記ではウィザード例どおり **`my-sso`**。ポータル単位の略称など、読み替えてよい）。

```bash
aws configure sso --profile billing-user
```

対話形式で以下を入力します（**プロファイル名の質問は出ません**。`--profile` で既に `billing-user` が決まっているためです）。

```
SSO session name (Recommended): my-sso
SSO start URL [None]: https://<your-domain>.awsapps.com/start
SSO region [None]: ap-northeast-1
SSO registration scopes [sso:account:access]: （Enterでデフォルト）
```

> **SSO session name** は `~/.aws/config` 内の `[sso-session …]` のラベルです。プロファイル名と同じにする必要はありません。命名の考え方は **付録「SSO session 名の付け方」** を参照してください。

ブラウザが自動的に開き、IAM Identity CenterのAWSアクセスポータルで認証します。認証後、ターミナルに戻ると続きが表示されます。

```
Default client Region [ap-northeast-1]: （利用リージョン。Enterで既定でも可）
CLI default output format (json if not specified) [None]: json
```
#### `~/.aws/config` に追記される内容（構造の意味）

`aws configure sso --profile billing-user` を完了すると、概ね次の **2 ブロック** が書かれます。次の例は **構造の説明用**であり、`sso_account_id` や `sso_start_url` の値はご自身の環境のものに読み替えてください。

**本記事の推奨**：**`[profile billing-user]`**（用途＝コスト閲覧用 CLI）と **`[sso-session my-sso]`**（単位＝このアクセスポータルへのブラウザログイン）を **名前を分ける**。ウィザードで SSO session name に **`my-sso`** を入れた場合の対応関係は次のとおり。

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

> **補足**：一時認証情報はデフォルトで8時間有効です。期限切れ後は再度 `aws sso login` を実行してください。

> **許可セットがコスト確認だけの場合**：Step 3 の許可セットに **S3 権限を付けていない**なら、`aws s3 ls` は **`AccessDenied` になるのが正常**です。SSO ログイン成功と `sts get-caller-identity` が返れば、CLI からの認証は問題ありません。接続確認は **`get-caller-identity`** か、後述の **`ce get-cost-and-usage`** など、付与した API に合わせてください。

---

## 5. AWSマネジメントコンソールでのコスト確認

### 5.1 Cost Explorer

マネジメントコンソール → **Cost Management** → **Cost Explorer** でコスト確認ができます。

**主な確認ポイント：**

- **期間指定**：日次・月次・年次を選択可能。デフォルトは当月
- **グループ化条件**：サービス別、リージョン別、タグ別など
- **フィルター**：料金タイプ（クレジット・使用料・税金）でフィルタリング可能

#### クレジット適用前後のコスト確認

2021年版でも触れていましたが、2025年現在も同様の確認方法です。

- **クレジット適用後（デフォルト）**：フィルターなしで表示されるのがクレジット適用済みの料金
- **クレジット適用前**：「料金タイプ」フィルターで「クレジット」を除外することで確認可能


> **2025年版の追加補足**
**AWS Budgets** は、設定した予算に対してしきい値を超えそうなときに メール等で通知する機能です。マネコンではCost Management → Budgetsから予算を作成する。過去の内訳は Cost Explorer、将来の「はみ出し予告」に近い使い方ができる。
**AWS Cost Anomaly Detection** は、コストの異常な増加を検知してアラートするサービス（Cost Management 内）。本文では扱わず、概要・導線・参考リンクは付録「コスト異常検知」にまとめてある。

---

## 6. AWS CLIでのコスト確認

**前提**：**4.2** のとおり **`export AWS_PROFILE=billing-user`** を済ませてあるものとする（付いていない場合は各コマンドに `--profile billing-user` を足す）。

### 6.1 基本コマンド：get-cost-and-usage

3章のマネコン（GUI）経由で取得した期間のデータを、AWS CLIで取得してみます。

まず、期間をシェル変数に設定します。

```bash
# 例：2026年3月執筆時点では「確定しやすい先月」を指定（読み替えてよい）
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

> **2025年版の追加補足**
>`aws ce get-cost-and-usage`などのCost Explorer APIは、**1リクエストあたり $0.01 の料金**が発生します。1日1回程度の呼び出しなら問題ありませんが、Lambda 等で短い間隔で何度も叩くと料金とスロットリングの両方で痛くなります。

### 6.5 【おまけ】サービス別コストをjqで整形する

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

---

## 7. まとめと次のステップ

本記事では、2025年のAWSベストプラクティスに沿った形でコスト監視の基礎を解説しました。

**前編のポイント：**

1. **IAMユーザーのアクセスキーは非推奨** → IAM Identity Center（SSO）を使った一時認証情報に移行する
2. **AWS CLI v2** の `aws configure sso --profile billing-user` などでセキュアなプロファイルを設定する
3. **Cost Explorer** でコストの詳細を確認する（クレジット前後の使い分けも重要）

後編では、本記事で確認したCLIコマンドをLambdaに実装し、Microsoft Teamsへの定期通知を自動化する方法を解説します。

## 参考リンク

- [AWS CLI v2 インストールガイド](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [IAM Identity Center で AWS CLI を設定する](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html)
- [Security best practices in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [Best practices for the AWS Cost Explorer API](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-api-best-practices.html)
- [get-cost-and-usage — AWS CLI リファレンス](https://docs.aws.amazon.com/cli/latest/reference/ce/get-cost-and-usage.html)

*Amazon Web Services、およびその他のAWS商標は、米国およびその他の諸国におけるAmazon.com, Inc.またはその関連会社の商標です。*
