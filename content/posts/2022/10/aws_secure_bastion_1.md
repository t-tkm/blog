+++ 
Categories = ["AWS"] 
Tags = ["AWS", "Terraform", "Bastion"] 
date = "2022-10-16T00:00:00+09:00" 
title = "踏台VPCの作成(Internet Gateway無し)(AWS)" 
archives = ["2022", "2022-10", "2022-10-16"]
+++

# はじめに
VPCにInternet GWを割当てず、プライベートサブネットにあるサーバ(EC2)を操作する方法です。Cloud9(Amazon Linux)と、Windows Server(ブラウザを使ったGUI操作)を説明します。

[参考][Systems Manager を使用してインターネットアクセスなしでプライベート EC2 インスタンスを管理できるように、VPC エンドポイントを作成するにはどうすればよいですか?](https://aws.amazon.com/jp/premiumsupport/knowledge-center/ec2-systems-manager-vpc-endpoints/)

# 構成
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img1.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img1.png?raw=true">}}

# 基盤構築(Terraform)
ネットワークなどの基盤構築には、Terraformを活用しました。コードは[こちら(GitHub)](https://github.com/t-tkm/AWS-Secure-Bastion.git)からダウンロードできます。
* VPCパラメータ
    ```tf
    resource "aws_vpc" "Main" {
    cidr_block       = var.main_vpc_cidr
    instance_tenancy = "default"
    enable_dns_hostnames = true
    enable_dns_support   = true
    tags = {
            Name = "${var.system_name}-vpc"
        }
    }
    ```

* Bastionサブネットパラメータ
    ```tf
    resource "aws_subnet" "private_subnet_bastion" {
    vpc_id =  aws_vpc.Main.id
    availability_zone = "${var.az1}"
    cidr_block = "${var.private_subnet_bastion}"
    tags = {
            Name = "${var.system_name}-private-bastion-subnet-1a"
        }
    }
    
    resource "aws_route_table" "PrivateRT_bastion" {
    vpc_id = aws_vpc.Main.id
    tags = {
            Name = "${var.system_name}-private-bastion-route-table"
        }

    }

    resource "aws_route_table_association" "PrivateRTassociation_bastion" {
        subnet_id = aws_subnet.private_subnet_bastion.id
        route_table_id = aws_route_table.PrivateRT_bastion.id
    }

    resource "aws_security_group" "private_bastion_sg" {
        name = "${var.system_name}-private-bastion-sg"
        vpc_id = aws_vpc.Main.id
        egress {
            from_port = 0
            to_port = 0
            protocol = "-1"
            cidr_blocks = ["0.0.0.0/0"]
        }
        description = "${var.system_name}-private-bastion-sg"
    }
    ```

* Egressサブネットパラメータ
    ```tf
    resource "aws_subnet" "private_subnet_egress" {
    vpc_id =  aws_vpc.Main.id
    availability_zone = "${var.az1}"
    cidr_block = "${var.private_subnet_egress}"
    tags = {
            Name = "${var.system_name}-private-egress-subnet-1a"
        }
    }
    
    resource "aws_route_table" "PrivateRT_egress" {
    vpc_id = aws_vpc.Main.id
    tags = {
            Name = "${var.system_name}-private-egress-route-table"
        }
    }

    resource "aws_route_table_association" "PrivateRTassociation_egress" {
        subnet_id = aws_subnet.private_subnet_egress.id
        route_table_id = aws_route_table.PrivateRT_egress.id
    }

    resource "aws_security_group" "private_egress_sg" {
        name = "${var.system_name}-private-egress-sg"
        vpc_id = aws_vpc.Main.id
        ingress {
            from_port = 443
            to_port = 443
            protocol = "tcp"
            cidr_blocks = ["${var.main_vpc_cidr}"]
        }
        egress {
            from_port = 0
            to_port = 0
            protocol = "-1"
            cidr_blocks = ["0.0.0.0/0"]
        }
        description = "${var.system_name}-private-egress-route-sg"
    }
    ```

* VPCエンドポイントパラメータ
    ```tf
    resource "aws_vpc_endpoint" "ssm" {
    vpc_id              = aws_vpc.Main.id
    service_name        = "com.amazonaws.${var.region}.ssm"
    vpc_endpoint_type   = "Interface"
    subnet_ids = [
        aws_subnet.private_subnet_egress.id
    ]
    security_group_ids = [
        aws_security_group.private_egress_sg.id
    ]
    private_dns_enabled = true
    tags = {
            Name = "vpce_ssm"
        }
    }

    resource "aws_vpc_endpoint" "ssmmessages" {
    vpc_id              = aws_vpc.Main.id
    service_name        = "com.amazonaws.${var.region}.ssmmessages"
    vpc_endpoint_type   = "Interface"
    subnet_ids = [
        aws_subnet.private_subnet_egress.id
    ]
    security_group_ids = [
        aws_security_group.private_egress_sg.id
    ]
    private_dns_enabled = true
    tags = {
            Name = "vpce_ssmmessages"
        }
    }

    resource "aws_vpc_endpoint" "ec2messages" {
    vpc_id              = aws_vpc.Main.id
    service_name        = "com.amazonaws.${var.region}.ec2messages"
    vpc_endpoint_type   = "Interface"
    subnet_ids = [
        aws_subnet.private_subnet_egress.id
    ]
    security_group_ids = [
        aws_security_group.private_egress_sg.id
    ]
    private_dns_enabled = true
    tags = {
            Name = "vpce_ec2messages"
        }
    }
    ```

# Linux Bastion(Cloud9)構築
* Cloud9のサービス画面より「Create environment」を選択。名前を適当に入力します。ここでは「secure-bastion-cloud9」としました。
{{< figure alt="img3" src="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img3.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img3.png?raw=true">}}

* 次に、次の2点に注意して設定します:
  * 「Environment type」->オープンなインバウンドポートを必要としないSystems Manager経由のアクセスオプションを選択
  * 「Network settings (advanced)」->前節で作成したVPCとサブネットを指定します。セキュリティグループはCloud9用のデフォルトが設定されますので、必要に応じ、後から(自身のセキュリティグループを割当てるなど)更新してください。
{{< figure alt="img4" src="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img4.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img4.png?raw=true">}}

* 確認画面で問題なければ「Create environment」をクリック。後は、バックでCloud Formationのスタック経由でリソース(EC2)が作成されます。
{{< figure alt="img5" src="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img5.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img5.png?raw=true">}}

* しばらく待ちます(2〜3分程度)。この画面が終了しない場合は、Engressサブネットの設定に問題がある可能性があります。
{{< figure alt="img6" src="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img6.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img6.png?raw=true">}}

* 無事にCloud9が起動しました。
{{< figure alt="img7" src="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img7.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img7.png?raw=true">}}

* インターネットへのアウトバウンド通信経路が無いので、パッケージを更新したり(yum update)、curlでインターネット上のサイトへ接続でき無い事がわかります。

    ```bash
    $ sudo yum update
    Loaded plugins: extras_suggestions, langpacks, priorities, update-motd

    ^C

    Exiting on user cancel
    $ curl https://www.google.com
    ^C
    ```


# Windows Bastion構築
* EC2作成画面より、OSとしてWindows Serverを選択します。
{{< figure alt="img8" src="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img8.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img8.png?raw=true">}}

* 新しくキーペアを作成しました。ネットワークは、上で作成したリソースを指定します。Cloud9とは異なり、セキュリティグループも指定可能でした。
{{< figure alt="img9" src="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img9.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img9.png?raw=true">}}

* EC2がSystems Managerへアクセスできるように、ロールを設定してあげます。
{{< figure alt="img10" src="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img10.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img10.png?raw=true">}}

* 簡単に、AWSマネージドなロール「AmazonSSMRoleForInstancesQuickSetup」を使いました。
{{< figure alt="img11" src="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img11.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img11.png?raw=true">}}

* しばらくすると、Windows Server上のSSM Agentが無事にSystems Managerへ連携し、マネージドインスタンスとして確認できるようになります。※(関係があるか不明ですが)筆者の環境では、10分程度してもマネージドインスタンスに表示されなかったため、EC2を再起動したり、後述「接続」からWindows Serverのパスワード取得をしてみたり、うろうろしているうちに、表示されました。
{{< figure alt="img12" src="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img12.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img12.png?raw=true">}}

* EC2の一覧画面より、該当するWindows Serverを選択し、「接続」をクリックします。
{{< figure alt="img12" src="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img13.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img13.png?raw=true">}}

* Fleet Manager経由でのRDP接続を選択します。
{{< figure alt="img12" src="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img14.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img14.png?raw=true">}}

* 今回はキーペアで接続しました。(プライベートキー「t-tkm-private-key.cer」は、ローカルPC上にあります)
{{< figure alt="img12" src="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img15.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img15.png?raw=true">}}

* 複数インスタンスがリスト表示されます(ここでは1つですが。。)
{{< figure alt="img12" src="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img16.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img16.png?raw=true">}}

* 特定インスタンスのRDPをブラウザから操作できます。
{{< figure alt="img12" src="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img17.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/main/2022/aws_secure_bastion/img17.png?raw=true">}}

以上になります。次回は、このVPCにNetwork Firewallを経由させてインターネットアウトバウンド通信を行う構成を見ていきます。