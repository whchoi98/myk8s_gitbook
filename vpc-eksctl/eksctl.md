---
description: 'update : 2021-10-01 / 30min'
---

# eksctl 구성

## eksctl 소개

`eksctl`은 관리형 Kubernetes 서비스 인 EKS에서 클러스터를 생성하기위한 간단한 CLI 도구입니다. Go로 작성되었으며 CloudFormation을 사용하며 [Weaveworks](https://www.weave.works) 가 작성했으며 단 하나의 명령으로 몇 분 안에 기본 클러스터를 만듭니다.

이것은 EKS를 구성하기 위한 도구 이며,  AWS 관리콘솔에서 제공하는 EKS UI, CDK, Terraform, Rancher 등 다양한 도구로도 구성이 가능합니다.

## eksctl을 통한 EKS 구성

### 1.eksctl 설치

아래와 같이 eksctl을 Cloud9에 설치하고 버전을 확인합니다.

```
# eksctl 설정 
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
# eksctl 자동완성 - bash
. <(eksctl completion bash)
eksctl version

```

### 2.VPC/Subnet 정보 확인

앞서 [Cloudformation 구성](cloudformation.md#3-stack)에서 생성한 VPC id, Subnet id를 확인합니다.

다음 aws cli 명령을 통해서 확인 할 수 있습니다. 결과값은 홈디렉토리 **`vpc_subnet.txt`** 에 저장합니다. 아래 명령을 실행하면 자동으로 저장됩니다.

```
cd ~/environment/
#VPC ID export
export vpc_ID=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values=eksworkshop | jq -r '.Vpcs[].VpcId')
echo $vpc_ID

#Subnet ID, CIDR, Subnet Name export
aws ec2 describe-subnets --filter Name=vpc-id,Values=$vpc_ID | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)'
echo $vpc_ID > vpc_subnet.txt
aws ec2 describe-subnets --filter Name=vpc-id,Values=$vpc_ID | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)' >> vpc_subnet.txt
cat vpc_subnet.txt

export PublicSubnet01=$(aws ec2 describe-subnets --filter Name=vpc-id,Values=$vpc_ID | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)' | awk '/eksworkshop-PublicSubnet01/{print $1}')
export PublicSubnet02=$(aws ec2 describe-subnets --filter Name=vpc-id,Values=$vpc_ID | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)' | awk '/eksworkshop-PublicSubnet02/{print $1}')
export PublicSubnet03=$(aws ec2 describe-subnets --filter Name=vpc-id,Values=$vpc_ID | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)' | awk '/eksworkshop-PublicSubnet03/{print $1}')
export PrivateSubnet01=$(aws ec2 describe-subnets --filter Name=vpc-id,Values=$vpc_ID | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)' | awk '/eksworkshop-PrivateSubnet01/{print $1}')
export PrivateSubnet02=$(aws ec2 describe-subnets --filter Name=vpc-id,Values=$vpc_ID | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)' | awk '/eksworkshop-PrivateSubnet02/{print $1}')
export PrivateSubnet03=$(aws ec2 describe-subnets --filter Name=vpc-id,Values=$vpc_ID | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)' | awk '/eksworkshop-PrivateSubnet03/{print $1}')
echo "export PublicSubnet01=${PublicSubnet01}" | tee -a ~/.bash_profile
echo "export PublicSubnet02=${PublicSubnet02}" | tee -a ~/.bash_profile
echo "export PublicSubnet03=${PublicSubnet03}" | tee -a ~/.bash_profile
echo "export PrivateSubnet01=${PrivateSubnet01}" | tee -a ~/.bash_profile
echo "export PrivateSubnet02=${PrivateSubnet02}" | tee -a ~/.bash_profile
echo "export PrivateSubnet03=${PrivateSubnet03}" | tee -a ~/.bash_profile
```

아래는 **`vpc_subnet.txt`** 에 저장된 예제입니다.

```
vpc-04a7f563ebe750a92
subnet-0db014e6a52f7b002 10.11.16.0/20 eksworkshop-PublicSubnet02
subnet-01a18de77c71a9d8d 10.11.32.0/20 eksworkshop-PublicSubnet03
subnet-0396e3d1dfe08d224 10.11.0.0/20 eksworkshop-PublicSubnet01
subnet-0e49bdf1a4b2a0f6a 10.11.80.0/20 eksworkshop-PrivateSubnet02
subnet-0914eefaede7a14c9 10.11.64.0/20 eksworkshop-PrivateSubnet01
subnet-01db4b6773a94e6a2 10.11.96.0/20 eksworkshop-PrivateSubnet03

```

저장해둔 Region 정보와 master\_arn을 확인합니다. 앞서 [인증/자격증명 및 환경구성](../eks/env-auth.md#undefined-1) 에서 이미 **`master_arn.txt`** 파일로 저장해 두었습니다. 관련 파일을 확인합니다.

```
#사용자 AWS Region
echo $AWS_REGION

#사용자 KMS Key ARM
echo $MASTER_ARN
cat master_arn.txt

```

VPC id, subnet id, region, master arn은 eksctl을 통해 EKS cluster를 배포하는 데 사용합니다.

### 3. eksctl 배포 yaml 수정

아래 파일을 생성합니다. 해당 파일의 예제는 앞서 복제한 git의 eksworkshop-cluster-3az.yaml과 동일합니다.

{% hint style="warning" %}
**vpc/subnet id , KMS CMK keyARN 등이 다를 경우 설치 에러가 발생합니다. 또한 Cloud9의 publickeyPath의 경로도 확인하고, 반드시 다음 단계를 진행하기 전에 다시 한번 Review 합니다.**
{% endhint %}

### 4. cluster 생성

eksctl을 통해 EKS Cluster를 생성하기 위해서, 아래와 같이 Shell 변수에 값을 입력하고, 정상적으로 값이 출력되는 지 확인합니다.&#x20;

```
export ekscluster_name="eksworkshop"
export eks_version="1.20"
export instance_type="m5.xlarge"
export public_selfmgmd_node="frontend-workloads"
export private_selfmgmd_node="backend-workloads"
export public_mgmd_node="managed-frontend-workloads"
export private_mgmd_node="managed-backend-workloads"
export publicKeyPath="/home/ec2-user/environment/eksworkshop.pub"

echo ${ekscluster_name}
echo ${AWS_REGION}
echo ${eks_version}
echo ${PublicSubnet01}
echo ${PublicSubnet02}
echo ${PublicSubnet03}
echo ${PrivateSubnet01}
echo ${PrivateSubnet02}
echo ${PrivateSubnet03}
echo ${MASTER_ARN}
echo ${instance_type}
echo ${public_selfmgmd_node}
echo ${private_selfmgmd_node}
echo ${public_mgmd_node}
echo ${private_mgmd_node}
echo ${publicKeyPath}

```

Shell Profile에 등록합니다. &#x20;

```
echo "export ekscluster_name=${ekscluster_name}" | tee -a ~/.bash_profile
echo "export eks_version=${eks_version}" | tee -a ~/.bash_profile
echo "export instance_type=${instance_type}" | tee -a ~/.bash_profile
echo "export public_selfmgmd_node=${public_selfmgmd_node}" | tee -a ~/.bash_profile
echo "export private_selfmgmd_node=${private_selfmgmd_node}" | tee -a ~/.bash_profile
echo "export public_mgmd_node=${public_mgmd_node}" | tee -a ~/.bash_profile
echo "export private_mgmd_node=${private_mgmd_node}" | tee -a ~/.bash_profile
echo "export publicKeyPath=${publicKeyPath}" | tee -a ~/.bash_profile

```

eksctl을 통해 EKS Cluster를 생성합니다.&#x20;

```
cat << EOF > ~/environment/myeks/eksworkshop.yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: ${ekscluster_name}
  region: ${AWS_REGION}
  version: "${eks_version}"  

vpc: 
  id: ${vpc_ID}
  subnets:
    public:
      PublicSubnet01:
        id: ${PublicSubnet01}
      PublicSubnet02:
        id: ${PublicSubnet02}
      PublicSubnet03:
        id: ${PublicSubnet03}
    private:
      PrivateSubnet01:
        id: ${PrivateSubnet01}
      PrivateSubnet02:
        id: ${PrivateSubnet02}
      PrivateSubnet03:
        id: ${PrivateSubnet03}
secretsEncryption:
  keyARN: ${MASTER_ARN}

nodeGroups:
  - name: ng-public-01
    instanceType: ${instance_type}
    subnets:
      - PublicSubnet01
      - PublicSubnet02
      - PublicSubnet03
    desiredCapacity: 3
    minSize: 3
    maxSize: 6
    volumeSize: 200
    volumeType: gp3 
    amiFamily: AmazonLinux2
    labels:
      nodegroup-type: "${public_selfmgmd_node}"
    ssh: 
        publicKeyPath: "${publicKeyPath}"
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
    instanceType: ${instance_type}
    subnets:
      - PrivateSubnet01
      - PrivateSubnet02
      - PrivateSubnet03
    desiredCapacity: 3
    privateNetworking: true
    minSize: 3
    maxSize: 9
    volumeSize: 200
    volumeType: gp3 
    amiFamily: AmazonLinux2
    labels:
      nodegroup-type: "${private_selfmgmd_node}"
    ssh: 
        publicKeyPath: "${publicKeyPath}"
        allow: true
    iam:
      attachPolicyARNs:
      withAddonPolicies:
        autoScaler: true
        cloudWatch: true
        ebs: true
        fsx: true
        efs: true

managedNodeGroups:
  - name: managed-ng-public-01
    instanceType: ${instance_type}
    subnets:
      - PublicSubnet01
      - PublicSubnet02
      - PublicSubnet03
    desiredCapacity: 3
    minSize: 3
    maxSize: 6
    volumeSize: 200
    volumeType: gp3 
    amiFamily: AmazonLinux2
    labels:
      nodegroup-type: "${public_mgmd_node}"
    ssh: 
        publicKeyPath: "${publicKeyPath}"
        allow: true
    iam:
      attachPolicyARNs:
      withAddonPolicies:
        autoScaler: true
        cloudWatch: true
        ebs: true
        fsx: true
        efs: true
        
  - name: managed-ng-private-01
    instanceType: ${instance_type}
    subnets:
      - PrivateSubnet01
      - PrivateSubnet02
      - PrivateSubnet03
    desiredCapacity: 3
    privateNetworking: true
    minSize: 3
    maxSize: 9
    volumeSize: 200
    volumeType: gp3 
    amiFamily: AmazonLinux2
    labels:
      nodegroup-type: "${private_mgmd_node}"
    ssh: 
        publicKeyPath: "${publicKeyPath}"
        allow: true
    iam:
      attachPolicyARNs:
      withAddonPolicies:
        autoScaler: true
        cloudWatch: true
        ebs: true
        fsx: true
        efs: true
        
cloudWatch:
    clusterLogging:
        enableTypes: ["api", "audit", "authenticator", "controllerManager", "scheduler"]
EOF
      
```

```
eksctl create cluster --config-file=/home/ec2-user/environment/myeks/eksworkshop.yaml
 
```

{% hint style="info" %}
Cluster를 생성하기 위해 20분 정도 시간이 소요됩니다.&#x20;
{% endhint %}

출력 결과 예시

```
whchoi:~/environment $ eksctl create cluster --config-file=/home/ec2-user/environment/myeks/eksworkshop-cluster-3az.yaml
2021-10-05 13:09:15 [ℹ]  eksctl version 0.68.0
2021-10-05 13:09:15 [ℹ]  using region ap-northeast-2
2021-10-05 13:09:16 [✔]  using existing VPC (vpc-0ec670bcdaf2efa2d) and subnets (private:map[PrivateSubnet01:{subnet-0c6430b44d5c98211 ap-northeast-2a 10.11.64.0/20} PrivateSubnet02:{subnet-0f1a181077dcfb1cb ap-northeast-2b 10.11.80.0/20} PrivateSubnet03:{subnet-061a967ec3d9c4d8e ap-northeast-2c 10.11.96.0/20}] public:map[PublicSubnet01:{subnet-0a225c208a7c4d1d6 ap-northeast-2a 10.11.0.0/20} PublicSubnet02:{subnet-005309b113283f38a ap-northeast-2b 10.11.16.0/20} PublicSubnet03:{subnet-034a4e2a275b9b3ad ap-northeast-2c 10.11.32.0/20}])
2021-10-05 13:09:16 [!]  custom VPC/subnets will be used; if resulting cluster doesn't function as expected, make sure to review the configuration of VPC/subnets
2021-10-05 13:09:16 [ℹ]  nodegroup "ng-public-01" will use "ami-086d30b42d304dfc3" [AmazonLinux2/1.20]
2021-10-05 13:09:16 [ℹ]  using SSH public key "/home/ec2-user/environment/eksworkshop.pub" as "eksctl-eksworkshop-nodegroup-ng-public-01-44:b6:1f:23:20:27:bf:99:aa:65:c0:34:0f:2a:17:fe" 
2021-10-05 13:09:16 [ℹ]  nodegroup "ng-private-01" will use "ami-086d30b42d304dfc3" [AmazonLinux2/1.20]
2021-10-05 13:09:16 [ℹ]  using SSH public key "/home/ec2-user/environment/eksworkshop.pub" as "eksctl-eksworkshop-nodegroup-ng-private-01-44:b6:1f:23:20:27:bf:99:aa:65:c0:34:0f:2a:17:fe" 
2021-10-05 13:09:16 [ℹ]  nodegroup "managed-ng-public-01" will use "" [AmazonLinux2/1.20]
2021-10-05 13:09:16 [ℹ]  using SSH public key "/home/ec2-user/environment/eksworkshop.pub" as "eksctl-eksworkshop-nodegroup-managed-ng-public-01-44:b6:1f:23:20:27:bf:99:aa:65:c0:34:0f:2a:17:fe" 
2021-10-05 13:09:16 [ℹ]  nodegroup "managed-ng-private-01" will use "" [AmazonLinux2/1.20]
2021-10-05 13:09:16 [ℹ]  using SSH public key "/home/ec2-user/environment/eksworkshop.pub" as "eksctl-eksworkshop-nodegroup-managed-ng-private-01-44:b6:1f:23:20:27:bf:99:aa:65:c0:34:0f:2a:17:fe" 
2021-10-05 13:09:16 [ℹ]  using Kubernetes version 1.20
2021-10-05 13:09:16 [ℹ]  creating EKS cluster "eksworkshop" in "ap-northeast-2" region with managed nodes and un-managed nodes
2021-10-05 13:09:16 [ℹ]  4 nodegroups (managed-ng-private-01, managed-ng-public-01, ng-private-01, ng-public-01) were included (based on the include/exclude rules)
2021-10-05 13:09:16 [ℹ]  will create a CloudFormation stack for cluster itself and 2 nodegroup stack(s)
2021-10-05 13:09:16 [ℹ]  will create a CloudFormation stack for cluster itself and 2 managed nodegroup stack(s)
2021-10-05 13:09:16 [ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=ap-northeast-2 --cluster=eksworkshop'
2021-10-05 13:09:16 [ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "eksworkshop" in "ap-northeast-2"
2021-10-05 13:09:16 [ℹ]  2 sequential tasks: { create cluster control plane "eksworkshop", 3 sequential sub-tasks: { 2 sequential sub-tasks: { wait for control plane to become ready, update CloudWatch logging configuration }, 1 task: { create addons }, 4 parallel sub-tasks: { create nodegroup "ng-public-01", create nodegroup "ng-private-01", create managed nodegroup "managed-ng-public-01", create managed nodegroup "managed-ng-private-01" } } }
2021-10-05 13:09:16 [ℹ]  building cluster stack "eksctl-eksworkshop-cluster"
2021-10-05 13:09:17 [ℹ]  deploying stack "eksctl-eksworkshop-cluster"
2021-10-05 13:09:47 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-cluster"
2021-10-05 13:24:20 [ℹ]  waiting for requested "LoggingUpdate" in cluster "eksworkshop" to succeed
2021-10-05 13:24:37 [✔]  configured CloudWatch logging for cluster "eksworkshop" in "ap-northeast-2" (enabled types: api, audit, authenticator, controllerManager, scheduler & no types disabled)
2021-10-05 13:26:37 [ℹ]  building managed nodegroup stack "eksctl-eksworkshop-nodegroup-managed-ng-private-01"
2021-10-05 13:26:37 [ℹ]  building nodegroup stack "eksctl-eksworkshop-nodegroup-ng-public-01"
2021-10-05 13:26:37 [ℹ]  building nodegroup stack "eksctl-eksworkshop-nodegroup-ng-private-01"
2021-10-05 13:26:37 [ℹ]  building managed nodegroup stack "eksctl-eksworkshop-nodegroup-managed-ng-public-01"
2021-10-05 13:26:38 [ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-managed-ng-public-01"
2021-10-05 13:26:38 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-managed-ng-public-01"
2021-10-05 13:26:38 [ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-managed-ng-private-01"
2021-10-05 13:26:38 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-managed-ng-private-01"
2021-10-05 13:26:38 [ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-ng-public-01"
2021-10-05 13:26:38 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-public-01"
2021-10-05 13:26:38 [ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-ng-private-01"
2021-10-05 13:26:38 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-private-01"
2021-10-05 13:26:53 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-public-01"
2021-10-05 13:26:56 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-managed-ng-private-01"
2021-10-05 13:26:57 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-managed-ng-public-01"
2021-10-05 13:32:46 [ℹ]  nodegroup "ng-private-01" has 3 node(s)
2021-10-05 13:32:46 [ℹ]  node "ip-10-11-78-241.ap-northeast-2.compute.internal" is ready
2021-10-05 13:32:46 [ℹ]  node "ip-10-11-95-61.ap-northeast-2.compute.internal" is ready
2021-10-05 13:32:46 [ℹ]  node "ip-10-11-99-248.ap-northeast-2.compute.internal" is ready
2021-10-05 13:32:46 [ℹ]  nodegroup "managed-ng-public-01" has 3 node(s)
2021-10-05 13:32:46 [ℹ]  node "ip-10-11-13-217.ap-northeast-2.compute.internal" is ready
2021-10-05 13:32:46 [ℹ]  node "ip-10-11-16-169.ap-northeast-2.compute.internal" is ready
2021-10-05 13:32:46 [ℹ]  node "ip-10-11-45-25.ap-northeast-2.compute.internal" is ready
2021-10-05 13:32:46 [ℹ]  waiting for at least 3 node(s) to become ready in "managed-ng-public-01"
2021-10-05 13:32:46 [ℹ]  nodegroup "managed-ng-public-01" has 3 node(s)
2021-10-05 13:32:46 [ℹ]  node "ip-10-11-13-217.ap-northeast-2.compute.internal" is ready
2021-10-05 13:32:46 [ℹ]  node "ip-10-11-16-169.ap-northeast-2.compute.internal" is ready
2021-10-05 13:32:46 [ℹ]  node "ip-10-11-45-25.ap-northeast-2.compute.internal" is ready
2021-10-05 13:32:46 [ℹ]  nodegroup "managed-ng-private-01" has 3 node(s)
2021-10-05 13:32:46 [ℹ]  node "ip-10-11-104-129.ap-northeast-2.compute.internal" is ready
2021-10-05 13:32:46 [ℹ]  node "ip-10-11-72-242.ap-northeast-2.compute.internal" is ready
2021-10-05 13:32:46 [ℹ]  node "ip-10-11-81-97.ap-northeast-2.compute.internal" is ready
2021-10-05 13:32:46 [ℹ]  waiting for at least 3 node(s) to become ready in "managed-ng-private-01"
2021-10-05 13:32:46 [ℹ]  nodegroup "managed-ng-private-01" has 3 node(s)
2021-10-05 13:32:46 [ℹ]  node "ip-10-11-104-129.ap-northeast-2.compute.internal" is ready
2021-10-05 13:32:46 [ℹ]  node "ip-10-11-72-242.ap-northeast-2.compute.internal" is ready
2021-10-05 13:32:46 [ℹ]  node "ip-10-11-81-97.ap-northeast-2.compute.internal" is ready
2021-10-05 13:34:48 [ℹ]  kubectl command should work with "/home/ec2-user/.kube/config", try 'kubectl get nodes'
2021-10-05 13:34:48 [✔]  EKS cluster "eksworkshop" in "ap-northeast-2" region is ready
```

### 5. Cluster 생성 확인

정상적으로 Cluster가 생성되었는지 확인합니다.

```
kubectl get nodes

```

출력 결과 예시

```
whchoi:~/environment $ kubectl get nodes
NAME                                               STATUS   ROLES    AGE   VERSION
ip-10-11-104-129.ap-northeast-2.compute.internal   Ready    <none>   28m   v1.20.7-eks-135321
ip-10-11-13-217.ap-northeast-2.compute.internal    Ready    <none>   28m   v1.20.7-eks-135321
ip-10-11-16-169.ap-northeast-2.compute.internal    Ready    <none>   28m   v1.20.7-eks-135321
ip-10-11-17-190.ap-northeast-2.compute.internal    Ready    <none>   27m   v1.20.7-eks-135321
ip-10-11-45-25.ap-northeast-2.compute.internal     Ready    <none>   28m   v1.20.7-eks-135321
ip-10-11-46-97.ap-northeast-2.compute.internal     Ready    <none>   27m   v1.20.7-eks-135321
ip-10-11-7-117.ap-northeast-2.compute.internal     Ready    <none>   27m   v1.20.7-eks-135321
ip-10-11-72-242.ap-northeast-2.compute.internal    Ready    <none>   28m   v1.20.7-eks-135321
ip-10-11-78-241.ap-northeast-2.compute.internal    Ready    <none>   26m   v1.20.7-eks-135321
ip-10-11-81-97.ap-northeast-2.compute.internal     Ready    <none>   28m   v1.20.7-eks-135321
ip-10-11-95-61.ap-northeast-2.compute.internal     Ready    <none>   26m   v1.20.7-eks-135321
ip-10-11-99-248.ap-northeast-2.compute.internal    Ready    <none>   26m   v1.20.7-eks-135321
```

* 생성된 VPC와 Subnet, Internet Gateway, NAT Gateway, Route Table등을 확인해 봅니다.
* 생성된 EC2 Worker Node들도 확인해 봅니다.
* EKS와 eksctl을 통해 생생된 Cloudformation도 확인해 봅니다.

다음과 같은 구성도가 완성되었습니다.

![](<../.gitbook/assets/image (221) (1) (1) (1) (1) (1).png>)

### 6.eksctl yaml code 참조 (option)

eksctl 배포를 위한 EKS Cluster yaml 파일은 다음과 같습니다. 각자의 Cloud9 콘솔에서 파일을 확인해 봅니다.

```
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksworkshop
  region: ap-northeast-2
  version: "1.20"  

vpc: 
  id: vpc-0cac82bae42538fd7
  subnets:
    public:
      PublicSubnet01:
        id: subnet-0cd13d352f0960e86
      PublicSubnet02:
        id: subnet-065f091e585fea002
      PublicSubnet03:
        id: subnet-0d767589be30c617f
    private:
      PrivateSubnet01:
        id: subnet-056e4159357931f83
      PrivateSubnet02:
        id: subnet-02bb9d3ae75d5c333
      PrivateSubnet03:
        id: subnet-006b575333b696f4b
secretsEncryption:
  keyARN: arn:aws:kms:ap-northeast-2:794454221194:key/e9d049ae-38f1-4f38-b084-02ce197b0894

nodeGroups:
  - name: ng-public-01
    instanceType: m5.xlarge
    subnets:
      - PublicSubnet01
      - PublicSubnet02
      - PublicSubnet03
    desiredCapacity: 3
    minSize: 3
    maxSize: 6
    volumeSize: 200
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
    subnets:
      - PrivateSubnet01
      - PrivateSubnet02
      - PrivateSubnet03
    desiredCapacity: 3
    privateNetworking: true
    minSize: 3
    maxSize: 9
    volumeSize: 200
    volumeType: gp3 
    amiFamily: AmazonLinux2
    labels:
      nodegroup-type: "backend-workloads"
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

managedNodeGroups:
  - name: managed-ng-public-01
    instanceType: m5.xlarge
    subnets:
      - PublicSubnet01
      - PublicSubnet02
      - PublicSubnet03
    desiredCapacity: 3
    minSize: 3
    maxSize: 6
    volumeSize: 200
    volumeType: gp3 
    amiFamily: AmazonLinux2
    labels:
      nodegroup-type: "managed-frontend-workloads"
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
        
  - name: managed-ng-private-01
    instanceType: m5.xlarge
    subnets:
      - PrivateSubnet01
      - PrivateSubnet02
      - PrivateSubnet03
    desiredCapacity: 3
    privateNetworking: true
    minSize: 3
    maxSize: 9
    volumeSize: 200
    volumeType: gp3 
    amiFamily: AmazonLinux2
    labels:
      nodegroup-type: "managed-backend-workloads"
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
        
cloudWatch:
    clusterLogging:
        enableTypes: ["api", "audit", "authenticator", "controllerManager", "scheduler"]

```
