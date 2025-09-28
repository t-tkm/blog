+++ 
Categories = ["AWS"] 
Tags = ["AWS", "ECS", "Minecraft", "CDK", "MCP", "Infrastructure", "SRE", "Container", "DevOps"] 
date = "2025-09-28T00:00:00+09:00" 
title = "Minecraftサーバー管理で学ぶ！AWS ECS + MCP連携によるコンテナ運用（前編）" 
archives = ["2025", "2025-09", "2025-09-15"]
+++

# 1. 概要
本記事では、AWS ECS上でMinecraftサーバーを構築し、MCP（Model Context Protocol）を通じて
Claude Desktopから管理できる環境を構築します。前編では、AWS CDKを使用したインフラストラクチャの
構築からポートフォワードの設定までを詳しく説明します。

## 1-1. この記事で学べること

この記事では、Minecraftという身近なゲームを通して、ECSコンテナサーバの運用を学ぼうという思いもあり記載しました。

### 技術（個人的な備忘録も兼ねて）
- **コンテナオーケストレーション**：ECS + Fargateでの運用
- **インターネット経由のコンテナ操作**：ECS Exec機能の活用
- **監視・観測性**：CloudWatch Logs/メトリクスでのデバッグ
- **セキュリティ運用**：SSM Session Managerによる暗号化通信
- **コスト最適化**：ECR移行、リソース効率化
- **可用性管理**：Blue-Green デプロイメント、データ整合性確保
- **インシデント対応**：リアルタイム監視、自動復旧

### Minecraftを題材にした理由
1. **リアルなSLA**：家族のゲーム時間を守るという目標
2. **ステートフルサービス**：(Minecraftサーバの)データロック、シングルインスタンス制約など実務的な課題
3. **多様な接続方式**：ポートフォワード、ECS Exec、MCP連携など複数のアプローチ
4. **実践的な運用**：プレイヤー管理、サーバー監視、障害対応

加えて、昨今の生成AIの登場により、「天気を晴れにして」「全員にダイヤモンドを配って」のような自然言語でのサーバー操作が現実的になりました。

これを可能にするのがMCP（Model Context Protocol）です。MCPを活用することで、AIアシスタント（例：Claude Desktop）からMinecraftサーバーを自然言語で自在に管理できるようになります。

こうした最新技術を体験しつつ、AWS運用技術も同時に身につけられるといいなと思います。

