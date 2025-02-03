---
description: Update 2025-01-30
---

# 스케쥴링-Karpenter

## Karpenter 소개

### **Karpenter란?**

Karpenter는 Amazon EKS(EKS) 및 Kubernetes 클러스터에서 동적으로 워크로드 수요에 맞게 노드를 프로비저닝하는 자동 스케일링 솔루션입니다. Kubernetes 클러스터의 자원을 효율적으로 관리할 수 있도록 도와주며, 필요할 때만 노드를 추가 및 제거하여 비용 최적화를 가능하게 합니다.

***

### **Karpenter 설치 순서**

1. **환경 변수 설정 및 Kubernetes 버전 확인**
   * Karpenter와 Kubernetes 버전 호환성을 확인
   * AWS 환경 변수를 설정하여 Karpenter 설치 준비
2. **Karpenter Node IAM Role 및 Instance Profile 생성**
   * Karpenter가 생성하는 노드에 필요한 IAM 역할 및 인스턴스 프로필 생성
3. **aws-auth ConfigMap에 Karpenter Node Role 추가**
   * 노드가 클러스터와 연결할 수 있도록 IAM 역할을 추가
4. **Karpenter Controller IAM Role 생성**
   * Karpenter 컨트롤러가 필요한 권한을 가질 수 있도록 IAM 역할 생성
5. **EC2 Spot Service Linked Role 생성**
   * EC2 Spot 사용을 위한 서비스 연결 역할 생성
6. **Helm을 이용한 Karpenter 설치**
   * Helm Chart를 사용하여 Karpenter를 EKS 클러스터에 배포
7. **Karpenter 모니터링 도구들에 대한 설치**
   * `eks-node-viewer`를 설치하여 노드 리소스 사용량을 시각적으로 확인

***

## **1. 환경 변수 설정 및 Kubernetes 버전 확인**

### **1.1 Kubernetes 버전 호환성 확인**

```sh
echo "🔍 Fetching Karpenter Versions..."
# GitHub API에서 최신 Karpenter 릴리즈 목록 가져오기
RELEASES=$(curl -sL "https://api.github.com/repos/aws/karpenter/releases" | jq -r '.[].tag_name')
# Karpenter 0.34.x 및 0.37.x, 1.0.5, 1.1.x 및 1.2.x 버전 필터링
KARPENTER_VERSION_0_34=$(echo "$RELEASES" | grep "^v0\.34\." | sort -V | tail -n 1 | sed 's/v//')
KARPENTER_VERSION_0_37=$(echo "$RELEASES" | grep "^v0\.37\." | sort -V | tail -n 1 | sed 's/v//')
KARPENTER_VERSION_1_1=$(echo "$RELEASES" | grep "^v1\.0\." | sort -V | tail -n 1 | sed 's/v//')
KARPENTER_VERSION_1_1=$(echo "$RELEASES" | grep "^v1\.1\." | sort -V | tail -n 1 | sed 's/v//')
KARPENTER_VERSION_1_2=$(echo "$RELEASES" | grep "^v1\.2\." | sort -V | tail -n 1 | sed 's/v//')
echo "$LINE"
# Latest Karpenter Versions
echo "🚀 Latest Karpenter Versions:"
echo " - Karpenter 0.34.x (For K8s 1.29)  : $KARPENTER_VERSION_0_34"
echo " - Karpenter 0.37.x (For K8s 1.30)  : $KARPENTER_VERSION_0_37"
echo " - Karpenter 1.0.5  (For K8s 1.31)  : $KARPENTER_VERSION_1_0_5"
echo " - Karpenter 1.1.x  (For K8s 1.31+) : $KARPENTER_VERSION_1_1"
echo " - Karpenter 1.2.x  (For K8s 1.32)  : $KARPENTER_VERSION_1_2"
echo "$LINE"
# Kubernetes Version Compatibility
echo "🟢 Kubernetes Version Compatibility:"
echo "  - Kubernetes 1.29 → Karpenter $KARPENTER_VERSION_0_34"
echo "  - Kubernetes 1.30 → Karpenter $KARPENTER_VERSION_0_37"
echo "  - Kubernetes 1.31 → Karpenter $KARPENTER_VERSION_1_0_5 또는 $KARPENTER_VERSION_1_1"
echo "  - Kubernetes 1.32 → Karpenter $KARPENTER_VERSION_1_2"
echo "$LINE"
echo "reference : https://karpenter.sh/v1.2/upgrading/compatibility "
echo "$LINE"

```

### **1.2 환경 변수 설정**

