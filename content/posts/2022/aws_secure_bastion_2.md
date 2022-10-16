+++ 
Categories = ["AWS"] 
Tags = ["AWS", "Terraform", "Bastion"] 
date = "2022-10-16T00:00:00+09:00" 
title = "踏台VPCの作成(Internet Gateway有り)(AWS)" 
+++

# はじめに
前回は、VPCにInternet Gatewayを設置しない完全にプライベートな環境での、ブラウザを使うEC2操作を見ました。
しかしこのままでは、yumの更新、npmなど各種パッケージ経由でのツール導入、(インターネットにあるRESTエンドポイントを使う)AWS CLI操作などできません。

そこで今回は、インターネットへのアウトバウンド通信を許可するためにNAT GWを設置しようと思います。トラフィックを少し管理するために。Network Firewall
というIPS/IDS相当のサービスを設置したいと思います。

なお、今回の検証で、Network Firewallは1,400円/日程度の費用が発生しました。(1 endpoints x 24 hours x 0.395 USD = 9.48 USD 148円/USD 換算)

# 参考
1. [[AWS Black Belt Online Seminar] AWS Network Firewall 入門 資料公開](https://aws.amazon.com/jp/blogs/news/webinar-bb-awsnetworkfirewallintroduction-2021/)
2. [[AWS Black Belt Online Seminar]AWS Network Firewall 応用編1 資料公開](https://aws.amazon.com/jp/blogs/news/webinar-bb-aws-network-firewall-advanced-part1-2021/)
3. [AWSブログ: 【開催報告】アップデート紹介とちょっぴり DiveDeep する AWS の時間 第二十二回 (09/29)](https://aws.amazon.com/jp/blogs/news/update-divedeep-series-22/)
4. [AWSアーキテクチャセンター: AWS Network Firewall を使用した検査デプロイモデル](https://d1.awsstatic.com/architecture-diagrams/ArchitectureDiagrams/inspection-deployment-models-with-AWS-network-firewall-ra.pdf?did=wp_card&trk=wp_card)

# 構成
{{< figure alt="img2" src="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img2.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img2.png?raw=true">}}

# 基盤構築(Terraform)
ネットワークなどの基盤構築には、Terraformを活用しました。コードは[こちら(GitHub)](https://github.com/t-tkm/AWS-Secure-Bastion.git)からダウンロードできます。
こちらのmain.tfコードの「step2」でコメントアウトしている部分が、今回追加されるリソース定義となります。

* Internet Gatewayパラメータ
    ```tf
    resource "aws_internet_gateway" "IGW" {
        vpc_id =  aws_vpc.Main.id
    }
    ```

* パブリックサブネットパラメータ
    ```tf
    resource "aws_subnet" "public_subnet" {
    vpc_id =  aws_vpc.Main.id
    availability_zone = "${var.az1}"
    cidr_block = "${var.public_subnet}"
    tags = {
            Name = "secure-public-1a"
        }
    }

    resource "aws_route_table" "PublicRT" {
        vpc_id =  aws_vpc.Main.id
        route {
            cidr_block = "0.0.0.0/0"
            gateway_id = aws_internet_gateway.IGW.id
        }
    tags = {
            Name = "secure-public-rt"
        }
    }

    resource "aws_route_table_association" "PublicRTassociation" {
        subnet_id = aws_subnet.public_subnet.id
        route_table_id = aws_route_table.PublicRT.id
    }

    resource "aws_eip" "nateIP" {
    vpc   = true
    }

    resource "aws_nat_gateway" "NATgw" {
    allocation_id = aws_eip.nateIP.id
    subnet_id = aws_subnet.public_subnet.id
    }
    ```

* Firewallサブネットパラメータ
    ```tf
    resource "aws_subnet" "private_subnet_firewall" {
    vpc_id =  aws_vpc.Main.id
    availability_zone = "${var.az1}"
    cidr_block = "${var.private_subnet_firewall}"
    tags = {
            Name = "secure-private-firewall-1a"
        }
    }
    
    resource "aws_route_table" "PrivateRT_firewall" {
    vpc_id = aws_vpc.Main.id
    route {
        cidr_block = "0.0.0.0/0"
        nat_gateway_id = aws_nat_gateway.NATgw.id
    }
    tags = {
            Name = "secure-private-firewall-rt"
        }
    }

    resource "aws_route_table_association" "PrivateRTassociation_firewall" {
        subnet_id = aws_subnet.private_subnet_firewall.id
        route_table_id = aws_route_table.PrivateRT_firewall.id
    }
    ```

* Network Firewallパラメータ
    ```tf
    resource "aws_networkfirewall_rule_group" "my_ips" {
    capacity = 100
    name     = "example"
    type     = "STATEFUL"
    rule_group {
        rules_source {
        rules_source_list {
            generated_rules_type = "DENYLIST"
            target_types         = ["HTTP_HOST"]
            targets              = ["www.yahoo.co.jp"]
        }
        }
    }
    }

    resource "aws_networkfirewall_firewall_policy" "firewall_policy" {
    name = "network-firewall-policy"

    firewall_policy {
        stateless_default_actions          = ["aws:forward_to_sfe"]
        stateless_fragment_default_actions = ["aws:forward_to_sfe"]
        stateful_rule_group_reference {
        resource_arn = aws_networkfirewall_rule_group.my_ips.arn
        }
    }
    }

    resource "aws_networkfirewall_firewall" "firewall" {
    name                = "network-firewall"
    firewall_policy_arn = aws_networkfirewall_firewall_policy.firewall_policy.arn
    vpc_id              = aws_vpc.Main.id

    subnet_mapping {
        subnet_id     = aws_subnet.private_subnet_firewall.id
    }
    }

    resource "aws_s3_bucket" "example" {
    bucket = "t-tkm-firewall-logs"
    }

    resource "aws_s3_bucket_acl" "example" {
    bucket = aws_s3_bucket.example.id
    acl    = "private"
    }

    resource "aws_networkfirewall_logging_configuration" "firewall_logging" {
    firewall_arn = aws_networkfirewall_firewall.firewall.arn
    logging_configuration {
        log_destination_config {
        log_destination = {
            bucketName = aws_s3_bucket.example.bucket
            prefix     = "flow"
        }
        log_destination_type = "S3"
        log_type             = "FLOW"
        }
    log_destination_config {
        log_destination = {
            bucketName = aws_s3_bucket.example.bucket
            prefix     = "alert"
        }
        log_destination_type = "S3"
        log_type             = "ALERT"
        }
    }
    }
    ```

# ルートテーブル設定(手動)
ルートテーブルのエントリにFirewall Endpointを指定する必要があり、TFの[ドキュメント](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/networkfirewall_firewall)より「endpoint_id」を使えばよさそうですが、GitHubの[issue](https://github.com/hashicorp/terraform-provider-aws/issues/16759)にあるように、なかなか簡単に設定できずorz...手動で設定することにしました。

* この部分を手動で設定します。
{{< figure alt="img18" src="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img18.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img18.png?raw=true">}}

* 該当するルートテーブルは2箇所です。
{{< figure alt="img19" src="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img19.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img19.png?raw=true">}}

* 作成されたエンドポイントタイプは「GatewayLoadBalancer」で、(他のよくみるENIの「Interface」やS3などの「Gateway」と異なり)少し特別にみえます。 
{{< figure alt="img20" src="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img20.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img20.png?raw=true">}}

* このようになっていればokです。
{{< figure alt="img21" src="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img21.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img21.png?raw=true">}}

以上で完了です。

* 注意点としては、(当然)TFの設定とドリフトするため、terraform applyすると元に戻ります。ただ、パブリックサブネットの方はドリフトを検知してますが、プライベートサブネットは(作成パラメータにrouteを明に記載しておらず)ドリフトとして検知されてないようです。いろいろと危険なので、本番環境ではちゃんと設計する必要があります。
{{< figure alt="img22" src="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img22.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img22.png?raw=true">}}

# インターネット接続検証
それでは、BationサブネットにあるCloud9から、インターネット(アウトバウンド)通信ができるか確認します。

```bash
AWSReservedSSO_AWSAdministratorAccess_6c2a494b14560455:~/environment $ sudo yum update
Loaded plugins: extras_suggestions, langpacks, priorities, update-motd
amzn2-core                                                                                                                                                | 3.7 kB  00:00:00     
amzn2extra-docker                                                                                                                                         | 3.0 kB  00:00:00     
amzn2extra-epel                                                                                                                                           | 3.0 kB  00:00:00     
amzn2extra-lamp-mariadb10.2-php7.2                                                                                                                        | 3.0 kB  00:00:00     
hashicorp                                                                                                                                                 | 1.4 kB  00:00:00     
(1/4): amzn2-core/2/x86_64/group_gz                                                                                                                       | 2.5 kB  00:00:00     
(2/4): amzn2-core/2/x86_64/updateinfo                                                                                                                     | 505 kB  00:00:00     
(3/4): hashicorp/x86_64/primary                                                                                                                           | 112 kB  00:00:00     
(4/4): amzn2-core/2/x86_64/primary_db                                                                                                                     |  66 MB  00:00:01     
hashicorp                                                                                                                                                                799/799
255 packages excluded due to repository priority protections
Resolving Dependencies
--> Running transaction check
---> Package golang.x86_64 0:1.18.3-1.amzn2 will be updated
(snip)
```
ちゃんと、yumの更新ができました。

# おまけ
S3に保存されているログを確認してみましょう。上記のTerraformの設定パラメータの中で、www.yahoo.co.jpに対するHTTPを拒否する設定をしています。試しに、2つのサーバからアクセスすると、ちゃんとブロック(接続不可)されていました。
* Cloud9からcurlでhttp://www.yahoo.co.jp
* Windows Serverのブラウザ(Edge)からhttp://www.yahoo.co.jpへアクセス

S3のアラートログに、このように証跡が残ります。
```json
{
    "firewall_name": "network-firewall",
    "availability_zone": "ap-northeast-1a",
    "event_timestamp": "1665911043",
    "event": {
        "timestamp": "2022-10-16T09:04:03.390268+0000",
        "flow_id": 506371426733738,
        "event_type": "alert",
        "src_ip": "10.0.20.91",
        "src_port": 50820,
        "dest_ip": "183.79.219.252",
        "dest_port": 80,
        "proto": "TCP",
        "tx_id": 0,
        "alert": {
            "action": "blocked",
            "signature_id": 1,
            "rev": 1,
            "signature": "matching HTTP denylisted FQDNs",
            "category": "",
            "severity": 1
        },
        "http": {
            "hostname": "www.yahoo.co.jp",
            "url": "/",
            "http_user_agent": "curl/7.79.1",
            "http_method": "GET",
            "protocol": "HTTP/1.1",
            "length": 0
        },
        "app_proto": "http"
    }
}
{
    "firewall_name": "network-firewall",
    "availability_zone": "ap-northeast-1a",
    "event_timestamp": "1665911055",
    "event": {
        "timestamp": "2022-10-16T09:04:15.530778+0000",
        "flow_id": 651244969364562,
        "event_type": "alert",
        "src_ip": "10.0.20.243",
        "src_port": 50489,
        "dest_ip": "182.22.16.251",
        "dest_port": 80,
        "proto": "TCP",
        "tx_id": 0,
        "alert": {
            "action": "blocked",
            "signature_id": 1,
            "rev": 1,
            "signature": "matching HTTP denylisted FQDNs",
            "category": "",
            "severity": 1
        },
        "http": {
            "hostname": "www.yahoo.co.jp",
            "url": "/",
            "http_user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/105.0.0.0 Safari/537.36 Edg/105.0.1343.33",
            "http_method": "GET",
            "protocol": "HTTP/1.1",
            "length": 0
        },
        "app_proto": "http"
    }
}
```

他のログと同じように、Athenaを用いて分析もできます。
https://docs.aws.amazon.com/ja_jp/athena/latest/ug/querying-network-firewall-logs.html