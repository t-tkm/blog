+++
Categories = ["AWS"]
Tags = ["AWS", "Lambda", "SAM", "Cost", "Teams", "FinOps"]
date = "2026-03-28T00:00:00+09:00"
title = "AWS費用監視ツール（後編：Lambda活用）"
archives = ["2026", "2026-03", "2026-03-28"]
+++

# はじめに

> 前編（マネジメントコンソール & AWS CLI 編）は [こちら](./aws_costcheck_cli.md) をご参照ください。

前編では、AWSマネジメントコンソールとAWS CLIでのコスト確認方法を紹介しました。後編では、
Python（boto3）による費用取得とAWS SAMを使ったLambdaへのデプロイを解説します。

本記事の手順・コードは、以下の実装リポジトリと整合するように書いています。最新のソースや
テスト、`sam/`配下のそのまま動く構成はGitHubを正としてください。

<a href="https://github.com/t-tkm/aws-cost-explore-lambda" target="_blank" rel="noopener">
    <img src="https://gh-card.dev/repos/t-tkm/aws-cost-explore-lambda.svg">
</a>

2021年版からの主な変更点は以下のとおりです。

| 項目 | 2021年版 | 2026年版（リポジトリ準拠） |
|------|----------|----------|
| Pythonバージョン | 3.8 | 3.12（Lambda は `python3.12`。3.8 は2024年10月EOL） |
| Python仮想環境 | Miniconda（conda） | uv（`uv sync` で仮想環境と依存管理を一括） |
| Lambda Architecture | x86_64 | arm64（Graviton2、約20%コスト削減） |
| SAMデプロイ方法 | `sam package` + `sam deploy` | `sam build` + `sam deploy --parameter-overrides` |
| Teams通知方法 | Office 365コネクタ Webhook | ワークフロー用 Webhook（Adaptive Card形式 + textフォールバック） |
| Webhook URL の管理 | スクリプト変数またはSAMパラメータ | ローカルは環境変数、LambdaはAWS Secrets Manager（`TEAMS_SECRET_ARN`） |
| エラーハンドリング | なし | DLQ（SQS）＋ CloudWatch Alarm |

---

# Python実行環境の準備

## Python 3.8 EOLについて

2021年版ではPython 3.8 + Minicondaを使用していました。しかしPython 3.8は2024年10月にEOL（End of Life）を迎えており、AWS LambdaもPython 3.8ランタイムのサポートを終了しています。
リポジトリではPython 3.12を採用しています。

## ローカル環境の準備（uv）

