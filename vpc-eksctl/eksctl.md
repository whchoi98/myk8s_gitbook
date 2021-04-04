---
description: 'update : 2020-11-15'
---

# eksctl 구성

## eksctl 소개

`eksctl`은 관리형 Kubernetes 서비스 인 EKS에서 클러스터를 생성하기위한 간단한 CLI 도구입니다. Go로 작성되었으며 CloudFormation을 사용하며 [Weaveworks](https://www.weave.works/) 가 작성했으며 단 하나의 명령으로 몇 분 안에 기본 클러스터를 만듭니다.

이것은 EKS를 구성하기 위한 도구 이며,  AWS 관리콘솔에서 제공하는 EKS UI, CDK, Terraform, Rancher 등 다양한 도구로도 구성이 가능합니다.

## eksctl을 통한 EKS 구성

### 1.eksctl 설치

아래와 같이 eksctl을 설치하고 버전을 확인합니다.

```text
# eksctl 설
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
# eksctl 자동완성 - bash
. <(eksctl completion bash)
eksctl version

```

### 2.VPC/Subnet 정보 확인

앞서 [Cloudformation 구성](cloudformation.md#3-stack)에서 생성한 VPC id, Subnet id를 확인합니다.

다음 aws cli 명령을 통해서 확인 할 수 있습니다. 결과값은 홈디렉토리 **`vpc_subnet.txt`** 에 저장합니다. 아래 명령을 실행하면 자동으로 저장됩니다.

```text
cd ~/environment/
#VPC ID export
export vpc_ID=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values=eksworkshop | jq -r '.Vpcs[].VpcId')
echo $vpc_ID

#Subnet ID, CIDR, Subnet Name export
aws ec2 describe-subnets --filter Name=vpc-id,Values=$vpc_ID | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)'
echo $vpc_ID > vpc_subnet.txt
aws ec2 describe-subnets --filter Name=vpc-id,Values=$vpc_ID | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)' >> vpc_subnet.txt
cat vpc_subnet.txt

```

아래는 **`vpc_subnet.txt`** 에 저장된 예제입니다.

```text
vpc-04a7f563ebe750a92
subnet-0db014e6a52f7b002 10.11.16.0/20 eksworkshop-PublicSubnet02
subnet-01a18de77c71a9d8d 10.11.32.0/20 eksworkshop-PublicSubnet03
subnet-0396e3d1dfe08d224 10.11.0.0/20 eksworkshop-PublicSubnet01
subnet-0e49bdf1a4b2a0f6a 10.11.80.0/20 eksworkshop-PrivateSubnet02
subnet-0914eefaede7a14c9 10.11.64.0/20 eksworkshop-PrivateSubnet01
subnet-01db4b6773a94e6a2 10.11.96.0/20 eksworkshop-PrivateSubnet03

```

저장해둔 Region 정보와 master\_arn을 확인합니다. 앞서 [인증/자격증명 및 환경구성](../eks/env-auth.md#undefined-1) 에서 이미 **`master_arn.txt`** 파일로 저장해 두었습니다. 관련 파일을 확인합니다.

```text
#사용자 AWS Region
echo $AWS_REGION

#사용자 KMS Key ARM
echo $MASTER_ARN
cat master_arn.txt

```

VPC id, subnet id, region, master arn은 eksctl을 통해 EKS cluster를 배포하는 데 사용합니다.

### 3. eksctl 배포 yaml 수정

Cloud9 IDE 편집기에서 아래와 같이 수정합니다. 수정내용은 현재 생성된 VPC, Subnet ID , key 위치 입니다.

![](../.gitbook/assets/image%20%28164%29.png)

수정할 블록의 예시입니다.

```text
vpc: 
  id: vpc-04a7f563ebe750a92
  subnets:
    private:
      ap-northeast-2a: { id: subnet-0914eefaede7a14c9}
      ap-northeast-2b: { id: subnet-0e49bdf1a4b2a0f6a}
      ap-northeast-2c: { id: subnet-01db4b6773a94e6a2}
      ap-northeast-2d: { id: subnet-034369344c2f8e598}
    public:
      ap-northeast-2a: { id: subnet-0396e3d1dfe08d224}
      ap-northeast-2b: { id: subnet-0db014e6a52f7b002}
      ap-northeast-2c: { id: subnet-01a18de77c71a9d8d}
      ap-northeast-2d: { id: subnet-0786a534253c21a1e}

secretsEncryption:
  keyARN: arn:aws:kms:ap-northeast-2:584172017494:key/25a2f579-9f22-4d79-ad6f-1a468d06244b

nodeGroups:
  - name: ng-public-01
중략 
    ssh: 
        publicKeyPath: "/home/ec2-user/environment/eksworkshop.pub"
중략  
  - name: ng-private-01
중략  
    ssh: 
        publicKeyPath: "/home/ec2-user/environment/eksworkshop.pub"
```

{% hint style="warning" %}
**vpc/subnet id , KMS CMK keyARN 등이 다를 경우 설치 에러가 발생합니다. 또한 Cloud9의 publickeyPath의 경로도 확인하고, 반드시 다음 단계를 진행하기 전에 다시 한번 Review 합니다.**
{% endhint %}

### 4. cluster 생성

eksctl을 통해 EKS Cluster를 생성합니다. default는 1.18입니다. 하지만 git을 통해 다운 받은 eksctl 용 yaml파일에 1.19 설치가 선언되어 있습니다.

```text
eksctl create cluster --config-file=/home/ec2-user/environment/myeks/eksworkshop-cluster-3az.yaml
 
```

{% hint style="info" %}
Cluster를 생성하기 위해 20분 정도 시간이 소요됩니다. 
{% endhint %}

출력 결과 예시

```text
eksctl create cluster --config-file=/home/ec2-user/environment/myeks/eksworkshop-cluster-4az.yaml
2021-04-02 16:34:31 [ℹ]  eksctl version 0.43.0
2021-04-02 16:34:31 [ℹ]  using region ap-northeast-2
2021-04-02 16:34:31 [✔]  using existing VPC (vpc-0872d2a79110e4289) and subnets (private:map[ap-northeast-2a:{subnet-0074fd1438d35d4ef ap-northeast-2a 10.11.64.0/20} ap-northeast-2b:{subnet-0c82a7e8f40fa4268 ap-northeast-2b 10.11.80.0/20} ap-northeast-2c:{subnet-083b57160bcdc8320 ap-northeast-2c 10.11.96.0/20} ap-northeast-2d:{subnet-0693a31f5373e153b ap-northeast-2d 10.11.112.0/20}] public:map[ap-northeast-2a:{subnet-07b001f1937567fb7 ap-northeast-2a 10.11.0.0/20} ap-northeast-2b:{subnet-03811b74f76270aed ap-northeast-2b 10.11.16.0/20} ap-northeast-2c:{subnet-02ce8af3b04458b8f ap-northeast-2c 10.11.32.0/20} ap-northeast-2d:{subnet-0c23440ad2800c0ef ap-northeast-2d 10.11.48.0/20}])
2021-04-02 16:34:31 [!]  custom VPC/subnets will be used; if resulting cluster doesn't function as expected, make sure to review the configuration of VPC/subnets
2021-04-02 16:34:31 [ℹ]  nodegroup "ng-public-01" will use "ami-046c8f0c9b0f1ff98" [AmazonLinux2/1.19]
2021-04-02 16:34:31 [ℹ]  using SSH public key "/home/ec2-user/environment/eksworkshop.pub" as "eksctl-eksworkshop-nodegroup-ng-public-01-02:ff:18:ce:c0:1f:0f:60:77:af:2a:a4:5a:60:d4:88" 
2021-04-02 16:34:32 [ℹ]  nodegroup "ng-private-01" will use "ami-046c8f0c9b0f1ff98" [AmazonLinux2/1.19]
2021-04-02 16:34:32 [ℹ]  using SSH public key "/home/ec2-user/environment/eksworkshop.pub" as "eksctl-eksworkshop-nodegroup-ng-private-01-02:ff:18:ce:c0:1f:0f:60:77:af:2a:a4:5a:60:d4:88" 
2021-04-02 16:34:32 [ℹ]  using Kubernetes version 1.19
2021-04-02 16:34:32 [ℹ]  creating EKS cluster "eksworkshop" in "ap-northeast-2" region with un-managed nodes
2021-04-02 16:34:32 [ℹ]  2 nodegroups (ng-private-01, ng-public-01) were included (based on the include/exclude rules)
2021-04-02 16:34:32 [ℹ]  will create a CloudFormation stack for cluster itself and 2 nodegroup stack(s)
2021-04-02 16:34:32 [ℹ]  will create a CloudFormation stack for cluster itself and 0 managed nodegroup stack(s)
2021-04-02 16:34:32 [ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=ap-northeast-2 --cluster=eksworkshop'
2021-04-02 16:34:32 [ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "eksworkshop" in "ap-northeast-2"
2021-04-02 16:34:32 [ℹ]  2 sequential tasks: { create cluster control plane "eksworkshop", 3 sequential sub-tasks: { 2 sequential sub-tasks: { wait for control plane to become ready, update CloudWatch logging configuration }, create addons, 2 parallel sub-tasks: { create nodegroup "ng-public-01", create nodegroup "ng-private-01" } } }
2021-04-02 16:34:32 [ℹ]  building cluster stack "eksctl-eksworkshop-cluster"
2021-04-02 16:34:32 [ℹ]  deploying stack "eksctl-eksworkshop-cluster"
2021-04-02 16:35:02 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-cluster"
2021-04-02 16:45:34 [ℹ]  waiting for requested "LoggingUpdate" in cluster "eksworkshop" to succeed
2021-04-02 16:46:08 [✔]  configured CloudWatch logging for cluster "eksworkshop" in "ap-northeast-2" (enabled types: api, audit, authenticator, controllerManager, scheduler & no types disabled)
2021-04-02 16:46:09 [ℹ]  building nodegroup stack "eksctl-eksworkshop-nodegroup-ng-public-01"
2021-04-02 16:46:09 [ℹ]  building nodegroup stack "eksctl-eksworkshop-nodegroup-ng-private-01"
2021-04-02 16:46:09 [ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-ng-public-01"
2021-04-02 16:46:09 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-public-01"
2021-04-02 16:46:09 [ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-ng-private-01"
2021-04-02 16:46:09 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-private-01"
2021-04-02 16:46:26 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-private-01"
2021-04-02 16:50:50 [ℹ]  waiting for the control plane availability...
2021-04-02 16:50:50 [✔]  saved kubeconfig as "/home/ec2-user/.kube/config"
2021-04-02 16:50:50 [ℹ]  no tasks
2021-04-02 16:50:50 [✔]  all EKS cluster resources for "eksworkshop" have been created
2021-04-02 16:50:50 [ℹ]  adding identity "arn:aws:iam::584172017494:role/eksctl-eksworkshop-nodegroup-ng-p-NodeInstanceRole-JXTTVHOPAH3E" to auth ConfigMap
2021-04-02 16:50:50 [ℹ]  nodegroup "ng-public-01" has 0 node(s)
2021-04-02 16:50:50 [ℹ]  waiting for at least 4 node(s) to become ready in "ng-public-01"
2021-04-02 16:51:47 [ℹ]  nodegroup "ng-public-01" has 4 node(s)
2021-04-02 16:51:47 [ℹ]  node "ip-10-11-15-156.ap-northeast-2.compute.internal" is ready
2021-04-02 16:51:47 [ℹ]  node "ip-10-11-23-11.ap-northeast-2.compute.internal" is ready
2021-04-02 16:51:47 [ℹ]  node "ip-10-11-41-214.ap-northeast-2.compute.internal" is ready
2021-04-02 16:51:47 [ℹ]  node "ip-10-11-60-225.ap-northeast-2.compute.internal" is ready
2021-04-02 16:51:47 [ℹ]  adding identity "arn:aws:iam::584172017494:role/eksctl-eksworkshop-nodegroup-ng-p-NodeInstanceRole-TUY3DFENOYT2" to auth ConfigMap
2021-04-02 16:51:47 [ℹ]  nodegroup "ng-private-01" has 0 node(s)
2021-04-02 16:51:47 [ℹ]  waiting for at least 4 node(s) to become ready in "ng-private-01"
2021-04-02 16:53:08 [ℹ]  nodegroup "ng-private-01" has 4 node(s)
2021-04-02 16:53:08 [ℹ]  node "ip-10-11-105-9.ap-northeast-2.compute.internal" is ready
2021-04-02 16:53:08 [ℹ]  node "ip-10-11-120-87.ap-northeast-2.compute.internal" is ready
2021-04-02 16:53:08 [ℹ]  node "ip-10-11-66-117.ap-northeast-2.compute.internal" is ready
2021-04-02 16:53:08 [ℹ]  node "ip-10-11-85-11.ap-northeast-2.compute.internal" is ready
2021-04-02 16:53:09 [ℹ]  kubectl command should work with "/home/ec2-user/.kube/config", try 'kubectl get nodes'
2021-04-02 16:53:09 [✔]  EKS cluster "eksworkshop" in "ap-northeast-2" region is ready
```

### 5. Cluster 생성 확인

정상적으로 Cluster가 생성되었는지 확인합니다.

```text
kubectl get nodes

```

출력 결과 예시

```text
whchoi98:~/environment $ kubectl get nodes
NAME                                              STATUS   ROLES    AGE     VERSION
ip-10-11-105-9.ap-northeast-2.compute.internal    Ready    <none>   4m58s   v1.19.6-eks-49a6c0
ip-10-11-120-87.ap-northeast-2.compute.internal   Ready    <none>   5m4s    v1.19.6-eks-49a6c0
ip-10-11-15-156.ap-northeast-2.compute.internal   Ready    <none>   5m57s   v1.19.6-eks-49a6c0
ip-10-11-23-11.ap-northeast-2.compute.internal    Ready    <none>   5m55s   v1.19.6-eks-49a6c0
ip-10-11-41-214.ap-northeast-2.compute.internal   Ready    <none>   5m55s   v1.19.6-eks-49a6c0
ip-10-11-60-225.ap-northeast-2.compute.internal   Ready    <none>   5m55s   v1.19.6-eks-49a6c0
ip-10-11-66-117.ap-northeast-2.compute.internal   Ready    <none>   5m      v1.19.6-eks-49a6c0
ip-10-11-85-11.ap-northeast-2.compute.internal    Ready    <none>   5m4s    v1.19.6-eks-49a6c0
```

* 생성된 VPC와 Subnet, Internet Gateway, NAT Gateway, Route Table등을 확인해 봅니다.
* 생성된 EC2 Worker Node들도 확인해 봅니다.
* EKS와 eksctl을 통해 생생된 Cloudformation도 확인해 봅니다.

다음과 같은 구성도가 완성되었습니다.



### 6.eksctl yaml code 참조 \(option\)

eksctl 배포를 위한 EKS Cluster yaml 파일은 다음과 같습니다. 각자의 Cloud9 콘솔에서 파일을 확인해 봅니다.

```text
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksworkshop
  region: ap-northeast-2
  version: "1.19"

vpc: 
  id: vpc-0872d2a79110e4289
  subnets:
    public:
      ap-northeast-2a: { id: subnet-07b001f1937567fb7}
      ap-northeast-2b: { id: subnet-03811b74f76270aed}
      ap-northeast-2c: { id: subnet-02ce8af3b04458b8f}
    private:
      ap-northeast-2a: { id: subnet-0074fd1438d35d4ef}
      ap-northeast-2b: { id: subnet-0c82a7e8f40fa4268}
      ap-northeast-2c: { id: subnet-083b57160bcdc8320}

secretsEncryption:
  keyARN: arn:aws:kms:ap-northeast-2:584172017494:key/25a2f579-9f22-4d79-ad6f-1a468d06244b

nodeGroups:
  - name: ng-public-01
    instanceType: m5.xlarge
    desiredCapacity: 4
    minSize: 4
    maxSize: 8
    volumeSize: 100
    volumeType: gp3 
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

  - name: ng-private-01
    instanceType: m5.xlarge
    desiredCapacity: 4
    privateNetworking: true
    minSize: 4
    maxSize: 8
    volumeSize: 100
    volumeType: gp3 
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



