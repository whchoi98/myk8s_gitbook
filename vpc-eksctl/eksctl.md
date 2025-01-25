---
description: 'update : 2025-01-25 / 20min'
---

# eksctl 구성

## eksctl 소개

eksctl은 Amazon EKS (Elastic Kubernetes Service) 클러스터를 쉽게 생성, 관리 및 삭제할 수 있도록 설계된 CLI 도구입니다. eksctl은 Kubernetes 클러스터를 AWS에서 빠르고 간단하게 설정할 수 있는 방법을 제공하며, 클러스터 설정과 관리의 복잡성을 줄이는 데 중점을 둡니다.

이 도구는 Weaveworks와 AWS가 협력하여 개발했으며, Kubernetes 클러스터의 자동화된 네트워크 설정, 노드 그룹 생성, IAM 역할 구성 등의 작업을 간단한 명령으로 수행할 수 있습니다.

## eksctl을 통한 EKS 구성

### 1.eksctl 설치

사전 환경 구성에서 이미 설치되어 있습니다. 이 단계는 생략이 가능합니다.

아래와 같이 eksctl을 IDE 터미널에 설치하고 버전을 확인합니다.

eksctl 버전이 낮은 경우에는 EKS 최신버전을 설치할 경우 , 원할하게 설치 되지 않을 수 있습니다.

