+++
Categories = ["AWS"]
Tags = ["AWS", "Lambda", "SAM", "Cost", "Teams", "FinOps"]
date = "2026-04-01T00:00:00+09:00"
title = "AWS費用監視ツール（後編：Lambda活用）"
archives = ["2026", "2026-04", "2026-04-01"]
+++

# 1. はじめに

本記事は、[AWS費用監視ツール（前編：AWSマネジメントコンソール & AWS CLI）](https://t-tkm.github.io/blog/posts/2026/03/aws_costcheck_cli/) の続編です。

前編では、マネジメントコンソールの Cost Explorer で見える内容を、AWS CLI の `aws ce get-cost-and-usage` で同じように取り出すところまでを整理しました。CLI で取得できるようになれば、後編の「Lambda によるデイリー通知の自動化」への道がぐっと近くなる、という流れでした。

後編では、その Lambda 側の実装に踏み込みます。具体的には、Python（boto3）で Cost Explorer API を呼び出し、Microsoft Teams へ通知しつつ、AWS SAM で定期実行するまでの一連の手順をまとめます。

## 1.1 2021年版（Qiita）記事からの読み替え

本記事は、筆者が以前 Qiita に公開していた「[AWS費用監視ツール(後編:Lambda活用)](https://qiita.com/t-taku/items/56f0e79dda55d89107e4)」の更新版として読めます。当時の記事は `sam init` で生成した雛形をベースに、`app_shell.py` をローカル用、`lambda_handler` を Lambda 用として切り替える構成でした。2026年版では 1 本の `app.py` にローカル実行と Lambda をまとめ、仮想環境・ランタイム・Teams の通知方式・シークレット管理・SAM のデプロイ手順まで、前提が変わっている箇所が多いです。

次の表は、Qiita の章立てと本記事の対応です。迷ったら、まず同じ番号の章を読み替えてください。

| Qiita（2021年版） | 本記事（2026年版）で扱う内容 |
|-------------------|------------------------------|
| §2「プログラム(Python)による確認」 | §3（boto3・ディレクトリ構成・ローカル実行）。Miniconda ではなく uv |
| §3「Lambda へのデプロイ（AWS SAM）」 | §5（`template.yaml` の差分・デプロイ）。`sam package` 経由が前提ではなくなった点に注意 |
| 付録「Miniconda」 | §2（uv に置き換え） |
| （Qiita 本文内の Teams 説明） | §4（Office 365 コネクタ廃止とワークフロー Webhook） |

本記事の手順・コードは、以下の実装リポジトリと整合するように書いています。最新のソースやテスト、`sam/` 配下のそのまま動く構成は GitHub を正としてください。

<a href="https://github.com/t-tkm/aws-cost-explore-lambda" target="_blank" rel="noopener">
    <img src="https://gh-card.dev/repos/t-tkm/aws-cost-explore-lambda.svg">
</a>

> **注:** 本記事および実装リポジトリは、従来の Qiita 記事実装（[t-tkm/aws-cost-explore](https://github.com/t-tkm/aws-cost-explore)）をベースに、構成し直しました。
>
> ディレクトリ構成や設計も刷新しており差分があるため、古いリポジトリと混同しないようご注意ください。

## 1.2 本記事の位置づけ（前編・認証）

前編で触れた IAM Identity Center（SSO）と AWS CLI v2 のセットアップは、ローカルで `sam/app/app.py` を試すときにもそのまま使えます。まだの方は、前編の付録（許可セットと CLI のセットアップ）を先に済ませておくとスムーズです。

Qiita 版では `export AWS_PROFILE=billing-user` のように 名前付きプロファイル を前提にしていましたが、いまは SSO プロファイル を `AWS_PROFILE` に指定する想定です（認証の中身は「長期アクセスキー」から「一時認証情報」へ変わっています）。

また、前編のとおり Cost Explorer API はリクエストごとに料金が発生します。Lambda を「1日1回」など低頻度で動かす想定であれば、個人アカウントの試用範囲では大きな問題になりにくい一方、短い間隔で何度も呼び出すと API 料金とスロットリングの両面で負荷が増える点は、デプロイ後も意識しておくとよいです。

## 1.3 2021年版（Qiita）との主な差分一覧

Qiita の手順をそのまま追うと動かなくなる典型は、(1) Teams の Webhook URL の取り方 と (2) Python 3.8 Lambda ランタイムの終了 です。まずはこの表を俯瞰してから、各章に進むと理解しやすいです。

| 観点 | 2021年版（Qiita） | 2026年版（本リポジトリ） |
|------|-------------------|------------------------|
| Python（ローカル／Lambda） | 3.8、Miniconda + `conda install` | 3.12、uv（`uv sync`） |
| エントリコード | `app_shell.py`（ローカル）と Lambda 用の切り分け説明 | `sam/app/app.py` 1 ファイル（`__main__` と `lambda_handler` 共存） |
| Teams 通知 | Office 365 コネクタの Webhook URL | ワークフロー Webhook（Adaptive Card／テキスト切替） |
| Webhook の渡し方 | SAM パラメータ `TeamsWebhookUrl` で 平文 | Secrets Manager（`TEAMS_SECRET_ARN`） |
| SAM デプロイ | `sam build` → `sam package` → `sam deploy`（`packaged.yaml`） | `sam build` → `sam deploy`（中間の package を前提にしない） |
| Lambda アーキテクチャ | x86_64 | arm64（Graviton） |
| 失敗時の扱い | 記事上は未整理 | DLQ（SQS） と CloudWatch Alarm |

# 2. Python 実行環境の準備

## 2.1 Python 3.8 のサポート終了

2021年版の 3.8 + Miniconda は、構成として古くなっています。Python 3.8 は EOL を迎えており、Lambda のマネージドランタイムもサポート終了済みです。今回は Python 3.12 を採用しています。

## 2.2 uv によるローカル環境構築（Qiita「付録: Miniconda」の置き換え）

Qiita では仮想環境 `billing-3.8` を `conda create` で作り、`conda install boto3 requests` としていました。2026年版ではパッケージマネージャに [uv](https://docs.astral.sh/uv/) を使います。`uv sync` 一発で `.venv` の作成から依存インストールまで完結するのがとにかく快適です。

```bash
git clone https://github.com/t-tkm/aws-cost-explore-lambda.git
cd aws-cost-explore-lambda

# 仮想環境の作成 & 依存パッケージインストール
uv sync
```

`pyproject.toml` に基づき、`boto3` や `requests`、`pytest` 等がセットアップされます。

# 3. Python（boto3）によるコスト取得
[![img2](https://imgur.com/E7zVnDk.png)](https://imgur.com/E7zVnDk.png)

## 3.1 認証

Cost Explorer API は `us-east-1` のエンドポイントを叩く必要があります。ローカル実行時は SSO プロファイルを `AWS_PROFILE` で指定し、Lambda では IAM ロール経由で認証します。

## 3.2 ソースコードの構成（Qiita §2-2 からの変化）

Qiita では `app_shell.py` に処理を書き、Lambda 用には「`if __name__ == '__main__':` を削って `lambda_handler` を足す」という手順でした。2026年版では 同じファイル にローカル用のエントリと Lambda ハンドラの両方を持たせ、テストやデバッグの行き来を減らしています。

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

次の簡略版は流れの把握用です。`get_config()` はローカル実行向けに `TEAMS_WEBHOOK_URL` のみを見ていますが、実装本体では Lambda 時に `TEAMS_SECRET_ARN` から Secrets Manager へ取りにいく分岐があります（下の「全コードを表示」がリポジトリと一致します）。

```python {linenos=true}
import os
from datetime import datetime, timedelta, date
from typing import Any, Dict, List, Tuple
import boto3
import requests

# 記事向けの簡略版:
# - 主要フローのみ残す
# - 詳細実装は (snip) で省略
REGION_NAME = "us-east-1"
def get_config() -> dict:
    """環境変数から実行設定を取得する。"""
    use_teams = os.environ.get("USE_TEAMS_POST", "no").lower() == "yes"
    webhook_url = (os.environ.get("TEAMS_WEBHOOK_URL") or "").strip()
    return {"USE_TEAMS_POST": use_teams, "TEAMS_WEBHOOK_URL": webhook_url}
class CostExplorer:
    """Cost Explorer API を扱う最小ラッパー。"""
    def __init__(self, client):
        self.client = client
    def get_cost_and_usage(self, period: Dict[str, str], include_credit: bool) -> Dict[str, Any]:
        # (snip) include_credit に応じた Filter、GroupBy などの詳細
        return self.client.get_cost_and_usage(
            TimePeriod=period,
            Granularity="MONTHLY",
            Metrics=["AmortizedCost"],
            # (snip)
        )["ResultsByTime"][0]

    def get_total_cost(self, data: Dict[str, Any]) -> float:
        # (snip) Total が無い場合のフォールバック処理
        return float(data["Total"]["AmortizedCost"]["Amount"])
def get_client():
    return boto3.client("ce", region_name=REGION_NAME)
def get_date_range() -> Tuple[str, str]:
    """
    集計期間を取得する。
    Cost Explorer の TimePeriod は End が排他的で、Start は End より前である必要がある。
    月初当日だけ Start と「今日」を End にすると同一日になり無効になるため、
    その場合は End を月初の翌日に補正する。
    """
    month_start = date.today().replace(day=1)
    today = date.today()
    end_date = today
    if end_date <= month_start:
        end_date = month_start + timedelta(days=1)
    return month_start.isoformat(), end_date.isoformat()
def build_title(account_id: str, start_date: str, end_date: str, total_cost: float, include_credit: bool) -> str:
    """通知用タイトルを組み立てる。"""
    start_day = datetime.strptime(start_date, "%Y-%m-%d").strftime("%m/%d")
    end_day = (datetime.strptime(end_date, "%Y-%m-%d") - timedelta(days=1)).strftime("%m/%d")
    credit_label = "後" if include_credit else "前"
    return (
        f"AWSアカウント {account_id}\n"
        f"{start_day}～{end_day}のクレジット適用{credit_label}費用は、{total_cost:.2f} USD です。"
    )
def post_to_teams(title: str, lines: List[str], webhook_url: str) -> bool:
    payload = {"text": f"{title}\n\n" + "\n".join(lines)}
    # (snip) Adaptive Card フォールバック、HTTP リトライ設定
    r = requests.post(webhook_url, json=payload, timeout=10)
    r.raise_for_status()
    return True
def run_report(explorer: CostExplorer, period: Dict[str, str], account_id: str, include_credit: bool) -> Tuple[str, List[str]]:
    """1種類のレポート（適用前 or 適用後）を作る。"""
    data = explorer.get_cost_and_usage(period, include_credit=include_credit)
    total_cost = explorer.get_total_cost(data)
    # (snip) サービス別内訳の整形。例: ['- Amazon EC2: 5.00 USD', ...]
    services = ["- Amazon EC2: 5.00 USD"]
    title = build_title(account_id, period["Start"], period["End"], total_cost, include_credit)
    return title, services
def notify_if_needed(enabled: bool, webhook_url: str, title: str, services: List[str]) -> None:
    if enabled and webhook_url:
        post_to_teams(title, services, webhook_url)
def main() -> None:
    config = get_config()
    explorer = CostExplorer(get_client())
    account_id = boto3.client("sts").get_caller_identity()["Account"]
    start_date, end_date = get_date_range()
    period = {"Start": start_date, "End": end_date}
    # クレジット適用後
    title_after, services_after = run_report(explorer, period, account_id, include_credit=True)
    print(title_after)
    print("\n".join(services_after))
    notify_if_needed(config["USE_TEAMS_POST"], config["TEAMS_WEBHOOK_URL"], title_after, services_after)
    # クレジット適用前
    title_before, services_before = run_report(explorer, period, account_id, include_credit=False)
    print(title_before)
    print("\n".join(services_before))
    notify_if_needed(config["USE_TEAMS_POST"], config["TEAMS_WEBHOOK_URL"], title_before, services_before)
def lambda_handler(event: dict, context: Any) -> dict:
    try:
        main()
        return {"statusCode": 200, "body": "Cost report generated successfully."}
    except Exception as e:
        return {"statusCode": 500, "body": str(e)}
if __name__ == "__main__":
    main()
```

<details>
<summary>全コードを表示</summary>

```python {linenos=true}
import os
import json
import logging
from datetime import datetime, timedelta, date
from typing import Tuple, List, Dict, Any, Optional
from urllib.parse import urlparse

import boto3
import botocore.client
import botocore.exceptions
import requests

from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

# --------------------------------------------------------------------
# 定数定義
# --------------------------------------------------------------------
REGION_NAME = "us-east-1"
GRANULARITY = "MONTHLY"
COST_METRIC = "AmortizedCost"
SERVICE_GROUP_DIMENSION = "SERVICE"
RECORD_TYPE_DIMENSION = "RECORD_TYPE"
CREDIT_RECORD_TYPE = "Credit"
MIN_BILLING_THRESHOLD = 0.01
TEAMS_REQUEST_TIMEOUT_SEC = 10  # Webhook POST のタイムアウト（秒）
MAX_RETRIES = 3

# ロギング設定
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


def _parse_webhook_url(raw: str) -> str:
    """
    Secrets Manager に JSON で保存している場合（例: {\"url\":\"https://...\"}）に対応する。
    平文の URL のときはそのまま返す。
    """
    text = (raw or "").strip()
    if not text.startswith("{"):
        return text
    try:
        obj = json.loads(text)
        if isinstance(obj, dict):
            for key in ("webhook_url", "url", "TEAMS_WEBHOOK_URL", "value"):
                v = obj.get(key)
                if isinstance(v, str) and v.strip().startswith("http"):
                    return v.strip()
    except json.JSONDecodeError:
        pass
    return text


# --------------------------------------------------------------------
# 実行時に環境変数を取得する関数
# --------------------------------------------------------------------
def get_config() -> dict:
    """
    環境変数を実行時に取得して返す。
    Teams 通知が有効な場合、Webhook URL は次のいずれかから取得する。

    1. TEAMS_WEBHOOK_URL … ローカル実行向け（平文のため Lambda では使わない想定）
    2. TEAMS_SECRET_ARN … Lambda 向け（Secrets Manager に URL を保存）

    Returns:
        dict: USE_TEAMS_POST, TEAMS_WEBHOOK_URL をキーに含む辞書

    Raises:
        ValueError: Teams 有効だが URL の取得元がどちらも無い場合
        RuntimeError: Secrets Manager からの取得に失敗した場合
    """
    use_teams = os.environ.get("USE_TEAMS_POST", "no").lower() == "yes"
    webhook_url: Optional[str] = None

    if use_teams:
        direct = (os.environ.get("TEAMS_WEBHOOK_URL") or "").strip()
        if direct:
            logger.info("Teams Webhook URL は環境変数 TEAMS_WEBHOOK_URL を使用します（ローカル向け）。")
            webhook_url = _parse_webhook_url(direct)
        else:
            secret_arn = (os.environ.get("TEAMS_SECRET_ARN") or "").strip()
            if not secret_arn:
                raise ValueError(
                    "Teams 通知を有効にするには、TEAMS_WEBHOOK_URL（ローカル）または "
                    "TEAMS_SECRET_ARN（Secrets Manager の ARN）のいずれかを設定してください。"
                    " ターミナルでは必ず export してください（export なしの代入は echo では見えても "
                    "uv run の子プロセスには渡りません）。"
                )
            try:
                sm_client = boto3.client("secretsmanager")
                secret = sm_client.get_secret_value(SecretId=secret_arn)
                raw = (secret.get("SecretString") or "").strip()
                webhook_url = _parse_webhook_url(raw)
                logger.info("Secrets Manager から Teams Webhook URL を取得しました。")
            except botocore.exceptions.ClientError as e:
                logger.error(f"Secrets Manager からの取得に失敗しました: {e}")
                raise RuntimeError("Teams Webhook URL の取得に失敗しました。") from e

        if not webhook_url:
            raise ValueError("Teams Webhook URL が空です。Secrets の値または TEAMS_WEBHOOK_URL を確認してください。")

        parsed = urlparse(webhook_url)
        if parsed.scheme not in ("http", "https") or not parsed.netloc:
            raise ValueError(
                "Teams Webhook URL が無効です（http(s):// で始まる実際の URL ではありません）。"
                "README のプレースホルダー「（Webhook URL）」などをそのまま使っていないか確認し、"
                "Teams / Power Automate でコピーした https://... の文字列を設定してください。"
            )

    return {
        "USE_TEAMS_POST": use_teams,
        "TEAMS_WEBHOOK_URL": webhook_url,
    }


# --------------------------------------------------------------------
# クラス・関数定義
# --------------------------------------------------------------------
class CostExplorer:
    """
    AWS Cost Explorer API を用いてコスト情報を取得するクラス。
    """

    def __init__(self, client: botocore.client.BaseClient) -> None:
        self.client = client

    def get_cost_and_usage(
        self,
        period: Dict[str, str],
        include_credit: bool,
        group_by_dimension: Optional[str] = None
    ) -> Dict[str, Any]:
        """
        指定期間のコストと使用状況を取得する。
        """
        try:
            filter_params: Dict[str, Any] = {}
            if not include_credit:
                filter_params = {
                    "Filter": {
                        "Not": {
                            "Dimensions": {
                                "Key": RECORD_TYPE_DIMENSION,
                                "Values": [CREDIT_RECORD_TYPE]
                            }
                        }
                    }
                }

            group_by = []
            if group_by_dimension:
                group_by = [{"Type": "DIMENSION", "Key": group_by_dimension}]

            response = self.client.get_cost_and_usage(
                TimePeriod=period,
                Granularity=GRANULARITY,
                Metrics=[COST_METRIC],
                GroupBy=group_by,
                **filter_params
            )
            return response["ResultsByTime"][0]

        except botocore.exceptions.ClientError as e:
            logger.error(f"Failed to fetch cost and usage data: {e}")
            raise RuntimeError(f"Error calling AWS Cost Explorer API: {e}") from e

    def get_total_cost(self, cost_and_usage_data: Dict[str, Any]) -> float:
        """
        コストと使用状況のデータから合計費用を取得する。
        """
        try:
            if not cost_and_usage_data.get("Total"):
                total_cost = sum(
                    max(0, float(group["Metrics"][COST_METRIC]["Amount"]))
                    for group in cost_and_usage_data.get("Groups", [])
                )
                logger.info(f"Calculated total cost from Groups: {total_cost:.2f} USD")
                return total_cost

            return float(cost_and_usage_data["Total"][COST_METRIC]["Amount"])

        except KeyError as e:
            logger.error(f"Metric '{COST_METRIC}' not found in response data: {e}")
            return 0.0

    def get_service_costs(self, cost_and_usage_data: Dict[str, Any]) -> List[Dict[str, Any]]:
        """
        コストと使用状況のデータからサービスごとの費用を取得する。
        """
        service_groups = cost_and_usage_data.get("Groups", [])
        result = []
        for item in service_groups:
            billing_amount = float(item["Metrics"][COST_METRIC]["Amount"])
            result.append({
                "service_name": item["Keys"][0],
                "billing": billing_amount
            })
        return result


def get_client() -> botocore.client.BaseClient:
    """
    boto3 Cost Explorer クライアントを返す。
    """
    return boto3.client("ce", region_name=REGION_NAME)


def get_date_range() -> Tuple[str, str]:
    """
    集計期間を取得する。
    Cost Explorer の TimePeriod は End が排他的で、Start は End より前である必要がある。
    月初当日だけ Start と「今日」を End にすると同一日になり無効になるため、
    その場合は End を月初の翌日に補正する。
    """
    month_start = date.today().replace(day=1)
    today = date.today()
    end_date = today
    if end_date <= month_start:
        end_date = month_start + timedelta(days=1)
    return month_start.isoformat(), end_date.isoformat()


def format_service_costs(service_billings: List[Dict[str, Any]]) -> List[str]:
    """
    サービスごとの費用を表示用に整形する。
    """
    formatted_services = []
    for item in service_billings:
        billing = item["billing"]
        if billing >= MIN_BILLING_THRESHOLD:
            formatted_services.append(f"- {item['service_name']}: {billing:.2f} USD")
        else:
            logger.debug(f"Excluded negligible cost: {item['service_name']} ({billing:.5f})")
    return formatted_services


def handle_cost_report(
    explorer: CostExplorer,
    period: Dict[str, str],
    include_credit: bool,
    start_day: str,
    end_day: str
) -> Tuple[str, List[str]]:
    """
    費用レポート（クレジット適用前/後）の取得と整形を行う。
    """
    cost_and_usage = explorer.get_cost_and_usage(
        period,
        include_credit=include_credit,
        group_by_dimension=SERVICE_GROUP_DIMENSION
    )
    total_cost = explorer.get_total_cost(cost_and_usage)
    services_cost = explorer.get_service_costs(cost_and_usage)
    formatted_services = format_service_costs(services_cost)

    credit_text = "後" if include_credit else "前"
    title = f"{start_day}～{end_day}のクレジット適用{credit_text}費用は、{total_cost:.2f} USD です。"
    return title, formatted_services


def print_report(title: str, services_cost: List[str]) -> None:
    """
    レポートを標準出力に表示する。
    """
    print("------------------------------------------------------")
    print(title)
    if services_cost:
        print("\n".join(services_cost))
    else:
        print("サービスごとの費用データはありません。")
    print("------------------------------------------------------\n")


def _teams_payload_legacy_adaptive(title: str, services_text: str) -> Dict[str, Any]:
    """
    旧 src/cost_report.py で Teams に表示できていた形式と同一。
    （attachments のみ・AdaptiveCard 1.2・TextBlock に markdown: true）
    """
    return {
        "attachments": [
            {
                "contentType": "application/vnd.microsoft.card.adaptive",
                "content": {
                    "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
                    "type": "AdaptiveCard",
                    "version": "1.2",
                    "body": [
                        {
                            "type": "TextBlock",
                            "text": f"### {title}\n\n{services_text}",
                            "wrap": True,
                            "markdown": True,
                        }
                    ],
                },
            }
        ]
    }


def _teams_payload_text(title: str, services_text: str) -> Dict[str, str]:
    """Microsoft 公式 curl 例と同じ {\"text\": \"...\"}（フォールバック用）。"""
    return {"text": f"{title}\n\n{services_text}"}


def post_to_teams(title: str, services_cost: List[str], webhook_url: str) -> bool:
    """
    Teams Webhook に POST する。
    既定は旧 cost_report.py と同じ Adaptive（legacy_adaptive）。失敗時のみ {\"text\": ...}。

    TEAMS_WEBHOOK_FORMAT=text のときは text のみ送る。

    Returns:
        bool: いずれかの形式で成功すれば True。すべて失敗なら False（ログのみ）。
    """

    services_text = "\n".join(services_cost) if services_cost else "サービスごとの費用データはありません。"

    fmt = (os.environ.get("TEAMS_WEBHOOK_FORMAT") or "").strip().lower()
    if fmt == "text":
        strategies: List[Tuple[str, Dict[str, Any]]] = [
            ("text", _teams_payload_text(title, services_text)),
        ]
    else:
        strategies = [
            ("legacy_adaptive", _teams_payload_legacy_adaptive(title, services_text)),
            ("text", _teams_payload_text(title, services_text)),
        ]

    # HTTPリトライの設定
    session = requests.Session()
    retries = Retry(
        total=MAX_RETRIES,
        backoff_factor=1,
        status_forcelist=[500, 502, 503, 504],
        allowed_methods=["POST"]
    )
    session.mount("https://", HTTPAdapter(max_retries=retries))

    last_error: Optional[requests.exceptions.RequestException] = None
    for idx, (name, payload) in enumerate(strategies):
        try:
            if name == "legacy_adaptive":
                # 旧実装どおり json を文字列化して data= で送る（Incoming Webhook での表示実績あり）
                response = session.post(
                    webhook_url,
                    data=json.dumps(payload, ensure_ascii=False),
                    headers={"Content-Type": "application/json"},
                    timeout=TEAMS_REQUEST_TIMEOUT_SEC,
                )
            else:
                response = session.post(
                    webhook_url,
                    json=payload,
                    headers={"Content-Type": "application/json; charset=utf-8"},
                    timeout=TEAMS_REQUEST_TIMEOUT_SEC,
                )
            response.raise_for_status()
            # Power Automate 等は 202 で本文が空のことが多い。HTTP 成功でもチャネル投稿は別ステップ依存。
            body_preview = (response.text or "").strip()[:400]
            if fmt == "text" and len(strategies) == 1:
                logger.info(
                    "Teams: TEAMS_WEBHOOK_FORMAT=text のため、Adaptive は送らず text のみ送信します。"
                )
            elif idx > 0:
                logger.info(
                    "Teams: 形式「%s」が失敗したため「%s」で再送し、HTTP %s になりました（直前の WARNING に失敗理由があります）。",
                    strategies[idx - 1][0],
                    name,
                    response.status_code,
                )
            logger.info(
                "Teams Webhook へ HTTP %s（形式: %s）。応答本文の先頭: %r",
                response.status_code,
                name,
                body_preview if body_preview else "(空)",
            )
            return True
        except requests.exceptions.RequestException as e:
            last_error = e
            resp = getattr(e, "response", None)
            extra = ""
            if resp is not None:
                snippet = (resp.text or "")[:800]
                extra = f" HTTP {resp.status_code} body={snippet!r}"
            logger.warning(
                "Teams Webhook（形式: %s）が失敗しました。%s",
                name,
                extra or str(e),
            )

    if last_error is not None:
        resp = getattr(last_error, "response", None)
        tail = ""
        if resp is not None:
            tail = f" HTTP {resp.status_code} body={(resp.text or '')[:800]!r}"
        logger.error(
            "Teams Webhook への通知はすべての形式で失敗しました（処理は継続します）: %s%s",
            last_error,
            tail,
        )
    return False


def get_account_id(sts_client: Optional[botocore.client.BaseClient] = None) -> str:
    """
    AWSアカウントIDを取得する。

    Args:
        sts_client: STSクライアント。省略時は boto3 デフォルトクライアントを使用。
    """
    try:
        client = sts_client or boto3.client("sts")
        account_id = client.get_caller_identity()["Account"]
        return account_id
    except botocore.exceptions.ClientError as e:
        logger.error(f"Failed to fetch AWS Account ID: {e}")
        raise RuntimeError("AWS Account IDの取得に失敗しました。") from e


def main() -> None:
    """
    メイン関数。
    Webhook URLの取得・バリデーションは get_config() 内で完結しているため、
    main() では設定取得後すぐに処理を開始できる。
    """
    # get_config() で Webhook URL の取得・検証まで済ませている
    config = get_config()
    use_teams_post = config["USE_TEAMS_POST"]
    teams_webhook_url = config["TEAMS_WEBHOOK_URL"]

    # AWSアカウントIDを取得
    account_id = get_account_id()
    logger.info(f"AWS Account ID: {account_id}")

    # boto3 CostExplorer クライアントをモック化できるよう必ず get_client() 経由にする
    client = get_client()
    explorer = CostExplorer(client)

    start_date, end_date = get_date_range()
    period = {"Start": start_date, "End": end_date}
    start_day_str = datetime.strptime(start_date, "%Y-%m-%d").strftime("%m/%d")
    # Cost Explorer API の End は「その日の 0:00」を指すため、
    # 表示上は1日引いて「昨日まで（= 実質の集計最終日）」を表示する
    end_day_str = (datetime.strptime(end_date, "%Y-%m-%d") - timedelta(days=1)).strftime("%m/%d")

    # --- クレジット適用後 ---
    title_after, services_after = handle_cost_report(
        explorer, period, include_credit=True, start_day=start_day_str, end_day=end_day_str
    )
    title_after = f"AWSアカウント {account_id}\n" + title_after
    print_report(title_after, services_after)
    if use_teams_post:
        # Teams 通知が失敗しても Lambda 全体はエラーにしない（ログに警告）
        if not post_to_teams(title_after, services_after, webhook_url=teams_webhook_url):
            logger.warning("クレジット適用後レポートの Teams 通知に失敗しました。処理を継続します。")

    # --- クレジット適用前 ---
    title_before, services_before = handle_cost_report(
        explorer, period, include_credit=False, start_day=start_day_str, end_day=end_day_str
    )
    title_before = f"AWSアカウント {account_id}\n" + title_before
    print_report(title_before, services_before)
    if use_teams_post:
        if not post_to_teams(title_before, services_before, webhook_url=teams_webhook_url):
            logger.warning("クレジット適用前レポートの Teams 通知に失敗しました。処理を継続します。")


def lambda_handler(event: dict, context: Any) -> dict:
    # API Gateway 等と整合しやすいよう statusCode / body を返す
    try:
        main()
        return {"statusCode": 200, "body": "Cost report generated successfully."}
    except Exception as e:
        logger.exception("Unexpected error occurred during execution.")
        return {"statusCode": 500, "body": str(e)}


if __name__ == "__main__":
    main()
```
</details>

### コードの処理概要
主な処理フローは次の通りです。
- 設定値の取得と初期化
   - `get_config()` で環境変数や Teams 向けの設定を一元的に取得します。Teams 通知を有効にしたときは、ローカルでは `TEAMS_WEBHOOK_URL`、Lambda では `TEAMS_SECRET_ARN` 経由で Secrets Manager から Webhook URL を取る処理もここにまとめています。
- AWS アカウント ID の付与
   - `main()` 内の `get_account_id()`（STS の `get_caller_identity`）でアカウント ID を取得し、レポートのタイトル行に付けます。複数アカウント運用時にもどの環境か判別しやすくなります。
- Cost Explorer クライアントのラップ
   - `CostExplorer` クラスで boto3 の Cost Explorer API をラップし、API呼び出しやクレジット適用/除外のフィルタ処理など、共通化が必要なロジックを一箇所に集約しています。
- コストレポートの生成
   - 指定された期間（通常は当月の初日～前日）について、AWS サービス毎の利用料金サマリを生成。クレジット適用後・適用前という2パターンの料金をまとめてレポートします。
- Teams 通知機能
   - レポートは標準出力に出すだけでなく、必要に応じて Microsoft Teams にも送信可能です。Teams 通知は `requests` ライブラリで Webhook 経由で送信。まず Adaptive Card 形式で送信し、うまく表示できなければシンプルなテキストでも送る二重化設計になっています（`TEAMS_WEBHOOK_FORMAT=text` で明示切替も可）。なお、Teams Webhook は Power Automate Workflow などを使った URL に置き換える必要があり、旧 Office 365 Connector 専用のものは使えません。

## 3.3 ローカルでの動作確認

認証を通したあと、`uv run` で実行します。

```bash
# SSO ログイン
aws sso login --profile <your-sso-profile>

# 出力サンプル
# Attempting to automatically open the SSO authorization page in your default browser.
# If the browser does not open or you wish to use a different device to authorize this request, open the following URL:
# 
# https://example-sso.awsapps.com/start/#/device
# 
# Then enter the code:
# 
# MQXV-VQHQ
# Successfully logged into Start URL: https://example-sso.awsapps.com/start/

export AWS_PROFILE=<your-sso-profile>
export AWS_DEFAULT_REGION=ap-northeast-1
export AWS_PAGER=
```
`AWS_PROFILE` は `uv run` 自体の必須要件ではありませんが、ローカル実行時に `boto3` がどの認証情報を使うかを明示するため、`default` 以外のプロファイルを使う場合は設定しておくのを推奨します。同じコマンド行で `AWS_PROFILE=your-profile uv run python sam/app/app.py` のように書けば、その1回の実行には環境変数が渡ります。対話シェルで `AWS_PROFILE=...` と代入したあと、別のコマンドとして `uv run` する場合は子プロセスへ引き継がれないことがあるため、`export AWS_PROFILE=...` を使うか、実行のたびにコマンド先頭へ付けてください。

```bash
# 環境変数の設定（Teams 通知を使う場合）
export USE_TEAMS_POST=yes
export TEAMS_WEBHOOK_URL='https://example.com/your-teams-webhook-url'
```

実行コマンドは次のとおりです。

```bash
uv run python sam/app/app.py
```

実行すると、標準出力に以下のようなレポートが出ます。

```text
INFO:__main__:AWS Account ID: 123456789012
INFO:__main__:Calculated total cost from Groups: 0.00 USD
------------------------------------------------------
AWSアカウント 123456789012
01/01～01/30のクレジット適用後費用は、0.00 USD です。
サービスごとの費用データはありません。
------------------------------------------------------

INFO:__main__:Calculated total cost from Groups: 9.99 USD
------------------------------------------------------
AWSアカウント 123456789012
01/01～01/30のクレジット適用前費用は、9.99 USD です。
- AWS Config: 1.00 USD
- AWS Cost Explorer: 2.00 USD
- AWS Secrets Manager: 0.50 USD
- Amazon EC2 Container Registry (ECR): 0.75 USD
- Amazon GuardDuty: 1.25 USD
- Amazon Simple Storage Service: 4.49 USD
------------------------------------------------------
```

# 4. Microsoft Teams 通知の移行

## 4.1 Office 365 コネクタの廃止（重要）

ここが一番のハマりどころです。Qiita のコードコメントにあった `webhook.office.com` 形式の Office 365 コネクタ は、すでに廃止されています。旧 URL はそのままでは使えません。今後は Power Automate などの ワークフロー を経由した Webhook に差し替える必要があります。

## 4.2 ワークフローの作成

Power Automate で「Teams webhook 要求が受信されたとき」をトリガーにするフローを作成し、発行された URL を使います。

リポジトリのコードは Adaptive Card 形式を送りますが、表示が崩れる場合は `TEAMS_WEBHOOK_FORMAT=text` を設定すれば、シンプルなテキスト形式に切り替えられるようにしてあります。

# 5. AWS SAM による Lambda デプロイ
[![img3](https://imgur.com/ZCfzpPX.png)](https://imgur.com/ZCfzpPX.png)

## 5.1 SAM テンプレート
Qiita では `TeamsWebhookUrl` をパラメータで渡し、環境変数にそのまま載せる形でした。2026年版では Secrets Manager を前提にし、Webhook URL をテンプレートやコマンドラインに残しにくくしています。

その他の変更点をざっと挙げると、ランタイムは `python3.12` / `arm64`（Graviton2）、スケジュールは `cron(0 0 * * ? *)` で毎日 JST 9:00 実行です。加えて DLQ（SQS）と CloudWatch Alarm を組み合わせ、失敗時にイベントを取りこぼさないようにしてあります。個人アカウントでここまで必要か、という気もしますが、Lambda が壊れていても気づかず通知が途絶え続けるのは嫌なので入れておきました。

## 5.2 デプロイ手順
Qiita では `sam package` で `packaged.yaml` を生成してから `sam deploy` していました。2026年版の手順では、`sam build` のあと `sam deploy` を直接呼ぶ形に寄せています（プロジェクトや CI の方針に合わせて、従来型の `package` フローを取ることも可能です）。

まず Teams 通知用の URL を Secrets Manager に登録します。

```bash
aws secretsmanager create-secret \
  --name teams-webhook-url \
  --secret-string "https://example.com/your-teams-webhook-url"
```

あとはビルドしてデプロイします。

```bash
cd sam
sam build
sam deploy --parameter-overrides \
    UseTeamsPost=yes \
    TeamsWebhookSecretArn=arn:aws:secretsmanager:...
```

スタックを片付けるときは、Qiita 版と同様に `sam delete` で CloudFormation スタックと関連リソースをまとめて削除できます（リージョンやスタック名は環境に合わせてください）。

# 6. まとめ

Qiita の後編で扱った「boto3 で Cost Explorer を叩き、SAM で Lambda に載せ、Teams に飛ばす」という骨格はそのままです。変わったのは主に次の4点です。まず Python が 3.8 から 3.12 へ、パッケージ管理が Miniconda から uv に変わりました。次に Teams 通知は、廃止された Office 365 コネクタから Power Automate のワークフロー Webhook に置き換えています。Webhook URL の扱いも見直しており、SAM テンプレートに平文で残す代わりに Secrets Manager で管理する形にしました。SAM のデプロイ手順は `sam build` → `sam deploy` に整理し、Lambda は arm64 に変更、DLQ と CloudWatch Alarm も加えています。

Qiita のまとめ（前編〜後編通して）で触れたように、マネコンの UI は変わりやすい一方、CLI や API は比較的ゆっくり移行します。本記事の更新も、その延長線上で「同じことを 2026年の前提でやり直す」イメージです。

毎日自動で費用サマリが届くようになると、開発の心理的安全性がだいぶ上がる、というのが筆者の実感です。細かい実装やテストコードは [GitHub](https://github.com/t-tkm/aws-cost-explore-lambda) を参照してください。
