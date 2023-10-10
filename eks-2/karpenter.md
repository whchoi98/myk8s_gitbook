---
description: 'Update: 2023-01-27'
---

# 스케쥴링-Karpenter

## Karpenter 소개&#x20;

![](<../.gitbook/assets/image (465).png>)

Karpenter는 Kubernetes 클러스터의 애플리케이션을 처리하는 데 적합한 컴퓨팅 리소스만 자동으로 시작하고 빠르고 간단한 컴퓨팅 프로비저닝으로 클라우드를 최대한 활용할 수 있도록 설계 되었습니다.&#x20;

Karpenter는 AWS로 구축된 유연한 오픈 소스의 고성능 Kubernetes 클러스터 오토스케일러입니다. 애플리케이션 로드의 변화에 대응하여 적절한 크기의 컴퓨팅 리소스를 신속하게 실행함으로써 애플리케이션 가용성과 클러스터 효율성을 개선할 수 있습니다. 또한 Karpenter는 애플리케이션의 요구 사항을 충족하는 컴퓨팅 리소스를 적시에 제공하며, 앞으로 클러스터의 컴퓨팅 리소스 공간을 자동으로 최적화하여 비용을 절감하고 성능을 개선 할수 있습니다.&#x20;

Karpenter 이전에는 Kubernetes 사용자가 [Amazon EC2 Auto Scaling 그룹](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html)과 [Kubernetes Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)를 사용하는 애플리케이션을 지원하기 위해 클러스터의 컴퓨팅 파워를 동적으로 조정해야 했습니다. EKS를 사용하는 많은 고객들이 Kubernetes Cluster Autoscaler를 사용하여 클러스터 Auto Scaling을 구성하기가 어렵고 구성할 수 있는 범위가 제한적인 것에 대해 개선을 요구했습니다&#x20;

Karpenter가 클러스터에 설치되면 Karpenter는 예약되지 않은 포드의 전체 리소스 요청을 관찰하고 새 노드를 시작하고 종료하는 결정을 내림으로써 예약 대기 시간과 인프라 비용을 줄입니다. 이를 위해 Karpenter는 Kubernetes 클러스터 내의 이벤트를 관찰한 다음 Amazon EC2와 같은 기본 클라우드 공급자의 컴퓨팅 서비스로 명령을 전송합니다.

![](<../.gitbook/assets/image (473).png>)

