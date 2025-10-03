+++ 
Categories = ["AWS"] 
Tags = ["AWS", "ECS", "Minecraft", "CDK", "MCP", "Infrastructure", "SRE", "Container", "DevOps"] 
date = "2025-10-03T17:00:00+09:00" 
# lastmod = "2025-10-03T17:00:00+09:00"
title = "Minecraftサーバー管理で学ぶ！AWS ECS + MCP連携によるコンテナ運用（後編）" 
archives = ["2025", "2025-10", "2025-10-03"]
+++

# 1. 概要
本記事では、前編で構築したAWS ECS上のMinecraftサーバーとClaude DesktopをMCP（Model Context Protocol、以降MCP）
経由で連携させ、自然言語でサーバー管理を行える革新的な環境を構築します。

前編では、AWS CDKを使用したインフラストラクチャの構築からポートフォワードの設定までを詳しく説明し、
MinecraftクライアントやRCONクライアント（rcon-cli、ECS Exec経由）を使った接続確認も実施しました。

本記事（後編）では、MCPサーバーの構築とClaude Desktopとの連携方法を解説し、
**「時間を夜にして」「全員にダイヤモンドを配って」「空中に花火を打ち上げて」**
といった自然な言葉でMinecraftサーバーを操作できる革新的な環境の実現に挑戦します。

### この記事の内容
前編で学んだAWS運用技術を基盤として、最新のAI連携技術を体験します。

- **AI連携による自動化**：MCPプロトコルを活用した自然言語でのサーバー操作
- **プロトコル設計**：MCPサーバーの実装とツール設計

### 実現する革新的な体験
MCPを活用することで、**従来のコマンドライン操作を自然言語で行える**ようにします。
技術的な知識がなくても、直感的にMinecraftサーバーを管理できるようになることを目指します。

**例：従来の操作**
```zsh
rcon-cli --host localhost --port 25575 --password $RCON_PASSWORD "time set night"
rcon-cli --host localhost --port 25575 --password $RCON_PASSWORD "give @a minecraft:diamond 64"
```

**例：MCP連携後の操作**
```text
「時間を夜にして」
「全員にダイヤモンドを64個配って」
```

この技術により、専門家でなくても直感的にサーバーを管理できるようになり、
**運用の民主化**を実現します。

