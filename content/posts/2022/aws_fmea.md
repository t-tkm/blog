+++ 
Categories = ["AWS"] 
Tags = ["AWS", "FMEA", "reliability", "resiliency"] 
date = "2022-12-03T00:00:00+09:00" 
title = "信頼性とFMEA(信頼性を高めるための工学技法)" 
+++

# はじめに
この記事は、[Japan APN Ambassador Advent Calendar 2022](https://qiita.com/advent-calendar/2022/japan-aws-ambassador)の9日目のエントリになります。

2022.10月に「[金融リファレンスアーキテクチャ 日本版](https://github.com/aws-samples/baseline-environment-on-aws-for-financial-services-institute)」も一般公開され、今後は金融だけでなく、各種ミッションクリティカルなシステムへのクラウド活用が加速していくと思われます。

そのような潮流のなか、システムの信頼性(安全性)に対しては、製造業が従来から活用している「FMEA(Ailure Mode and Effects Analysis)」(故障モード・影響解析)が有効なアプローチではないかと思い、ここに簡単にご紹介したいと思います。

# きっかけ
今から7年ほど前の2015年頃ですが、私は新しいストレージ製品開発のプロジェクトのチームに参加し、要件定義のお手伝いをしておりました。その当時、私のチームにはカイロ出身で宇宙工学の専門家がいて、その方が「このプロジェクトで品質分析にFMEA/FMECAを活用できる」と提案され、私はそこで初めてこのキーワードを知ることになりました。ただ、私がその後、急遽別プロジェクトの応援でチームから離れてしまったため、FMEA活用活動がどうなったか抑えておらず、なんとなしに「ソフトウェア工学の世界でももっと活用して有益な技法なのではないか」と頭の片隅に残ったまま月日が流れることになりました。

2019年秋、本格的にクラウド(AWS)を勉強し始め、少しずつ社内クラウド案件の技術支援を任されるようになりました。その中で、特に金融機関を対象とするシステムの非機能要求水準は高く、当時駆出しクラウドSAだった私は「W-Aの他に、何か良い学習リソースを紹介いただけますか？」とAWS SAの方に相談したところ、下記ホワイトペーパーを教えてもらいました。

「[AWSでのミッションクリティカルな金融サービスアプリケーションの構築](https://d1.awsstatic.com/Industries/Financial%20Services/Overview/Resilient%20Applications%20on%20AWS%20for%20Financial%20Services.pdf)」

このホワイトペーパーを読んでいくと、なんと「運用回復力(アプリケーションテスト手法)」でFMEAを紹介しているではありませんか？！(しかも、付録に一覧の纏めまであり「Appendix F: Failure Modes and Effects Analysis」)

という背景の下、数年ぶりに「FMEA」というキーワードと再会したのも何かの縁と感じ、改めて少しポテンシャルを探ってみようと思う一方、今もソフトウェア分野ではまだそこまで馴染みのない技法(キーワード)かと思われますので、ここに紹介することに致しました。

# FMEA
さて「肝心のFMEAとは？」ですが、私自身この分野の専門家でもなく、また活用経験も無いためここに語ることができません(いきなりすいません)。幸いなことに、ネット上にはWikipediaをはじめ、動画(ex. YouTube)にも沢山の解説記事がございますので、興味をもたれた場合はそちらを参考にされてください。

(※)書籍「[新FMEA技法 (信頼性技術叢書) ](https://www.amazon.co.jp/dp/4817194421/)」の第4章には「事務用洋ハサミ」を例にFMEA分析されていて、イメージ理解にとても参考になりました。

本記事の以降の部分は、先のAWSホワイトペーパーに使われているFMEAを題材に見ていきたいと思います。

# FMEAサンプル(1) -Application Layer FMEA
アプリケーション層から、認証機能について、下記2つの故障モードを確認してみます。

1. クライアントが認証できない
2. 認証が遅い、もしくは挙動が怪しい(信頼できない)

実際のFMEAは下記テーブルのようになります:

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

## FMEAサンプル(2) -Infrastructure FMEAの例
もう一つ、インフラ層の故障モードも確認してみましょう。AZの耐久性について、

1. 永続的AZダウン(火災、洪水によるデータセンタ破壊レベルで、復旧見込みに相応の時間が必要なケース)
2. 一時的AZダウン(電源、冷却装置、停電などデータセンタの一部コンポーネント障害、再起動によるケース) 

この故障モードでは、故障による深刻度は#1が#2と比較しかなり高くなっています。確率的には#1は#2より1/10程度低いものの、深刻度による影響度の差により、RPNは#1が高いと分析しています。検出については、AZレベル障害は「設計」段階で見逃すものでないため、最小の「1」が設定されています。

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

# 最後に
以上、いかがでしたでしょうか？実はFMEAという用語こそ使わずとも、リスク分析のプロセスで同様な分析は行われるため、実際目新しさはないのかもしれません(実際、私もはじめて聞いた時は「これって(改めて学ぶというより)いつもやってるリスク分析のことだよなぁ」と感じておりました。)

ただ、工学的に体系立てて整理されていてるため活用しない手はないと思いますし、例えば、指標に発見困難度(DET)が盛り込まれているのは私にとっては新たな発見でした。

ホワイトペーパーには
> In practice, the easiest way to reduce RPN is to add observability, so users aren’t working blind. (実際には、RPNを削減する最も簡単な⽅法は可観測性を追加することであり、ユーザーが盲⽬的に作業することはありません。)

とあり、昨今の「Observability(可観測性)」を大切にすることは効率的なRPN低減対策であるといえます。

>A useful mantra to repeat during this stage is that “a chain is only as strong as the weakest link.(この段階で繰り返すのに役⽴つマントラは「チェーンは最も弱いリンクと同じくらい強い」というものです。)

こちらは、各種対策は相対的なRPNに影響がある点を端的に示しています。例えばセキュリティ事象のFMEAを作っておけば、重箱の隅をつつくような対策にコストをかけるより、もっと優先度の高い点(ex. S3バケット設定見直)の発見に貢献できるかもしれません。

# 追伸
(故障モードに限らず)一般的に、リスクを「事象の影響度 x 発生確率」で評価することも多いかと思います。FMEAの世界でも、
- C(Critical) := SEV x PROB

とリスクを定義し、FMEA+CでFMECAと呼ばれる分析もあるようです。実際、記事冒頭で紹介した(宇宙工学)専門家の方は「FMEA」でなく「FMECA」と言っていたので、グローバルには「FMECA」の方が通じ易いのかもしれません。




