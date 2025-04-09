---
description: 'Updaet : 2025.04.09'
---

# Basic Node Pool

## 1. 기본 NodePool 생성

Karpenter가 실제로 어떻게 작동하는지 이해하기 위해 기본적인 프로비저닝을 수행해 보겠습니다. Karpenter를 사용하면 **스케줄되지 않은 Pod**을 처리하기 위해 자동으로 노드를 추가하고, 필요하지 않은 경우 이를 제거할 수 있습니다. 이 과정에서 **NodePool**과 **EC2NodeClass**가 중요한 역할을 합니다.

***

### 1.1 NodePool이란?

**NodePool**은 **Karpenter의 핵심 리소스(Custom Resource, CRD)**&#xB85C;, Karpenter의 동작을 설정하는 역할을 합니다.

#### ① NodePool의 역할

Karpenter는 아래와 같은 작업을 자동으로 수행합니다.

* **스케줄되지 않은 Pod을 감지하고 이를 위한 노드를 추가**
* **Pod을 새로운 노드에 스케줄링**
* **노드가 더 이상 필요하지 않을 경우 자동으로 제거**

이러한 동작을 제어하려면 **NodePool을 반드시 생성해야 하며, NodePool이 없으면 Karpenter는 비활성화된 상태**로 유지됩니다.

***

### 1.2 EC2NodeClass란?

**EC2NodeClass**는 **AWS EC2 관련 설정을 정의하는 Karpenter의 Custom Resource**입니다.\
각 **NodePool**은 반드시 하나의 `EC2NodeClass`를 참조해야 합니다.

* `spec.template.spec.nodeClassRef` 속성을 통해 NodePool과 연결됩니다.
* AWS 환경에서 EC2 관련 설정을 따로 관리할 수 있도록 도와줍니다.

🔗 **공식 문서 참고:**

