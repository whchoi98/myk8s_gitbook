---
description: 'update : 2022-10-22 / 20min'
---

# eksctl 구성

## eksctl 소개

`eksctl`은 관리형 Kubernetes 서비스 인 EKS에서 클러스터를 생성하기위한 간단한 CLI 도구입니다. Go로 작성되었으며 CloudFormation을 사용하며 [Weaveworks](https://www.weave.works/) 가 작성했으며 단 하나의 명령으로 몇 분 안에 기본 클러스터를 만듭니다.

이것은 EKS를 구성하기 위한 도구 이며,  AWS 관리콘솔에서 제공하는 EKS UI, CDK, Terraform, Rancher 등 다양한 도구로도 구성이 가능합니다.

## eksctl을 통한 EKS 구성

### 1.eksctl 설치

아래와 같이 eksctl을 Cloud9에 설치하고 버전을 확인합니다.

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

앞서 [Cloudformation 구성](cloudformation.md#3-stack)에서 생성한 VPC 자원들에 대한 고유의 자원 값을 추출해서, Cloud9 내에서 환경 변수에 저장합니다

```
~/environment/myeks/shell/eks_shell.sh

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

```
2022-04-21 13:29:20 [ℹ]  eksctl version 0.93.0
2022-04-21 13:29:20 [ℹ]  using region ap-northeast-2
2022-04-21 13:29:20 [✔]  using existing VPC (vpc-03e394fb459d894fd) and subnets (private:map[PrivateSubnet01:{subnet-0c5f9613e45bbf12d ap-northeast-2a 10.11.64.0/20} PrivateSubnet02:{subnet-068c86b0cd8fc9bbc ap-northeast-2b 10.11.80.0/20} PrivateSubnet03:{subnet-0bf66408a64af0812 ap-northeast-2c 10.11.96.0/20}] public:map[PublicSubnet01:{subnet-0a7d18f788f913a4d ap-northeast-2a 10.11.0.0/20} PublicSubnet02:{subnet-01f00b8cd50a95661 ap-northeast-2b 10.11.16.0/20} PublicSubnet03:{subnet-0f6b932d0e8db351f ap-northeast-2c 10.11.32.0/20}])
2022-04-21 13:29:20 [ℹ]  nodegroup "ng-public-01" will use "ami-0faa1b4cd7d224b2d" [AmazonLinux2/1.21]
2022-04-21 13:29:20 [ℹ]  using SSH public key "/home/ec2-user/environment/eksworkshop.pub" as "eksctl-eksworkshop-nodegroup-ng-public-01-c7:3c:65:44:87:bc:7d:af:86:b5:e5:9a:c0:02:72:1f" 
2022-04-21 13:29:20 [ℹ]  nodegroup "ng-private-01" will use "ami-0faa1b4cd7d224b2d" [AmazonLinux2/1.21]
2022-04-21 13:29:20 [ℹ]  using SSH public key "/home/ec2-user/environment/eksworkshop.pub" as "eksctl-eksworkshop-nodegroup-ng-private-01-c7:3c:65:44:87:bc:7d:af:86:b5:e5:9a:c0:02:72:1f" 
2022-04-21 13:29:20 [ℹ]  nodegroup "managed-ng-public-01" will use "" [AmazonLinux2/1.21]
2022-04-21 13:29:20 [ℹ]  using SSH public key "/home/ec2-user/environment/eksworkshop.pub" as "eksctl-eksworkshop-nodegroup-managed-ng-public-01-c7:3c:65:44:87:bc:7d:af:86:b5:e5:9a:c0:02:72:1f" 
2022-04-21 13:29:21 [ℹ]  nodegroup "managed-ng-private-01" will use "" [AmazonLinux2/1.21]
2022-04-21 13:29:21 [ℹ]  using SSH public key "/home/ec2-user/environment/eksworkshop.pub" as "eksctl-eksworkshop-nodegroup-managed-ng-private-01-c7:3c:65:44:87:bc:7d:af:86:b5:e5:9a:c0:02:72:1f" 
2022-04-21 13:29:21 [ℹ]  using Kubernetes version 1.21
2022-04-21 13:29:21 [ℹ]  creating EKS cluster "eksworkshop" in "ap-northeast-2" region with managed nodes and un-managed nodes
2022-04-21 13:29:21 [ℹ]  4 nodegroups (managed-ng-private-01, managed-ng-public-01, ng-private-01, ng-public-01) were included (based on the include/exclude rules)
2022-04-21 13:29:21 [ℹ]  will create a CloudFormation stack for cluster itself and 2 nodegroup stack(s)
2022-04-21 13:29:21 [ℹ]  will create a CloudFormation stack for cluster itself and 2 managed nodegroup stack(s)
2022-04-21 13:29:21 [ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "eksworkshop" in "ap-northeast-2"
2022-04-21 13:29:21 [ℹ]  configuring CloudWatch logging for cluster "eksworkshop" in "ap-northeast-2" (enabled types: api, audit, authenticator, controllerManager, scheduler & no types disabled)
2022-04-21 13:29:21 [ℹ]  
2 sequential tasks: { create cluster control plane "eksworkshop", 
    2 sequential sub-tasks: { 
        wait for control plane to become ready,
        4 parallel sub-tasks: { 
            create nodegroup "ng-public-01",
            create nodegroup "ng-private-01",
            create managed nodegroup "managed-ng-public-01",
            create managed nodegroup "managed-ng-private-01",
        },
    } 
}
2022-04-21 13:29:21 [ℹ]  building cluster stack "eksctl-eksworkshop-cluster"
2022-04-21 13:29:21 [ℹ]  deploying stack "eksctl-eksworkshop-cluster"
2022-04-21 13:29:51 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-cluster"
2022-04-21 13:42:22 [ℹ]  building nodegroup stack "eksctl-eksworkshop-nodegroup-ng-public-01"
2022-04-21 13:42:22 [ℹ]  building nodegroup stack "eksctl-eksworkshop-nodegroup-ng-private-01"
2022-04-21 13:42:22 [ℹ]  building managed nodegroup stack "eksctl-eksworkshop-nodegroup-managed-ng-private-01"
2022-04-21 13:42:22 [ℹ]  building managed nodegroup stack "eksctl-eksworkshop-nodegroup-managed-ng-public-01"
2022-04-21 13:42:22 [ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-managed-ng-private-01"
2022-04-21 13:42:22 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-managed-ng-private-01"
2022-04-21 13:42:22 [ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-ng-public-01"
2022-04-21 13:42:22 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-public-01"
2022-04-21 13:42:22 [ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-ng-private-01"
2022-04-21 13:42:22 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-private-01"
2022-04-21 13:42:22 [ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-managed-ng-public-01"
2022-04-21 13:42:22 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-managed-ng-public-01"
2022-04-21 13:42:40 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-public-01"
2022-04-21 13:42:42 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-managed-ng-private-01"
2022-04-21 13:42:42 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-private-01"

2022-04-21 13:46:12 [ℹ]  waiting for the control plane availability...
2022-04-21 13:46:12 [✔]  saved kubeconfig as "/home/ec2-user/.kube/config"
2022-04-21 13:46:12 [✔]  all EKS cluster resources for "eksworkshop" have been created
2022-04-21 13:46:12 [ℹ]  adding identity "arn:aws:iam::511357390802:role/eksctl-eksworkshop-nodegroup-ng-p-NodeInstanceRole-1WO7W7CJXOLYY" to auth ConfigMap
2022-04-21 13:46:13 [ℹ]  nodegroup "ng-public-01" has 0 node(s)
2022-04-21 13:46:13 [ℹ]  waiting for at least 3 node(s) to become ready in "ng-public-01"
2022-04-21 13:46:36 [ℹ]  nodegroup "ng-public-01" has 3 node(s)
2022-04-21 13:46:36 [ℹ]  node "ip-10-11-14-197.ap-northeast-2.compute.internal" is ready
2022-04-21 13:46:36 [ℹ]  node "ip-10-11-21-169.ap-northeast-2.compute.internal" is ready
2022-04-21 13:46:36 [ℹ]  node "ip-10-11-38-99.ap-northeast-2.compute.internal" is ready
2022-04-21 13:46:36 [ℹ]  adding identity "arn:aws:iam::511357390802:role/eksctl-eksworkshop-nodegroup-ng-p-NodeInstanceRole-MONX9B0T65E3" to auth ConfigMap
2022-04-21 13:46:36 [ℹ]  nodegroup "ng-private-01" has 0 node(s)
2022-04-21 13:46:36 [ℹ]  waiting for at least 3 node(s) to become ready in "ng-private-01"
2022-04-21 13:47:09 [ℹ]  nodegroup "ng-private-01" has 3 node(s)
2022-04-21 13:47:09 [ℹ]  node "ip-10-11-73-205.ap-northeast-2.compute.internal" is ready
2022-04-21 13:47:09 [ℹ]  node "ip-10-11-95-138.ap-northeast-2.compute.internal" is ready
2022-04-21 13:47:09 [ℹ]  node "ip-10-11-99-40.ap-northeast-2.compute.internal" is ready
2022-04-21 13:47:09 [ℹ]  nodegroup "managed-ng-public-01" has 3 node(s)
2022-04-21 13:47:09 [ℹ]  node "ip-10-11-14-39.ap-northeast-2.compute.internal" is ready
2022-04-21 13:47:09 [ℹ]  node "ip-10-11-17-179.ap-northeast-2.compute.internal" is ready
2022-04-21 13:47:09 [ℹ]  node "ip-10-11-34-10.ap-northeast-2.compute.internal" is ready
2022-04-21 13:47:09 [ℹ]  waiting for at least 3 node(s) to become ready in "managed-ng-public-01"
2022-04-21 13:47:09 [ℹ]  nodegroup "managed-ng-public-01" has 3 node(s)
2022-04-21 13:47:09 [ℹ]  node "ip-10-11-14-39.ap-northeast-2.compute.internal" is ready
2022-04-21 13:47:09 [ℹ]  node "ip-10-11-17-179.ap-northeast-2.compute.internal" is ready
2022-04-21 13:47:09 [ℹ]  node "ip-10-11-34-10.ap-northeast-2.compute.internal" is ready
2022-04-21 13:47:09 [ℹ]  nodegroup "managed-ng-private-01" has 3 node(s)
2022-04-21 13:47:09 [ℹ]  node "ip-10-11-100-116.ap-northeast-2.compute.internal" is ready
2022-04-21 13:47:09 [ℹ]  node "ip-10-11-67-237.ap-northeast-2.compute.internal" is ready
2022-04-21 13:47:09 [ℹ]  node "ip-10-11-88-144.ap-northeast-2.compute.internal" is ready
2022-04-21 13:47:09 [ℹ]  waiting for at least 3 node(s) to become ready in "managed-ng-private-01"
2022-04-21 13:47:09 [ℹ]  nodegroup "managed-ng-private-01" has 3 node(s)
2022-04-21 13:47:09 [ℹ]  node "ip-10-11-100-116.ap-northeast-2.compute.internal" is ready
2022-04-21 13:47:09 [ℹ]  node "ip-10-11-67-237.ap-northeast-2.compute.internal" is ready
2022-04-21 13:47:09 [ℹ]  node "ip-10-11-88-144.ap-northeast-2.compute.internal" is ready
2022-04-21 13:47:10 [ℹ]  kubectl command should work with "/home/ec2-user/.kube/config", try 'kubectl get nodes'
2022-04-21 13:47:10 [✔]  EKS cluster "eksworkshop" in "ap-northeast-2" region is ready

```

### 5. Cluster 생성 확인

정상적으로 Cluster가 생성되었는지 확인합니다.

```
kubectl get nodes

```

출력 결과 예시

```
$  kubectl get nodes
NAME                                              STATUS   ROLES    AGE     VERSION
ip-10-11-1-225.ap-northeast-2.compute.internal    Ready    <none>   2m16s   v1.22.15-eks-fb459a0
ip-10-11-15-121.ap-northeast-2.compute.internal   Ready    <none>   5m26s   v1.22.15-eks-fb459a0
ip-10-11-27-197.ap-northeast-2.compute.internal   Ready    <none>   2m17s   v1.22.15-eks-fb459a0
ip-10-11-27-216.ap-northeast-2.compute.internal   Ready    <none>   5m14s   v1.22.15-eks-fb459a0
ip-10-11-33-12.ap-northeast-2.compute.internal    Ready    <none>   2m17s   v1.22.15-eks-fb459a0
ip-10-11-47-74.ap-northeast-2.compute.internal    Ready    <none>   5m25s   v1.22.15-eks-fb459a0
ip-10-11-50-128.ap-northeast-2.compute.internal   Ready    <none>   83s     v1.22.15-eks-fb459a0
ip-10-11-59-180.ap-northeast-2.compute.internal   Ready    <none>   5m34s   v1.22.15-eks-fb459a0
ip-10-11-69-123.ap-northeast-2.compute.internal   Ready    <none>   5m23s   v1.22.15-eks-fb459a0
ip-10-11-74-5.ap-northeast-2.compute.internal     Ready    <none>   84s     v1.22.15-eks-fb459a0
ip-10-11-80-166.ap-northeast-2.compute.internal   Ready    <none>   79s     v1.22.15-eks-fb459a0
ip-10-11-90-157.ap-northeast-2.compute.internal   Ready    <none>   5m30s   v1.22.15-eks-fb459a0
```

* 생성된 VPC와 Subnet, Internet Gateway, NAT Gateway, Route Table등을 확인해 봅니다.
* 생성된 EC2 Worker Node들도 확인해 봅니다.
* EKS와 eksctl을 통해 생생된 Cloudformation도 확인해 봅니다.

다음과 같은 구성도가 완성되었습니다.

![](<../.gitbook/assets/image (205).png>)



