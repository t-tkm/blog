+++
Categories = ["AWS"]
Tags = ["AWS","Vibe","Coding","開発"]
date = "2025-06-30T12:00:00+09:00"
title = "Vibe CodingでDify環境構築"
archives = ["2025", "2025-06", "2025-06-30"]
+++

# 1. 概要
最近、Difyが注目を集めており、クラウドで簡単に試せる環境構築の相談を受ける機会が増えています。つい先日、Azure版でDify環境構築の案件があったのですが、せっかくの機会なのでAWSでも同様の環境を準備してみようと考えました。

しかし、私自身は日常業務において、IaCコードを積極的に記述する機会が限られており、ゼロベースで本格的なインフラ構築を行うとなると、相応の学習時間と開発期間を要すると想定していました。

そこで、昨今話題となっているVibe Codingアプローチを実際に検証してみることにしました。結果的に、当初の想定を大きく上回る効果を得ることができ、短期間でDify環境の構築を完了することができました。

本記事では、その実践過程と得られた知見を共有いたします。

今回Vibe Codingで作成・修正したプロジェクトはこちらです。
<a href="https://github.com/t-tkm/VibeCodingSample-AWS-CKD.git" target="_blank" rel="noopener">
    <img src="https://gh-card.dev/repos/t-tkm/VibeCodingSample-AWS-CKD.svg">
</a>

## 1.1 Vibe Codingとは
Vibe Codingは、自然言語による要件定義とAIとの対話的開発を組み合わせた次世代型の開発手法です。従来の開発プロセスでは、詳細な技術仕様書作成から実装まで、段階的な工程を経る必要がありましたが、本手法では以下の特徴により開発効率を革新します：

- **自然言語ベースの要件定義**: 複雑なシステム要件を直感的に表現
- **反復的品質向上プロセス**: AIとの対話による段階的な最適化
- **自動ベストプラクティス適用**: 業界標準と推奨アーキテクチャパターンの自動適用

この手法は、特に概念検証フェーズや迅速なプロトタイピングにおいて、従来比で大幅な開発効率向上を実現します。

