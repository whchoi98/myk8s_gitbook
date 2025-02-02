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

## **기존 EKS 클러스터에 Karpenter v1.2.1 설치 방법**

#### ✅ **기존 EKS 클러스터에 Karpenter 1.2.1 설치하는 방법**

**환경:**

* Karpenter version 1.2.1
* 기존 EKS 클러스터 사용 (`eksworkshop`)
* AWS 리전: `ap-northeast-2`
* 기존 서브넷 및 보안 그룹 사용

***

### **1. Karpenter 설치를 위한 환경 변수 설정**

```sh
# Karpenter 관련 정보
export KARPENTER_VERSION="1.2.1"
export KARPENTER_NAMESPACE=karpenter
export AWS_PARTITION="aws"

# IAM Role 이름 설정
export KARPENTER_NODE_ROLE="KarpenterNodeRole-${EKSCLUSTER_NAME}"
export KARPENTER_CONTROLLER_ROLE="${EKSCLUSTER_NAME}-karpenter"
export KARPENTER_IAM_POLICY="KarpenterControllerPolicy-${EKSCLUSTER_NAME}"

# 클러스터 엔드포인트 가져오기
export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name ${EKSCLUSTER_NAME} --query "cluster.endpoint" --region ${AWS_REGION} --output text)"
export KARPENTER_IAM_ROLE_ARN="arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:role/${EKSCLUSTER_NAME}-karpenter"
echo "export CLUSTER_ENDPOINT=${CLUSTER_ENDPOINT}" | tee -a ~/.bash_profile
source ~/.bash_profile
```

***

### **2. 서브넷 태그 추가 (Karpenter가 사용할 서브넷 지정)**

Karpenter가 노드를 배포할 서브넷에 태그 추가:

```sh
aws ec2 create-tags --resources "$PublicSubnet01" --tags Key="karpenter.sh/discovery",Value="${EKSCLUSTER_NAME}" --region ${AWS_REGION}
aws ec2 create-tags --resources "$PublicSubnet02" --tags Key="karpenter.sh/discovery",Value="${EKSCLUSTER_NAME}" --region ${AWS_REGION}
aws ec2 create-tags --resources "$PublicSubnet03" --tags Key="karpenter.sh/discovery",Value="${EKSCLUSTER_NAME}" --region ${AWS_REGION}
aws ec2 create-tags --resources "$PrivateSubnet01" --tags Key="karpenter.sh/discovery",Value="${EKSCLUSTER_NAME}" --region ${AWS_REGION}
aws ec2 create-tags --resources "$PrivateSubnet02" --tags Key="karpenter.sh/discovery",Value="${EKSCLUSTER_NAME}" --region ${AWS_REGION}
aws ec2 create-tags --resources "$PrivateSubnet03" --tags Key="karpenter.sh/discovery",Value="${EKSCLUSTER_NAME}" --region ${AWS_REGION}
```

***

### **3. OIDC Provider 설정**

Kubernetes와 AWS IAM 간 인증을 위해 OIDC Provider 설정:

```sh
eksctl utils associate-iam-oidc-provider \
    --region ${AWS_REGION} \
    --cluster ${EKSCLUSTER_NAME} \
    --approve
```

***

### **4. Karpenter Node를 위한 IAM Role 생성**

#### **4.1. CloudFormation을 이용하여 IAM Role 생성**

```sh
mkdir -p ~/environment/karpenter
export KARPENTER_CF="$HOME/environment/karpenter/k-node-iam-role.yaml"

curl -fsSL "https://raw.githubusercontent.com/aws/karpenter-provider-aws/v${KARPENTER_VERSION}/website/content/en/preview/getting-started/getting-started-with-karpenter/cloudformation.yaml" -o "$KARPENTER_CF"

aws cloudformation deploy \
  --region ${AWS_REGION} \
  --stack-name "Karpenter-${EKSCLUSTER_NAME}" \
  --template-file "${KARPENTER_CF}" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides "ClusterName=${EKSCLUSTER_NAME}"
```

#### **Service linked role**

```
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true

```

#### **4.2. IAM Role을 Kubernetes `aws-auth`에 Mapping**

```sh
eksctl create iamidentitymapping \
  --region ${AWS_REGION} \
  --username system:node:{{EC2PrivateDNSName}} \
  --cluster ${EKSCLUSTER_NAME} \
  --arn "arn:aws:iam::${ACCOUNT_ID}:role/${KARPENTER_NODE_ROLE}" \
  --group system:bootstrappers \
  --group system:nodes
```

✅ **`aws-auth` ConfigMap에 추가되었는지 확인:**

```sh
kubectl get configmap -n kube-system aws-auth -o yaml

```

아래와 같이 신규 mapRoles가 생성됩니다.

