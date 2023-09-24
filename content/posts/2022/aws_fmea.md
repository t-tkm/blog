+++ 
Categories = ["AWS"] 
Tags = ["AWS", "FMEA", "reliability", "resiliency"] 
date = "2022-12-09T00:00:00+09:00" 
title = "AWSホワイトペーパを題材にFMEAを見る" 
archives = ["2022", "2022-12", "2022-12-09"]
+++

この記事は、[Japan APN Ambassador Advent Calendar 2022](https://qiita.com/advent-calendar/2022/japan-aws-ambassador)の9日目のエントリになります。

# はじめに
2022年10月に「[金融リファレンスアーキテクチャ 日本版](https://github.com/aws-samples/baseline-environment-on-aws-for-financial-services-institute)」も一般公開され、ミッションクリティカルなシステムへのクラウド活用が加速していくと思われます。

そのような潮流のなか、システムの信頼性(安全性)に対し、従来製造業を中心に活用されている「FMEA(Failure Mode and Effects Analysis)」(故障モード・影響解析)が、ソフトウェア開発でも有効なアプローチではないかと思い、ここに簡単にご紹介したいと思います。

# サマリ
- AWSホワイトペーパ「AWSでのミッションクリティカルな金融サービスアプリケーションの構築」から、FMEAサンプル2件(アプリ分析、インフラ分析)を確認します
- ソフトウェア開発へのFMEA活用について簡単に考察してみました

# きっかけ
今から7年ほど前の2015年頃ですが、私は新しいストレージ製品開発のプロジェクトのチームに参加し、要件定義支援のお仕事をしておりました。プロジェクトチームには、宇宙工学の専門家も参画しており、その方が「このプロジェクトで品質分析にFMEA/FMECAを活用できる」と提案・コンサルされていて、私はそこで初めてこのキーワードを知りました。FMEAは、同じプロジェクトに参加していたハードウェア開発部隊は(概念やその有用性など)理解しているものの、ソフトウェア開発部隊にとっては「何それ？」という感じで、チームにFMEAの有用性を理解してもらうのに大変苦労されていたのを覚えています。

その後、私は急遽別プロジェクトの応援でこのプロジェクトから離れてしまったため、このFMEA活用がどうなったのか顛末不明なのですが、その時以来、「ソフトウェア開発でも活用して有益な技法なのではないか」と頭の片隅に残ったまま、月日が流れることになりました。

それから時が過ぎ、本格的にクラウド(AWS)を勉強し始めた2018〜2019年頃(だいぶクラウドの波に乗遅れたスタート)、少しずつ社内クラウド案件の技術支援を任されるようになりました。ある時、金融機関を対象とするシステムの非機能面を検討することになり、その非機能要求水準に面食らいつつ、当時駆出しクラウドSAだった私は

「W-Aの他に、何か良い学習リソースを紹介いただけますか？」

とAWS SAの方に相談したところ、下記ホワイトペーパーを教えてもらいました。

「[AWSでのミッションクリティカルな金融サービスアプリケーションの構築](https://d1.awsstatic.com/Industries/Financial%20Services/Overview/Resilient%20Applications%20on%20AWS%20for%20Financial%20Services.pdf)」

このホワイトペーパーを読んでいくと、なんと「運用回復力(アプリケーションテスト手法)」でFMEAを紹介しているではありませんか？！しかも、付録「Appendix F: Failure Modes and Effects Analysis」に一章使って纏めまであり！

数年ぶりに「FMEA」というキーワードと再会したのも何かの縁。改めて少しポテンシャルを探ってみようと思い、ここに紹介することに致しました。

※今もソフトウェア開発の分野では、そこまで馴染みのある技法(キーワード)では無いのかしらと思いますが、いかがでしょう？(一応、Software FMEAというキーワードはあるようです)

# FMEAサンプル(AWS)
さて「肝心のFMEAとは？」ですが、(上記背景の通り)私自身この分野の専門家でもなく、また活用経験も無いためここに語ることができません(いきなりすいません)。幸いなことに、ネット上にはWikipediaをはじめ、動画(ex. YouTube)にも沢山の解説記事がございますので、興味をもたれた場合はそちらを参考にお願いします。

この記事を記載する上で、書籍「[新FMEA技法 (信頼性技術叢書) ](https://www.amazon.co.jp/dp/4817194421/)」を購入、少し読んでみました。第4章に「事務用洋ハサミ」を例にしたFMEA分析があり、イメージ理解にとても参考になりました。

以降では、先のAWSホワイトペーパーに使われているFMEA分析をサンプル(題材)として、見ていきたいと思います。

# Application Layer FMEA
アプリケーション層から、認証機能について、下記2つの故障モードを確認してみます。

1. クライアントが認証できない
2. 認証が遅い、もしくは挙動が怪しい(信頼できない)

実際のFMEAは下記テーブルのようになります:

(ホワイトペーパー:Table 12 一部抜粋)
| Item.Function | Potenrial Failure Mode(s) | Potential Effect(s) of Failure | SEV | Potential Cause(s)/Mechanism(s) of Failure | PROB | Current Design Controls | DET | RPN | Recommended Action(s) |
|:--------------|:--------------------------|:-------------------------------|:----|:-------------------------------------------|:-----|:------------------------|:----|:----|:----------------------|
| Authentication | Client can'tauthenticat | Can'tconnect application | 5 | Certificate timeout, version mismatch, account not setup, credential changed | 3 | Log and alert on authentication failures | 3 | 45 | |
| | Slow or unreliable authenticati on |Slow start for application | 4 | Auth service overloaded, high error and retry rate | 3 | Log and alert on high authentication latency and errors | 4 | 48 | |
| (以下省略)| | | | | | | | | |
| | | | | | | | | | |

各行は、故障モードといわれる事象(いわゆる本来要求されている機能、サービスが提供されない事象)が並び、各々3つの指標(KPI)で評価します:
- SEV(Severity): 故障の影響の厳しさ
- PROB(Probability): 発生頻度
- DET(Detection): 故障の検出度

そして、これらをかけ合せたものをRPN(Risk Priority Number)と呼び、対策の優先づけ(リスク分析)に利用します。
- RPN(Risk Priority Number): SEV x PRPB x DET

ここでは、各指標を10段階評価していますが(RPN1,000点)、特にこの数字に意味は無く、プロジェクト毎に定義してもよさそうです。

この故障モードでは、故障による深刻度は#1の方が#2のケースより高いのですが、故障検知について#2が#1より困難であるため、結果RPNは#2の方が高くなるという分析をしています。

(指標を抜粋)
- SEV(Severity)
  - 5: Low: System inoperable without damage
  - 4: Very Low: System operable with significant degradation of performance
- PROB(Probability)
  - 3: Low: Relatively few failures (1/15,000)
- DET(Detection)
  - 4: Moderately High: Moderately High chance the design control will detect potential cause/mechanism and subsequent failure mode
  - 3: High: High chance the design control will detect potential cause/mechanism and subsequent failure mode

# Infrastructure Layer FMEA
もう一つ、インフラ層の故障モードも確認してみましょう。AZの耐久性について、

1. 永続的AZダウン(火災、洪水によるデータセンタ破壊レベルで、復旧見込みに相応の時間が必要なケース)
2. 一時的AZダウン(電源、冷却装置、停電などデータセンタの一部コンポーネント障害、再起動によるケース) 

この故障モードでは、故障による深刻度は#1が#2と比較しかなり高くなっています。確率的には#1は#2より1/10程度低いものの、深刻度による影響度の差により、RPNは#1が高いと分析しています。検出については、AZレベル障害は「設計」段階で見逃すものでないため、最小の「1」が設定されています。

(ホワイトペーパー:Table 13 一部抜粋)
| Item.Function | Potenrial Failure Mode(s) | Potential Effect(s) of Failure | SEV | Potential Cause(s)/Mechanism(s) of Failure | PROB | Current Design Controls | DET | RPN | Recommended Action(s) |
|:--------------|:--------------------------|:-------------------------------|:----|:-------------------------------------------|:-----|:------------------------|:----|:----|:----------------------|
| Availability Zone Durability | Permanent destruction of zone | Total data loss in zone | 8 |Fire or flood inside building or destruction of data center building | 2 | Cross zone synchronous replication to over 10Km away | 1 | 16 | |
| | Temporary loss of zone | Loss of compute capacity and nondurable state in zone | 5 | Power or cooling outage causes reboot of part or all of a data center building | 3 | LCross zone synchronous replication to over 10Km away | 1 | 15 | |
| (以下省略)| | | | | | | | | |
| | | | | | | | | | |

(指標を抜粋)
- SEV(Severity)
  - 8: System inoperable with destructive failure without compromising safety
  - 5: System inoperable without damage
- PROB(Probability)
  - 3: Low: Relatively few failures (1/15,000)
  - 2: Low: Relatively few failures (1/150,000)
- DET(Detection)
  - 1: Almost Certain (Design control will detect potential cause/mechanism and subsequent failure mode)

# 考察(所感)
2つのサンプルを眺めてみましたが、いかがでしたでしょうか？

実はFMEAという用語こそ使わずとも、リスク分析のプロセスで同様な分析は行われるため、目新しさはないかもしれません。実際、私もはじめて聞いた時は「これって(改めて学ぶというより)いつもやっるリスク分析のことだよなぁ」と感じておりました。

ただ、工学的に体系立てて整理されているという点は安心感もありますし、よくよく調べてみると、新たな気づきもあるかもしれません。私の場合は、リスク(RPN)の評価指標に、発見困難度(DET)が盛り込まれているのは「面白いな」と思いました。ホワイトペーパーに
> In practice, the easiest way to reduce RPN is to add observability, so users aren’t working blind. (実際には、RPNを削減する最も簡単な⽅法は可観測性を追加することであり、ユーザーが盲⽬的に作業することはありません。)

と解説されている点は、昨今の「Observability(可観測性)」の重要性が、FMEA分析の観点でも有効である事を示していると思います。

また、下記一文は、セキュリティのリスク対策でも散々言われ続けている重要観点かと思います。

>A useful mantra to repeat during this stage is that “a chain is only as strong as the weakest link.(この段階で繰り返すのに役⽴つマントラは「チェーンは最も弱いリンクと同じくらい強い」というものです。)

FMEA分析(RPN活用)の意義を端的に示しています。例えば、セキュリティ事象のFMEA分析により、(効果の低い)重箱の隅をつつくような対策にコストをかけず、もっと優先度の高い対策(ex. S3バケット設定見直し)の発見につながる、といった効果も期待できます。

# まとめ
以上、筆者がFMEAに着目しているきっかけを背景に、AWSホワイトペーパを題材としてFMEA活用サンプルを確認しました。また、今後ソフトウェア開発に対してのFMEA活用可能性について考えてみました。実際の案件・プロジェクトへ適用する場合は、もっと考えなければならない点があるのですが(※)、まずはFMEAを知るきっかけになっていただければ幸いです。

(*)FMEAは、好ましく無い事象(故障モード)に対し、システム全体への影響を分析するボトムアップ分析アプローチですが、そもそも各事象をどのように列挙していけばよいか、が難しい課題です。一つは、FTA(Fault Tree Analysis)と呼ばれる、好ましくない事象をトップダウンにブレークダウンしていく手法もあるようです。

# 追伸
(故障モードに限らず)一般的に、リスクを「事象の影響度 x 発生確率」で評価することも多いかと思います。FMEAの世界でも、
- C(Critical) := SEV x PROB

とリスクを定義し、FMEA+CでFMECAと呼ばれる分析もあるようです。実際、記事冒頭で紹介した(宇宙工学)専門家の方は「FMEA」でなく「FMECA」と言っていたので、グローバルには「FMECA」の方が通じ易いのかもしれません。