<pre class="language-sh"><code class="lang-sh"><strong>#이 설치가이드에서는 karpenter 1.1.3 으로 구성합니다.
</strong><strong>export KARPENTER_VERSION="1.1.3"
</strong>export TOKEN=`curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"`
export CLUSTER_NAME=$(eksctl get clusters -o json | jq -r '.[0].Name')
export AWS_REGION=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
export ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.endpoint" --output text)"
export KARPENTER_NAMESPACE="kube-system"

echo CLUSTER_NAME:$CLUSTER_NAME AWS Region:$AWS_REGION Account ID:$ACCOUNT_ID Cluster Endpoint:$CLUSTER_ENDPOINT Karpenter Namespace:$KARPENTER_NAMESPACE

</code></pre>

***

## **2. Karpenter Node IAM Role 및 Instance Profile 생성**

```sh
mkdir -p ~/environment/karpenter
export KARPENTER_CF="$HOME/environment/karpenter/k-node-iam-role.yaml"

# CloudFormation 템플릿 다운로드 및 실행
curl -fsSL "https://raw.githubusercontent.com/aws/karpenter-provider-aws/v${KARPENTER_VERSION}/website/content/en/preview/getting-started/getting-started-with-karpenter/cloudformation.yaml" -o "$KARPENTER_CF"

aws cloudformation deploy \
  --region ${AWS_REGION} \
  --stack-name "Karpenter-${CLUSTER_NAME}" \
  --template-file "${KARPENTER_CF}" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides "ClusterName=${CLUSTER_NAME}"
```

***

### **2.1 생성되는 리소스 개요**

Karpenter의 CloudFormation 템플릿을 실행하면 다음과 같은 AWS 리소스들이 생성됩니다. 각 리소스의 역할과 기능을 아래에서 설명합니다.

**① InstanceStateChangeRule**

* EC2 인스턴스의 상태 변경(예: 실행 중, 종료됨 등)에 대한 이벤트를 감지하고 이를 기반으로 자동 조치를 수행하는 AWS EventBridge 규칙입니다.
* Karpenter가 노드의 상태를 실시간으로 감지하여 효율적인 노드 관리를 가능하게 합니다.

#### **② KarpenterControllerPolicy**

* Karpenter 컨트롤러가 EC2 인스턴스를 관리하고 필요한 AWS API에 접근할 수 있도록 하는 IAM 정책입니다.
* EC2 인스턴스 생성, 삭제, 태그 관리 등의 권한을 포함합니다.

#### **③ KarpenterInterruptionQueue**

* Karpenter에서 사용되는 Amazon SQS(Simple Queue Service) 대기열입니다.
* 스팟 인스턴스의 종료 이벤트, 유지보수 이벤트 등의 중단(interruption) 관련 메시지를 받아서 처리합니다.

#### **④ KarpenterInterruptionQueuePolicy**

* `KarpenterInterruptionQueue`에 대한 액세스를 제어하는 IAM 정책입니다.
* AWS의 중단 관련 이벤트가 해당 SQS 대기열로 전달될 수 있도록 허용합니다.

#### **⑤ KarpenterNodeRole**

* Karpenter가 생성하는 모든 EC2 노드(Worker Node)에 부여되는 IAM 역할입니다.
* 컨테이너 실행, 네트워크 설정, 로깅, EBS 볼륨 연결 등의 작업을 수행할 수 있도록 권한을 부여합니다.
* 일반적으로 AmazonEKSWorkerNodePolicy 및 기타 EC2 관련 권한이 포함됩니다.

#### **⑥ RebalanceRule**

* AWS EC2의 `Instance Rebalance Recommendation` 이벤트를 감지하는 AWS EventBridge 규칙입니다.
* 스팟 인스턴스가 조기 종료될 위험이 있을 경우 이를 Karpenter가 사전에 감지하고 대비할 수 있도록 합니다.

#### **⑦ ScheduledChangeRule**

* 예약된 이벤트를 기반으로 Karpenter의 노드 배치를 자동화하는 AWS EventBridge 규칙입니다.
* 예를 들어, 특정 시간대에 새로운 노드를 추가하거나 제거하는 자동화 정책을 설정할 수 있습니다.

#### **⑧ SpotInterruptionRule**

* AWS EC2의 스팟 인스턴스 중단(Interruption) 이벤트를 감지하는 AWS EventBridge 규칙입니다.
* 스팟 인스턴스가 종료될 예정일 때 Karpenter가 이를 감지하고, 새로운 노드를 자동으로 프로비저닝하여 서비스 중단을 방지할 수 있도록 합니다.

***

## **3. aws-auth ConfigMap에 Karpenter Node Role 추가**

