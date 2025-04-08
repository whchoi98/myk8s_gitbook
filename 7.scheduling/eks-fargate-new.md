---
description: 'Update : 2025.04.08'
---

# EKS Fargate (New)

## EKS Fargate

Amazon EKS는 컨테이너 실행을 위한 두 가지 방식의 노드를 지원합니다. 하나는 EC2 기반의 노드 그룹, 다른 하나는 Fargate입니다. Fargate는 서버를 직접 관리하지 않아도 되는 Serverless 방식의 Pod 실행 환경을 제공합니다.

Fargate를 활용하면 운영 부담을 줄이면서 빠르게 Kubernetes 워크로드를 실행할 수 있으며, 사용한 리소스에 대해서만 비용을 지불하게 됩니다.\


### 1. Fargate 개요

• Serverless 방식으로 Kubernetes Pod를 실행

• EC2 인스턴스를 직접 프로비저닝하거나 관리할 필요 없음

• Pod 단위로 격리된 실행 환경 제공 (보안/자원 관점에서 유리)

• CPU/메모리 단위로 과금 (초 단위 기준)

> 📌 Tip: EC2 기반 노드보다 Pod 시작 시간이 다소 느릴 수 있으며, GMSA, HostPath 등 일부 Kubernetes 기능은 제한됩니다.



### 2. Fargate의 주요 구성 요소

#### 2.1 Fargate Profile

Fargate에서 Pod가 실행되기 위해서는 Fargate Profile이 필요합니다. 이 Profile은 특정 네임스페이스나 선택된 라벨의 Pod에 대해 Fargate를 사용할지 정의합니다.

• namespace: Fargate에서 실행될 Pod의 네임스페이스

• labels: 선택적. 특정 label이 부여된 Pod만 Fargate에서 실행

• subnet: Pod가 실행될 서브넷. 반드시 Private Subnet을 사용해야 함

• executionRoleArn: Fargate Pod가 AWS 리소스에 접근할 수 있도록 허용하는 IAM Role

```
# 예시: Fargate Profile
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-cluster
  region: ap-northeast-2
fargateProfiles:
  - name: default
    selectors:
      - namespace: fargate-app
    subnets:
      - subnet-xxxxxx
    iam:
      roleArn: arn:aws:iam::123456789012:role/eks-fargate-pod-role
```

### 3. Fargate의 실행 환경 및 제약 사항

#### 3.1 VPC 구성

• Fargate는 Pod마다 Elastic Network Interface(ENI)를 생성

• Private Subnet에 ENI가 할당되므로 인터넷 접근을 위해 NAT Gateway 필요

• Subnet 별로 ENI 제한 수가 있으므로 Pod 수 제한에 유의\


#### 3.2 지원/비지원 기능

<table data-header-hidden><thead><tr><th width="349.11328125"></th><th></th></tr></thead><tbody><tr><td>기능 항목</td><td>지원 여부</td></tr><tr><td>EBS 마운트</td><td>❌ 미지원</td></tr><tr><td>HostPath</td><td>❌ 미지원</td></tr><tr><td>DaemonSet</td><td>❌ 미지원</td></tr><tr><td>GMSA (Windows AD 연동)</td><td>❌ 미지원</td></tr><tr><td>Secrets (AWS Secrets Manager / Parameter Store 연동)</td><td>✅ 지원</td></tr><tr><td>IAM Roles for Service Accounts(IRSA)</td><td>✅ 지원</td></tr><tr><td>Fluent Bit 로깅</td><td>✅ 지원 (AWS 공식 Fluent Bit Daemonset 사용 불가 → Sidecar 방식으로 가능)</td></tr></tbody></table>

> ⚠️ 주의: Fargate는 DaemonSet을 사용할 수 없기 때문에, 로그 수집, 모니터링 등의 기능을 Sidecar 또는 App 내 처리 방식으로 대체해야 합니다.\
>

### 4. 비용 구조

Fargate는 사용한 vCPU와 Memory 리소스에 대해 초 단위로 과금합니다.

| 리소스    | 요금 (서울 리전 기준)          |
| ------ | ---------------------- |
| vCPU   | 약 $0.04048 / vCPU-hour |
| Memory | 약 $0.004445 / GB-hour  |

예: 0.25 vCPU, 0.5 GB 메모리를 사용하는 Pod를 1시간 실행한 경우:

• vCPU: 0.25 × $0.04048 = $0.01012

• Memory: 0.5 × $0.004445 = $0.00222

• 합계: $0.01234/hour

> 💡 팁: 일정 이상의 워크로드에는 EC2 기반 노드 그룹이 비용적으로 유리할 수 있습니다. 실시간 워크로드가 적은 경우에 Fargate가 효율적입니다.\
>

### 5. 실무 활용 예시

#### 5.1 간단한 API 서버 배포 예시

```
apiVersion: v1
kind: Namespace
metadata:
  name: fargate-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
  namespace: fargate-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: hello
          image: public.ecr.aws/nginx/nginx:latest
          ports:
            - containerPort: 80
```

#### 5.2 Fargate에 Fluent Bit Sidecar 적용 예시

```
containers:
  - name: app
    image: your-app
  - name: fluent-bit
    image: amazon/aws-for-fluent-bit:latest
    volumeMounts:
      - name: varlog
        mountPath: /var/log
volumes:
  - name: varlog
    emptyDir: {}
```

### 6. Fargate를 사용할 때 고려할 점

• 워크로드가 크고 복잡하거나, Pod 간 통신 및 성능을 정밀하게 조정해야 한다면 EC2 기반 노드가 더 유리

• 보안적으로 격리된 Pod 실행이 필요할 경우 (ex. 팀별 네임스페이스 격리)에는 Fargate가 적합

