+++ 
Categories = ["AWS"] 
Tags = ["AWS", "CUR", "Cost", "FinOps", "Athena"] 
date = "2023-12-30T00:00:00+09:00" 
title = "AWS Cost Usage Reportの可視化(3) -生成AI SQL (Cube + LangChain)" 
archives = ["2023", "2023-12", "2023-12-30"]
+++

# はじめに
「[AWS Cost Usage Reportの可視化(2) -ヘッドレスBIツールCubeを試してみる](http://localhost:1313/blog/posts/2023/09/aws_cost_usage_report2/)」の続編になります。

今年のre:Invent(2023)も盛り沢山！まだまだ全体は把握できておりませんが、やはり、生成系AIのトピックが多かった印象です。
その中で、私は「Amazon Q generative SQL in Amazon Redshift」が目にとまりました。

「[AWS Black Belt Online Seminar re:Invent 2023アップデート速報](https://pages.awscloud.com/rs/112-TZM-766/images/AWS-Black-Belt_2023_reInvent2023digest_1201_v1.pdf)」(スライド#40)

下記ツイートは、(1年ほど前)ChatGPTを使い始めた頃のものです。当時は、本番サービスとして提供されるとは夢にも思いませんでした！
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report3/img10.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report3/img10.png">}}

このサービスは、ユーザが自然言語で問合せると推奨SQLクエリを生成してくれるというものですが、似たようなシステムのDIY可能でしょうか？少し調べてみたところ、Cubeの開発者ブログに、CubeとLangChainを統合する記事がありました。

そこで、お勉強も兼ね、この記事を参考に、生成AI SQLシステムをDIYしてみます。

(参考サイト)
[Introducing the LangChain integration](https://cube.dev/blog/introducing-the-langchain-integration)

# 全体構成
前回構築した構成ベースに拡張します。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report3/img1.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report3/img1.png">}}

# お試し(参考記事のサンプル)
まずは、参考記事に従って、デモを動かしてみます。デモシナリオは問題なく動きました。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report3/img2.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report3/img2.png">}}

英語でなく、他の言語で問合せしてみてはどうでしょう？問題なく動いてます。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report3/img3.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report3/img3.png">}}

タイミングによっては、失敗するケースもあります。この辺は、AI(モデル)を用いたシステムの難しさでもあります。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report3/img4.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report3/img4.png">}}

# システム構築
それでは、自宅システムに、このLangChainアプリを移植(DIY)してみます。

(GitHub) https://github.com/t-tkm/aws_cru_langchain

**1. Cube起動**

詳細は、前回の記事「[AWS Cost Usage Reportの可視化(2) -ヘッドレスBIツールCubeを試してみる](https://t-tkm.github.io/blog/posts/2023/09/aws_cost_usage_report2/)」のStep1を参考にして下さい。

はじめに、起動パラメータを準備しておきます。
```zsh
touch .env
```

.envファイルの中身はこのようになります。
```file
CUBEJS_AWS_KEY=<AWSアクセスキー>
CUBEJS_AWS_SECRET=<AWSシークレットキー>
CUBEJS_AWS_REGION=ap-northeast-1
CUBEJS_AWS_S3_OUTPUT_LOCATION=s3://<S3バケット名>
CUBEJS_DB_TYPE=athena
CUBEJS_EXTERNAL_DEFAULT=true
CUBEJS_SCHEDULED_REFRESH_DEFAULT=true
CUBEJS_DEV_MODE=true
CUBEJS_SCHEMA_PATH=model
```

環境変数が準備できたので、cubeコンテナを起動します。
```zsh
docker compose up -d
```

ブラウザでcubeへ接続し(http://localhost:4000/)、(Athenaの)モデル作成します。

**2. LangChainアプリ起動**

python環境を準備します。ventやvirtualenvなど、お好みのツールでok。ここでは、(筆者の好みで)conda(Anacondaディストリビューション)を使いました。
```zsh
(base)$ conda create -n aws_cur_langchain phthon=3.1
(base)$ conda activate aws_cur_langchain

# アプリに必要なpythonライブラリ群をインストール
(aws_cur_langchain)$ pip install -r requirements.txt
```

次に、OpenAI APIの(シークレット)キーを取得します(ex. sk-XXXXXXXXXXX)。ここでは詳細は省略します。
また、Cube APIのシークレットキーも取得しておきます。ブラウザでCubeアクセスし、「Playground」->「Code」から取得できます。
```
(snip)
const cubejsApi = cubejs(
'{ヘッダ}.{ペイロード}.{署名}',
{ apiUrl: 'http://localhost:4000/cubejs-api/v1' }
);
(snip)
```  
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report3/img11.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report3/img11.png">}}

OpenAI APIと、Cubeのそれぞれの(シークレット)キーが準備できたら、.envファイルに次のように追加します。

```file
OPENAI_API_KEY=<OpenAI APIのキー>
CUBE_API_URL=http://localhost:4000/cubejs-api/v1
CUBE_API_SECRET=<Cube APIシークレットキー>
#DATABASE_URL=postgresql://localhost:4000/test
DATABASE_URL=host=localhost port=15432 dbname=test user=username password=password
```

準備ができたら、アプリを起動します。
```zsh
(aws_cur_langchain)$ streamlit run streamlit_app.py
``` 
初回起動時は、Ingestion処理のため(「Loading context from Cube API...」)5分程度待ちます。

GUIが表示されたらクエリを投入してみましょう(ex. pricing_public_on_demand_costの合計は？)。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report3/img9.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report3/img9.png">}}

一応動いていますが、このコードでは殆ど使い物になりません^^;。カラム名の名前解決など、チューニングが必要です！

最後に、コードの動きを少しだけ覗いておきたいと思います。

# デバッグ
アプリ初回起動時は、ローカルディスクに「vectorstore」がありません。Ingestion処理により、vectorestoreが構築されます。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report3/img5.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report3/img5.png">}}

VSCodeで中身を確認します。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report3/img7.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report3/img7.png">}}

ローカルディスクに「vectorestore」がある場合(2回目以降など)、Ingestion処理はスキップされます。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report3/img6.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report3/img6.png">}}

VSCodeで中身を確認します。RAGの仕組みが良くわかります。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report3/img8.png" link="https://github.com/t-tkm/blog_images/raw/main/2023/aws_cost_usage_report3/img8.png">}}

# まとめ
Cubeを使ったAWSコスト分析システムに、LangChainを組合せた生成AIシステムを構築してみました。
今回は、実用レベルにはほど遠い(AI)精度でしたが、LangChainを用いたLLMを活用する処理(RAGと呼ばれるアプローチ)を
理解することができました。引き続き、実用に耐えるレベルの精度向上に向け、チャレンジしていこうと思います。