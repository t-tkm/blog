+++ 
Categories = ["AWS"] 
Tags = ["AWS","ECS","Fargate","Sesar2","legacy"] 
date = "2025-03-30T17:00:00+09:00" 
title = "AP-DB分離パターンにおけるレイテンシー影響確認" 
archives = ["2025", "2025-03", "2025-03-30"] 
+++

# 1. 概要
オンプレミスで稼働しているレガシーアプリケーション（※）をクラウドに移行する方法として、「Replatform to Containers」というアプローチがあります。

※本記事では、かつて主流だったSeasar2ベースのアプリケーションをレガシーアプリケーションの例として取り上げています。

この手法では、アプリケーションコードの修正を最小限にとどめてコンテナ化し、AWS Fargateなどのマネージドサービス上で実行することで、仮想マシン（VM）の保守運用から解放されるという利点があります。

ただし、データベース（DB）も同時にクラウドへ移行できれば理想的ですが、以下のような理由からオンプレミスに残すケースも少なくありません。

- 段階的な移行フェーズである
- データ特性や規制による制約

こうした状況では、アプリケーションとDBの物理的距離が広がることで、ネットワークレイテンシーによる性能劣化が懸念されます。

アプリケーションとDBが離れている場合の性能影響は、アプリケーションコードのDBアクセス部分を精査することである程度予測可能です。しかし、アプリケーションが複雑である場合には、すべての挙動を事前に把握するのは困難です。

そこで、本記事では、実際の構成で性能を測定し、レイテンシーの影響を可視化するアプローチをご紹介します。
特にクラウド環境では、こうした事前検証が比較的容易に実施できます。

以前筆者が相談を受けた案件において、手元の環境で実施した検証をベースに、AP-DB間の距離が性能に
与える影響を検証する方法を紹介します。

# 2. システム構成

本記事で使用するシステム構成を紹介します。オンプレミス環境の代替として自宅ネットワークを活用し、
自宅とAWS間の接続には **AWS Site-to-Site VPN** を使用しています。構築方法に興味がある方は、以下の記事もご覧ください。

