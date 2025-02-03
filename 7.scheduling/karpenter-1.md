---
description: Update 2025-01-30
---

# 스케쥴링-Karpenter (작업중)

## Karpenter 소개

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

Karpenter는 Kubernetes 클러스터의 애플리케이션을 처리하는 데 적합한 컴퓨팅 리소스만 자동으로 시작하고 빠르고 간단한 컴퓨팅 프로비저닝으로 클라우드를 최대한 활용할 수 있도록 설계 되었습니다.&#x20;

Karpenter는 AWS로 구축된 유연한 오픈 소스의 고성능 Kubernetes 클러스터 오토스케일러입니다. 애플리케이션 로드의 변화에 대응하여 적절한 크기의 컴퓨팅 리소스를 신속하게 실행함으로써 애플리케이션 가용성과 클러스터 효율성을 개선할 수 있습니다. 또한 Karpenter는 애플리케이션의 요구 사항을 충족하는 컴퓨팅 리소스를 적시에 제공하며, 앞으로 클러스터의 컴퓨팅 리소스 공간을 자동으로 최적화하여 비용을 절감하고 성능을 개선 할수 있습니다.&#x20;

Karpenter 이전에는 Kubernetes 사용자가 [Amazon EC2 Auto Scaling 그룹](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html)과 [Kubernetes Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)를 사용하는 애플리케이션을 지원하기 위해 클러스터의 컴퓨팅 파워를 동적으로 조정해야 했습니다. EKS를 사용하는 많은 고객들이 Kubernetes Cluster Autoscaler를 사용하여 클러스터 Auto Scaling을 구성하기가 어렵고 구성할 수 있는 범위가 제한적인 것에 대해 개선을 요구했습니다&#x20;

Karpenter가 클러스터에 설치되면 Karpenter는 예약되지 않은 포드의 전체 리소스 요청을 관찰하고 새 노드를 시작하고 종료하는 결정을 내림으로써 예약 대기 시간과 인프라 비용을 줄입니다. 이를 위해 Karpenter는 Kubernetes 클러스터 내의 이벤트를 관찰한 다음 Amazon EC2와 같은 기본 클라우드 공급자의 컴퓨팅 서비스로 명령을 전송합니다.

<figure><img src="../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