• 클러스터 자동 확장이나 HPA (Horizontal Pod Autoscaler)와 조합하여 유연한 처리 가능



다음은 앞서 작성한 Fargate 개요 및 설명에 이어지는 7번 이후 실습 중심 내용을 GitBook 스타일로 정리한 전체 흐름입니다.

***

## Fargate Profile 생성 및 구성 실습

### 7.Fargate Profile 생성

7.1 Fargate Profile 생성

eksctl을 사용해 EKS 클러스터에 Fargate Profile을 생성합니다.

```
# Fargate Profile 생성
eksctl create fargateprofile \
  --cluster ${EKSCLUSTER_NAME} \
  --name fargate-game-2048 \
  --namespace fargate-game-2048
```

Fargate에서 실행되는 Pod는 이미지 다운로드 등 AWS 리소스를 사용할 수 있어야 하므로, Fargate Pod 실행 역할(PodExecutionRole)이 필요합니다. 위 명령어 실행 시 자동으로 해당 역할이 생성되며, Fargate Profile이 활성화되기까지 몇 분 정도 소요될 수 있습니다.

Profile 생성 완료 후 아래 명령어로 확인할 수 있습니다.

```
eksctl get fargateprofile \
  --cluster ${EKSCLUSTER_NAME} \
  -o yaml
```

예시 출력

```
- name: fargate-game-2048
  podExecutionRoleARN: arn:aws:iam::410691154611:role/eksctl-eksworkshop-fargate-FargatePodExecutionRole-VC3M4TH43PN2
  selectors:
  - namespace: fargate-game-2048
  status: ACTIVE
  subnets:
  - subnet-044fd202626076e79
  - subnet-046e417018b50d939
  - subnet-0d3c2d117533c431d
```

#### 7.2 Subnet 확인

Fargate는 Private Subnet에서만 실행 가능하며, Public IP는 부여되지 않습니다. EKS를 생성할 때 지정한 VPC에 Private Subnet이 포함되어 있어야 하며, 실습 환경에서는 다음과 같이 확인할 수 있습니다.

```
cat ~/.bash_profile | grep Private
```

```
export PrivateSubnet01=subnet-044fd202626076e79
export PrivateSubnet02=subnet-046e417018b50d939
export PrivateSubnet03=subnet-0d3c2d117533c431d
```

### 8. 애플리케이션 배포 (2048 게임)

8.1 배포 준비

AWS Load Balancer Controller가 클러스터에 설치되어 있어야 외부 서비스 노출이 가능합니다. 이 실습에서는 미리 설치되었다고 가정합니다. ( [EKS Ingress 단원에서 설치](../5.eks-ingress/alb-ingress.md) )

2048 게임 애플리케이션을 배포합니다.

```
kubectl apply -f ~/environment/myeks/alb-controller/fargate_2048_full.yaml
kubectl -n fargate-game-2048 rollout status deployment deployment-2048
```

### 9. Fargate Pod 및 노드 상태 확인

9.1 노드 확인

```
kubectl get nodes -o wide
```

예시 출력

```
NAME                                                      STATUS   ROLES    AGE     VERSION                INTERNAL-IP    ...
fargate-ip-10-11-64-72.ap-northeast-2.compute.internal    Ready    <none>   6m27s   v1.23.17-eks-f4dc2c0   10.11.64.72    ...
fargate-ip-10-11-69-151.ap-northeast-2.compute.internal   Ready    <none>   6m18s   v1.23.17-eks-f4dc2c0   10.11.69.151   ...
...
```

> ✅ Fargate는 Pod당 1개의 Node가 생성됩니다. fargate\_2048\_full.yaml 파일에는 Replica가 5개로 설정되어 있어, 5개의 Fargate 노드가 생성됩니다.

### 10. Replica 수 조정 및 노드 변화 확인

#### 10.1 Pod 및 노드 확인

```
# Pod 확인
kubectl -n fargate-game-2048 get pod -o wide

# Replica 수 줄이기
kubectl -n fargate-game-2048 scale deployment deployment-2048 --replicas=3

# 결과 확인
kubectl -n fargate-game-2048 get pod -o wide
kubectl get nodes -o wide
```

예시 출력

```
NAME                               READY   STATUS    AGE   IP             NODE
deployment-2048-xxx-xxx            1/1     Running   18m   10.11.79.252   fargate-ip-10-11-79-252...
deployment-2048-xxx-xxx            1/1     Running   18m   10.11.93.229   fargate-ip-10-11-93-229...
deployment-2048-xxx-xxx            1/1     Running   18m   10.11.64.72    fargate-ip-10-11-64-72...

kubectl get nodes -o wide
NAME                                STATUS   AGE   INTERNAL-IP     ...
fargate-ip-10-11-64-72...           Ready    17m   10.11.64.72     ...
fargate-ip-10-11-79-252...          Ready    17m   10.11.79.252    ...
fargate-ip-10-11-93-229...          Ready    17m   10.11.93.229    ...
```

### 11. Fargate 리소스 확인 위치

Fargate는 EC2 기반 노드가 아니므로 EC2 콘솔에서 확인할 수 없습니다.&#x20;

다음 위치에서 확인 가능합니다:

• AWS 콘솔 > EKS > Clusters > \[클러스터명] > Compute > Fargate profiles

<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

### 12.Fargate 의 서비스 확인

아래 명령을 통해서 Fargate에 서비스가 정상적으로 배포되었는지 확인합니다.

```
kubectl -n fargate-game-2048 get all,ingress

```

아래에서 ingress 주소를 확인하고 웹브라우저에서 실행 해 봅니다.

```
echo "fargate 2048 Game Ingress 주소는: http://$(kubectl -n fargate-game-2048 get ingress -o jsonpath='{.items[*].status.loadBalancer.ingress[*].hostname}')"

```

