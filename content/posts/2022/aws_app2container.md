+++
Categories = ["AWS"]
Tags = ["AWS", "Container", "ECS", "EKS", "App2Container"]
date = "2022-03-06T00:00:00+09:00"
title = "AWS App2Container(お試し編)"
+++

# はじめに
TBD 導入文章書く。

# 参考
- [Accelerating your Migration to AWS](https://aws.amazon.com/jp/blogs/architecture/accelerating-your-migration-to-aws/)
- [AWS App2Container Uwer Guide](https://docs.aws.amazon.com/app2container/latest/UserGuide/what-is-a2c.html)
- [Modernize with AWS App2Container Workshop](https://catalog.us-east-1.prod.workshops.aws/workshops/2c1e5f50-0ebe-4c02-a957-8a71ba1e8c89/en-US/)

# Install & Setup (App2Container)
```bash
$ sudo su -
```

```bash
root@ip-10-0-1-112:~# pwd
/root
```

```bash
root@ip-10-0-1-112:~# ll
total 32
drwx------  6 root root 4096 Feb 17 11:09 ./
drwxr-xr-x 23 root root 4096 Mar  6 05:24 ../
-rw-r--r--  1 root root 3106 Apr  9  2018 .bashrc
drwx------  3 root root 4096 Feb 17 11:03 .cache/
drwx------  3 root root 4096 Feb 17 11:09 .config/
-rw-r--r--  1 root root  148 Aug 17  2015 .profile
drwx------  2 root root 4096 Feb 17 10:51 .ssh/
drwxr-xr-x  3 root root 4096 Feb 17 10:51 snap/
```

```bash
root@ip-10-0-1-112:~# df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            959M     0  959M   0% /dev
tmpfs           195M  780K  194M   1% /run
/dev/nvme0n1p1  291G  8.6G  283G   3% /
tmpfs           973M     0  973M   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           973M     0  973M   0% /sys/fs/cgroup
/dev/loop0       56M   56M     0 100% /snap/core18/2284
/dev/loop1       27M   27M     0 100% /snap/amazon-ssm-agent/5163
/dev/loop2       25M   25M     0 100% /snap/amazon-ssm-agent/4046
/dev/loop3      111M  111M     0 100% /snap/core/12725
/dev/loop4       44M   44M     0 100% /snap/snapd/14978
tmpfs           195M     0  195M   0% /run/user/1000
```

```bash
root@ip-10-0-1-112:~# curl -o AWSApp2Container-installer-linux.tar.gz https://app2container-release-us-east-1.s3.us-east-1.amazonaws.com/latest/linux/AWSApp2Container-installer-linux.tar.gz
(snip)
```

```bash
root@ip-10-0-1-112:~# tar xvf AWSApp2Container-installer-linux.tar.gz
install.sh
README
termsandconditions.txt
security/
security/app2container.cert
security/app2container.sig
AWSApp2Container.tar.gz
```

```bash
root@ip-10-0-1-112:~# ./install.sh 
Determining user ...
Determining Installer Path ...
Checking the current installed A2C version
The AWS App2Container tool is licensed as "AWS Content" under the terms and conditions of the AWS Customer Agreement, located at https://aws.amazon.com/agreement and the Service Terms, located at https://aws.amazon.com/service-terms. By installing, using or accessing the AWS App2Container tool, you agree to such terms and conditions. The term "AWS Content" does not include software and assets distributed under separate license terms (such as code licensed under an open source license).
Do you accept the terms and conditions above? (y/n): y
Installing AWS App2Container ...
AWSApp2Container/
AWSApp2Container/THIRD_PARTY_LICENSES

(snip)

Installation of AWS App2Container completed successfully!
You are currently running version 1.13.
To get started, run 'sudo app2container init'
AWS App2Container was installed under /usr/local/app2container/AWSApp2Container.
```


```bash
root@ip-10-0-1-112:~# app2container help
app2container is an application from Amazon Web Services (AWS),
that provides commands to discover and containerize applications.

Commands 
  Getting Started 🌱
    init                  Sets up workspace for artifacts.
 
 (snip)
```

```bash
root@ip-10-0-1-112:~# aws configure
AWS Access Key ID [None]: <<Your Access Key>>
AWS Secret Access Key [None]: <<Your Secret Key>>
Default region name [None]: ap-northeast-1
Default output format [None]: 
```

```bash
root@ip-10-0-1-112:~# app2container init
Workspace directory path for artifacts[default: /root/app2container]: 
AWS Profile (configured using 'aws configure --profile')[default: default]: 
Optional S3 bucket for application artifacts: t-tkm-app2container
Report usage metrics to AWS? (Y/N)[default: y]: n
Automatically upload logs and App2Container generated artifacts on crashes and internal errors? (Y/N)[default: y]: n
Require images to be signed using Docker Content Trust (DCT)? (Y/N)[default: n]: 
Configuration saved
All application artifacts will be created under /root/app2container. Please ensure that the folder permissions are secure.
```

## Install & Setup (Tomcat)
- https://tomcat.apache.org/download-90.cgi

```bash
VERSION=9.0.59
wget https://www-eu.apache.org/dist/tomcat/tomcat-9/v${VERSION}/bin/apache-tomcat-${VERSION}.tar.gz
(snip)
```

```bash
root@ip-10-0-1-112:~# sudo useradd -m -U -d /opt/tomcat -s /bin/false tomcat
```

```bash
root@ip-10-0-1-112:~# tar -xf ./apache-tomcat-${VERSION}.tar.gz -C /opt/tomcat/
```

```bash
root@ip-10-0-1-112:~# chown -R tomcat: /opt/tomcat
```

```bash
root@ip-10-0-1-112:~# ln -s /opt/tomcat/apache-tomcat-${VERSION} /opt/tomcat/latest
```

```bash
root@ip-10-0-1-112:/opt/tomcat# vi /etc/systemd/system/tomcat.service
```

**tomcat.service**
```vim
[Unit]
Description=Tomcat 9 servlet container
After=network.target

[Service]
Type=forking

User=tomcat
Group=tomcat

Environment="JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64"
Environment="JAVA_OPTS=-Djava.security.egd=file:///dev/urandom -Djava.awt.headless=true"

Environment="CATALINA_BASE=/opt/tomcat/latest"
Environment="CATALINA_HOME=/opt/tomcat/latest"
Environment="CATALINA_PID=/opt/tomcat/latest/temp/tomcat.pid"
Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"

ExecStart=/opt/tomcat/latest/bin/startup.sh
ExecStop=/opt/tomcat/latest/bin/shutdown.sh

[Install]
WantedBy=multi-user.target
```

```bash
root@ip-10-0-1-112:/opt/tomcat# systemctl daemon-reload
```

```bash
root@ip-10-0-1-112:/opt/tomcat# systemctl enable --now tomcat
Created symlink /etc/systemd/system/multi-user.target.wants/tomcat.service → /etc/systemd/system/tomcat.service.
```

サンプルwarをデプロイ。
https://tomcat.apache.org/tomcat-7.0-doc/appdev/sample/

```bash
root@ip-10-0-1-112:/opt/tomcat/latest/webapps# ll
total 28
drwxr-x---  7 tomcat tomcat 4096 Feb 21 21:01 ./
drwxr-xr-x  9 tomcat tomcat 4096 Mar  6 05:44 ../
drwxr-x---  3 tomcat tomcat 4096 Mar  6 05:44 ROOT/
drwxr-x--- 15 tomcat tomcat 4096 Mar  6 05:44 docs/
drwxr-x---  7 tomcat tomcat 4096 Mar  6 05:44 examples/
drwxr-x---  6 tomcat tomcat 4096 Mar  6 05:44 host-manager/
drwxr-x---  6 tomcat tomcat 4096 Mar  6 05:44 manager/
```

```bash
root@ip-10-0-1-112:/opt/tomcat/latest/webapps# wget https://tomcat.apache.org/tomcat-7.0-doc/appdev/sample/sample.war
```

```bash
root@ip-10-0-1-112:/opt/tomcat/latest/webapps# ll
total 40
drwxr-x---  8 tomcat tomcat 4096 Mar  6 05:57 ./
drwxr-xr-x  9 tomcat tomcat 4096 Mar  6 05:44 ../
drwxr-x---  3 tomcat tomcat 4096 Mar  6 05:44 ROOT/
drwxr-x--- 15 tomcat tomcat 4096 Mar  6 05:44 docs/
drwxr-x---  7 tomcat tomcat 4096 Mar  6 05:44 examples/
drwxr-x---  6 tomcat tomcat 4096 Mar  6 05:44 host-manager/
drwxr-x---  6 tomcat tomcat 4096 Mar  6 05:44 manager/
drwxr-x---  5 tomcat tomcat 4096 Mar  6 05:57 sample/
-rw-r--r--  1 root   root   4606 Mar 31  2012 sample.war
```
## Install & Setup (Spring Boot)
```bash
root@ip-10-0-1-112:~# curl -s "https://get.sdkman.io" | bash
```

```bash
root@ip-10-0-1-112:~# source "$HOME/.sdkman/bin/sdkman-init.sh"
```

```bash
root@ip-10-0-1-112:~# sdk list springboot
================================================================================
Available Springboot Versions
================================================================================
     2.6.4               2.3.4.RELEASE       2.0.8.RELEASE       1.4.1.RELEASE  
     2.6.3               2.3.3.RELEASE       2.0.7.RELEASE       1.4.0.RELEASE  
     2.6.2               2.3.2.RELEASE       2.0.6.RELEASE       1.3.8.RELEASE  
(snip)
     2.3.7.RELEASE       2.1.1.RELEASE       1.4.4.RELEASE       1.0.1.RELEASE  
     2.3.6.RELEASE       2.1.0.RELEASE       1.4.3.RELEASE       1.0.0.RELEASE  
     2.3.5.RELEASE       2.0.9.RELEASE       1.4.2.RELEASE                      

================================================================================
+ - local version
* - installed
> - currently in use
================================================================================
```

```bash
root@ip-10-0-1-112:~# sdk install springboot
```

かきに従う
- [2.1. CLI を使用したアプリケーションの実行](
https://spring.pleiades.io/spring-boot/docs/current/reference/html/cli.html#cli.using-the-cli.run)

```bash
root@ip-10-0-1-112:~# vi hello.groovy
```

```bash
root@ip-10-0-1-112:~# cat hello.groovy 
@RestController
class WebApplication {

    @RequestMapping("/")
    String home() {
        "Hello World!"
    }

}
```

```bash
spring run hello.groovy -- --server.port=8081

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.6.4)

2022-03-06 08:27:48.301  INFO 3803 --- [       runner-0] o.s.boot.SpringApplication               : Starting application using Java 11.0.13 on ip-10-0-1-112 with PID 3803 (started by root in /root)
2022-03-06 08:27:48.307  INFO 3803 --- [       runner-0] o.s.boot.SpringApplication               : No active profile set, falling back to 1 default profile: "default"
2022-03-06 08:27:50.482  INFO 3803 --- [       runner-0] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8081 (http)
202
(snip)
```

## App2Container
```bash
root@ip-10-0-1-112:~# app2container inventory
{
                "java-generic-6ef9339e": {
                                "processId": 6120,
                                "cmdline": "/usr/lib/jvm/java-11-openjdk-amd64/bin/java ... run hello.groovy -- --server.port=8081 ",
                                "applicationType": "java-generic",
                                "webApp": ""
                },
                "java-tomcat-5da060de": {
                                "processId": 4672,
                                "cmdline": "/usr/lib/jvm/java-11-openjdk-amd64/bin/java ... -Dcatalina.home=/opt/tomcat/latest -Djava.io.tmpdir=/opt/tomcat/latest/temp org.apache.catalina.startup.Bootstrap start ",
                                "applicationType": "java-tomcat",
                                "webApp": "sample"
                }
}
```

```bash
root@ip-10-0-1-112:~# ll app2container/
total 12
drwxr-xr-x  3 root root 4096 Mar  6 05:36 ./
drwx------ 12 root root 4096 Mar  6 06:07 ../
drwxr-xr-x  2 root root 4096 Mar  6 05:36 log/
root@ip-10-0-1-112:~# pwd
/root
```

```bash
root@ip-10-0-1-112:~# app2container analyze --application-id java-tomcat-5da060de
✔ Created artifacts folder /root/app2container/java-tomcat-5da060de
✔ Generated analysis data in /root/app2container/java-tomcat-5da060de/analysis.json
👍 Analysis successful for application java-tomcat-5da060de

💡 Next Steps:
1. View the application analysis file at /root/app2container/java-tomcat-5da060de/analysis.json.
2. Edit the application analysis file as needed.
3. Start the containerization process using this command: app2container containerize --application-id java-tomcat-5da060de
```

```bash
root@ip-10-0-1-112:~# app2container analyze --application-id java-generic-6ef9339e
✔ Created artifacts folder /root/app2container/java-generic-6ef9339e
✔ Generated analysis data in /root/app2container/java-generic-6ef9339e/analysis.json
👍 Analysis successful for application java-generic-6ef9339e

💡 Next Steps:
1. View the application analysis file at /root/app2container/java-generic-6ef9339e/analysis.json.
2. Edit the application analysis file as needed.
3. Start the containerization process using this command: app2container containerize --application-id java-generic-6ef9339e
```

```bash
root@ip-10-0-1-112:~# ll app2container/
total 20
drwxr-xr-x  5 root root 4096 Mar  6 06:14 ./
drwx------ 12 root root 4096 Mar  6 06:07 ../
drwxr-xr-x  2 root root 4096 Mar  6 06:14 java-generic-6ef9339e/
drwxr-xr-x  2 root root 4096 Mar  6 06:13 java-tomcat-5da060de/
drwxr-xr-x  2 root root 4096 Mar  6 05:36 log/
```

```bash
root@ip-10-0-1-112:~# apt install tree
root@ip-10-0-1-112:~# tree app2container/java-tomcat-5da060de/
app2container/java-tomcat-5da060de/
└── analysis.json
root@ip-10-0-1-112:~# tree app2container/java-generic-6ef9339e/
app2container/java-generic-6ef9339e/
└── analysis.json
```

```bash
root@ip-10-0-1-112:~# docker image ls
REPOSITORY      TAG          IMAGE ID       CREATED         SIZE
lambci/lambda   python3.8    094248252696   13 months ago   524MB
lambci/lambda   nodejs12.x   22a4ada8399c   13 months ago   390MB
lambci/lambda   nodejs10.x   db93be728e7b   13 months ago   385MB
lambci/lambda   python3.7    22b4b6fd9260   13 months ago   946MB
lambci/lambda   python3.6    177c85a10179   13 months ago   894MB
lambci/lambda   python2.7    d96a01fe4c80   13 months ago   763MB
lambci/lambda   nodejs8.10   5754fee26e6e   13 months ago   813MB
```

```bash
root@ip-10-0-1-112:~# app2container analyze --application-id java-generic-6ef9339e
✔ Created artifacts folder /root/app2container/java-generic-6ef9339e
root@ip-10-0-1-112:~# app2container containerize --application-id java-tomcat-5da060de
✔ AWS prerequisite check succeeded
✔ Docker prerequisite check succeeded
✔ Extracted container artifacts for application
✔ Entry file generated
✔ Dockerfile generated under /root/app2container/java-tomcat-5da060de/Artifacts
✔ Generated dockerfile.update under /root/app2container/java-tomcat-5da060de/Artifacts
✔ Generated deployment file at /root/app2container/java-tomcat-5da060de/deployment.json
✔ Deployment artifacts generated.
✔ Pre-validation succeeded.
👍 Containerization successful. Generated docker image java-tomcat-5da060de

💡 You're all set to test and deploy your container image.

Next Steps:
1. View the container image with "docker images" and test the application with "docker run --name java-tomcat-5da060de -Pit java-tomcat-5da060de".
2. When you're ready to deploy to AWS, adjust the appropriate fields in /root/app2container/java-tomcat-5da060de/deployment.json to generate the desired deployment artifact. Note that by default "createEcsArtifacts" is set to true.
3. Generate deployment artifacts using "app2container generate app-deployment --application-id java-tomcat-5da060de".
```

```bash
root@ip-10-0-1-112:~# tree app2container/java-tomcat-5da060de/
app2container/java-tomcat-5da060de/
├── Artifacts
│   ├── ContainerFiles.tar
│   ├── Dockerfile
│   ├── Dockerfile.update
│   ├── entryfile
│   ├── excludedFiles
│   └── includeFiles
├── analysis.json
└── deployment.json

1 directory, 8 files
```

```bash
root@ip-10-0-1-112:~# docker image ls
REPOSITORY             TAG          IMAGE ID       CREATED              SIZE
java-tomcat-5da060de   latest       cc2de1db6ef8   About a minute ago   802MB
ubuntu                 18.04        eb2556e0f6e4   2 days ago           63.2MB
(snip)
```

```bash
root@ip-10-0-1-112:~# docker run -d --rm -p 8082:8080 java-tomcat-5da060de
55dc392f0ee619afb61298b6a2ee05758b495a10c7f6b6bc2eb082d34e2dcd79

root@ip-10-0-1-112:~# docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED         STATUS                            PORTS                                                 NAMES
55dc392f0ee6   java-tomcat-5da060de   "/bin/sh -c /entryfi…"   3 seconds ago   Up 2 seconds (health: starting)   8005/tcp, 0.0.0.0:8082->8080/tcp, :::8082->8080/tcp   charming_varahamihira
                   
root@ip-10-0-1-112:~# docker stop 55d
55d
```

```bash
root@ip-10-0-1-112:~# app2container containerize --application-id java-generic-6ef9339e
✔ AWS prerequisite check succeeded
✔ Docker prerequisite check succeeded
✔ Extracted container artifacts for application
✔ Entry file generated
✔ Dockerfile generated under /root/app2container/java-generic-6ef9339e/Artifacts
✔ Generated dockerfile.update under /root/app2container/java-generic-6ef9339e/Artifacts
✔ Generated deployment file at /root/app2container/java-generic-6ef9339e/deployment.json
✔ Deployment artifacts generated.
✔ Pre-validation succeeded.
👍 Containerization successful. Generated docker image java-generic-6ef9339e

💡 You're all set to test and deploy your container image.

Next Steps:
1. View the container image with "docker images" and test the application with "docker run --name java-generic-6ef9339e -Pit java-generic-6ef9339e".
2. When you're ready to deploy to AWS, adjust the appropriate fields in /root/app2container/java-generic-6ef9339e/deployment.json to generate the desired deployment artifact. Note that by default "createEcsArtifacts" is set to true.
3. Generate deployment artifacts using "app2container generate app-deployment --application-id java-generic-6ef9339e".
```

```bash
root@ip-10-0-1-112:~# docker image ls
REPOSITORY              TAG          IMAGE ID       CREATED          SIZE
java-generic-6ef9339e   latest       8ef7ef3e7db0   10 minutes ago   16.1GB
java-tomcat-5da060de    latest       cc2de1db6ef8   30 minutes ago   802MB
ubuntu                  18.04        eb2556e0f6e4   2 days ago       63.2MB
(snip)
```

```bash
root@ip-10-0-1-112:~# docker run -d --rm -p 8082:8081 java-generic-6ef9339e                                                                                                                               
7fa1c72fbc37cf4352264622b8c8dae485fdac8fd8041a6aef6b5ec21a1a7073

root@ip-10-0-1-112:~# docker ps
CONTAINER ID   IMAGE                   COMMAND                  CREATED         STATUS                            PORTS                                       NAMES
7fa1c72fbc37   java-generic-6ef9339e   "/bin/sh -c /entryfi…"   7 seconds ago   Up 6 seconds (health: starting)   0.0.0.0:8082->8081/tcp, :::8082->8081/tcp   goofy_tereshkova

root@ip-10-0-1-112:~# docker stop 7fa
7fa
```