해당 명령어는 Karpenter가 생성한 노드가 EKS 클러스터에 정상적으로 연결될 수 있도록 IAM 역할을 `aws-auth` ConfigMap에 추가하는 과정입니다.

```sh
eksctl create iamidentitymapping \
  --username system:node:{{EC2PrivateDNSName}} \
  --cluster "${CLUSTER_NAME}" \
  --arn "arn:aws:iam::${ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}" \
  --group system:bootstrappers \
  --group system:nodes

kubectl describe configmap -n kube-system aws-auth

```

#### **3.1 명령어 역할 설명**

아래 명령어를 실행하면 Karpenter가 프로비저닝하는 노드들이 Amazon EKS 클러스터에 적절한 권한을 가지고 연결될 수 있도록 설정됩니다.

```sh
eksctl create iamidentitymapping \
  --username system:node:{{EC2PrivateDNSName}} \
  --cluster "${CLUSTER_NAME}" \
  --arn "arn:aws:iam::${ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}" \
  --group system:bootstrappers \
  --group system:nodes
```

① **eksctl create iamidentitymapping**

* EKS 클러스터에 IAM 역할을 매핑하는 명령어입니다.
* 해당 역할을 통해 EC2 인스턴스(Worker Node)가 EKS 클러스터에 인증 및 접근할 수 있도록 설정됩니다.

② **--username system:node:\{{EC2PrivateDNSName\}}**

* `system:node:{{EC2PrivateDNSName}}`는 각 Karpenter 노드에 대한 Kubernetes 인증 사용자 이름을 정의합니다.
* `{{EC2PrivateDNSName}}`는 해당 노드의 개인 DNS 이름을 의미하며, 노드가 EKS 클러스터 내에서 유일한 ID를 갖도록 합니다.

③ **--cluster "${CLUSTER\_NAME}"**

* `--cluster` 옵션을 사용하여 특정 Amazon EKS 클러스터에 IAM 역할을 매핑합니다.
* `"${CLUSTER_NAME}"`는 현재 사용 중인 클러스터의 이름을 나타냅니다.

④ **--arn "arn:aws:iam::${ACCOUNT\_ID}:role/KarpenterNodeRole-${CLUSTER\_NAME}"**

* EKS 클러스터에 연결할 IAM 역할(Instance Role)의 ARN(Amazon Resource Name)을 지정합니다.
* `"KarpenterNodeRole-${CLUSTER_NAME}"`는 Karpenter가 프로비저닝하는 노드들이 사용할 IAM 역할입니다.

⑤ **--group system:bootstrappers**

* `system:bootstrappers` 그룹은 노드가 클러스터 초기 설정(부트스트래핑)을 수행할 수 있도록 권한을 부여합니다.

⑥ **--group system:nodes**

* `system:nodes` 그룹은 Kubernetes에서 해당 노드를 Worker Node로 인식하도록 설정합니다.
* Worker Node는 Kubernetes에서 워크로드를 실행하는 역할을 하며, 이를 위해 적절한 인증 및 권한이 필요합니다.

***

#### **3.2 aws-auth ConfigMap에서 설정 확인하기**

```sh
kubectl describe configmap -n kube-system aws-auth
```

⑦ **ConfigMap을 조회하여 매핑된 IAM 역할 확인**

* `aws-auth` ConfigMap은 Amazon EKS에서 IAM 역할을 Kubernetes 사용자 및 그룹과 매핑하는 역할을 합니다.
* 위 명령어를 실행하면 현재 `aws-auth` ConfigMap에 포함된 IAM Role 및 User 목록을 확인할 수 있습니다.
* 정상적으로 반영되었다면 `KarpenterNodeRole-${CLUSTER_NAME}` 역할이 추가되어 있어야 합니다.

***



## **4. Karpenter Controller IAM Role 생성**

Karpenter가 Amazon EKS 클러스터에서 정상적으로 동작하기 위해서는 EC2 인스턴스를 생성하고 관리할 수 있는 권한이 필요합니다. 이를 위해 Karpenter 컨트롤러에 IAM 역할을 할당해야 합니다.

***

#### **4.1 Karpenter Controller IAM Role 생성 과정**

아래 명령어를 실행하여 Karpenter 컨트롤러의 IAM 역할을 생성하고, Amazon EKS 클러스터의 OIDC(Identity Provider)와 연결합니다.

```sh
eksctl utils associate-iam-oidc-provider --cluster ${CLUSTER_NAME} --approve
```

① **OIDC Provider와 클러스터 연결**