```
$ kubectl get configmap -n kube-system aws-auth -o yaml
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::xxxxxxxxxxxx:role/KarpenterNodeRole-eksworkshop
      username: system:node:{{EC2PrivateDNSName}}
```

### **5. Karpenter Service Account 생성 (IAM Role 매핑)**

Karpenter 컨트롤러가 AWS 리소스에 접근할 수 있도록 IAM Service Account 생성:

```sh
eksctl create iamserviceaccount \
  --region ${AWS_REGION} \
  --cluster "${EKSCLUSTER_NAME}" \
  --namespace ${KARPENTER_NAMESPACE} \
  --name karpenter \
  --role-name "${KARPENTER_CONTROLLER_ROLE}" \
  --attach-policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/${KARPENTER_IAM_POLICY}" \
  --role-only \
  --override-existing-serviceaccounts \
  --approve
```

IAM Role ARN 저장:

<pre class="language-sh"><code class="lang-sh"><strong>
</strong><strong>export KARPENTER_IAM_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/${KARPENTER_CONTROLLER_ROLE}"
</strong>echo "export KARPENTER_IAM_ROLE_ARN=${KARPENTER_IAM_ROLE_ARN}" | tee -a ~/.bash_profile
source ~/.bash_profile
</code></pre>

***

### **🔹 6. Karpenter 설치 (Helm Chart 사용)**

Karpenter 컨트롤러를 클러스터에 배포합니다.

```bash
# Logout of helm registry to perform an unauthenticated pull against the public ECR
helm registry logout public.ecr.aws
helm repo add karpenter https://charts.karpenter.sh
helm repo update
helm registry logout public.ecr.aws

helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version "${KARPENTER_VERSION}" \
  --namespace "${KARPENTER_NAMESPACE}" --create-namespace \
  --set "settings.clusterName=${EKSCLUSTER_NAME}" \
  --set "settings.interruptionQueue=${EKSCLUSTER_NAME}" \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --set "nodeSelector.nodegroup-type=managed-backend-workloads" \
  --debug \
  --wait

```

```sh
helm repo add karpenter https://charts.karpenter.sh
helm repo update
helm registry logout public.ecr.aws
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version ${KARPENTER_VERSION} \
  --namespace ${KARPENTER_NAMESPACE} --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=${KARPENTER_IAM_ROLE_ARN} \
  --set settings.aws.clusterName=${EKSCLUSTER_NAME} \
  --set settings.aws.clusterEndpoint=${K_CLUSTER_ENDPOINT} \
  --set settings.aws.defaultInstanceProfile=KarpenterNodeInstanceProfile-${EKSCLUSTER_NAME} \
  --set settings.aws.interruptionQueueName=${EKSCLUSTER_NAME} \
  --debug \
  --wait
```

✅ **Karpenter Pod 배포 확인**

```sh
kubectl get pods --namespace ${KARPENTER_NAMESPACE}
kubectl get deployment -n ${KARPENTER_NAMESPACE}
```

***

### **🔹 7. Karpenter Provisioner 생성**

Karpenter가 자동으로 노드를 추가할 수 있도록 **Provisioner** 리소스를 생성합니다.

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.k8s.aws/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  limits:
    resources:
      cpu: 1000
      memory: 1000Gi
  provider:
    instanceProfile: "KarpenterNodeInstanceProfile-${EKSCLUSTER_NAME}"
  consolidation:
    enabled: true
  ttlSecondsAfterEmpty: 30
EOF
```

✅ **Provisioner 확인**

```sh
kubectl get provisioners
```

***

### **🔹 8. Karpenter 동작 테스트**

Karpenter가 자동으로 노드를 생성하는지 테스트:

#### ✅ **8.1. 스케일링 테스트**

```sh
kubectl run test-pod --image=nginx --replicas=5
```

*   새로운 노드가 추가되는지 확인:

    ```sh
    kubectl get nodes
    ```
*   Karpenter 이벤트 로그 확인:

    ```sh
    kubectl get events -A | grep Karpenter
    ```

***

### **🔹 9. Karpenter 삭제 (옵션)**

테스트가 끝난 후 Karpenter를 삭제하려면:

```sh
helm uninstall karpenter --namespace ${KARPENTER_NAMESPACE}
aws cloudformation delete-stack --stack-name "Karpenter-${EKSCLUSTER_NAME}"
eksctl delete iamserviceaccount --name karpenter --namespace ${KARPENTER_NAMESPACE} --cluster ${EKSCLUSTER_NAME}
```

***

### 🎯 **결론**

✅ 기존 EKS 클러스터에 Karpenter 1.2.1을 설치.\
✅ OIDC, IAM Role, Helm을 사용하여 설정.\
✅ **자동 노드 프로비저닝 및 스케일링이 가능하도록 구성.** 🚀





