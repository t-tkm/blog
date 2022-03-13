+++ 
Categories = ["AWS"] 
Tags = ["AWS", "Hugo"] 
date = "2022-03-12T00:00:00+09:00" 
title = "Hugoでハンズオンサイト(AWS)" 
+++

# はじめに
サンプル画像
{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/blob/aws_hugo/imgs/img1.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/aws_hugo/imgs/img1.png?raw=true">}}

# 参考
- [Deploy and Host Hugo Content on AWS](https://hosting-hugo-content.workshop.aws/)

# command
参考サイト(AWS Workshop)に従ってお試し。

## Hugo Setup
Hugoをインストールします。Goで作られたシングルバイナリなので、ダウンロード後に解凍し、パスの通ったディレクトリへ実行ファイルを置くだけで利用できます。
```bash
$ wget https://github.com/gohugoio/hugo/releases/download/v0.71.0/hugo_extended_0.71.0_Linux-64bit.tar.gz -O hugo_extended_0.71.0_Linux-64bit.tar.gz
$ HUGO_TAR="$(find . -name "*Linux-64bit.tar.gz")"
$ tar -xzf $HUGO_TAR
$ chmod +x hugo
$ mkdir bin
$ mv hugo bin/
$ cd ~/environment/

# Hugoがインストールできているかの確認
$ hugo version
Hugo Static Site Generator v0.71.0-06150C87/extended linux/amd64 BuildDate: 2020-05-18T16:15:29Z
```

## Amplify Setup
次に、Amplify CLIをインストールします。Cloud9はデフォルトでNode.jsがインストール済みなので、npmでパッケージインストールするだけです。
```bash
$ npm install -g @aws-amplify/cli@4.21.1
$ amplify -v
Scanning for plugins...
Plugin scan successful
4.21.1
```

以上で準備が整いました。

# Hugoサイト作成
```bash
# cd into the Cloud9 project directory
$ cd /home/ec2-user/environment

# Create the Hugo project skeleton
$ hugo new site unicorn-inc-smart-factories

# cd into the project folder
$ cd unicorn-inc-smart-factories

$ ll
total 4
drwxrwxr-x 2 ec2-user ec2-user 24 Mar  6 11:24 archetypes
-rw-rw-r-- 1 ec2-user ec2-user 82 Mar  6 11:24 config.toml
drwxrwxr-x 2 ec2-user ec2-user  6 Mar  6 11:24 content
drwxrwxr-x 2 ec2-user ec2-user  6 Mar  6 11:24 data
drwxrwxr-x 2 ec2-user ec2-user  6 Mar  6 11:24 layouts
drwxrwxr-x 2 ec2-user ec2-user  6 Mar  6 11:24 static
drwxrwxr-x 2 ec2-user ec2-user  6 Mar  6 11:24 themes
```

## Git Setup
```bash
# Initialize the Hugo project directory as a GIT Repo
git init

# ブランチ名がmasterだったので、mainへ変更
$ git branch -m master main
$ git config --global user.name "Takumi Tomita"
$ git config --global user.email Takumi.tomita.vf@gmail.com
$ cat ~/.gitconfig 
[credential]
        helper = !aws codecommit credential-helper $@
        UseHttpPath = true
[core]
        editor = nano
[user]
        name = Takumi Tomita
        email = Takumi.tomita.vf@gmail.com
```

```bash
# CD into the themes folder and clone the Hugo learn theme as a GIT sub-module
$ cd themes
$ git submodule add https://github.com/matcornic/hugo-theme-learn.git

# Return to the parent Hugo project folder
$ cd ..
```

config.toml更新

```vim
baseURL = "http://example.org/"
languageCode = "en-us"
title = "Unicorn Inc - Smart Factories Workshop"
theme = "hugo-theme-learn"
uglyurls = true
```

```bash
hugo new --kind chapter _index.md
```

下記を変更
```vim
+++
title = ""
date = 2022-03-06T11:34:31Z
weight = 5
chapter = true
pre = "<b>X. </b>"
+++

### Chapter X

# Some Chapter title

Lorem Ipsum.
```
↓
```vim
---
title: "Unicorn Inc - Smart Factories Workshop"
draft: false
weight: 0
pre: "<b>0. </b>"
---

### Unicorn Inc - Smart Factories Workshop
![Unicorn Inc Logo](/images/unicorn-inc.png?classes=border)
# Home

Welcome to Unicorn Inc - Smart Factories Workshop.
Add further content here......
```

```bash
# Create the images directory in the static folder
$ mkdir -p static/images
# wget the image and save to images/unicorn-inc.png (Apply as one line)
$ wget -O static/images/unicorn-inc.png https://d1.awsstatic.com/events/GameDay250.96690fe1668e09bf9a5012de5dd78b6e9293b25f.png
```

```bash
$ hugo
$ hugo server --port 8080
```

## コンテンツ作成
```bash
# In the new terminal, cd into the Unicorn Inc Hugo project folder
$ cd unicorn-inc-smart-factories/

# Create the site landing page
$ hugo new --kind chapter _index.md

# Create the Introduction chapter, chapter landing page and content pages
$ hugo new --kind chapter 10_introduction/_index.md
$ hugo new 10_introduction/10_about.md
$ hugo new 10_introduction/20_people.md

# Create the Smart Factories chapter, chapter landing page and content pages
$ hugo new --kind chapter 20_smart-factories/_index.md
$ hugo new 20_smart-factories/10_industry40.md

# Create the Deploying Smart Widgets chapter, chapter landing page and content pages
$ hugo new --kind chapter 30_smart-widgets/_index.md
$ hugo new 30_smart-widgets/10_smart-widget01.md
$ hugo new 30_smart-widgets/20_smart-widget02.md
$ hugo new 30_smart-widgets/30_smart-widget03.md
$ hugo new 30_smart-widgets/40_smart-widget04.md

# Create the Conclusion chapter, chapter and landing page
$ hugo new --kind chapter 40_conclusion/_index.md
```

```bash
# Create the **partials** folder in the **layouts** directory:
$ mkdir -p layouts/partials

# Create the **logo.html** partials file:
$ touch layouts/partials/logo.html
```

```bash
$ rm -Rf public
$ hugo 
```

## Amplifyお試し
```bash
# Make sure you are in the Unicorn Inc Hugo project folder created in the 'Develop Content' section.
$ cd /home/ec2-user/environment/unicorn-inc-smart-factories

# Initialize AWS Amplify
$ amplify init
Note: It is recommended to run this command from the root of your app directory
? Enter a name for the project unicorninc
? Enter a name for the environment main
? Choose your default editor: None
? Choose the type of app that you're building javascript
Please tell us about your project
? What javascript framework are you using none
? Source Directory Path:  content
? Distribution Directory Path: public
? Build Command:  hugo
? Start Command: hugo server --port 8080
Using default provider  awscloudformation
For more information on AWS Profiles, see:
https://docs.aws.amazon.com/cli/latest/userguide/cli-multiple-profiles.html

? Do you want to use an AWS profile? Yes
? Please choose the profile you want to use default
Adding backend environment main to AWS Amplify Console app: d1ulerjzp99i9
⠧ Initializing project in the cloud...

CREATE_IN_PROGRESS amplify-unicorninc-main-115454 AWS::CloudFormation::Stack Sun Mar 06 2022 11:54:58 GMT+0000 (Coordinated Universal Time) User Initiated

$ amplify status

Current Environment: main

| Category | Resource name | Operation | Provider plugin |
| -------- | ------------- | --------- | --------------- |

```

```bash
$ amplify add hosting
? Select the plugin module to execute Amazon CloudFront and S3
? Select the environment setup: PROD (S3 with CloudFront using HTTPS)
? hosting bucket name unicorninc-20220306115929-hostingbucket
Static webhosting is disabled for the hosting bucket when CloudFront Distribution is enabled.

You can now publish your app using the following command:
Command: amplify publish

$ amplify status

Current Environment: main

| Category | Resource name   | Operation | Provider plugin   |
| -------- | --------------- | --------- | ----------------- |
| Hosting  | S3AndCloudFront | Create    | awscloudformation |

```

```bash
$ amplify publish
✔ Successfully pulled backend environment main from the cloud.

Current Environment: main

| Category | Resource name   | Operation | Provider plugin   |
| -------- | --------------- | --------- | ----------------- |
| Hosting  | S3AndCloudFront | Create    | awscloudformation |
? Are you sure you want to continue? Yes

(snip)

✔ All resources are updated in the cloud

Hosting endpoint: https://<<ランダム文字列>>.cloudfront.net

                   | EN  
-------------------+-----
  Pages            | 19  
  Paginator pages  |  0  
  Non-page files   |  1  
  Static files     | 76  
  Processed images |  0  
  Aliases          |  0  
  Sitemaps         |  1  
  Cleaned          |  0  

Total in 183 ms
frontend build command exited with code 0
Publish started for S3AndCloudFront
✔ Uploaded files successfully.
Your app is published successfully.
https://<<ランダム>>.cloudfront.net

$ amplify status

Current Environment: main

| Category | Resource name   | Operation | Provider plugin   |
| -------- | --------------- | --------- | ----------------- |
| Hosting  | S3AndCloudFront | No Change | awscloudformation |

Hosting endpoint: https://<<ランダム文字列>>.cloudfront.net

```

US-EAST-1以外のリージョンだと、DNSが浸透するまで時間がかかる、その間はAccessDeniedとのこと。
```
Note: If you deploy this lab in any region other than US-EAST-1 it can take up to a few hours for DNS to replicate which will cause an AccessDenied error to be displayed. This is just for the first time the S3 bucket is created. Any future updates will not have a delay.
```