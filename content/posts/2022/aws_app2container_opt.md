+++
Categories = ["AWS"]
Tags = ["AWS", "Container", "ECS", "EKS", "App2Container"]
date = "2022-03-12T00:00:00+09:00"
title = "AWS App2Container(イメージ最適化編)"
+++

# はじめに
前回(「AWS App2Container(お試し編)」)で試した非サポートアプリの場合(ex. SpringBootアプリ)、サポートされるtomcatアプリと比較すると(802MB)、デフォルトで生成されたコンテナイメージは16.1GBとかなりのサイズとなっていました。

{{< figure alt="img13" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img13.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img13.png">}}
```bash
root@ip-10-0-1-112:~# docker image ls
REPOSITORY              TAG          IMAGE ID       CREATED          SIZE
java-generic-6ef9339e   latest       8ef7ef3e7db0   38 minutes ago   16.1GB
java-tomcat-5da060de    latest       cc2de1db6ef8   58 minutes ago   802MB
```
そこで、ここでは、下記ガイドラインに従いサイズをスリム化してみようと思います。

[Optimize AWS App2Container generated Docker images](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/optimize-aws-app2container-generated-docker-images.html)

ポイントは、App2Containerは、分析結果であるanalysis.jsonをベースにイメージを生成するため、不要なファイルを含めないようにanalysis.jsonを編集することになります。
{{< figure alt="img15" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img15.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img15.png">}}

# コンテナイメージ分析(dive)
はじめに、コンテナイメージの中身を確認できるdiveというツールを導入して、既存のコンテナイメージを分析してみます。
```bash
root@ip-10-0-1-112:~# wget https://github.com/wagoodman/dive/releases/download/v0.9.2/dive_0.9.2_linux_amd64.deb
root@ip-10-0-1-112:~# apt install ./dive_0.9.2_linux_amd64.deb
root@ip-10-0-1-112:~# dpkg -l | grep dive
ii  dive                             0.9.2                               amd64        no description given
root@ip-10-0-1-112:~# dive --version
dive 0.9.2
```

使い方は簡単で、解析したいコンテナイメージ名とタグ(option)をdiveコマンドの引数に指定するだけです。

> this can take a while for large images

とあるように、16.1GBのコンテナイメージの方は、解析に少し時間がかかります。。。

**tomcatアプリのコンテナイメージ**
{{< figure alt="img16" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img16.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img16.png">}}

**SpringBootアプリのコンテナイメージ**
{{< figure alt="img17" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img17.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img17.png">}}


# AWS製分析ツール導入(optimizeImage.sh)
少し分かりにくいのですが、ガイドラインページの最下部からツールをダウンロードできます。

https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/optimize-aws-app2container-generated-docker-images.html


{{< figure alt="img14" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img14.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img14.png">}}

```bash
# ツールをダウンロード
root@ip-10-0-1-112:~# wget https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/samples/p-attach/dc756bff-1fcd-4fd2-8c4f-dc494b5007b9/attachments/attachment.zip

#解凍
root@ip-10-0-1-112:~# unzip attachment.zip 
Archive:  attachment.zip
  inflating: sample_analysis.json    
  inflating: optimizeImage.sh        

#実行権限付与
root@ip-10-0-1-112:~# chmod +x optimizeImage.sh 

#確認
root@ip-10-0-1-112:~# ll
total 824540
(snip)
-rw-r--r--  1 root    root       3981 Jul 29  2021 sample_analysis.json
-rwxr-xr-x  1 root    root       2366 Jul 29  2021 optimizeImage.sh*
(snip)
```
以上でツールの導入は完了です。

# 分析(Tomcatアプリ)
「-d 1 -t / -s 1000000 -v」の結果、ディレクトリが/usrと/optのみとなっていることがわかります。
```bash
# ContainerFiles.tarを確認
root@ip-10-0-1-112:~# ll -h app2container/java-tomcat-5da060de/Artifacts/
total 152M
(snip)
-r-- 1 root root 152M Mar  6 06:17 ContainerFiles.tar

# TARGET設定
TAR_TARGET=./app2container/java-tomcat-5da060de/Artifacts/ContainerFiles.tar

# 全体サイズを確認
root@ip-10-0-1-112:~# ./optimizeImage.sh -p $TAR_TARGET -d 1 -t / -s 1000000 -v
./app2container/java-tomcat-5da060de/Artifacts/ContainerFiles.tar exists.
tar -Ptvf ./app2container/java-tomcat-5da060de/Artifacts/ContainerFiles.tar|tr -s ' '|cut -d ' ' -f3,6| awk '$2 ~/$/'| awk '$2 ~/^\//' |cut -f1-2 -d'/'|awk '{ if ($1>= 1000000) arr[$2]+=$1 } END { for (key in arr) { if(1) printf("%-50s\t%-50s\n", key, arr[key]) else printf("%s,\n", key) } } '|sort -k2 -nr
===============================================================================================
/usr                                                    142264753                                         
/opt                                                    4856216                                           

# 特定ディレクトリをドリルダウン
root@ip-10-0-1-112:~# ./optimizeImage.sh -p $TAR_TARGET -d 2 -t /usr -s 1000000 -v
./app2container/java-tomcat-5da060de/Artifacts/ContainerFiles.tar exists.
tar -Ptvf ./app2container/java-tomcat-5da060de/Artifacts/ContainerFiles.tar|tr -s ' '|cut -d ' ' -f3,6| awk '$2 ~/$/'| awk '$2 ~/^\/usr/' |cut -f1-3 -d'/'|awk '{ if ($1>= 1000000) arr[$2]+=$1 } END { for (key in arr) { if(1) printf("%-50s\t%-50s\n", key, arr[key]) else printf("%s,\n", key) } } '|sort -k2 -nr
===============================================================================================
/usr/lib                                                142264753                                         
```

# 分析(SpringBootアプリ)
一方、「-d 1 -t / -s 1000000 -v」の結果、ディレクトリが/usrと/opt以外にも多くのディレクトリが含まれていることがわかります。このディレクトリの中から、アプリ稼働に関係がないディレクトリを除外する事で、コンテナイメージのスリム化を行います。

```bash
# ContainerFiles.tarを確認
root@ip-10-0-1-112:~# ll -h app2container/java-generic-6ef9339e/Artifacts/
-rw-r--r-- 1 root root 7.7G Mar  6 06:27 ContainerFiles.tar
(snip)

# TARGET設定
root@ip-10-0-1-112:~# TAR_TARGET=./app2container/java-generic-6ef9339e/Artifacts/ContainerFiles.tar

# 全体サイズを確認
root@ip-10-0-1-112:~# ./optimizeImage.sh -p $TAR_TARGET -d 0 -t / -v
./app2container/java-generic-6ef9339e/Artifacts/ContainerFiles.tar exists.
tar -Ptvf ./app2container/java-generic-6ef9339e/Artifacts/ContainerFiles.tar|tr -s ' '|cut -d ' ' -f3,6| awk '$2 ~/$/'| awk '$2 ~/^\//' |cut -f1-1 -d'/'|awk '{ if ($1>= 0) arr[$2]+=$1 } END { for (key in arr) { if(1) printf("%-50s\t%-50s\n", key, arr[key]) else printf("%s,\n", key) } } '|sort -k2 -nr
===============================================================================================
                                                        7919225008  
# サイズの大きい順に/ディレクトリ以下を表示
root@ip-10-0-1-112:~# ./optimizeImage.sh -p $TAR_TARGET -d 1 -t / -s 1000000 -v
./app2container/java-generic-6ef9339e/Artifacts/ContainerFiles.tar exists.
tar -Ptvf ./app2container/java-generic-6ef9339e/Artifacts/ContainerFiles.tar|tr -s ' '|cut -d ' ' -f3,6| awk '$2 ~/$/'| awk '$2 ~/^\//' |cut -f1-2 -d'/'|awk '{ if ($1>= 1000000) arr[$2]+=$1 } END { for (key in arr) { if(1) printf("%-50s\t%-50s\n", key, arr[key]) else printf("%s,\n", key) } } '|sort -k2 -nr
===============================================================================================
/usr                                                    2412836670                                        
/var                                                    1319669369                                        
/root                                                   953114743                                         
/snap                                                   658552014                                         
/opt                                                    305844542                                         
/home                                                   283062895                                         
/lib                                                    32995202                                          
/sbin                                                   5033368                                           
/bin                                                    3175800  

# /usrディレクトリをドリルダウン
root@ip-10-0-1-112:~# ./optimizeImage.sh -p $TAR_TARGET -d 2 -t /usr -s 1000000 -v
./app2container/java-generic-6ef9339e/Artifacts/ContainerFiles.tar exists.
tar -Ptvf ./app2container/java-generic-6ef9339e/Artifacts/ContainerFiles.tar|tr -s ' '|cut -d ' ' -f3,6| awk '$2 ~/$/'| awk '$2 ~/^\/usr/' |cut -f1-3 -d'/'|awk '{ if ($1>= 1000000) arr[$2]+=$1 } END { for (key in arr) { if(1) printf("%-50s\t%-50s\n", key, arr[key]) else printf("%s,\n", key) } } '|sort -k2 -nr
===============================================================================================
/usr/lib                                                1048688240                                        
/usr/local                                              605362445                                         
/usr/bin                                                576847128                                         
/usr/libexec                                            114789284                                         
/usr/share                                              36225952                                          
/usr/sbin                                               27243896                                          
/usr/src                                                3679725

# /varディレクトリをドリルダウン
root@ip-10-0-1-112:~# ./optimizeImage.sh -p $TAR_TARGET -d 2 -t /var -s 1000000 -v                                                                                                                                                   
./app2container/java-generic-6ef9339e/Artifacts/ContainerFiles.tar exists.
tar -Ptvf ./app2container/java-generic-6ef9339e/Artifacts/ContainerFiles.tar|tr -s ' '|cut -d ' ' -f3,6| awk '$2 ~/$/'| awk '$2 ~/^\/var/' |cut -f1-3 -d'/'|awk '{ if ($1>= 1000000) arr[$2]+=$1 } END { for (key in arr) { if(1) printf("%-50s\t%-50s\n", key, arr[key]) else printf("%s,\n", key) } } '|sort -k2 -nr
===============================================================================================
/var/lib                                                709307201                                         
/var/swapfile                                           512000000                                         
/var/cache                                              114379497                      

root@ip-10-0-1-112:~# ./optimizeImage.sh -p $TAR_TARGET -d 2 -t /home -s 1000000 -v                                                                                                                                                  
./app2container/java-generic-6ef9339e/Artifacts/ContainerFiles.tar exists.
tar -Ptvf ./app2container/java-generic-6ef9339e/Artifacts/ContainerFiles.tar|tr -s ' '|cut -d ' ' -f3,6| awk '$2 ~/$/'| awk '$2 ~/^\/home/' |cut -f1-3 -d'/'|awk '{ if ($1>= 1000000) arr[$2]+=$1 } END { for (key in arr) { if(1) printf("%-50s\t%-50s\n", key, arr[key]) else printf("%s,\n", key) } } '|sort -k2 -nr
===============================================================================================
/home/ubuntu                                            292002225
```


# スリム化(SpringBootアプリ)
ここでは、アプリに関係が無いと想定される、下記4つのディレクトリを除外指定してみます。

**./app2container/java-generic-6ef9339e/analysis.json**
{{< figure alt="img18" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img18.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_app2container/img18.png">}}

再ビルドします。
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

5.4GB程、スリム化できました。
```bash
root@ip-10-0-1-112:~# docker images
REPOSITORY              TAG          IMAGE ID       CREATED             SIZE
java-generic-6ef9339e   latest       d54fccdd4dfc   3 minutes ago       10.7GB
```