([https://github.com/eksctl-io/eksctl/releases](https://github.com/eksctl-io/eksctl/releases))

```
# eksctl 설정 
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
# eksctl 자동완성 - bash
. <(eksctl completion bash)
eksctl version

```

### 2.VPC/Subnet 정보 확인

앞서 [Cloudformation 구성](cloudformation.md#3-stack)에서 생성한 VPC 자원들에 대한 고유의 자원 값을 추출해서, 터미널 내에서 환경 변수에 저장합니다

```
~/environment/myeks/shell/eks_shell.sh

```

```
#아래와 같이 EKS 버전 입력을 요구하면, EKS Version 을 입력합니다.
#Enter the EKS version (e.g., 1.29): 1.29
```

{% hint style="info" %}
cat \~/.bash\_profile 을 실행해서 환경 변수가 정상적으로 입력되었는 지 확인해 봅니다.
{% endhint %}

### 3. eksctl 배포를 위한 yaml 생성&#x20;

eksctl yaml 생성을 위해 아래 Shell을 실행합니다.&#x20;

```
# eksctl yaml 실행 
~/environment/myeks/shell/eksctl_shell.sh
 
```

생성된 eksctl yaml 파일을 dry-run을 실행시켜서 확인해 봅니다.

```
eksctl create cluster --config-file=/home/ec2-user/environment/myeks/eksworkshop.yaml --dry-run

```

아래와 같이 명령을 실행시켜 eks cluster를 생성합니다.&#x20;

```
# eksctl로 cluster 만들기 
eksctl create cluster --config-file=/home/ec2-user/environment/myeks/eksworkshop.yaml
 
```

{% hint style="info" %}
Cluster를 생성하기 위해 20분 정도 시간이 소요됩니다.&#x20;
{% endhint %}

출력 결과 예시

<pre><code>025-01-25 07:43:25 [ℹ]  eksctl version 0.202.0
2025-01-25 07:43:25 [ℹ]  using region ap-northeast-2
2025-01-25 07:43:25 [✔]  using existing VPC (vpc-0a8f9ba32a7ccd63e) and subnets (private:map[PrivateSubnet01:{subnet-01f755ec59aea4fbd ap-northeast-2a 10.11.48.0/20 0 } PrivateSubnet02:{subnet-0b6fe2f1a97454cab ap-northeast-2b 10.11.64.0/20 0 } PrivateSubnet03:{subnet-0d59308be5c533ed8 ap-northeast-2c 10.11.80.0/20 0 }] public:map[PublicSubnet01:{subnet-08e88afe890bb0dc0 ap-northeast-2a 10.11.0.0/20 0 } PublicSubnet02:{subnet-0ae04fc52d12ae17b ap-northeast-2b 10.11.16.0/20 0 } PublicSubnet03:{subnet-09361b2357369c56f ap-northeast-2c 10.11.32.0/20 0 }])
2025-01-25 07:43:25 [!]  custom VPC/subnets will be used; if resulting cluster doesn't function as expected, make sure to review the configuration of VPC/subnets
2025-01-25 07:43:25 [ℹ]  nodegroup "ng-public-01" will use "ami-00da1360b43239c87" [AmazonLinux2/1.29]
2025-01-25 07:43:26 [ℹ]  nodegroup "ng-private-01" will use "ami-00da1360b43239c87" [AmazonLinux2/1.29]
2025-01-25 07:43:26 [ℹ]  nodegroup "managed-ng-public-01" will use "" [AmazonLinux2/1.29]
2025-01-25 07:43:26 [ℹ]  nodegroup "managed-ng-private-01" will use "" [AmazonLinux2/1.29]
2025-01-25 07:43:26 [ℹ]  using Kubernetes version 1.29
2025-01-25 07:43:26 [ℹ]  creating EKS cluster "eksworkshop" in "ap-northeast-2" region with managed nodes and un-managed nodes
2025-01-25 07:43:26 [ℹ]  4 nodegroups (managed-ng-private-01, managed-ng-public-01, ng-private-01, ng-public-01) were included (based on the include/exclude rules)
2025-01-25 07:43:26 [ℹ]  will create a CloudFormation stack for cluster itself and 2 nodegroup stack(s)
2025-01-25 07:43:26 [ℹ]  will create a CloudFormation stack for cluster itself and 2 managed nodegroup stack(s)
2025-01-25 07:43:26 [ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=ap-northeast-2 --cluster=eksworkshop'
2025-01-25 07:43:26 [ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "eksworkshop" in "ap-northeast-2"
2025-01-25 07:43:26 [ℹ]  configuring CloudWatch logging for cluster "eksworkshop" in "ap-northeast-2" (enabled types: api, audit, authenticator, controllerManager, scheduler &#x26; no types disabled)
2025-01-25 07:43:26 [ℹ]  default addons metrics-server were not specified, will install them as EKS addons
2025-01-25 07:43:26 [ℹ]  
2 sequential tasks: { create cluster control plane "eksworkshop", 
    2 sequential sub-tasks: { 
        5 sequential sub-tasks: { 
            1 task: { create addons },
            wait for control plane to become ready,
            associate IAM OIDC provider,
            no tasks,
            update VPC CNI to use IRSA if required,
        },
        2 parallel sub-tasks: { 
            2 parallel sub-tasks: { 
                create nodegroup "ng-public-01",
                create nodegroup "ng-private-01",
            },
            2 parallel sub-tasks: { 
                create managed nodegroup "managed-ng-public-01",
                create managed nodegroup "managed-ng-private-01",
            },
        },
    } 
}
2025-01-25 07:43:26 [ℹ]  building cluster stack "eksctl-eksworkshop-cluster"
2025-01-25 07:43:26 [ℹ]  deploying stack "eksctl-eksworkshop-cluster"
2025-01-25 07:43:56 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-cluster"
2025-01-25 07:50:26 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-cluster"
2025-01-25 07:50:28 [!]  IRSA config is set for "vpc-cni" addon, but since OIDC is disabled on the cluster, eksctl cannot configure the requested permissions; the recommended way to provide IAM permissions for "vpc-cni" addon is via pod identity associations; after addon creation is completed, add all recommended policies to the config file, under `addon.PodIdentityAssociations`, and run `eksctl update addon`
2025-01-25 07:50:28 [ℹ]  creating addon
2025-01-25 07:50:28 [ℹ]  successfully created addon
2025-01-25 07:50:29 [ℹ]  creating addon
2025-01-25 07:50:29 [ℹ]  successfully created addon
2025-01-25 07:50:29 [ℹ]  creating addon
2025-01-25 07:50:30 [ℹ]  successfully created addon
2025-01-25 07:50:30 [ℹ]  creating addon
2025-01-25 07:50:30 [ℹ]  successfully created addon
2025-01-25 07:52:32 [ℹ]  addon "vpc-cni" active
2025-01-25 07:52:32 [ℹ]  deploying stack "eksctl-eksworkshop-addon-vpc-cni"
2025-01-25 07:52:32 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-addon-vpc-cni"
2025-01-25 07:53:02 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-addon-vpc-cni"
2025-01-25 07:53:02 [ℹ]  updating addon
2025-01-25 07:53:13 [ℹ]  addon "vpc-cni" active
2025-01-25 07:53:13 [ℹ]  building nodegroup stack "eksctl-eksworkshop-nodegroup-ng-private-01"
2025-01-25 07:53:13 [ℹ]  building nodegroup stack "eksctl-eksworkshop-nodegroup-ng-public-01"
2025-01-25 07:53:13 [!]  subnets contain a mix of both local and availability zones
2025-01-25 07:53:13 [ℹ]  building managed nodegroup stack "eksctl-eksworkshop-nodegroup-managed-ng-private-01"
2025-01-25 07:53:13 [!]  subnets contain a mix of both local and availability zones
2025-01-25 07:53:13 [ℹ]  building managed nodegroup stack "eksctl-eksworkshop-nodegroup-managed-ng-public-01"
2025-01-25 07:53:13 [ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-managed-ng-private-01"
2025-01-25 07:53:13 [ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-ng-private-01"
2025-01-25 07:53:13 [ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-ng-public-01"
2025-01-25 07:53:13 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-managed-ng-private-01"
2025-01-25 07:53:13 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-private-01"
2025-01-25 07:53:13 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-public-01"
2025-01-25 07:53:13 [ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-managed-ng-public-01"
2025-01-25 07:53:13 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-managed-ng-public-01"
<strong>2025-01-25 07:53:43 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-managed-ng-private-01"
</strong>2025-01-25 07:53:43 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-private-01"
2025-01-25 07:53:43 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-public-01"
2025-01-25 07:53:43 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-managed-ng-public-01"
2025-01-25 07:54:23 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-private-01"
2025-01-25 07:54:26 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-managed-ng-private-01"
2025-01-25 07:54:32 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-managed-ng-public-01"
2025-01-25 07:54:39 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-public-01"
2025-01-25 07:55:19 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-managed-ng-private-01"
2025-01-25 07:55:35 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-managed-ng-public-01"
2025-01-25 07:55:47 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-public-01"
2025-01-25 07:55:58 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-managed-ng-private-01"
2025-01-25 07:56:10 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-private-01"
2025-01-25 07:56:51 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-public-01"
2025-01-25 07:56:51 [ℹ]  waiting for the control plane to become ready
2025-01-25 07:56:52 [✔]  saved kubeconfig as "/home/ec2-user/.kube/config"
2025-01-25 07:56:52 [ℹ]  no tasks
2025-01-25 07:56:52 [✔]  all EKS cluster resources for "eksworkshop" have been created
2025-01-25 07:56:52 [ℹ]  nodegroup "ng-public-01" has 3 node(s)
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-13-43.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-30-94.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-39-158.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [ℹ]  waiting for at least 3 node(s) to become ready in "ng-public-01"
2025-01-25 07:56:52 [ℹ]  nodegroup "ng-public-01" has 3 node(s)
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-13-43.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-30-94.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-39-158.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [ℹ]  nodegroup "ng-private-01" has 3 node(s)
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-52-252.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-67-238.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-94-141.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [ℹ]  waiting for at least 3 node(s) to become ready in "ng-private-01"
2025-01-25 07:56:52 [ℹ]  nodegroup "ng-private-01" has 3 node(s)
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-52-252.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-67-238.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-94-141.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [✔]  created 2 nodegroup(s) in cluster "eksworkshop"
2025-01-25 07:56:52 [ℹ]  nodegroup "managed-ng-public-01" has 3 node(s)
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-11-0.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-17-71.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-47-6.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [ℹ]  waiting for at least 3 node(s) to become ready in "managed-ng-public-01"
2025-01-25 07:56:52 [ℹ]  nodegroup "managed-ng-public-01" has 3 node(s)
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-11-0.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-17-71.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-47-6.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [ℹ]  nodegroup "managed-ng-private-01" has 3 node(s)
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-61-150.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-79-93.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-81-61.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [ℹ]  waiting for at least 3 node(s) to become ready in "managed-ng-private-01"
2025-01-25 07:56:52 [ℹ]  nodegroup "managed-ng-private-01" has 3 node(s)
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-61-150.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-79-93.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [ℹ]  node "ip-10-11-81-61.ap-northeast-2.compute.internal" is ready
2025-01-25 07:56:52 [✔]  created 2 managed nodegroup(s) in cluster "eksworkshop"
2025-01-25 07:56:53 [ℹ]  IRSA is set for "aws-ebs-csi-driver" addon; will use this to configure IAM permissions
2025-01-25 07:56:53 [!]  the recommended way to provide IAM permissions for "aws-ebs-csi-driver" addon is via pod identity associations; after addon creation is completed, run `eksctl utils migrate-to-pod-identity`
2025-01-25 07:56:53 [ℹ]  creating role using provided policies for "aws-ebs-csi-driver" addon
2025-01-25 07:56:54 [ℹ]  deploying stack "eksctl-eksworkshop-addon-aws-ebs-csi-driver"
2025-01-25 07:56:54 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-addon-aws-ebs-csi-driver"
2025-01-25 07:57:24 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-addon-aws-ebs-csi-driver"
2025-01-25 07:58:11 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-addon-aws-ebs-csi-driver"
2025-01-25 07:58:11 [ℹ]  creating addon
2025-01-25 07:59:08 [ℹ]  addon "aws-ebs-csi-driver" active
2025-01-25 07:59:09 [ℹ]  kubectl command should work with "/home/ec2-user/.kube/config", try 'kubectl get nodes'
2025-01-25 07:59:09 [✔]  EKS cluster "eksworkshop" in "ap-northeast-2" region is ready
</code></pre>

Cloudformation 에는 6개의 Stack이 만들어졌습니다.

<figure><img src="../.gitbook/assets/image (501).png" alt=""><figcaption></figcaption></figure>

### 5. Cluster 생성 확인

정상적으로 Cluster가 생성되었는지 확인합니다.

```
kubectl get nodes

```

출력 결과 예시

```
$ kubectl get nodes
NAME                                              STATUS   ROLES    AGE     VERSION
ip-10-11-0-140.ap-northeast-2.compute.internal    Ready    <none>   8m38s   v1.25.16-eks-5e0fdde
ip-10-11-15-69.ap-northeast-2.compute.internal    Ready    <none>   10m     v1.25.16-eks-5e0fdde
ip-10-11-21-240.ap-northeast-2.compute.internal   Ready    <none>   8m37s   v1.25.16-eks-5e0fdde
ip-10-11-23-43.ap-northeast-2.compute.internal    Ready    <none>   10m     v1.25.16-eks-5e0fdde
ip-10-11-27-62.ap-northeast-2.compute.internal    Ready    <none>   9m57s   v1.25.16-eks-5e0fdde
ip-10-11-30-4.ap-northeast-2.compute.internal     Ready    <none>   8m36s   v1.25.16-eks-5e0fdde
ip-10-11-33-221.ap-northeast-2.compute.internal   Ready    <none>   8m38s   v1.25.16-eks-5e0fdde
ip-10-11-41-90.ap-northeast-2.compute.internal    Ready    <none>   10m     v1.25.16-eks-5e0fdde
ip-10-11-45-180.ap-northeast-2.compute.internal   Ready    <none>   10m     v1.25.16-eks-5e0fdde
ip-10-11-45-245.ap-northeast-2.compute.internal   Ready    <none>   8m39s   v1.25.16-eks-5e0fdde
ip-10-11-6-70.ap-northeast-2.compute.internal     Ready    <none>   8m37s   v1.25.16-eks-5e0fdde
ip-10-11-8-132.ap-northeast-2.compute.internal    Ready    <none>   9m57s   v1.25.16-eks-5e0fdde
```

* 생성된 VPC와 Subnet, Internet Gateway, NAT Gateway, Route Table등을 확인해 봅니다.
* 생성된 EC2 Worker Node들도 확인해 봅니다.
* EKS와 eksctl을 통해 생생된 Cloudformation도 확인해 봅니다.

다음과 같은 구성도가 완성되었습니다.

![](<../.gitbook/assets/image (205).png>)



