---
description: 'Update : 2025.01.30'
---

# Scaling Application

## 1. 애플리케이션 확장 (Scaling Application)

Karpenter는 스케일링이 필요한 `unscheduled pods`를 감지하여 자동으로 노드를 확장하는 기능을 제공합니다. Kubernetes에서 새로운 Pod이 생성되면, Kubernetes 스케줄러는 해당 Pod을 클러스터 내 노드에 배치합니다. 하지만 적절한 노드가 없을 경우, 해당 Pod은 `Pending` 상태로 남아있게 됩니다.\
Karpenter는 이러한 Pending 상태의 Pod을 감지하고, 요구 사항을 충족하는 새로운 노드를 프로비저닝하여 클러스터를 자동으로 확장합니다.\
이 실습에서는 Karpenter가 애플리케이션 Pod을 스케일링할 때 새로운 노드를 생성하는 과정을 실습해 보겠습니다.

***

### 1.1 애플리케이션 배포 (Deploy Application)

#### ① 애플리케이션 배포 (0개의 Replica)

먼저, Replica 수가 `0`인 상태에서 애플리케이션을 배포합니다.\
아래 `Deployment` 스펙을 확인하면 다음과 같은 설정이 포함되어 있습니다.

* 컨테이너의 CPU 요청 값이 `1`로 설정됨
* `NodePool`을 생성할 때 추가한 `"eks-team: my-team"` 레이블을 `nodeSelector`에 추가\
  → 이렇게 설정하면, 생성된 Pod이 항상 Karpenter `NodePool`에서 생성한 노드에 할당됩니다.

아래 명령어를 실행하여 애플리케이션을 배포합니다.

```sh
# Karpenter 관련 작업 디렉토리로 이동
cd ~/environment/karpenter
kubectl create namespace karpenter-test

# basic-deploy.yaml 파일을 생성
cat <<EoF> ~/environment/karpenter/basic-deploy.yaml
apiVersion: apps/v1  # API 버전 (Kubernetes에서 사용되는 앱 관련 리소스 정의)
kind: Deployment  # 배포(Deployment) 리소스 정의
metadata:
  name: inflate  # Deployment 이름 지정
  namespace: karpenter-test  # Deployment가 배포될 네임스페이스 지정
spec:
  replicas: 0  # 초기 Pod 개수를 0으로 설정 (추후 스케일링할 예정)
  selector:
    matchLabels:
      app: inflate  # 이 Deployment가 관리할 Pod의 label 지정
  template:  # Pod 템플릿 정의
    metadata:
      labels:
        app: inflate  # Pod에 부여될 label (Deployment의 selector와 일치해야 함)
    spec:
      terminationGracePeriodSeconds: 0  # Pod 종료 대기 시간을 0으로 설정 (즉시 종료)
      containers:
        - name: inflate  # 컨테이너 이름
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.7  # 컨테이너 이미지 지정 (pause 컨테이너 사용)
          resources:
            requests:
              cpu: 1  # 이 컨테이너가 요청하는 CPU 리소스 (1 vCPU)
      nodeSelector:
        eks-team: my-team  # 특정 노드 그룹(eks-team: my-team)에서만 실행되도록 지정
EoF

# 위에서 생성한 Deployment를 Kubernetes 클러스터에 적용
kubectl apply -f ~/environment/karpenter/basic-deploy.yaml

```

**실행 결과**

```sh
deployment.apps/inflate created
```

#### ② 현재 클러스터의 노드 확인

현재 EKS 클러스터에 존재하는 노드를 확인하기 위해 새로운 터미널을 열어 `eks-node-viewer`를 실행합니다.\
아래 명령어를 실행하여 현재 클러스터에 몇 개의 노드가 있는지 확인합니다.

```sh
kubectl get -n nodes
```

앞서 생성한 eks-node-viewer, kube-ops-view, k9s 등을 활용합니다.

***

### 1.2 애플리케이션 확장 (Scale Application)

#### ① Deployment 스케일링 (0 → 5 Replicas)

이제 Deployment를 5개의 Replica로 확장합니다.

```sh
kubectl scale deployment -n karpenter-test inflate --replicas 5
```

**실행 결과**

```sh
deployment.apps/inflate scaled
```

#### ② Karpenter 로그 확인

이제 Karpenter가 새로운 노드를 프로비저닝하는지 확인하기 위해 로그를 조회합니다.\
아래 명령어를 실행하여 `karpenter controller` Pod의 로그에서 `launched nodeclaim` 메시지를 확인합니다.

```sh
kubectl -n $KARPENTER_NAMESPACE logs deployment/karpenter --all-containers=true --since=10m
```

```
kubectl -n $KARPENTER_NAMESPACE logs deployment/karpenter --all-containers=true | grep "launched nodeclaim" | jq -s
```

**로그 예시 (출력 결과)**

