+++
Categories = ["AWS"]
Tags = ["AWS", "Container", "ECS", "EKS", "App2Container"]
date = "2022-03-12T00:00:00+09:00"
title = "AWS App2Container(お試し編)"
+++

# はじめに
AWS App2Containerは、起動中のjavaアプリをコンテナイメージに変換し、ECSやEKSで稼働させるためのテンプレートを生成するツールになります。

[Accelerating your Migration to AWS](https://aws.amazon.com/jp/blogs/architecture/accelerating-your-migration-to-aws/)

{{< figure alt="img4" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img4.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img4.png">}}

下記、App2ContainerのUerGuideにある通り、サポートされるプラットフォームと、そうでない場合で挙動が変わるため、本記事ではその辺を試してみたいと思います(ここでは、ECSやEKSでの稼働検証は含まれません)。

> For supported application frameworks, App2Container targets only the application files and dependencies that are needed for containerization, thereby minimizing the size of the resulting container image. This is known as application mode.

> If App2Container does not find a supported framework running on your application server, or if you have other dependent processes running on your server, App2Container takes a conservative approach to identifying dependencies. This is known as process mode. For process mode, all non-system files on the application server are included in the container image.

以降は、次の流れで検証します。
- App2Containerのインストール&セットアップ
- Tomcatアプリ準備
- SpringBootアプリ準備
- App2Containerの利用

図にすると、こんな感じになります。
{{< figure alt="img5" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img5.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img5.png">}}

# App2Containerのインストール&セットアップ
今回の作業は、Cloud9(Ubuntu)を使い実施しました。

{{< figure alt="img3" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img3.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img3.png">}}

Cloud9へログインして、shellで作業します。App2Container(コマンド)の実行はroot権限が必要なのでroot昇格します。

```bash
$ sudo su -
```

App2Containerは結構なディスクを使います。デフォルトのEBSディスクサイズが小さかったため、事前にEBSサイズを10GBから300GBへ拡張しています。

```bash
root@ip-10-0-1-112:~# df -h
Filesystem      Size  Used Avail Use% Mounted on
(snip)
/dev/nvme0n1p1  291G  8.6G  283G   3% /
(snip)
```

App2Containerのインストールパッケージをダウンロード。

```bash
root@ip-10-0-1-112:~# curl -o AWSApp2Container-installer-linux.tar.gz https://app2container-release-us-east-1.s3.us-east-1.amazonaws.com/latest/linux/AWSApp2Container-installer-linux.tar.gz
(snip)
```

パッケージを解凍して、解凍したディレクトリへ移動。

```bash
root@ip-10-0-1-112:~# tar xvf AWSApp2Container-installer-linux.tar.gz
install.sh
README
termsandconditions.txt
security/
security/app2container.cert
security/app2container.sig
AWSApp2Container.tar.gz
```

インストールスクリプトを実行。途中、利用規約に同意するか聞かれるので「y」を押して継続。

```bash
root@ip-10-0-1-112:~# ./install.sh 
Determining user ...
Determining Installer Path ...
Checking the current installed A2C version
The AWS App2Container tool is licensed as "AWS Content" under the terms and conditions of the AWS Customer Agreement, located at https://aws.amazon.com/agreement and the Service Terms, located at https://aws.amazon.com/service-terms. By installing, using or accessing the AWS App2Container tool, you agree to such terms and conditions. The term "AWS Content" does not include software and assets distributed under separate license terms (such as code licensed under an open source license).
Do you accept the terms and conditions above? (y/n): y
Installing AWS App2Container ...
AWSApp2Container/
AWSApp2Container/THIRD_PARTY_LICENSES

(snip)

Installation of AWS App2Container completed successfully!
You are currently running version 1.13.
To get started, run 'sudo app2container init'
AWS App2Container was installed under /usr/local/app2container/AWSApp2Container.
```

App2Containerコマンドが使用できれば、インストール完了。

```bash
root@ip-10-0-1-112:~# app2container help
app2container is an application from Amazon Web Services (AWS),
that provides commands to discover and containerize applications.

Commands 
  Getting Started 🌱
    init                  Sets up workspace for artifacts.
 
 (snip)
```

次に、App2Containerを初期化します。App2ContainerはAWS CLIを使うため、事前にプロファイルも設定しておきます。成果物を格納するS3バケットも適当に準備しておきます。

```bash
root@ip-10-0-1-112:~# aws configure
AWS Access Key ID [None]: <<Your Access Key>>
AWS Secret Access Key [None]: <<Your Secret Key>>
Default region name [None]: ap-northeast-1
Default output format [None]: 
```

initオプションでApp2Containerの初期化。使ったパラメータは下記。(AWSへのレポートは「y」で良かったかも^^;;)
  Patam   | Value 
---------------|----------
  Workspace directory path | デフォルト(/root/app2container)
  AWS Profile  |  デフォルト(defalut)
  Optional S3 bucket for application artifacts | 適当なS3バケット指定
  Report usage metrics to AWS? | n
  Automatically upload logs and App2Container generated artifacts on crashes and internal errors? | n
  Require images to be signed using Docker Content Trust (DCT)? | デフォルト(n)

以上でApp2Containerの使用準備は完了。

```bash
root@ip-10-0-1-112:~# app2container init
Workspace directory path for artifacts[default: /root/app2container]: 
AWS Profile (configured using 'aws configure --profile')[default: default]: 
Optional S3 bucket for application artifacts: t-tkm-app2container
Report usage metrics to AWS? (Y/N)[default: y]: n
Automatically upload logs and App2Container generated artifacts on crashes and internal errors? (Y/N)[default: y]: n
Require images to be signed using Docker Content Trust (DCT)? (Y/N)[default: n]: 
Configuration saved
All application artifacts will be created under /root/app2container. Please ensure that the folder permissions are secure.
```

## Tomcatアプリ準備
今回は、APTパッケージを使わず、一般的なインストールを行いました。

```bash
root@ip-10-0-1-112:~# VERSION=9.0.59
root@ip-10-0-1-112:~# wget https://www-eu.apache.org/dist/tomcat/tomcat-9/v${VERSION}/bin/apache-tomcat-${VERSION}.tar.gz
(snip)
root@ip-10-0-1-112:~# sudo useradd -m -U -d /opt/tomcat -s /bin/false tomcat
root@ip-10-0-1-112:~# tar -xf ./apache-tomcat-${VERSION}.tar.gz -C /opt/tomcat/
root@ip-10-0-1-112:~# chown -R tomcat: /opt/tomcat
root@ip-10-0-1-112:~# ln -s /opt/tomcat/apache-tomcat-${VERSION} /opt/tomcat/latest
root@ip-10-0-1-112:/opt/tomcat# vi /etc/systemd/system/tomcat.service
```
ワンショットで試すだけなので、(/opt/tomcat/latest/bin/startup.shの)シェルスクリプト実行だけで良く、サービス化は不要かもしれませんが、念の為Unitファイルも作成。

**tomcat.service**
```vim
[Unit]
Description=Tomcat 9 servlet container
After=network.target
[Service]
Type=forking
User=tomcat
Group=tomcat
Environment="JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64"
Environment="JAVA_OPTS=-Djava.security.egd=file:///dev/urandom -Djava.awt.headless=true"
Environment="CATALINA_BASE=/opt/tomcat/latest"
Environment="CATALINA_HOME=/opt/tomcat/latest"
Environment="CATALINA_PID=/opt/tomcat/latest/temp/tomcat.pid"
Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"
ExecStart=/opt/tomcat/latest/bin/startup.sh
ExecStop=/opt/tomcat/latest/bin/shutdown.sh
[Install]
WantedBy=multi-user.target
```

```bash
root@ip-10-0-1-112:/opt/tomcat# systemctl daemon-reload
root@ip-10-0-1-112:/opt/tomcat# systemctl enable --now tomcat
Created symlink /etc/systemd/system/multi-user.target.wants/tomcat.service → /etc/systemd/system/tomcat.service.
```

アプリケーションとして、こちらのサンプルwarをデプロイ。

https://tomcat.apache.org/tomcat-7.0-doc/appdev/sample/

```bash
root@ip-10-0-1-112:/opt/tomcat/latest/webapps# ll
total 28
drwxr-x---  7 tomcat tomcat 4096 Feb 21 21:01 ./
drwxr-xr-x  9 tomcat tomcat 4096 Mar  6 05:44 ../
drwxr-x---  3 tomcat tomcat 4096 Mar  6 05:44 ROOT/
drwxr-x--- 15 tomcat tomcat 4096 Mar  6 05:44 docs/
drwxr-x---  7 tomcat tomcat 4096 Mar  6 05:44 examples/
drwxr-x---  6 tomcat tomcat 4096 Mar  6 05:44 host-manager/
drwxr-x---  6 tomcat tomcat 4096 Mar  6 05:44 manager/

root@ip-10-0-1-112:/opt/tomcat/latest/webapps# wget https://tomcat.apache.org/tomcat-7.0-doc/appdev/sample/sample.war

root@ip-10-0-1-112:/opt/tomcat/latest/webapps# ll
total 40
drwxr-x---  8 tomcat tomcat 4096 Mar  6 05:57 ./
drwxr-xr-x  9 tomcat tomcat 4096 Mar  6 05:44 ../
drwxr-x---  3 tomcat tomcat 4096 Mar  6 05:44 ROOT/
drwxr-x--- 15 tomcat tomcat 4096 Mar  6 05:44 docs/
drwxr-x---  7 tomcat tomcat 4096 Mar  6 05:44 examples/
drwxr-x---  6 tomcat tomcat 4096 Mar  6 05:44 host-manager/
drwxr-x---  6 tomcat tomcat 4096 Mar  6 05:44 manager/
drwxr-x---  5 tomcat tomcat 4096 Mar  6 05:57 sample/
-rw-r--r--  1 root   root   4606 Mar 31  2012 sample.war
```

動作確認します。GUIの確認は、Cloud9のPreview機能が使えます。但し、任意のポート番号が使えるのでなく、下記3ポートに限られているような点に注意が必要です。
> IDE 内からアプリケーションをプレビューするには、まずポート 8080、8081、または8082 を経由して、IP 127.0.0.1、localhost または 0.0.0.0 で HTTP を使用して、AWS Cloud9開発環境で実行している必要があります。

https://docs.aws.amazon.com/ja_jp/cloud9/latest/user-guide/app-preview.html#app-preview-run-app

{{< figure alt="img10" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img10.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img10.png">}}

別タブ(ウインドウ)で表示することもできます！
{{< figure alt="img11" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img11.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img11.png">}}



## SpringBootアプリ準備
Spring Bootのインストールには、SDKMAN!を利用しました。

```bash
root@ip-10-0-1-112:~# curl -s "https://get.sdkman.io" | bash

root@ip-10-0-1-112:~# source "$HOME/.sdkman/bin/sdkman-init.sh"

root@ip-10-0-1-112:~# sdk list springboot
================================================================================
Available Springboot Versions
================================================================================
     2.6.4               2.3.4.RELEASE       2.0.8.RELEASE       1.4.1.RELEASE  
     2.6.3               2.3.3.RELEASE       2.0.7.RELEASE       1.4.0.RELEASE  
     2.6.2               2.3.2.RELEASE       2.0.6.RELEASE       1.3.8.RELEASE  
(snip)
     2.3.7.RELEASE       2.1.1.RELEASE       1.4.4.RELEASE       1.0.1.RELEASE  
     2.3.6.RELEASE       2.1.0.RELEASE       1.4.3.RELEASE       1.0.0.RELEASE  
     2.3.5.RELEASE       2.0.9.RELEASE       1.4.2.RELEASE                      

================================================================================
+ - local version
* - installed
> - currently in use
================================================================================

root@ip-10-0-1-112:~# sdk install springboot
(snip)

```

サンプルアプリは、下記を使います。

[2.1. CLI を使用したアプリケーションの実行](
https://spring.pleiades.io/spring-boot/docs/current/reference/html/cli.html#cli.using-the-cli.run)

```bash
root@ip-10-0-1-112:~# cat hello.groovy 
@RestController
class WebApplication {
    @RequestMapping("/")
    String home() {
        "Hello World!"
    }
}
```

ポートは、tomcat(8080)と被らないように8081で起動します。
```bash
root@ip-10-0-1-112:~# spring run hello.groovy -- --server.port=8081

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.6.4)

2022-03-06 08:27:48.301  INFO 3803 --- [       runner-0] o.s.boot.SpringApplication               : Starting application using Java 11.0.13 on ip-10-0-1-112 with PID 3803 (started by root in /root)
2022-03-06 08:27:48.307  INFO 3803 --- [       runner-0] o.s.boot.SpringApplication               : No active profile set, falling back to 1 default profile: "default"
2022-03-06 08:27:50.482  INFO 3803 --- [       runner-0] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8081 (http)
202
(snip)
```

動作確認します。大丈夫のようです。
{{< figure alt="img12" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img12.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img12.png">}}



## App2Containerの利用
準備ができたので、早速アプリケーションをinventoryしてみます。無事に検出できているようです。
```bash
root@ip-10-0-1-112:~# app2container inventory
{
                "java-generic-6ef9339e": {
                                "processId": 6120,
                                "cmdline": "/usr/lib/jvm/java-11-openjdk-amd64/bin/java ... run hello.groovy -- --server.port=8081 ",
                                "applicationType": "java-generic",
                                "webApp": ""
                },
                "java-tomcat-5da060de": {
                                "processId": 4672,
                                "cmdline": "/usr/lib/jvm/java-11-openjdk-amd64/bin/java ... -Dcatalina.home=/opt/tomcat/latest -Djava.io.tmpdir=/opt/tomcat/latest/temp org.apache.catalina.startup.Bootstrap start ",
                                "applicationType": "java-tomcat",
                                "webApp": "sample"
                }
}
```
2つのjavaアプリが検出され、idが付与されています。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img1.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img1.png">}}


次に、それぞれのアプリを分析(analyze)します。この作業は一瞬で完了します。
```bash
root@ip-10-0-1-112:~# app2container analyze --application-id java-tomcat-5da060de
✔ Created artifacts folder /root/app2container/java-tomcat-5da060de
✔ Generated analysis data in /root/app2container/java-tomcat-5da060de/analysis.json
👍 Analysis successful for application java-tomcat-5da060de

💡 Next Steps:
1. View the application analysis file at /root/app2container/java-tomcat-5da060de/analysis.json.
2. Edit the application analysis file as needed.
3. Start the containerization process using this command: app2container containerize --application-id java-tomcat-5da060de
```

```bash
root@ip-10-0-1-112:~# app2container analyze --application-id java-generic-6ef9339e
✔ Created artifacts folder /root/app2container/java-generic-6ef9339e
✔ Generated analysis data in /root/app2container/java-generic-6ef9339e/analysis.json
👍 Analysis successful for application java-generic-6ef9339e

💡 Next Steps:
1. View the application analysis file at /root/app2container/java-generic-6ef9339e/analysis.json.
2. Edit the application analysis file as needed.
3. Start the containerization process using this command: app2container containerize --application-id java-generic-6ef9339e
```

分析の結果、work directory(ex. app2container)にJSONファイルが作成されます。
```bash
root@ip-10-0-1-112:~# ll app2container/
total 20
drwxr-xr-x  5 root root 4096 Mar  6 06:14 ./
drwx------ 12 root root 4096 Mar  6 06:07 ../
drwxr-xr-x  2 root root 4096 Mar  6 06:14 java-generic-6ef9339e/
drwxr-xr-x  2 root root 4096 Mar  6 06:13 java-tomcat-5da060de/
drwxr-xr-x  2 root root 4096 Mar  6 05:36 log/
```

```bash
#treeツールをインストール
root@ip-10-0-1-112:~# apt install tree
(snip)

root@ip-10-0-1-112:~# tree app2container/java-tomcat-5da060de/
app2container/java-tomcat-5da060de/
└── analysis.json
root@ip-10-0-1-112:~# tree app2container/java-generic-6ef9339e/
app2container/java-generic-6ef9339e/
└── analysis.json
```

分析のタイミングでは、コンテナイメージなどは作成されません。(Cloud9(Ubuntu)作成時にデフォルトでpullされているイメージのみ)
```bash
root@ip-10-0-1-112:~# docker image ls
REPOSITORY      TAG          IMAGE ID       CREATED         SIZE
lambci/lambda   python3.8    094248252696   13 months ago   524MB
lambci/lambda   nodejs12.x   22a4ada8399c   13 months ago   390MB
lambci/lambda   nodejs10.x   db93be728e7b   13 months ago   385MB
lambci/lambda   python3.7    22b4b6fd9260   13 months ago   946MB
lambci/lambda   python3.6    177c85a10179   13 months ago   894MB
lambci/lambda   python2.7    d96a01fe4c80   13 months ago   763MB
lambci/lambda   nodejs8.10   5754fee26e6e   13 months ago   813MB
```

tomcatアプリのコンテナイメージを作成します(5分ほどかかりました)。
```bash
root@ip-10-0-1-112:~# app2container containerize --application-id java-tomcat-5da060de
✔ AWS prerequisite check succeeded
✔ Docker prerequisite check succeeded
✔ Extracted container artifacts for application
✔ Entry file generated
✔ Dockerfile generated under /root/app2container/java-tomcat-5da060de/Artifacts
✔ Generated dockerfile.update under /root/app2container/java-tomcat-5da060de/Artifacts
✔ Generated deployment file at /root/app2container/java-tomcat-5da060de/deployment.json
✔ Deployment artifacts generated.
✔ Pre-validation succeeded.
👍 Containerization successful. Generated docker image java-tomcat-5da060de

💡 You're all set to test and deploy your container image.

Next Steps:
1. View the container image with "docker images" and test the application with "docker run --name java-tomcat-5da060de -Pit java-tomcat-5da060de".
2. When you're ready to deploy to AWS, adjust the appropriate fields in /root/app2container/java-tomcat-5da060de/deployment.json to generate the desired deployment artifact. Note that by default "createEcsArtifacts" is set to true.
3. Generate deployment artifacts using "app2container generate app-deployment --application-id java-tomcat-5da060de".
```

成果物(work directory)と、生成されたコンテナイメージです。
```bash
root@ip-10-0-1-112:~# tree app2container/java-tomcat-5da060de/
app2container/java-tomcat-5da060de/
├── Artifacts
│   ├── ContainerFiles.tar
│   ├── Dockerfile
│   ├── Dockerfile.update
│   ├── entryfile
│   ├── excludedFiles
│   └── includeFiles
├── analysis.json
└── deployment.json

1 directory, 8 files
```

802MBのコンテナイメージが生成されました。
```bash
root@ip-10-0-1-112:~# docker image ls
REPOSITORY             TAG          IMAGE ID       CREATED              SIZE
java-tomcat-5da060de   latest       cc2de1db6ef8   About a minute ago   802MB
ubuntu                 18.04        eb2556e0f6e4   2 days ago           63.2MB
(snip)
```

コンテナを起動し、アプリケーションの動作を確認します。
```bash
root@ip-10-0-1-112:~# docker run -d --rm -p 8082:8080 java-tomcat-5da060de
55dc392f0ee619afb61298b6a2ee05758b495a10c7f6b6bc2eb082d34e2dcd79

root@ip-10-0-1-112:~# docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED         STATUS                            PORTS                                                 NAMES
55dc392f0ee6   java-tomcat-5da060de   "/bin/sh -c /entryfi…"   3 seconds ago   Up 2 seconds (health: starting)   8005/tcp, 0.0.0.0:8082->8080/tcp, :::8082->8080/tcp   charming_varahamihira
```

動作確認できました。
{{< figure alt="img6" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img6.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img6.png">}}
{{< figure alt="img7" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img7.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img7png">}}

お片付けします(コンテナ削除)。
```bash
root@ip-10-0-1-112:~# docker stop 55d
55d
```

次に、springbootアプリのコンテナイメージを作成します(15分〜20分ほどかかりました)。
```bash
root@ip-10-0-1-112:~# app2container containerize --application-id java-generic-6ef9339e
✔ AWS prerequisite check succeeded
✔ Docker prerequisite check succeeded
✔ Extracted container artifacts for application
✔ Entry file generated
✔ Dockerfile generated under /root/app2container/java-generic-6ef9339e/Artifacts
✔ Generated dockerfile.update under /root/app2container/java-generic-6ef9339e/Artifacts
✔ Generated deployment file at /root/app2container/java-generic-6ef9339e/deployment.json
✔ Deployment artifacts generated.
✔ Pre-validation succeeded.
👍 Containerization successful. Generated docker image java-generic-6ef9339e

💡 You're all set to test and deploy your container image.

Next Steps:
1. View the container image with "docker images" and test the application with "docker run --name java-generic-6ef9339e -Pit java-generic-6ef9339e".
2. When you're ready to deploy to AWS, adjust the appropriate fields in /root/app2container/java-generic-6ef9339e/deployment.json to generate the desired deployment artifact. Note that by default "createEcsArtifacts" is set to true.
3. Generate deployment artifacts using "app2container generate app-deployment --application-id java-generic-6ef9339e".
```

16.1GBのコンテナイメージが生成されました。
```bash
root@ip-10-0-1-112:~# docker image ls
REPOSITORY              TAG          IMAGE ID       CREATED          SIZE
java-generic-6ef9339e   latest       8ef7ef3e7db0   10 minutes ago   16.1GB
java-tomcat-5da060de    latest       cc2de1db6ef8   30 minutes ago   802MB
ubuntu                  18.04        eb2556e0f6e4   2 days ago       63.2MB
(snip)
```
これは、UserGuideにも記載ある通り、悲サポートなアプリの場合は余計なリソース含めて丸っとコンテナ化するためサイズが大きくなっています。
> App2Containerは依存関係を識別するために保守的なアプローチを取ります。 これは、プロセスモードと呼ばれます。 プロセスモードの場合、アプリケーションサーバー上のすべての非システムファイルがコンテナイメージに含まれます。

こちらも、コンテナを起動し、アプリケーションの動作を確認しておきましょう。
```bash
root@ip-10-0-1-112:~# docker run -d --rm -p 8082:8081 java-generic-6ef9339e                                                                                                                               
7fa1c72fbc37cf4352264622b8c8dae485fdac8fd8041a6aef6b5ec21a1a7073

root@ip-10-0-1-112:~# docker ps
CONTAINER ID   IMAGE                   COMMAND                  CREATED         STATUS                            PORTS                                       NAMES
7fa1c72fbc37   java-generic-6ef9339e   "/bin/sh -c /entryfi…"   7 seconds ago   Up 6 seconds (health: starting)   0.0.0.0:8082->8081/tcp, :::8082->8081/tcp   goofy_tereshkova
```

動作確認できました。
{{< figure alt="img8" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img8.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img8.png">}}
{{< figure alt="img9" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img9.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img9.png">}}

お片付けします(コンテナ削除)。
```bash
root@ip-10-0-1-112:~# docker stop 7fa
7fa
```

以上を図にすると、こんな感じになります。
{{< figure alt="img2" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img2.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img2.png">}}

非サポートjavaアプリ(SpringBootアプリ)の方は、コンテナイメージのサイズが16.3GBと大きく取り回しが難しいため、サイズを小さくする方法について、次の記事で紹介します。