* Amazon EKS는 IAM OIDC(Identity Provider)를 사용하여 서비스 계정을 IAM 역할과 연결할 수 있도록 합니다.
* 위 명령어를 실행하면 현재 사용 중인 클러스터에 OIDC Provider가 연결됩니다.
* OIDC Provider가 설정되지 않으면 IAM 역할을 서비스 계정과 매핑할 수 없습니다.

***

#### **4.2 Karpenter Controller의 IAM 역할 생성 및 연결**

```sh
eksctl create iamserviceaccount \
  --cluster "${CLUSTER_NAME}" --name karpenter --namespace $KARPENTER_NAMESPACE \
  --role-name "${CLUSTER_NAME}-karpenter" \
  --attach-policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/KarpenterControllerPolicy-${CLUSTER_NAME}" \
  --role-only \
  --approve
```

② **IAM Service Account 생성 및 IAM 역할 할당**

* Karpenter 컨트롤러는 Amazon EKS 내부에서 실행되는 Kubernetes Pod이므로, 이를 IAM 역할과 연결하기 위해 서비스 계정(Service Account)을 생성합니다.
* `eksctl create iamserviceaccount` 명령어를 실행하면 Amazon EKS 클러스터의 특정 네임스페이스(Kube-system)에 Karpenter 컨트롤러를 위한 서비스 계정이 생성됩니다.

③ **IAM 역할과 정책 연결**

* `--role-name "${CLUSTER_NAME}-karpenter"`를 통해 IAM 역할을 생성하고,
* `--attach-policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/KarpenterControllerPolicy-${CLUSTER_NAME}"`를 사용하여 Karpenter 컨트롤러가 EC2 관련 API를 호출할 수 있도록 적절한 IAM 정책을 연결합니다.

***

#### **4.3 IAM 역할 생성 확인**

```sh
export KARPENTER_IAM_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/${CLUSTER_NAME}-karpenter"
echo $KARPENTER_IAM_ROLE_ARN
```

④ **생성된 IAM 역할 ARN 확인**

* 위 명령어를 실행하면 생성된 IAM 역할의 ARN이 출력됩니다.
* 이 ARN은 이후 Helm을 사용하여 Karpenter를 설치할 때 필요하므로 반드시 확인해야 합니다.

```sh
eksctl get iamserviceaccount --cluster $CLUSTER_NAME --namespace $KARPENTER_NAMESPACE
```

⑤ **IAM 서비스 계정 확인**

* Amazon EKS에서 `eksctl get iamserviceaccount` 명령어를 사용하여 해당 IAM 역할이 Karpenter 컨트롤러에 정상적으로 할당되었는지 확인할 수 있습니다.

***

#### **4.4 설정의 필요성과 의미**

⑥ **설정의 의미**

* Karpenter는 EC2 인스턴스를 자동으로 생성하고 관리하는 Kubernetes 컨트롤러입니다.
* 따라서 EC2 인스턴스를 실행 및 종료하는 API 호출이 가능하도록 적절한 IAM 역할을 부여해야 합니다.
* IAM OIDC Provider를 통해 Karpenter 컨트롤러가 Kubernetes 클러스터 내부에서 실행될 때, AWS API에 안전하게 접근할 수 있도록 설정됩니다.

***

## **5. EC2 Spot Service Linked Role 생성**

Amazon EC2에서 스팟 인스턴스를 처음 사용할 경우, EC2 서비스와 AWS IAM 간의 연결을 자동으로 관리하는 **Service Linked Role**을 생성해야 합니다.

```sh
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true
```

***

## **6. Helm을 이용한 Karpenter 설치**

Karpenter는 Helm Chart를 사용하여 Amazon EKS 클러스터에 배포됩니다. Helm은 Kubernetes 애플리케이션을 패키징하고 배포하는 도구로, Karpenter의 설치 및 업그레이드를 보다 쉽게 관리할 수 있습니다.

```sh
helm registry logout public.ecr.aws
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter --version "${KARPENTER_VERSION}" \
  --namespace "${KARPENTER_NAMESPACE}" --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=${KARPENTER_IAM_ROLE_ARN} \
  --set settings.clusterName=${CLUSTER_NAME} \
  --set settings.clusterEndpoint=${CLUSTER_ENDPOINT} \
  --set settings.interruptionQueue=${CLUSTER_NAME} \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --debug \
  --wait

```



***

## **7. Karpenter 모니터링 도구 설치**

```sh
wget -O eks-node-viewer https://github.com/awslabs/eks-node-viewer/releases/download/v0.6.0/eks-node-viewer_Linux_x86_64
chmod +x eks-node-viewer
sudo mv -v eks-node-viewer /usr/local/bin

eks-node-viewer
```

이제 Karpenter가 정상적으로 설치되고 모니터링할 수 있습니다! 🚀
