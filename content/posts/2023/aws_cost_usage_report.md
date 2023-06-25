+++ 
Categories = ["AWS"] 
Tags = ["AWS", "CUR", "Cost", "FinOps", "Athena"] 
date = "2023-06-18T00:00:00+09:00" 
title = "AWS Cost Usage Reportの可視化" 
+++

# はじめに
本記事では、ローカルPC上でGrafana、Redashの可視化ツールを起動させ、AWSの費用レポートを
データソースに可視化(接続)する手順を確認します。

{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img1.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img1.png">}}

近年益々「オブザーバビリティ(Observability)」や「ビジネスインテリジェンス(BI)」といったデータ可視化が重要になっています。そのような中、データ活用の入口として、OSSに代表されるような多様な可視化ツールの味見やデモに
対するニーズも増えてきている気がします。
- Observability系: Nagios、Graphite、Grafana(Prometheus)、Kibana(ELK)、OpenTelemetry、OpenCensus、Jaeger etc.
- BI系: Redash、Apache Superset、Metabase etc.

AWSには「Amazon Managed Grafana/Amazon CloudWatch」や「Amazon QuickSight」とマネージドなサービスが揃っており、素直にこれらを活用するのが素直な選択ではあります。

一方、私のように「多様なツールを(隙間時間で？)手元でちょこっと味見してみたい」といったケースでは、ローカルPC上のコンテナを使うのが便利です(AMGやQuickSightをサインアップするのが煩わしいといった場合^^;;)。

以前は、ローカルPC上にGrafanaのコンテナを起動させ、ダミーシステム(or お宅システム)の監視を中心に
データ分析、ダッシュボード作成を試していました。ただシステムメトリクスのデータだけでは、
(ドリルダウンによる分析くらいなので)BI系ツールの味見がイマイチです。

そこで、システムメトリクスに加え、「クラウド費用データ」の活用に着目してみました。「費用データ」
はビジネスサイドの方にも関心を持っていただけるため、ビジネス面のメンバに対してのツールデモシナリオ
にも適しているかと思います。

# 構築
次の手順で進めます:
- ステップ(1): AWS Cost and Usate Reportの有効化
- ステップ(2): Amazon Athenaからレポート確認
- ステップ(3): 分析ツールからレポート確認
  
{{< figure alt="img2" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img2.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img2.png">}}

## ステップ(1): AWS Cost and Usate Reportの有効化
AWS費用データをS3へ貯める設定を行います。これには、AWS Cost and Usage Report(略してCUR)
を使います。デフォルトではCURは有効でないため、改めて設定する必要があります。

費用レポートは、「Billing」サービスの「Cost & udage reports」から設定します。
パラメータは以下のようにしました:
- レポート名: billing-report-20230608
- デフォルトの明細項目: 全てチェック
- 追加の明細項目: 「リソースIDのインクルード」をチェック
- データ更新の設定: 「自動的に更新」をチェック
- S3バケットの設定: t-tkm-billing-reports-20230608
- S3パスプレフィックス: billing
- レポートバージョニング: 既存レポートを上書き
- レポートデータの統合: Amazon Athena
- 圧縮タイプ: Parquet

※圧縮タイプは、Athena統合を選択すると自動的にParquetになります(これは、技術的な理由のようで、
将来改善される可能性は高いと思います)。
Parquet圧縮形式では、Amazon Redshift/Amazon QuickSightと連携できないため、その場合は、
別途レポートを準備しましょう。
{{< figure alt="img3" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img3.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img3.png">}}

設定後、24時間以内にレポートが生成されます。裏を返すと、1日待ちましょうという事で、直ぐにレポート
確認はできない点は注意が必要です。最終的に、指定したS3バケットの中身は、このような感じに
なります:

{{< figure alt="img4" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img4.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img4.png">}}

費用データを確認してみます。ただし、費用データはParquetで圧縮されているため、エクセルなどで
確認できるようにCSVなどに変換しました。

billing-report-20230608-00001.csv(元ファイルはbilling-report-20230608-00001.snappy.parquet)
が費用データ本体です。現地点で141列のカラムがあります(分析のやりがい有り！)

{{< figure alt="img5" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img5.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img5.png">}}

ParquetをCSVへ変換するpythonスクリプト:
```python3
import pyarrow.parquet as pq
import pandas as pd

INPUT_FILE_NAME = 'cost_and_usage_data_status.parquet'
OUTPUT_FILE_NAME = 'cost_and_usage_data_status.csv'

table = pq.read_table(INPUT_FILE_NAME)
df = table.to_pandas()
df.to_csv(OUTPUT_FILE_NAME, index=False)
```

## ステップ(2): Amazon Athenaからレポート確認
GrafanaやRedashから、直接S3バケットにある費用データ(parquet)はデータソースにできないため、
データソースに指定できるAthenaを(SQLエンジンとして)設定します。

請求書レポートの出力先として指定したS3バケットを確認します。その中に「crawler-cfn.yml」という
CloudFormationのテンプレートができていると思います。これをCloud Formationから読み込み
実行するだけ！(スタック名はなんでも良いです)

```yaml
AWSTemplateFormatVersion: 2010-09-09
Resources:

  AWSCURDatabase:
    Type: 'AWS::Glue::Database'
    Properties:
      DatabaseInput:
        Name: 'athenacurcfn_billing_report_20230608'
      CatalogId: !Ref AWS::AccountId
(snip)
```

<details>
  <summary>crawler-cfn.yml(展開)</summary>

```yaml
AWSTemplateFormatVersion: 2010-09-09
Resources:

  AWSCURDatabase:
    Type: 'AWS::Glue::Database'
    Properties:
      DatabaseInput:
        Name: 'athenacurcfn_billing_report_20230608'
      CatalogId: !Ref AWS::AccountId

         

  AWSCURCrawlerComponentFunction:
    Type: 'AWS::IAM::Role'
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - glue.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Path: /
      ManagedPolicyArns:
        - !Sub 'arn:${AWS::Partition}:iam::aws:policy/service-role/AWSGlueServiceRole'
      Policies:
        - PolicyName: AWSCURCrawlerComponentFunction
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - 'logs:CreateLogGroup'
                  - 'logs:CreateLogStream'
                  - 'logs:PutLogEvents'
                Resource: !Sub 'arn:${AWS::Partition}:logs:*:*:*'
              - Effect: Allow
                Action:
                  - 'glue:UpdateDatabase'
                  - 'glue:UpdatePartition'
                  - 'glue:CreateTable'
                  - 'glue:UpdateTable'
                  - 'glue:ImportCatalogToGlue'
                Resource: '*'
              - Effect: Allow
                Action:
                  - 's3:GetObject'
                  - 's3:PutObject'
                Resource: !Sub 'arn:${AWS::Partition}:s3:::t-tkm-billing-reports-20230608/billing/billing-report-20230608/billing-report-20230608*'
        - PolicyName: AWSCURKMSDecryption
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - 'kms:Decrypt'
                Resource: '*'
       

  AWSCURCrawlerLambdaExecutor:
    Type: 'AWS::IAM::Role'
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - lambda.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Path: /
      Policies:
        - PolicyName: AWSCURCrawlerLambdaExecutor
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - 'logs:CreateLogGroup'
                  - 'logs:CreateLogStream'
                  - 'logs:PutLogEvents'
                Resource: !Sub 'arn:${AWS::Partition}:logs:*:*:*'
              - Effect: Allow
                Action:
                  - 'glue:StartCrawler'
                Resource: '*'
       

  AWSCURCrawler:
    Type: 'AWS::Glue::Crawler'
    DependsOn:
      - AWSCURDatabase
      - AWSCURCrawlerComponentFunction
    Properties:
      Name: AWSCURCrawler-billing-report-20230608
      Description: A recurring crawler that keeps your CUR table in Athena up-to-date.
      Role: !GetAtt AWSCURCrawlerComponentFunction.Arn
      DatabaseName: !Ref AWSCURDatabase
      Targets:
        S3Targets:
          - Path: 's3://t-tkm-billing-reports-20230608/billing/billing-report-20230608/billing-report-20230608'
            Exclusions:
              - '**.json'
              - '**.yml'
              - '**.sql'
              - '**.csv'
              - '**.gz'
              - '**.zip'
      SchemaChangePolicy:
        UpdateBehavior: UPDATE_IN_DATABASE
        DeleteBehavior: DELETE_FROM_DATABASE
       

  AWSCURInitializer:
    Type: 'AWS::Lambda::Function'
    DependsOn: AWSCURCrawler
    Properties:
      Code:
        ZipFile: >
          const AWS = require('aws-sdk');
          const response = require('./cfn-response');
          exports.handler = function(event, context, callback) {
            if (event.RequestType === 'Delete') {
              response.send(event, context, response.SUCCESS);
            } else {
              const glue = new AWS.Glue();
              glue.startCrawler({ Name: 'AWSCURCrawler-billing-report-20230608' }, function(err, data) {
                if (err) {
                  const responseData = JSON.parse(this.httpResponse.body);
                  if (responseData['__type'] == 'CrawlerRunningException') {
                    callback(null, responseData.Message);
                  } else {
                    const responseString = JSON.stringify(responseData);
                    if (event.ResponseURL) {
                      response.send(event, context, response.FAILED,{ msg: responseString });
                    } else {
                      callback(responseString);
                    }
                  }
                }
                else {
                  if (event.ResponseURL) {
                    response.send(event, context, response.SUCCESS);
                  } else {
                    callback(null, response.SUCCESS);
                  }
                }
              });
            }
          };
      Handler: 'index.handler'
      Timeout: 30
      Runtime: nodejs16.x
      ReservedConcurrentExecutions: 1
      Role: !GetAtt AWSCURCrawlerLambdaExecutor.Arn
     

  AWSStartCURCrawler:
    Type: 'Custom::AWSStartCURCrawler'
    Properties:
      ServiceToken: !GetAtt AWSCURInitializer.Arn
     

  AWSS3CUREventLambdaPermission:
    Type: AWS::Lambda::Permission
    Properties:
      Action: 'lambda:InvokeFunction'
      FunctionName: !GetAtt AWSCURInitializer.Arn
      Principal: 's3.amazonaws.com'
      SourceAccount: !Ref AWS::AccountId
      SourceArn: !Sub 'arn:${AWS::Partition}:s3:::t-tkm-billing-reports-20230608'
     

  AWSS3CURLambdaExecutor:
    Type: 'AWS::IAM::Role'
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - lambda.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Path: /
      Policies:
        - PolicyName: AWSS3CURLambdaExecutor
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - 'logs:CreateLogGroup'
                  - 'logs:CreateLogStream'
                  - 'logs:PutLogEvents'
                Resource: !Sub 'arn:${AWS::Partition}:logs:*:*:*'
              - Effect: Allow
                Action:
                  - 's3:PutBucketNotification'
                Resource: !Sub 'arn:${AWS::Partition}:s3:::t-tkm-billing-reports-20230608'
       

  AWSS3CURNotification:
    Type: 'AWS::Lambda::Function'
    DependsOn:
    - AWSCURInitializer
    - AWSS3CUREventLambdaPermission
    - AWSS3CURLambdaExecutor
    Properties:
      Code:
        ZipFile: >
          const AWS = require('aws-sdk');
          const response = require('./cfn-response');
          exports.handler = function(event, context, callback) {
            const s3 = new AWS.S3();
            const putConfigRequest = function(notificationConfiguration) {
              return new Promise(function(resolve, reject) {
                s3.putBucketNotificationConfiguration({
                  Bucket: event.ResourceProperties.BucketName,
                  NotificationConfiguration: notificationConfiguration
                }, function(err, data) {
                  if (err) reject({ msg: this.httpResponse.body.toString(), error: err, data: data });
                  else resolve(data);
                });
              });
            };
            const newNotificationConfig = {};
            if (event.RequestType !== 'Delete') {
              newNotificationConfig.LambdaFunctionConfigurations = [{
                Events: [ 's3:ObjectCreated:*' ],
                LambdaFunctionArn: event.ResourceProperties.TargetLambdaArn || 'missing arn',
                Filter: { Key: { FilterRules: [ { Name: 'prefix', Value: event.ResourceProperties.ReportKey } ] } }
              }];
            }
            putConfigRequest(newNotificationConfig).then(function(result) {
              response.send(event, context, response.SUCCESS, result);
              callback(null, result);
            }).catch(function(error) {
              response.send(event, context, response.FAILED, error);
              console.log(error);
              callback(error);
            });
          };
      Handler: 'index.handler'
      Timeout: 30
      Runtime: nodejs16.x
      ReservedConcurrentExecutions: 1
      Role: !GetAtt AWSS3CURLambdaExecutor.Arn
     

  AWSPutS3CURNotification:
    Type: 'Custom::AWSPutS3CURNotification'
    Properties:
      ServiceToken: !GetAtt AWSS3CURNotification.Arn
      TargetLambdaArn: !GetAtt AWSCURInitializer.Arn
      BucketName: 't-tkm-billing-reports-20230608'
      ReportKey: 'billing/billing-report-20230608/billing-report-20230608'
     

  AWSCURReportStatusTable:
    Type: 'AWS::Glue::Table'
    DependsOn: AWSCURDatabase
    Properties:
      DatabaseName: athenacurcfn_billing_report_20230608
      CatalogId: !Ref AWS::AccountId
      TableInput:
        Name: 'cost_and_usage_data_status'
        TableType: 'EXTERNAL_TABLE'
        StorageDescriptor:
          Columns:
            - Name: status
              Type: 'string'
          InputFormat: 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat'
          OutputFormat: 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat'
          SerdeInfo:
            SerializationLibrary: 'org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe'
          Location: 's3://t-tkm-billing-reports-20230608/billing/billing-report-20230608/cost_and_usage_data_status/'
```
</details>

Cloud Fromationのデザイナで確認した結果:

{{< figure alt="img6" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img6.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img6.png">}}


AWS Glueの(AWS費用用にCrawlerが生成した)データカタログも確認します。Glueクローラにより、
列などのスキーマが自動で設定されています(データの141列＋パーテション(year,month)で143列)。
{{< figure alt="img7" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img7.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img7.png">}}

Athenaから、クエリを実行して費用データが取得できることを確認します。クエリの実行には、結果を
保存するS3バケットの指定が必要です。適当に指定します(ex. t-tkm-athena-query-20230608)
{{< figure alt="img8" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img8.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img8.png">}}

尚、Athenaだけでも、分析は十分可能です。興味がある方は、AWS W-A Labsにあるクエリライブラリ
を参考にして下さい↓

[AWS CUR QUERY LIBRARY](https://wellarchitectedlabs.com/cost/300_labs/300_cur_queries/#queries)

## ステップ(3): 分析ツールからレポート確認
### IAMユーザ作成
Grafana/RedashからAWSへのアクセスは、今回は簡単のためアクセスキー&シークレットキーを使います。
IAMユーザを作成し、「AWSQuicksightAthenaAccess」「AmazonS3FullAccess」を付与しました。
{{< figure alt="img20" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img20.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img20.png">}}

### Grafana構築
それでは、手元のローカルPC(iMac/Intel Core i5/Ventura13.4)で、こちら([Run Grafana via Docker Compose](https://grafana.com/docs/grafana/latest/setup-grafana/installation/docker/#run-grafana-via-docker-compose))の手順に従いGrafana(コンテナ)を動かします。

コマンドログ
```zsh
% cd grafana
% touch docker-compose.yml
% docker compose up -d
 ```

※コンテナイメージは、(enterpriseでも良いと思いますが)OSS版に変更しました。

image: grafana/grafana-enterprise -> image: grafana/grafana-oss

docker-compose.yml
```yaml
version: "3.8"
services:
  grafana:
    image: grafana/grafana-oss
    container_name: grafana
    restart: unless-stopped
    ports:
     - '3000:3000'
    volumes:
     - 'grafana_storage:/var/lib/grafana'
volumes:
  grafana_storage: {}
```

コンテナが起動したら、ブラウザからhttp://localhost:3000/へ接続してみます。
{{< figure alt="img9" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img9.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img9.png">}}

無事ログインできました。

Athena pluginの設定は、こちら([Grafana plugins/Amazon Athena](https://grafana.com/grafana/plugins/grafana-athena-datasource/))の手順に従い設定します。

※手順では、よりS3への権限を絞ったポリシーの雛形がありますが、今回は(§冒頭で作成したIAMユーザ
を利用するため、このカスタムポリシーは)使用していません。

{{< figure alt="img10" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img10.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img10.png">}}

接続確認してみます。okです。

※接続先テーブルは「"athenacurcfn_billing_report_20230608"."cost_and_usage_data_status"」
{{< figure alt="img16" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img16.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img16.png">}}

Athena pluginにはCUR用のテンプレート(ダッシュボード)が準備されています。早速使ってみます。
{{< figure alt="img15" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img15.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img15.png">}}

データが少なく、寂しい感じになっております。。。
{{< figure alt="img11" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img11.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img11.png">}}

こういう時は、サンプルデータを活用しましょう！AWS W-A Labsの「[LEVEL 200: COST AND USAGE ANALYSIS](https://wellarchitectedlabs.com/cost/200_labs/200_4_cost_and_usage_analysis/1_verify_cur/)」
のサンプルを使ってみます。2018年10月〜12月のサンプルデータを、サイトの手順に従いAthenaで分析
できる準備をします。Athenaの準備ができれば、もう一度Grafanaのダッシュボードで確認します。
こんな感じになりました。

※以降は、(私の環境から取得したデータは少ないため)このサンプルデータでを使っていきます。

{{< figure alt="img14" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img14.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img14.png">}}

### Redash構築
こちら([Redash/Docker Based Developer Installation Guide](https://redash.io/help/open-source/dev-guide/docker))
の手順に従い、ローカルPCでRedash(コンテナ)を動かします。Grafanaと異なり、リソースのビルドが
必要で少し手間がかかります。

コマンドログ
```zsh
% git clone https://github.com/getredash/redash.git
% cd redash
% touch .env

// .envにREDASH_COOKIE_SECRETの設定が必要。任意の文字列で良さそうだが、今回はpwgenで
//シークレットを生成し、.envファイルに記載。
% pwgen -1s 32

//これでRedashサーバは起動するが、後述フロントエンド(GUI)のビルド、DB構築が必要
% docker compose up -d

//手順通りyarnコマンドを実行するが、nodejsのバージョンエラー。nodejsのバージョンを変更。 
% asdf local nodejs 16.20.0
% yarn --frozen-lockfile\n
% docker-compose run --rm server create_db
% docker-compose run --rm postgres psql -h postgres -U postgres -c "create database tests"
% yarn build
```

以上、コンテナが起動したら、ブラウザからhttp://localhost:5001/へ接続し、初期設定をします。
ユーザ名、メールアドレス、パスワードを入力します。
{{< figure alt="img12" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img12.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img12.png">}}

次に、データソースを追加します(こちらはAthenaのアイコンが古いw)。
{{< figure alt="img13" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img13.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img13.png">}}

データソース作成後、「Test Connection」で「Success」と表示されることを確認します。

Grafanaのケースと異なり、CUR用の公式ダッシュボードがあるようでは無さそうです。AWSリソース毎
の費用推移を集計するクエリを自前で準備します。

※費用データは、先ほどAWS W-A Labsの「[LEVEL 200: COST AND USAGE ANALYSIS](https://wellarchitectedlabs.com/cost/200_labs/200_4_cost_and_usage_analysis/1_verify_cur/)」
のサンプルを使っています。Glueカタログのデータベース名、テーブル名は「"cost"."t_tkm_billing_sample"」。

```sql
SELECT
    DATE_TRUNC('day', line_item_usage_start_date) AS date,
    line_item_product_code AS AWS_Service,
    SUM(line_item_unblended_cost) AS total_cost
FROM 
    "cost"."t_tkm_billing_sample"
WHERE
    year = '2018'
GROUP BY
    DATE_TRUNC('day', line_item_usage_start_date),
    line_item_product_code
ORDER BY 
    date, total_cost desc;
```

クエリエディタの下「Add Visualization」でグラフ(Chart)を作れます。
{{< figure alt="img17" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img17.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img17.png">}}

作成したクエリを元に、ダッシュボードを作成しましょう。
{{< figure alt="img18" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img18.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img18.png">}}

# まとめ
以上、AWS Cost Usage Report(AWS CUR)サービスを有効化し、AWS費用データをS3バケットに格納
しました。次に、Athenaで格納されたS3データ(parquet圧縮ファイル)をSQL分析するために、
Glueサービスを用いてレポートのクローリング、及びデータカタログを作成しました。最後に、Grafana、
RedashをローカルPCで起動させ、Athenaをデータソースにクエリ実行、ダッシュボードが作成できる事を
確認できました。

今回クライアント(Grafana/Redash)で登録したSQLデータソース(Athena)は、RDB(SQL)とは違うという点は注意
が必要です。Athenaへのクエリ性能や、クエリ毎に費用がかかる、という点は理解しておく必要があります。

Next Stepとして、参考文献を活用し、(例えば下記のようなダッシュボードなど)本格的な「分析」
を楽しみたいですね！

CUDOSダッシュボード(参考文献[5])↓

※Cost and Usage Dashboards Operations Solution (CUDOS)
{{< figure alt="img21" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img21.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img21.png">}}

[[AWS Blog]Visualize and gain insights into your AWS cost and usage with Cloud Intelligence Dashboards and CUDOS using Amazon QuickSight](
https://aws.amazon.com/jp/blogs/mt/visualize-and-gain-insights-into-your-aws-cost-and-usage-with-cloud-intelligence-dashboards-using-amazon-quicksight/)

(蛇足)
私が駆け出しエンジニア(15年以上前)の頃は、このようなシステムを構築するにもVMを準備したり、
各種OSS(ミドルウェア)をプロビジョニングしたり等、そこそこ大変でした。

一方今は、クラウドのマネージドサービスや、コンテナ技術のお陰で、半日もあれば一通りのシステムが
構築できてしまい「便利になったなぁ」と痛感します。
(勿論、本番環境への導入など考慮すべき設計を無視したお試し環境ではありますが)

# 参考
1. [AWSコスト最適化ガイドブック](https://www.amazon.co.jp/dp/4046053550)
2. [AWSではじめるデータレイク](https://www.amazon.co.jp/dp/491031301X)
3. [AWS Well-Architected Labs](https://wellarchitectedlabs.com/)
4. [AWS W-A Labs/COST OPTIMIZATION](https://wellarchitectedlabs.com/cost/)
5. [CUDOSダッシュボードサンプル](https://d1s0yx3p3y3rah.cloudfront.net/anonymous-embed?dashboard=cudos)
6. [GitHub: aws-samples/aws-cudos-framework-deployment](https://github.com/aws-samples/aws-cudos-framework-deployment)
7. [AWS費用監視ツール(前編:AWS マネジメントコンソール&AWS CLI)](https://qiita.com/t-taku/items/ef0e7edc79f89929d466)
8. [AWS費用監視ツール(後編:Lambda活用)](https://qiita.com/t-taku/items/56f0e79dda55d89107e4)

# 付録 ローカルPC上のコンテナ
最終的に、ローカルPC上ではこのような感じでGrafana/Redashコンテナが起動しております。
<details>
  <summary>Grafana用docker-compose.yml(展開)</summary>

```yaml
version: "3.8"
services:
  grafana:
    image: grafana/grafana-oss
    container_name: grafana
    restart: unless-stopped
    ports:
     - '3000:3000'
    volumes:
     - 'grafana_storage:/var/lib/grafana'
volumes:
  grafana_storage: {}
```
</details>

<details>
  <summary>Redash用docker-compose.yml(展開)</summary>

```yaml
# This configuration file is for the **development** setup.
# For a production example please refer to getredash/setup repository on GitHub.
version: "2.2"
x-redash-service: &redash-service
  build:
    context: .
    args:
      skip_frontend_build: "true"  # set to empty string to build
  volumes:
    - .:/app
  env_file:
    - .env
x-redash-environment: &redash-environment
  REDASH_LOG_LEVEL: "INFO"
  REDASH_REDIS_URL: "redis://redis:6379/0"
  REDASH_DATABASE_URL: "postgresql://postgres@postgres/postgres"
  REDASH_RATELIMIT_ENABLED: "false"
  REDASH_MAIL_DEFAULT_SENDER: "redash@example.com"
  REDASH_MAIL_SERVER: "email"
  REDASH_ENFORCE_CSRF: "true"
  REDASH_GUNICORN_TIMEOUT: 60
  # Set secret keys in the .env file
services:
  server:
    <<: *redash-service
    command: dev_server
    depends_on:
      - postgres
      - redis
    ports:
      - "5001:5000"
      - "5678:5678"
    environment:
      <<: *redash-environment
      PYTHONUNBUFFERED: 0
  scheduler:
    <<: *redash-service
    command: dev_scheduler
    depends_on:
      - server
    environment:
      <<: *redash-environment
  worker:
    <<: *redash-service
    command: dev_worker
    depends_on:
      - server
    environment:
      <<: *redash-environment
      PYTHONUNBUFFERED: 0
  redis:
    image: redis:3-alpine
    restart: unless-stopped
  postgres:
    image: postgres:9.5-alpine
    ports:
      - "15432:5432"
    # The following turns the DB into less durable, but gains significant performance improvements for the tests run (x3
    # improvement on my personal machine). We should consider moving this into a dedicated Docker Compose configuration for
    # tests.
    command: "postgres -c fsync=off -c full_page_writes=off -c synchronous_commit=OFF"
    restart: unless-stopped
    environment:
      POSTGRES_HOST_AUTH_METHOD: "trust"
  email:
    image: maildev/maildev
    ports:
      - "1080:80"
    restart: unless-stopped
```
</details>
{{< figure alt="img19" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img19.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report/img19.png">}}