リポジトリでは、Pythonのパッケージマネージャとして[uv](https://docs.astral.sh/uv/)を採用しています。
`uv sync`一発で仮想環境（`.venv`）の作成と依存パッケージのインストールが完了するので、
従来の`venv` + `pip`よりもセットアップが簡潔です。

```bash
git clone https://github.com/t-tkm/aws-cost-explore-lambda.git
cd aws-cost-explore-lambda

# uv がインストール済みであることを確認
uv --version

# 仮想環境の作成 & 依存パッケージインストール
uv sync
```

`uv sync`を実行すると、`pyproject.toml`と`uv.lock`に基づいて`boto3`、`requests`、`pytest`などが
インストールされます。

---

# Python（boto3）によるコスト取得

## 認証の考え方

Cost Explorer APIはus-east-1のクライアントを使います。
ローカル実行時はIAM Identity Center（SSO）のプロファイルを`AWS_PROFILE`で指定し、
Lambda実行時はIAMロールをboto3が自動解決します。

## ソースコードの構成

リポジトリのディレクトリ構成は以下のとおりです。

```
aws-cost-explore-lambda/
├── sam/
│   ├── template.yaml          ← SAMテンプレート
│   ├── app/
│   │   ├── app.py             ← Lambda ハンドラ & ローカル実行兼用
│   │   └── requirements.txt   ← Lambda用（requests のみ）
│   └── events/
│       └── event.json         ← テスト用イベント
├── tests/                     ← テストスイート
├── img/                       ← ドキュメント用画像
├── pyproject.toml             ← プロジェクト設定（uv用）
├── uv.lock                    ← 依存ロックファイル
└── README.md
```

エントリポイントは`sam/app/app.py`の一本です。ローカルでもLambdaでも同じファイルを使い、
`if __name__ == "__main__":`でローカル実行、`lambda_handler`でLambda実行を分けています。

実装の要点は以下のとおりです。

- `CostExplorer`クラスで`get_cost_and_usage`をラップし、クレジット除外フィルタやサービス別`GroupBy`を扱う
- `get_config()`関数で環境変数を解決。Teams通知が有効な場合、Webhook URLは`TEAMS_WEBHOOK_URL`（ローカル向け）または`TEAMS_SECRET_ARN`（Lambda向け、Secrets Managerから取得）のいずれかから取得する
- Teams通知は`requests`でJSON POST。Adaptive Card形式を試し、失敗時は`{"text": ...}`にフォールバックする戦略
- `TEAMS_WEBHOOK_FORMAT=text`を設定すると、Adaptiveを試さずtextのみ送信
- レポート見出しにSTSの`get_caller_identity`で取得したAWSアカウントIDを付与
- Lambda用の追加パッケージは`sam/app/requirements.txt`（現状`requests`のみ。boto3はランタイム同梱）

## ローカル実行確認

認証・環境変数の設定後、`uv run`で実行します。

```bash
# SSO でログイン
aws sso login --profile <your-sso-profile>
```

続けて、環境変数を設定します。READMEに記載のコピペ用テンプレートを参考にしてください。

```bash
# --- AWS（必須）---
export AWS_PROFILE=my-company-sso
export AWS_DEFAULT_REGION=ap-northeast-1
export AWS_PAGER=

# --- Teams（使うときだけ # を外して値を埋める）---
# export USE_TEAMS_POST=yes
# export TEAMS_WEBHOOK_URL="https://<your-domain>/workflows/<your-webhook-url>"
```

実行は以下のコマンドです。

```bash
uv run python sam/app/app.py
```

`uv run`は子プロセスとして実行されるため、環境変数は必ず`export`しておく必要があります
（`export`なしの代入だと`uv run`の子プロセスには渡りません）。

実行すると、以下のようなレポートが標準出力に表示されます。`USE_TEAMS_POST=yes`なら
同一内容がTeamsにも投稿されます。

```
❯ uv run python sam/app/app.py
INFO:__main__:AWS Account ID: 123456789012
INFO:__main__:Calculated total cost from Groups: 0.00 USD
------------------------------------------------------
AWSアカウント 123456789012
03/01～03/27のクレジット適用後費用は、0.00 USD です。
サービスごとの費用データはありません。
------------------------------------------------------

INFO:__main__:Calculated total cost from Groups: 12.34 USD
------------------------------------------------------
AWSアカウント 123456789012
03/01～03/27のクレジット適用前費用は、12.34 USD です。
- Amazon EC2: 5.00 USD
- Amazon Simple Storage Service: 3.00 USD
- AWS Lambda: 2.00 USD
- Amazon CloudWatch: 1.50 USD
- その他: 0.84 USD
------------------------------------------------------
```

---

# Microsoft Teams通知の移行

## Office 365コネクタの廃止

2021年版では、Microsoft TeamsのIncoming WebhookとしてOffice 365コネクタのURLを使用していました。
しかし、Microsoftは2024年8月15日をもってOffice 365コネクタの新規作成を停止し、
2024年10月1日にすべてのコネクタを廃止しています。

既存のWebhook URLはすでに機能しないため、Power Automateワークフローへの移行が必須です。

## ワークフロー（Webhook）の作成

Office 365コネクタの代わりに、Power AutomateなどでHTTP POSTを受け取りTeamsに投稿する
ワークフローを用意し、そのURLを`TEAMS_WEBHOOK_URL`（ローカル）や`TEAMS_SECRET_ARN`
（Lambda / Secrets Manager）に設定します。

手順の例（Power Automate）：

1. [Power Automate](https://make.powerautomate.com/) にサインイン
2. 「作成」→「自動化したクラウドフロー」を選択
3. トリガーとして「Teams webhook 要求が受信されたとき」を選択
4. アクションに「チャットまたはチャネルにメッセージを投稿する」を追加
5. 保存後に発行されるHTTP POST URLをコピーする

リポジトリの`post_to_teams`はAdaptive Card形式のJSONを送ります。
ワークフロー側は受け取ったJSONをそのままTeamsに渡せる構成にするか、`parse JSON`で
Adaptive Cardを解釈するよう調整してください。なお、Adaptiveでうまく表示されない場合は
`TEAMS_WEBHOOK_FORMAT=text`を設定すると`{"text": ...}`形式のみで送信します。

> Power Automateのフローボットはプライベートチャネルへのメッセージ投稿が制限されています。
> パブリックチャネルを使用するか、ユーザーとして投稿する設定を選んでください。

---

# AWS SAMによるLambdaデプロイ

SAM用のテンプレートと関数コードはリポジトリの`sam/`ディレクトリにあります。

## 前提

- [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html)がインストール済みであること
- [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) と IAM Identity Center（SSO）による認証が使えること

## SAMテンプレート（`sam/template.yaml`）の概要

リポジトリの`template.yaml`は、2021年版からかなり強化されています。実ファイルと差分がある場合は
[GitHub上のtemplate.yaml](https://github.com/t-tkm/aws-cost-explore-lambda/blob/main/sam/template.yaml)を
優先してください。

主なリソースと設計ポイントをまとめます。

Parameters:
| パラメータ | 説明 |
|-----------|------|
| `UseTeamsPost` | Teams通知の有効/無効（`yes` / `no`、デフォルト`no`） |
| `TeamsWebhookSecretArn` | Webhook URLを格納したSecrets ManagerシークレットのARN。`UseTeamsPost=yes`の場合は必須 |

Lambda関数（`BillingNotificationFunction`）:
- `Runtime: python3.12`、`Architectures: arm64`（Graviton2で約20%コスト削減）
- `ReservedConcurrentExecutions: 1`（コスト通知は同時1実行で十分）
- `Timeout: 120`秒（外部API呼び出しのバッファ）
- スケジュール: `cron(0 0 * * ? *)`（UTC 0:00 = JST 9:00に毎日実行）
- 環境変数として`USE_TEAMS_POST`と`TEAMS_SECRET_ARN`を注入（Conditionで制御）

IAMポリシー:
テンプレートではインラインの`Policies`で以下を許可しています。

- `AWSLambdaBasicExecutionRole`（CloudWatch Logs）
- `ce:GetCostAndUsage`（Cost Explorer読み取り）
- `sts:GetCallerIdentity`（アカウントID取得）
- `secretsmanager:GetSecretValue`（Teams有効時のみ、Conditionで制御）

エラーハンドリング:
- Dead Letter Queue（SQS）: Lambda失敗時にイベントを14日間保持する`BillingNotificationDLQ`
- CloudWatch Alarm: Lambdaエラー数が1以上になると発火する`LambdaErrorAlarm`（評価期間1日）

## Secrets Managerへの登録

Teams通知を使う場合は、デプロイ前にWebhook URLをSecrets Managerに登録しておきます。

```bash
# 1. シークレットを作成
aws secretsmanager create-secret \
  --name teams-webhook-url \
  --secret-string "https://<your-domain>/workflows/<your-webhook-url>"

# 2. ARN を確認
aws secretsmanager describe-secret \
  --secret-id teams-webhook-url \
  --query ARN --output text
# 出力例:
# arn:aws:secretsmanager:ap-northeast-1:123456789012:secret:teams-webhook-url-xxxxxx
```

シークレットの値がJSON（例: `{"url":"https://..."}`）でも、コード側でURLを取り出す仕組みに
なっているため、平文・JSON どちらでも動作します。

## ビルドとデプロイ

```bash
cd aws-cost-explore-lambda/sam
sam build
```

Teams通知を使う場合:

```bash
sam deploy \
  --parameter-overrides \
    UseTeamsPost=yes \
    TeamsWebhookSecretArn=arn:aws:secretsmanager:<region>:<account-id>:secret:teams-webhook-url-xxxxxx
```

Teams通知を使わない場合:

```bash
sam deploy --parameter-overrides UseTeamsPost=no
```

初回デプロイ時は`sam deploy --guided`で対話形式にすると、スタック名やリージョンなどを
確認しながら設定できるので便利です。

## ローカルテスト（SAM Local）

```bash
cd aws-cost-explore-lambda/sam
sam build
sam local invoke BillingNotificationFunction --event events/event.json
```

## 削除

```bash
sam delete --stack-name <your-stack-name>
```

---

# まとめ

本記事では、2021年版後編の内容を、実装リポジトリ
[aws-cost-explore-lambda](https://github.com/t-tkm/aws-cost-explore-lambda)と揃えたうえで
整理しました。

後編のポイント:
- Python 3.8 → 3.12: Lambda ランタイムもリポジトリの`template.yaml`準拠
- パッケージ管理は uv: `uv sync`で仮想環境と依存を一括セットアップ
- エントリは`sam/app/app.py`一本: ローカル（`uv run python sam/app/app.py`）とLambda（`lambda_handler`）を兼用
- Webhook URLはSecrets Manager管理: Lambda環境では`TEAMS_SECRET_ARN`経由で安全に取得
- Teams通知はAdaptive Card + textフォールバック: `TEAMS_WEBHOOK_FORMAT=text`で形式の切り替えも可能
- Office 365コネクタは廃止済み: Power Automateワークフロー用Webhookへ移行する
- 運用面の強化: DLQ（SQS）とCloudWatch Alarmで障害検知をカバー
- arm64（Graviton2）採用: 約20%のコスト削減

---

# 参考リンク
- [t-tkm/aws-cost-explore-lambda（本記事の実装）](https://github.com/t-tkm/aws-cost-explore-lambda)
- [uv - Python package manager](https://docs.astral.sh/uv/)
- [Python 3.12 runtime now available in AWS Lambda](https://aws.amazon.com/blogs/compute/python-3-12-runtime-now-available-in-aws-lambda/)
- [Install the AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html)
- [Retirement of Office 365 connectors within Microsoft Teams](https://devblogs.microsoft.com/microsoft365dev/retirement-of-office-365-connectors-within-microsoft-teams/)
- [AWS Secrets Manager ドキュメント](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)

---

Amazon Web Services、およびその他のAWS商標は、米国およびその他の諸国におけるAmazon.com, Inc.またはその関連会社の商標です。
Microsoft 365、Microsoft Teamsは、米国Microsoft Corporationおよびその関連会社の米国およびその他の国における登録商標または商標です。
