---
description: 'update : 2024-01-27 / 20min'
---

# eksctl 구성

## eksctl 소개

`eksctl`은 관리형 Kubernetes 서비스 인 EKS에서 클러스터를 생성하기위한 간단한 CLI 도구입니다. Go로 작성되었으며 CloudFormation을 사용하며 [Weaveworks](https://www.weave.works/) 가 작성했으며 단 하나의 명령으로 몇 분 안에 기본 클러스터를 만듭니다.

이것은 EKS를 구성하기 위한 도구 이며,  AWS 관리콘솔에서 제공하는 EKS UI, CDK, Terraform, Rancher 등 다양한 도구로도 구성이 가능합니다.

## eksctl을 통한 EKS 구성

### 1.eksctl 설치

Cloud9 사전 환경 구성에서 이미 설치되어 있습니다. 이 단계는 생략이 가능합니다.

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
2024-01-28 12:33:12 [ℹ]  eksctl version 0.169.0
2024-01-28 12:33:12 [ℹ]  using region ap-northeast-2
2024-01-28 12:33:13 [✔]  using existing VPC (vpc-0bd8bbf4026631bcd) and subnets (private:map[PrivateSubnet01:{subnet-0932849f4143fbe5a ap-northeast-2a 10.11.0.0/20 0 } PrivateSubnet02:{subnet-02b053c433cc38bb2 ap-northeast-2b 10.11.16.0/20 0 } PrivateSubnet03:{subnet-0ef308c4e1e467895 ap-northeast-2c 10.11.32.0/20 0 }] public:map[PublicSubnet01:{subnet-0932849f4143fbe5a ap-northeast-2a 10.11.0.0/20 0 } PublicSubnet02:{subnet-02b053c433cc38bb2 ap-northeast-2b 10.11.16.0/20 0 } PublicSubnet03:{subnet-0ef308c4e1e467895 ap-northeast-2c 10.11.32.0/20 0 }])
2024-01-28 12:33:13 [!]  custom VPC/subnets will be used; if resulting cluster doesn't function as expected, make sure to review the configuration of VPC/subnets
2024-01-28 12:33:13 [ℹ]  nodegroup "ng-public-01" will use "ami-0515851299ec725f7" [AmazonLinux2/1.25]
2024-01-28 12:33:13 [ℹ]  nodegroup "ng-private-01" will use "ami-0515851299ec725f7" [AmazonLinux2/1.25]
2024-01-28 12:33:13 [ℹ]  nodegroup "managed-ng-public-01" will use "" [AmazonLinux2/1.25]
2024-01-28 12:33:13 [ℹ]  nodegroup "managed-ng-private-01" will use "" [AmazonLinux2/1.25]
2024-01-28 12:33:13 [ℹ]  using Kubernetes version 1.25
2024-01-28 12:33:13 [ℹ]  creating EKS cluster "eksworkshop" in "ap-northeast-2" region with managed nodes and un-managed nodes
2024-01-28 12:33:13 [ℹ]  4 nodegroups (managed-ng-private-01, managed-ng-public-01, ng-private-01, ng-public-01) were included (based on the include/exclude rules)
2024-01-28 12:33:13 [ℹ]  will create a CloudFormation stack for cluster itself and 2 nodegroup stack(s)
2024-01-28 12:33:13 [ℹ]  will create a CloudFormation stack for cluster itself and 2 managed nodegroup stack(s)
2024-01-28 12:33:13 [ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=ap-northeast-2 --cluster=eksworkshop'
2024-01-28 12:33:13 [ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "eksworkshop" in "ap-northeast-2"
2024-01-28 12:33:13 [ℹ]  configuring CloudWatch logging for cluster "eksworkshop" in "ap-northeast-2" (enabled types: api, audit, authenticator, controllerManager, scheduler & no types disabled)
2024-01-28 12:33:13 [ℹ]  
2 sequential tasks: { create cluster control plane "eksworkshop", 
    2 sequential sub-tasks: { 
        5 sequential sub-tasks: { 
            wait for control plane to become ready,
            associate IAM OIDC provider,
            no tasks,
            restart daemonset "kube-system/aws-node",
            1 task: { create addons },
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
2024-01-28 12:33:13 [ℹ]  building cluster stack "eksctl-eksworkshop-cluster"
2024-01-28 12:33:13 [ℹ]  deploying stack "eksctl-eksworkshop-cluster"
2024-01-28 12:33:43 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-cluster"
2024-01-28 12:43:15 [ℹ]  daemonset "kube-system/aws-node" restarted
2024-01-28 12:43:16 [ℹ]  creating role using provided policies ARNs
2024-01-28 12:43:16 [ℹ]  deploying stack "eksctl-eksworkshop-addon-vpc-cni"
2024-01-28 12:43:16 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-addon-vpc-cni"
2024-01-28 12:43:46 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-addon-vpc-cni"
2024-01-28 12:43:46 [ℹ]  creating addon
2024-01-28 12:43:57 [ℹ]  addon "vpc-cni" active
2024-01-28 12:43:57 [ℹ]  building nodegroup stack "eksctl-eksworkshop-nodegroup-ng-private-01"
2024-01-28 12:43:57 [ℹ]  building nodegroup stack "eksctl-eksworkshop-nodegroup-ng-public-01"
2024-01-28 12:43:57 [!]  subnets contain a mix of both local and availability zones
2024-01-28 12:43:57 [ℹ]  building managed nodegroup stack "eksctl-eksworkshop-nodegroup-managed-ng-private-01"
2024-01-28 12:43:57 [ℹ]  building managed nodegroup stack "eksctl-eksworkshop-nodegroup-managed-ng-public-01"
2024-01-28 12:43:57 [ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-managed-ng-private-01"
2024-01-28 12:43:57 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-managed-ng-private-01"
2024-01-28 12:43:57 [ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-managed-ng-public-01"
2024-01-28 12:43:57 [ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-ng-private-01"
2024-01-28 12:43:57 [ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-ng-public-01"
2024-01-28 12:43:57 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-managed-ng-public-01"
2024-01-28 12:47:28 [ℹ]  waiting for the control plane to become ready
2024-01-28 12:47:29 [✔]  saved kubeconfig as "/home/ec2-user/.kube/config"
2024-01-28 12:47:29 [ℹ]  no tasks
2024-01-28 12:47:29 [✔]  all EKS cluster resources for "eksworkshop" have been created
2024-01-28 12:47:29 [ℹ]  nodegroup "ng-public-01" has 3 node(s)
2024-01-28 12:47:29 [ℹ]  node "ip-10-11-0-140.ap-northeast-2.compute.internal" is ready
2024-01-28 12:47:29 [ℹ]  node "ip-10-11-30-4.ap-northeast-2.compute.internal" is ready
2024-01-28 12:47:29 [ℹ]  node "ip-10-11-33-221.ap-northeast-2.compute.internal" is ready
2024-01-28 12:47:29 [ℹ]  waiting for at least 3 node(s) to become ready in "ng-public-01"
2024-01-28 12:47:29 [ℹ]  nodegroup "ng-private-01" has 3 node(s)
2024-01-28 12:47:29 [ℹ]  node "ip-10-11-21-240.ap-northeast-2.compute.internal" is ready
2024-01-28 12:47:29 [ℹ]  node "ip-10-11-45-245.ap-northeast-2.compute.internal" is ready
2024-01-28 12:47:29 [ℹ]  node "ip-10-11-6-70.ap-northeast-2.compute.internal" is ready
2024-01-28 12:47:29 [ℹ]  waiting for at least 3 node(s) to become ready in "ng-private-01"
2024-01-28 12:47:29 [ℹ]  nodegroup "managed-ng-public-01" has 3 node(s)
2024-01-28 12:47:29 [ℹ]  node "ip-10-11-27-62.ap-northeast-2.compute.internal" is ready
2024-01-28 12:47:29 [ℹ]  node "ip-10-11-41-90.ap-northeast-2.compute.internal" is ready
2024-01-28 12:47:29 [ℹ]  node "ip-10-11-8-132.ap-northeast-2.compute.internal" is ready
2024-01-28 12:47:29 [ℹ]  waiting for at least 3 node(s) to become ready in "managed-ng-public-01"
2024-01-28 12:47:29 [ℹ]  nodegroup "managed-ng-private-01" has 3 node(s)
2024-01-28 12:47:29 [ℹ]  node "ip-10-11-15-69.ap-northeast-2.compute.internal" is ready
2024-01-28 12:47:29 [ℹ]  node "ip-10-11-23-43.ap-northeast-2.compute.internal" is ready
2024-01-28 12:47:29 [ℹ]  node "ip-10-11-45-180.ap-northeast-2.compute.internal" is ready
2024-01-28 12:47:29 [ℹ]  waiting for at least 3 node(s) to become ready in "managed-ng-private-01"
2024-01-28 12:47:30 [ℹ]  no recommended policies found, proceeding without any IAM
2024-01-28 12:47:30 [ℹ]  creating addon
2024-01-28 12:47:41 [ℹ]  addon "coredns" active
2024-01-28 12:47:41 [ℹ]  no recommended policies found, proceeding without any IAM
2024-01-28 12:47:41 [ℹ]  creating addon
2024-01-28 12:48:24 [ℹ]  addon "kube-proxy" active
2024-01-28 12:48:25 [ℹ]  creating role using provided well known policies
2024-01-28 12:48:25 [ℹ]  deploying stack "eksctl-eksworkshop-addon-aws-ebs-csi-driver"
2024-01-28 12:48:25 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-addon-aws-ebs-csi-driver"
2024-01-28 12:48:55 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-addon-aws-ebs-csi-driver"
2024-01-28 12:49:34 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-addon-aws-ebs-csi-driver"
2024-01-28 12:49:34 [ℹ]  creating addon
2024-01-28 12:50:33 [ℹ]  addon "aws-ebs-csi-driver" active
2024-01-28 12:50:34 [ℹ]  kubectl command should work with "/home/ec2-user/.kube/config", try 'kubectl get nodes'
2024-01-28 12:50:34 [✔]  EKS cluster "eksworkshop" in "ap-northeast-2" region is ready
```

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



