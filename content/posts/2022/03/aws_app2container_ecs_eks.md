+++
Categories = ["AWS"]
Tags = ["AWS", "Container", "ECS", "EKS", "App2Container"]
date = "2022-03-16T00:00:00+09:00"
title = "AWS App2Container(ECS/EKSデプロイ編)"
archives = ["2022", "2022-03", "2022-03-16"]

+++

# 1. はじめに
ここでは、前回の記事「AWS App2Container(お試し編)」では触れられなかったデプロイについて試してみます。
実際は、次の3ステップでデプロイできて衝撃的でした！

1. deployment.jsonを編集し、「ECS」か「EKS」を選択(AppRunnerも選択可)。
2. app2containerコマンド(オプションgenerate app-deployment)でマニフェスト作成(一部S3に格納される)
3. CloudFormationでデプロイ

そして、ステップ2の「app2containerコマンド」の引数オプションとして「--deploy」を指定すると、ステップ3すら不要です^^

{{< figure alt="img8" src="https://github.com/t-tkm/blog_images/blob/aws_app2container_ecs_eks/2022/aws_app2container_ecs_eks/img8.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/aws_app2container_ecs_eks/2022/aws_app2container_ecs_eks/img8.png?raw=true">}}

それでは試してみましょう。

# 2. マニフェスト作成

{{< figure alt="img1" src="https://github.com/t-tkm/blog_images/blob/aws_app2container_ecs_eks/2022/aws_app2container_ecs_eks/img1.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/aws_app2container_ecs_eks/2022/aws_app2container_ecs_eks/img1.png?raw=true">}}

## 2-1. S3へ格納される成果物(ECS)
```bash
$ tree a2c-java-tomcat-3c173836/ecs 
a2c-java-tomcat-3c173836/ecs
└── subtemplates
    ├── ecs-cluster.yml
    ├── ecs-gmsa-automation-doc.yml
    ├── ecs-gmsa-execute.yml
    ├── ecs-gmsa-iam-roles.yml
    ├── ecs-gmsa-lambda-functions.yml
    ├── ecs-gmsa.yml
    ├── ecs-lb-webapp.yml
    ├── ecs-master.yml
    ├── ecs-private-app.yml
    ├── ecs-private-load-balancer.yml
    ├── ecs-public-load-balancer.yml
    ├── ecs-route53.yml
    └── ecs-vpc.yml
```

## 2-2. S3へ格納される成果物(EKS)
<details>
<summary>(snip)展開</summary>

```bash
$ tree a2c-java-tomcat-3c173836/eks 
a2c-java-tomcat-3c173836/eks
├── functions
│   └── packages
│       ├── CleanupLambdas
│       │   └── lambda.zip
│       ├── CleanupLoadBalancers
│       │   └── lambda.zip
│       ├── CleanupSecurityGroupDependencies
│       │   └── lambda.zip
│       ├── DeleteBucketContents
│       │   └── lambda.zip
│       ├── EksClusterResource
│       │   └── awsqs-eks-cluster.zip
│       ├── FargateProfile
│       │   └── lambda.zip
│       ├── GetCallerArn
│       │   └── lambda.zip
│       ├── HelmReleaseResource
│       │   └── awsqs-kubernetes-helm.zip
│       ├── KubeGet
│       │   └── lambda.zip
│       ├── KubeManifest
│       │   └── lambda.zip
│       ├── ResourceReader
│       │   └── lambda.zip
│       ├── awscliLayer
│       │   └── lambda.zip
│       ├── boto3Layer
│       │   └── lambda.zip
│       ├── crhelperLayer
│       │   └── lambda.zip
│       ├── kubectlLayer
│       │   └── lambda.zip
│       ├── kubernetesResources
│       │   ├── awsqs_kubernetes_apply.zip
│       │   ├── awsqs_kubernetes_apply_vpc.zip
│       │   ├── awsqs_kubernetes_get.zip
│       │   └── awsqs_kubernetes_get_vpc.zip
│       ├── registerCustomResource
│       │   └── lambda.zip
│       └── registerType
│           └── lambda.zip
├── scripts
│   ├── CustomResourceDefinition.yml
│   ├── Deploy-CredSpec.ps1
│   ├── Patch-Coredns.ps1
│   ├── Setup-Webhook.ps1
│   ├── create-signed-cert.ps1
│   ├── credspec-cluster-role-template.yaml
│   ├── gmsa-credspec-template.json
│   ├── gmsa-webhook-template.yaml
│   ├── patch-coredns-template.yaml
│   ├── patch-webhook-template.ps1
│   ├── windows-gmsa-setup.sh
│   └── windows-setup.sh
├── submodules
│   ├── quickstart-amazon-eks-nodegroup
│   │   ├── LICENSE.TXT
│   │   └── templates
│   │       └── amazon-eks-nodegroup.template.yaml
│   └── quickstart-aws-vpc
│       ├── ci
│       │   ├── aws-vpc-3az-complete.json
│       │   ├── aws-vpc-3az-public.json
│       │   ├── aws-vpc-3az.json
│       │   ├── aws-vpc-4az-complete.json
│       │   ├── aws-vpc-4az-public.json
│       │   ├── aws-vpc-4az.json
│       │   ├── aws-vpc-complete.json
│       │   ├── aws-vpc-dedicated.json
│       │   ├── aws-vpc-defaults.json
│       │   ├── aws-vpc-public.json
│       │   ├── aws-vpc-sa-east-1.json
│       │   └── taskcat.yml
│       ├── docs
│       │   └── utils
│       │       ├── generate_parms.py
│       │       └── requirements.txt
│       └── templates
│           ├── aws-vpc.template
│           └── aws-vpc.template.yaml
└── templates
    ├── amazon-eks-advanced-configuration.template.yaml
    ├── amazon-eks-alb-ingress.template.yaml
    ├── amazon-eks-application-workload.yaml
    ├── amazon-eks-cluster-autoscaler.template.yaml
    ├── amazon-eks-controlplane.template.yaml
    ├── amazon-eks-efs.template.yaml
    ├── amazon-eks-fargate-profile.template.yaml
    ├── amazon-eks-functions.template.yaml
    ├── amazon-eks-gmsa-coredns.yaml
    ├── amazon-eks-gmsa-crd.yaml
    ├── amazon-eks-gmsa-credspec.yaml
    ├── amazon-eks-gmsa-deployments.yaml
    ├── amazon-eks-gmsa-webhook.yaml
    ├── amazon-eks-grafana.template.yaml
    ├── amazon-eks-iam.template.yaml
    ├── amazon-eks-per-account-resources.template.yaml
    ├── amazon-eks-per-region-resources.template.yaml
    ├── amazon-eks-prometheus.template.yaml
    ├── amazon-eks-test.template.yaml
    ├── amazon-eks-windows-support-workload.template.yaml
    ├── amazon-eks.yaml
    ├── ecs-gmsa-automation-doc.yml
    ├── ecs-gmsa-execute.yml
    ├── ecs-gmsa-iam-roles.yml
    ├── ecs-gmsa-lambda-functions.yml
    ├── ecs-gmsa.yml
    ├── ecs-route53.yml
    └── eks-quickstart-region-upgrade.yaml
```

