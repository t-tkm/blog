+++ 
Categories = ["AWS"] 
Tags = ["AWS", "VPN", "Transit Gateway", "AWS Site-to-Site VPN", "CDK"] 
date = "2024-03-29T00:00:00+09:00" 
title = "お手軽にAWS Site-to-Site VPNを試してみよう(AWSシミュレーション環境)" 
archives = ["2024", "2024-03", "2024-03-29"]
+++

# はじめに
自宅環境とAWS間を、お手軽にAWS Site-to-Site VPNで接続する方法を紹介したいと思います。
仕事がら、たまに自分の環境でVPNを張り、いろいろ実験したいことが稀にあります。
しかし、常時VPNを接続するのは費用も安く無いため(「付録」参照)、実験したい時にアドホックにVPNを
構築したいわけです。そこで、筆者がVPN構築に使っている方法を、(自身の備忘録も兼ねて)
紹介したいと思います。

# コンテンツ
前編は、純粋にAWS環境だけでVPNをシミュレートする方式で、後編で自宅とAWSをVPNルータ
を用いる方法を説明します。

- [お手軽にAWS Site-to-Site VPNを試してみよう(AWSシミュレーション環境)](https://t-tkm.github.io/blog/posts/2024/03/aws_vpn/) ←本編
- [お手軽にAWS Site-to-Site VPNを試してみよう(自宅VPN環境)](https://t-tkm.github.io/blog/posts/2024/03/aws_vpn2/)

# NW構成
AWS Site-to-Site VPNのネットワークトポロジーには、いくつかバリエーションがあります([AWS Transit Gateway + AWS Site-to-Site VPN](https://docs.aws.amazon.com/whitepapers/latest/aws-vpc-connectivity-options/aws-transit-gateway-vpn.html))。
ここでは「Transit Gateway」を活用したトポロジーを使います。

{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img20.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img20.png">}}

## AWSシミュレーション環境
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img1.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img1.png">}}


# 構築
こちらの記事と、サンプルコード(CDK)を活用します。
- [AWS Blog: "Simulating Site-to-Site VPN Customer Gateways Using strongSwan"](https://aws.amazon.com/jp/blogs/networking-and-content-delivery/simulating-site-to-site-vpn-customer-gateways-strongswan/)
- [GitHub: aws-samples/aws-cdk-simulated-vpn](https://github.com/aws-samples/aws-cdk-simulated-vpn/tree/main)

AWS初学者の方で、上記が難しいと思われる方は、下記「ハンズオンセミナー」がお勧めです。
解説動画に従って、ステップバイステップで構築(理解)進めることができます。

<u>AWS Hands-on for Beginners - Network編#2</u>

[Amazon VPC間およびAmazon VPCとオンプレミスのプライベートネットワーク接続](https://pages.awscloud.com/JAPAN-event-OE-Hands-on-for-Beginners-Network2-2022-reg-event.html)

READMEの「Deploy the stack」に従えば良いですが、筆者の環境では下記の
エラーでデプロイが失敗しました。

{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img17.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img17.png">}}

そこで、aws-cdkとaws-cdk-libのバージョンを上げて対応しました。デプロイにかかった時間は、15分程度でした(950.76s)。

```sh
takumi@iMac % asdf list nodejs
  10.3.0
  16.20.0
 *18.10.0

takumi@iMac % git clone https://github.com/aws-samples/aws-cdk-simulated-vpn.git
takumi@iMac % cd aws-cdk-simulated-vpn
takumi@iMac % npm install
takumi@iMac % cd lib/runtime/describeGatewayConfig
takumi@iMac % npm install
takumi@iMac % cd ../../..

takumi@iMac % npx cdk deploy VpnSimStack
This CDK CLI is not compatible with the CDK library used by your application. Please upgrade the CLI to the latest version.
(Cloud assembly schema version mismatch: Maximum schema version supported is 21.0.0, but found 31.0.0)

# 最新版をインストール
takumi@iMac % npm install aws-cdk@latest

changed 1 package, and audited 412 packages in 1s

# とりあえず更新
takumi@iMac % npm audit fix --force     
takumi@iMac % npm WARN using --force Recommended protections disabled.

changed 1 package, and audited 412 packages in 852ms

# 再度実行。上手くいった模様。
takumi@iMac % npx cdk deploy VpnSimStack

✨  Synthesis time: 5.72s

(snip)

✨  Total time: 950.76s

# テスト用EC2をデプロイ
takumi@iMac % npx cdk deploy ComputeStack
```

package.jsonの差分↓
```zsh {hl_lines=["10-11", "18-19"]}
takumi@iMac % git diff package.json
diff --git a/package.json b/package.json
index b875068..7290e77 100644
--- a/package.json
+++ b/package.json
@@ -14,14 +14,14 @@
     "@types/jest": "^27.5.2",
     "@types/node": "10.17.27",
     "@types/prettier": "2.6.0",
-    "aws-cdk": "2.50.0",
+    "aws-cdk": "^2.133.0",
     "jest": "^27.5.1",
     "ts-jest": "^27.1.4",
     "ts-node": "^10.9.1",
     "typescript": "~3.9.7"
   },
   "dependencies": {
-    "aws-cdk-lib": "2.80.0",
+    "aws-cdk-lib": "^2.133.0",
     "cdk-nag": "^2.22.37",
     "constructs": "^10.0.0",
     "source-map-support": "^0.5.21"
```

# 実験(遅延/帯域)
2つのテストサーバ(Amazon Linux2)にログインし、iperf3をインストールします。
- 受信側: OnPremisInstance(10.0.0.241)
- 送信側: CloudInstance(10.10.0.41)

```zsh
sh-4.2$ sudo yum update -y && sudo yum install -y iperf3
```
 {{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img22.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img22.png">}}

先に、受信側でiperf3サーバを起動し待ち受けます。
```sh
sh-4.2$ ip a
(snip)
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 06:60:92:e9:fa:5f brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.241/24 brd 10.0.0.255 scope global dynamic eth0
       valid_lft 3503sec preferred_lft 3503sec
(snip)

sh-4.2$ iperf3 -s
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
```

SGの5201ポート(TCP)を許可しておきます。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img13.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img13.png">}}

送信側へログインし、測定します。
```sh
sh-4.2$ ip a
(snip)
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 06:29:0b:ce:49:f7 brd ff:ff:ff:ff:ff:ff
    inet 10.10.0.41/24 brd 10.10.0.255 scope global dynamic eth0
       valid_lft 2831sec preferred_lft 2831sec
(snip)

# 遅延確認
sh-4.2$ ping -c 10.0.0.241
PING 10.0.0.241 (10.0.0.241) 56(84) bytes of data.
64 bytes from 10.0.0.241: icmp_seq=1 ttl=253 time=2.54 ms
64 bytes from 10.0.0.241: icmp_seq=2 ttl=253 time=2.27 ms
64 bytes from 10.0.0.241: icmp_seq=3 ttl=253 time=2.33 ms
64 bytes from 10.0.0.241: icmp_seq=4 ttl=253 time=2.22 ms
64 bytes from 10.0.0.241: icmp_seq=5 ttl=253 time=2.46 ms

--- 10.0.0.241 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4006ms
rtt min/avg/max/mdev = 2.241/2.368/2.545/0.125 ms

# 帯域確認
sh-4.2$ iperf3 -c 10.0.0.241
Connecting to host 10.0.0.241, port 5201
[  4] local 10.10.0.41 port 53644 connected to 10.0.0.241 port 5201
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-1.00   sec   120 MBytes  1.00 Gbits/sec  212    777 KBytes
[  4]   1.00-2.00   sec   109 MBytes   912 Mbits/sec    0    872 KBytes
[  4]   2.00-3.00   sec   100 MBytes   839 Mbits/sec    0    950 KBytes
[  4]   3.00-4.00   sec   115 MBytes   965 Mbits/sec    0   1.01 MBytes
[  4]   4.00-5.00   sec   122 MBytes  1.03 Gbits/sec    0   1.09 MBytes
[  4]   5.00-6.00   sec   118 MBytes   986 Mbits/sec  414    628 KBytes
[  4]   6.00-7.00   sec   111 MBytes   933 Mbits/sec    0    747 KBytes
[  4]   7.00-8.00   sec   112 MBytes   944 Mbits/sec    0    848 KBytes
[  4]   8.00-9.00   sec   102 MBytes   860 Mbits/sec   13    670 KBytes
[  4]   9.00-10.00  sec   111 MBytes   933 Mbits/sec    0    780 KBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-10.00  sec  1.09 GBytes   940 Mbits/sec  639             sender
[  4]   0.00-10.00  sec  1.09 GBytes   938 Mbits/sec                  receiver

iperf Done.
```
流石にAWS内ネットワークだけあり、十分な速度がでています。

# 遅延/帯域制御
オンプレとAWS間のネットワーク性能(遅延/帯域)について、より現実的なシミュレーション
を行う場合は、Linuxのtcコマンド(Traffic Control)を活用できます。筆者の環境では、
自宅は横浜にあり、以前の測定では遅延が10ms〜15ms、帯域は20Mbps程度でした。

従って、下記仮想VPNルータに対し、tcコマンドにて遅延15ms、帯域は20Mbps程度の
制限を設定します。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img25.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img25.png">}}

```zsh
# tcコマンドインストール
sh-4.2$ sudo yum -y install iproute-tc

# NICのTCによる制限状態確認(default)
sh-4.2$ sudo tc qdisc show dev eth0
qdisc mq 0: root
qdisc pfifo_fast 0: parent :2 bands 3 priomap 1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1
qdisc pfifo_fast 0: parent :1 bands 3 priomap 1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1

# 下記2つのコマンドで、遅延と帯域制限を設定
# 遅延設定(片道7msの遅延挿入)
sudo tc qdisc add dev eth0 root handle 1:0 netem delay 7ms
# 帯域設定(20Mbps制限)
sudo tc qdisc add dev eth0 parent 1:1 handle 10:0 tbf rate 20mbit burst 50kb limit 500kb
```

片道7ms(往復14ms)、20Mbpsの制限が設定できました。
```zsh
# NICのTCによる制限状態確認
sh-4.2$ sudo tc qdisc show dev eth0
qdisc netem 1: root refcnt 3 limit 1000 delay 7ms
qdisc tbf 10: parent 1:1 rate 20Mbit burst 50Kb lat 184ms
```
# 再実験(遅延/帯域)
先の実験同様、送信側へログインし、測定します。
```zsh
sh-4.2$ ping 10.0.0.241
PING 10.0.0.241 (10.0.0.241) 56(84) bytes of data.
64 bytes from 10.0.0.241: icmp_seq=1 ttl=253 time=16.5 ms
64 bytes from 10.0.0.241: icmp_seq=2 ttl=253 time=16.4 ms
64 bytes from 10.0.0.241: icmp_seq=3 ttl=253 time=16.4 ms
64 bytes from 10.0.0.241: icmp_seq=4 ttl=253 time=16.4 ms
64 bytes from 10.0.0.241: icmp_seq=5 ttl=253 time=16.4 ms
64 bytes from 10.0.0.241: icmp_seq=6 ttl=253 time=16.4 ms
64 bytes from 10.0.0.241: icmp_seq=7 ttl=253 time=16.4 ms
^C
--- 10.0.0.241 ping statistics ---
7 packets transmitted, 7 received, 0% packet loss, time 6011ms
rtt min/avg/max/mdev = 16.432/16.478/16.543/0.033 ms

sh-4.2$ iperf3 -c 10.0.0.241
Connecting to host 10.0.0.241, port 5201
[  4] local 10.10.0.178 port 52894 connected to 10.0.0.241 port 5201
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-1.00   sec  2.83 MBytes  23.7 Mbits/sec    1    343 KBytes
[  4]   1.00-2.00   sec  2.44 MBytes  20.5 Mbits/sec    0    459 KBytes
[  4]   2.00-3.00   sec  2.32 MBytes  19.5 Mbits/sec    0    559 KBytes
[  4]   3.00-4.00   sec  2.08 MBytes  17.4 Mbits/sec   33    433 KBytes
[  4]   4.00-5.00   sec  2.69 MBytes  22.6 Mbits/sec    0    491 KBytes
[  4]   5.00-6.00   sec  2.02 MBytes  16.9 Mbits/sec    0    532 KBytes
[  4]   6.00-7.00   sec  2.02 MBytes  16.9 Mbits/sec    0    559 KBytes
[  4]   7.00-8.00   sec  2.44 MBytes  20.5 Mbits/sec    4    401 KBytes
[  4]   8.00-9.00   sec  2.02 MBytes  16.9 Mbits/sec    0    435 KBytes
[  4]   9.00-10.00  sec  2.02 MBytes  16.9 Mbits/sec    0    462 KBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-10.00  sec  22.9 MBytes  19.2 Mbits/sec   38             sender
[  4]   0.00-10.00  sec  21.8 MBytes  18.3 Mbits/sec                  receiver

iperf Done.
```
性能面でも現実的なシミュレーションができました。

それでは、次は実際のインターネット回線を介したVPN接続を
設定したいと思います。

[後編へ→](https://t-tkm.github.io/blog/posts/2024/03/aws_vpn2/)

# 付録
VPN料金は、おおよそ1,200円(8USD)/日程度でした。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img24.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img24.png">}}

※テストサーバを停止し忘れていました。。。