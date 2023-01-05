+++
Categories = ["AWS"]
Tags = ["AWS", "IoT", "ECS", "FreeRTOS"]
date = "2023-01-05T00:00:00+09:00"
title = "AWS FAN!"
+++

# はじめに
明けましておめでとうございます。2023年もスタートし、一発目の記事になります。私は普段、AWSを中心としたクラウド活用のソリューション設計を専門に仕事している一方、個人としてもAWSユーザとして、様々なクラウドサービスを楽しませていただいています。

本格的に触り始めたのは2018年頃からとだいぶ遅いスタートではありますが、少し振り返ってみようと思います。

※本コンテンツは2022年の年末、とあるAWSコミュニティーの「お祭りイベント」で公演させて頂いたものになります。

# スマートホーム(2018年〜2019年頃)
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img1.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img1.png">}}
{{< figure alt="img2" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img2.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img2.png">}}

会社の小集団活動も活用し、趣味のラズパイやArduinoといったマイコンいじりであれこれプロトタイピングを楽しんでいました。ただ、当時このシステムを作ったきっかけは、Vueを使ったフロントエンドの開発の勉強が目的でした。

{{< figure alt="img3" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img3.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img3.png">}}

モニター分離の体重計は、[こちら](https://www.amazon.co.jp/dp/B00A1ZU2XY)の商品を活用させて頂きましたが、現在Amazonでは「この商品は現在お取り扱いできません。」となっていますね。また、赤外線の信号パターンは自分で解読しましたが、当時先輩からは「メーカーに背景を伝えた上で仕様を聞いてみたら？」というアドバイスも頂きました。なんとなく稼働していたのでそれで良しと済ませましたが、今ならすぐに問合せしてみるかなと思います。

# オンラインゲーム(2020年頃)
{{< figure alt="img4" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img4.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img4.png">}}
{{< figure alt="img5" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img5.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img5.png">}}

詳細は「[我が家のマイクラサーバー(AWS編)](https://t-tkm.github.io/blog/posts/2021/03/minecraft_server_aws_ecs/)」へ。

# 音声アシスタント(2021年頃)
{{< figure alt="img6" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img6.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img6.png">}}
{{< figure alt="img7" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img7.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img7.png">}}

詳細は「[Alexaでかけ算ゲーム](https://t-tkm.github.io/blog/posts/2021/03/aws_alexa_practice/)」へ。

# AI・画像処理1(WIP!)(2021年秋)
{{< figure alt="img9" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img9.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img9.png">}}

待望のプロトタイピング集「[IoTデバイス×Webアプリでホームネットワーク AWS クラウドサービス開発テクニック](https://www.amazon.co.jp/dp/4798064289/)」、大変勉強になります！

{{< figure alt="img10" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img10.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img10.png">}}
{{< figure alt="img11" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img11.gif" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img11.gif">}}

こちらのプロトタイプも、AWSでの分析よりは、エッジでのAI分析(PyTorchとYOLOv3による物体検出)がメイン。ラズパイのCPUでは画像処理がおぼつかなく(1FPS程度)、Googleが設計・開発した[Coral USB Accelerator(Edge TPU coprocessor)](https://coral.ai/products/accelerator/)で5倍速するなど遊んでおりました(机上にUSBドングルが写っておりますが、これがCoral USB Accelerator)。

ただ、もともとやりたかった「子供達のデバイス利用時間(スクリーンの前にいる時間)計測」のユースケースでは、リアルタイム物体検出は不要で、定期的に写真を取得し分析すれば良いだけで済むものです。要件整理不十分、実装に先走り「あれ、そもそもリアルタイム物体検出不要だったな」と後から思い返した次第ですw

(ちなみに、USB Acceleratorを使わずラズパイ3のCPUで頑張ったケースは↓。解析がもたついております。)
{{< figure alt="img19" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img19.gif" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img19.gif">}}

# リアルタイムOS(WIP!)(2022年春)
{{< figure alt="img12" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img12.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img12.png">}}
{{< figure alt="img13" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img13.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img13.png">}}

こちらの書籍「[内藤春雄(著)「Arduinoで楽しむ鉄道模型 ~簡単なプログラムで信号機や踏切遮断機を動かす!](https://www.amazon.co.jp/gp/product/4774199192/)」をお手本にさせていただきました。

{{< figure alt="img14" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img14.gif" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img14.gif">}}
{{< figure alt="img15" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img15.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img15.png">}}

通常のNゲージは、線路に電気を流し、コントローラで電圧(0〜12V)を調整し車両のモーターを動かす「アナログ方式」です([Nゲージのしくみ](https://www.kato-start.com/blank-1))。これではクラウドから車両を制御といって、やれる事は制限されるのですが、最近では「DCC(Digital Command Control)」と呼ばれる車両にマイコン(デコーダーという)を載せて、コントローラで制御する方式があります。車両のデコーダーは2千円程度で購入でき、コントローラも例えば[DSshield2](https://desktopstation.net/wiki/doku.php/dsshield2)などでDIYできそうです。時間を見つけてチャレンジしてみたいですね！

{{< figure alt="img16" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img16.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img16.png">}}

その前に、FreeRTOS/OTAを理解を深める必要がありますが、AWS公式ドキュメントも充実しており、大変助かります。
- [FreeRTOS 無線通信経由更新](https://docs.aws.amazon.com/ja_jp/freertos/latest/userguide/freertos-ota-dev.html)
- [リアルタイムOS新時代に突入！クラウド接続もスタンドアロンも！Amazon×マイコン FreeRTOS入門](https://interface.cqpub.co.jp/magazine/202104/)

# (再)AI・画像処理(WIP!)(2022年秋)
{{< figure alt="img17" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img17.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img17.png">}}
{{< figure alt="img18" src="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img18.png" link="https://github.com/t-tkm/blog_images/raw/main/2022/aws_fan/img18.png">}}

現在のマイブームは天体観測。この世界はとても奥が深く、とても興味深いものです。自動化をクラウド(AWS)活用してと考えてみたものの、ちょっと的外れかと思い直しました。。。天体画像を処理するソフトウェア(OSS)もありますが、高度な画像処理は、Photoshopや、天体専用の商用ソフトが主流であり、OSSなどどの程度活用可能か、技術的に課題ありです。また、生データ(素材)の良さを保ちつつ、いかに画像処理を行うか、この辺は職人芸(アート)の世界でもあり、AI・ソフトウェアによるお任せニーズは低い気がします。活用ユースケースとしては、レントゲン写真から病気を抽出するといった医療分野などが適切な気がします。ただ、技術的には、本質は似ている点もありそうです^^

# さいごに
以上、私個人の趣味(マイコンいじり)と、AWSサービスとの関わりを振り返ってみました。もちろん、ここに挙げたサンプルは一部で、他にも魅力的なサービスは多々あります。クラウドの良さは、簡単にサービス・情報にアクセスでき、お試し(プロトタイピング)しやすい点にあります。これからも、いつでも、どこでも、簡単にアクセスできる利点を活かし、どんどん遊んでみたいと思います！