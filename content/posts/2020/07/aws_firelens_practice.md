+++
Categories = ["AWS"]
Tags = ["AWS", "Alexa"]
date = "2020-07-26T00:00:00+09:00"
title = "FireLensを使ってFargateコンテナのファイルログを転送してみた"

+++

# はじめに
通常は[Twelve-Factor App](https://12factor.net/ja/)に「ログをイベントストリームとして扱う」とあるように、コンテナアプリケーションのログは標準出力を集約先とするのが設計原則です。しかし、もともと仮想マシンで動かしていたサーバをコンテナ化しようとする時など、

**「まずはアプリケーションコンテナを極力改修せず(ログ出力先を、ファイルから標準出力に変更)になんとかならないか。。。」**

というニーズもあるかと思います。ここでは、ファイル出力されたアプリケーションコンテナのログをFireLens(Fluentbit)を使ってCloudWatchへ転送する方法を紹介します。尚、FireLensを使わずとも、サイドカー(Fluentd)を自前で準備する方法として[「Fargateで起動するコンテナのログをFluentd経由でS3に保存してみた」](https://dev.classmethod.jp/articles/fargate-fluentd-s3/#toc-6)というのもあり、ご一緒に確認されると良いかと思います。

# 検証の流れ
検証は、次の手順で進めています。
  1. Fluentbitコンテナ作成 :設定ファイルの取込みと、ECRへイメージpush
  2. アプリケーションコンテナ作成 : tomcat上にsample.warをデプロイと、ECRへイメージpush
  3. Task定義作成 : CloudFormationのテンプレートを準備したので、それを使ってスタック作成
  4. Fargateクラスタ&サービス作成: AWSマネジメントコンソールから操作
  5. アプリケーションへアクセス : ブラウザから
  6. ログの確認(Cloud Watch Logs) 

システム構成は、こんな感じになります。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/blob/main/2020/07/aws_firelens_practice/img1.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2020/07/aws_firelens_practice/img1.png?raw=true">}}

また、コンテナをビルドしたりするために使用したファイル(プロジェクト)はこちらです。

```zsh
$ tree 
.
├── my-fluentbit
│   ├── Dockerfile
│   └── extra.conf //Fluentbit用設定ファイル
├── my-tomcat
|   └── Dockerfile
├── task-def.yaml  //Cloud Formationにこのテンプレートファイルを指定して、スタック作成
```
# コンテナイメージ作成
## Fluentbitコンテナ
Fluentbitの設定ファイル(extra.conf)を作成します。

```:./my-fluentbit/extra.conf
[INPUT]
    Name tail
    Path /usr/local/tomcat/logs/localhost_access_log.*.txt
    Tag file-app-logs

[OUTPUT]
    Name cloudwatch
    Match fargate-fluentbit-app*
    region ap-northeast-1
    log_group_name fluentbit-cloudwatch-stdout
    log_stream_prefix fluentbit-stdout-
    auto_create_group true

[OUTPUT]
    Name cloudwatch
    Match file-app-logs*
    region ap-northeast-1
    log_group_name fluentbit-cloudwatch-file
    log_stream_prefix fluentbit-file-
    auto_create_group true
```
[補足]tailプラグインではpos_fileで、どのファイルのどの部分まで読込み済みかチェックするための指定ができたりしますが、今回は省略しています。


次に、AWS公式イメージ(fluentbit)元に、上記設定ファイルを取り込むためのDockerfileを作成。

```dockerfile:./my-fluentbit/Dockerfile
FROM amazon/aws-for-fluent-bit
COPY extra.conf /fluent-bit/etc/extra.conf
```

最後に、ECRへ登録します。

```zsh
$ aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin <AWSアカウントID>.dkr.ecr.ap-northeast-1.amazonaws.com 
$ docker build -t my-fluentbit .
$ docker tag my-fluentbit:latest <AWSアカウントID>.dkr.ecr.ap-northeast-1.amazonaws.com/my-fluentbit:latest
$ docker push <AWSアカウントID>.dkr.ecr.ap-northeast-1.amazonaws.com/my-fluentbit:latest  
```

## Tomcatコンテナ
今回はtomcat:9.0をベースイメージにしました。tomcatだけだとブラウザでアクセス確認できないので、sampleアプリケーションをデプロイ(単にtomcatのwebappにコピーしているだけですが)します。

```dockerfile:./my-tomcat/Dockerfile
FROM tomcat:9.0
ENV CATALINA_HOME /usr/local/tomcat
ENV PATH $CATALINA_HOME/bin:$PATH
WORKDIR $CATALINA_HOME

RUN set -ex; \
	cd webapps && \
	wget https://tomcat.apache.org/tomcat-9.0-doc/appdev/sample/sample.war
```

先ほど同様、次のコマンドでこちらもECRに登録します。

```zsh
$ aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin <AWSアカウントID>.dkr.ecr.ap-northeast-1.amazonaws.com 
$ docker build -t my-tomcat .
$ docker tag my-tomcat:latest <AWSアカウントID>.dkr.ecr.ap-northeast-1.amazonaws.com/my-tomcat:latest
$ docker push <AWSアカウントID>.dkr.ecr.ap-northeast-1.amazonaws.com/my-tomcat:latest  
```
AWSのマネジメントコンソールから、ECRサービス画面で、2つのコンテナが無事登録できているか確認します。
{{< figure alt="img2" src="https://github.com/t-tkm/blog_images/blob/main/2020/07/aws_firelens_practice/img2.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2020/07/aws_firelens_practice/img2.png?raw=true">}}

# Task定義作成
Fargate用のタスクは、下記のCloud Formation用テンプレートで作成します。

```yaml:./ecs-task-def.yaml

AWSTemplateFormatVersion: '2010-09-09'
Description: ecs task definition
Parameters:
  projectName:
    Type: String
    Default: 'fargate-fluentbit'

Resources:
  ecsTaskExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement: 
          - Effect: "Allow"
            Principal: 
              Service: 
                - "ecs-tasks.amazonaws.com"
            Action: 
              - "sts:AssumeRole"
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
        - arn:aws:iam::aws:policy/AmazonS3FullAccess
        - arn:aws:iam::aws:policy/CloudWatchFullAccess
      RoleName: !Sub ${projectName}-ecsTaskExecutionRole
  Taskdefinition:
    Type: AWS::ECS::TaskDefinition
    DependsOn: ecsTaskExecutionRole
    Properties:
      Family: !Sub ${projectName}-task
      RequiresCompatibilities:
        - FARGATE
      Cpu: 1024
      Memory: 2048
      NetworkMode: awsvpc
      ExecutionRoleArn: !GetAtt ecsTaskExecutionRole.Arn
      TaskRoleArn: !GetAtt ecsTaskExecutionRole.Arn
      ContainerDefinitions:
        - Name: !Sub ${projectName}-fluentbit
          Image: !Sub ${AWS::AccountId}.dkr.ecr.ap-northeast-1.amazonaws.com/my-fluentbit:latest
          Cpu: 512
          MemoryReservation: 1024
          MountPoints: 
            - SourceVolume: "volume-for-file-logs"
              ContainerPath: "/usr/local/tomcat/logs/"
          FirelensConfiguration:
            Type: fluentbit
            Options:
              config-file-type: file
              config-file-value: "/fluent-bit/etc/extra.conf"
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-region: !Ref AWS::Region
              awslogs-group: "firelens-container"
              awslogs-stream-prefix: !Sub ${projectName}-fluentbit
              awslogs-create-group: true
          Essential: true
        - Name: !Sub ${projectName}-app
          Image: !Sub ${AWS::AccountId}.dkr.ecr.ap-northeast-1.amazonaws.com/my-tomcat:latest
          Cpu: 512
          MemoryReservation: 1024
          MountPoints: 
            - SourceVolume: "volume-for-file-logs"
              ContainerPath: "/usr/local/tomcat/logs/"
          PortMappings:
            - ContainerPort: 8080
          LogConfiguration:
            LogDriver: awsfirelens
          Essential: true
      Volumes: 
        - Name: "volume-for-file-logs"
Outputs:
  ecsTaskExecutionRole:
    Value: !Ref ecsTaskExecutionRole
    Export:
      Name: ecsTaskExecutionRole
  ecsTaskExecutionRoleArn:
    Value: !GetAtt ecsTaskExecutionRole.Arn
    Export:
      Name: ecsTaskExecutionRoleArn
  taskdefinition:
    Value: !Ref Taskdefinition
    Export:
      Name: my-task-definition

```

# 検証
## Fargateクラスタ構成
なにはともあれ、検証にはFargateクラスタが必要です。今回は下記のようなクラスタ構成を使いました。 例) VPC(10.0.0.0/16)内の2つの(パブリック)サブネット(10.0.0.0/24と10.0.1.0/24)上にFargateクラスタ構成。

{{< figure alt="img3" src="https://github.com/t-tkm/blog_images/blob/main/2020/07/aws_firelens_practice/img3.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2020/07/aws_firelens_practice/img3.png?raw=true">}}

## サービスからTask起動
※Task定義同様、CloudFormation用のテンプレートを作れば良かったのですが、今回は手を抜いてマネジメントコンソールから作成しました。。。

クラスタから、新規にサービスを作成します。特に特別な設定はないと思います。気を付けるとしtら、先ほど作成したTask定義を指定する事と、アプリケーションにアクセスするためのSGの設定をする事くらいです。

{{< figure alt="img4" src="https://github.com/t-tkm/blog_images/blob/main/2020/07/aws_firelens_practice/img4.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2020/07/aws_firelens_practice/img4.png?raw=true">}}

{{< figure alt="img5" src="https://github.com/t-tkm/blog_images/blob/main/2020/07/aws_firelens_practice/img5.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2020/07/aws_firelens_practice/img5.png?raw=true">}}

{{< figure alt="img6" src="https://github.com/t-tkm/blog_images/blob/main/2020/07/aws_firelens_practice/img6.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2020/07/aws_firelens_practice/img6.png?raw=true">}}

{{< figure alt="img7" src="https://github.com/t-tkm/blog_images/blob/main/2020/07/aws_firelens_practice/img7.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2020/07/aws_firelens_practice/img7.png?raw=true">}}

## ブラウザからアプリにアクセスしてみよう
サービスを設定し、暫くするとTaskが起動します。Task内のコンテナが「RUNNING」 であることを確認し、アプリケーションにブラウザでアクセスしてみましょう。問題なければ、tomcatのsampleアプリがブラウザに表示されます。
{{< figure alt="img8" src="https://github.com/t-tkm/blog_images/blob/main/2020/07/aws_firelens_practice/img8.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2020/07/aws_firelens_practice/img8.png?raw=true">}}

## ログの確認
Cloud Watch Logsの画面で、ログが出力されていることを確認します。
{{< figure alt="img9" src="https://github.com/t-tkm/blog_images/blob/main/2020/07/aws_firelens_practice/img9.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2020/07/aws_firelens_practice/img9.png?raw=true">}}

# まとめ
FireLensのFluentbitを使って、アプリケーションコンテナが出力するログファイルを転送してみました。

本番環境で使う場合は、予期しないコンテナ再起動などによる必要なログの抜漏れ対策に最新の注意が必要になると思います。冒頭でも述べましたが、アプリログはコンテナ内部に出力するのでなく、標準出力経由で集約できないかの検討も考慮いただければと思います。

# 参考
- [カスタムログルーティング](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/userguide/using_firelens.html)
- [FireLens を利用した Fargate タスクのログルーティング](https://yasuharu519.hatenablog.com/entry/2019/12/19/142454)
- [Under the hood: FireLens for Amazon ECS Tasks](https://aws.amazon.com/jp/blogs/containers/under-the-hood-firelens-for-amazon-ecs-tasks/)

# [付録1] FireLensを使うメリット？
自前でサイドカーとしてFluentコンテナを準備する事もできる中、あえてFireLensを使うと嬉しい点につきましては、こちらの記事[Under the hood: FireLens for Amazon ECS Tasks](https://aws.amazon.com/jp/blogs/containers/under-the-hood-firelens-for-amazon-ecs-tasks/)の中で、Wesley氏は次の2点だと述べています: 
> 
Why not simply recommend Fluentd and Fluent Bit? Why FireLens?
1. Those who want a simple way to send logs anywhere, powered by Fluentd and Fluent Bit.
2. Those who want the full power of Fluentd and Fluent Bit, with AWS managing the undifferentiated labor that’s needed to pipe a Task’s logs to these log routers.

1点目について、[こちら](https://github.com/aws-samples/amazon-ecs-firelens-examples)にある豊富なExampleが使えます。
2点目について、Fluentd/Fluentbitの設定ファイルを自由にカスタマイズでき、S3に設定ファイルを置く事ができます。

ただ、2020.7時点では、AWSによる[ドキュメント](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/userguide/using_firelens.html)によると、Fargateでは残念ながらS3に設定ファイルを置く事ができないようですorz。(ですので、今回は設定ファイルをコンテナ内に取込みました)。今後のアップデートに期待ですね！

>
{{< figure alt="img10" src="https://github.com/t-tkm/blog_images/blob/main/2020/07/aws_firelens_practice/img10.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2020/07/aws_firelens_practice/img10.png?raw=true">}}

# [付録2] Tomcatコンテナの中身確認
tomcatコンテナに、どのようなログが出力されているかの確認。

```zsh
$ docker container run --rm -p 8080:8080 --name tomcat my-tomcat
$ docker ps                                                    
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                    NAMES
7350e84b95b9        my-tomcat           "catalina.sh run"   9 seconds ago       Up 8 seconds        0.0.0.0:8080->8080/tcp   tomcat
$ docker container exec -it tomcat /bin/bash                   
root@7350e84b95b9:/usr/local/tomcat# cd logs/
root@7350e84b95b9:/usr/local/tomcat/logs# pwd
/usr/local/tomcat/logs
root@7350e84b95b9:/usr/local/tomcat/logs# ls -l
total 8
-rw-r----- 1 root root 5131 Jul 26 00:36 catalina.2020-07-26.log
-rw-r----- 1 root root    0 Jul 26 00:36 host-manager.2020-07-26.log
-rw-r----- 1 root root    0 Jul 26 00:36 localhost.2020-07-26.log
-rw-r----- 1 root root    0 Jul 26 00:36 localhost_access_log.2020-07-26.txt
-rw-r----- 1 root root    0 Jul 26 00:36 manager.2020-07-26.log
root@7350e84b95b9:/usr/local/tomcat/logs# 
```
