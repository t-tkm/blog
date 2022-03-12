+++
Categories = ["AWS"]
Tags = ["AWS", "Container", "ECS", "EKS", "App2Container"]
date = "2022-03-06T00:00:00+09:00"
title = "AWS App2Container(イメージ最適化編)"
+++

# はじめに
TBD 導入文章書く。

# 参考
- [Optimize AWS App2Container generated Docker images](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/optimize-aws-app2container-generated-docker-images.html)

# 準備
```bash
root@ip-10-0-1-112:~# docker image ls
REPOSITORY              TAG          IMAGE ID       CREATED          SIZE
java-generic-6ef9339e   latest       8ef7ef3e7db0   38 minutes ago   16.1GB
java-tomcat-5da060de    latest       cc2de1db6ef8   58 minutes ago   802MB

(snip)

```

```bash
root@ip-10-0-1-112:~# wget https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/samples/p-attach/dc756bff-1fcd-4fd2-8c4f-dc494b5007b9/attachments/attachment.zip
```

```bash
root@ip-10-0-1-112:~# unzip attachment.zip 
Archive:  attachment.zip
  inflating: sample_analysis.json    
  inflating: optimizeImage.sh        
```

```bash
root@ip-10-0-1-112:~# chmod +x optimizeImage.sh 
```

```bash
root@ip-10-0-1-112:~# ll
total 824540
(snip)
-rw-r--r--  1 root    root       3981 Jul 29  2021 sample_analysis.json
-rwxr-xr-x  1 root    root       2366 Jul 29  2021 optimizeImage.sh*
(snip)
```

## Tomcat
```bash
root@ip-10-0-1-112:~# ll -h app2container/java-tomcat-5da060de/Artifacts/
total 152M
(snip)
-r-- 1 root root 152M Mar  6 06:17 ContainerFiles.tar
(snip)
```

```bash
TAR_TARGET=./app2container/java-tomcat-5da060de/Artifacts/ContainerFiles.tar
```

```bash
root@ip-10-0-1-112:~# ./optimizeImage.sh -p $TAR_TARGET -d 1 -t / -s 1000000 -v
./app2container/java-tomcat-5da060de/Artifacts/ContainerFiles.tar exists.
tar -Ptvf ./app2container/java-tomcat-5da060de/Artifacts/ContainerFiles.tar|tr -s ' '|cut -d ' ' -f3,6| awk '$2 ~/$/'| awk '$2 ~/^\//' |cut -f1-2 -d'/'|awk '{ if ($1>= 1000000) arr[$2]+=$1 } END { for (key in arr) { if(1) printf("%-50s\t%-50s\n", key, arr[key]) else printf("%s,\n", key) } } '|sort -k2 -nr
===============================================================================================
/usr                                                    142264753                                         
/opt                                                    4856216                                           
```

```bash
root@ip-10-0-1-112:~# ./optimizeImage.sh -p $TAR_TARGET -d 2 -t /usr -s 1000000 -v
./app2container/java-tomcat-5da060de/Artifacts/ContainerFiles.tar exists.
tar -Ptvf ./app2container/java-tomcat-5da060de/Artifacts/ContainerFiles.tar|tr -s ' '|cut -d ' ' -f3,6| awk '$2 ~/$/'| awk '$2 ~/^\/usr/' |cut -f1-3 -d'/'|awk '{ if ($1>= 1000000) arr[$2]+=$1 } END { for (key in arr) { if(1) printf("%-50s\t%-50s\n", key, arr[key]) else printf("%s,\n", key) } } '|sort -k2 -nr
===============================================================================================
/usr/lib                                                142264753                                         
```

## Spring
```bash
root@ip-10-0-1-112:~# ll -h app2container/java-generic-6ef9339e/Artifacts/
(snip)
-rw-r--r-- 1 root root 7.7G Mar  6 06:27 ContainerFiles.tar
(snip)
```

```bash
root@ip-10-0-1-112:~# TAR_TARGET=./app2container/java-generic-6ef9339e/Artifacts/ContainerFiles.tar
```

```bash
root@ip-10-0-1-112:~# ./optimizeImage.sh -p $TAR_TARGET -d 0 -t / -v
./app2container/java-generic-6ef9339e/Artifacts/ContainerFiles.tar exists.
tar -Ptvf ./app2container/java-generic-6ef9339e/Artifacts/ContainerFiles.tar|tr -s ' '|cut -d ' ' -f3,6| awk '$2 ~/$/'| awk '$2 ~/^\//' |cut -f1-1 -d'/'|awk '{ if ($1>= 0) arr[$2]+=$1 } END { for (key in arr) { if(1) printf("%-50s\t%-50s\n", key, arr[key]) else printf("%s,\n", key) } } '|sort -k2 -nr
===============================================================================================
                                                        7919225008  
```

```bash
root@ip-10-0-1-112:~# ./optimizeImage.sh -p $TAR_TARGET -d 1 -t / -s 1000000 -v
./app2container/java-generic-6ef9339e/Artifacts/ContainerFiles.tar exists.
tar -Ptvf ./app2container/java-generic-6ef9339e/Artifacts/ContainerFiles.tar|tr -s ' '|cut -d ' ' -f3,6| awk '$2 ~/$/'| awk '$2 ~/^\//' |cut -f1-2 -d'/'|awk '{ if ($1>= 1000000) arr[$2]+=$1 } END { for (key in arr) { if(1) printf("%-50s\t%-50s\n", key, arr[key]) else printf("%s,\n", key) } } '|sort -k2 -nr
===============================================================================================
/usr                                                    2412836670                                        
/var                                                    1319669369                                        
/root                                                   953114743                                         
/snap                                                   658552014                                         
/opt                                                    305844542                                         
/home                                                   283062895                                         
/lib                                                    32995202                                          
/sbin                                                   5033368                                           
/bin                                                    3175800  
```

```bash
root@ip-10-0-1-112:~# ./optimizeImage.sh -p $TAR_TARGET -d 2 -t /usr -s 1000000 -v
./app2container/java-generic-6ef9339e/Artifacts/ContainerFiles.tar exists.
tar -Ptvf ./app2container/java-generic-6ef9339e/Artifacts/ContainerFiles.tar|tr -s ' '|cut -d ' ' -f3,6| awk '$2 ~/$/'| awk '$2 ~/^\/usr/' |cut -f1-3 -d'/'|awk '{ if ($1>= 1000000) arr[$2]+=$1 } END { for (key in arr) { if(1) printf("%-50s\t%-50s\n", key, arr[key]) else printf("%s,\n", key) } } '|sort -k2 -nr
===============================================================================================
/usr/lib                                                1048688240                                        
/usr/local                                              605362445                                         
/usr/bin                                                576847128                                         
/usr/libexec                                            114789284                                         
/usr/share                                              36225952                                          
/usr/sbin                                               27243896                                          
/usr/src                                                3679725
```