+++ 
Categories = ["AWS"] 
Tags = ["AWS", "CUR", "Cost", "FinOps", "Athena"] 
date = "2023-09-24T00:00:00+09:00" 
title = "AWS Cost Usage Reportの可視化(2) -ヘッドレスBIツールCubeを試してみる" 
archives = ["2023", "2023-09", "2023-09-24"]
+++

# はじめに
「[AWS Cost Usage Reportの可視化](http://localhost:1313/blog/posts/2023/06/aws_cost_usage_report/)」の続編になります。

前回記事では、AWS費用のレポートがS3に定期的に溜め、S3データをSQLで検索できるマネージドサービスAthenaを構築し、AthenaをデータソースとしてGrafanaやRedashといったBIツールで可視化基盤を構築しました。

その後も、Apache Supersetや各種NotebooksといったBIツールで可視化していたものの、今度はReactやVueといった
フロントエンド開発を通し、カスタムダッシュボートを開発・構築したい、という思いに至りました。
そこで目にしたのが、記事の副表題にもあるヘッドレスBIツール[Cube Core](https://cube.dev/docs/product/getting-started/core)というOSSになります。

このOSSですが、[Cube](https://cube.dev/)という「セマンティックレイヤー」(ヘッドレスBI)」を実現するサービスの一部で、
SaaS版(Cube Cloud)も提供されています。「セマンティックレイヤー」がどのようなコンセプトかにつきましては、
Cubeのサイトにあるコンセプト画像がとてもわかりやすく、興味ある方はそちらもぜひご確認頂ければと思います。

(私自身)フロントエンド学習環境として、React、Angular、Vueや、REST API、GraphQLなど、バックエンド(APIサーバ)の準備を「面倒だなぁ」
と思う事もあり、このツールはまとめて学習できる多機能な点が気に入りました。

※ちなみに今回は、データソースとして前回構築したAthenaを使いましたが、cube公式サイトによる公開PostgreSQLもありますので、
手持ちのデータソースが無くても、すぐにお試しすることができます。

# 全体構成
今回Cube Coreを試した全体構成です。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img1.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img1.png">}}

以降では、Step1としてローカルPCにCube Coreを構築し、次に、Step2として同じツールをAWS上にデプロイしてみようと思います。

ご参考に、最終のプロジェクト構成はこのようになりました。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img12.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img12.png">}} 

# Step1: ローカル開発
ローカルPCにCube Coreを導入します。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img2.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img2.png">}}

こちらの[(公式)導入ドキュメント](https://cube.dev/docs/product/getting-started/core/create-a-project)に従います。

```zsh
% mkdir cubejs-trial
% cd cubejs-trial
% touch docker-compose.yml
```

docker-compose.ymlに、下記を記載。
```yaml
version: "2.2"
 
services:
  cube:
    image: cubejs/cube:latest
    ports:
      - 4000:4000
      - 15432:15432
    environment:
      - CUBEJS_DEV_MODE=true
    volumes:
      - .:/cube/conf
```

後はアプリ(コンテナ)を起動するのみ！
```zsh
% docker compose up -d
```

http://localhost:4000にアクセスすると、Databse connection画面が表示されますので、AWS Athenaを選択して、
必要な情報を入力します。

※公式による公開PostgreSQLを使う場合は、データソースの情報として↓が使えます。

https://cube.dev/docs/product/getting-started/core/create-a-project#connect-a-data-source

{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img4.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img4.png">}}

うまく接続できれば、プロジェクトフォルダに「.env」ファイルが作成されます。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img5.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img5.png">}}
```
CUBEJS_AWS_KEY=<AWSアクセスキー>
CUBEJS_AWS_SECRET=<AWSシークレットキー>
CUBEJS_AWS_REGION=ap-northeast-1
CUBEJS_AWS_S3_OUTPUT_LOCATION=s3://<バケット名>
CUBEJS_DB_TYPE=athena
CUBEJS_API_SECRET=<ランダム数字>
CUBEJS_EXTERNAL_DEFAULT=true
CUBEJS_SCHEDULED_REFRESH_DEFAULT=true
CUBEJS_DEV_MODE=true
CUBEJS_SCHEMA_PATH=model
```
Athenaから取得されたテーブルが表示されるので、テーブルを選択して「Generate Data Model」をクリック。
自動でモデルが作成されます。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img6.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img6.png">}}

あとは「Playground」を使うだけ。これは、他のBIツールと同じくピボットレーブルやダッシュボードを作れます。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img7.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img7.png">}}

{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img8.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img8.png">}}

このPlaygroundでは、可視化だけでなく、React、Angular、VueといったUIフレームワークの選択、データ可視化のJavaScriptライブラリであるChart.js、D3、バックエンドサーバにREST API(JSON)だけでなくGraphQLを使うなどなど、多機能を試すことが可能です。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img13.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img13.png">}}

更に、オンラインエディタ[CodeSandbox](https://codesandbox.io/)で編集させてみることもできます！(学習者には至れり尽くせりといった印象です)
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img9.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img9.png">}}

# Step2: AWSデプロイ
AWSへのデプロイには、CDKを使いました。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img3.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img3.png">}} 

プロジェクトフォルダに「cdk」というサブフォルダを作成し、CDKプロジェクトを構成します。

```zsh
% mkdir cdk
% cd cdk 
% npx cdk init sample-app --language typescript
```

Athena接続設定として、下記ドキュメントに従います。

[Documentation / Configuration / Connecting to data sources / AWS Athena](
https://cube.dev/docs/product/configuration/data-sources/aws-athena)

```zsh
% touch .env
```
「.env」は次のようにしました。
```
CUBEJS_DEV_MODE=true
CUBEJS_DB_TYPE=athena
CUBEJS_AWS_KEY=<AWSアクセスキー>
CUBEJS_AWS_SECRET=<AWSシークレットキー>
CUBEJS_AWS_REGION=ap-northeast-1
CUBEJS_AWS_S3_OUTPUT_LOCATION=s3://<S3バケット名>
CUBEJS_AWS_ATHENA_WORKGROUP=primary
CUBEJS_AWS_ATHENA_CATALOG=AwsDataCatalog
```

```zsh
# .envをgit管理しないように注意
% echo .env >> .gitignore
# AWS認証情報など環境変数を.envファイルでCDKスタックへ渡すためのツール
% npm i -D dotenv
```

CDKの本体、lib/cdk-stack.tsを次のようにします:
- 8-9: .envにある環境変数を取込むためのツール設定
- 15, 60-64: インターネット経由でAWSロードバランサALBへアクセスする際、自分のIPからのみ接続許可

```typescript {linenos=true, hl_lines=["8-9", 15, "60-64"]}
import { Duration, Stack, StackProps } from 'aws-cdk-lib';
import ec2 = require('aws-cdk-lib/aws-ec2');
import ecs = require('aws-cdk-lib/aws-ecs');
import ecs_patterns = require('aws-cdk-lib/aws-ecs-patterns');
import { SecurityGroup } from 'aws-cdk-lib/aws-ec2';
import { ApplicationLoadBalancer } from 'aws-cdk-lib/aws-elasticloadbalancingv2';
import { Construct } from 'constructs';
import * as dotenv from 'dotenv';
dotenv.config(); // .envファイルから環境変数を読み込む

export class CdkStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const MY_IP: string = "xxx.xxx.xxx.xxx/32"

    // 1. 仮想プライベートクラウド（VPC）を作成
    const vpc = new ec2.Vpc(this, 'MyVpc', { maxAzs: 2 });

    // 2. Amazon ECSクラスタを作成
    const cluster = new ecs.Cluster(this, 'MyCluster', {
      vpc,
    });
    
    // 3. Amazon ECS Fargateタスク定義を作成
    const task = new ecs.FargateTaskDefinition(this, 'MyTask', {
      memoryLimitMiB: 4096,
      cpu: 2048
    });

    // 4. コンテナイメージを指定してコンテナをタスクに追加
    const container = task.addContainer('MyContainer', {
      image: ecs.ContainerImage.fromRegistry('cubejs/cube:latest'),
      environment: {
        // 5. 環境変数の設定（デフォルト値または環境変数から読み込む）
        CUBEJS_DEV_MODE: process.env.CUBEJS_DEV_MODE ?? "",
        CUBEJS_DB_TYPE: process.env.CUBEJS_DB_TYPE ?? "",
        CUBEJS_AWS_KEY: process.env.CUBEJS_AWS_KEY ?? "",
        CUBEJS_AWS_SECRET: process.env.CUBEJS_AWS_SECRET ?? "",
        CUBEJS_AWS_REGION: process.env.CUBEJS_AWS_REGION ?? "",
        CUBEJS_AWS_S3_OUTPUT_LOCATION: process.env.CUBEJS_AWS_S3_OUTPUT_LOCATION ?? "",
        CUBEJS_AWS_ATHENA_WORKGROUP: process.env.CUBEJS_AWS_ATHENA_WORKGROUP ?? "",
        CUBEJS_AWS_ATHENA_CATALOG: process.env.CUBEJS_AWS_ATHENA_CATALOG ?? ""
      }
    });

    container.addPortMappings({
      containerPort: 4000,
    });
    container.addPortMappings({
      containerPort: 15432,
    });

    // 6. アプリケーションロードバランサー(ALB)のセキュリティグループを作成
    const alb_securityGroup = new SecurityGroup(this, "MySG", {
      vpc: vpc,
      allowAllOutbound: true,
    });

    // 7. ALBへのアクセスを指定したIPアドレスに制限
    alb_securityGroup.addIngressRule(
      ec2.Peer.ipv4(MY_IP),
      ec2.Port.tcp(80),
    );

    // 8. アプリケーションロードバランサー(ALB)を作成
    const alb = new ApplicationLoadBalancer(this,'MyALB',{
      internetFacing: true,
      loadBalancerName: 'my-alb',
      securityGroup: alb_securityGroup,
      vpc
    })

    // 9. アプリケーションロードバランサー(ALB)を使用するFargateサービスを作成
    const loadBalancedFargateService = new ecs_patterns.ApplicationLoadBalancedFargateService(this, 'MyService', {
      cluster,
      taskDefinition: task,
      desiredCount: 1,
      loadBalancer: alb,
      openListener: false
    });
  }
}
```

準備ができたらデプロイ。(AWSアカウントへの認証設定は省いています)
```zsh
% npx cdk deploy
```

実行結果。筆者の環境では、5分程度で完了しました。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img11.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img11.png">}}

今回は、Athenaへの接続は完了しているので、Data Modelタブから「Generate Data Model」でモデルを生成。
無事確認できました。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img10.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img10.png">}}

※AWSデプロイ版では、(恐らくセキュリティ周りの設定かと思いますが)Edit(Code Sandbox連携)での画面表示がErrorとなってしまったので、
おいおい確認してみたいと思います。

作業が完了したら、リソースを削除します。
```zsh
% npx cdk destroy
```

# まとめ
AWS Athena(AWS費用レポートテーブル)をデータソースとして、ヘッドレスBIツールCube Coreを試してみました。
手元ですぐに試してみたいといったニーズでは、シングルコンテナ構成という事もあり、煩わしい依存関係に悩む事なく、
簡単に手元の開発PC、AWSのコンテナ基盤で動かすことができました。

冒頭にある「セマンティックレイヤー」コンセプト理解や、フロントエンド開発の流れを学習するために、もう少し
遊んでみようと思います。

# 補足
今回は、手軽にという事で諸所省略した部分があります。「ちょっとお試ししてみる」以上に使いこなす場合は、公式ドキュメント
や、AWSベスプラに従った設計が必要です。
- 環境変数は、パラメータストアやSecret Managerに格納するのがベスプラです。
  (特に、今回の方法ではAWSアクセスキーや、シークレットキーがタスク(定義)で見えてしまうのでご注意！)
- ローカル開発では、アプリデータはプロジェクトディレクトリに永続化されていますが、AWSデプロイ版ではエフェメラルな
  一時ディスクへの永続化になります。コンテナ再デプロイ時にデータが失われますので、必要に応じEFSをマウントするなど
  で対応します。
  - ```yaml
      (snip)
      volumes:
      - .:/cube/conf
    ```
- 今回はお試し環境ということでシングルコンテナでしたが(環境変数「CUBEJS_DEV_MODE=true」を利用)、公式ドキュメントには、よりスケール
するような構成も準備されています。
  - [Documentation / Deployment / Cube Core](https://cube.dev/docs/product/deployment/core)

# 付録
環境変数についてはChatGPTさんも指摘してくれました。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img15.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img15.png">}}

元のコードでは、コメントも書いていなかったのですが、その部分はChatGPTさんへお願いして修正しました。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img14.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report2/img14.png">}}