## 1-2. 我が家のMinecraft進化史
数年前、[我が家のマイクラサーバー(AWS編)](https://t-tkm.github.io/blog/posts/2021/03/minecraft_server_aws_ecs/)という記事で、息子3人（小学生）と母親が楽しむMinecraftサーバーをAWS ECS+Fargate+EFSで構築した話を書きました。
[![img4](https://imgur.com/JnPzGKj.png)](https://imgur.com/JnPzGKj.png)

あれから数年が経ち、息子２人は中学生となりました。実は、あれ以来ずっとマイクラを遊び続けていた
わけではなく、一時期は部活や勉強、他の趣味に夢中になっていた時期もありました。
しかし最近になって、なぜか急に「またみんなでマイクラやりたい！」という声が家族の中で盛り上がり、
久しぶりにサーバーを新たに建てることになりました。

今度はどんな冒険が始まるのか、親子でワクワクしています。
[![img5](https://imgur.com/syPLLhT.png)](https://imgur.com/syPLLhT.png)

(作品集)
[![img3](https://imgur.com/V1aPgtT.png)](https://imgur.com/V1aPgtT.png)

私は依然としてサーバー管理者で、もっぱらみんなのプレイを鑑賞したり、保守作業に従事したり
していますが、**たまにはちょっと驚かせたいな**、という気持ちもあったりします。

そこで、今度はMCPという最新技術を使い、
自然言語でMinecraftサーバーを操作できる革新的な環境を構築することにしました。  
**「時間を夜にして」「全員にダイヤモンドを配って」「空中に花火を打ち上げて」**
といった自然な言葉でMinecraftサーバーを操作し、子どもたちの驚く顔を見たい、というのが
モチベーションです。

本記事で構築する環境では、MinecraftサーバーをAWS ECS上で運用します（当時はGUI上での手作業で構築していましたが、今回はIaCを活用）。MCP経由でClaude Desktopからサーバー管理を行えるようにし、自然言語でMinecraftサーバーを操作できる革新的な環境を実現します。

# 2. システム構成
本記事（前編・後編）で構築するシステムは以下になります。

- MinecraftサーバーをAWS ECS（Fargate）上で稼働させ、永続データはEFS（Elastic File System）に保存
- EC2(proxy)を介し、Minecraftクライアントからの接続やRCON（リモートコンソール）通信を中継
- MCPサーバーを構築し、Claude DesktopなどのAIクライアントから自然言語でMinecraftサーバーを操作(後編)

前編（本記事）で「MinecraftサーバーのECS上構築」「EFS永続化」「ポートフォワード」などインフラ基盤の構築にフォーカスし、
後編では、MCPサーバーのセットアップやClaude Desktopとの連携、実際の自然言語操作の流れを解説します。

前編、後編を通して利用するシステム全体のイメージ図は以下の通りです。
[![img2](https://imgur.com/kgTBVGz.png)](https://imgur.com/kgTBVGz.png)

# 3. 本記事で使うツール
本記事で紹介するインフラ構築手順やCDKコードの一部は、GitHub上にサンプルプロジェクトとして公開しています。  詳細なコードや最新の実装例については、必要に応じて以下のリポジトリもご参照ください。（記事内の説明とGitHub上のコードに差分がある場合は、GitHubの内容が最新となります。）


<a href="https://github.com/t-tkm/minecraft-mcp-aws-ecs-fargate.git" target="_blank" rel="noopener">
    <img src="https://gh-card.dev/repos/t-tkm/minecraft-mcp-aws-ecs-fargate.svg">
</a>

# 4. AWS環境構築
## 4-1. IaCによる構築

本プロジェクトでは、AWS CDK（Cloud Development Kit）を使ってインフラをコードで管理・構築します。 
以下は基本的なセットアップ手順の概要です。

1. **リポジトリのクローン**
   ```zsh
    git clone <repository-url>
    cd minecraft-mcp-for-aws-fargate
   ```

2. **環境設定**
   ```zsh
   # 環境変数ファイルをコピー
   cp env.example .env
   
   # .envファイルを編集して実際の値を設定
   nano .env
   
   # 環境変数を読み込み
   set -a
   source .env
   set +a
   ```

3. **AWS認証の設定**
   ```zsh
   # AWS SSOログイン
   aws sso login
   # 実行例: 
   # $aws sso login
   # Attempting to automatically open the SSO authorization page in your default browser.
   # If the browser does not open or you wish to use a different device to authorize this request, open the following URL:
   # https://d-xxxxxxxxxx.awsapps.com/start/#/device
   # Then enter the code:
   # SWSC-ZQQX
   # Successfully logged into Start URL: https://d-xxxxxxxxxx.awsapps.com/start/

   # 認証情報の確認
   aws sts get-caller-identity
   # 実行例:
   # {
   #     "UserId": "AROAEXAMPLEID:yourname",
   #     "Account": "123456789012",
   #     "Arn": "arn:aws:sts::123456789012:assumed-role/YourRole/yourname"
   # }
   ```

4. **インフラストラクチャのデプロイ**
   ```zsh
   # CDKの依存関係をインストール
   cd cdk
   python -m venv .venv
   source .venv/bin/activate  # Windows: .venv\Scripts\activate
   pip install -r requirements.txt
   
   # CDKのデプロイ
   npx cdk bootstrap  # 初回のみ
   npx cdk synth
   npx cdk deploy
   ```

デプロイが完了すると、以下のような出力が表示されます：
```zsh
✨  Total time: 380.82s
```

私の環境でおよそ6分程度で構築(デプロイ)完了しました。
詳細なパラメータやカスタマイズ方法については、リポジトリ内のREADMEや`cdk.json`、`lib/`配下の
コードを参照してください。

# 5. Minecraftサーバーへの接続確認
図の通り、Minecraftサーバーへの接続に、3つの経路を考えます。
[![img9](https://imgur.com/pw7LKrw.png)](https://imgur.com/pw7LKrw.png)

前述の2つ（Minecraftクライアント）およびrcon-cliツールから接続する場合は、
AWS SSMセッションを利用してlocalhostとのトンネル（ポートフォワード）を確立する必要があります。
自作のポートフォワード用ツールを使って設定を行います

```zsh
takumi@iMac minecraft-mcp-for-aws-fargate % ./scripts/minecraft-manager.sh start
[STATUS] Starting Minecraft Manager...
[STATUS] Detecting port forward resources
[STATUS] Running resource detection...
[STATUS] Port forward resource detection completed
[STATUS] Validating configuration..
[STATUS] Checking dependencies..
[STATUS] Dependencies verified
[STATUS] Checking AWS authentication...
[STATUS] AWS authentication verified
[STATUS] Configuration validation passed

Do you want to stop existing processes and start new port forwards? (y/N): y
takumi@iMac minecraft-mcp-for-aws-fargate % 
```

[![img6](https://imgur.com/nsJ3lJD.png)](https://imgur.com/nsJ3lJD.png)

ポートフォワードが開始されると、以下のサービスにアクセスできるようになります：
- **Minecraftサーバー**: `localhost:25565`
- **RCON**: `localhost:25575`

今回はポートフォワード用のスクリプトを作成しましたが(minecraft-manager.sh)、参考に、ポートフォワード処理の概要は次のようになっています。
```bash
# Minecraftポート（25565）のフォワード
nohup aws ssm start-session \
    --target "$EC2_INSTANCE_ID" \
    --document-name AWS-StartPortForwardingSessionToRemoteHost \
    --parameters "{\"host\":[\"$NLB_DNS_NAME\"],\"portNumber\":[\"25565\"],\"localPortNumber\":[\"25565\"]}" \
    --region "$AWS_REGION" > "$MINECRAFT_LOG_FILE" 2>&1 &

# RCONポート（25575）のフォワード  
nohup aws ssm start-session \
    --target "$EC2_INSTANCE_ID" \
    --document-name AWS-StartPortForwardingSessionToRemoteHost \
    --parameters "{\"host\":[\"$NLB_DNS_NAME\"],\"portNumber\":[\"25575\"],\"localPortNumber\":[\"25575\"]}" \
    --region "$AWS_REGION" > "$RCON_LOG_FILE" 2>&1 &
```

## 5-1. Minecraftクライアントでの接続(localhost:25565経由)
Minecraftクライアントを起動したら、「マルチプレイ」→「サーバーを追加」から
サーバーアドレスに `localhost`（必要に応じて `localhost:25565` など
ポート番号付きでも可）を入力して接続します。  
※ポート番号（:25565）はデフォルトの場合省略可能ですが、
設定を変更している場合は指定してください。

ポートフォワードが正しく設定されていれば、ローカルPCからAWS上のMinecraftサーバーに
直接アクセスできるようになります。  サーバー一覧に自分のサーバーが表示され、
通常通りマルチプレイを楽しむことができます。

※「localhost」は自分のPCを指す特別なアドレスで、ポートフォワードによってAWS上の
サーバーと安全に通信できる仕組みです。

[![img7](https://imgur.com/gm5CFBM.png)](https://imgur.com/gm5CFBM.png)

## 5-2. RCON CLIでの接続(localhost:25575経由)
以下のコマンドは、`rcon-cli` ツールを使ってローカルのMinecraftサーバー（RCONポート: 25575）に接続し、
現在オンラインのプレイヤー一覧を取得したり、ゲーム内の時間を昼や夜に変更する例です。

> 💡 `rcon-cli`(Remote CONsol)はitzg氏の開発されたツールです。詳細やインストール方法についてはリポジトリ（[https://github.com/itzg/rcon-cli](https://github.com/itzg/rcon-cli)）もご参照ください。

```zsh
# プレイヤーリストを取得（現在オンラインのプレイヤーを確認）
takumi@iMac minecraft-mcp-for-aws-fargate % rcon-cli --host localhost --port 25575 --password $RCON_PASSWORD list
There are 1 of a max of 20 players online: TousanX0911
```
コマンドの実行結果として「There are 1 of a max of 20 players online: TousanX0911」と表示され、
現在オンラインのプレイヤーを確認できます。

```zsh
# 時間を夜に設定
takumi@iMac minecraft-mcp-for-aws-fargate % rcon-cli --host localhost --port 25575 --password $RCON_PASSWORD "time set night"
Set the time to 13000
```
コマンドを実行すると「Set the time to 13000」と出力され、
時刻が夜（13000はマイクラ内の夜の時刻を表します）に変更されたことが分かります。

```zsh
# 時間を昼に戻す
takumi@iMac minecraft-mcp-for-aws-fargate % rcon-cli --host localhost --port 25575 --password $RCON_PASSWORD "time set day"  
Set the time to 1000  
```
コマンドを実行すると「Set the time to 1000」と出力され、時刻が昼に変更されたことが分かります。

下記スクリーンショットでは、「time set night」「time set day」コマンドでゲーム内の時刻を夜や昼に変更できている
様子が分かります。
[![img8](https://imgur.com/U7N7IWA.png)](https://imgur.com/U7N7IWA.png)

## 5-3. RCON on AWS ECS EXECでの操作(インターネット経由)

`./scripts/ecs-exec.sh` は、**AWS ECS上のTask(Minecraftコンテナサーバー)に対して**RCONコマンドを実行するためのスクリプトです。
ECSタスクやコンテナ情報を自動で検出し、AWS CLIのECS Exec機能を使って、サーバー内で直接コマンドを実行します。

例えば `./scripts/ecs-exec.sh list` でオンラインプレイヤー一覧を取得、  
`./scripts/ecs-exec.sh rcon "time set night"` で時刻を夜に変更できます。

**この方式の大きな特徴は、ポートフォワードが不要で、AWS環境から安全にサーバー管理ができることです。**


```zsh
takumi@iMac minecraft-mcp-for-aws-fargate % ./scripts/ecs-exec.sh list
[STATUS] Starting ECS Exec Script...
[STATUS] Detecting ECS resources
[STATUS] Running resource detection...
[STATUS] ECS resource detection completed
[STATUS] Validating configuration..
[STATUS] Checking dependencies..
[STATUS] Dependencies verified
[STATUS] Checking AWS authentication...
[STATUS] AWS authentication verified
[STATUS] Configuration validation passed

The Session Manager plugin was installed successfully. Use the AWS CLI to start a session.


Starting session with SessionId: ecs-execute-command-go6j4p27vurokf6iffp8ipu6v4
There are 1 of a max of 20 players online: TousanX0911


Exiting session with sessionId: ecs-execute-command-go6j4p27vurokf6iffp8ipu6v4.
```


```zsh
(.venv) takumi@iMac minecraft-mcp-for-aws-fargate % ./scripts/ecs-exec.sh rcon "time set night" 
[STATUS] Starting ECS Exec Script...
(冗長なので省略)

Starting session with SessionId: ecs-execute-command-b8rrbk4ofluu6doilb9l2grqrq
Set the time to 13000


Exiting session with sessionId: ecs-execute-command-b8rrbk4ofluu6doilb9l2grqrq.
```

```zsh
(.venv) takumi@iMac minecraft-mcp-for-aws-fargate % ./scripts/ecs-exec.sh rcon "time set day"   
[STATUS] Starting ECS Exec Script...
(冗長なので省略)

The Session Manager plugin was installed successfully. Use the AWS CLI to start a session.


Starting session with SessionId: ecs-execute-command-94d2afdqzbnfbp586ub523lyju
Set the time to 1000


Exiting session with sessionId: ecs-execute-command-94d2afdqzbnfbp586ub523lyju.
```

`docker exec -it`のように、ECS Execを使ってコンテナ内にシェルで入ることができます。  
コンテナ内には `rcon-cli` ツールがインストールされているため、これを利用してサーバー操作が可能です。
```zsh
(.venv) takumi@iMac minecraft-mcp-for-aws-fargate % ./scripts/ecs-exec.sh shell
[STATUS] Starting ECS Exec Script...
(冗長なので省略)

The Session Manager plugin was installed successfully. Use the AWS CLI to start a session.


Starting session with SessionId: ecs-execute-command-kr663ovbjntxsiriv2aqzpkthi

// コンテナ内の操作になります。
// rcon-cli を起動して対話モードに入ることも可能です:
// `exit` でrcon-cli対話モードを終了します
root@ip-10-1-0-30:/data# rcon-cli 
> exit

// rcon-cliにパラメーターを渡して実行させることもできます。
root@ip-10-1-0-30:/data# rcon-cli op TousanX0911
Made TousanX0911 a server operator

// コンテナから抜けるには `exit` コマンドを入力します
root@ip-10-1-0-30:/data# exit
exit

Exiting session with sessionId: ecs-execute-command-kr663ovbjntxsiriv2aqzpkthi.
```

# 6. まとめ
本記事では、AWS ECS上でMinecraftサーバーを構築し、コンテナ運用技術を実践的に学べる環境を構築しました。

- **コンテナオーケストレーション**：ECS + Fargateでの本格運用
- **インターネット経由のコンテナ操作**：ECS Exec機能による安全なリモートアクセス
- **セキュリティ運用**：SSM Session Managerによる暗号化通信
- **複数の接続方式**：ポートフォワード vs ECS Exec の使い分け
- **運用自動化**：自作スクリプトによる効率的な管理

Minecraftサーバーという身近な例を通して、リアルな運用目標（家族のゲーム時間を守る）を体験しながら、本格的なAWS運用技術を身につけることができました。

後編では、MCPサーバーとClaude Desktopとの連携により、
「時間を夜にして」「全員にダイヤモンドを配って」「空中に花火を打ち上げて」といった自然言語でのサーバー操作を実現し、さらに高度な自動化・生成AI技術の連携を紹介したいと思います。