* [NodePool 공식 문서](https://karpenter.sh/docs/concepts/nodepools/)
* [NodeClass 공식 문서](https://karpenter.sh/docs/concepts/nodeclasses/)

***

### 1.3 사전 요구 사항

Karpenter를 사용하기 전에 다음과 같은 환경이 준비되어 있어야 합니다.

#### ① 필수 구성 요소

* **Karpenter 설치 완료**
* **EKS 클러스터 및 IAM 역할 설정 완료**

이제 NodePool과 NodeClass를 배포하겠습니다.

***

### 1.4 NodePool 및 NodeClass 배포

이제 NodePool과 EC2NodeClass를 생성하여 Karpenter가 노드를 자동으로 관리할 수 있도록 설정합니다.

#### ① 주요 설정 설명

아래 `basic.yaml` 파일을 통해 NodePool과 EC2NodeClass를 정의합니다.

<table><thead><tr><th width="155.65234375">설정 항목</th><th>설명</th></tr></thead><tbody><tr><td><strong>consolidateAfter</strong></td><td>노드가 저활용 상태일 때 몇 초 후 제거할지 설정. (예제에서는 30초)</td></tr><tr><td><strong>requirements</strong></td><td>생성할 노드의 제약 조건. <br><code>instance-category</code>, <code>architecture</code>, <code>capacity-type</code> 등 설정 가능</td></tr><tr><td><strong>labels</strong></td><td>노드에 추가할 라벨. (Pod을 특정 노드에 스케줄링하는 데 사용)</td></tr><tr><td><strong>nodeClassRef</strong></td><td>AWS EC2 관련 설정을 참조하는 <code>EC2NodeClass</code> 연결</td></tr><tr><td><strong>limits</strong></td><td>클러스터 전체 리소스 제한. (예제에서는 <code>cpu: 10</code>)</td></tr></tbody></table>

***

### 1.5 NodePool 및 NodeClass 매니페스트 작성

아래 매니페스트를 사용하여 NodePool과 EC2NodeClass를 생성합니다.

```bash
cd ~/environment/karpenter
cat <<EoF> ~/environment/karpenter/basic.yaml
apiVersion: karpenter.sh/v1
kind: NodePool  # Karpenter의 NodePool 리소스를 정의합니다.
metadata:
  name: default  # NodePool의 이름을 지정합니다. 여러 개의 NodePool을 만들 경우 구분할 수 있도록 네이밍합니다.

spec:
  template:
    metadata:
       labels:
          eks-team: my-team  # 생성되는 모든 노드에 이 라벨이 추가됩니다.
          # 특정 Pod을 특정 NodePool에서만 실행되도록 nodeSelector를 사용할 때 활용됩니다.

    spec:
      nodeClassRef:
        group: karpenter.k8s.aws  # 사용하려는 NodeClass가 속한 API 그룹
        kind: EC2NodeClass  # AWS EC2와 연동되는 Karpenter의 Custom Resource
        name: default  # 사용할 NodeClass의 이름 (아래 정의한 EC2NodeClass의 이름과 일치해야 함)

      expireAfter: Never  # 노드의 만료 기간을 설정합니다. 'Never'로 설정하면 노드가 자동으로 제거되지 않습니다. 30초로 설정합니다.

      requirements:  # 생성할 노드의 조건을 설정합니다.
        - key: "karpenter.k8s.aws/instance-generation"  # 인스턴스의 세대 필터링
          operator: Gt  # Greater than (>) 연산자로 특정 세대 이상만 허용
          values: ["5"]  # 5세대보다 최신인 인스턴스 타입만 사용

        - key: karpenter.k8s.aws/instance-category  # 인스턴스 카테고리 필터링
          operator: In  # In 연산자로 특정 카테고리만 허용
          values: ["c", "m", "r"]  # "c" (Compute), "m" (General Purpose), "r" (Memory Optimized) 인스턴스만 허용

        - key: kubernetes.io/arch  # 아키텍처 필터링
          operator: In
          values: ["amd64"]  # 64비트 x86 아키텍처를 사용하는 노드만 허용

        - key: karpenter.sh/capacity-type  # EC2 용량 유형 지정
          operator: In
          values: ["on-demand"]  # 온디맨드(즉시 가용) 인스턴스만 사용하도록 설정

        - key: kubernetes.io/os  # 운영체제 필터링
          operator: In
          values: ["linux"]  # 리눅스 운영체제를 사용하는 노드만 허용

  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized  # 노드가 비어있거나 활용률이 낮을 경우 통합 정책 적용
    consolidateAfter: 30s  # 30초 동안 낮은 활용률을 보이면 노드를 축소 (제거) 

  limits:
    cpu: "10"  # 클러스터 전체에서 Karpenter가 관리하는 노드들의 CPU 총량 제한 (10 vCPU 이하로 설정)

---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass  # AWS EC2 노드와 관련된 설정을 정의하는 Custom Resource
metadata:
  name: default  # NodePool에서 참조할 EC2NodeClass의 이름

spec:
  amiSelectorTerms:
    - alias: al2023@latest  # 최신 Amazon Linux 2023 AMI를 사용하도록 설정

  role: karpenterNodeRole-${CLUSTER_NAME}  # Karpenter 노드가 사용할 IAM Role 지정
  # EC2 인스턴스가 필요로 하는 IAM 역할 (EKS와 통합되어야 함)

  securityGroupSelectorTerms:
  - tags:
      karpenter.sh/discovery: ${CLUSTER_NAME}  # 클러스터에서 사용 중인 Security Group을 자동으로 선택

  subnetSelectorTerms:
  - tags:
      karpenter.sh/discovery: ${CLUSTER_NAME}  # 클러스터의 Subnet을 자동으로 선택

  tags:
    intent: apps  # EC2 인스턴스에 추가할 태그 (운영 목적을 구분할 때 사용 가능)
EoF

```

* Node Pool의 이름은 default 이며, 여러개의 Node Pool을 생성해서 관리할 수 있습니다.
* Karpenter에 의해 생성된 노드에는 `eks-team: my-team` 라벨이 추가 됩니다.
* Node Pool에서 생성될 인스턴스 타입의 조건을 정의합니다.
  * 인스턴스 세대 : 5세대 인스턴스 이상
  * 인스턴스 타입 : c,m,r type 인스턴스
  * 인트턴스 아키텍쳐 : x86 아키텍쳐
  * 인스턴스 Capacity Type : on-demand
  * 인스턴스 운영체제 : Linux OS

### 1.6 NodePool 및 NodeClass 적용

아래 명령어를 실행하여 생성한 매니페스트를 적용합니다.

```bash
kubectl apply -f ~/environment/karpenter/basic.yaml
```

성공적으로 생성되면 다음과 같은 메시지가 출력됩니다.

```bash
nodepool.karpenter.sh/default created
ec2nodeclass.karpenter.k8s.aws/default created
```

***

### 1.7 Subnet과 Security Group 에 Tag 설정

다음은 **기존 EKS 클러스터, 서브넷 및 보안 그룹에 `karpenter.sh/discovery` 태그를 추가하는 방법**입니다.

***

#### **① EKS 클러스터 이름 변수 설정**

```sh
export CLUSTER_NAME=$(eksctl get clusters -o json | jq -r '.[0].Name')
echo ${CLUSTER_NAME}

```

현재 AWS 계정에서 첫 번째 EKS 클러스터의 이름을 가져와 `CLUSTER_NAME` 변수에 저장합니다.

클러스터에 설정된 모든 태그를 조회합니다.

```sh
aws eks list-tags-for-resource --resource-arn $(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.arn" --output text)
```

아래 명령을 통해 " karpenter.sh/discovery" tag가 있는지 확인합니다.

```
aws eks list-tags-for-resource --resource-arn $(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.arn" --output text) | jq -e '.tags["karpenter.sh/discovery"]' && echo "✅ 태그 존재" || echo "⚠️ 태그 없음"

```

#### **② Cluster에  `karpenter.sh/discovery` 태그 추가**

```sh
aws eks tag-resource \
    --resource-arn $(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.arn" --output text) \
    --tags karpenter.sh/discovery=${CLUSTER_NAME}
```

EKS 클러스터에 Karpenter가 인식할 `discovery` 태그 추가를 합니다.

&#x20;태그가 정상적으로 추가되었는지 확인합니다.

```sh
aws eks list-tags-for-resource \
    --resource-arn $(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.arn" --output text)
```

아래 명령을 통해 " karpenter.sh/discovery" tag가 있는지 확인합니다.

```
aws eks list-tags-for-resource --resource-arn $(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.arn" --output text) | jq -e '.tags["karpenter.sh/discovery"]' && echo "✅ 태그 존재" || echo "⚠️ 태그 없음"

```

#### **③ Subnet에  `karpenter.sh/discovery` 태그 추가**

EKS 클러스터가 배포된 서브넷의 ID를 가져와 변수에 저장합니다.

```sh
export PublicSubnet01=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=*Public*" --query "Subnets[0].SubnetId" --output text)
export PublicSubnet02=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=*Public*" --query "Subnets[1].SubnetId" --output text)
export PublicSubnet03=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=*Public*" --query "Subnets[2].SubnetId" --output text)

export PrivateSubnet01=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=*Private*" --query "Subnets[0].SubnetId" --output text)
export PrivateSubnet02=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=*Private*" --query "Subnets[1].SubnetId" --output text)
export PrivateSubnet03=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=*Private*" --query "Subnets[2].SubnetId" --output text)
```

Private 서브넷에 `karpenter.sh/discovery` 태그를 추가합니다.

```sh
aws ec2 create-tags --resources "$PrivateSubnet01" --tags Key="karpenter.sh/discovery",Value="${CLUSTER_NAME}"
aws ec2 create-tags --resources "$PrivateSubnet02" --tags Key="karpenter.sh/discovery",Value="${CLUSTER_NAME}"
aws ec2 create-tags --resources "$PrivateSubnet03" --tags Key="karpenter.sh/discovery",Value="${CLUSTER_NAME}"
```

Private 서브넷에 `karpenter.sh/discovery` 태그가 정상적으로 추가되었는지 확인합니다.

```sh
for id in $PublicSubnet01 $PublicSubnet02 $PublicSubnet03 $PrivateSubnet01 $PrivateSubnet02 $PrivateSubnet03; do TAG=$(aws ec2 describe-tags --filters "Name=resource-id,Values=$id" "Name=key,Values=karpenter.sh/discovery" --query "Tags[0].Value" --output text); [[ "$TAG" == "$CLUSTER_NAME" ]] && echo "✅ $id: 태그 존재 ($TAG)" || echo "❌ $id: 태그 없음"; done

```

#### **④ Security Group에  `karpenter.sh/discovery` 태그 추가**

클러스터가 사용하는 기본 보안 그룹의 ID를 가져와 변수에 저장합니다.

```sh
export SECURITY_GROUP_ID=$(aws eks describe-cluster \
    --name ${CLUSTER_NAME} \
    --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" \
    --output text)
```

**보안 그룹에 `karpenter.sh/discovery` 태그를 추가합니다.**

```sh
aws ec2 create-tags \
  --resources ${SECURITY_GROUP_ID} \
  --tags Key="karpenter.sh/discovery",Value="${CLUSTER_NAME}"
```

보안 그룹에 `karpenter.sh/discovery` 태그가 정상적으로 추가되었는지 확인합니다.

```sh
aws ec2 describe-tags --filters "Name=resource-id,Values=$SECURITY_GROUP_ID" "Name=key,Values=karpenter.sh/discovery" --query "Tags[0].Value" --output text | grep -q "$CLUSTER_NAME" && echo "✅ 태그 있음 ($SECURITY_GROUP_ID)" || echo "❌ 태그 없음 ($SECURITY_GROUP_ID)"

```

***

* **EKS 클러스터**, **서브넷**, **보안 그룹**에 `karpenter.sh/discovery` 태그를 추가하면 Karpenter가 노드를 자동으로 프로비저닝할 수 있습니다.
* 태그 적용 후 Karpenter의 로그를 확인하여 정상적으로 서브넷을 감지하는지 점검할 수 있습니다.

```sh
kubectl -n $KARPENTER_NAMESPACE logs deployment/karpenter --all-containers=true --since=10m | grep subnet
```

위 과정이 완료되면 Karpenter가 정상적으로 작동할 가능성이 높습니다. 🚀

### 1.8 요약

✅ **NodePool**을 생성하여 Karpenter가 클러스터 내에서 동작할 수 있도록 설정하였습니다.\
✅ **EC2NodeClass**를 사용하여 AWS EC2 관련 설정을 관리할 수 있도록 구성하였습니다.\
✅ 이제 Karpenter는 **스케줄되지 않은 Pod을 감지하고 자동으로 노드를 추가**할 것입니다.

***

이제 Karpenter의 기본적인 노드 프로비저닝이 완료되었습니다. 🚀