# 2. システム構成
前編で構築したシステムに加え、今回は新たにMCPサーバーとClaude Desktopを組み込んだ
構成となっています。
[![img12](https://imgur.com/O25HQE6.png)](https://imgur.com/O25HQE6.png)

MCPサーバーとClaude Desktop、Minecraftサーバーがどのように連携し、
自然言語コマンドが実際のサーバー操作に変換されるかの流れを図示します。
[![img16](https://imgur.com/8yeBq0R.png)](https://imgur.com/8yeBq0R.png)

# 3. MCPサーバーの構築
## 3-1. 前提条件の確認
前編で構築した環境が正常に動作していることを確認してください。

```zsh
# ポートフォワードの確認
./scripts/minecraft-manager.sh status

# ECS Execでの接続確認
./scripts/ecs-exec.sh list
```

これらのコマンドが正常に実行できることを確認してから、MCPサーバーの構築に進みましょう。

ecs-exec.shスクリプトが正しく動作することを確認します：
```zsh
# コマンド実行サンプル
./scripts/ecs-exec.sh list
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


Starting session with SessionId: ecs-execute-command-9a2obn3d9avl5xs8o67fejksly
There are 0 of a max of 20 players online: 


Exiting session with sessionId: ecs-execute-command-9a2obn3d9avl5xs8o67fejksly.
```


## 3-2. MCPサーバーのセットアップ（起動）
MCPサーバーは、前編で使用した`rcon-tool`をベースに構築します。

セットアップ（初回のみ）
```zsh
# プロジェクトディレクトリに移動
cd rcon-tool

# 仮想環境の作成とアクティベート
python -m venv .venv
source .venv/bin/activate

# FastMCPと依存関係のインストール
pip install -e .
```
**環境変数の確認（毎回）**
```zsh
# .envファイルの作成（プロジェクトルートに）
cp ../env.example .env

# 必要な環境変数を設定
# CLUSTER_NAME=your-cluster-name
# SERVICE_NAME=your-service-name
# CONTAINER_NAME=minecraft
# AWS_REGION=ap-northeast-1
# AWS_PROFILE=your-profile
```

**MCPサーバー起動**
```zsh
# サーバーの起動
python rcon.py
```
[![img15](https://imgur.com/A3gH2hh.png)](https://imgur.com/A3gH2hh.png)

サーバーが起動できることが確認できれば、一度このサーバーは停止して構いません。
後述の、MCPクライアント（Claude Desktop）がアプリ起動の延長で、毎回このMCPサーバーを起動します。



# 4. Claude Desktopの設定方法と使用例
## 4-1. Claude Desktopのインストール
まず、Claude Desktopをインストールします。

```zsh
# macOSの場合
brew install --cask claude

# または公式サイトからダウンロード
# https://claude.ai/download
```

Claude Desktopは無料で利用でき、MCPサーバーとの連携も簡単に設定できます。

## 4-2. MCP設定ファイルの作成
Claude DesktopでMCPサーバーを使用するには、設定ファイルを作成する必要があります。
> **参考**: [Model Context Protocol - Connect to local MCP servers](https://modelcontextprotocol.io/docs/develop/connect-local-servers)
```zsh
~/Library/Application Support/Claude/claude_desktop_config.json
```

サーバーの情報は次のように追加します。

```json
{
  "mcpServers": {
    "minecraft-rcon-ecs": {
      "command": "/path/to/your/project/rcon-tool/.venv/bin/python",
      "args": [
        "/path/to/your/project/rcon-tool/rcon.py"
      ],
      "env": {
        "AWS_PROFILE": "your-aws-profile",
        "AWS_REGION": "ap-northeast-1"
      }
    }
  }
}
```
## 4-3. 接続確認
設定が完了したら、Claude Desktopを再起動して接続を確認します：

**Claude Desktopの起動**
[![img18](https://imgur.com/Ou0rz3o.png)](https://imgur.com/Ou0rz3o.png)


**MCP接続の確認**
Claude Desktopで新しいチャットを開始し、以下のようなメッセージを送信してみます：

```text
「マイクラの時間を夜にして」
```

Claude DesktopがMCPサーバーに正しく接続できていれば、自然言語で送信したメッセージがMCPサーバー経由でRCONコマンドに変換され、Minecraftサーバーに反映されます。
[![img19](https://imgur.com/6vLL80W.png)](https://imgur.com/6vLL80W.png)
MCPサーバーとClaude Desktop間で、自然言語で送信したコマンドが正しく
Minecraftサーバーへ反映されることを確認できました。これで、Claude Desktopと
MCPサーバーの連携は完了です。

# 5. 動作検証
ここからは、Claude DesktopとMCPサーバーを連携させて、実際に自然言語でMinecraftサーバー
を操作します。

まずは、「マイクラの全員にダイヤモンドを配って」と指示してみましょう。
この一言で、MCPサーバーが裏側でRCONコマンド（`give @a minecraft:diamond 64`）
に変換し、Minecraftサーバー上の全プレイヤーにダイヤモンドが配布されていることが
確認できます。

従来はコマンドラインで適切なコマンドを入力する必要がありましたが、MCP連携により、
自然言語(日本語)の指示だけで、サーバー操作が実現できることが体験できました。
[![img20](https://imgur.com/T8vux2q.png)](https://imgur.com/T8vux2q.png)

ここまでの内容を踏まえ、当初掲げたシナリオ(**「マイクラの空に花火を打ち上げてみて」**)を実際に動作検証してみます。どこまで期待通りに動作するかもあわせて確認していきましょう。
[![img21](https://imgur.com/QLCD4ou.png)](https://imgur.com/QLCD4ou.png)

下記テーブルは、MCPサーバーやClaude Desktopを通じて「マイクラの空に花火を打ち上げて」といった自然言語コマンドを送信した際、MCPサーバーがどのようなRCONコマンドに変換してMinecraftサーバーへ送信したか、その実行例と内容・目的をまとめたものです：

| No. | 実行コマンド                                                                                                                                                                                                                                | 備考                                |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------ |
| 1   | `data get entity @p Pos`                                                                                                                                                                                                              | プレイヤー（最も近いプレイヤー）の座標を取得               |
| 2   | `summon firework_rocket 23 120 9 {LifeTime:20,FireworksItem:{id:"firework_rocket",count:1,components:{"minecraft:fireworks":{explosions:[{shape:"large_ball",colors:[I;16711680,16776960,65280],fade_colors:[I;16711935,65535]}]}}}}` | 花火ロケットを指定位置に召喚（複雑な爆発エフェクト付き）※構文エラーあり |
| 3   | `summon firework_rocket 23 120 9 {LifeTime:20,FireworksItem:{id:"firework_rocket",count:1,components:{"minecraft:fireworks":{explosions:[{shape:"large_ball",colors:[I;16711680,16776960,65280]}]}}}}`                                | 花火ロケット召喚（fade_colors省略）※構文エラーあり      |
| 4   | `summon firework_rocket 23 120 9 {LifeTime:20}`                                                                                                                                                                                       | 花火ロケット召喚（シンプルなパラメータ）成功             |
| 5   | `summon firework_rocket 20 125 8 {LifeTime:25}`                                                                                                                                                                                       | 花火ロケット召喚（座標・寿命変更）成功                |
| 6   | `summon firework_rocket 26 118 10 {LifeTime:22}`                                                                                                                                                                                      | 花火ロケット召喚（別座標・寿命22）成功               |

実際に試してみたところ、**ロケット花火が3発だけ、ひっそりと静かに打ち上がるだけ**という、なんとも寂しい結果に終わってしまいました。
あまりにも一瞬で終わってしまい、期待していた華やかな演出もなく、キャプチャすら残せませんでした。

私がコマンドやパラメータの調整に悪戦苦闘している間、息子はわずか5分程度で本格的な花火装置を作り上げてしまいました。その完成度とスピードには思わず脱帽です。

下記がその息子の作品です(画像をクリックすると、GIFアニメーションで華やかな花火の様子をご覧いただけます)。もともと、今回のAIエージェントに期待していたのは、まさに下記のような華やかな花火演出でした。
[![img13](https://imgur.com/UeKoIJm.png)](https://github.com/t-tkm/minecraft-mcp-aws-ecs-fargate/blob/0fca7b5bc29ae7c6932d6df10fc402a0e250ac11/img/hanabi_created_by_human.gif)

**分かったこと**

今回のプロンプト設計やツールの作り方自体にも工夫の余地がありそうで、プロンプトをもっと工夫すれば、より高度な自動化や演出も実現できるかもしれません。

熟練者の持つナレッジや技術を、どこまでAIに精度高く実装できるのか、その進化の過程を体験できるのも非常に楽しみです。今後、AIが人間のノウハウをどこまで再現し、さらなる創造的な演出や自動化を実現できるのか、引き続き挑戦していきたいと思います。

# 6. まとめ
本記事では、前編で構築したAWS ECS上のMinecraftサーバーにMCPサーバーを連携させ、
Claude Desktopから自然言語でサーバー操作を行える革新的な環境を構築しました。

Minecraftという身近なゲームを通して、最新のAI技術とAWS運用技術を組み合わせた革新的な環境を体験できました。今回の実験により、AIの自動化や演出面にはまだまだ改良の余地があることが分かりましたが、今後はより高度なプロンプト設計やツールの改良を通じて、人間の創造性に近づけるよう挑戦していきたいと思います。

# 付録
## A. Claude Desktopのサーバログ確認方法
### A-1. Claude Desktopのログ確認
```zsh
 ~/Library/Logs/Claude  tree .                                                                                                                        <aws:demo2-admin> <region:ap-northeast-1>
.
├── main.log
├── mcp-server-minecraft-rcon-ecs.log
├── mcp.log
└── window.log
```

> **注意**: ログファイルの場所は、Claude DesktopのバージョンやOSによって異なる場合があります。上記のパスで見つからない場合は、Claude Desktopの設定ディレクトリ内を確認してください。


**ログの内容例（mcp.log）**
```zsh
[~/Library/Logs/Claude]tail -f mcp.log
2025-10-03T07:51:59.739Z [info] [minecraft-rcon-ecs] Message from client: {"method":"tools/call","params":{"name":"rcon","arguments":{"command":"time set night"}},"jsonrpc":"2.0","id":4}
2025-10-03T07:52:10.577Z [info] [minecraft-rcon-ecs] Message from server: {"jsonrpc":"2.0","id":4,"result":{"content":[{"type":"text","text":"The Session Manager plugin was installed successfully. Use the AWS CLI to start a session.\nStarting session with SessionId: ecs-execute-command-dn3o4bvfpd4glgscernutbvjne\nSet the time to 13000\nExiting session with sessionId: ecs-execute-command-dn3o4bvfpd4glgscernutbvjne."}],"structuredContent":{"result":"The Session Manager plugin was installed successfully. Use the AWS CLI to start a session.\nStarting session with SessionId: ecs-execute-command-dn3o4bvfpd4glgscernutbvjne\nSet the time to 13000\nExiting session with sessionId: ecs-execute-command-dn3o4bvfpd4glgscernutbvjne."},"isError":false}}
2025-10-03T07:52:19.017Z [info] [minecraft-rcon-ecs] Shutting down server...
2025-10-03T07:52:19.017Z [info] [minecraft-rcon-ecs] Client transport closed
2025-10-03T07:52:19.018Z [info] [minecraft-rcon-ecs] Server transport closed
2025-10-03T07:52:19.018Z [info] [minecraft-rcon-ecs] Client transport closed
2025-10-03T07:52:19.018Z [info] [minecraft-rcon-ecs] Server transport closed (intentional shutdown)
2025-10-03T07:52:19.018Z [info] [minecraft-rcon-ecs] Client transport closed
2025-10-03T07:52:19.039Z [info] [minecraft-rcon-ecs] Server transport closed
2025-10-03T07:52:19.039Z [info] [minecraft-rcon-ecs] Client transport closed```
```

**ログの内容例（mcp-server-minecraft-rcon-ecs.log）**
```zsh
# 最新のログファイルを確認
[~/Library/Logs/Claude]tail -f -100 mcp-server-minecraft-rcon-ecs.log
[MCP] Set TF_VAR_aws_ecs_container_java_memory_heap=6G (was: not set)
[MCP] Set TF_VAR_allowed_ips='["0.0.0.0/0"]' (was: not set)
[MCP] Set TF_VAR_rcon_password=${RCON_PASSWORD} (was: not set)
[MCP] Set TF_VAR_ssh_public_key_path=~/.ssh/minecraft-proxy-key.pub (was: not set)
[MCP] Set TF_VAR_minecraft_version=1.21.8 (was: not set)
[MCP] Set TF_VAR_docker_image=xxxxxxxxxxxx.dkr.ecr.ap-northeast-1.amazonaws.com/my-minecraft-server:Intel_20250925 (was: not set)
[MCP] Set TF_VAR_my_ip=xxx.xxx.xxx.xxx/32AWS_REGION=ap-northeast-1 (was: not set)
[MCP] Environment variables:
[MCP]   PROJECT_ROOT: /Users/username/repos/minecraft-mcp-for-aws-fargate
[MCP]   AWS_REGION: ap-northeast-1
[MCP]   ENVIRONMENT: dev
[MCP]   PROJECT_NAME: your-project-name
INFO:resource_detector:ResourceDetector initialized: env=dev, project=your-project-name, region=ap-northeast-1
INFO:resource_detector:AWS CLI path: /usr/local/bin/aws
INFO:resource_detector:Starting resource detection...
INFO:resource_detector:Detecting ECS cluster...
INFO:resource_detector:Found cluster by tags: your-project-cluster
INFO:resource_detector:Found cluster by tags: your-project-cluster
INFO:resource_detector:Detecting ECS service in cluster: your-project-cluster
INFO:resource_detector:Found service by naming pattern: your-project-cdk-stack-ECSMinecraftService4B964AE5-xxxxxxxxx
INFO:resource_detector:Found service by tags: your-project-cdk-stack-ECSMinecraftService4B964AE5-xxxxxxxxx
INFO:resource_detector:Detecting running task for service: your-project-cdk-stack-ECSMinecraftService4B964AE5-xxxxxxxxx
INFO:resource_detector:Found running task: arn:aws:ecs:ap-northeast-1:xxxxxxxxxxxx:task/your-project-cluster/xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
INFO:resource_detector:Detecting container name for task: arn:aws:ecs:ap-northeast-1:xxxxxxxxxxxx:task/your-project-cluster/xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
INFO:resource_detector:Found container: your-project-container
INFO:resource_detector:Detecting EC2 instance...
INFO:resource_detector:Found EC2 instance by tags: i-xxxxxxxxxxxxxxxxx
INFO:resource_detector:Detecting NLB DNS name...
WARNING:resource_detector:NLB not found for project 'your-project-name' - this is optional
INFO:resource_detector:Resource detection completed: ResourceConfig(cluster_name='your-project-cluster', service_name='your-project-cdk-stack-ECSMinecraftService4B964AE5-xxxxxxxxx', container_name='your-project-container', task_arn='arn:aws:ecs:ap-northeast-1:xxxxxxxxxxxx:task/your-project-cluster/xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx', ec2_instance_id='i-xxxxxxxxxxxxxxxxx', nlb_dns_name='no-nlb-configured', detection_mode='auto', project_name='your-project-name', environment='dev')
[MCP] Resource detection successful:
[MCP]   Cluster: your-project-cluster
[MCP]   Service: your-project-cdk-stack-ECSMinecraftService4B964AE5-xxxxxxxxx
[MCP]   Container: your-project-container
[MCP]   Detection mode: auto
[MCP] MCP Server initialized
[MCP] Starting Minecraft RCON MCP server...


╭────────────────────────────────────────────────────────────────────────────╮
│                                                                            │
│        _ __ ___  _____           __  __  _____________    ____    ____     │
│       _ __ ___ .'____/___ ______/ /_/  |/  / ____/ __ \  |___ \  / __ \    │
│      _ __ ___ / /_  / __ `/ ___/ __/ /|_/ / /   / /_/ /  ___/ / / / / /    │
│     _ __ ___ / __/ / /_/ (__  ) /_/ /  / / /___/ ____/  /  __/_/ /_/ /     │
│    _ __ ___ /_/    \____/____/\__/_/  /_/\____/_/      /_____(*)____/      │
│                                                                            │
│                                                                            │
│                                FastMCP  2.0                                │
│                                                                            │
│                                                                            │
│                 🖥️  Server name:     minecraft-rcon-ecs                     │
│                 📦 Transport:       STDIO                                  │
│                                                                            │
│                 🏎️  FastMCP version: 2.12.4                                 │
│                 🤝 MCP SDK version: 1.15.0                                 │
│                                                                            │
│                 📚 Docs:            https://gofastmcp.com                  │
│                 🚀 Deploy:          https://fastmcp.cloud                  │
│                                                                            │
╰────────────────────────────────────────────────────────────────────────────╯


[10/03/25 16:51:43] INFO     Starting MCP server                  server.py:1502
                             'minecraft-rcon-ecs' with transport                
                             'stdio'                                            
2025-10-03T07:51:43.655Z [minecraft-rcon-ecs] [info] Message from server: {"jsonrpc":"2.0","id":0,"result":{"protocolVersion":"2025-06-18","capabilities":{"experimental":{},"prompts":{"listChanged":false},"resources":{"subscribe":false,"listChanged":false},"tools":{"listChanged":true}},"serverInfo":{"name":"minecraft-rcon-ecs","version":"1.15.0"}}} { metadata: undefined }
2025-10-03T07:51:43.656Z [minecraft-rcon-ecs] [info] Message from client: {"method":"notifications/initialized","jsonrpc":"2.0"} { metadata: undefined }
2025-10-03T07:51:43.657Z [minecraft-rcon-ecs] [info] Message from client: {"method":"tools/list","params":{},"jsonrpc":"2.0","id":1} { metadata: undefined }
2025-10-03T07:51:43.658Z [minecraft-rcon-ecs] [info] Message from client: {"method":"prompts/list","params":{},"jsonrpc":"2.0","id":2} { metadata: undefined }
2025-10-03T07:51:43.658Z [minecraft-rcon-ecs] [info] Message from client: {"method":"resources/list","params":{},"jsonrpc":"2.0","id":3} { metadata: undefined }
INFO:mcp.server.lowlevel.server:Processing request of type ListToolsRequest
INFO:mcp.server.lowlevel.server:Processing request of type ListPromptsRequest
INFO:mcp.server.lowlevel.server:Processing request of type ListResourcesRequest
2025-10-03T07:51:43.662Z [minecraft-rcon-ecs] [info] Message from server: {"jsonrpc":"2.0","id":1,"result":{"tools":[{"name":"rcon","description":"Perform RCON operations on the Minecraft server. Core principles and command reference:\n\nCORE SAFETY PRINCIPLES:\n1. Always get player coordinates before building operations\n   Use: data get entity <player_name> Pos\n   Returns: [X.XXd, Y.XXd, Z.XXd]\n   Store and reuse as base for safe building operations\n   日本語補足: プレイヤー座標を事前取得することで安全で予測可能な建築が可能になります\n\n2. Prefer absolute coordinates for structures  \n   Avoid building large structures relative to ~ (current position)\n   Confirm positions before execution to prevent accidents\n   日本語補足: 相対座標~は便利ですが誤って位置がずれることがあります。安全な構造物再現には絶対座標を推奨します\n\n3. Validate before executing dangerous operations\n   Check block existence (especially modded blocks)\n   Be cautious with fill/clone commands (large ranges can overwrite builds)\n   Always ask for confirmation on operations affecting >100 blocks\n   日本語補足: Mod追加ブロックは存在確認が必要です。fillやcloneは範囲指定を誤ると既存建築を破壊する危険があります\n\nPOSITION AND BUILDING COMMANDS:\n- Get player position: data get entity <player_name> Pos\n- Place single block: setblock <x> <y> <z> <block_type>[properties]\n- Fill area: fill <x1> <y1> <z1> <x2> <y2> <z2> <block_type> [replace|keep|outline|hollow]\n- Clone structures: clone <x1> <y1> <z1> <x2> <y2> <z2> <dest_x> <dest_y> <dest_z> [replace|masked]\n日本語補足: setblockは単体ブロック、fillは範囲、cloneは複製です。座標の範囲指定は始点と終点の両方を含むため意図した大きさを意識してください\n\nENTITY MANAGEMENT:\n- Summon entity: summon <entity_type> <x> <y> <z> [nbt]\n- Teleport entities: tp @e[type=<entity_type>] <x> <y> <z>\n- Execute as entity: execute as @e[type=<entity_type>] at @s run <command>\n- Kill specific entities: kill @e[type=<entity_type>,distance=..10]\n日本語補足: summonは新規生成、tpは移動、executeは特定エンティティとしてコマンド実行します。モブ制御やイベント演出に有効です\n\nPLAYER TELEPORTATION AND VIEW:\n- Teleport player: tp @p <x> <y> <z>\n- Spectate entity: spectate <target> [player]\n- Execute from position: execute positioned <x> <y> <z> run <command>\n- Set spawn point: spawnpoint <player> <x> <y> <z>\n日本語補足: 観戦モードやテレポートで視点操作が可能です。execute positionedは特定座標からのコマンド実行に便利です\n\nITEMS AND EFFECTS:\nGive Items (Modern 1.20.5+ syntax):\n- give <player> <item> [count]\n- give <player> iron_sword[enchantments={levels:{\"minecraft:sharpness\":5}}] 1\n- give @a iron_pickaxe[unbreakable={}]\n- give <player> potion[potion_contents={potion:\"minecraft:fire_resistance\"}]\n\nStatus Effects:\n- effect give @a speed 300 2\n- effect give <player> night_vision 1000 1  \n- effect give <player> water_breathing infinite 1 true\n- effect clear <player> [effect]\n日本語補足: giveはアイテム配布、effectはステータス効果付与です。クリエイティブでのテストやイベント演出に使えます\n\nWORLD MANAGEMENT:\n- Weather: weather clear|rain|thunder [duration]\n- Time: time set day|night|noon|midnight or time set <value>\n- Game rules: gamerule <rule> <value> (keepInventory, mobGriefing, etc.)\n- World border: worldborder set <size>, worldborder center <x> <z>\n日本語補足: 天候・時間・ゲームルール・ワールドボーダーの制御が可能です\n\nTARGETING SELECTORS:\n- @a: all players\n- @p: nearest player  \n- @r: random player\n- @e[type=<entity>]: all entities of specific type\n- @e[type=<entity>,limit=1]: single entity of type\n- <player_name>: specific player by name\n\nSelector Arguments:\n- distance=..10: within 10 blocks\n- x=100,y=64,z=100,distance=..5: near specific coordinates  \n- level=10..20: experience levels 10-20\n- gamemode=creative: creative mode players only\n日本語補足: ターゲット指定子は柔軟に使えます。@aで全員、プレイヤー名で個別指定可能です\n\nBLOCK STATES AND PROPERTIES:\n- Syntax: block_type[property=value]\n- Example: lantern[hanging=true]\n- Multiple properties: block_type[prop1=value1,prop2=value2]\n- Common properties: facing, waterlogged, lit, open, powered\n日本語補足: ブロックの状態（点灯・向きなど）はプロパティで指定します。\n細かい制御が可能です\n\nCOORDINATE SYSTEMS:\n- Absolute: <x> <y> <z> (exact world position)\n- Relative: ~ (current position), ~1 (+1 offset), ~-1 (-1 offset)\n- Local: ^left ^up ^forward (relative to entity facing)\n- Coordinate ranges are INCLUSIVE: ~0 to ~15 = 16 blocks total\n日本語補足: ~は便利ですが誤差で大規模建築にズレが出やすいです。\n基本は絶対座標を使うのが安全です\n\nHIGH RISK OPERATIONS - USE WITH EXTREME CAUTION:\n- fill with large ranges (>1000 blocks)\n- clone operations affecting existing builds\n- kill @e (kills ALL entities including items)\n- /stop or /restart commands\n日本語補足: 大規模なfill・clone操作や全エンティティ削除は\n既存建築を破壊する危険があります\n\nSAFETY CHECKLIST BEFORE MAJOR OPERATIONS:\n1. Get player position and survey area\n2. Calculate exact block count affected\n3. Verify all block types exist (especially modded blocks)\n4. Confirm operation with user if affecting >100 blocks\n5. Provide undo method or backup strategy\n日本語補足: 大規模操作前は必ず範囲確認・ブロック存在確認・\nバックアップ戦略を用意してください\n\nCOMMON GOTCHAS TO AVOID:\n- Never use large relative fills (~) for structures\n- Remember both corners are inclusive in fill/clone ranges\n- Some commands need player context (e.g., locate)\n- Test modded blocks before using in large operations\n- Coordinates Y<-64 or Y>320 may be invalid in some versions\n日本語補足: よくある失敗は「相対座標で建築してズレる」\n「範囲指定を誤って破壊」「存在しないブロック指定」です\n\nEMERGENCY FIXES:\n- Undo fill: fill <x1> <y1> <z1> <x2> <y2> <z2> air\n- Restore player: tp <player> 0 100 0\n- Clear effects: effect clear @a\n- Reset weather: weather clear\n日本語補足: 緊急時は該当範囲をairで埋める、\nプレイヤーを安全な場所にテレポートなどで対処します\n\nVERSION COMPATIBILITY NOTES:\n- 1.20.5+: New item component syntax with brackets\n- 1.19+: Deep dark blocks (sculk family)\n- 1.17+: Caves & cliffs blocks, extended height limits\n- 1.16+: Nether update blocks\n- 1.13+: Block ID flattening (minecraft:stone vs stone)\n日本語補足: バージョンによりブロックIDや構文が異なります。特に1.13以降のフラット化と1.20.5以降のアイテム構文変更に注意してください\n\nAlways prioritize safety over convenience. When in doubt, use smaller operations and absolute coordinates.\nAlways explain what each command does and potential risks to the user.\n            ","inputSchema":{"properties":{"command":{"type":"string"}},"required":["command"],"type":"object"},"outputSchema":{"properties":{"result":{"type":"string"}},"required":["result"],"type":"object","x-fastmcp-wrap-result":true},"_meta":{"_fastmcp":{"tags":[]}}}]}} { metadata: undefined }
2025-10-03T07:51:43.662Z [minecraft-rcon-ecs] [info] Message from server: {"jsonrpc":"2.0","id":2,"result":{"prompts":[]}} { metadata: undefined }
2025-10-03T07:51:43.663Z [minecraft-rcon-ecs] [info] Message from server: {"jsonrpc":"2.0","id":3,"result":{"resources":[]}} { metadata: undefined }
2025-10-03T07:51:59.739Z [minecraft-rcon-ecs] [info] Message from client: {"method":"tools/call","params":{"name":"rcon","arguments":{"command":"time set night"}},"jsonrpc":"2.0","id":4} { metadata: undefined }
INFO:mcp.server.lowlevel.server:Processing request of type CallToolRequest
[MCP] execute_rcon_command() called with command: time set night
[MCP] DEBUG: Passing environment variables to ecs-exec.sh:
[MCP]   CLUSTER_NAME: your-project-cluster
[MCP]   SERVICE_NAME: your-project-cdk-stack-ECSMinecraftService4B964AE5-xxxxxxxxx
[MCP]   CONTAINER_NAME: your-project-container
[MCP]   TASK_ARN: arn:aws:ecs:ap-northeast-1:xxxxxxxxxxxx:task/your-project-cluster/xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
[MCP]   AWS_PROFILE: your-aws-profile
[MCP] Running: /Users/username/repos/minecraft-mcp-for-aws-fargate/scripts/ecs-exec.sh rcon "time set night"
[MCP] RCON command result: The Session Manager plugin was installed successfully. Use the AWS CLI to start a session.
Starting session with SessionId: ecs-execute-command-xxxxxxxxxxxxxxxxxxxxxxxx
Set the time to 13000
Exiting session with sessionId: ecs-execute-command-xxxxxxxxxxxxxxxxxxxxxxxx.
2025-10-03T07:52:10.577Z [minecraft-rcon-ecs] [info] Message from server: {"jsonrpc":"2.0","id":4,"result":{"content":[{"type":"text","text":"The Session Manager plugin was installed successfully. Use the AWS CLI to start a session.\nStarting session with SessionId: ecs-execute-command-dn3o4bvfpd4glgscernutbvjne\nSet the time to 13000\nExiting session with sessionId: ecs-execute-command-dn3o4bvfpd4glgscernutbvjne."}],"structuredContent":{"result":"The Session Manager plugin was installed successfully. Use the AWS CLI to start a session.\nStarting session with SessionId: ecs-execute-command-dn3o4bvfpd4glgscernutbvjne\nSet the time to 13000\nExiting session with sessionId: ecs-execute-command-dn3o4bvfpd4glgscernutbvjne."},"isError":false}} { metadata: undefined }
2025-10-03T07:52:19.017Z [minecraft-rcon-ecs] [info] Client transport closed { metadata: undefined }
2025-10-03T07:52:19.018Z [minecraft-rcon-ecs] [info] Server transport closed { metadata: undefined }
2025-10-03T07:52:19.018Z [minecraft-rcon-ecs] [info] Client transport closed { metadata: undefined }
2025-10-03T07:52:19.018Z [minecraft-rcon-ecs] [info] Server transport closed (intentional shutdown) { metadata: undefined }
2025-10-03T07:52:19.018Z [minecraft-rcon-ecs] [info] Client transport closed { metadata: undefined }
2025-10-03T07:52:19.016Z [minecraft-rcon-ecs] [info] Shutting down server... { metadata: undefined }
2025-10-03T07:52:19.039Z [minecraft-rcon-ecs] [info] Server transport closed { metadata: undefined }
2025-10-03T07:52:19.039Z [minecraft-rcon-ecs] [info] Client transport closed { metadata: undefined }
```

## B. システムプロンプト
### 概要
`rcon.py`で定義されているMCPツールのシステムプロンプトですが、chatGPT-5を使って作成しております。
Minecraft RCON操作における安全性と効率性を両立させる包括的なガイドラインを提供しています。
Minecraftサーバーに対して安全で適切なコマンドを実行できるよう検討してもらいました。

上記のサーバーログ（mcp-server-minecraft-rcon-ecs.log）にも記録されていますが、
これはrcon.pyで設定されているシステムプロンプトの内容です。
```json
{
  "timestamp": "2025-10-03T07:51:43.662Z",
  "service": "minecraft-rcon-ecs",
  "level": "info",
  "message": "Message from server",
  "data": {
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
      "tools": [
        {
          "name": "rcon",
          "description": "Perform RCON operations on the Minecraft server. Core principles and command reference:\n\nCORE SAFETY PRINCIPLES:\n1. Always get player coordinates before building operations\n   Use: data get entity <player_name> Pos\n   Returns: [X.XXd, Y.XXd, Z.XXd]\n   Store and reuse as base for safe building operations\n   日本語補足: プレイヤー座標を事前取得することで安全で予測可能な建築が可能になります\n\n2. Prefer absolute coordinates for structures  \n   Avoid building large structures relative to ~ (current position)\n   Confirm positions before 
(長いので省略)
    }
  },
  "metadata": null
}
```

`rcon.py`の中で「RCON操作の安全原則・コマンドリファレンス」をdocstringとして詳細に記述し、Claude DesktopやMCPサーバーが安全かつ効率的にMinecraftサーバーを操作できるよう設計しています。

---
**rcon.py抜粋例：**
```python
   def _register_tools(self):
        """MCPツールを登録"""
        @self.mcp.tool()
        def rcon(command: str) -> str:
            """Perform RCON operations on the Minecraft server. Core principles and command reference:

CORE SAFETY PRINCIPLES:
1. Always get player coordinates before building operations
   Use: data get entity <player_name> Pos
   Returns: [X.XXd, Y.XXd, Z.XXd]
   Store and reuse as base for safe building operations
   日本語補足: プレイヤー座標を事前取得することで安全で予測可能な建築が可能になります

2. Prefer absolute coordinates for structures  
   Avoid building large structures relative to ~ (current position)
   Confirm positions before execution to prevent accidents
   日本語補足: 相対座標~は便利ですが誤って位置がずれることがあります。安全な構造物再現には絶対座標を推奨します

3. Validate before executing dangerous operations
   Check block existence (especially modded blocks)
   Be cautious with fill/clone commands (large ranges can overwrite builds)
   Always ask for confirmation on operations affecting >100 blocks
   日本語補足: Mod追加ブロックは存在確認が必要です。fillやcloneは範囲指定を誤ると既存建築を破壊する危険があります

POSITION AND BUILDING COMMANDS:
- Get player position: data get entity <player_name> Pos
- Place single block: setblock <x> <y> <z> <block_type>[properties]
- Fill area: fill <x1> <y1> <z1> <x2> <y2> <z2> <block_type> [replace|keep|outline|hollow]
- Clone structures: clone <x1> <y1> <z1> <x2> <y2> <z2> <dest_x> <dest_y> <dest_z> [replace|masked]
日本語補足: setblockは単体ブロック、fillは範囲、cloneは複製です。座標の範囲指定は始点と終点の両方を含むため意図した大きさを意識してください

ENTITY MANAGEMENT:
- Summon entity: summon <entity_type> <x> <y> <z> [nbt]
- Teleport entities: tp @e[type=<entity_type>] <x> <y> <z>
- Execute as entity: execute as @e[type=<entity_type>] at @s run <command>
- Kill specific entities: kill @e[type=<entity_type>,distance=..10]
日本語補足: summonは新規生成、tpは移動、executeは特定エンティティとしてコマンド実行します。モブ制御やイベント演出に有効です

PLAYER TELEPORTATION AND VIEW:
- Teleport player: tp @p <x> <y> <z>
- Spectate entity: spectate <target> [player]
- Execute from position: execute positioned <x> <y> <z> run <command>
- Set spawn point: spawnpoint <player> <x> <y> <z>
日本語補足: 観戦モードやテレポートで視点操作が可能です。execute positionedは特定座標からのコマンド実行に便利です

ITEMS AND EFFECTS:
Give Items (Modern 1.20.5+ syntax):
- give <player> <item> [count]
- give <player> iron_sword[enchantments={levels:{"minecraft:sharpness":5}}] 1
- give @a iron_pickaxe[unbreakable={}]
- give <player> potion[potion_contents={potion:"minecraft:fire_resistance"}]

Status Effects:
- effect give @a speed 300 2
- effect give <player> night_vision 1000 1  
- effect give <player> water_breathing infinite 1 true
- effect clear <player> [effect]
日本語補足: giveはアイテム配布、effectはステータス効果付与です。クリエイティブでのテストやイベント演出に使えます

WORLD MANAGEMENT:
- Weather: weather clear|rain|thunder [duration]
- Time: time set day|night|noon|midnight or time set <value>
- Game rules: gamerule <rule> <value> (keepInventory, mobGriefing, etc.)
- World border: worldborder set <size>, worldborder center <x> <z>
日本語補足: 天候・時間・ゲームルール・ワールドボーダーの制御が可能です

TARGETING SELECTORS:
- @a: all players
- @p: nearest player  
- @r: random player
- @e[type=<entity>]: all entities of specific type
- @e[type=<entity>,limit=1]: single entity of type
- <player_name>: specific player by name

Selector Arguments:
- distance=..10: within 10 blocks
- x=100,y=64,z=100,distance=..5: near specific coordinates  
- level=10..20: experience levels 10-20
- gamemode=creative: creative mode players only
日本語補足: ターゲット指定子は柔軟に使えます。@aで全員、プレイヤー名で個別指定可能です

BLOCK STATES AND PROPERTIES:
- Syntax: block_type[property=value]
- Example: lantern[hanging=true]
- Multiple properties: block_type[prop1=value1,prop2=value2]
- Common properties: facing, waterlogged, lit, open, powered
日本語補足: ブロックの状態（点灯・向きなど）はプロパティで指定します。
細かい制御が可能です

COORDINATE SYSTEMS:
- Absolute: <x> <y> <z> (exact world position)
- Relative: ~ (current position), ~1 (+1 offset), ~-1 (-1 offset)
- Local: ^left ^up ^forward (relative to entity facing)
- Coordinate ranges are INCLUSIVE: ~0 to ~15 = 16 blocks total
日本語補足: ~は便利ですが誤差で大規模建築にズレが出やすいです。
基本は絶対座標を使うのが安全です

HIGH RISK OPERATIONS - USE WITH EXTREME CAUTION:
- fill with large ranges (>1000 blocks)
- clone operations affecting existing builds
- kill @e (kills ALL entities including items)
- /stop or /restart commands
日本語補足: 大規模なfill・clone操作や全エンティティ削除は
既存建築を破壊する危険があります

SAFETY CHECKLIST BEFORE MAJOR OPERATIONS:
1. Get player position and survey area
2. Calculate exact block count affected
3. Verify all block types exist (especially modded blocks)
4. Confirm operation with user if affecting >100 blocks
5. Provide undo method or backup strategy
日本語補足: 大規模操作前は必ず範囲確認・ブロック存在確認・
バックアップ戦略を用意してください

COMMON GOTCHAS TO AVOID:
- Never use large relative fills (~) for structures
- Remember both corners are inclusive in fill/clone ranges
- Some commands need player context (e.g., locate)
- Test modded blocks before using in large operations
- Coordinates Y<-64 or Y>320 may be invalid in some versions
日本語補足: よくある失敗は「相対座標で建築してズレる」
「範囲指定を誤って破壊」「存在しないブロック指定」です

EMERGENCY FIXES:
- Undo fill: fill <x1> <y1> <z1> <x2> <y2> <z2> air
- Restore player: tp <player> 0 100 0
- Clear effects: effect clear @a
- Reset weather: weather clear
日本語補足: 緊急時は該当範囲をairで埋める、
プレイヤーを安全な場所にテレポートなどで対処します

VERSION COMPATIBILITY NOTES:
- 1.20.5+: New item component syntax with brackets
- 1.19+: Deep dark blocks (sculk family)
- 1.17+: Caves & cliffs blocks, extended height limits
- 1.16+: Nether update blocks
- 1.13+: Block ID flattening (minecraft:stone vs stone)
日本語補足: バージョンによりブロックIDや構文が異なります。特に1.13以降のフラット化と1.20.5以降のアイテム構文変更に注意してください

Always prioritize safety over convenience. When in doubt, use smaller operations and absolute coordinates.
Always explain what each command does and potential risks to the user.
            """
```