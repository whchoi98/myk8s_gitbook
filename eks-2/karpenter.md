---
description: 'Update: 2022-05-08'
---

# 스케쥴링-Karpenter

## Karpenter 소개&#x20;

![](<../.gitbook/assets/image (229).png>)

Karpenter는 Kubernetes 클러스터의 애플리케이션을 처리하는 데 적합한 컴퓨팅 리소스만 자동으로 시작하고 빠르고 간단한 컴퓨팅 프로비저닝으로 클라우드를 최대한 활용할 수 있도록 설계 되었습니다.&#x20;

Karpenter는 AWS로 구축된 유연한 오픈 소스의 고성능 Kubernetes 클러스터 오토스케일러입니다. 애플리케이션 로드의 변화에 대응하여 적절한 크기의 컴퓨팅 리소스를 신속하게 실행함으로써 애플리케이션 가용성과 클러스터 효율성을 개선할 수 있습니다. 또한 Karpenter는 애플리케이션의 요구 사항을 충족하는 컴퓨팅 리소스를 적시에 제공하며, 앞으로 클러스터의 컴퓨팅 리소스 공간을 자동으로 최적화하여 비용을 절감하고 성능을 개선 할수 있습니다.&#x20;

Karpenter 이전에는 Kubernetes 사용자가 [Amazon EC2 Auto Scaling 그룹](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html)과 [Kubernetes Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)를 사용하는 애플리케이션을 지원하기 위해 클러스터의 컴퓨팅 파워를 동적으로 조정해야 했습니다. EKS를 사용하는 많은 고객들이 Kubernetes Cluster Autoscaler를 사용하여 클러스터 Auto Scaling을 구성하기가 어렵고 구성할 수 있는 범위가 제한적인 것에 대해 개선을 요구했습니다&#x20;

Karpenter가 클러스터에 설치되면 Karpenter는 예약되지 않은 포드의 전체 리소스 요청을 관찰하고 새 노드를 시작하고 종료하는 결정을 내림으로써 예약 대기 시간과 인프라 비용을 줄입니다. 이를 위해 Karpenter는 Kubernetes 클러스터 내의 이벤트를 관찰한 다음 Amazon EC2와 같은 기본 클라우드 공급자의 컴퓨팅 서비스로 명령을 전송합니다.

![](<../.gitbook/assets/image (236).png>)