## 1.2 Difyとは何か
[Dify](https://dify.ai/jp)は、ノーコード/ローコードでAIアプリケーションを構築できるオープンソースプラットフォームです。

**主な特徴**
- Docker Composeベースの簡単なデプロイメント
- 複数のLLMプロバイダーサポート（OpenAI、Anthropic、Azure OpenAI等）
- ワークフロー機能による複雑なAI処理の構築
- REST APIによる外部システム連携

Difyは、API、Worker、Web、PostgreSQL、Redis、Nginx、Weaviateなどのコンポーネントから構成されています。詳細な技術仕様については、[公式ドキュメント](https://docs.dify.ai/)をご参照ください。

## 1.3 本プロジェクトで構築した構成
{{< figure alt="img7" src="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img1.png" link="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img1.png">}}

# 2. Vibe CodingによるIaCコード生成
既存のAzure版の構成を参考として、今回はアーキテクチャ図を基に、要件テキスト（プロンプト）を用いて、AWS CDKのコード、およびプロジェクト全体をAI（CursorのClaude4 Agent）によって構築しました。

## 2.1 プロンプト
複数回の検証と調査を重ね、Difyの内部アーキテクチャとAWS CDKのベストプラクティスを踏まえた、以下の要件定義プロンプトを策定いたしました。

#### 要件テキスト
```
この図を再現するCDK pythonのIaCコードを作成してください。

# 要件
- 図はAzureですが、AWSで同様な構成を期待します。Azure Bastionの代わりに、AWS Fleet Managerを使います。
- IaCコードは、保守しやすいようにベストプラクティスに従って、適切にモジュール化してください。
- リソースは、複数の同様な環境を構築する際に重複しないように、適切にリソースグループ名や、タグを設計してください。
- 構築されるVMは、Windows VM、Linux VMの2台です。いずれもログイン認証はユーザー名とパスフレーズでログインできるようにしてください。
- Difyはシンプルな構成で「docker compose up -d」コマンドで起動しましょう（docker-composeコマンドは使わないでください）。例えば、このコンテナ程度でどうでしょうか？（api / worker / web / db / redis / nginx / weaviate）。ssl_proxyやcertbotは不要です。設定は、docker-composeのoverrideで上書きするようにしてください。nginxコンテナは、リバースプロキシとして、http(80)でアクセスするために使われます。ただし、インターネットからのアクセスは不要です（許可しないでください）。ローカルネットワークからのみアクセスできることを想定しています。
- difyは、/optへインストールします。docker composeのファイル群は、/opt/dify/dockerに格納します。「docker compose up -d」も/opt/dify/dockerで実行します。
- コメント、READMEは全て日本語で説明してください。
- gitプロジェクトです。適切な.gitignoreを作成してください。
- READMEには、ディレクトリツリーと、アーキテクチャ図を挿入してください。アーキテクチャ図は、mermaid記法で作成してください。
```

#### 参照アーキテクチャ
別途実装したAzure版Dify環境のアーキテクチャを参考基盤といたしました。
{{< figure alt="img7" src="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img4.png" link="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img4.png">}}

## 2.2 AIとの対話
CursorのAIチャットに、要件テキストと図をインプットしました：
{{< figure alt="img7" src="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img5.png" link="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img5.png">}}


# 3. 確認
AI生成されたCDKプロジェクトの実装確認を実施いたします。

## 3.1 CDKデプロイメント
**事前準備**
```bash
# リポジトリのクローン
git clone https://github.com/t-tkm/VibeCodingSample-AWS-CKD.git
cd VibeCodingSample-AWS-CKD

# 仮想環境の作成と有効化
python -m venv .venv
source .venv/bin/activate  # Linux/macOSの場合
# .venv\Scripts\activate.bat  # Windowsの場合

# 依存関係のインストール
pip install -r requirements.txt

# 環境変数の設定（必要に応じて）
cp .env.example .env
# .envファイルを適切に編集

# CDKのブートストラップ（初回のみ）
cdk bootstrap
```

**デプロイ実行**
```bash
# 構文チェック
cdk synth

# デプロイ
cdk deploy

# 確認
cdk list
```

**リソースの削除**
```bash
# 不要になった場合のクリーンアップ
cdk destroy
```

#### CDK実行画面（キャプチャ）

{{< figure alt="img11" src="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img11.png" link="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img11.png">}}

CDKデプロイメントは219秒で完了しましたが、Linux VM上でのDifyコンテナプロビジョニングには追加で数分を要します。


### 3.2 Fleet ManagerによるRDP接続
AWS Systems Manager Fleet Managerは、従来のSSHやRDPに代わる安全なリモート管理ソリューションです。

**主な利点**
- **セキュリティ強化**: インバウンドポートの開放が不要
- **監査機能**: すべてのセッションが記録される
- **統合管理**: AWSコンソールから直接操作可能
- **権限制御**: IAMポリシーによる細粒度な権限制御

**技術的仕組み**
1. EC2インスタンスにSSM Agentが導入
2. インスタンスからSSMサービスへのアウトバウンド接続
3. ユーザーはAWSコンソール経由でセッション開始
4. 暗号化されたトンネル経由での安全な通信

{{< figure alt="img12" src="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img12.png" link="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img12.png">}}
{{< figure alt="img13" src="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img13.png" link="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img13.png">}}
{{< figure alt="img14" src="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img14.png" link="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img14.png">}}

## 3.3 ローカルクライアントからの接続
AWS Fleet Managerを使用する場合、**日本語入力ができない**という制約があります。この制約により、日本語での作業が必要な場面では実用性に限界があるため、実際の運用においては、前述のローカルクライアントからのRDP接続（ポートフォワード方式）を主に活用しております。

```
リモートデスクトップは言語入力として英語のみをサポートしています。
```
https://docs.aws.amazon.com/ja_jp/systems-manager/latest/userguide/fleet-manager-remote-desktop-connections.html


ローカルPCからの接続には、AWS CLIのポートフォワード機能を使用します。

**接続手順**
```bash
# RDP接続用ポートフォワード（Windows VM）
aws ssm start-session \
    --target i-xxxxxxxxx \
    --document-name AWS-StartPortForwardingSession \
    --parameters "portNumber=3389,localPortNumber=13389"
```

ポートフォワードの実行状況（キャプチャ）
{{< figure alt="img8" src="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img8.png" link="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img8.png">}}

正常にローカル環境（PC）からの操作が可能になりました。
{{< figure alt="img9" src="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img9.png" link="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img9.png">}}

{{< figure alt="img10" src="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img10.png" link="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img10.png">}}



# 4. パフォーマンスとコスト考慮
## 4.1 推奨リソースサイズ
実際の運用を通じて、以下のリソース構成を推奨いたします：

**Windows VM（開発・テスト用）**
- インスタンスタイプ: t3.medium (2 vCPU, 4GB RAM)
- ストレージ: 30GB EBS gp3
- 月額概算: $30-40

**Linux VM（Dify本体）**
- インスタンスタイプ: t3.large (2 vCPU, 8GB RAM)
- ストレージ: 50GB EBS gp3
- 月額概算: $60-80

**その他のAWSサービス**
- VPC、サブネット: 無料
- NAT Gateway: $32/月
- Systems Manager: 無料（セッション料金のみ）

**総月額コスト: 約$120-150**

## 4.2 コスト最適化のポイント
**開発環境での節約施策**
```bash
# インスタンスの自動停止スケジューリング
# 平日夜間・休日の自動停止で約60%のコスト削減
aws ec2 create-tags --resources i-xxxxxxxxx \
    --tags Key=Schedule,Value=office-hours
```

**リソース監視**
- CloudWatch Dashboardでリソース使用率監視
- Cost Budgetアラートの設定
- Trusted Advisorによるコスト最適化提案の確認

# 5. まとめ
普段IaCコードを記述する機会が限られている状況において、従来の手法であれば相当な学習期間と開発時間を要したであろう開発も、
片手間に完了できたことは、Vibe Codingの効果を改めて思い知りました。

今回の件で、IaCコードの記述経験が限定的な技術者であっても、適切なプロンプトエンジニアリングとAIとの対話的開発により、
実運用レベルのインフラストラクチャを構築できる事がわかりました。

# 付録1：アーキテクチャ図生成プロセス
構成図作成においても、AIとの協働により効率化を図りました。

**AIへの指示**
{{< figure alt="img7" src="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img6.png" link="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img6.png">}}

**初期生成版**
{{< figure alt="img7" src="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img2.png" link="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img2.png">}}

全体構成は適切でしたが、一部に技術的な誤りを確認したため、修正を指示いたしました。
{{< figure alt="img7" src="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img7.png" link="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img7.png">}}

**修正完了版**
{{< figure alt="img7" src="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img3.png" link="https://github.com/t-tkm/blog_images/raw/main/2025/06/aws_vibe_coding/img3.png">}}

このプロセスにより、従来であれば数時間を要する作図作業を大幅に短縮することができました。


# 付録2：プロジェクト規模分析

**開発成果物**
- **コード規模**: Python 1,403行（Windows構築478行、Linux構築364行、セキュリティ180行）
- **ドキュメント**: README 368行、設定ファイル5個、アーキテクチャ図3個
- **開発期間**: 10日間（実作業5日間）
- **技術スタック**: AWS CDK(Python)、Docker Compose、Systems Manager

**主要マイルストーン**
1. プロジェクト基盤構築・初期コミット
2. 認証情報外部化・セキュリティ対応
3. IMDSv2対応・インフラ設定最適化
4. アーキテクチャ図生成・ドキュメント整備
5. 最終調整・品質確保

本プロジェクトは、Vibe Codingによる開発効率化の定量的な実証例として、中規模インフラプロジェクトの迅速な立ち上げを可能にしました。