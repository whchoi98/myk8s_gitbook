---
description: 'Update : 2020-11-11'
---

# 노드

## 노드

쿠버네티스는 컨테이너를 파드내에 배치하고 노드에서 실행함으로 워크로드를 수행합니다.  노드는 클러스터에 따라 가상머신 또는 물리적 머신일 수 있습니다.  각 노드에는 [컨트롤 플레인](https://kubernetes.io/ko/docs/reference/glossary/?all=true#term-control-plane)이라는 [파드](https://kubernetes.io/ko/docs/concepts/workloads/pods/pod-overview/)를 실행하는데 필요한 서비스가 포함되어 있다.

일반적으로 클러스터에는 여러 개의 노드가 있으며, 구성 방법에 따라서 한개의 노드로 구성할 수도 있습니다.

각 노드의 [컴포넌트](https://kubernetes.io/ko/docs/concepts/overview/components/#%EB%85%B8%EB%93%9C-%EC%BB%B4%ED%8F%AC%EB%84%8C%ED%8A%B8)에는 [kubelet](https://kubernetes.io/docs/reference/generated/kubelet), [컨테이너 런타임](https://kubernetes.io/docs/setup/production-environment/container-runtimes) 그리고 [kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)가 포함됩니다.

## 관리

[API 서버](https://kubernetes.io/docs/reference/generated/kube-apiserver/)에 노드를 추가하는 두가지 주요 방법이 있습니다.

1. 노드의 kubelet으로 컨트롤 플레인에 등록
2. 사용자 또는 다른 사용자가 노드 오브젝트를 수동으로 추가

노드 오브젝트 또는 노드의 kubelet으로 등록한 후 컨트롤 플레인은 새 노드 오브젝트가 유효한지 확인합니다.  예를 들어 다음 JSON 매니페스트에서 노드를 만들려는 경우입니다.

```text
{
  "kind": "Node",
  "apiVersion": "v1",
  "metadata": {
    "name": "10.240.79.157",
    "labels": {
      "name": "my-first-k8s-node"
    }
  }
}
```

쿠버네티스는 내부적으로 노드 오브젝트를 생성합니다.  쿠버네티스는 kubelet이 노드의 `metadata.name` 필드와 일치하는 API 서버에 등록이 되어있는지 확인합니다.  노드가 정상이면\(필요한 모든 서비스가 실행중인 경우\) 파드를 실행할 수 있게 됩니다.  그렇지 않으면, 해당 노드는 정상이 될때까지 모든 클러스터 활동에 대해 무시됩니다.

{% hint style="info" %}
**참고:** 쿠버네티스는 유효하지 않은 노드 오브젝트를 유지하고, 노드가 정상적인지 확인합니다.  상태 확인을 중지하려면 사용자 또는 [컨트롤러](https://kubernetes.io/ko/docs/concepts/architecture/controller/)에서 노드 오브젝트를 명시적으로 삭제해야 합니다.
{% endhint %}

### 노드에 대한 자체-등록

kubelet 플래그 `--register-node`는 참\(기본값\)일 경우, kubelet 은 API 서버에 스스로 등록을 시도할 것입니다. 이것는 대부분의 배포판에 의해 이용되는, 선호하는 패턴입니다.

자체-등록에 대해, kubelet은 다음 옵션과 함께 시작됩니다.

* `--kubeconfig` - apiserver에 스스로 인증하기 위한 자격증명에 대한 경로.
* `--cloud-provider` - 자신에 대한 메터데이터를 읽기 위해 어떻게 [클라우드 제공자](https://kubernetes.io/docs/concepts/cluster-administration/cloud-providers)와 통신할 것인지에 대한 방법 .
* `--register-node` - 자동으로 API 서버에 등록.
* `--register-with-taints` - 주어진 [테인트\(taint\)](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) 리스트\(콤마로 분리된 `<key>=<value>:<effect>`\)를 가진 노드 등록.

  `register-node`가 거짓이면 동작 안 함.

* `--node-ip` - 노드의 IP 주소.
* `--node-labels` - 클러스터에 노드를 등록할 때 추가 할 [레이블](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/labels)\([NodeRestriction admission plugin](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction)에 의해 적용되는 레이블 제한 사항 참고\).
* `--node-status-update-frequency` - 얼마나 자주 kubelet이 마스터에 노드 상태를 게시할 지 정의.

[Node authorization mode](https://kubernetes.io/docs/reference/access-authn-authz/node/)와 [NodeRestriction admission plugin](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction)이 활성화 되면, kubelets 은 자신의 노드 리소스를 생성/수정할 권한을 가집니다.

**수동 노드 관리**

[kubectl](https://kubernetes.io/docs/user-guide/kubectl-overview/)을 사용해서 노드 오브젝트를 생성하고 수정할 수 있습니다.

노드 오브젝트를 수동으로 생성하려면 kubelet 플래그를 `--register-node=false` 로 설정합니다.

`--register-node` 설정과 관계 없이 노드 오브젝트를 수정할 수 있습니다 . 예를 들어 기존 노드에 레이블을 설정하거나, 스케줄 불가로 표시할 수 있습니다.

파드의 노드 셀렉터와 함께 노드의 레이블을 사용해서 스케줄링을 제어할 수 있습니다 . 예를 들어, 사용 가능한 노드의 하위 집합에서만 실행되도록 파드를 제한할 수 있습니다.

노드를 스케줄 불가로 표시하면 스케줄러가 해당 노드에 새 파드를 배치할 수 없지만, 노드에 있는 기존 파드에는 영향을 미치지 않습니다 . 이는 노드 재부팅 또는 기타 유지보수 준비 단계에서 유용합니다.

노드를 스케줄 불가로 표시하려면 다음을 실행합니다.

```text
kubectl cordon $NODENAME
```

{% hint style="info" %}
참고: [데몬셋\(DaemonSet\)](https://kubernetes.io/ko/docs/concepts/workloads/controllers/daemonset)에 포함되는 일부 파드는 스케줄 불가능한 노드에서 실행될 수 있습니다 . 일반적으로 데몬셋은 워크로드 애플리케이션이 없는 경우에도 노드에서 실행되어야 하는 노드 로컬 서비스를 제공합니다 .
{% endhint %}

노드 오브젝트의 이름은 유효한 [DNS 서브도메인 이름](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/names/#dns-%EC%84%9C%EB%B8%8C%EB%8F%84%EB%A9%94%EC%9D%B8-%EC%9D%B4%EB%A6%84%EB%93%A4)이어야 합니다.

## 노드 상태

노드의 상태는 다음의 정보를 포함합니다.

* [주소](https://kubernetes.io/ko/docs/concepts/architecture/nodes/#addresses)
* [컨디션](https://kubernetes.io/ko/docs/concepts/architecture/nodes/#condition)
* [용량과 할당가능](https://kubernetes.io/ko/docs/concepts/architecture/nodes/#capacity)
* [정보](https://kubernetes.io/ko/docs/concepts/architecture/nodes/#info)

`kubectl` 을 사용해서 노드 상태와 기타 세부 정보를 볼수 있습니다.

```text
kubectl describe node <insert-node-name-here>
```

출력되는 각 섹션은 아래에 설명되어 있습니다.

### 주소

이 필드 클라우드 제공사업자 또는 베어메탈 구성에 따라 다양합니다.

* HostName: 노드의 커널에 의해 알려진 호스트명입니다. `--hostname-override` 파라미터를 통해 치환될 수 있습니다.
* ExternalIP: 일반적으로 노드의 IP 주소는 외부로 라우트 가능 \(클러스터 외부에서 이용 가능\) 합니다.
* InternalIP: 일반적으로 노드의 IP 주소는 클러스터 내에서만 라우트 가능합니다.

### 컨디션

`conditions` 필드는 모든 `Running` 노드의 상태를 기술한다. 컨디션의 예로 다음을 포함합니다.

| 노드 컨디션 | 설명 |
| :--- | :--- |
| `Ready` | 노드가 상태 양호하며 파드를 수용할 준비가 되어 있는 경우 `True`, 노드의 상태가 불량하여 파드를 수용하지 못할 경우 `False`, 그리고 노드 컨트롤러가 마지막 `node-monitor-grace-period` \(기본값 40 기간 동안 노드로부터 응답을 받지 못한 경우\) `Unknown` |
| `DiskPressure` | 디스크 사이즈 상에 압박이 있는 경우, 즉 디스크 용량이 넉넉치 않은 경우 `True`, 반대의 경우 `False` |
| `MemoryPressure` | 노드 메모리 상에 압박이 있는 경우, 즉 노드 메모리가 넉넉치 않은 경우 `True`, 반대의 경우 `False` |
| `PIDPressure` | 프로세스 상에 압박이 있는 경우, 즉 노드 상에 많은 프로세스들이 존재하는 경우 `True`, 반대의 경우 `False` |
| `NetworkUnavailable` | 노드에 대해 네트워크가 올바르게 구성되지 않은 경우 `True`, 반대의 경우 `False` |

{% hint style="info" %}
참고 : 커맨드 라인 도구를 사용해서 코드화된 노드의 세부 정보를 출력하는 경우 조건에는 `SchedulingDisabled` 이 포함됩니다.`SchedulingDisabled` 은 쿠버네티스 API의 조건이 아니며, 대신 코드화된 노드는 사양에 스케줄 불가로 표시됩니다.
{% endhint %}

노드 컨디션은 JSON 오브젝트 형태로 표현됩니다. 예를 들어, 다음 응답은 상태 양호한 노드를 나타냅니다.

```text
"conditions": [
  {
    "type": "Ready",
    "status": "True",
    "reason": "KubeletReady",
    "message": "kubelet is posting ready status",
    "lastHeartbeatTime": "2019-06-05T18:38:35Z",
    "lastTransitionTime": "2019-06-05T11:41:27Z"
  }
]
```

ready 컨디션의 상태가 `pod-eviction-timeout` \([kube-controller-manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)에 전달된 인수\) 보다 더 길게 `Unknown` 또는 `False`로 유지되는 경우, 노드 상에 모든 파드는 노드 컨트롤러에 의해 삭제 되도록 스케줄 됩니다.  기본 타임아웃 기간은 **5분** 이입니다. 노드에 접근이 불가할 때와 같은 경우, apiserver는 노드 상의 kubelet과 통신이 불가능 합니다.apiserver와의 통신이 재개될 때까지 파드 삭제에 대한 결정은 kubelet에 전해질 수 없습니. 그  사이, 삭제되도록 스케줄 되어진 파드는 분할된 노드 상에서 계속 동작할 수도 있습니다.

노드 컨트롤러가 클러스터 내 동작 중지된 것을 확신할 때까지는 파드를 강제로 삭제하지 않습니. 파드가 `Terminating` 또는 `Unknown` 상태로 있을 때 접근 불가한 노드 상에서 동작되고 있는 것을 보게 될 수도 있습니다.노드가 영구적으로 클러스터에서 삭제되었는지에 대한 여부를 쿠버네티스가 해당 인프라로 부터 유추할 수 없는 경우, 노드가 클러스터를 영구적으로 탈퇴하게 되면, 클러스터 관리자는 직 노드 오브젝트를 삭제해야 할 수도 있습니다. 쿠버네티스에서 노드 오브젝트를 삭제하면 노드 상에서 동작중인 모든 파드 오브젝트가 apiserver로부터 삭제되어 그 이름을 사용할 수 있는 결과를 초래할 수도 있습니다.

노드 수명주기 컨트롤러는 자동으로 컨디션을 나타내는 [테인트\(taints\)](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)를 생성합니. 스케줄러는 파드를 노드에 할당 할 때 노드의 테인트를 고려합니다. 또한 파드는 노드의 테인트를 극복\(tolerate\)할 수 있는 톨러레이션\(toleration\)을 가질 수 있다.

자세한 내용은 [컨디션별 노드 테인트하기](https://kubernetes.io/ko/docs/concepts/scheduling-eviction/taint-and-toleration/#%EC%BB%A8%EB%94%94%EC%85%98%EB%B3%84-%EB%85%B8%EB%93%9C-%ED%85%8C%EC%9D%B8%ED%8A%B8%ED%95%98%EA%B8%B0)를 참조합니다.

### 용량과 할당가능

노드 상에 사용 가능한 리소스를 나타낸다. 리소스에는 CPU, 메모리 그리고 노드 상으로 스케줄 되어질 수 있는 최대 파드 수가 있습니다.

용량 블록의 필드는 노드에 있는 리소스의 총량을 나타냅니. 할당가능 블록은 일반 파드에서 사용할 수 있는 노드의 리소스 양을 나타냅니다.

### 정보

커널 버전, 쿠버네티스 버전 \(kubelet과 kube-proxy 버전\), \(사용하는 경우\) Docker 버전, OS 이름과 같은노드에 대한 일반적인 정보를 보여 줍니. 이 정보는 Kubelet에 의해 노드로부터 수집됩니다.

### 노드 컨트롤러

노드 [컨트롤러](https://kubernetes.io/ko/docs/concepts/architecture/controller/)는 노드의 다양한 측면을 관리하는 쿠버네티스 컨트롤 플레인 컴포넌트입니다.

노드 컨트롤러는 노드가 생성되어 유지되는 동안 다양한 역할을 합니. 첫째는 등록 시점에 \(CIDR 할당이 사용토록 설정된 경우\) 노드에 CIDR 블럭을 할당하는 것입니다.

두 번째는 노드 컨트롤러의 내부 노드 리스트를 클라우드 제공사업자의 사용 가능한 머신 리스트 정보를 근거로 최신상태로 유지하는 것입니다. 클라우드 환경에서 동작 중일 경우, 노드상태가 불량할 때마다, 노드 컨트롤러는 해당 노드용 VM이 여전히 사용 가능한지에 대해 클라우드 제공사업자에게 질의합니다. 사용 가능하지 않을 경우, 노드 컨트롤러는 노드 리스트로부터 그 노드를 삭제합니다.

세 번째는 노드의 동작 상태를 모니터링 하는 것입니. 노드 컨트롤러는 노드가 접근 불가할 경우 \(즉 노드 컨트롤러가 어떠한 사유로 하트비트 수신을 중지하는 경우, 예를 들어 노드 다운과 같은 경우이다.\) NodeStatus의 NodeReady 컨디션을 ConditionUnknown으로 업데이트 하는 책임을 지고, 노드가 계속 접근 불가할 경우 나중에 노드로부터 \(정상적인 종료를 이용하여\) 모든 파드를 축출시킵니다. ConditionUnknown을 알리기 시작하는 기본 타임아웃 값은 40초 이고, 파드를 축출하기 시작하는 값은 5분이다.\) 노드 컨트롤러는 매 `--node-monitor-period` 초 마다 각 노드의 상태를 체크합니다.

### **하트비트**

쿠버네티스 노드에서 보내는 하트비트는 노드의 가용성을 결정하는데 도움이 됩니다.

하트비트의 두 가지 형태는 `NodeStatus` 와 [리스\(Lease\) 오브젝트](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#lease-v1-coordination-k8s-io) 입니. 각 노드에는 `kube-node-lease` 라는 [네임스페이스](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/namespaces) 에 관련된 리스 오브젝트가 있습니. 리스는 경량 리소스로, 클러스터가 확장될 때 노드의 하트비트 성능을 향상 시킵니다.

kubelet은 `NodeStatus` 와 리스 오브젝트를 생성하고 업데이트 할 의무가 있습니.

* kubelet은 상태가 변경되거나 구성된 상태에 대한 업데이트가 없는 경우, `NodeStatus` 를 업데이트 합니. `NodeStatus` 의 기본 업데이트 주기는 5분니다. \(연결할 수 없는 노드의 시간 제한인 40초 보다 훨씬 깁니다.\)
* kubelet은 10초마다 리스 오브젝트를 생성하고 업데이트 합니다.\(기본 업데이트 주기\). 리스 업데이트는 `NodeStatus` 업데이트와는 독립적으로 발생합니. 리스 업데이트가 실패하면 kubelet에 의해 재시도하며 7초로 제한된 지수 백오프를 200 밀리초에서 부터 시작합니다.

### **안정성**

대부분의 경우, 노드 컨트롤러는 초당 `--node-eviction-rate`\(기본값 0.1\)로 축출 비율을 제한합니다 . 이 것은 10초당 1개의 노드를 초과하여 파드 축출을 하지 않는다는 의미가 됩니다.

노드 축출 행위는 주어진 가용성 영역 내 하나의 노드가 상태가 불량할 경우 변화합니다 . 노드 컨트롤러는 영역 내 동시에 상태가 불량한 노드의 퍼센티지가 얼마나 되는지 체크합니다.\(NodeReady 컨디션은 ConditionUnknown 또는 ConditionFalse \). 상태가 불량한 노드의 일부가 최소 `--unhealthy-zone-threshold` 기본값 0.55\) 가 되면 제 비율은 감소합니다.

 클러스터가 작으면 \(즉 `--large-cluster-size-threshold` 노드 이하면 - 기본값 50\) 축출은 중지되고, 그렇지 않으면 축출 비율은 초당 `--secondary-node-eviction-rate`\(기본값 0.01\)로 감소됩니다 . 이 정책들이 가용성 영역 단위로 실행되어지는 이유는 나머지가 연결되어 있는 동안 하나의 가용성 영역이 마스터로부터 분할되어 질 수도 있기 때문입니다. 만약 클러스터가 여러 클라우드 제공사업자의 가용성 영역에 걸쳐 있지 않으면, 오직 하나의 가용성 영역만 \(전체 클러스터\) 존재하게 됩니다.

노드가 가용성 영역들에 걸쳐 퍼져 있는 주된 이유는 하나의 전체 영역이 장애가 발생할 경우 워크로드가 상태 양호한 영역으로 이동하기 위함입니다 . 그러므로, 하나의 영역 내 모든 노드들이 상태가 불량하면 노드 컨트롤러는 `--node-eviction-rate` 의 정상 비율로 축합니다. 

코너 케이스란 모든 영역이 완전히 상태불량 \(즉 클러스터 내 양호한 노드가 없는 경우\) 한 경우입니다 . 이러한 경우, 노드 컨트롤러는 마스터 연결에 문제가 있어 일부 연결이 복원될 때까지 모든 제을 중지하는 것으로 간주합니다.

또한, 노드 컨트롤러는 파드가 테인트를 허용하지 않을 때 `NoExecute` 테인트 상태의 노드에서 동작하는 파드에 대한 제 책임을 가지고 있습니다. 추가로, 노드 컨틀로러는 연결할 수 없거나, 준비되지 않은 노드와 같은 노드 문제에 상응하는 [테인트](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)를 추가합니. 이것 스케줄러가 비정상적인 노드에 파드를 배치하지 않게 됩니다.

{% hint style="warning" %}
주의 : `kubectl cordon` 은 노드를 'unschedulable'로 표기하는데, 이는 서비스 컨트롤러가 이전에 자격 있는 로드밸런서 노드 대상 목록에서 해당 노드를 제거하기에 사실상 cordon 된 노드에서 들어오는 로드 밸런서 트래픽을 제거하는 부작용을 갖습니다.
{% endhint %}

### 노드 용량

노드 오브젝트는 노드 리소스 용량에 대한 정보\(예: 사용 가능한 메모리의 양과 CPU의 수\)를 추적합니. 노드의 [자체 등록](https://kubernetes.io/ko/docs/concepts/architecture/nodes/#%EB%85%B8%EB%93%9C%EC%97%90-%EB%8C%80%ED%95%9C-%EC%9E%90%EC%B2%B4-%EB%93%B1%EB%A1%9D)은 등록하는 중에 용량을 보고합니. [수동](https://kubernetes.io/ko/docs/concepts/architecture/nodes/#%EC%88%98%EB%8F%99-%EB%85%B8%EB%93%9C-%EA%B4%80%EB%A6%AC)으로 노드를 추가하는 경우 추가할 때 노드의 용량 정보를 설정해야 합니다 .

쿠버네티스 [스케줄러](https://kubernetes.io/docs/reference/generated/kube-scheduler/)는 노드 상에 모든 노드에 대해 충분한 리소스가 존재하도록 보장합니다. 스케줄러는 노드 상에 컨테이너에 대한 요청의 합이 노드 용량보다 더 크지 않도록 체크합니. 요청의 합은 kubelet에서 관리하는 모든 컨테이너를 포함하지만, 컨테이너 런타임에 의해 직접적으로 시작된 컨 테이너는 제외되고 kubelet의 컨트롤 범위 밖에서 실행되는 모든 프로세스도 제외됩니다.

{% hint style="info" %}
**참고:** 파드 형태가 아닌 프로세스에 대해 명시적으로 리소스를 확보하려면, [시스템 데몬에 사용할 리소스 예약하기](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#system-reserved)을 참조합니다.
{% endhint %}

## 노드 토폴로지

**FEATURE STATE:** `Kubernetes v1.16 [alpha]`

`TopologyManager` [기능 게이트\(feature gate\)](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)를 활성화 시켜두면, kubelet이 리소스 할당 결정을 할 때 토폴로지 힌트를 사용할 수 있습니. 자세한 내용은 [노드의 컨트롤 토폴로지 관리 정책](https://kubernetes.io/docs/tasks/administer-cluster/topology-manager/)을 참조합니다.