Karpenter는 [Apache License 2.0](https://github.com/awslabs/karpenter/blob/main/LICENSE)을 통해 라이선스가 부여되는 오픈 소스 프로젝트입니다. 모든 주요 클라우드 공급업체 및 온프레미스 환경을 포함하여, 모든 환경에서 실행되는 모든 Kubernetes 클러스터와 함께 작동하도록 설계되었습니다.&#x20;

상세한 내용은 아래 URL을 참조하기 바랍니다.

```
https://karpenter.sh/
```

## Karpenter 설치

### 1.환경설정 및 VPC 구성

ap-northeast-1 (도쿄) region에 새로운 Cluster를 설치하기 위한 환경변수를 설정합니다.

```
echo "export KARPENTER_ACCOUNT_ID=${ACCOUNT_ID}" | tee -a ~/.bash_profile
export KARPENTER_AWS_REGION=ap-northeast-1
echo "export KARPENTER_AWS_REGION=$KARPENTER_AWS_REGION" | tee -a ~/.bash_profile

```

KMS 를 ap-northeast-1  도쿄리전에서 사용할 수 있도록 설정합니다.

```
# kms 를 생성
aws kms create-alias --alias-name alias/k-eksworkshop --target-key-id --region ap-northeast-1 $(aws kms create-key --query KeyMetadata.Arn --region ap-northeast-1 --output text)

# kms 값을 환경변수에 저장합니다.
export K_MASTER_ARN=$(aws kms describe-key --key-id alias/k-eksworkshop --query KeyMetadata.Arn --region ap-northeast-1 --output text)
echo "export K_MASTER_ARN=${K_MASTER_ARN}" | tee -a ~/.bash_profile
echo $K_MASTER_ARN

```

ap-northeast-1 에서 사용할 VPC를 구성합니다.

```
## 앞서 myeks reop를 다운 받았습니다.
cd ~/environment/myeks/
aws cloudformation deploy \
  --region ap-northeast-1 \
  --stack-name "eksworkshop" \
  --template-file "karpenter_vpc.yml" \
  --capabilities CAPABILITY_NAMED_IAM 
  
```

아래와 같이 Karpenter 설치를 위한 환경변수들을 추가로 설정합니다.

```
### make env for the karpenter test      

export k_ekscluster_name=k-eksworkshop
export k_public_mgmd_node="frontend"
export k_private_mgmd_node="backend"
export KARPENTER_VERSION="v0.27.5"
echo ${k_ekscluster_name}
echo ${k_public_mgmd_node}
echo ${k_private_mgmd_node}
echo ${KARPENTER_VERSION}
echo "export k_ekscluster_name=${k_ekscluster_name}" | tee -a ~/.bash_profile
echo "export k_public_mgmd_node=${k_public_mgmd_node}" | tee -a ~/.bash_profile
echo "export k_private_mgmd_node=${k_private_mgmd_node}" | tee -a ~/.bash_profile
#echo "export eks_version=${eks_version}" | tee -a ~/.bash_profile
echo "export KARPENTER_VERSION=${KARPENTER_VERSION}" | tee -a ~/.bash_profile
source ~/.bash_profile

```

eksctl을 사용해서 새로운 Cluster를 생성하기 위해, 앞서 구성한 VPC들의 주요 정보들을 환경 변수에 저장합니다.&#x20;

```
### VPC 정보 
cd ~/environment/
#VPC ID export
export k_vpc_ID=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values=eksworkshop --region ap-northeast-1| jq -r '.Vpcs[].VpcId')
echo $k_vpc_ID

#Subnet ID, CIDR, Subnet Name export
aws ec2 describe-subnets --filter Name=vpc-id,Values=$k_vpc_ID --region ap-northeast-1 | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)'
aws ec2 describe-subnets --filter Name=vpc-id,Values=$k_vpc_ID | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)' >> k_vpc_subnet.txt

# VPC, Subnet ID 환경변수 저장 
export k_PublicSubnet01=$(aws ec2 describe-subnets --filter Name=vpc-id,Values=$k_vpc_ID --region ap-northeast-1 | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)' | awk '/eksworkshop-PublicSubnet01/{print $1}')
export k_PublicSubnet02=$(aws ec2 describe-subnets --filter Name=vpc-id,Values=$k_vpc_ID --region ap-northeast-1 | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)' | awk '/eksworkshop-PublicSubnet02/{print $1}')
export k_PublicSubnet03=$(aws ec2 describe-subnets --filter Name=vpc-id,Values=$k_vpc_ID --region ap-northeast-1 | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)' | awk '/eksworkshop-PublicSubnet03/{print $1}')
export k_PrivateSubnet01=$(aws ec2 describe-subnets --filter Name=vpc-id,Values=$k_vpc_ID --region ap-northeast-1 | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)' | awk '/eksworkshop-PrivateSubnet01/{print $1}')
export k_PrivateSubnet02=$(aws ec2 describe-subnets --filter Name=vpc-id,Values=$k_vpc_ID --region ap-northeast-1 | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)' | awk '/eksworkshop-PrivateSubnet02/{print $1}')
export k_PrivateSubnet03=$(aws ec2 describe-subnets --filter Name=vpc-id,Values=$k_vpc_ID --region ap-northeast-1 | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)' | awk '/eksworkshop-PrivateSubnet03/{print $1}')
echo "export k_vpc_ID=${k_vpc_ID}" | tee -a ~/.bash_profile
echo "export k_PublicSubnet01=${k_PublicSubnet01}" | tee -a ~/.bash_profile
echo "export k_PublicSubnet02=${k_PublicSubnet02}" | tee -a ~/.bash_profile
echo "export k_PublicSubnet03=${k_PublicSubnet03}" | tee -a ~/.bash_profile
echo "export k_PrivateSubnet01=${k_PrivateSubnet01}" | tee -a ~/.bash_profile
echo "export k_PrivateSubnet02=${k_PrivateSubnet02}" | tee -a ~/.bash_profile
echo "export k_PrivateSubnet03=${k_PrivateSubnet03}" | tee -a ~/.bash_profile
#echo "export publicKeyPath=${publicKeyPath}" | tee -a ~/.bash_profile
source ~/.bash_profile

```

### 2.Cluster 구성

새로운 Cluster 구성을 위해 yaml 파일을 생성합니다.

```
### create cluster yaml file for the karpenter      

cat << EOF > ~/environment/myeks/karpenter_cluster.yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: ${k_ekscluster_name}
  region: ${KARPENTER_AWS_REGION}
  version: "${eks_version}"  
  tags:
    karpenter.sh/discovery: ${k_ekscluster_name}

vpc: 
  id: ${k_vpc_ID}
  subnets:
    public:
      k_PublicSubnet01:
        az: ${KARPENTER_AWS_REGION}a
        id: ${k_PublicSubnet01}
      k_PublicSubnet02:
        az: ${KARPENTER_AWS_REGION}c
        id: ${k_PublicSubnet02}
      k_PublicSubnet03:
        az: ${KARPENTER_AWS_REGION}d
        id: ${k_PublicSubnet03}
    private:
      k_PrivateSubnet01:
        az: ${KARPENTER_AWS_REGION}a
        id: ${k_PrivateSubnet01}
      k_PrivateSubnet02:
        az: ${KARPENTER_AWS_REGION}c
        id: ${k_PrivateSubnet02}
      k_PrivateSubnet03:
        az: ${KARPENTER_AWS_REGION}d
        id: ${k_PrivateSubnet03}
secretsEncryption:
  keyARN: ${K_MASTER_ARN}

managedNodeGroups:
  - name: public
    instanceType: ${instance_type}
    subnets:
      - ${k_PublicSubnet01}
      - ${k_PublicSubnet02}
      - ${k_PublicSubnet03}
    desiredCapacity: 3
    minSize: 3
    maxSize: 6
    volumeSize: 200
    volumeType: gp3 
    amiFamily: AmazonLinux2
    labels:
      nodegroup-type: "${k_public_mgmd_node}"
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
        
  - name: private
    instanceType: ${instance_type}
    subnets:
      - ${k_PrivateSubnet01}
      - ${k_PrivateSubnet02}
      - ${k_PrivateSubnet03}
    desiredCapacity: 3
    privateNetworking: true
    minSize: 3
    maxSize: 9
    volumeSize: 200
    volumeType: gp3 
    amiFamily: AmazonLinux2
    labels:
      nodegroup-type: "${k_private_mgmd_node}"
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

EOF


```

생성된 clsuter yaml파일이 정상적으로 구성되었는지 dry-run을 통해 확인해 봅니다.

```
eksctl create cluster --config-file=/home/ec2-user/environment/myeks/karpenter_cluster.yaml --dry-run

```

아래와 같이 명령을 실행시켜 eks cluster를 생성합니다.&#x20;

```
eksctl create cluster --config-file=/home/ec2-user/environment/myeks/karpenter_cluster.yaml

```



### 3.Karpenter 구성을 위한 환경 구성

Karpenter 시험 환경 구성을 위해 아래와 같이 환경변수를 구성합니다.&#x20;

```
### Karpenter Cluster Endpoint 변수 설정
export K_CLUSTER_ENDPOINT="$(aws eks describe-cluster --name ${k_ekscluster_name} --query "cluster.endpoint" --region ap-northeast-1 --output text)"
echo "export K_CLUSTER_ENDPOINT=${K_CLUSTER_ENDPOINT}" | tee -a ~/.bash_profile
source ~/.bash_profile

```

Subnet에 karpenter 환경을 위한 새로운 Tag를 설정합니다. Tag가 설정된 Subnet에 배포될 것입니다.

```
aws ec2 create-tags --resources "$k_PublicSubnet01" --tags Key="karpenter.sh/discovery",Value="${k_ekscluster_name}" --region ap-northeast-1
aws ec2 create-tags --resources "$k_PublicSubnet02" --tags Key="karpenter.sh/discovery",Value="${k_ekscluster_name}" --region ap-northeast-1
aws ec2 create-tags --resources "$k_PublicSubnet03" --tags Key="karpenter.sh/discovery",Value="${k_ekscluster_name}" --region ap-northeast-1

```

### 4. Karpenter 노드 권한 설정

kubernetes와 IAM간 인증을 위해 OIDC Provider를 생성합니다. &#x20;

```
eksctl utils associate-iam-oidc-provider \
    --region ${KARPENTER_AWS_REGION} \
    --cluster ${k_ekscluster_name} \
    --approve
    
```

Karpenter Node들을 위한 IAM Role을 생성합니다. karpenter node를 위한 IAM Role Template을 다운로드 합니다.&#x20;

```
## Karpenter Node에 Role을 적용하기 위한 Template 다운로드를 합니다.
mkdir /home/ec2-user/environment/karpenter
export KARPENTER_CF="/home/ec2-user/environment/karpenter/k-node-iam-role.yaml"
echo ${KARPENTER_CF}
curl -fsSL https://karpenter.sh/"${KARPENTER_VERSION}"/getting-started/getting-started-with-karpenter/cloudformation.yaml  > $KARPENTER_CF
## sed -i 's/\${ClusterName}/k-eksworkshop/g' $KARPENTER_CF

## 구성한 Node Role Template을 생성합니다.
aws cloudformation deploy \
  --region ${KARPENTER_AWS_REGION} \
  --stack-name "Karpenter-${k_ekscluster_name}" \
  --template-file "${KARPENTER_CF}" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides "ClusterName=${k_ekscluster_name}"
  
```

Karpenter Node들을 위해 생성된 IAM Role을 eksctl을 통해 kubernetes 권한에 Mapping 합니다.&#x20;

```
eksctl create iamidentitymapping \
  --region ${KARPENTER_AWS_REGION} \
  --username system:node:{{EC2PrivateDNSName}} \
  --cluster ${k_ekscluster_name} \
  --arn "arn:aws:iam::${ACCOUNT_ID}:role/KarpenterNodeRole-${k_ekscluster_name}" \
  --group system:bootstrappers \
  --group system:nodes

```

Kube-system Configmap/aws-auth에 정상적으로 Mapping 되었는지 확인합니다.&#x20;

{% hint style="info" %}
이미 서울리전에 Cluster가 1개 생성되어 있습니다. Cluster간 명령과 구성 등을 편리하게 하기 위해서 아래 kubectl ctx Plugin을 설치합니다. Cluster간 이동을 편리하게 합니다.
{% endhint %}

kube krew를 설치하고, 아래와 같은 Plugin을 구성합니다.

```
##Kube Krew 설치
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)

##kube krew 경로 설정
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
source ~/.bashrc

##kube ctx 설치
kubectl krew install ctx

```

아래와 같이 _**`kubect ctx`**_ 명령을 통해서 도쿄리전에 설치된 Cluster로 이동합니다.

```
kubectl ctx
# kubectl ctx {xxxx@k-eksworkshop.ap-northeast-1.eksctl.io}
```

도쿄리전에 구성된 kube 인증을 확인해 봅니다.&#x20;

```
kubectl describe -n kube-system configmap/aws-auth 

```

아래와 같이 추가되었습니다.&#x20;

```
Name:         aws-auth
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>

Data
====
mapRoles:
----
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:aws:iam::224149737230:role/eksctl-k-eksworkshop-nodegroup-pu-NodeInstanceRole-TRO569QNFBC1
  username: system:node:{{EC2PrivateDNSName}}
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:aws:iam::224149737230:role/eksctl-k-eksworkshop-nodegroup-pr-NodeInstanceRole-JDAJCTGD59TB
  username: system:node:{{EC2PrivateDNSName}}
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:aws:iam::224149737230:role/KarpenterNodeRole-k-eksworkshop
  username: system:node:{{EC2PrivateDNSName}}

mapUsers:
----
[]


BinaryData
====


```

### 5. Service Account 생성 (ISRA)

eksctl로 Kubernetes Service Account를 생성하고, 앞서 생성한 IAM Role을  Mapping 합니다.&#x20;

```
### Service Account를 위한 IAM Role을 매핑합니다.

eksctl create iamserviceaccount \
  --region ${KARPENTER_AWS_REGION} \
  --cluster "${k_ekscluster_name}" --name karpenter --namespace karpenter \
  --role-name "${k_ekscluster_name}-karpenter" \
  --attach-policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/KarpenterControllerPolicy-${k_ekscluster_name}" \
  --role-only \
  --override-existing-serviceaccounts \
  --approve

# KARPENTER IAM ROLE ARN을 변수에 저장해 둡니다. 
export KARPENTER_IAM_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/${k_ekscluster_name}-karpenter"
echo ${KARPENTER_IAM_ROLE_ARN}
echo "export export KARPENTER_IAM_ROLE_ARN=${KARPENTER_IAM_ROLE_ARN}" | tee -a ~/.bash_profile

```

### 6. Karpenter Pod설치

Helm을 사용하여 Karpenter Pod를 클러스터에 배포합니다.&#x20;

Helm Chart 를 설치하기 전에 Repo를 Helm에 추가해야 하므로 다음 명령을 실행하여 Repo를 추가합니다.

```
cd ~/environment
curl -L https://git.io/get_helm.sh | bash -s -- --version v3.8.2

helm repo add karpenter https://charts.karpenter.sh/
helm repo update

```

Cluster의 상세 정보 및 Karpenter Role ARN을 전달하는  Helm Chart를 설치합니다.&#x20;

```
docker logout public.ecr.aws
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter --version ${KARPENTER_VERSION} --namespace karpenter --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=${KARPENTER_IAM_ROLE_ARN} \
  --set settings.aws.clusterName=${k_ekscluster_name} \
  --set settings.aws.clusterEndpoint=${K_CLUSTER_ENDPOINT} \
  --set settings.aws.defaultInstanceProfile=KarpenterNodeInstanceProfile-${k_ekscluster_name} \
  --set settings.aws.interruptionQueueName=${k_ekscluster_name} \
  --wait

```

karpenter pod가 정상적으로 설치 되었는지 확인합니다.&#x20;

```
kubectl get pods --namespace karpenter
kubectl get deployment -n karpenter

```

Karpenter Pod는 Controller와 Webhook을 담당하는 컨테이너가 배치되어 있습니다.

### 6.Provisioner 구성

Karpenter 구성은 Provisioner CRD(Custom Resource Definition) 형식으로 제공됩니다. 단일 Karpenter Provisioner는 다양한 Pod를 구성할 수 있습니다. Karpenter는 Label 및 Affinity와 같은 Pod의 속성을 기반으로 Scheduling 및 프로비저닝 결정을 할 수 있습니다. Karpenter는 다양한 노드 그룹을 관리할 필요가 없습니다.

아래 명령을 사용하여 기본 프로비저닝 도구를 만들기 위한 yaml을 정의합니다.이 프로비저닝 도구는 securityGroupSelector 및 subnetSelector를 사용하여 노드를 시작하는 데 사용되는 리소스를 검색합니다. 위의 eksctl 명령에 karpenter.sh/discovery 태그를 적용했습니다.&#x20;

아래와 같이 Spot 인스턴스를 사용하는 Provisioner를 먼저 생성해 봅니다.

```
cat << EOF > ~/environment/karpenter/karpenter-provisioner1.yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: provisioner1
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot"]
  limits:
    resources:
      cpu: 1000
  provider:
    subnetSelector:
      karpenter.sh/discovery: ${k_ekscluster_name}
    securityGroupSelector:
      karpenter.sh/discovery: ${k_ekscluster_name}
  ttlSecondsAfterEmpty: 30
EOF

## 생성된 provisioner.yaml을 실행합니다. 
kubectl apply -f ~/environment/karpenter/karpenter-provisioner1.yaml

```

* **`instanceProfile`**: Karpenter에서 시작한 인스턴스는 컨테이너를 실행하고 네트워킹을 구성하는 데 필요한 권한을 부여하는 InstanceProfile로 실행해야 합니다.&#x20;
* **`requirements`** : Provisioner CRD는 인스턴스 type 및 AZ와 같은 노드 속성을 선택할 수 있습니다. 예를 들어 topology.kubernetes.io/zone=ap-northeast-2a 레이블에 대한 응답으로 Karpenter는 해당 가용성 영역에 노드를 프로비저닝합니다. 이 예에서는 karpenter.sh/capacity-type을 설정하여 EC2 스팟 인스턴스를 사용합니다. 여기에서 사용할 수 있는 다른 속성을 확인할 수 있습니다. 이번 랩에서 몇 가지 더 작업할 것입니다.&#x20;
* **`limit`** : provisioner는 클러스터에 할당된 CPU 및 메모리 수의 제한을 정의할 수 있습니다. ttlSecondsAfterEmpty: 값은 빈 노드를 종료하도록 Karpenter를 구성합니다.&#x20;
* **`provider:tags`** : EC2 인스턴스가 생성될 때 가지게 되는 Tag를 정의할 수도 있습니다. 이것은 EC2 수준에서 Billing 및 거버넌스를 활성화하는 데 도움이 됩니다.
* **`ttlSecondsAfterEmpty`** : 값은 Karpenter가 노드에 자원이 배치가 없는 경우 종료하도록 구성합니다.값을 정의하지 않은 상태로 두면 이 동작을 비활성화할 수 있습니다. 이 경우 빠른 시연을 위해 30초 값으로 설정했습니다.

### 7. 자동 노드 프로비저닝 1

Spot을 구동하기 위해 아래와 같이 EC2 Spot Service에 대한 설정을 합니다.

```
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true

```

Deployment 하기 전에 웹브라우저에서 앞서 생성한 kube-ops-view를 열어두고 배포를 살펴 봅니다.&#x20;

아래 kube-ops-view를 먼저 설치합니다.

```
kubectl create namespace kube-tools
helm repo add christianknell https://christianknell.github.io/helm-charts
helm repo update
helm install my-release christianknell/kube-ops-view \
--namespace kube-tools \
--set service.type=LoadBalancer \
--set nodeSelector.nodegroup-type=${k_public_mgmd_node} \
--set rbac.create=True \
--set service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-scheme"="internet-facing"


```

kube-ops-view 의 FQDN LB 주소를 확인하고 접속해 봅니다.&#x20;

```
kubectl -n kube-tools get svc my-release-kube-ops-view  | tail -n 1 | awk '{ print "my-release-kube-ops-view URL = http://"$4 }'
 
```

아래와 같은 현재 Node와 Pod의 구성 배치도를 확인 할 수 있습니다.&#x20;

<figure><img src="../.gitbook/assets/image (117).png" alt=""><figcaption></figcaption></figure>

Karpenter는 이제 활성화되었으며 노드 프로비저닝을 시작할 준비가 되었습니다. Deployment를 사용하여 Pod를 만들고 Karpenter가 노드를 프로비저닝하는 것을 확인해 봅니다.자동 노드 프로비저닝 이 배포는 [pause image](https://www.ianlewis.org/en/almighty-pause-container)를 사용하고 replica가  없는 상태에서 시작합니다.

```
kubectl create namespace karpenter-inflate
cat << EOF > ~/environment/karpenter/karpenter-inflate1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate1
  namespace: karpenter-inflate  
spec:
  replicas: 0
  selector:
    matchLabels:
      app: inflate1
  template:
    metadata:
      labels:
        app: inflate1
    spec:
      terminationGracePeriodSeconds: 0
      containers:
        - name: inflate1
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.2
          resources:
            requests:
              cpu: 1
      nodeSelector:
        karpenter.sh/capacity-type: spot
EOF
## 생성된 karpenter-inflate.yaml을 실행합니다. 
kubectl apply -f ~/environment/karpenter/karpenter-inflate1.yaml

```

replica를 늘려가면서 시험해 봅니다.

```
kubectl -n karpenter-inflate scale deployment inflate1 --replicas 5

```

Terminal 에서 로그를 확인해 봅니다.

```
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller

```

아래에서 처럼 새로운 Spot Instance가 할당되는 것을 확인할 수 있습니다.

```
2023-01-29T06:08:37.738Z        DEBUG   controller.provisioner  discovered subnets      {"commit": "5a7faa0-dirty", "subnets": ["subnet-092f608fed124f61a (ap-northeast-1c)", "subnet-0869716c80d96d66b (ap-northeast-1a)"]}
2023-01-29T06:08:37.858Z        DEBUG   controller.provisioner  discovered EC2 instance types zonal offerings for subnets       {"commit": "5a7faa0-dirty", "subnet-selector": "{\"karpenter.sh/discovery\":\"k-eksworkshop\"}"}
2023-01-29T06:08:38.048Z        INFO    controller.provisioner  found provisionable pod(s)      {"commit": "5a7faa0-dirty", "pods": 5}
2023-01-29T06:08:38.048Z        INFO    controller.provisioner  computed new node(s) to fit pod(s)      {"commit": "5a7faa0-dirty", "newNodes": 1, "pods": 5}
2023-01-29T06:08:38.048Z        INFO    controller.provisioner  launching node with 5 pods requesting {"cpu":"5125m","pods":"7"} from types c5d.metal, r6id.16xlarge, r5d.8xlarge, c5.24xlarge, m5dn.8xlarge and 196 other(s)   {"commit": "5a7faa0-dirty", "provisioner": "default"}
2023-01-29T06:08:38.236Z        DEBUG   controller.provisioner.cloudprovider    discovered security groups      {"commit": "5a7faa0-dirty", "provisioner": "default", "security-groups": ["sg-0ca6d78e6e9c402f9", "sg-0e58a7ca488f9b652", "sg-04976538139e6ff95", "sg-030cbd50908fc2e77"]}
2023-01-29T06:08:38.240Z        DEBUG   controller.provisioner.cloudprovider    discovered kubernetes version   {"commit": "5a7faa0-dirty", "provisioner": "default", "kubernetes-version": "1.22"}
2023-01-29T06:08:38.311Z        DEBUG   controller.provisioner.cloudprovider    discovered new ami      {"commit": "5a7faa0-dirty", "provisioner": "default", "ami": "ami-0db66a825cfe82f8f", "query": "/aws/service/eks/optimized-ami/1.22/amazon-linux-2/recommended/image_id"}
2023-01-29T06:08:38.477Z        DEBUG   controller.provisioner.cloudprovider    created launch template {"commit": "5a7faa0-dirty", "provisioner": "default", "launch-template-name": "Karpenter-k-eksworkshop-8022396668081538733", "launch-template-id": "lt-081ac3b8a0f85f3af"}
2023-01-29T06:08:41.110Z        INFO    controller.provisioner.cloudprovider    launched new instance   {"commit": "5a7faa0-dirty", "provisioner": "default", "id": "i-0790ae0c76c905f84", "hostname": "ip-10-21-4-182.ap-northeast-1.compute.internal", "instance-type": "c5.2xlarge", "zone": "ap-northeast-1a", "capacity-type": "spot"}
```

kube-ops-view 에서도 신규 노드가 할당된 것을 확인 할 수 있습니다.

<figure><img src="../.gitbook/assets/image (118).png" alt=""><figcaption></figcaption></figure>

### 8. 자동 노드 프로비저닝 2

Karpenter Provisioner CRD를 새로운 형태로 만들어 봅니다.

이 구성은 특정 인스턴스 타입을 Taint와 Toleration 등을 조합하여, 적용해 보는 예제입니다.

* **인스턴스 타입 : C5.xlarge**
* **Zone : ap-northeast-1a**
* **인스턴스 Capa: On Demand**

```
cat << EOF > ~/environment/karpenter/karpenter-provisioner2.yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: provisioner2
spec:
  taints:
    - key: cpuIntensive
      value: "true"
      effect: NoSchedule
  labels:
    phase: test2
    nodeType: cpu-node
  requirements:
    - key: "node.kubernetes.io/instance-type"
      operator: In
      values: ["c5.xlarge"]
    - key: "topology.kubernetes.io/zone"
      operator: In
      values: ["ap-northeast-1a"]
    - key: "karpenter.sh/capacity-type"
      operator: In
      values: ["on-demand"]
  limits:
    resources:
      cpu: 1000
  provider:
    subnetSelector:
      karpenter.sh/discovery: ${k_ekscluster_name}
    securityGroupSelector:
      karpenter.sh/discovery: ${k_ekscluster_name}
  ttlSecondsAfterEmpty: 30
EOF

## 생성된 provisioner.yaml을 실행합니다. 
kubectl apply -f ~/environment/karpenter/karpenter-provisioner2.yaml

```

새로운 Deployment Yaml을 배포합니다.

```
cat << EOF > ~/environment/karpenter/karpenter-inflate2.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate2
  namespace: karpenter-inflate  
spec:
  replicas: 0
  selector:
    matchLabels:
      app: inflate2
  template:
    metadata:
      labels:
        app: inflate2
    spec:
      terminationGracePeriodSeconds: 0
      containers:
        - name: inflate1
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.2
          resources:
            requests:
              cpu: 1
      nodeSelector:
        nodeType: cpu-node
      tolerations:
      - effect: "NoSchedule"
        key: "cpuIntensive"
        operator: "Equal"
        value: "true"
EOF
## 생성된 karpenter-inflate2.yaml을 실행합니다. 
kubectl apply -f ~/environment/karpenter/karpenter-inflate2.yaml

```

아래와 같이 5개 Pod를 배포하고, ap-northeast-1a Zone에 C5.xlarge 인스턴스가 배치 되는 지 확인해 봅니다.&#x20;

```
kubectl -n karpenter-inflate scale deployment inflate2 --replicas 5

```

아래 로그에서 확인이 가능합니다.&#x20;

```
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller

```





