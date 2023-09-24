+++
Categories = ["AWS"]
Tags = ["AWS", "Alexa"]
date = "2021-03-26T00:00:00+09:00"
title = "Alexaでかけ算ゲーム"
archives = ["2021", "2021-03", "2021-03-26"]
+++

# Alexaでかけ算ゲーム
# はじめに
今年の正月は、コロナ禍ということで帰省ができなかったため、以前より気になっていたAlexaに手を出してみました。あわよくば、今回の勉強を通して「Alexa認定」も取得できるかも？と思ったのですが、そこまで甘い世界ではありませんでした[^1]。。。

# Alexa全体像の理解
さて、実際始めようとすると、今まで公私共に全く触った事の無いサービスという事もあり、どこから手をつけてよいものかわかりません。。。そこでとっかかりとして、[Alexa公式 動画シリーズ「Alexa道場」](https://developer.amazon.com/ja-JP/alexa/alexa-skills-kit/get-deeper/webinars)から始めることにしました。こちらの教材がとても丁寧で、ストーリを追って段階的に学習できお勧めです！一通り動画を視聴し終わると、なんとなく(開発含めた)全体像が把握できると思います(とりあえずスキルを作ってみたいというのであれば、Season1〜3で十分)。

#スキル開発
では早速開発、と言ってもゼロベースでコードを作成するのは流石に大変ですで、お手本になるサンプル探しを開始。そこで気がついた事は、Alexaのスキル自体はたくさん公開されているのですが、コードはさほどネットには落ちてない感じでした(私の探し方が十分で無い可能性もございます)。そこで見つけたのが、クジラ飛行机さんの[やさしくはじめる スマートスピーカープログラミング](https://www.amazon.co.jp/dp/4839967229)。このサンプルをベースに、スキル作成にチャレンジさせて頂きました。

# ユースケース
ちょうど、小学２年の息子が「冬休みの宿題」として九九の練習を持ち帰ってきており、親が問題を出すという箇所を、Alexaにお手伝いしてもらうことにしました。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/blob/main/2021/03/aws_alexa_practice/img1.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2021/03/aws_alexa_practice/img1.png?raw=true">}}

実際のアクター(息子)とシステム(Alexa)のやりとりは↓
{{< figure alt="img2" src="https://github.com/t-tkm/blog_images/blob/main/2021/03/aws_alexa_practice/img2.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2021/03/aws_alexa_practice/img2.png?raw=true">}}

# システム概要
今回はじめて知ったのですが、AlexaサービスはAWSでなくAmazonなのですね。また、Alexaを開発するコンソールも、日々進化しているようで、書籍・記事によっては追い付けていないものも多いと思われますので、最新がどうなっているのか確認が必要になります(Alexaに限らず、クラウドサービス全般に言えることではございます)。

{{< figure alt="img3" src="https://github.com/t-tkm/blog_images/blob/main/2021/03/aws_alexa_practice/img3.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2021/03/aws_alexa_practice/img3.png?raw=true">}}

# サンプル(lambda)
今回作成したコードは、こちらに置いております。
https://gitlab.com/t-tkm/alexa-kakezan-game

```index.js
const Alexa = require('ask-sdk');
const TABLE_NAME = 'myScore';

let skill;
exports.handler = async function (event, context) {
  if (skill) return skill.invoke(event);
  skill = Alexa.SkillBuilders.standard()
    .addRequestHandlers(
      LaunchRequestHandler,
      NumberIntentHandler,
      CancelIntentHandler)
    .withTableName(TABLE_NAME)
    .withAutoCreateTable(true)
    .create();
  return skill.invoke(event);
};

const TITLE = 'かけ算ゲーム';

function getResponse(h, msg) {
  return h.responseBuilder
    .speak(msg).reprompt(msg).withSimpleCard(TITLE, msg)
    .getResponse();
}

async function getHighScore(h) {
  const pa = await h.attributesManager
    .getPersistentAttributes();
  // スコア初期化
  if (!pa.highScore) {
    pa.highScore = 0;
    setHighScore(h, pa.highScore);
  }
  console.log('high score=' + pa.highScore);
  return pa.highScore;
}

async function setHighScore(h, value) {
  const pa = await h.attributesManager
    .getPersistentAttributes();
  pa.highScore = value;
  h.attributesManager.setPersistentAttributes(pa);
  await h
    .attributesManager
    .savePersistentAttributes();
}

const LaunchRequestHandler = {
  canHandle(h) {
    const type = h.requestEnvelope.request.type;
    return (type === 'LaunchRequest');
  },
  async handle(h) {
    const attr = h.attributesManager.getSessionAttributes();
    attr.score = 0;   // 初期点数設定
    const highScore = await getHighScore(h);
    attr.one = Math.floor(Math.random() * 10) + 1;
    attr.two = Math.floor(Math.random() * 10) + 1;
    let question_str = String(attr.one) + 'かけ' + String(attr.two) + 'は？';
    return getResponse(h, 
      TITLE + 'にようこそ。' + 
      'これまでのスコアは、' + highScore + '点です。' +
      question_str);
  }
};

const NumberIntentHandler = {
  canHandle(h) {
    const type = h.requestEnvelope.request.type;
    const name = h.requestEnvelope.request.intent.name;
    return (type === 'IntentRequest') && 
           (name === 'NumberIntent');
  },
  async handle(h) {
    const attr = h.attributesManager.getSessionAttributes();
    const highScore = await getHighScore(h);
    const answer = attr.one * attr.two;
    const req = h.requestEnvelope.request;
    const user = parseInt(req.intent.slots.num.value);
    let msg = '';
    if (answer === user) {
      attr.score += 1;
      msg += '正解！';
    } else {
      msg += 'はずれです。';
      msg += '正解は、' + answer + 'でした。';
      attr.score -= 1;
    }
    msg += '現在、' + attr.score + '点です。';

    attr.one = Math.floor(Math.random() * 10) + 1;
    attr.two = Math.floor(Math.random() * 10) + 1;
    let question_str = String(attr.one) + 'かけ' + String(attr.two) + 'は？';
    msg += question_str;
    return getResponse(h, msg);
  }
};

const CancelIntentHandler = {
  canHandle(h) {
    const type = h.requestEnvelope.request.type;
    const name = h.requestEnvelope.request.intent.name;
    return (type === 'IntentRequest') && 
           (name === 'AMAZON.CancelIntent' ||
            name === 'AMAZON.StopIntent')
  },
　async handle(h) {
    const attr = h.attributesManager.getSessionAttributes();
    const highScore = await getHighScore(h);
    const result_score = attr.score + highScore;
    await setHighScore(h, result_score);
    const msg = '終わります。' + 
    '今回' + attr.score +  '点取得しました。' + 
    '現在までの累積スコアは、' + result_score + '点です。';
    return h.responseBuilder.speak(msg).getResponse();
  }
}
```

# 最後に
今回初めてVUI(Voice UI)というものに触れてみたのですが、使い方次第でいろいろな可能性(DX)をもった技術だと、再認識しました。思いつくままに列挙しますとこんな感じになります:

* Amazon側で準備してくれている「音声認識」が凄い！
    * 以前AIの「自然言語処理(NPL)」をかじった事があるのですが、コンピュータに人間が話す言葉を理解させる事は本当に大変です。今回その難しさは、利用者・開発者から隠蔽されており、純粋に(アプリ)ロジックだけ考えればよい点が大変魅力的でした。
    * ただ息子は、Alexaがたまに発音を聞き取れず「申し訳ございません。意味がよく理解できませんでした。」とのレスポンスに「Alexa、頭悪い！」と。。。ユーザ(利用者)の要求水準はとても高いw！
* 品質をあげるための工夫は大変
    * 個人の趣味程度であれば、品質はそこそこでも許されるかもしれませんが、例えば「業務に活用」となると、今までGUIなどで提供していたインタフェース以上に例外処理の作り込みが大変な気がします。VUIといった新しい分野のキャッチアップの必要性を認識しました。音声でなくとも、すでにボイスチャットという形で業務活用は始まっていますので、後戻りはできない感触。業務想定での設計につきましては、下記も参考になるかと思われます。
    * [Voice User Interface設計　本格的なAlexaスキルの作り方](https://www.amazon.co.jp/dp/4822292592)
* 今まで以上にUXが重要
    * (ブラウザの画面などのように)GUIだとある程度入力を制限できると思うのですが、言葉によるやりとりは想定範囲が広く、またユーザ体験に直結するので設計は難しいなと思いました。今回息子に試してもらったのですが、弟が横から割り込んで邪魔ばかりして(わざと間違えを言って点数を下げる^^;)使い物になりませんでした。本人だけの音声を認識させる、など、色々と検討すべき事が多いかと想像します。
* クジラ飛行机さんの書籍では、Alexa(Amazon)とGoogleHome(GCP)の両サンプルがございます。それぞれのデバイスを購入するのも手ですが、より中身の理解を深める方法としてDIYという手もあります。私はこちらの方が興味あり笑
    * [ラズベリー・パイで作るAIスピーカ[プリント基板付き]](https://www.amazon.co.jp/dp/4789850315)
    * {{< figure alt="img4" src="https://github.com/t-tkm/blog_images/blob/main/2021/03/aws_alexa_practice/img4.jpeg?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2021/03/aws_alexa_practice/img4.jpeg?raw=true">}}

[^1]: 「AWS認定 Alexa スキルビルダー-専門知識」が2021.3月に終了するとの事もあり、このサンプル作成(勉強)を通して認定をゲットできないかとの思いもあったですが、現実はそこまで甘いものではありませんでした^^;。Webサイトの各種記事の中には「Alexa認定は比較的取得が容易」とのコメントも見受けられるのですが、個人的には、ちゃんと体系立てて学習しないと難しい試験(だったの)かと思います。今回このレベルの実装を少しかじる程度で(2週間くらい)、認定試験のスコアはおおよそ半分くらいでした。
