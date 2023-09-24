+++
Categories = ["AWS"]
Tags = ["AWS", "ECS"]
date = "2021-03-21T00:00:00+09:00"
title = "我が家のマイクラサーバー(AWS編)"
archives = ["2021", "2021-03", "2021-03-21"]
+++

# はじめに
マイクラサーバ(javaアプリ)をECS+Fargate+EFSで動かしてみた備忘録になります。趣味であるこのシステムの運用を通じて、今後周辺ノウハウを強化していきたいというのが(私、インフラエンジニア)目的ですw

我が家はマイクラのヘビーユーザで、息子3人(小学生)と母親が、同じ世界(マイクラではワールドと呼ぶ)にログインして、各々の建築を見せあったり、道具を揃えて洞窟を冒険したり楽しんでいます。私はそのマルチサーバをお守りしており、以前はVM上にサーバを構築して運用していました。しかし、1つのサーバで展開できるワールドは1つであり、色々複数のワールドを家族に提供するには、複数サーバを立てる必要があります。そこで、最近ではアプリをitzg氏が作成・メンテしているコンテナ[^1]を使い、複数ワールドをフットプリント軽く運用していました。

# ECS+Fargateに目をつけた理由
VMにDockerをインストールして、複数コンテナを運用していたのですが「Fargateを使えばVMのメンテが不要になるのでは？」と漠然と考えていました。しかし、マイクラサーバは、ワールドデータを自身のディスク(ファイルシステム)に永続化する必要があるため、Fargateでは諦めていたところ、[2020.4月にFargateがEFSをサポート！](https://aws.amazon.com/jp/about-aws/whats-new/2020/04/aws-fargate-launches-platform-version-14/)というアナウンスを受け、これは試すしかないと思ったのがきっかけになります。
※試してみたのは昨年4月なのですが、今頃記事を書いているのは単に私の怠慢ですw

# 注意
* サーバー運営で発生するリスクは全て自己責任でお願いします。Mineraftに限る話ではないですが、サーバ構築・動作には様々なリスクがあります。自身が何をしているのか十分理解する必要があります。
* また、Minecraftの利用規約やガイドラインについても確認するようにして下さい(例えば[こちら](https://www.minecraft.net/ja-jp/download/server/))。規約にある通り、全般的に個人で楽しむ使い方には好意的に見受けられますが、一方でビジネス(お金儲け)についての利用は厳しく線引き(制限)されています。

# システム概要
マイクラは今や世界的に展開されいるゲームであり、対応プラットフォームもPCだけでなく様々です。ここで記載するのはPCのJava版と呼ばれるエディションによる、クライアント・サーバ形式のものを使い、そのサーバ部分をECS(Fargate)化してみよう、というのがゴールになります。クライアント(プレイヤー)の方は、我が家では[こちらのサイト](https://www.minecraft.net/ja-jp/store/minecraft-java-edition)から、5名分のアカウント(3,000円/人、買切り〜2020.4月地点〜)を購入して楽しんでいます[^2]。

{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img1.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img1.png?raw=true">}}

今回デプロイしてみた構成図は下記になります。
{{< figure alt="img2" src="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img2.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img2.png?raw=true">}}

#各種設定概略
以下、概略程度になりますが、各種サービス設定と、その時のポイントを共有したいと思います。

## ELBの設定
* マイクラサーバはHTTP(S)などL7プロトコルでなく、ポート番号25565を使った独自プロトコル(Java RMI?)になるため、ELBもNLB(L4)を使います
* 外部からデフォルトポート番号でアクセスさせるのは少し気になったので、外からは50000で受付け、それをアプリ(25565)に転送させています
{{< figure alt="img3" src="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img3.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img3.png?raw=true">}}

## EFSの設定
* 特段な事は行わず、素直な設定でいきました
{{< figure alt="img8" src="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img8.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img8.png?raw=true">}}

## ECSタスクの設定
* コンテナイメージは、itgz氏の下記を利用しました
    * https://hub.docker.com/r/itzg/minecraft-server
* タスクサイズは、CPUとして2vCPU(1024x2)と、メモリ4GiBを確保。
    * 今回は、タスクの中に定義するコンテナは1つなのでこれで十分ですが、ロギング用などサイドカーコンテナも一緒に入れておくなど1タスクにに複数コンテナを定義する場合は、個別コンテナに対してリソース制限を設定する事も可能です。ただし、AWSドキュメントにも記載されている通り、あまり神経質になる必要はないようです[^3]。
* [コンテナの使い方](https://github.com/itzg/docker-minecraft-server)にもある通り、コンテナ起動時に環境変数「EULA=TRUE」が必要です。
* アプリデータの永続化に、先に作成したEFSのファイルシステムを指定したボリュームを指定します。アプリ(コンテナ)内のマウントポイントは「/data」です。

{{< figure alt="img4" src="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img4.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img4.png?raw=true">}}
{{< figure alt="img5" src="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img5.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img5.png?raw=true">}}

## ECSサービス設定
* 2020.4月の段階では、プラットフォームバージョンを「latest」にするとFargateは「1.3.0」で起動し、EFSマウント機能(1.4.0)が使えなかったので明示的に指定していました。最近latestが1.4.0になったようですので↓、今後はlatestで良いのかもしれません。
    * https://aws.amazon.com/jp/about-aws/whats-new/2021/03/aws-fargate-updates-platform-version-1-4-0-to-be-the-latest-version/
* マイクラサーバ(アプリ)はシングルサーバ構成であるため、タスク数は1になります。タスク数を2以上にして複数コンテナを立ち上げようとすると、アプリ側で起動失敗するようになっているようです。**コンテナ化によるオートスケールの恩恵を受けたい場合、アプリもステートレスなどでスケールできる作りにしておく必要がある。**という、分かりやすい例かと思いました。
    * {{< figure alt="img12" src="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img12.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img12.png?raw=true">}}


{{< figure alt="img6" src="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img6.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img6.png?raw=true">}}
{{< figure alt="img7" src="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img7.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img7.png?raw=true">}}

## 自動起動・停止ジョブ(lambda)
* コンテナの起動、停止もサービス経由で宣言的に操作できるのが嬉しいですね♪
    * [宣言的？ Declarative?どういうこと？](https://qiita.com/Hiroyuki_OSAKI/items/f3f88ae535550e95389d)
* スケジュールはcronフォーマットで指定しますが、いつものようにUTC指定であることに注意が必要です。私の場合は、15:00〜21:00(JST)でスケジューリングしているので、指定は「cron(0 6 * * ? *)〜cron(0 12 * * ? *)」となっています
* このスケジューリングではまだまだ無駄が多く、もう少しダイナミックに細かく制御したいと思っています。それこそAlexaを活用し、家族が遊ぶ時に起動・停止できるようにするのも一案と考えてます(このlambdaを特定の発話(ex. 「起動」「停止」)に対して呼び出すだけなので、さほど手間ではないかもしれません)。参考→[Alexaでかけ算ゲーム](https://qiita.com/t-taku/items/f009aefcc7267755ab44)

{{< figure alt="img9" src="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img9.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img9.png?raw=true">}}
{{< figure alt="img10" src="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img10.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img10.png?raw=true">}}

# 最後に
本当に、たったこれだけのシンプルな設定で、今後VMのお守りから開放されるメリットは大きいと感じました。一方(どんなPaaSにも共有でいえることですが)隠蔽されている部分が当然ありますので、その辺の見極め(お作法整備)は重要になってくると思います。

例えばですが、このシステムではコンテナの起動に数分かかりました(アプリケーションの初期化処理-ブートストラップは除く)。通常、ローカルでコンテナを起動する場合は、一度ローカルにpullしてきたコンテナイメージはローカル(OS)にキャッシュされるため、次回以降の起動は一瞬なのですが、Fargateの場合、ローカルキャッシュは無く、毎回コンテナレジストリにpullしにいっているようです。ですので、サイズの巨大なイメージを、頻繁に毎回外部レジストリに取りにいくような設計をしていますと、性能、コスト面で問題になる可能性があります。こういう細かな所は、座学だけでのキャッチアップは難しく、実際に動かしながら学んでいくのが効率的だなと常々感じているところです。

最後になりますが、このシステムでざっくりコストは70円/日くらいのようでした(すいません、細かくみておらず、他で使っているサービスもあるのですがw)。以前AWSトレーニングの中で、Capacity Providerを使う設定や、Cost Savings Planを活用すると結構なコスト低減が可能と伺がいました。今後「コスト最適化」観点でも、まだまだ勉強する事は沢山ありそうです☆

{{< figure alt="img11" src="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img11.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2021/03/minecraft_server_aws_ecs/img11.png?raw=true">}}


[^1]: https://hub.docker.com/r/itzg/minecraft-server
[^2]: 今回のようにサーバを立てなくとも、マイクラ自体は個々のPCやデバイスローカルで楽しむこともできます(「シングルプレイ」と呼ばれ、実際はそちらの方が多いのかもしれません)。
[^3]: https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/AWS_Fargate.html
      > Fargate の Amazon ECS タスク定義では、CPU とメモリをタスクレベルで指定する必要があります。Fargate タスクのコンテナレベルで CPU とメモリを指定することもできますが、これはオプションです。ほとんどのユースケースでは、タスクレベルでこれらのリソースを指定するだけで十分です。