# AWS費用監視ツール（後編：Lambda活用）

> **この記事は [2021年版](https://qiita.com/t-taku/items/ef0e7edc79f89929d466) の後編を最新のAWSベストプラクティスに沿ってアップデートしたものです。**
> 前編（マネジメントコンソール & AWS CLI 編）は [こちら](./aws_costcheck_cli.md) をご参照ください。

## 1. はじめに

前編では、AWSマネジメントコンソールとAWS CLIでのコスト確認方法を紹介しました。後編では、**Python（boto3）による費用取得** と **AWS SAMを使ったLambdaへのデプロイ** を解説します。

**本記事の手順・コードは、実装リポジトリ [t-tkm/aws-cost-explore](https://github.com/t-tkm/aws-cost-explore) と整合するように書いています。** 最新のソースやテスト、`sam/` 配下のそのまま動く構成は GitHub を正としてください。

2021年版からの主な変更点は以下のとおりです。

| 項目 | 2021年版 | 2025年版（リポジトリ準拠） |
|------|----------|----------|
| Pythonバージョン | 3.8 | **3.12**（Lambda は `python3.12`。3.8 は2024年10月EOL） |
| Python仮想環境 | Miniconda（conda） | **venv + pip**（リポジトリ README と同じ） |
| Lambda Runtimeの設定 | python3.8 | **python3.12** |
| SAMデプロイ方法 | `sam package` + `sam deploy` | **`buildAnddeploy.sh`**（package + S3 + deploy）または **`sam deploy --guided`** |
| Teams通知方法 | Office 365コネクタ Webhook | **ワークフロー用 Webhook**（リポジトリは **Adaptive Card** 形式で POST） |
| Webhook URL の管理 | スクリプト変数またはSAMパラメータ | **SAM パラメータ**（`NoEcho`）＋ローカルは **環境変数**（`sample.env` 等） |

---

## 2. Python実行環境の準備

### 2.1 Python 3.8 EOLについて

2021年版では Python 3.8 + Miniconda を使用していました。しかし Python 3.8 は **2024年10月にEOL（End of Life）** を迎えており、AWS LambdaもPython 3.8ランタイムのサポートを終了しています。**Python 3.12以上**へ移行してください。

2025年11月時点で、AWS Lambdaは **Python 3.14** までサポートしています。安定性と実績のある **Python 3.12** の使用を推奨します。

### 2.2 ローカル環境の準備（venv + pip）

[リポジトリの README](https://github.com/t-tkm/aws-cost-explore/blob/main/README.md) と同じく、Python 標準の **`venv`** と **`pip`** で仮想環境を用意します（**GitHub 上の手順は uv を使っていません**）。

```bash
git clone https://github.com/t-tkm/aws-cost-explore.git
cd aws-cost-explore

# Python 3.12 系が入っていることを確認（Lambda と揃えるなら 3.12 推奨）
python3 --version

# 仮想環境の作成（README と同じくディレクトリ名は venv）
python3 -m venv venv

# アクティベート（macOS / Linux）
source venv/bin/activate

# アクティベート（Windows）
venv\Scripts\activate

# ルートの requirements.txt から依存を導入（boto3 / requests / pytest など）
pip install -r requirements.txt
```

Lambda 用の SAM ビルドは **`sam/app/requirements.txt`**（現状は `requests` のみ。boto3 はランタイム同梱）を SAM CLI が読みます。

---

## 3. Python（boto3）によるコスト取得

### 3.1 認証の考え方

2021年版のコードは `boto3.client('ce', region_name='us-east-1')` を使い、**環境変数のAWSプロファイル（アクセスキー）** に依存していました。

リポジトリでも Cost Explorer API は **us-east-1** のクライアントを使います。ローカル実行時は **IAM Identity Center（SSO）のプロファイル** や `~/.aws/credentials`、Lambda 実行時は **IAM 実行ロール** を boto3 が解決します。

### 3.2 ソースコードの置き場所

[aws-cost-explore](https://github.com/t-tkm/aws-cost-explore) では次のように役割が分かれています。

| パス | 役割 |
|------|------|
| [`src/cost_report.py`](https://github.com/t-tkm/aws-cost-explore/blob/main/src/cost_report.py) | ローカル実行向けエントリ（標準出力・任意で Teams） |
| [`sam/app/app.py`](https://github.com/t-tkm/aws-cost-explore/blob/main/sam/app/app.py) | Lambda 用（`lambda_handler` が `main()` を呼び出し） |

実装の要点（詳細は上記リンクのソースを参照してください）。

- **`CostExplorer` クラス**で `get_cost_and_usage` をラップし、クレジット除外フィルタやサービス別 `GroupBy` を扱う。
- **環境変数 `USE_TEAMS_POST`**（`yes` のときのみ投稿）と **`TEAMS_WEBHOOK_URL`**。Lambda では `template.yaml` の `Environment` から注入される。
- Teams 通知は **`requests`** で JSON を POST。ペイロードは **Adaptive Card**（`application/vnd.microsoft.card.adaptive`）形式。
- レポート見出しに **STS の `get_caller_identity` で取得した AWS アカウント ID** を付与する。
- Lambda ランタイムに同梱の **boto3** を利用し、追加パッケージは [`sam/app/requirements.txt`](https://github.com/t-tkm/aws-cost-explore/blob/main/sam/app/requirements.txt)（現状 `requests` のみ）で管理。

> **注**：リポジトリ内の `sam/app/app.py` 先頭コメントは `src/cost_report.py` の名残です。Lambda では **`sam/app/app.py`** がデプロイ対象です。

### 3.3 ローカル実行確認

ルートの [README.md](https://github.com/t-tkm/aws-cost-explore/blob/main/README.md) と同様に、`USE_TEAMS_POST` / `TEAMS_WEBHOOK_URL` を必要に応じて設定してから `src/cost_report.py` を実行します。

```bash
# SSOプロファイルでログイン（前編参照）
aws sso login --profile billing-user
export AWS_PROFILE=billing-user

# リポジトリルートで（venv を有効化済み想定）
python src/cost_report.py
```

Teams へ投稿する場合は `export USE_TEAMS_POST=yes` と `TEAMS_WEBHOOK_URL` を設定します（未設定で `yes` のときは `ValueError`）。具体例はリポジトリの `sample.env` を参照してください。

---

## 4. Microsoft Teams通知の移行（重要）

### 4.1 Office 365コネクタの廃止

2021年版では、Microsoft TeamsのIncoming WebhookとしてOffice 365コネクタのURLを使用していました。しかし、Microsoftは **2024年8月15日をもってOffice 365コネクタの新規作成を停止**し、**2024年10月1日にすべてのコネクタを廃止**しています。

既存のWebhook URLはすでに機能しないため、**Power Automate ワークフロー**への移行が必須です。

### 4.2 ワークフロー（Webhook）の作成

Office 365 コネクタの代わりに、**Power Automate** などで **HTTP POST を受け取り Teams に投稿するワークフロー**を用意し、その URL を `TEAMS_WEBHOOK_URL` に設定します。

**手順の例（Power Automate）：**

1. [Power Automate](https://make.powerautomate.com/) にサインイン
2. 「作成」→「自動化したクラウドフロー」を選択
3. トリガーとして「**Teams webhook 要求が受信されたとき**」（または「HTTP要求を受け取ったとき」）を選択
4. アクションに「**チャットまたはチャネルにメッセージを投稿する**」を追加
   - 投稿者：フローボット
   - 投稿先：チャネル（対象のチームとチャネルを選択）
   - メッセージ：トリガーが渡す本文や Adaptive Card の内容を参照して整形
5. 保存後に発行される **HTTP POST の URL** をコピーする

この URL を SAM パラメータ `TeamsWebhookUrl` やローカルの環境変数に設定します。

> **リポジトリ実装との対応**：[aws-cost-explore](https://github.com/t-tkm/aws-cost-explore) の `post_to_teams` は **`attachments` 配下に Adaptive Card を載せた JSON** を送ります（単純な `{ "title", "text" }` ではありません）。ワークフロー側は **受け取った JSON をそのまま Teams に渡せる構成**にするか、必要に応じて `parse JSON` して Adaptive Card を解釈するよう調整してください。実際のスキーマは [`sam/app/app.py` の `post_to_teams`](https://github.com/t-tkm/aws-cost-explore/blob/main/sam/app/app.py) を参照してください。

> **注意**：Power Automate のフローボットはプライベートチャネルへのメッセージ投稿が制限されています。パブリックチャネルを使用するか、ユーザーとして投稿する設定を選んでください。

---

## 5. AWS SAMによるLambdaデプロイ

SAM 用のテンプレートと関数コードはリポジトリの **`sam/`** ディレクトリにあります（ルートの `src/` はローカル実行用）。

### 5.1 前提とディレクトリ構成

SAM CLI の導入は [AWS 公式手順（Install the SAM CLI）](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html) に従ってください（`pip install aws-sam-cli` やパッケージマネージャーなど）。リポジトリ自体は **uv 非依存**です。

[aws-cost-explore](https://github.com/t-tkm/aws-cost-explore) をクローン済みで、作業ディレクトリを **`sam/`** に移します。

```
aws-cost-explore/
├── src/                    ← ローカル用 cost_report.py
├── sam/
│   ├── template.yaml
│   ├── app/
│   │   ├── app.py          ← Lambda ハンドラ
│   │   └── requirements.txt   ← 現状は requests のみ
│   ├── events/
│   │   └── event.json
│   ├── buildAnddeploy.sh   ← package + S3 + deploy の例
│   └── tests/
└── requirements.txt        ← ローカル開発用（pytest 等）
```

### 5.2 SAMテンプレート（`sam/template.yaml`）

リポジトリの定義を整理した抜粋です（実ファイルと差分がある場合は [GitHub 上の template.yaml](https://github.com/t-tkm/aws-cost-explore/blob/main/sam/template.yaml) を優先してください）。

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Notify AWS billing to Teams using Lambda

Globals:
  Function:
    Timeout: 60

Parameters:
  TeamsWebhookUrl:
    Type: String
    Description: "Webhook URL for Teams notifications"
    NoEcho: true

  UseTeamsPost:
    Type: String
    Description: "Flag to enable or disable Teams posting (yes/no)"
    Default: "false"

Resources:
  BillingIamRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: "sts:AssumeRole"
      Policies:
        - PolicyName: "BillingNotificationToTeamsLambdaPolicy"
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource: "arn:aws:logs:*:*:*"
              - Effect: Allow
                Action:
                  - ce:GetCostAndUsage
                Resource: "*"
              - Effect: Allow
                Action:
                  - sns:Publish
                Resource: "arn:aws:sns:*:*:*"

  BillingNotificationFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: app/
      Handler: app.lambda_handler
      Runtime: python3.12
      Architectures:
        - x86_64
      Role: !GetAtt BillingIamRole.Arn
      Environment:
        Variables:
          USE_TEAMS_POST: !Ref UseTeamsPost
          TEAMS_WEBHOOK_URL: !Ref TeamsWebhookUrl
      Events:
        NotifyTeams:
          Type: Schedule
          Properties:
            Schedule: cron(0 0 * * ? *)

Outputs:
  BillingNotificationLambdaArn:
    Description: "ARN of the Billing Notification Lambda Function"
    Value: !GetAtt BillingNotificationFunction.Arn
  BillingNotificationLambdaRoleArn:
    Description: "ARN of the IAM Role for the Billing Notification Lambda Function"
    Value: !GetAtt BillingIamRole.Arn
```

**2021年版からの主な変更点（テンプレート観点）：**

- `Runtime: python3.12`、`Architectures: x86_64` を明示
- Webhook URL と Teams 投稿有効化を **パラメータ**（`TeamsWebhookUrl` / `UseTeamsPost`）と **環境変数**で渡す
- スケジュールは **`cron(0 0 * * ? *)`**（UTC 0:00 = 同日 9:00 JST）
- IAM に **`sns:Publish`** が含まれる（現行コードパスでは未使用でもテンプレート上は残っている）

### 5.3 ビルドとデプロイ

**方法 A：`sam deploy --guided`（対話形式）**

```bash
cd aws-cost-explore/sam
sam build
sam deploy --guided
```

プロンプトで `TeamsWebhookUrl`（ワークフローの URL）と `UseTeamsPost` を渡します。アプリは環境変数 `USE_TEAMS_POST` を **小文字で `yes` と一致するときだけ** Teams に POST します（それ以外・未設定は投稿しません）。テンプレートのデフォルトは `"false"` です。

**方法 B：リポジトリの `buildAnddeploy.sh`**

リポジトリルートに **`.env`** を用意し（`buildAnddeploy.sh` は `../.env` を読み込みます）、次を設定したうえで `sam` ディレクトリから実行します。

- `S3_BUCKET` … アーティファクト配置用バケット
- `TEAMS_WEBHOOK_URL` / `USE_TEAMS_POST` … `sam deploy` の `--parameter-overrides` に渡る値

```bash
cd aws-cost-explore/sam
./buildAnddeploy.sh
```

スクリプトは **`sam package`（S3 経由）→ `sam deploy`** の流れで、スタック名は **`NotifyBillingToTeams`** です。バケットやスタック名を変えたい場合はスクリプトを編集してください。

### 5.4 ローカルテスト（SAM Local）

```bash
cd aws-cost-explore/sam
sam build
# events/event.json は空の JSON（{}）でよい例がリポジトリに含まれます
sam local invoke BillingNotificationFunction --event events/event.json
```

### 5.5 削除

```bash
cd aws-cost-explore/sam
sam delete --stack-name NotifyBillingToTeams
```

スタック名を `sam deploy --guided` で変えた場合は、その名前に合わせてください。

---

## 6. まとめ

本記事では、2021年版後編の内容を、実装リポジトリ [aws-cost-explore](https://github.com/t-tkm/aws-cost-explore) と揃えたうえで整理しました。

**後編のポイント：**

1. **Python 3.8 は EOL** → Lambda は **python3.12**（リポジトリの `template.yaml` 準拠）
2. **ローカル開発は venv + pip** → リポジトリ README どおり `python -m venv venv` と `pip install -r requirements.txt`（ルート）。SAM の依存は **`sam/app/requirements.txt`**
3. **エントリが二系統** → 手元実行は **`src/cost_report.py`**、クラウドは **`sam/app/app.py`**（`lambda_handler` → `main()`）
4. **Teams は Adaptive Card + `requests`** → ワークフロー側の受け取り形式と整合を取る（§4 参照）
5. **Office 365 コネクタは廃止済み** → ワークフロー用 Webhook へ移行する
6. **デプロイ** → **`sam build` + `sam deploy --guided`** またはリポジトリの **`buildAnddeploy.sh`**（`sam package` + S3）

---

## 参考リンク

- [t-tkm/aws-cost-explore（本記事の実装）](https://github.com/t-tkm/aws-cost-explore)
- [Python 3.12 runtime now available in AWS Lambda](https://aws.amazon.com/blogs/compute/python-3-12-runtime-now-available-in-aws-lambda/)
- [Install the AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html)
- [Retirement of Office 365 connectors within Microsoft Teams](https://devblogs.microsoft.com/microsoft365dev/retirement-of-office-365-connectors-within-microsoft-teams/)
- [Teams webhook を使ったワークフローへの移行](https://techcommunity.microsoft.com/discussions/teamsdeveloper/simple-workflow-to-replace-teams-incoming-webhooks/4225270)
- [AWS SAM deploy コマンドリファレンス](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-cli-command-reference-sam-deploy.html)

---

*Amazon Web Services、およびその他のAWS商標は、米国およびその他の諸国におけるAmazon.com, Inc.またはその関連会社の商標です。*
*Microsoft 365、Microsoft Teamsは、米国Microsoft Corporationおよびその関連会社の米国およびその他の国における登録商標または商標です。*
