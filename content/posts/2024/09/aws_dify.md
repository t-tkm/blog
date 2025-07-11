+++ 
Categories = ["AWS"] 
Tags = ["AWS", "ECS", "Party Rock", "Dify", "Generative AI"] 
date = "2024-09-29T21:34:00+09:00" 
title = "ベイズの定理を活用する！Party RockとDifyで複数のLLMを比較検証" 
archives = ["2024", "2024-09", "2024-09-28"]
+++

# 概要
本記事では、近年注目を集めるLLMの性能を比較検証し、生成AIの活用可能性を探ります。
特に、ベイズの定理のような統計的な問題に対するLLMの回答精度に焦点を当て、実用的な側面から
考察します。

以前(今から1年半ほど前)、こちらの記事に「[ChatGPTによる認知バイアスエラー回避の可能性](https://t-tkm.github.io/blog/posts/2023/05/essay_cognitive_psychology/)」で、
合理的な判断を助けるために生成AIを活用しよう、ただし、生成AIは計算が苦手なようで、正しい回答を得るため
には生成AIへの問合せ(プロンプト)に工夫が必要だと述べました。

```
一方、現時点では、 「問合せ(文)の仕方」やタイミングによって、正しい答えが得られないケースもある
ことを確認できました。従って、ChatGPT利用には適切な 「問い」を投げかける事と、
回答の妥当性を正しく評価する事が大事だという点も合わせて確認できました。
```

```
ChatGPTは、英語の学習データで訓練されているため、英語で質問するのが良いとも言われています。
```

更に今では、LLMモデルもバリエーションが豊富になり、かつ洗練されており、適材適所で利用者が使うモデルを
好きなように選択できるなど、これらの懸念が大幅に軽減されています。

ここでは、およそ1年半ぶりに、複数のLLMで同じ問いかけを行い、回答精度の改善を
体験してみたいと思います。

```
【問い】
夜、1 台のタクシーがひき逃げをしました。この市では、緑タクシーと青タクシーの 2 社が営業しています。
事件とタクシー会社については、次の情報が与えられています。
・市内を走るタクシーの 85%は緑タクシーで、15％が青タクシーである。
・目撃者は、タクシーが青だったと証言している。裁判所は、事件当夜と同じ状況で目撃者の信頼性をテストした結果、
この目撃者は青か緑かを 80％の頻度で正しく識別し、20％の頻度でまちがえた。
では、ひき逃げをしたのが青タクシーである確率は何％でしょうか？

【正解】
41.38%

【ポイント】
直感では80%と答える人が多いそうです。実は1つ目の情報「市内を走るタクシーの 85%は緑タクシーで、
15％が青タクシーである」が結果に大きく関与するが、直感ではこの情報を無視しがち、という
認知バイアスのため結果が大きくズレる。
→なので、こういう統計的な判断に、コンピュータ(生成AI)を活用しましょう、というお話でした。
```

以降の進め方: 
- 本記事では、複数のLLMモデルによる回答を比較検証し、生成AIの進化と実用性を検証します
- AWS Party RockとDifyという2つのノーコードAIアプリ開発プラットフォームを用います
- DifyをAWS上にクラウドネイティブな形で構築する、参考プロジェクトを紹介します

# Party Rockでの実験
[AWS Party Rock](https://partyrock.aws/)は、Amazon Bedrockを基盤とした生成AIアプリケーション
開発プラットフォームです([Amazon Bedrock プラットフォーム Party Rock を発表](https://aws.amazon.com/about-aws/whats-new/2023/11/partyrock-amazon-bedrock-playground/))。

プログラミングの知識がなくても、自然言語で指示を入力するだけでアプリを作成できる
ユーザーフレンドリーな設計が特徴で、ウィジェットを組み合わせてアプリを構築し、
画像生成やテキスト生成など多様な機能を活用できます(Party Rockに限らず、昨今、本当にノーコード
ツールが洗練され、充実してきていますね。)

無料で利用可能で、AWSアカウントやクレジットカードの登録が不要というのも、使用のハードルを大きく
下げています。

作成したアプリはURLを通じて簡単に共有でき、他のユーザーも利用可能で、Amazon Bedrockを基盤に、
多様な生成AIモデルから選択して使用できます。もちろん、データプライバシーにも配慮され、
ユーザーはデータのオプトアウトが可能とのこと。

まずは、このサービスを使ってみましょう。

## 「生成AIモデル 比べるくん」
 [[AWSブログ]誰でも簡単に生成 AI を活用！ AWS Japan メンバーが作った Party Rock アプリ集]( https://aws.amazon.com/jp/blogs/news/collection-of-partyrock-apps-made-by-aws-japan-members/)

{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/img1.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/img1.png">}}

## Claude 3 Haikuの回答
優等生！
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/img6.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/img6.png">}}

### Titan Text Liteの回答
おぉ！ある意味、人間らしい？！
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/img7.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/img7.png">}}

### 全モデル回答サンプル(DL版)
Word形式でExportする機能があります([PartyRock-dump.docx](https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/PartyRock-dump.docx))。
{{< figure alt="img1" width="300" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/img4.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/img4.png">}}

※Wordで確認できない方のために、PDF形式に変換して格納しておきます([PartyRock-dump.pdf](https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/PartyRock-dump.pdf))。

次に、ChatGPTやGeminiといったLLMでも試してみたいのですが、Party RockはAmazon Bedrockを
基盤としているためか、現在(2024年9月)は、これらChatGPTやGeminiは直接使えないようです。

そこで、代替ツール(サービス)を検討してみます。

# Difyでの実験
[Dify](https://dify.ai/jp)は、ノーコードでAIアプリケーションを開発できるオープンソースのプラット
フォームです。DifyはAIアプリ開発の民主化を目指し、多くの人々がAI技術を活用できる環境を
提供し、直感的なUIを提供することを目的としています。また、Party Rock同様、プログラミングの
知識がなくても簡単に
アプリ開発ができます。**OpenAIやHugging Faceなどの多様なLLM(AIモデル)をサポートし、
ユーザーは目的に応じて最適なモデルを選択できます。** 

## Dify設定(ローカル環境)
ローカル環境では、Docker(+Docker Compose)さえあれば、コマンド一発でサービス起動できます。

詳しくは、[このページ](https://docs.dify.ai/getting-started/install-self-hosted/docker-compose)を
確認してください。このページで、DifyをDocker Composeでインストールする手順が説明されています。
システムのハードウェア要件を確認した後、Difyリポジトリをクローンし、
環境変数を設定してDockerコンテナを起動する方法が記載されています。
また、アップグレード手順やウェブインターフェースへのアクセス方法、環境構成の
カスタマイズについても説明があります。

無事Difyが起動できたら、アプリケーションを構築します。今回は、こちらのYutube解説動画
を参考にさせていただきました。

[【Dify劇的スピードアップ】Dify並列処理完全解説](https://youtu.be/4A00eqrTPJ4) (※)

※こちらの解説動画では、Dify 0.8に追加された並列処理機能について詳しく解説されています。
並列処理により処理速度が大幅に向上したおかげもあって、特にLLMの比較では大きな効果を発揮して
いると思います。

作成したアプリ(ワークフロー)はこのようになります。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/img3.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/img3.png">}}

構築したDSLファイルをエクスポートしておきます
([Multi-LLM-parallel.yml](https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/Multi-LLM-parallel.yml))。
興味ある方は、Dify環境でインポートしてご利用ください。

**注意**: このMulti-LLM-parallel.ymlのDSLは、Dify 0.8以降のバージョンに依存しています。並列処理機能はDify 0.8で追加された機能のため、それ以前のバージョンでは正常に動作しません。

```yml
app:
  description: ''
  icon: snowman_without_snow
  icon_background: '#D3F8DF'
  mode: advanced-chat
  name: Multi-LLM-parallel
(省略)
```
<details>
<summary>全て表示</summary>

```yml
app:
  description: ''
  icon: snowman_without_snow
  icon_background: '#D3F8DF'
  mode: advanced-chat
  name: Multi-LLM-parallel
  use_icon_as_answer_icon: false

kind: app
version: 0.1.2

workflow:
  conversation_variables: []
  environment_variables: []
  features:
    file_upload:
      image:
        enabled: false
        number_limits: 3
        transfer_methods:
          - local_file
          - remote_url
    opening_statement: ''
    retriever_resource:
      enabled: false
    sensitive_word_avoidance:
      enabled: false
    speech_to_text:
      enabled: false
    suggested_questions: []
    suggested_questions_after_answer:
      enabled: false
    text_to_speech:
      enabled: false
      language: ''
      voice: ''
  
  graph:
    edges:
      - data:
          isInIteration: false
          sourceType: start
          targetType: llm
        id: 1711527768326-source-1722696301519-target
        selected: false
        source: '1711527768326'
        sourceHandle: source
        target: '1722696301519'
        targetHandle: target
        type: custom
        zIndex: 0
      - data:
          isInIteration: false
          sourceType: start
          targetType: llm
        id: 1711527768326-source-17226964257760-target
        selected: false
        source: '1711527768326'
        sourceHandle: source
        target: '17226964257760'
        targetHandle: target
        type: custom
        zIndex: 0
      - data:
          isInIteration: false
          sourceType: start
          targetType: llm
        id: 1711527768326-source-17227289485050-target
        selected: false
        source: '1711527768326'
        sourceHandle: source
        target: '17227289485050'
        targetHandle: target
        type: custom
        zIndex: 0
      - data:
          isInIteration: false
          sourceType: llm
          targetType: template-transform
        id: 1722696301519-source-1727486533733-target
        source: '1722696301519'
        sourceHandle: source
        target: '1727486533733'
        targetHandle: target
        type: custom
        zIndex: 0
      - data:
          isInIteration: false
          sourceType: llm
          targetType: template-transform
        id: 17226964257760-source-1727486533733-target
        source: '17226964257760'
        sourceHandle: source
        target: '1727486533733'
        targetHandle: target
        type: custom
        zIndex: 0
      - data:
          isInIteration: false
          sourceType: llm
          targetType: template-transform
        id: 17227289485050-source-1727486533733-target
        source: '17227289485050'
        sourceHandle: source
        target: '1727486533733'
        targetHandle: target
        type: custom
        zIndex: 0
      - data:
          isInIteration: false
          sourceType: template-transform
          targetType: answer
        id: 1727486533733-source-1727486641616-target
        source: '1727486533733'
        sourceHandle: source
        target: '1727486641616'
        targetHandle: target
        type: custom
        zIndex: 0
    
    nodes:
      - data:
          desc: ''
          selected: false
          title: Start
          type: start
          variables: []
        dragging: false
        height: 54
        id: '1711527768326'
        position:
          x: 70.49713976718482
          y: 383.8160879564906
        positionAbsolute:
          x: 70.49713976718482
          y: 383.8160879564906
        selected: false
        sourcePosition: right
        targetPosition: left
        type: custom
        width: 244
      - data:
          context:
            enabled: false
            variable_selector: []
          desc: ''
          memory:
            query_prompt_template: |
              # 議論するテーマ
              {{#sys.query#}}
            role_prefix:
              assistant: ''
              user: ''
            window:
              enabled: false
              size: 50
          model:
            completion_params:
              temperature: 0.7
            mode: chat
            name: gpt-4o
            provider: openai
          prompt_template:
            - id: 705e23ff-6139-43e1-acfd-8d1eb1911177
              role: system
              text: 以下のインプットに回答してください。
          selected: false
          title: gpt-4o
          type: llm
          variables: []
          vision:
            configs:
              detail: high
            enabled: true
        height: 98
        id: '1722696301519'
        position:
          x: 378.5649343027309
          y: 256.26932858056904
        positionAbsolute:
          x: 378.5649343027309
          y: 256.26932858056904
        selected: false
        sourcePosition: right
        targetPosition: left
        type: custom
        width: 244
      - data:
          context:
            enabled: false
            variable_selector: []
          desc: ''
          memory:
            query_prompt_template: |
              # 議論するテーマ
              {{#sys.query#}}
            role_prefix:
              assistant: ''
              user: ''
            window:
              enabled: false
              size: 50
          model:
            completion_params:
              temperature: 0.7
            mode: chat
            name: claude-3-5-sonnet-20240620
            provider: anthropic
          prompt_template:
            - id: 705e23ff-6139-43e1-acfd-8d1eb1911177
              role: system
              text: 以下のインプットに回答して下さい。
          selected: false
          title: claude-3-5-sonnet
          type: llm
          variables: []
          vision:
            configs:
              detail: high
            enabled: true
        height: 98
        id: '17226964257760'
        position:
          x: 379.3384242880875
          y: 383.8160879564906
        positionAbsolute:
          x: 379.3384242880875
          y: 383.8160879564906
        selected: false
        sourcePosition: right
        targetPosition: left
        type: custom
        width: 244
      - data:
          context:
            enabled: false
            variable_selector: []
          desc: ''
          memory:
            query_prompt_template: |
              # 議論するテーマ
              {{#sys.query#}}
            role_prefix:
              assistant: ''
              user: ''
            window:
              enabled: false
              size: 50
          model:
            completion_params:
              temperature: 0.7
            mode: chat
            name: gemini-1.5-flash-latest
            provider: google
          prompt_template:
            - id: 705e23ff-6139-43e1-acfd-8d1eb1911177
              role: system
              text: 以下のインプットに回答して下さい。
          selected: false
          title: gemini-1.5-flash
          type: llm
          variables: []
          vision:
            configs:
              detail: high
            enabled: true
        height: 98
        id: '17227289485050'
        position:
          x: 380.46235777696234
          y: 517.8277578261361
        positionAbsolute:
          x: 380.46235777696234
          y: 517.8277578261361
        selected: false
        sourcePosition: right
        targetPosition: left
        type: custom
        width: 244
      - data:
          desc: ''
          selected: false
          template: |
            # GPT-4o
            {{ x1 }}

            # Claude 3.5 Sonnet
            {{ x2 }}

            # Gemini 1.5 Flash
            {{ x3 }}
          title: テンプレート
          type: template-transform
          variables:
            - value_selector:
                - '1722696301519'
                - text
              variable: x1
            - value_selector:
                - '17226964257760'
                - text
              variable: x2
            - value_selector:
                - '17227289485050'
                - text
              variable: x3
        height: 54
        id: '1727486533733'
        position:
          x: 680.6011332032019
          y: 383.8160879564906
        positionAbsolute:
          x: 680.6011332032019
          y: 383.8160879564906
        selected: true
        sourcePosition: right
        targetPosition: left
        type: custom
        width: 244
      - data:
          answer: '{{#1727486533733.output#}}'
          desc: ''
          selected: false
          title: 回答
          type: answer
          variables: []
        height: 107
        id: '1727486641616'
        position:
          x: 974.6352210800394
          y: 383.8160879564906
        positionAbsolute:
          x: 974.6352210800394
          y: 383.8160879564906
        selected: false
        sourcePosition: right
        targetPosition: left
        type: custom
        width: 244
  
  viewport:
    x: 158.47203156901264
    y: 91.80676681762424
    zoom: 0.7383379612578201
```
</details>

## 実験の様子
今回は外部LLMとして、OpenAI、Anthoropic、Google Cloudのサービスを利用しました。各サービスの手順に従い、
外部LLMを使うためのAPIキーを設定すます。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/img9.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/img9.png">}}

デモは以下の通りです。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/img2.gif" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/img2.gif">}}

## 生成AIによる回答
```
# GPT-4o
この問題はベイズ定理を使って解くことができます。ベイズ定理を使うことで、
目撃者の証言とタクシーの割合から、ひき逃げをしたタクシーが青である確率を
求めることができます。

(省略)

したがって、目撃者が青だと証言したときに、実際にひき逃げをしたタクシーが
青である確率は約41.4%です。

# Claude 3.5 Sonnet
このような問題は、ベイズの定理を使って解くことができます。まず、必要な情報
を整理しましょう。

(省略)

したがって、ひき逃げをしたのが青タクシーである確率は約 41.38% です。

この結果は直感に反するかもしれませんが、ベイズの定理は事前確率（タクシーの割合）
と新しい証拠（目撃証言）を組み合わせて、より正確な事後確率を計算しています。
青タクシーの数が少ないため、目撃者の証言があっても、
青タクシーである確率は50%を下回っています。

# Gemini 1.5 Flash
これは、ベイズの定理を用いて解くことができる問題です。

(省略)

**答え**

ひき逃げをしたタクシーが青タクシーである確率は約 41.4% です。

```

<details>
<summary>全て表示</summary>

```
# GPT-4o
この問題はベイズ定理を使って解くことができます。ベイズ定理を使うことで、目撃者の証言とタクシーの割合から、ひき逃げをしたタクシーが青である確率を求めることができます。

まず、与えられた情報を整理しましょう：
- 市内を走るタクシーの85%は緑タクシーで、15%は青タクシーです。
- 目撃者はタクシーが青だったと証言しています。
- 目撃者は青か緑かを80%の確率で正しく識別し、20%の確率で間違えます。

ベイズ定理を適用すると以下のようになります：

\[ P(A|B) = \frac{P(B|A) \cdot P(A)}{P(B)} \]

ここで：
- \( A \) は「タクシーが青である」という事象。
- \( B \) は「目撃者がタクシーを青だと証言する」という事象。

求めたいのは \( P(A|B) \)、つまり「目撃者が青と言ったときに、タクシーが実際に青である確率」です。

1. \( P(A) \) は「タクシーが青である確率」で、これは0.15です。
2. \( P(B|A) \) は「タクシーが青であるときに目撃者が青と言う確率」で、これは0.80です。
3. \( P(B) \) は「目撃者が青と言う確率」です。これは全体の確率の和で計算します。

\[ P(B) = P(B|A) \cdot P(A) + P(B|\neg A) \cdot P(\neg A) \]

ここで \( \neg A \) は「タクシーが青でない（つまり緑である）」という事象です。
- \( P(B|\neg A) \) は「タクシーが緑であるときに目撃者が青と言う確率」で、これは0.20です。
- \( P(\neg A) \) は「タクシーが緑である確率」で、これは0.85です。

したがって、
\[ P(B) = (0.80 \times 0.15) + (0.20 \times 0.85) \]
\[ P(B) = 0.12 + 0.17 \]
\[ P(B) = 0.29 \]

これをベイズ定理に代入すると、
\[ P(A|B) = \frac{0.80 \times 0.15}{0.29} \]
\[ P(A|B) = \frac{0.12}{0.29} \]
\[ P(A|B) \approx 0.414 \]

したがって、目撃者が青だと証言したときに、実際にひき逃げをしたタクシーが青である確率は約41.4%です。

# Claude 3.5 Sonnet
このような問題は、ベイズの定理を使って解くことができます。まず、必要な情報を整理しましょう。

1. P(緑) = 0.85 (緑タクシーの割合)
2. P(青) = 0.15 (青タクシーの割合)
3. P(青と証言|実際に青) = 0.80 (目撃者の正確性)
4. P(青と証言|実際に緑) = 0.20 (目撃者の誤り率)

求めたいのは、P(実際に青|青と証言)です。

ベイズの定理を使うと：

P(実際に青|青と証言) = 
P(青と証言|実際に青) * P(青) / [P(青と証言|実際に青) * P(青) + P(青と証言|実際に緑) * P(緑)]

数値を代入すると：

P(実際に青|青と証言) = 
(0.80 * 0.15) / [(0.80 * 0.15) + (0.20 * 0.85)]

= 0.12 / (0.12 + 0.17)
= 0.12 / 0.29
≈ 0.4138

したがって、ひき逃げをしたのが青タクシーである確率は約 41.38% です。

この結果は直感に反するかもしれませんが、ベイズの定理は事前確率（タクシーの割合）と新しい証拠（目撃証言）を組み合わせて、より正確な事後確率を計算しています。青タクシーの数が少ないため、目撃者の証言があっても、青タクシーである確率は50%を下回っています。

# Gemini 1.5 Flash
これは、ベイズの定理を用いて解くことができる問題です。

**ベイズの定理**

P(A|B) = [P(B|A) * P(A)] / P(B)

ここで、

* P(A|B): 事象Bが起こった場合に、事象Aが起こる確率（事後確率）
* P(B|A): 事象Aが起こった場合に、事象Bが起こる確率（尤度）
* P(A): 事象Aが起こる確率（事前確率）
* P(B): 事象Bが起こる確率

**今回の問題への適用**

* A: ひき逃げをしたタクシーが青である
* B: 目撃者がタクシーが青だったと証言する

求めるのは、P(A|B) すなわち、目撃者が青と証言したときに、実際に青タクシーがひき逃げをした確率です。

**各確率の計算**

* P(A): 青タクシーが市内を走る確率 = 15% = 0.15
* P(B|A): 青タクシーが実際に走っている場合に、目撃者が青と証言する確率 = 80% = 0.8
* P(B|not A): 緑タクシーが実際に走っている場合に、目撃者が青と証言する確率 = 20% = 0.2
* P(not A): 緑タクシーが市内を走る確率 = 85% = 0.85

**P(B) の計算**

P(B) は、目撃者が青と証言する確率で、以下の式で計算できます。

P(B) = P(B|A) * P(A) + P(B|not A) * P(not A)
P(B) = 0.8 * 0.15 + 0.2 * 0.85 = 0.29

**ベイズの定理を用いて P(A|B) を計算**

P(A|B) = [P(B|A) * P(A)] / P(B)
P(A|B) = (0.8 * 0.15) / 0.29
P(A|B) ≈ 0.414

**答え**

ひき逃げをしたタクシーが青タクシーである確率は約 41.4% です。
```
</details>

この3モデルは、問題なく正解を導けたようです。今回Claude 3.5 Sonnetは、
補足を提示しており、興味深かったです！

*この結果は直感に反するかもしれませんが、ベイズの定理は事前確率（タクシーの割合）
と新しい証拠（目撃証言）を組み合わせて、より正確な事後確率を計算しています。
青タクシーの数が少ないため、目撃者の証言があっても、
青タクシーである確率は50%を下回っています。*

Difyは、ログ、トレースも取得できて便利です。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/img5.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/img5.png">}}

# Dify on AWSについての参考情報
公式ドキュメントの[このページ](https://docs.dify.ai/getting-started/dify-premium-on-aws)には、
Dify PremiumをEC2インスタンスへデプロイする方法の記載があります。

DifyはSaaSサービスとして提供されていますが([Dify Cloud](https://docs.dify.ai/getting-started/cloud))、
データの所在地に配慮したい企業や、Dify Cloudではリソースが足りない企業向けに設計されているよう
です。Amazon Machine Image(AMI)を使い、EC2インスタンスのパブリックIPでセットアップを行い、
Dockerで環境をカスタマイズします(環境変数のカスタマイズやアップグレード方法も記載あり)。

しかし、これでは費用もそこそこかかりますし(為替にもよりますが、1,000〜1,500円/日程度と想定)、
コンテナベースでマイクロサービス化されていることを考えると、マネージドサービス上で稼働させたく
なります。

{{< figure alt="img1" src="https://raw.githubusercontent.com/langgenius/dify/refs/heads/main/docker/docker-compose.png" link="https://raw.githubusercontent.com/langgenius/dify/refs/heads/main/docker/docker-compose.png">}}

※図は、[Dify公式プロジェクト](https://github.com/langgenius/dify.git)の「docker/docker-compose.png」より

ECSを活用する方式、EKSを活用する方式、それぞれ参考になりそうなプロジェクトがGitHubにありました。
※感謝です！

## AWSへのデプロイ
コンテナを活用するAWSへのデプロイ方法について、2つのプロジェクトを紹介します。今後はこれらのプロジェクトを参考に自前Dify
を運用していきたいと思います。

1. [[GitHub]Dify on AWS with CDK](https://github.com/aws-samples/dify-self-hosted-on-aws) (ECSを活用)
2. [[GitHub]Dify Enterprise on AWS](https://github.com/langgenius/aws-cdk-for-dify.git) (EKSを活用)

# まとめ
本記事では、AWS Party RockとDifyというノーコードAIアプリ開発プラットフォームを紹介し、
複数のLLMモデルを比較検証しました。その結果、GPT-4o、Claude 3.5 Sonnet、Gemini 1.5 Flashの
3モデルは、同一の問題に対して正確な回答を導出しました。
今後、生成AI技術はさらに進化し、ビジネスにおける応用が期待されます。

# 付録1. 
筆者の場合、週末など限定された時間帯で遊ぶケースが多く、節約のためにweekdayはリソース費用を安く抑えたい
状況です。そこでケチケチ作戦として、
[[GitHub]Dify on AWS with CDK](https://github.com/aws-samples/dify-self-hosted-on-aws)
を少しカスタマイズし、cdkコマンドでnat gatewayとECSタスクを停止できるようにしました。

[[GitHub]Dify on AWS with CDK(Frugal)](https://github.com/t-tkm/dify-on-aws.git)

```
# 起動
NAT_GATEWAY_COUNT=1 DESIRED_TASK_COUNT=1 cdk deploy

# 一時停止(ECSタスクを停止、NAT Gatewayを削除)
NAT_GATEWAY_COUNT=0 DESIRED_TASK_COUNT=0 cdk deploy
```

cdk destroyして作り直しという手もありますが、その場合はDifyアプリの設定内容なども再設定が必要で、
すこし手間が必要です。この方法(コンテナを停止)であれば、Difyアプリの設定も継続して利用でき快適です。

{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/img8.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/img8.png">}}

EC2の$0.06は、恐らくNAT Gatewayの設置料金ですね↓
- NAT Gatewayの料金
  {{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/img10.png" link="https://aws.amazon.com/jp/vpc/pricing/">}}

最後に、オリジナル版からカスタマイズしたのは以下になります。
```diff
diff --git a/bin/cdk.ts b/bin/cdk.ts
index cc66104..ed419b9 100644
--- a/bin/cdk.ts
+++ b/bin/cdk.ts
@@ -6,7 +6,7 @@ import { DifyOnAwsStack } from '../lib/dify-on-aws-stack';
 const app = new cdk.App();
 new DifyOnAwsStack(app, 'DifyOnAwsStack', {
   env: {
-    region: 'us-west-2',
+    region: 'ap-northeast-1',
     // You need to explicitly set AWS account ID when you look up an existing VPC.
     // account: '123456789012'
   },
diff --git a/lib/constructs/dify-services/api.ts b/lib/constructs/dify-services/api.ts
index 76820c5..8fed477 100644
--- a/lib/constructs/dify-services/api.ts
+++ b/lib/constructs/dify-services/api.ts
@@ -21,6 +21,7 @@ export interface ApiServiceProps {
   imageTag: string;
   sandboxImageTag: string;
   allowAnySyscalls: boolean;
+  desiredTaskCount?: number;

省略
```
<details>
<summary>全て表示</summary>

```diff
diff --git a/bin/cdk.ts b/bin/cdk.ts
index cc66104..ed419b9 100644
--- a/bin/cdk.ts
+++ b/bin/cdk.ts
@@ -6,7 +6,7 @@ import { DifyOnAwsStack } from '../lib/dify-on-aws-stack';
 const app = new cdk.App();
 new DifyOnAwsStack(app, 'DifyOnAwsStack', {
   env: {
-    region: 'us-west-2',
+    region: 'ap-northeast-1',
     // You need to explicitly set AWS account ID when you look up an existing VPC.
     // account: '123456789012'
   },
diff --git a/lib/constructs/dify-services/api.ts b/lib/constructs/dify-services/api.ts
index 76820c5..8fed477 100644
--- a/lib/constructs/dify-services/api.ts
+++ b/lib/constructs/dify-services/api.ts
@@ -21,6 +21,7 @@ export interface ApiServiceProps {
   imageTag: string;
   sandboxImageTag: string;
   allowAnySyscalls: boolean;
+  desiredTaskCount?: number;
 
   /**
    * If true, enable debug outputs
@@ -187,6 +188,7 @@ export class ApiService extends Construct {
     const service = new ecs.FargateService(this, 'FargateService', {
       cluster,
       taskDefinition,
+      desiredCount: props.desiredTaskCount ?? 1,
       capacityProviderStrategies: [
         {
           capacityProvider: 'FARGATE',
diff --git a/lib/constructs/dify-services/web.ts b/lib/constructs/dify-services/web.ts
index aa0c119..306783d 100644
--- a/lib/constructs/dify-services/web.ts
+++ b/lib/constructs/dify-services/web.ts
@@ -9,6 +9,7 @@ export interface WebServiceProps {
   alb: Alb;
 
   imageTag: string;
+  desiredTaskCount?: number;
 
   /**
    * If true, enable debug outputs
@@ -67,6 +68,7 @@ export class WebService extends Construct {
     const service = new ecs.FargateService(this, 'FargateService', {
       cluster,
       taskDefinition,
+      desiredCount: props.desiredTaskCount ?? 1,
       capacityProviderStrategies: [
         {
           capacityProvider: 'FARGATE',
diff --git a/lib/constructs/dify-services/worker.ts b/lib/constructs/dify-services/worker.ts
index 72c8826..0bd78a6 100644
--- a/lib/constructs/dify-services/worker.ts
+++ b/lib/constructs/dify-services/worker.ts
@@ -16,6 +16,7 @@ export interface WorkerServiceProps {
   encryptionSecret: ISecret;
 
   imageTag: string;
+  desiredTaskCount?: number;
 
   /**
    * If true, enable debug outputs
@@ -93,6 +94,7 @@ export class WorkerService extends Construct {
     const service = new ecs.FargateService(this, 'FargateService', {
       cluster,
       taskDefinition,
+      desiredCount: props.desiredTaskCount ?? 1,
       capacityProviderStrategies: [
         {
           capacityProvider: 'FARGATE',
diff --git a/lib/dify-on-aws-stack.ts b/lib/dify-on-aws-stack.ts
index 3eb476f..c3cd064 100644
--- a/lib/dify-on-aws-stack.ts
+++ b/lib/dify-on-aws-stack.ts
@@ -88,6 +88,18 @@ interface DifyOnAwsStackProps extends cdk.StackProps {
    * @default false
    */
   allowAnySyscalls?: boolean;
+
+  /**
+   * NAT Gatewayの数
+   * @default 1
+   */
+  natGatewayCount?: number;
+
+  /**
+   * タスクのDesiredCount
+   * @default 1
+   */
+  desiredTaskCount?: number;
 }
 
 export class DifyOnAwsStack extends cdk.Stack {
@@ -98,6 +110,8 @@ export class DifyOnAwsStack extends cdk.Stack {
       difyImageTag: imageTag = 'latest',
       difySandboxImageTag: sandboxImageTag = 'latest',
       allowAnySyscalls = false,
+      natGatewayCount = process.env.NAT_GATEWAY_COUNT ? Number(process.env.NAT_GATEWAY_COUNT) : 1,
+      desiredTaskCount = process.env.DESIRED_TASK_COUNT ? Number(process.env.DESIRED_TASK_COUNT) : 1,
     } = props;
 
     let vpc: IVpc;
@@ -105,12 +119,12 @@ export class DifyOnAwsStack extends cdk.Stack {
       vpc = Vpc.fromLookup(this, 'Vpc', { vpcId: props.vpcId });
     } else {
       vpc = new Vpc(this, 'Vpc', {
+        natGateways: natGatewayCount,
         ...(props.cheapVpc
           ? {
               natGatewayProvider: NatProvider.instanceV2({
                 instanceType: InstanceType.of(InstanceClass.T4G, InstanceSize.NANO),
               }),
-              natGateways: 1,
             }
           : {}),
         maxAzs: 2,
@@ -170,12 +184,14 @@ export class DifyOnAwsStack extends cdk.Stack {
       imageTag,
       sandboxImageTag,
       allowAnySyscalls,
+      desiredTaskCount,
     });
 
     new WebService(this, 'WebService', {
       cluster,
       alb,
       imageTag,
+      desiredTaskCount,
     });
 
     new WorkerService(this, 'WorkerService', {
@@ -185,6 +201,7 @@ export class DifyOnAwsStack extends cdk.Stack {
       storageBucket,
       encryptionSecret: api.encryptionSecret,
       imageTag,
+      desiredTaskCount,
     });
 
     new cdk.CfnOutput(this, 'DifyUrl', {
```
</details>

# 付録2. 
セキュリティについては、Difyアプリで認証を行うようにしていますが、必要に応じ、アクセス元IPアドレスを制限することで
セキュリティを高めることができます。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/img11.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_dify/img11.png">}}