```json
Found 2 pods, using pod/karpenter-77c95d88bd-g7xdw
[
  {
    "level": "INFO",
    "time": "2025-02-03T16:35:32.978Z",
    "logger": "controller",
    "message": "launched nodeclaim",
    "commit": "1bfdc3d",
    "controller": "nodeclaim.lifecycle",
    "controllerGroup": "karpenter.sh",
    "controllerKind": "NodeClaim",
    "NodeClaim": {
      "name": "default-g7qbz"
    },
    "namespace": "",
    "name": "default-g7qbz",
    "reconcileID": "a581f48d-3b9d-4051-92a7-6bcc973e0f9c",
    "provider-id": "aws:///ap-northeast-2b/i-02fd00557cbb11e88",
    "instance-type": "c5a.2xlarge",
    "zone": "ap-northeast-2b",
    "capacity-type": "on-demand",
    "allocatable": {
      "cpu": "7910m",
      "ephemeral-storage": "17Gi",
      "memory": "14162Mi",
      "pods": "58",
      "vpc.amazonaws.com/pod-eni": "38"
    }
  }
]
```

***

#### ③ 새로운 노드 생성 확인

`eks-node-viewer`를 실행 중인 터미널을 보면, `c6a.2xlarge` 인스턴스 타입의 새로운 노드가 생성된 것을 확인할 수 있습니다.\
또한 아래 명령어를 실행하여 Karpenter `NodePool`에서 관리하는 노드 목록을 조회할 수 있습니다.

```sh
kubectl get nodes -l eks-team=my-team
```

**출력 예시**

```sh
$ kubectl get nodes -l eks-team=my-team
NAME                                             STATUS   ROLES    AGE     VERSION
ip-10-11-72-27.ap-northeast-2.compute.internal   Ready    <none>   7m29s   v1.29.12-eks-aeac579
```

이제 Karpenter가 새롭게 프로비저닝한 노드가 추가되었음을 확인할 수 있습니다.

#### ④ 새롭게 추가된 노드에 Pod이 배치됨 확인

아래 `k9s` 또는 `kubectl`을 사용하여 새롭게 생성된 노드에 몇 개의 Pod이 배치되었는지 확인할 수 있습니다.

```sh
kubectl get pods -n karpenter-test -o wide
```

새롭게 생성된 노드에는 5개의 Pod이 배치된 것을 확인할 수 있습니다.

#### ⑤ Karpenter가 생성한 노드의 메타데이터 확인

아래 명령어를 실행하여 Karpenter가 생성한 노드에 추가된 메타데이터(레이블)를 조회할 수 있습니다.

```sh
kubectl get node -l eks-team=my-team -o json | jq -r '.items[0].metadata.labels'
```

**출력 예시**

```json
{
  "beta.kubernetes.io/arch": "amd64",
  "beta.kubernetes.io/instance-type": "c5a.2xlarge",
  "beta.kubernetes.io/os": "linux",
  "eks-team": "my-team",
  "failure-domain.beta.kubernetes.io/region": "ap-northeast-2",
  "failure-domain.beta.kubernetes.io/zone": "ap-northeast-2b",
  "k8s.io/cloud-provider-aws": "cb04bd7d485bcb7826560d19bab33a1b",
  "karpenter.k8s.aws/ec2nodeclass": "default",
  "karpenter.k8s.aws/instance-category": "c",
  "karpenter.k8s.aws/instance-cpu": "8",
  "karpenter.k8s.aws/instance-cpu-manufacturer": "amd",
  "karpenter.k8s.aws/instance-cpu-sustained-clock-speed-mhz": "3300",
  "karpenter.k8s.aws/instance-ebs-bandwidth": "3170",
  "karpenter.k8s.aws/instance-encryption-in-transit-supported": "true",
  "karpenter.k8s.aws/instance-family": "c5a",
  "karpenter.k8s.aws/instance-generation": "5",
  "karpenter.k8s.aws/instance-hypervisor": "nitro",
  "karpenter.k8s.aws/instance-memory": "16384",
  "karpenter.k8s.aws/instance-network-bandwidth": "2500",
  "karpenter.k8s.aws/instance-size": "2xlarge",
  "karpenter.sh/capacity-type": "on-demand",
  "karpenter.sh/initialized": "true",
  "karpenter.sh/nodepool": "default",
  "karpenter.sh/registered": "true",
  "kubernetes.io/arch": "amd64",
  "kubernetes.io/hostname": "ip-10-11-72-27.ap-northeast-2.compute.internal",
  "kubernetes.io/os": "linux",
  "node.kubernetes.io/instance-type": "c5a.2xlarge",
  "topology.ebs.csi.aws.com/zone": "ap-northeast-2b",
  "topology.k8s.aws/zone-id": "apne2-az2",
  "topology.kubernetes.io/region": "ap-northeast-2",
  "topology.kubernetes.io/zone": "ap-northeast-2b"
}
```

***

### 1.3 정리

이번 실습을 통해 Karpenter가 `Pending` 상태의 Pod을 감지하여 새로운 노드를 자동으로 생성하는 과정을 확인하였습니다.\
이를 통해 **Karpenter를 활용한 Kubernetes 클러스터의 자동 확장 기능을 효과적으로 운영할 수 있습니다.** 🚀
