---
description: 'update : 2020-11-12'
---

# eksctl 구성

## eksctl 소개

`eksctl`은 관리형 Kubernetes 서비스 인 EKS에서 클러스터를 생성하기위한 간단한 CLI 도구입니다. Go로 작성되었으며 CloudFormation을 사용하며 [Weaveworks](https://www.weave.works/) 가 작성했으며 단 하나의 명령으로 몇 분 안에 기본 클러스터를 만듭니다.

## eksctl을 통한 EKS 구성

### 1.eksctl 설치

아래와 같이 eksctl을 설치합니다.

```text
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version

```

### 2.VPC/Subnet 정보 확인

앞서 [Cloudformation 구성](cloudformation.md#3-stack)에서 생성한 VPC id, Subnet id를 확인합니다.

다음 명령을 통해서 확인 할 수 있습니다.

```text
aws ec2 describe-vpcs --filters Name=tag:Name,Values=eksworkshop | jq -r '.Vpcs[].VpcId'
aws ec2 describe-subnets  --filters "Name=cidr-block,Values=10.11.*" --query 'Subnets[*].[CidrBlock,SubnetId,AvailabilityZone]' --output table

```

아래는 예제입니다.

```text
VPC ID - vpc-0bdd67cbc64aba483
Public subnet 01 - subnet-07128799309969cc4
Public subnet 02 - subnet-0240b3e1a59fb4802
Public subnet 03 - subnet-0ff369e1bb376c450
Private subnet 01 - subnet-062dad2e0fc99a677
Private subnet 02 - subnet-0859b55ff38586ab1
Private subnet 03 - subnet-0cf39d70d89565812
```

저장해둔 Region 정보와 master\_arn을 확인합니다.

```text
echo $AWS_REGION
echo $MASTER_ARN

```

VPC id, subnet id, region, master arn은 eksctl을 통해 EKS cluster를 배포하는 데 사용합니다.

### 3. eksctl 배포 yaml 수정

Cloud9 IDE 편집기에서 아래와 같이 수정합니다. 수정내용은 현재 생성된 VPC, Subnet ID , key 위치 입니다.

![](../.gitbook/assets/image%20%28145%29.png)

수정할 블록의 예시입니다.

```text
metadata:
  name: eksworkshop
  region: ap-northeast-2

vpc: 
  id: vpc-0bdd67cbc64aba483
  subnets:
    public:
      ap-northeast-2a: { id: subnet-07128799309969cc4}
      ap-northeast-2b: { id: subnet-0240b3e1a59fb4802}
      ap-northeast-2c: { id: subnet-0ff369e1bb376c450}
    private:
      ap-northeast-2a: { id: subnet-062dad2e0fc99a677}
      ap-northeast-2b: { id: subnet-0859b55ff38586ab1}
      ap-northeast-2c: { id: subnet-0cf39d70d89565812}

secretsEncryption:
  keyARN: arn:aws:kms:ap-northeast-2:584172017494:key/6ba77a10-cf0b-4af8-b7b7-54bb5ea2f5c9
```

### 4. cluster 생성

eksctl을 통해 EKS Cluster를 생성합니다.

```text
eksctl create cluster --config-file=/home/ec2-user/environment/myeks/eksworkshop-cluster.yaml 
```

{% hint style="info" %}
Cluster를 생성하기 위해 20~30분 정도 시간이 소요됩니다.
{% endhint %}

출력 결과 예시

```text
~/environment $ eksctl create cluster --config-file=/home/ec2-user/environment/myeks/eksworkshop-cluster.yaml 
[ℹ]  eksctl version 0.31.0
[ℹ]  using region ap-northeast-2
[✔]  using existing VPC (vpc-0bdd67cbc64aba483) and subnets (private:[subnet-0cf39d70d89565812 subnet-062dad2e0fc99a677 subnet-0859b55ff38586ab1] public:[subnet-07128799309969cc4 subnet-0240b3e1a59fb4802 subnet-0ff369e1bb376c450])
[!]  custom VPC/subnets will be used; if resulting cluster doesn't function as expected, make sure to review the configuration of VPC/subnets
[ℹ]  nodegroup "ng1-public" will use "ami-025592e84db381916" [AmazonLinux2/1.17]
[ℹ]  using SSH public key "/home/ec2-user/environment/eksworkshop.pub" as "eksctl-eksworkshop-nodegroup-ng1-public-02:ff:18:ce:c0:1f:0f:60:77:af:2a:a4:5a:60:d4:88" 
[ℹ]  nodegroup "ng2-private" will use "ami-025592e84db381916" [AmazonLinux2/1.17]
[ℹ]  using SSH public key "/home/ec2-user/environment/eksworkshop.pub" as "eksctl-eksworkshop-nodegroup-ng2-private-02:ff:18:ce:c0:1f:0f:60:77:af:2a:a4:5a:60:d4:88" 
[ℹ]  using Kubernetes version 1.17
[ℹ]  creating EKS cluster "eksworkshop" in "ap-northeast-2" region with un-managed nodes
[ℹ]  2 nodegroups (ng1-public, ng2-private) were included (based on the include/exclude rules)
[ℹ]  will create a CloudFormation stack for cluster itself and 2 nodegroup stack(s)
[ℹ]  will create a CloudFormation stack for cluster itself and 0 managed nodegroup stack(s)
[ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=ap-northeast-2 --cluster=eksworkshop'
[ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "eksworkshop" in "ap-northeast-2"
[ℹ]  2 sequential tasks: { create cluster control plane "eksworkshop", 2 sequential sub-tasks: { update CloudWatch logging configuration, 2 parallel sub-tasks: { create nodegroup "ng1-public", create nodegroup "ng2-private" } } }
[ℹ]  building cluster stack "eksctl-eksworkshop-cluster"
[ℹ]  deploying stack "eksctl-eksworkshop-cluster"
[✔]  configured CloudWatch logging for cluster "eksworkshop" in "ap-northeast-2" (enabled types: api, audit, authenticator, controllerManager, scheduler & no types disabled)
[ℹ]  building nodegroup stack "eksctl-eksworkshop-nodegroup-ng2-private"
[ℹ]  building nodegroup stack "eksctl-eksworkshop-nodegroup-ng1-public"
[ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-ng1-public"
[ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-ng2-private"
[ℹ]  waiting for the control plane availability...
[✔]  saved kubeconfig as "/home/ec2-user/.kube/config"
[ℹ]  no tasks
[✔]  all EKS cluster resources for "eksworkshop" have been created
[ℹ]  adding identity "arn:aws:iam::584172017494:role/eksctl-eksworkshop-nodegroup-ng1-NodeInstanceRole-12J9DVQEL021B" to auth ConfigMap
[ℹ]  nodegroup "ng1-public" has 0 node(s)
[ℹ]  waiting for at least 3 node(s) to become ready in "ng1-public"
[ℹ]  nodegroup "ng1-public" has 3 node(s)
[ℹ]  node "ip-10-11-4-228.ap-northeast-2.compute.internal" is ready
[ℹ]  node "ip-10-11-54-141.ap-northeast-2.compute.internal" is ready
[ℹ]  node "ip-10-11-65-66.ap-northeast-2.compute.internal" is ready
[ℹ]  adding identity "arn:aws:iam::584172017494:role/eksctl-eksworkshop-nodegroup-ng2-NodeInstanceRole-1LQ0HMEEW14WW" to auth ConfigMap
[ℹ]  nodegroup "ng2-private" has 0 node(s)
[ℹ]  waiting for at least 3 node(s) to become ready in "ng2-private"
[ℹ]  nodegroup "ng2-private" has 3 node(s)
[ℹ]  node "ip-10-11-114-209.ap-northeast-2.compute.internal" is ready
[ℹ]  node "ip-10-11-154-48.ap-northeast-2.compute.internal" is ready
[ℹ]  node "ip-10-11-191-152.ap-northeast-2.compute.internal" is ready
[ℹ]  kubectl command should work with "/home/ec2-user/.kube/config", try 'kubectl get nodes'
[✔]  EKS cluster "eksworkshop" in "ap-northeast-2" region is ready
```

### 5. Cluster 생성 확인

정상적으로 Cluster가 생성되었는지 확인합니다.

```text
kubectl get nodes
```

출력 결과 예시

```text
~/environment $ kubectl get nodes
NAME                                               STATUS   ROLES    AGE    VERSION
ip-10-11-114-132.ap-northeast-2.compute.internal   Ready    <none>   2d1h   v1.16.12-eks-904af05
ip-10-11-146-170.ap-northeast-2.compute.internal   Ready    <none>   2d1h   v1.16.12-eks-904af05
ip-10-11-189-67.ap-northeast-2.compute.internal    Ready    <none>   2d1h   v1.16.12-eks-904af05
ip-10-11-31-153.ap-northeast-2.compute.internal    Ready    <none>   2d1h   v1.16.12-eks-904af05
ip-10-11-53-186.ap-northeast-2.compute.internal    Ready    <none>   2d1h   v1.16.12-eks-904af05
ip-10-11-69-28.ap-northeast-2.compute.internal     Ready    <none>   2d1h   v1.16.12-eks-904af05
```

* 생성된 VPC와 Subnet, Internet Gateway, NAT Gateway, Route Table등을 확인해 봅니다.
* 생성된 EC2 Worker Node들도 확인해 봅니다.
* EKS와 eksctl을 통해 생생된 Cloudformation도 확인해 봅니다.

### 6.eksctl yaml code 참조

eksctl 배포를 위한 EKS Cluster yaml 파일

```text
# A simple example of ClusterConfig object:
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksworkshop
  region: ap-northeast-2
  
vpc: 
  id: vpc-086186d07739fc568	
  subnets:
    public:
      ap-northeast-2a: { id: subnet-0d4864467efbf04a4}
      ap-northeast-2b: { id: subnet-0662f5c059aac8576}
      ap-northeast-2c: { id: subnet-0253297add231a70d}
    private:
      ap-northeast-2a: { id: subnet-051b3655d99b2cf0b}
      ap-northeast-2b: { id: subnet-0e3c5d12ade472f91}
      ap-northeast-2c: { id: subnet-0dd1d98a956bf8227}

secretsEncryption:
  keyARN: arn:aws:kms:ap-northeast-2:909121566064:key/9a0c5a6c-be81-4463-90e4-e3b1252d96fc

nodeGroups:
  - name: ng1-public
    instanceType: m5.xlarge
    desiredCapacity: 3
    minSize: 3
    maxSize: 9
    volumeSize: 30
    volumeType: gp2 
    amiFamily: AmazonLinux2
    labels:
      nodegroup-type: "frontend-workloads"
    ssh: 
        publicKeyPath: "/home/ec2-user/environment/eksworkshop.pub"
        allow: true
    iam:
      attachPolicyARNs:
      withAddonPolicies:
        autoScaler: true
        cloudWatch: true
        ebs: true
        fsx: true
        efs: true

  - name: ng2-private
    instanceType: m5.xlarge
    desiredCapacity: 3
    privateNetworking: true
    minSize: 3
    maxSize: 9
    volumeSize: 30
    volumeType: gp2 
    amiFamily: AmazonLinux2
    labels:
      nodegroup-type: "backend-workloads"
    ssh: 
        publicKeyPath: "/home/ec2-user/environment/eksworkshop.pub"
        allow: true
    iam:
      withAddonPolicies:
        autoScaler: true
        cloudWatch: true
        ebs: true
        fsx: true
        efs: true


cloudWatch:
    clusterLogging:
        enableTypes: ["api", "audit", "authenticator", "controllerManager", "scheduler"]



```



