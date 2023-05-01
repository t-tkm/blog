+++
Categories = ["AWS"]
Tags = ["essay", "cognitive psychology", "Daniel Kahneman"]
date = "2023-04-23T00:00:00+09:00"
title = "ChatGPTによる認知バイアスエラー回避"
+++

# はじめに
随分前になりますが(7年前)、会社の先輩から「頭脳明瞭な経営者が、時に非合理な判断の結果、失敗するケースへの
理解になるかもしれない」と薦めてもらったのがこの本です。その後の私の人生で、世の中の見方
が大きく変わり、大切な判断を有する際に「ちょっと待てよ」と立ち止まれるようになりました。

読了後、あまりに面白く得るところが多かったため、部内で簡単な書籍紹介をしたのを覚えています。
紹介資料がまだ手元にあったので、後述「付録」に載せました。興味ある方は、ご参考下さい。

タニエル・カーネマン著「[ファスト&スロー ](https://www.amazon.co.jp/dp/B0716S2Z29/)」

{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/img1.jpg">}}

先日、片道2時間の出張の際、時間もあるので「少し骨太な本でも読もうかしら」とこの本をカバンに携え、数年ぶりに読み直してみました。

この本では、人間が様々な判断をする際に「認知バイアス」の影響で、合理的な判断ができないケースもあり、仮に(この本を熟読し)それら
バイアスを理解したとしても、いざ判断の場において、人が思考の上でそれらバイアスを避ける事は、(人間がもつ思考の特性から)難しい
ということが述べられています。

当時、書籍紹介の場で、これら非合理な判断(ミス)を回避するために「死亡前死因分析」が有効ですと紹介したのですが、参加者の
方から「集合知(プログラミング)」という手法もあるので、うまく活用できない？」とコメントを頂いたのを覚えています。

判断の際、人間(の思考)が合理性を欠く原因の一つとして、人間が(特に複雑に多くの要素が絡みあう)統計的な事実を
適切に扱うことが苦手であるといいます(※)。「集合知」と統計をうまく活用する事で、複雑な事実関係と、統計的な計算
部分を(人間の思考から)コンピュータへオフロードできるのでは？という指摘だったのかと思います。

先日、揺れる電車の中で、「最近流行りの生成系AI(ex. chatGPT)は、当時の指摘を具現化したものではないか？」という思いに至りました。

- 今の時代であれば、(当時、バイアス回避のためにカーネマン氏も推奨している「死亡前死因分析」より)「chatGPTに相談」を推奨できるのではないか？

※正確には、統計的な思考を苦手とするのは、書籍で「システム1」と言われる直感をメインとする思考システムになります。


# 非合理な判断しやすい事例(問題)
統計的な事実を無視しやすい事例として、書籍には次のような例が挙げられている:

```
夜、1 台のタクシーがひき逃げをしました。この市では、緑タクシーと青タクシーの 2 社が営業しています。
事件とタクシー会社については、次の情報が与えられています。
・市内を走るタクシーの 85%は緑タクシーで、15％が青タクシーである。
・目撃者は、タクシーが青だったと証言している。裁判所は、事件当夜と同じ状況で目撃者の信頼性をテストした結果、この目撃者は青か緑かを 80％の頻度で正しく識別し、20％の頻度でまちがえた。
では、ひき逃げをしたのが青タクシーである確率は何％でしょうか？
```
(第16章「原因と統計-驚くべき事実と驚くべき事例」)

この問題を出された被験者の大勢は、バイアスにより直感的に「80%」と答えた、とのことです。実際は、母集団に関する「緑色が85%」という情報が無視
してはならないキーであり、統計的に処理すれば「41%」が答えであるが、人間は得てして「緑色が85%」を直感的にはどう使っていいかわからず
(合理的判断の材料から)無視されがちというバイアスの例である。

# ChatGPTに聞いてみた
## ChatGPTの回答(正解)
chatGPTは、英語の学習データで訓練されているため、英語で質問するのが良いとも言われています。そこで、英語文面で問合せしました。

**結果、バイアスに左右されず正しく回答してくれることがわかりました^^**

{{< figure alt="img2" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/img2.png">}}

ただし、現地点では、技術が発展途上ということもあり「問合せ(文)の仕方」やタイミングにより、正しい答えが得られないケースもあるので注意が必要です。
即ち、適切な「問い」を投げかける事と、回答の妥当性を正しく評価する事が必要となります。

以下、不正解のサンプル回答を眺めてみます。

## ChatGPTの回答(不正解)
同じ文面です、ロジックは正しいようですが、計算ミスしています。
「0.8 * 0.15 + 0.2 * 0.85 = 0.29」ですね。。
{{< figure alt="img3" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/img3.png">}}

日本語でも、ロジックは正しいですが、こちらも計算ミスしております。。
## ChatGPTの回答(不正解)
{{< figure alt="img4" src="https://github.com/t-tkm/blog_images/raw/main/2023/essay_cognitive_psychology/img4.png">}}

# 結論
書籍「ファスト&スロー」にある、人間が直感的には正しい回答を導き出しづらい一つのサンプル問題を、chatGPTに問いかけてみた。
chatGPTの回答は、(人間の直感的な非合理回答と比べ)統計事実を正しく考慮した合理的なものである点が確認できた。

一方、現地点では、問合せの文面や、タイミングによって誤った結果を回答するケースも見受けられるため、適切な「問い」を投げかける事と、
回答の妥当性を正しく評価する事が必要であることが分かった。

# 付録
書籍「ファスト&スロー」は、ノーベル経済学賞を受賞されたダニエル・カーネマン氏による著作で、人間の思考が2つの異なるシステムに分かれていることを示しています。
システム1(ファスト)は、自動的かつ直感的な思考を司り、短時間での判断や反応に適しています。しかし、時には感情的な判断を引き起こし、誤った結論に至ることもあります。
一方、システム2(スロー)は、論理的で分析的な思考を司り、より長い時間をかけて問題に取り組むことができます。
このシステムは、より正確な判断や意思決定をするために必要ですが、人間の認知的負荷が高く、エネルギーを消費するため、継続的な使用は困難です。

例えば難しい経営判断などは、論理的で分析的な思考を司る「システム2」を使わないといけないですが、残念ながら、人間の思考の特性としてシステム2を
使う事は難しく、「システム1」の判断に従った結果、合理的でない判断を下す事もあるとの事です。

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