Karpenter는 [Apache License 2.0](https://github.com/awslabs/karpenter/blob/main/LICENSE)을 통해 라이선스가 부여되는 오픈 소스 프로젝트입니다. 모든 주요 클라우드 공급업체 및 온프레미스 환경을 포함하여, 모든 환경에서 실행되는 모든 Kubernetes 클러스터와 함께 작동하도록 설계되었습니다.&#x20;



## Karpenter 설치

### 1.환경 설정

Kubernetes Metric-server를 설치합니다. 앞서 설치하였으면 생략합니다.&#x20;

```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

```

metric server가 정상적으로 설치가 완료되면 아래와 같이 리소스 모니터링을 확인 할 수 있습니다. K9s에서도 Pod들의 CPU/Memory 사용량을 확인 할 수 있습니다.&#x20;

```
kubectl top pod --all-namespaces
```

Helm을 통해서 아래 kube-ops-view를 Cloud9에 설치합니다. 노드 배포를 확인하기 위해 kube-ops-view를 service type=LoadBalancer를 설치합니다.&#x20;

```
## 앞서 Lab에서 생성한 managed Node의 Public Subnet에 위치한 노드에 kube-ops-view를 설치합니다
kubectl create namespace kube-tools
helm install kube-ops-view \
stable/kube-ops-view \
--namespace kube-tools \
--set service.type=LoadBalancer \
--set nodeSelector.nodegroup-type=managed-frontend-workloads \
--version 1.2.4 \
--set rbac.create=True

## loadbalancer의 FQDN을 확인하고 웹브라우저에서 접속해 봅니다 
kubectl -n kube-tools get svc kube-ops-view | tail -n 1 | awk '{ print "Kube-ops-view URL = http://"$4 }'

```

URL을 접속하면 , 아래와 같이 노드와 배치된 PoD들을 확인해 볼 수 있습니다

![](<../.gitbook/assets/image (224).png>)

Karpenter 시험 환경 구성을 위해 아래와 같이 환경변수를 구성합니다.&#x20;

```
#앞서 ekscluster 이름을 구성하였다면 생략합니다.  
#export ekscluster_name=eksworkshop
#앞서 ACCOUNT_ID 를 환경변수에 등록하였다면 생략합니다. 
#export ACCOUNT_ID=$(aws sts get-caller-identity --region ap-northeast-2 --output text --query Account)
#새로운 노드 labeling을 위해 설정
export k_public_mgmd_node="k-managed-frontend-workloads"
export k_private_mgmd_node="k-managed-backend-workloads"
# EKS CLUSTER_ENDPOINT 값에 대한 환경변수 설정 
export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name ${ekscluster_name} --query "cluster.endpoint" --output text)"
# KARPENTER_VERSION 설정
export KARPENTER_VERSION="v0.9.1"
echo ${ekscluster_name}
echo ${ACCOUNT_ID}
echo ${k_public_mgmd_node}
echo ${k_private_mgmd_node}
echo ${CLUSTER_ENDPOINT}
echo ${KARPENTER_VERSION}
echo "export k_public_mgmd_node=${k_public_mgmd_node}" | tee -a ~/.bash_profile
echo "export k_private_mgmd_node=${k_private_mgmd_node}" | tee -a ~/.bash_profile
echo "export k_private_mgmd_node=${CLUSTER_ENDPOINT}" | tee -a ~/.bash_profile
```

### 2.Karpenter 시험 노드 설치

Karpenter 시험을 위한 새로운 노드 그룹 생성을 위한 yaml 파일을 생성합니다.&#x20;

```
cat << EOF > ~/environment/myeks/karpenter-nodegroup.yaml
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

managedNodeGroups:
  - name: k-managed-ng-public-01
    instanceType: ${instance_type}
    subnets:
      - PublicSubnet01
      - PublicSubnet02
      - PublicSubnet03
    desiredCapacity: 3
    minSize: 0
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
        
  - name: k-managed-ng-private-01
    instanceType: ${instance_type}
    subnets:
      - PrivateSubnet01
      - PrivateSubnet02
      - PrivateSubnet03
    desiredCapacity: 3
    privateNetworking: true
    minSize: 0
    maxSize: 6
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

Karpenter 시험을 위한 새로운 노드그룹을 eksctl 로 설치합니다.&#x20;

```
eksctl create nodegroup --config-file=/home/ec2-user/environment/myeks/karpenter-nodegroup.yaml

```

### 3. Karpenter 노드 권한 설정

kubernetes와 IAM간 인증을 위해 OIDC Provider를 생성합니다. 이미 앞서 Lab에서 생성하였다면 생략합니다.&#x20;

```
eksctl utils associate-iam-oidc-provider \
    --region ${AWS_REGION} \
    --cluster ${ekscluster_name} \
    --approve
    
```

Karpenter Node들을 위한 IAM Role을 생성합니다. karpenter node를 위한 IAM Role Template을 다운로드 합니다

```
mkdir /home/ec2-user/environment/karpenter
export KARPENTER_CF="/home/ec2-user/environment/karpenter/k-node-iam-role.yaml"
echo ${KARPENTER_CF}

curl -fsSL https://karpenter.sh/"${KARPENTER_VERSION}"/getting-started/getting-started-with-eksctl/cloudformation.yaml  > $KARPENTER_CF
sed -i 's/\${ClusterName}/eksworkshop/g' $KARPENTER_CF
#eksworkshop은 앞서 정의한 eks clustername 입니다. 다르게 설정한 경우 다른 값을 입력합니다 
```

AWS CLI를 통해서 IAM Role 구성을 위한 Cloudformation을 배포합니다.&#x20;

```
aws cloudformation deploy \
  --stack-name "Karpenter-${ekscluster_name}" \
  --template-file "${KARPENTER_CF}" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides ClusterName="${ekscluster_name}"
```

Karpenter Node들을 위해 생성된 IAM Role을 eksctl을 통해 kubernetes 권한에 Mapping 합니다

```
eksctl create iamidentitymapping \
  --username system:node:{{EC2PrivateDNSName}} \
  --cluster "${ekscluster_name}" \
  --arn "arn:aws:iam::${ACCOUNT_ID}:role/KarpenterNodeRole-${ekscluster_name}" \
  --group system:bootstrappers \
  --group system:nodes
  
```

Kube-system Configmap/aws-auth에 정상적으로 Mapping 되었는지 확인합니다.&#x20;

```
kubectl edit -n kube-system configmap/aws-auth 

```

아래와 같이 추가되었습니다

```
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::972012566617:role/KarpenterNodeRole-eksworkshop
      username: system:node:{{EC2PrivateDNSName}}
```

###

### 3.Provisioner 구성

