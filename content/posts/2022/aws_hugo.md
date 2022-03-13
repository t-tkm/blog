+++ 
Categories = ["AWS"] 
Tags = ["AWS", "Hugo"] 
date = "2022-03-12T00:00:00+09:00" 
title = "Hugoでハンズオンサイト(AWS)" 
+++

# はじめに
唐突ですが、AWS Workshopsというサイトをご存知でしょうか？

AWS Workshopsは、AWSが公式に提供しているハンズオンコンテンツで、AWS各種サービスについて、実際に手を動かしてみたい場合に大変重宝するサイトです。

https://workshops.aws/


さてこのサイト、とてもお洒落な作りになっていますが、どのように作られているのでしょうか？

答えは、Hugoと言われるGo製の静的サイトジェネレータと、Learnと呼ばれるテーマ(Theme)を使って作られているようです。
- https://gohugo.io/
- https://jamstackthemes.dev/theme/hugo-theme-learn/

実はこのAWS Workshopsに。このようなハンズオンサイトの作り方のハンズオンがあります！折角なので、試してみたいと思います。

https://hosting-hugo-content.workshop.aws/

{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/blob/aws_hugo/imgs/img1.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/aws_hugo/imgs/img1.png?raw=true">}}

※ハンズオンの主目的は、Hugoの使い方でなく、AWS Amplifyを活用したホスティングサイト構築手法(CICD込み)ですw

# Hugoセットアップ
詳細はハンズオンを見てもらうとして、Cloud9を使っていきます。初めにHugoをインストールします。Goで作られたシングルバイナリなので、ダウンロード後に解凍し、パスの通ったディレクトリへ実行ファイルを置くだけで利用できます。
```bash
$ wget https://github.com/gohugoio/hugo/releases/download/v0.71.0/hugo_extended_0.71.0_Linux-64bit.tar.gz -O hugo_extended_0.71.0_Linux-64bit.tar.gz
$ HUGO_TAR="$(find . -name "*Linux-64bit.tar.gz")"
$ tar -xzf $HUGO_TAR
$ chmod +x hugo
$ mkdir bin
$ mv hugo bin/
$ cd ~/environment/

# Hugoがインストールできているかの確認
$ hugo version
Hugo Static Site Generator v0.71.0-06150C87/extended linux/amd64 BuildDate: 2020-05-18T16:15:29Z
```

# サンプルサイトの確認
```bash
# 事前に、Hugoで作ったサンプルサイトをダウンロードします
$ git clone -b v1.0 --single-branch https://github.com/t-tkm/aws-amplify-hugo-hosting.git && cd aws-amplify-hugo-hosting

# 適当にブランチ名をつけておく。
$ git switch -c hugo

# themesの中身は空なので、新たに取得する
$ git submodule update --init
```

Hugoでサーバーを起動して、サイトを確認します。Hugo Serverはデフォルトで1313ポート番号を使いますが、Cloud9のPreviewを使うため、ポート番号として8080を指定してサーバを起動します。
```bash
$ hugo server -p 8080
```

{{< figure alt="img2" src="https://github.com/t-tkm/blog_images/blob/aws_hugo/imgs/img2.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/aws_hugo/imgs/img2.png?raw=true">}}

ちゃんと表示されました。

{{< figure alt="img3" src="https://github.com/t-tkm/blog_images/blob/aws_hugo/imgs/img3.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/aws_hugo/imgs/img3.png?raw=true">}}


# Amplifyセットアップ
サンプルサイトはローカルで動いているため、公開サイトとしてホスティングさせます。そこで、AWS Amplifyを使って簡単構築してみます。

Amplify CLIをインストールします。Cloud9はデフォルトでNode.jsがインストール済みなので、npmでパッケージインストールするだけです。
```bash
$ npm install -g @aws-amplify/cli@4.21.1

# インストールできているかの確認
$ amplify -v
Scanning for plugins...
Plugin scan successful
4.21.1
```

以上でAmplify CLIを使う準備が整いました。

# Amplifyでサイトを公開
Amplifyを初期化します。
```bash
# Initialize AWS Amplify
$ amplify init
```

設定パラメータは下記

{{< figure alt="img5" src="https://github.com/t-tkm/blog_images/blob/aws_hugo/imgs/img5.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/aws_hugo/imgs/img5.png?raw=true">}}

```bash
$ amplify add hosting
```

設定パラメータは下記

{{< figure alt="img6" src="https://github.com/t-tkm/blog_images/blob/aws_hugo/imgs/img6.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/aws_hugo/imgs/img6.png?raw=true">}}

```bash
$ amplify status
Current Environment: main
| Category | Resource name   | Operation | Provider plugin   |
| -------- | --------------- | --------- | ----------------- |
| Hosting  | S3AndCloudFront | Create    | awscloudformation |
```

以上により、プロジェクトファイルの各種設定がAmplify CLIで設定されました。次に、この情報に基づき、AWSリソースを実際に生成します。
{{< figure alt="img7" src="https://github.com/t-tkm/blog_images/blob/aws_hugo/imgs/img7.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/aws_hugo/imgs/img7.png?raw=true">}}

```bash
$ amplify publish
✔ Successfully pulled backend environment main from the cloud.

Current Environment: main

| Category | Resource name   | Operation | Provider plugin   |
| -------- | --------------- | --------- | ----------------- |
| Hosting  | S3AndCloudFront | Create    | awscloudformation |
? Are you sure you want to continue? Yes

(snip)

✔ All resources are updated in the cloud

Hosting endpoint: https://<<ランダム文字列>>.cloudfront.net

                   | EN  
-------------------+-----
  Pages            | 19  
  Paginator pages  |  0  
  Non-page files   |  1  
  Static files     | 76  
  Processed images |  0  
  Aliases          |  0  
  Sitemaps         |  1  
  Cleaned          |  0  

Total in 183 ms
frontend build command exited with code 0
Publish started for S3AndCloudFront
✔ Uploaded files successfully.
Your app is published successfully.
https://<<ランダム>>.cloudfront.net

$ amplify status

Current Environment: main

| Category | Resource name   | Operation | Provider plugin   |
| -------- | --------------- | --------- | ----------------- |
| Hosting  | S3AndCloudFront | No Change | awscloudformation |

Hosting endpoint: https://<<ランダム文字列>>.cloudfront.net
```

amplify publishコマンドの裏では、CloudFormationスタックが実行されAWSリソースが生成されます。

{{< figure alt="img4" src="https://github.com/t-tkm/blog_images/blob/aws_hugo/imgs/img4.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/aws_hugo/imgs/img4.png?raw=true">}}

# 公開サイト(CloudFront)の確認
無事、公開を確認できました。
{{< figure alt="gif" src="https://github.com/t-tkm/blog_images/blob/aws_hugo/imgs/gif.gif?raw=true" link="https://github.com/t-tkm/blog_images/blob/aws_hugo/imgs/gif.gif?raw=true">}}

# 補足
AWS Workshopsのガイドにも記載ありますが、US-EAST-1以外のリージョンだと、DNSが浸透するまで時間がかかり、その間はAccessDeniedになります。

>Note: If you deploy this lab in any region other than US-EAST-1 it can take up to a few hours for DNS to replicate which will cause an AccessDenied error to be displayed. This is just for the first time the S3 bucket is created. Any future updates will not have a delay.

https://aws.amazon.com/jp/premiumsupport/knowledge-center/s3-http-307-response/

素直に待つのが良いと思いますが「待ちきれない！」という方は、Amplifyの設定ファイルを少しいじって、次のようなやり方で試してみてもいいかもしれません。

{{< figure alt="img8" src="https://github.com/t-tkm/blog_images/blob/aws_hugo/imgs/img8.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/aws_hugo/imgs/img8.png?raw=true">}}
{{< figure alt="img9" src="https://github.com/t-tkm/blog_images/blob/aws_hugo/imgs/img9.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/aws_hugo/imgs/img9.png?raw=true">}}
