+++
Categories = ["AWS"]
Tags = ["AWS", "Lambda", "SAM", "Cost", "Teams", "FinOps"]
date = "2026-03-28T00:00:00+09:00"
title = "AWS費用監視ツール（後編：Lambda活用）"
archives = ["2026", "2026-03", "2026-03-28"]
+++

# 1. はじめに

本記事は、[AWS費用監視ツール（前編：AWSマネジメントコンソール & AWS CLI）](https://t-tkm.github.io/blog/posts/2026/03/aws_costcheck_cli/) の続編です。

前編では、マネジメントコンソールの Cost Explorer で見える内容を、AWS CLI の `aws ce get-cost-and-usage` で同じように取り出すところまでを整理しました。CLI で取得できるようになれば、後編の「Lambda によるデイリー通知の自動化」への道がぐっと近くなる、という流れでした。

後編では、その Lambda 側の実装に踏み込みます。具体的には、Python（boto3）で Cost Explorer API を呼び出し、Microsoft Teams へ通知しつつ、AWS SAM で定期実行するまでの一連の手順をまとめます。

## 1.1 2021年版（Qiita）記事からの読み替え

本記事は、筆者が以前 Qiita に公開していた「[AWS費用監視ツール(後編:Lambda活用)](https://qiita.com/t-taku/items/56f0e79dda55d89107e4)」の**更新版**として読めます。当時の記事は `sam init` で生成した雛形をベースに、`app_shell.py` をローカル用、`lambda_handler` を Lambda 用として切り替える構成でした。2026年版では **1 本の `app.py` にローカル実行と Lambda をまとめ**、仮想環境・ランタイム・Teams の通知方式・シークレット管理・SAM のデプロイ手順まで、前提が変わっている箇所が多いです。

次の表は、**Qiita の章立てと本記事の対応**です。迷ったら、まず同じ番号の章を読み替えてください。

| Qiita（2021年版） | 本記事（2026年版）で扱う内容 |
|-------------------|------------------------------|
| §2「プログラム(Python)による確認」 | §3（boto3・ディレクトリ構成・ローカル実行）。Miniconda ではなく **uv** |
| §3「Lambda へのデプロイ（AWS SAM）」 | §5（`template.yaml` の差分・デプロイ）。`sam package` 経由が前提ではなくなった点に注意 |
| 付録「Miniconda」 | §2（**uv** に置き換え） |
| （Qiita 本文内の Teams 説明） | §4（**Office 365 コネクタ廃止**とワークフロー Webhook） |

本記事の手順・コードは、以下の実装リポジトリと整合するように書いています。最新のソースやテスト、`sam/` 配下のそのまま動く構成は GitHub を正としてください。

<a href="https://github.com/t-tkm/aws-cost-explore-lambda" target="_blank" rel="noopener">
    <img src="https://gh-card.dev/repos/t-tkm/aws-cost-explore-lambda.svg">
</a>

## 1.2 本記事の位置づけ（前編・認証）

前編で触れた IAM Identity Center（SSO）と AWS CLI v2 のセットアップは、ローカルで `sam/app/app.py` を試すときにもそのまま使えます。まだの方は、前編の付録（許可セットと CLI のセットアップ）を先に済ませておくとスムーズです。

Qiita 版では `export AWS_PROFILE=billing-user` のように **名前付きプロファイル** を前提にしていましたが、いまは **SSO プロファイル** を `AWS_PROFILE` に指定する想定です（認証の中身は「長期アクセスキー」から「一時認証情報」へ変わっています）。

また、前編のとおり Cost Explorer API はリクエストごとに料金が発生します。Lambda を「1日1回」など低頻度で動かす想定であれば、個人アカウントの試用範囲では大きな問題になりにくい一方、短い間隔で何度も呼び出すと API 料金とスロットリングの両面で負荷が増える点は、デプロイ後も意識しておくとよいです。

## 1.3 2021年版（Qiita）との主な差分一覧

Qiita の手順をそのまま追うと動かなくなる典型は、**(1) Teams の Webhook URL の取り方** と **(2) Python 3.8 Lambda ランタイムの終了** です。まずはこの表を俯瞰してから、各章に進むと理解しやすいです。

| 観点 | 2021年版（Qiita） | 2026年版（本リポジトリ） |
|------|-------------------|------------------------|
| Python（ローカル／Lambda） | 3.8、Miniconda + `conda install` | 3.12、**uv**（`uv sync`） |
| エントリコード | `app_shell.py`（ローカル）と Lambda 用の切り分け説明 | **`sam/app/app.py` 1 ファイル**（`__main__` と `lambda_handler` 共存） |
| Teams 通知 | Office 365 コネクタの Webhook URL | **ワークフロー** Webhook（Adaptive Card／テキスト切替） |
| Webhook の渡し方 | SAM パラメータ `TeamsWebhookUrl` で **平文** | **Secrets Manager**（`TEAMS_SECRET_ARN`） |
| SAM デプロイ | `sam build` → `sam package` → `sam deploy`（`packaged.yaml`） | **`sam build` → `sam deploy`**（中間の package を前提にしない） |
| Lambda アーキテクチャ | x86_64 | **arm64**（Graviton） |
| 失敗時の扱い | 記事上は未整理 | **DLQ（SQS）** と **CloudWatch Alarm** |

# 2. Python 実行環境の準備

## 2.1 Python 3.8 のサポート終了

2021年版の 3.8 + Miniconda は、構成として古くなっています。Python 3.8 は EOL を迎えており、Lambda のマネージドランタイムもサポート終了済みです。今回は Python 3.12 を採用しています。

## 2.2 uv によるローカル環境構築（Qiita「付録: Miniconda」の置き換え）

Qiita では仮想環境 `billing-3.8` を `conda create` で作り、`conda install boto3 requests` としていました。2026年版ではパッケージマネージャに [uv](https://docs.astral.sh/uv/) を使います。`uv sync` 一発で `.venv` の作成から依存インストールまで完結するのがとにかく快適で、従来の `venv` + `pip` に戻る気は起きません。

```bash
git clone https://github.com/t-tkm/aws-cost-explore-lambda.git
cd aws-cost-explore-lambda

# 仮想環境の作成 & 依存パッケージインストール
uv sync
```

`pyproject.toml` に基づき、`boto3` や `requests`、`pytest` 等がセットアップされます。

# 3. Python（boto3）によるコスト取得

## 3.1 認証

Cost Explorer API は `us-east-1` のエンドポイントを叩く必要があります。ローカル実行時は SSO プロファイルを `AWS_PROFILE` で指定し、Lambda では IAM ロール経由で認証します。

## 3.2 ソースコードの構成（Qiita §2-2 からの変化）

Qiita では `app_shell.py` に処理を書き、Lambda 用には「`if __name__ == '__main__':` を削って `lambda_handler` を足す」という手順でした。2026年版では **同じファイル**にローカル用のエントリと Lambda ハンドラの両方を持たせ、テストやデバッグの行き来を減らしています。

リポジトリの主要なディレクトリ構成は次のとおりです。

```text
aws-cost-explore-lambda/
├── sam/
│   ├── template.yaml          ← SAM テンプレート
│   ├── app/
│   │   ├── app.py             ← Lambda ハンドラ & ローカル実行兼用
│   │   └── requirements.txt   ← Lambda 用
├── tests/                     ← pytest
├── pyproject.toml             ← uv 設定
└── README.md
```

エントリポイントは `sam/app/app.py` です。`if __name__ == "__main__":` を入れているので、ローカルでも Lambda と同じロジックをそのまま試せます。

実装のポイントは次のとおりです。

- `CostExplorer` クラスで API をラップし、クレジット除外フィルタなどを共通化している。
- `get_config()` で環境変数を集約し、Secrets Manager からの URL 取得もここで行う。
- Teams 通知は `requests` で POST し、Adaptive Card を試し、うまくいかなければテキストで送る二段構えにしてある。
- レポートに `get_caller_identity` で取得したアカウント ID を付与し、複数アカウント運用時にも見分けがつくようにしている。

## 3.3 ローカルでの動作確認

認証を通したあと、`uv run` で実行します。

```bash
# SSO ログイン
aws sso login --profile <your-sso-profile>

# 環境変数の設定（Teams 通知を使う場合）
export USE_TEAMS_POST=yes
export TEAMS_WEBHOOK_URL="https://..."
```

実行コマンドは次のとおりです。

```bash
uv run python sam/app/app.py
```

実行すると、標準出力に以下のようなレポートが出ます。

```text
INFO:__main__:AWS Account ID: 123456789012
------------------------------------------------------
AWSアカウント 123456789012
03/01～03/27のクレジット適用後費用は、12.34 USD です。
- Amazon EC2: 5.00 USD
- Amazon S3: 3.00 USD
...
------------------------------------------------------
```

# 4. Microsoft Teams 通知の移行

## 4.1 Office 365 コネクタの廃止（重要）

ここが一番のハマりどころです。Qiita のコードコメントにあった `webhook.office.com` 形式の **Office 365 コネクタ**は、すでに廃止されています。旧 URL はそのままでは使えません。今後は Power Automate などの **ワークフロー** を経由した Webhook に差し替える必要があります。

## 4.2 ワークフローの作成

Power Automate で「Teams webhook 要求が受信されたとき」をトリガーにするフローを作成し、発行された URL を使います。

リポジトリのコードは Adaptive Card 形式を送りますが、表示が崩れる場合は `TEAMS_WEBHOOK_FORMAT=text` を設定すれば、シンプルなテキスト形式に切り替えられるようにしてあります。

# 5. AWS SAM による Lambda デプロイ

`sam/` ディレクトリにテンプレートを置いています。

## 5.1 SAM テンプレート（`template.yaml`）

Qiita では `TeamsWebhookUrl` をパラメータで渡し、環境変数にそのまま載せる形でした。2026年版では **Secrets Manager** を前提にし、Webhook URL をテンプレートやコマンドラインに残しにくくしています。そのほか、次のような運用面の追記があります。

* **Runtime**: `python3.12` / `arm64`（Graviton2）
* **Schedule**: `cron(0 0 * * ? *)`。毎日 JST 9:00 に実行
* **Security**: Webhook URL は環境変数にベタ書きせず、Secrets Manager（`TEAMS_SECRET_ARN`）から動的に取得
* **Reliability**: DLQ（SQS）を設定し、失敗イベントをロストしないようにする。エラー時の CloudWatch Alarm もセットで作成

## 5.2 デプロイ手順（`sam package` 前提からの変更）

Qiita では `sam package` で `packaged.yaml` を生成してから `sam deploy` していました。2026年版の手順では、**`sam build` のあと `sam deploy` を直接**呼ぶ形に寄せています（プロジェクトや CI の方針に合わせて、従来型の `package` フローを取ることも可能です）。

まず Teams 通知用の URL を Secrets Manager に登録します。

```bash
aws secretsmanager create-secret \
  --name teams-webhook-url \
  --secret-string "https://..."
```

あとはビルドしてデプロイします。

```bash
cd sam
sam build
sam deploy --parameter-overrides \
    UseTeamsPost=yes \
    TeamsWebhookSecretArn=arn:aws:secretsmanager:...
```

スタックを片付けるときは、Qiita 版と同様に **`sam delete`** で CloudFormation スタックと関連リソースをまとめて削除できます（リージョンやスタック名は環境に合わせてください）。

# 6. まとめ

Qiita の後編で扱った「boto3 で Cost Explorer を叩き、SAM で Lambda に載せ、Teams に飛ばす」という骨格はそのままです。変わったのは主に次の点です。

* **Python 3.8 → 3.12** と **uv**：ローカルと Lambda の両方を現行ランタイムに載せる
* **Teams**：廃止されたコネクタから **ワークフロー Webhook** へ
* **Secrets Manager**：URL をパラメータ平文で渡さない
* **SAM**：`sam build` 中心のデプロイ、**arm64**、失敗時の **DLQ と Alarm**

Qiita のまとめ（前編〜後編通して）で触れたように、マネコンの UI は変わりやすい一方、CLI や API は比較的ゆっくり移行します。本記事の更新も、その延長線上で「同じことを 2026年の前提でやり直す」イメージです。

毎日自動で費用サマリが届くようになると、開発の心理的安全性がだいぶ上がる、というのが筆者の実感です。細かい実装やテストコードは [GitHub](https://github.com/t-tkm/aws-cost-explore-lambda) を参照してください。
