+++ 
Categories = ["AWS"] 
Tags = ["AWS", "VPN", "Transit Gateway", "AWS Site-to-Site VPN", "CDK"] 
date = "2024-03-29T00:00:00+09:00" 
title = "お手軽にAWS Site-to-Site VPNを試してみよう(自宅VPN環境)" 
archives = ["2024", "2024-03", "2024-03-29"]
+++

# はじめに
前編で構築したVPN環境に、自宅VPNルータを追加してみようと思います。

- [お手軽にAWS Site-to-Site VPNを試してみよう(AWSシミュレーション環境)](https://t-tkm.github.io/blog/posts/2024/03/aws_vpn/)
- [お手軽にAWS Site-to-Site VPNを試してみよう(自宅VPN環境)](https://t-tkm.github.io/blog/posts/2024/03/aws_vpn2/) ←本編

筆者の環境では、VPNルータとして[TP-Link Omada ギガビット マルチWAN VPNルーター ER605](https://www.amazon.co.jp/dp/B08MH4VLR3/)
を使っています(こちらは2022年6月12日に8,800円で購入しました)。

{{< figure alt="img1" width="300" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img19.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img19.png">}}

# 自宅VPN環境
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img2.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img2.png">}}

# 構築
基本的に、こちらの記事に従って構築させていただきました。

> [8800円の格安VPNルータで家とAWS間をサイト間VPN（IPSec）で接続する](https://level69.net/archives/30177)

※「VPN接続」「Transit GW」周りが少しだけ異なりますので、適宜確認お願いします。

<u>UTM - ER605間接続</u>
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img3.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img3.png">}}
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img4.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img4.png">}}

Link Upを確認します。
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img5.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img5.png">}}

<u>E605ファームウェアVUP</u>

ER605のファームウェアバージョンが「ER605(UN)_V1_1.2.0 Build 20220114」以上であれば、IKEv2を使うことができます。
 {{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img6.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img6.png">}}

<u>VPN2設定</u>

CGWを作成します作成
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img7.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img7.png">}}

VPN接続を作成します(AWS)
 {{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img8.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img8.png">}}

CGW(自宅VPNルータ)の設定が未だなので、(当然)Linkダウン状態です
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img9.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img9.png">}}

CGW(VPN設定(ER605))を設定します
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img10.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img10.png">}}

Linkアップしました
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img11.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img11.png">}}

<u>Transit GW設定</u>

Transit GWのルートテーブルに、静的ルートを作成します
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img14.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img14.png">}}
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img15.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img15.png">}}

VPC(サブネット)のルートテーブルに、(自宅向け)ルートを追加します
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img16.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img16.png">}}
  
以上で、自宅-AWS間のVPNが設定されました。

# 実験(遅延/帯域)
 {{< figure alt="img1" src="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img23.png" link="https://github.com/t-tkm/blog_images/raw/main/2024/aws_vpn/img23.png">}}

受信側でiperf3サーバを起動し待ち受けます。
```sh
takumi@iMac ~ % ifconfig 
(snip)
en1: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
	options=6460<TSO4,TSO6,CHANNEL_IO,PARTIAL_CSUM,ZEROINVERT_CSUM>
	ether a4:83:e7:2d:39:f5
	inet6 fe80::1cff:f84b:361d:dbda%en1 prefixlen 64 secured scopeid 0x6 
	inet 192.168.10.172 netmask 0xffffff00 broadcast 192.168.10.255
(snip)

takumi@iMac ~ % iperf3 -s
-----------------------------------------------------------
Server listening on 5201 (test #1)
-----------------------------------------------------------
```
送信側へログインし、測定します。
```sh
sh-4.2$ ip a
(snip)
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 06:29:0b:ce:49:f7 brd ff:ff:ff:ff:ff:ff
    inet 10.10.0.41/24 brd 10.10.0.255 scope global dynamic eth0
       valid_lft 2831sec preferred_lft 2831sec
(snip)

sh-4.2$ ping -c 5 192.168.10.172
PING 192.168.10.172 (192.168.10.172) 56(84) bytes of data.
64 bytes from 192.168.10.172: icmp_seq=1 ttl=62 time=7.70 ms
64 bytes from 192.168.10.172: icmp_seq=2 ttl=62 time=10.5 ms
64 bytes from 192.168.10.172: icmp_seq=3 ttl=62 time=97.2 ms
64 bytes from 192.168.10.172: icmp_seq=4 ttl=62 time=7.21 ms
64 bytes from 192.168.10.172: icmp_seq=5 ttl=62 time=40.7 ms

--- 192.168.10.172 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4006ms
rtt min/avg/max/mdev = 7.210/32.699/97.250/34.625 ms

sh-4.2$ iperf3 -c 192.168.10.172
Connecting to host 192.168.10.172, port 5201
[  4] local 10.10.0.41 port 57872 connected to 192.168.10.172 port 5201
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-1.00   sec  2.90 MBytes  24.3 Mbits/sec    5   69.2 KBytes
[  4]   1.00-2.00   sec  2.75 MBytes  23.1 Mbits/sec    7   71.9 KBytes
[  4]   2.00-3.00   sec  2.75 MBytes  23.1 Mbits/sec    2   94.5 KBytes
[  4]   3.00-4.00   sec  3.07 MBytes  25.7 Mbits/sec   10   78.5 KBytes
[  4]   4.00-5.00   sec  3.05 MBytes  25.6 Mbits/sec    2    101 KBytes
[  4]   5.00-6.00   sec  2.32 MBytes  19.5 Mbits/sec    0    117 KBytes
[  4]   6.00-7.00   sec  1.95 MBytes  16.4 Mbits/sec    0    129 KBytes
[  4]   7.00-8.00   sec  2.51 MBytes  21.1 Mbits/sec    2   78.5 KBytes
[  4]   8.00-9.00   sec  2.20 MBytes  18.4 Mbits/sec    3   35.9 KBytes
[  4]   9.00-10.00  sec  2.20 MBytes  18.5 Mbits/sec    1   51.9 KBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-10.00  sec  25.7 MBytes  21.6 Mbits/sec   32             sender
[  4]   0.00-10.00  sec  25.2 MBytes  21.2instalaws Mbits/sec                  receiver

iperf Done.

```
AWSと自宅(横浜)間の通信速度の結果は上記となりました。

# Cleanup
手動で追加した「カスタマーゲートウェア(CGW)」と「Site-to-Site VPN接続(VPN2)」をAWSコンソールから削除します。

その後、CDKリソースを削除します。
```sh
takumi@iMac % cdk destroy ComputeStack
takumi@iMac % cdk destroy VpnSimStack 
```
# まとめ
今回は、GitHubの[AWS Samplesリポジトリ](https://github.com/aws-samples)にある[aws-cdk-simulated-vpn](https://github.com/aws-samples/aws-cdk-simulated-vpn)プロジェクトを活用して、AWS Cloud Development Kit (CDK) を用いた簡単なAWSシミュレーション環境（VPN）の構築方法を紹介しました。また、性能面でより現実的なシミュレーションを行うために、Linuxのtcコマンドを用いたネットワーク遅延やパケットロスのシミュレーションについても解説しました。最後に、実際の自宅環境におけるVPNルータ（TP-Link ER605）を使用した環境構築の例も紹介しました。