</details>

```bash
$ tree a2c-java-tomcat-3c173836/eks 
a2c-java-tomcat-3c173836/eks
(snip)
└── templates
    ├── amazon-eks-advanced-configuration.template.yaml
    ├── amazon-eks-alb-ingress.template.yaml
    ├── amazon-eks-application-workload.yaml
    ├── amazon-eks-cluster-autoscaler.template.yaml
    ├── amazon-eks-controlplane.template.yaml
    ├── amazon-eks-efs.template.yaml
    ├── amazon-eks-fargate-profile.template.yaml
    ├── amazon-eks-functions.template.yaml
    ├── amazon-eks-gmsa-coredns.yaml
    ├── amazon-eks-gmsa-crd.yaml
    ├── amazon-eks-gmsa-credspec.yaml
    ├── amazon-eks-gmsa-deployments.yaml
    ├── amazon-eks-gmsa-webhook.yaml
    ├── amazon-eks-grafana.template.yaml
    ├── amazon-eks-iam.template.yaml
    ├── amazon-eks-per-account-resources.template.yaml
    ├── amazon-eks-per-region-resources.template.yaml
    ├── amazon-eks-prometheus.template.yaml
    ├── amazon-eks-test.template.yaml
    ├── amazon-eks-windows-support-workload.template.yaml
    ├── amazon-eks.yaml
    ├── ecs-gmsa-automation-doc.yml
    ├── ecs-gmsa-execute.yml
    ├── ecs-gmsa-iam-roles.yml
    ├── ecs-gmsa-lambda-functions.yml
    ├── ecs-gmsa.yml
    ├── ecs-route53.yml
    └── eks-quickstart-region-upgrade.yaml
```

# 3. ECSデプロイ
## 3-1. Cloud Formationでデプロイ実行
```bash
aws cloudformation deploy \
	--template-file /root/app2container/java-tomcat-3c173836/EcsDeployment/ecs-master.yml \
	--capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND \
	--stack-name a2c-java-tomcat-3c173836-ECS
```

## 3-2. Cloud Formationスタック
{{< figure alt="img2" src="https://github.com/t-tkm/blog_images/blob/aws_app2container_ecs_eks/2022/aws_app2container_ecs_eks/img2.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/aws_app2container_ecs_eks/2022/aws_app2container_ecs_eks/img2.png?raw=true">}}

## 3-3. 確認
{{< figure alt="img3" src="https://github.com/t-tkm/blog_images/blob/aws_app2container_ecs_eks/2022/aws_app2container_ecs_eks/img3.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/aws_app2container_ecs_eks/2022/aws_app2container_ecs_eks/img3.png?raw=true">}}

{{< figure alt="img4" src="https://github.com/t-tkm/blog_images/blob/aws_app2container_ecs_eks/2022/aws_app2container_ecs_eks/img4.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/aws_app2container_ecs_eks/2022/aws_app2container_ecs_eks/img4.png?raw=true">}}

# 4. EKSデプロイ
## 4-1. Cloud Formationでデプロイ実行
```bash
aws cloudformation deploy \
	--template-file /root/app2container/java-tomcat-3c173836/EksDeployment/amazon-eks-entrypoint-new-vpc.yaml \
	--capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND \
	--stack-name a2c-java-tomcat-3c173836-EKS
```

## 4-2. Cloud Formationスタック
{{< figure alt="img5" src="https://github.com/t-tkm/blog_images/blob/aws_app2container_ecs_eks/2022/aws_app2container_ecs_eks/img5.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/aws_app2container_ecs_eks/2022/aws_app2container_ecs_eks/img5.png?raw=true">}}

## 4-3. 確認
{{< figure alt="img6" src="https://github.com/t-tkm/blog_images/blob/aws_app2container_ecs_eks/2022/aws_app2container_ecs_eks/img6.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/aws_app2container_ecs_eks/2022/aws_app2container_ecs_eks/img6.png?raw=true">}}

{{< figure alt="img7" src="https://github.com/t-tkm/blog_images/blob/aws_app2container_ecs_eks/2022/aws_app2container_ecs_eks/img7.png?raw=true" link="https://github.com/t-tkm/blog_images/blob/aws_app2container_ecs_eks/2022/aws_app2container_ecs_eks/img7.png?raw=true">}}