[お手軽にAWS Site-to-Site VPNを試してみよう（自宅VPN環境）](https://t-tkm.github.io/blog/posts/2024/03/aws_vpn2/)

アプリケーションは、以下の3つの環境にデプロイします：

- オンプレミス（＝自宅）環境
- AWS 東京リージョン
- AWS 大阪リージョン

データベースはオンプレミス環境（自宅）で稼働させたままにし、各環境にデプロイしたアプリケーションから
DBへアクセスすることで、**AP-DB間の距離が性能に与える影響**を検証します。

{{< figure alt="img3" src="https://github.com/t-tkm/blog_images/raw/main/2025/03/aws_sesar2_vpn/img1.png" link="https://github.com/t-tkm/blog_images/raw/main/2025/03/aws_sesar2_vpn/img1.png">}}

# 3. 準備

## 3.1 ローカル環境での動作確認

この構成では、オンプレミス環境を模したEC2インスタンス上で、Seasar2ベースのレガシーJava WebアプリケーションをDockerコンテナで実行しています。
{{< figure alt="img3" src="https://github.com/t-tkm/blog_images/raw/main/2025/03/aws_sesar2_vpn/img2.png" link="https://github.com/t-tkm/blog_images/raw/main/2025/03/aws_sesar2_vpn/img2.png">}}

### (a) プロジェクト構成

- AWS Cloud9 を開発環境として使用
- Cloud9上で Docker Compose を用いて、アプリケーションコンテナ（Tomcat + Seasar2 アプリ）と PostgreSQL コンテナを起動
- アプリケーションは 8080 番ポートでアクセス可能
- DB は 5432 番ポートでリッスン
- PostgreSQL の JDBC URL は `10.0.0.254` を指し、オンプレミス内ネットワークを想定

### (b) 各構成要素の解説

- **Cloud9 / EC2**

  - Cloud9 IDEを使用し、EC2インスタンス内で開発・ビルドを実施
  - EC2のOSはAmazon Linux 2、プライベートIPは `10.0.0.254`

- **アプリケーション構成（右側のコンテナ群）**

  - **アプリコンテナ**
    - ベースイメージは `tomcat:9`
    - WARファイル（`sastruts-example-0.0.1.war`）を `webapps/` に配置
    - ポート8080で公開し、Seasar2 + Tomcat + Java 環境を構成
  - **DBコンテナ**
    - PostgreSQLを使用し、ポート5432でリッスン

### (c) 開発・デプロイ手順

- **ディレクトリ構成例**:

```bash
handson/
├── docker/
│   ├── docker-compose.yml
│   └── Dockerfile
├── postgres/
│   ├── docker-compose.yml
│   └── Dockerfile
├── pom.xml
├── src/
│   ├── main/java
│   ├── resources/jdbc.dicon
│   └── jdbc.properties
└── target/
    └── sastruts-example-0.0.1.war
```

- **アプリのビルドコマンド**:

```bash
rm -rf target && \
mvn package -DskipTests=true && \
cp ./target/sastruts-example-0.0.1.war ./docker
```

- **Dockerコンテナ起動**:

```bash
docker compose build && docker compose up
```

## 3.2 DBをAWSマネージドサービス（Aurora）へ変更

この構成では、ローカルで稼働していた PostgreSQL を AWS のマネージドサービスである **Amazon Aurora（PostgreSQL）** に置き換えました。
アプリケーションはそのままに、DBのみをクラウド上のAuroraに変更します。これにより、マネージドサービスの可用性・スケーラビリティの恩恵を得ながら、AP-DB間の距離による影響を測定できます。

{{< figure alt="img3" src="https://github.com/t-tkm/blog_images/raw/main/2025/03/aws_sesar2_vpn/img3.png" link="https://github.com/t-tkm/blog_images/raw/main/2025/03/aws_sesar2_vpn/img3.png">}}

ネットワーク設計（VPCサブネット、セキュリティグループ、ルーティングなど）により、AP-DB間の通信がクラウド内で完結するようになります。これにより、クラウド環境におけるネットワークレイテンシーや構成の検証が可能になります。

## 3.3 アプリケーションをECS Fargate上にデプロイ

次に、アプリケーション実行環境を **ECS Fargate** に移行します。従来EC2上で動作していたDockerコンテナを、Fargateタスクとして定義し、サーバレスで運用します。

{{< figure alt="img4" src="https://github.com/t-tkm/blog_images/raw/main/2025/03/aws_sesar2_vpn/img4.png" link="https://github.com/t-tkm/blog_images/raw/main/2025/03/aws_sesar2_vpn/img4.png">}}

この構成では、アプリケーション側のインフラ管理負荷が大幅に軽減され、将来的なスケーリングやデプロイの自動化も容易になります。なお、ECSサービスを使うことで高可用性を維持しながら運用が可能です。

## 3.4 DBをオンプレミスサービス（PostgreSQL）へ戻す

ここでは再び構成を変更し、データベースをオンプレミスに戻します。つまり、アプリケーションはAWS（Fargate）上で動作しつつ、DBはオンプレミスにあるという、**典型的なハイブリッド構成**になります。

### (i) ネットワーク接続の確認
まず、アプリケーションが動作する AWS VPC とオンプレミスネットワーク間でVPN接続が確立されていることが前提となります。
以下の構成要素を確認してください：

- Site-to-Site VPN接続が確立していること
- AWS VPC のルートテーブルにオンプレミス向けCIDRのルートがあること
- オンプレミス側のルータに AWS CIDR 向けのルートがあること
- 双方のファイアウォールおよびセキュリティグループで TCP:5432 を許可

### (ii) サブネットとセキュリティグループ設定
- Fargateタスクは、オンプレミスと接続可能な**プライベートサブネット**に配置します（パブリックサブネットは使用しません）
- タスクに付与するセキュリティグループでは、TCP:5432 のアウトバウンドを許可
- オンプレミス側DBのインバウンド設定でも、Fargate 側 CIDR/IP の許可が必要です

### (iii) JDBC接続設定の変更
アプリケーションの接続設定（例：`jdbc.properties`）を以下のように変更します：

```properties
jdbc.driver=org.postgresql.Driver
jdbc.user=postgres
jdbc.password=postgres
jdbc.url=jdbc:postgresql://10.0.0.254:5432/demo
```

### (iv) ECRへのイメージPushとタスク定義の更新
```bash
docker build -t demoimage .
docker tag demoimage:latest <AWS_ACCOUNT_ID>.dkr.ecr.ap-northeast-1.amazonaws.com/demoimage:latest
docker push <AWS_ACCOUNT_ID>.dkr.ecr.ap-northeast-1.amazonaws.com/demoimage:latest
```

ECSタスク定義では、上記イメージ名に更新し、環境変数でDB接続先を設定できるようにします。

### (v) 動作確認
ALB無し構成で、Fargateタスクに直接アクセスして動作確認を行います。
JSP画面に `startTime`, `endTime`, `execTime` が表示され、DBアクセスが正常に行えていれば完了です。

## 3.5 性能測定コードの追加
Seasar2アプリケーションに、DBアクセス処理の前後にタイムスタンプを記録するコードを追加し、実行時間をJSP上に表示する仕組みを導入します。

{{< figure alt="img5" src="https://github.com/t-tkm/blog_images/raw/main/2025/03/aws_sesar2_vpn/img5.png" link="https://github.com/t-tkm/blog_images/raw/main/2025/03/aws_sesar2_vpn/img5.png">}}

この改修により、既存のアプリケーション構造を大きく変更することなく、レイテンシーやDB処理時間を可視化できます。測定結果をもとに、ネットワーク設計やリージョン配置の検討材料とすることができます。

## 4. ECSによるコンテナ制御
ここでは、3章で準備した構成をもとに、実際にECSを使ってコンテナを起動する様子を紹介します。

以下の図は、東京リージョンと大阪リージョンにそれぞれ構築したECSクラスターの例です。各クラスターは、Fargate上のタスクを含む3つのデータプレーン（実行環境）を制御しています。

{{< figure alt="img10" src="https://github.com/t-tkm/blog_images/raw/main/2025/03/aws_sesar2_vpn/img10.png" link="https://github.com/t-tkm/blog_images/raw/main/2025/03/aws_sesar2_vpn/img10.png">}}

- **ECSクラスター**：ECSサービスがタスク（コンテナ）をスケジューリング・管理する単位。東京・大阪それぞれに存在。
- **データプレーン**：コンテナが実行される実環境（Fargate、EC2、ECS Anywhere など）
- **ECS Anywhere** を使用すると、オンプレミスのVMもECSクラスターに組み込むことが可能です。

## 5. 測定結果
ここでは、3つの構成パターンにおけるDBアクセス処理時間の測定結果を紹介します。アプリケーションとDBの距離やネットワーク構成の違いによる性能差を視覚的に比較できます。

### (1) オンプレミス同居構成（ローカルLAN）
{{< figure alt="img7" src="https://github.com/t-tkm/blog_images/raw/main/2025/03/aws_sesar2_vpn/img7.png" link="https://github.com/t-tkm/blog_images/raw/main/2025/03/aws_sesar2_vpn/img7.png">}}
- **URL**：`test2.localdomain.internal:8080`
- **構成**：アプリとDBが同一ホスト、または同一LAN内
- **処理時間**：**1ミリ秒**
- ローカル環境での最速ケースで、ネットワーク遅延はほぼゼロです。

### (2) 横浜–東京間（リージョン内）
{{< figure alt="img8" src="https://github.com/t-tkm/blog_images/raw/main/2025/03/aws_sesar2_vpn/img8.png" link="https://github.com/t-tkm/blog_images/raw/main/2025/03/aws_sesar2_vpn/img8.png">}}
- **URL**：`10.0.1.51:8080`
- **構成**：アプリはオンプレミス、DBはAWS東京リージョン（Aurora）
- **処理時間**：**12ミリ秒**
- 関東圏内の距離であり、最適化されたネットワーク構成なら十分な性能が得られます。

### (3) 横浜–大阪間（遠距離リージョン）
{{< figure alt="img9" src="https://github.com/t-tkm/blog_images/raw/main/2025/03/aws_sesar2_vpn/img9.png" link="https://github.com/t-tkm/blog_images/raw/main/2025/03/aws_sesar2_vpn/img9.png">}}
- **URL**：`10.1.2.111:8080`
- **構成**：アプリはオンプレミス、DBはAWS大阪リージョン（Aurora）
- **処理時間**：**22ミリ秒**
- 地理的距離が長く、ネットワーク遅延が明確に影響を与えています。

### 性能比較表
| 構成                            | 通信経路                  | 処理時間 | 備考                       |
|-------------------------------|---------------------------|----------|----------------------------|
| (1) ローカルLAN               | 同一ネットワーク内        | 1ms      | 最速、レイテンシー最小       |
| (2) 横浜–東京（近距離）       | リージョン間（近距離）    | 12ms     | 実用に耐える性能           |
| (3) 横浜–大阪（遠距離）       | リージョン間（長距離）    | 22ms     | 明確な性能劣化が見られる   |

## 6. まとめ
本記事では、オンプレミスで稼働していたレガシーな Seasar2 アプリケーションを最小限の改修でコンテナ化し、AWS上に再ホスト（Replatform）する過程で、AP-DB間の距離が性能に与える影響を検証しました。

### 検証ステップの振り返り
1. **オンプレミス構成をEC2上に再現**  
   - DockerでSeasar2 + PostgreSQL構成を構築
2. **DBをAuroraに変更**  
   - マネージドサービスを利用し、運用負荷を軽減
3. **アプリをFargate上にデプロイ**  
   - サーバレスな基盤への移行
4. **DBをオンプレミスに戻す構成でハイブリッド構成を模擬**  
   - VPNを介して遠隔DBと接続
5. **性能測定コードでレイテンシーを可視化**  
   - 実際の処理時間をJSP上で確認可能に

### 得られた示唆
- AP-DB間の物理距離が広がることで性能低下のリスクが高まる
- Replatform時にも、AP-DB構成は性能に大きく影響する
- 適切な構成設計と事前検証が極めて重要である

### 設計・運用のヒント
- **レイテンシー対策**
  - ElastiCache や RDS Proxy の活用
  - データのローカルレプリカ導入
  - DBアクセスの最適化（接続プール等）

- **Replatform移行時は事前検証を推奨**
  - 実構成に近い形での測定により、構成選定の判断材料を得る

クラウド移行は単なる「載せ替え」ではなく、設計・通信・運用すべての見直しを含む全体最適が求められます。構成別の性能を事前に「見える化」することで、より安全で納得感のあるクラウド活用が実現できます。

## 付録
- GitHubプロジェクト（Seasar2アプリ）
  - [Seasar2 app container on AWS](https://github.com/t-tkm/seasar2-app-container-on-AWS.git)

- 構成の代替案：AWS App2Container の紹介
  - 本記事ではCloud9上で手動コンテナ化しましたが、AWSの自動変換ツール「App2Container」も活用可能です。
  - [AWS App2Container（お試し編）](https://t-tkm.github.io/blog/posts/2022/03/aws_app2container/)

- Seasar2の代替：Springなど
  - Seasar2は現在開発・保守が停止しており、代替としてSpring Frameworkが広く使われています。
  - [cdk-microservices-labs](https://github.com/aws-samples/cdk-microservices-labs.git)
  - [spring-petclinic](https://github.com/spring-projects/spring-petclinic)
