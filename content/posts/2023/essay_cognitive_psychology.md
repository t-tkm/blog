+++
Categories = ["AWS"]
Tags = ["essay", "cognitive psychology", "Daniel Kahneman"]
date = "2023-05-07T00:00:00+09:00"
title = "ChatGPTによる認知バイアスエラー回避の可能性"
+++

# はじめに
随分前の話しなりますが(今から約7年前)、会社の先輩から
**「頭脳明瞭な経営者(経営委員会)が、時に非合理な判断の結果、失敗するケースへの理解になるかもしれない」**
と薦めてもらった面白い本があります。

タニエル・カーネマン著「[ファスト&スロー ](https://www.amazon.co.jp/dp/B0716S2Z29/)」

{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/img1.jpg">}}

本書は、ノーベル経済学賞を受賞されたダニエル・カーネマン氏による著作で、人間の思考が2つの異なるシステムに分かれていることを示しています。
システム1(ファスト)は、自動的かつ直感的な思考を司り、短時間での判断や反応に適しています。しかし、時には感情的な判断を引き起こし、誤った結論に至ることもあります。
一方、システム2(スロー)は、論理的で分析的な思考を司り、より長い時間をかけて問題に取り組むことができます。
このシステムは、より正確な判断や意思決定をするために必要ですが、人間の認知的負荷が高く、エネルギーを消費するため、継続的な使用は困難です。

さて、冒頭の「問い」に戻りますが、経営判断などは論理的で分析的な思考を司る「システム2」を使うのが適します。一方、残念ながら、
人間の思考の特性として「システム2」を使う事は難しく、(楽して？)「システム1」の判断に従った結果、合理的でない判断を下す事もある、というのが
本書で述べられている回答になります。

本書を読んで以降、私は世の中の見方が大きく変わりました！行動経済学、心理学、そして認知科学に興味のある方は、非常に貴重な情報源
となること間違いないので、是非手に取ってみて下さい。

あまりに面白かったため、当時簡単な書籍紹介をしたのを覚えています。その時に使った紹介資料がまだ手元にありましたので、
後述「付録」に載せました。興味ある方は、そちらもご参考になさって下さい。

# 本題
先日、片道2時間の出張の際、時間もあるので「少し骨太な本でも読もうかしら」とこの本をカバンに携え、数年ぶりに読み直してみて、
ふと思いついた事がありましたので(タイトルの件)、以下、書き残しておきたいと思います。

(7年前の)当時、書籍紹介の場で、「人間は元来システム2を使う事が不得意なので、プロジェクトなど重要な判断を有する
事案があった時には、非合理な判断(ミス)を回避するために(著者のお勧め通りに)「死亡前死因分析」(※)が有効です」と紹介しました。
その時参加者の方から
**「集合知(プログラミング)」という手法もあるので、うまく活用できない？」**
とコメントを頂いたのを覚えています。

指摘者の方が「集合知」というキーワードを出した理由ですが、私が話の中で「人間(の思考)が合理性を欠く原因の一つに、
(特に複雑に多くの要素が絡みあう)統計的な事実を適切に扱うことが苦手であるから」と説明したのを受けて、
指摘者の方は、であれば「集合知」と統計をうまく活用する事で、複雑な事実関係の統計的な計算部分を(人間の思考から)コンピュータへオフロード
できるのでは？と考えられたのではないかと想像しています。

今回揺れる電車の中で、ふと次のことが脳裏に浮かび、記事を残しておこうと思った次第でした:

**「最近流行りの生成系AI(ex. chatGPT)は、集合知を利用したらいいのではという当時の指摘を具現化したものではないだろうか？」**
**今であれば、(当時、バイアス回避のためにカーネマン氏も推奨している「死亡前死因分析」より)「chatGPTに相談」を推奨できるのではないか？**

では早速、試してみましょう。

# 非合理な判断しやすい事例(問題)
人間の直感的な思考が統計的な事実を無視しやすい事例として、書籍には次の例が挙げられています:

```
夜、1 台のタクシーがひき逃げをしました。この市では、緑タクシーと青タクシーの 2 社が営業しています。
事件とタクシー会社については、次の情報が与えられています。
・市内を走るタクシーの 85%は緑タクシーで、15％が青タクシーである。
・目撃者は、タクシーが青だったと証言している。裁判所は、事件当夜と同じ状況で目撃者の信頼性をテストした結果、
この目撃者は青か緑かを 80％の頻度で正しく識別し、20％の頻度でまちがえた。
では、ひき逃げをしたのが青タクシーである確率は何％でしょうか？
(第16章「原因と統計-驚くべき事実と驚くべき事例」)
```
書籍によると、この問題を出された被験者の大勢は、バイアスにより直感的に「80%」と答えた、とのことです。実際は、母集団に関する
「緑色が85%」という情報が無視してはならないキーであり、統計的に処理すれば「41%」が答えなのでありますが、人間は得てして
「緑色が85%」を直感的にはどう使っていいかわからず(合理的判断の材料から)無視されがちというバイアスの例であるそうです。

そこで、chatGPTは、この問いに対してどのような回答を生成するのか、少し試してみたいと思います。

# ChatGPTによる実験
## 正解例
chatGPTは、英語の学習データで訓練されているため、英語で質問するのが良いとも言われています。そこで、英語文面で問合せしています。
どうやら正しく回答しているようです^^。

{{< figure alt="img2" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/img2.png">}}

## 不正解例
現地点ではchatGPTも発展途上ということもあり、
**「問合せ(文)の仕方」やタイミングにより、正しい答えが得られないケースもあるので注意が必要です。即ち、適切な「問い」を投げかける事と、
回答の妥当性を正しく評価する事が必要(重要)となります。**

以下、不正解のサンプル回答を眺めてみます。

同じ文面です、ロジックは正しいようですが、計算ミスしています。「0.8 * 0.15 + 0.2 * 0.85 = 0.29」ですね。。。
{{< figure alt="img3" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/img3.png">}}

日本語でも試してみました。ロジックは正しそうですが、こちらも計算ミスしております。。。
{{< figure alt="img4" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/img4.png">}}

# 結論
書籍「ファスト&スロー」にある、人間が直感的には正しい回答を導き出しづらい一つのサンプル問題を、chatGPTに問いかけてみました。
chatGPTの回答は、(人間の直感的な非合理回答と比べ)統計事実を正しく考慮した合理的なものである点が確認できました。一方、現地点では、
「問合せ(文)の仕方」やタイミングにより、正しい答えが得られないケースもあることを確認できました。従って、chatGPT利用には適切な
「問い」を投げかける事と、回答の妥当性を正しく評価する事が大事だという点も合せて確認できました。

# 付録
以前、書籍紹介に使った資料を共有します。

{{< figure alt="slide1" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/slide1.jpeg">}}

{{< figure alt="slide2" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/slide2.jpeg">}}

{{< figure alt="slide3" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/slide3.jpeg">}}

{{< figure alt="slide4" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/slide4.jpeg">}}

{{< figure alt="slide5" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/slide5.jpeg">}}

{{< figure alt="slide6" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/slide6.jpeg">}}

{{< figure alt="slide7" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/slide7.jpeg">}}

{{< figure alt="slide8" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/slide8.jpeg">}}

{{< figure alt="slide9" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/slide9.jpeg">}}

{{< figure alt="slide10" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/slide10.jpeg">}}

{{< figure alt="slide11" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/slide11.jpeg">}}

{{< figure alt="slide12" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/slide12.jpeg">}}

{{< figure alt="slide13" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/slide13.jpeg">}}

{{< figure alt="slide14" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/slide14.jpeg">}}

{{< figure alt="slide15" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/slide15.jpeg">}}

{{< figure alt="slide16" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/slide16.jpeg">}}

{{< figure alt="slide17" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/slide17.jpeg">}}

{{< figure alt="slide18" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/slide18.jpeg">}}

{{< figure alt="slide19" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/slide19.jpeg">}}

# 参考文献
1. Daniel Kahneman, Thinking, Fast and Slow (1st edition), New York:Farrar, Straus and Giroux (2011).
2. Dan Ariely , Predictably Irrational: The Hidden Forces That Shape Our Decisions, New York: Harper Perennia, 2010.
3. 池田 信夫, 『イノベーションとは何か』,東洋経済新報社, 2011.
4. ルディー和子, 『合理的なのに愚かな戦略』, 2014.
5. 友野 典男, 『行動経済学; 経済は「感情」で動いている』, 2006.
6. Johnson, Eric J., and Daniel G. Goldstein. "Defaults and donation decisions."Transplantation 78.12 (2004): 1713-1716.
7. Ian Ayres , Super Crunchers: Why Thinking-By-Numbers is the New Way To Be Smart, New York: Bantam Dell, 2007.

# お勧めTED
- [A]Daniel Kahneman, “The riddle of experience vs. memory”, http://www.ted.com/talks/daniel_kahneman_the_riddle_of_experience_vs_memory, 2010.
- [B]Dan Ariely, “Are we in control of our own decisions?”, https://www.ted.com/talks/dan_ariely_asks_are_we_in_control_of_our_own_decisions, 2008.