Karpenter는 [Apache License 2.0](https://github.com/awslabs/karpenter/blob/main/LICENSE)을 통해 라이선스가 부여되는 오픈 소스 프로젝트입니다. 모든 주요 클라우드 공급업체 및 온프레미스 환경을 포함하여, 모든 환경에서 실행되는 모든 Kubernetes 클러스터와 함께 작동하도록 설계되었습니다.&#x20;

상세한 내용은 아래 URL을 참조하기 바랍니다.

```
https://karpenter.sh/
```

***

다음은 `${CLUSTER_NAME}`을 `${EKSCLUSTER_NAME}`으로 변경한 Karpenter 설치 가이드입니다.

***

## **기존 EKS 클러스터에 Karpenter 설치 가이드**

이 가이드는 **기존 Amazon EKS 클러스터에 Karpenter를 설치하고 설정하는 방법**을 상세하게 설명합니다.\
Karpenter는 Kubernetes에서 Auto Scaling을 담당하는 오픈소스 도구로, 필요한 시점에 적절한 크기의 **EC2 인스턴스를 자동으로 프로비저닝**하여 **운영 비용을 절감**하고 **리소스 활용도를 최적화**합니다. 🚀

***

### **1. Karpenter 설치를 위한 환경 변수 설정**

**설치 과정에서 사용할 변수들을 설정하여 운영을 편리하게 합니다.**

* Karpenter는 특정 IAM 역할과 정책을 필요로 하므로, IAM 관련 변수를 미리 설정합니다.
* Kubernetes 및 EKS API와 통신하기 위해 클러스터의 **API 엔드포인트**와 **OIDC 엔드포인트**를 가져옵니다.

#### ✅ **환경 변수 설정**

```sh
export KARPENTER_NAMESPACE="karpenter"           # Karpenter가 배포될 Kubernetes 네임스페이스
export KARPENTER_VERSION="1.1.3"                 # 설치할 Karpenter 버전
export AWS_PARTITION="aws"                        # AWS 파티션 정보 (일반적으로 "aws")
export AWS_REGION="ap-northeast-2"               # EKS 클러스터가 위치한 AWS 리전
export EKSCLUSTER_NAME="eksworkshop"             # EKS 클러스터 이름
export EKS_VERSION="1.29"                        # EKS 클러스터 버전
export KARPENTER_NODE_ROLE="KarpenterNodeRole-${EKSCLUSTER_NAME}"  # Karpenter Node IAM Role 이름
```

***

#### ✅ **EKS 클러스터의 API 엔드포인트 및 OIDC 설정**

```sh
# EKS API 서버 엔드포인트 가져오기
export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name ${EKSCLUSTER_NAME} --query "cluster.endpoint" --region ${AWS_REGION} --output text)"
echo "export CLUSTER_ENDPOINT=${CLUSTER_ENDPOINT}" | tee -a ~/.bash_profile
source ~/.bash_profile
```

> 🔹 **CLUSTER\_ENDPOINT**: Kubernetes API 서버와 통신하기 위한 엔드포인트입니다.

```sh
# EKS OIDC 엔드포인트 가져오기
export OIDC_ENDPOINT="$(aws eks describe-cluster --name ${EKSCLUSTER_NAME} --query "cluster.identity.oidc.issuer" --output text)"
```

> 🔹 **OIDC\_ENDPOINT**: AWS IAM과 연동하기 위한 OIDC(OpenID Connect) 엔드포인트입니다.

***

#### ✅ **IAM Role 및 정책 변수 설정**

```sh
export KARPENTER_CONTROLLER_ROLE="${EKSCLUSTER_NAME}-karpenter"   # Karpenter 컨트롤러용 IAM Role
export KarpenterNodeRole="KarpenterNodeRole-${EKSCLUSTER_NAME}"   # Karpenter에서 생성한 노드용 IAM Role
export KarpenterControllerPolicy="KarpenterControllerPolicy-${EKSCLUSTER_NAME}" # Karpenter 컨트롤러 IAM Policy
```

> 🔹 IAM 역할(Role)과 정책(Policy)을 미리 정의하여 Karpenter가 AWS 리소스에 접근할 수 있도록 준비합니다.

***

### **2. IAM 정책 및 역할 생성 (CloudFormation 활용)**

Karpenter는 EC2 인스턴스를 생성하고 관리해야 하므로, **IAM 정책과 역할**을 생성해야 합니다.\
AWS에서는 **CloudFormation**을 사용하여 이를 자동화할 수 있습니다.

#### ✅ **CloudFormation을 사용해 IAM Role 및 정책을 생성**

```sh
mkdir -p ~/environment/karpenter
export KARPENTER_CF="$HOME/environment/karpenter/k-node-iam-role.yaml"

# CloudFormation 템플릿 다운로드
curl -fsSL "https://raw.githubusercontent.com/aws/karpenter-provider-aws/v${KARPENTER_VERSION}/website/content/en/preview/getting-started/getting-started-with-karpenter/cloudformation.yaml" -o "$KARPENTER_CF"

# CloudFormation 실행하여 IAM Role 및 정책 생성
aws cloudformation deploy \
  --region ${AWS_REGION} \
  --stack-name "Karpenter-${EKSCLUSTER_NAME}" \
  --template-file "${KARPENTER_CF}" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides "ClusterName=${EKSCLUSTER_NAME}"
```

#### **📌 Karpenter 설치 시 생성되는 주요 AWS IAM 정책 및 역할 설명**

Karpenter를 설치할 때, 여러 개의 IAM 정책 및 역할(Role)이 필요합니다.\
각 정책 및 역할은 **EKS 노드 관리, EC2 인스턴스 생성, 중단 이벤트 처리** 등을 수행하는 데 사용됩니다.

### **1️⃣ InstanceStateChangeRule**

#### **📌 역할: EC2 인스턴스 상태 변경 감지 및 Karpenter 연동**

이 규칙은 EC2 인스턴스 상태가 변경될 때(예: 중지됨, 종료됨) Karpenter가 이를 감지하고,\
필요에 따라 새로운 노드를 배포할 수 있도록 돕습니다.

✅ **설명:**

* AWS **CloudWatch Event Rule**을 사용하여 EC2 상태 변경 이벤트를 감지
* **이벤트:** EC2 인스턴스의 `running`, `stopped`, `terminated` 상태 변경
* 감지된 이벤트를 **Amazon SQS** 또는 **AWS Lambda**로 전달하여 Karpenter가 처리

📌 **관련 서비스:**

* **EC2**: 인스턴스 상태 확인
* **CloudWatch Events**: 이벤트 트리거

***

### **2️⃣ KarpenterControllerPolicy**

#### **📌 역할: Karpenter 컨트롤러가 AWS 리소스를 관리할 수 있도록 허용**

이 IAM 정책은 Karpenter 컨트롤러가 **EC2 인스턴스를 생성, 종료, 관리**하는 데 필요한 **권한을 부여**합니다.

✅ **설명:**

* Karpenter가 **EC2 인스턴스를 생성 및 종료**할 수 있도록 **IAM 권한**을 설정
* **EKS 클러스터 정보를 조회**하여 Karpenter가 적절한 설정으로 노드를 배포할 수 있도록 허용
* **IAM Role을 통해 PassRole 권한 부여** (EC2 인스턴스에 IAM 역할을 할당할 수 있도록 설정)

📌 **주요 권한:**

* `ec2:RunInstances`, `ec2:TerminateInstances`, `ec2:DescribeInstances`
* `eks:DescribeCluster` (EKS 클러스터 정보 조회)
* `iam:PassRole` (EC2에 IAM 역할을 부여할 수 있도록 허용)

📌 **관련 서비스:**

* **EC2**: 인스턴스 관리
* **EKS**: 클러스터 정보 조회
* **IAM**: 역할 부여

***

### **3️⃣ KarpenterInterruptionQueue**

#### **📌 역할: Spot 인스턴스 중단 이벤트를 감지하기 위한 SQS 큐**

Spot 인스턴스가 중단될 경우 Karpenter가 **사전에 감지하여 새로운 인스턴스를 프로비저닝**할 수 있도록 하는 **Amazon SQS Queue**입니다.

✅ **설명:**

* AWS는 **Spot Instance** 중단 이벤트를 SQS에 전송
* Karpenter가 이를 감지하고 **새로운 노드를 배포**
* 중단이 감지되면, Karpenter는 **기존 워크로드를 새 노드로 이동**

📌 **관련 서비스:**

* **SQS**: Spot 중단 이벤트 대기
* **EC2**: Spot Instance 이벤트 감지

***

### **4️⃣ KarpenterInterruptionQueuePolicy**

#### **📌 역할: Karpenter가 Spot 중단 이벤트를 읽을 수 있도록 SQS 권한 부여**

Karpenter 컨트롤러가 `KarpenterInterruptionQueue`에 접근하여 **Spot Instance 중단 이벤트를 읽을 수 있도록** 하는 IAM 정책입니다.

✅ **설명:**

* Karpenter가 **SQS 메시지를 가져와서** 중단 이벤트를 확인할 수 있도록 권한 부여
* `sqs:ReceiveMessage`, `sqs:DeleteMessage` 등의 권한 포함

📌 **관련 서비스:**

* **SQS**: 메시지 읽기 및 삭제

***

### **5️⃣ KarpenterNodeRole**

#### **📌 역할: Karpenter가 생성하는 노드(EC2 인스턴스)에서 사용되는 IAM 역할**

Karpenter가 새로 생성하는 \*\*EC2 노드(Worker Node)\*\*가 사용할 **IAM Role**입니다.\
이 역할을 통해 EKS 클러스터에 정상적으로 연결되고, 컨테이너 이미지를 다운로드할 수 있습니다.

✅ **설명:**

* Karpenter에서 생성된 노드가 **EKS 클러스터에 정상적으로 참여할 수 있도록 설정**
* **ECR에서 컨테이너 이미지를 다운로드**할 수 있도록 허용
* EKS CNI(네트워크 인터페이스) 및 노드 관련 리소스에 접근할 수 있도록 허용

📌 **주요 권한:**

* `AmazonEKSWorkerNodePolicy` (EKS 노드로 등록할 수 있도록 허용)
* `AmazonEKS_CNI_Policy` (EKS 네트워크 CNI 관리)
* `AmazonEC2ContainerRegistryReadOnly` (ECR에서 컨테이너 이미지 다운로드)
* `AmazonSSMManagedInstanceCore` (AWS Systems Manager 활용 가능)

📌 **관련 서비스:**

* **EKS**: 워커 노드로 참여
* **ECR**: 컨테이너 이미지 다운로드
* **EC2**: 인스턴스 관리

***

### **6️⃣ RebalanceRule**

#### **📌 역할: EC2 인스턴스 리밸런싱 이벤트를 감지**

AWS가 Spot Instance 리밸런싱 이벤트를 트리거할 경우, Karpenter가 이를 감지하고 **워크로드를 안전하게 이동**할 수 있도록 하는 **CloudWatch Rule**입니다.

✅ **설명:**

* EC2 Spot Instance 리밸런싱 힌트(Spot Instance가 곧 종료될 가능성이 높음) 감지
* 감지된 이벤트를 기반으로 새로운 인스턴스를 생성하여 **워크로드를 미리 이동**

📌 **관련 서비스:**

* **EC2**: 인스턴스 리밸런싱 감지
* **CloudWatch Events**: 이벤트 트리거

***

### **7️⃣ ScheduledChangeRule**

#### **📌 역할: 예약된 변경 이벤트 감지**

AWS가 특정 EC2 인스턴스의 **종료, 재부팅, 상태 변경을 예약**할 경우 이를 감지하는 **CloudWatch Event Rule**입니다.

✅ **설명:**

* 예약된 변경 사항(예: EC2 인스턴스 종료 예약)이 있을 경우 사전에 감지
* 사전에 Karpenter가 **새로운 인스턴스를 배포**하여 다운타임 방지

📌 **관련 서비스:**

* **EC2**: 예약된 변경 감지
* **CloudWatch Events**: 이벤트 트리거

***

### **8️⃣ SpotInterruptionRule**

#### **📌 역할: EC2 Spot 인스턴스 중단 이벤트 감지**

Spot 인스턴스가 AWS에 의해 중단될 경우, 이를 감지하여 **Karpenter가 새로운 노드를 배포할 수 있도록 트리거하는 CloudWatch Rule**입니다.

✅ **설명:**

* AWS가 Spot 인스턴스를 중단하면 **CloudWatch Events**를 트리거
* Karpenter가 이를 감지하고 **새로운 노드를 생성**하여 애플리케이션 다운타임 방지

📌 **관련 서비스:**

* **EC2**: Spot Instance 종료 감지
* **CloudWatch Events**: 이벤트 트리거

<table><thead><tr><th width="263">역할</th><th width="296">설명</th><th>관련 서비스</th></tr></thead><tbody><tr><td><strong>InstanceStateChangeRule</strong></td><td>EC2 인스턴스 상태 변경 감지</td><td>EC2, CloudWatch Events</td></tr><tr><td><strong>KarpenterControllerPolicy</strong></td><td>Karpenter 컨트롤러가 AWS 리소스 관리</td><td>EC2, IAM, EKS</td></tr><tr><td><strong>KarpenterInterruptionQueue</strong></td><td>Spot 중단 이벤트를 대기하는 SQS Queue</td><td>SQS, EC2</td></tr><tr><td><strong>KarpenterInterruptionQueuePolicy</strong></td><td>SQS 메시지를 읽을 수 있도록 하는 IAM 정책</td><td>SQS, IAM</td></tr><tr><td><strong>KarpenterNodeRole</strong></td><td>Karpenter가 생성한 EC2 노드에서 사용되는 IAM Role</td><td>EC2, EKS, ECR</td></tr><tr><td><strong>RebalanceRule</strong></td><td>EC2 리밸런싱 이벤트 감지</td><td>EC2, CloudWatch Events</td></tr><tr><td><strong>ScheduledChangeRule</strong></td><td>예약된 변경 사항 감지</td><td>EC2, CloudWatch Events</td></tr><tr><td><strong>SpotInterruptionRule</strong></td><td>Spot 인스턴스 종료 감지 및 새로운 노드 배포</td><td>EC2, CloudWatch Events</td></tr></tbody></table>

### **3. OIDC Provider 설정**

Karpenter가 AWS IAM과 연동되려면 EKS 클러스터에서 **OIDC를 활성화**해야 합니다.

```sh
eksctl utils associate-iam-oidc-provider \
    --region ${AWS_REGION} \
    --cluster ${EKSCLUSTER_NAME} \
    --approve
```

> ✅ **OIDC Provider를 설정하면 Karpenter가 IAM 역할을 사용할 수 있도록 허용됩니다.**

***

### **4. Karpenter Controller IAM 역할 및 신뢰 관계 설정**

Karpenter 컨트롤러가 EKS 클러스터 및 AWS API와 통신할 수 있도록 IAM 역할(Role)을 생성해야 합니다.

이 역할은 OIDC (OpenID Connect)를 활용하여 EKS의 Service Account와 IAM Role을 연결합니다.

```sh
mkdir -p ~/environment/karpenter
cat <<EoF > ~/environment/karpenter/karpenter-controller-role.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:oidc-provider/$(aws eks describe-cluster --name ${EKSCLUSTER_NAME} --query "cluster.identity.oidc.issuer" --output text | cut -d'/' -f5)"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "$(aws eks describe-cluster --name ${EKSCLUSTER_NAME} --query "cluster.identity.oidc.issuer" --output text | cut -d'/' -f5):aud": "sts.amazonaws.com",
          "$(aws eks describe-cluster --name ${EKSCLUSTER_NAME} --query "cluster.identity.oidc.issuer" --output text | cut -d'/' -f5):sub": "system:serviceaccount:${KARPENTER_NAMESPACE}:karpenter"
        }
      }
    }
  ]
}
EoF
```

> ✅ **OIDC Provider와 연결하여 IAM 역할이 서비스 어카운트와 연동될 수 있도록 설정합니다.**

***

### **5. Helm을 사용한 Karpenter 설치**

Karpenter를 설치하고, 특정 노드 그룹에서 실행되도록 설정합니다.

```sh
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version "${KARPENTER_VERSION}" \
  --namespace "${KARPENTER_NAMESPACE}" --create-namespace \
  --set "settings.clusterName=${EKSCLUSTER_NAME}" \
  --set "settings.interruptionQueue=${EKSCLUSTER_NAME}" \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:role/${EKSCLUSTER_NAME}-karpenter" \
  --set "nodeSelector.nodegroup-type=managed-backend-workloads" \
  --set "tolerations[0].key=nodegroup-type" \
  --set "tolerations[0].operator=Equal" \
  --set "tolerations[0].value=managed-backend-workloads" \
  --set "tolerations[0].effect=NoSchedule" \
  --debug \
  --wait
```

> ✅ **`managed-backend-workloads` 라벨이 있는 노드에서만 Karpenter Pod가 실행되도록 설정합니다.**

***

### **6. Karpenter가 올바른 노드에서 실행되는지 확인**

```sh
kubectl get pods -n ${KARPENTER_NAMESPACE} -o wide
```

> ✅ Karpenter Pod가 실행되고 있는지 확인합니다.

```
kubectl get pods -n ${KARPENTER_NAMESPACE} -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\t"}{.spec.tolerations[*].value}{"\n"}{end}' | grep managed-backend-workloads

```

아래와 같은 결과를 얻을 수 있습니다.

```
karpenter-bbc4dd8f6-f6dd8       ip-10-11-68-224.ap-northeast-2.compute.internal managed-backend-workloads
karpenter-bbc4dd8f6-fhmrx       ip-10-11-89-224.ap-northeast-2.compute.internal managed-backend-workloads
```



### **7. Provisioner 구성**

Karpenter는 \*\*Provisioner CRD (Custom Resource Definition)\*\*를 통해 Pod의 요구 사항에 맞춰 EC2 인스턴스를 자동으로 생성하고 관리합니다.\
Provisioner는 **라벨(Label), Affinity, Taint & Toleration** 등의 Pod 속성을 기반으로 **적절한 노드를 자동 배포**합니다.

***

### **1️⃣ Spot 인스턴스를 사용하는 Provisioner 생성**

아래의 Provisioner는 **EC2 Spot Instance**를 사용하여 비용 절감을 극대화합니다.\
`securityGroupSelector` 및 `subnetSelector`를 사용하여 **Karpenter가 적절한 리소스를 자동 검색**하도록 설정합니다.

📌 **Provisioner YAML 생성 및 적용**

```sh
cat << EOF > ~/environment/karpenter/karpenter-provisioner1.yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: provisioner1
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot"]  # EC2 Spot 인스턴스를 사용
  limits:
    resources:
      cpu: 1000  # 최대 1000 vCPU 제한
  provider:
    subnetSelector:
      karpenter.sh/discovery: ${EKSCLUSTER_NAME}  # Karpenter가 사용할 서브넷 자동 검색
    securityGroupSelector:
      karpenter.sh/discovery: ${EKSCLUSTER_NAME}  # 보안 그룹 자동 검색
  ttlSecondsAfterEmpty: 30  # 30초 동안 사용되지 않은 노드는 자동 종료
EOF

# Provisioner 적용
kubectl apply -f ~/environment/karpenter/karpenter-provisioner1.yaml
```

***

### **8. 자동 노드 프로비저닝 1 (Spot Instances)**

### **1️⃣ Spot 인스턴스를 사용하기 위한 설정**

Spot 인스턴스를 사용하기 위해 **AWS Spot Service**에 대한 설정이 필요합니다.

```sh
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true
```

위 명령은 AWS에 Spot 인스턴스를 사용하기 위한 역할을 생성합니다.

***

### **2️⃣ Kube-ops-view 설치 (클러스터 상태 모니터링)**

현재 노드와 Pod의 배치 상태를 시각적으로 확인하기 위해 **Kube-ops-view**를 설치합니다.

```sh
kubectl create namespace kube-tools
helm repo add geek-cookbook https://geek-cookbook.github.io/charts/
helm install kube-ops-view geek-cookbook/kube-ops-view --version 1.2.2 --namespace kube-tools
kubectl patch svc -n kube-tools kube-ops-view -p '{"spec":{"type":"LoadBalancer"}}'

```

웹 브라우저에서 해당 URL을 열면 클러스터 상태를 실시간으로 모니터링할 수 있습니다.

```
# Kube-ops-view 서비스 주소 확인
kubectl -n kube-tools get svc kube-ops-view | tail -n 1 | awk '{ print "kube-ops-view URL = http://"$4":8080" }'

```

***

### **3️⃣ EKS Node Viewer 설치**

`eks-node-viewer`는 CLI 환경에서 클러스터 노드 상태를 확인하는 도구입니다.

```sh
# eks-node-viewer설치를 위한 Go Install
curl -LO https://go.dev/dl/go1.21.5.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.21.5.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
echo 'export GOPATH=$HOME/go' >> ~/.bashrc
echo 'export PATH=$PATH:$GOPATH/bin' >> ~/.bashrc
source ~/.bashrc
go version

# EKS Node Viewer 설치
mkdir -p ~/go/bin
go install github.com/awslabs/eks-node-viewer/cmd/eks-node-viewer@v0.5.0
```

**새로운 터미널에서 eks-node-viewer를 실행합니다.**

```
# 새로운 터미널에서 실행
source ~/.bashrc
cd ~/go/bin
./eks-node-viewer
```

### **4️⃣ Karpenter 자동 노드 프로비저닝 테스트**

Karpenter가 Spot 인스턴스를 자동으로 생성하는지 확인하기 위해 **테스트용 Deployment**를 생성합니다.

```sh
# Karpenter 테스트용 네임스페이스 생성
kubectl create namespace karpenter-inflate

# Deployment 정의 파일 생성
cat << EOF > ~/environment/karpenter/karpenter-inflate1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate1
  namespace: karpenter-inflate
spec:
  replicas: 0  # 초기 Pod 개수 (추후 Karpenter가 노드를 자동 생성)
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
        karpenter.sh/capacity-type: spot  # Karpenter가 Spot 인스턴스를 사용하도록 지정
EOF

# Deployment 적용
kubectl apply -f ~/environment/karpenter/karpenter-inflate1.yaml
```

📌 **Replica 수를 늘려서 Karpenter의 동작을 확인합니다.**

```sh
kubectl -n karpenter-inflate scale deployment inflate1 --replicas 5
```

📌 **Karpenter의 로그를 실시간으로 확인하여 새로운 노드가 생성되는지 모니터링합니다.**

```sh
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller
```

📌 **Spot Instance가 정상적으로 생성되었는지 확인합니다.**

```sh
kubectl get nodes
```

***

### **9. 자동 노드 프로비저닝 2 (On-Demand Instances)**

Spot Instance 외에도 **On-Demand Instance**를 사용하도록 새로운 Provisioner를 생성할 수 있습니다.

### **1️⃣ On-Demand Instance를 사용하는 Provisioner 생성**

```sh
cat << EOF > ~/environment/karpenter/karpenter-provisioner2.yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: provisioner2
spec:
  taints:
    - key: cpuIntensive
      value: "true"
      effect: NoSchedule  # 특정 Pod가 Toleration 없이 이 노드에서 실행되지 않도록 설정
  labels:
    phase: test2
    nodeType: cpu-node  # CPU 집약적인 워크로드를 위한 노드 유형 태그
  requirements:
    - key: "node.kubernetes.io/instance-type"
      operator: In
      values: ["c5.xlarge"]  # 생성할 노드 유형을 c5.xlarge로 제한
    - key: "topology.kubernetes.io/zone"
      operator: In
      values: ["ap-northeast-1a"]  # 특정 가용 영역 지정
    - key: "karpenter.sh/capacity-type"
      operator: In
      values: ["on-demand"]  # 온디맨드 인스턴스 사용
  limits:
    resources:
      cpu: 1000
  provider:
    subnetSelector:
      karpenter.sh/discovery: ${EKSCLUSTER_NAME}
    securityGroupSelector:
      karpenter.sh/discovery: ${EKSCLUSTER_NAME}
  ttlSecondsAfterEmpty: 30
EOF

# Provisioner 적용
kubectl apply -f ~/environment/karpenter/karpenter-provisioner2.yaml
```

***

### **2️⃣ On-Demand Instance용 Deployment 생성**

```sh
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

# Deployment 적용
kubectl apply -f ~/environment/karpenter/karpenter-inflate2.yaml
```

📌 **Replica를 늘려 새로운 On-Demand Instance가 생성되는지 확인합니다.**

```sh
kubectl -n karpenter-inflate scale deployment inflate2 --replicas 5
```

이제 Karpenter가 자동으로 **On-Demand 인스턴스를 프로비저닝**하는 것을 확인할 수 있습니다. 🚀



