---
description: 'Update : 2025.01.29'
---

# Pod 오버헤드 (Pod Overhead) (작업중)

### Pod 오버헤드 (Pod Overhead)

#### 개요

Kubernetes에서 Pod이 Node에서 실행될 때, Pod 자체가 시스템 리소스를 소비합니다. 이러한 리소스는 Pod 내부에서 실행되는 컨테이너가 필요로 하는 리소스 외에도 추가로 요구됩니다. **Pod Overhead**는 컨테이너의 요청 및 제한 사항 외에도 Pod 인프라가 소비하는 리소스를 고려하는 방식입니다.

Kubernetes에서는 Pod의 오버헤드가 **Admission Time(승인 시점)** 에 설정되며, 이는 해당 Pod의 **RuntimeClass**에 연결된 오버헤드 값에 따라 결정됩니다.

Pod의 오버헤드는 다음과 같은 경우에 고려됩니다:

* **Pod 스케줄링**: Pod의 컨테이너 요청 합계에 오버헤드를 추가하여 스케줄링에 반영됨
* **Pod Cgroup 크기 조정**: kubelet이 Pod의 Cgroup을 크기 조정할 때 오버헤드를 포함
* **Pod 축출(Eviction) 순위 결정**: 오버헤드를 포함한 전체 리소스 사용량을 고려하여 우선순위를 평가

***

### Pod 오버헤드 설정하기

Pod 오버헤드를 활용하려면 **overhead 필드가 정의된 RuntimeClass**를 사용해야 합니다.

#### 사용 예제

아래는 **Kata Containers** 및 **Firecracker 가상 머신 모니터**를 사용하는 컨테이너 런타임 예제입니다. 이 경우, Pod당 약 **120MiB**의 메모리가 VM과 게스트 OS를 위해 사용됩니다.

```yaml
mkdir -p ~/environment/pod_overhead
cat <<EoF > ~/environment/pod_overhead/kata-fc.yaml
apiVersion: node.k8s.io/v1  # API 버전: node.k8s.io/v1
kind: RuntimeClass  # 객체 유형: RuntimeClass (Pod의 실행 환경을 정의)

metadata:
  name: kata-fc  # RuntimeClass 이름 (Pod에서 runtimeClassName으로 지정 가능)

handler: kata-fc  # 컨테이너 런타임 핸들러 (Kata Containers - Firecracker 기반)

overhead:
  podFixed:  # Pod에 고정적으로 추가될 오버헤드 리소스 정의
    memory: "120Mi"  # Pod 실행을 위한 추가적인 메모리 오버헤드 (120MiB)
    cpu: "250m"  # Pod 실행을 위한 추가적인 CPU 오버헤드 (250 millicores = 0.25 vCPU)
EoF
kubectl apply -f ~/environment/pod_overhead/kata-fc.yaml

```

* 위의 RuntimeClass를 사용하면, `kata-fc`를 지정한 Pod은 **리소스 쿼터 계산**, **노드 스케줄링**, **Pod Cgroup 크기 조정** 시 `memory: 120Mi`, `cpu: 250m`의 오버헤드를 반영하게 됩니다.

#### 예제 설명:

* **`RuntimeClass`란?**
  * Kubernetes에서 Pod의 실행 환경을 지정하는 리소스로, 특정 컨테이너 런타임을 사용할 수 있도록 정의함.
  * 이 예제에서는 **Kata Containers + Firecracker** 기반의 가상화 컨테이너 런타임을 사용하도록 설정.
* **`handler: kata-fc`**
  * 실제 컨테이너 런타임과 매칭되는 핸들러 이름.
  * `kata-fc`는 Kata Containers와 Firecracker 기반의 가상화된 컨테이너 실행을 의미.
* **`overhead.podFixed`**
  * Kubernetes에서 **Pod 자체가 소비하는 리소스를 정의**.
  * 이 RuntimeClass를 사용하는 모든 Pod은 **추가적으로 CPU 250m, 메모리 120MiB**를 소비함.
  * 이는 가상 머신 기반 컨테이너 런타임의 **VM, 게스트 OS 운영에 필요한 리소스**를 반영하기 위함.

#### 예제 Pod 정의

다음과 같은 Pod을 정의하면, `kata-fc` RuntimeClass를 사용하여 Pod 오버헤드를 자동으로 적용할 수 있습니다.

```yaml
cat <<EoF > ~/environment/pod_overhead/podoverhead_sample.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod  # Pod 이름 지정
spec:
  runtimeClassName: kata-fc  # 이 Pod은 'kata-fc' RuntimeClass를 사용하여 실행됨 (가상화 컨테이너 런타임)
  containers:
  - name: busybox-ctr  # 첫 번째 컨테이너: busybox
    image: busybox:1.28  # busybox 컨테이너 이미지 사용
    stdin: true  # 표준 입력을 활성화하여 인터랙티브 셸 실행 가능
    tty: true  # TTY 지원 활성화
    resources:
      limits:
        cpu: 500m  # 이 컨테이너가 사용할 최대 CPU 제한 (500 millicores = 0.5 vCPU)
        memory: 100Mi  # 최대 메모리 제한 (100MiB)
  - name: nginx-ctr  # 두 번째 컨테이너: nginx
    image: nginx  # nginx 컨테이너 이미지 사용
    resources:
      limits:
        cpu: 1500m  # 이 컨테이너가 사용할 최대 CPU 제한 (1500 millicores = 1.5 vCPU)
        memory: 100Mi  # 최대 메모리 제한 (100MiB)
EoF
kubectl apply -f ~/environment/pod_overhead/podoverhead_sample.yaml

```

* 위 Pod의 컨테이너 리소스 제한값(Limit):
  * busybox-ctr: `CPU: 500m, Memory: 100Mi`
  * nginx-ctr: `CPU: 1500m, Memory: 100Mi`
  * 총합: `CPU: 2000m, Memory: 200Mi`
* `kata-fc` RuntimeClass로 인해 추가되는 오버헤드:
  * `CPU: 250m`, `Memory: 120Mi`
  * 최종적으로 **스케줄러가 고려하는 총 리소스 요구량**:\
    `CPU: 2250m`, `Memory: 320Mi`

`예제 설명`

* **RuntimeClass 사용 (`runtimeClassName: kata-fc`)**\
  이 Pod은 `kata-fc`라는 **RuntimeClass**를 사용하므로, Kubernetes는 이 Pod의 **오버헤드**를 고려하여 추가적인 리소스를 할당합니다.
* **두 개의 컨테이너 포함**
  * `busybox-ctr`: 간단한 BusyBox 컨테이너로, TTY 지원을 활성화하여 인터랙티브한 셸을 사용할 수 있음.
  * `nginx-ctr`: Nginx 웹 서버를 실행하는 컨테이너.
* **리소스 제한 (`limits`) 적용**
  * `cpu`와 `memory`의 상한을 지정하여 특정한 리소스 내에서 실행되도록 제어.
  * `kata-fc` RuntimeClass가 적용되면, 추가적으로 **CPU 250m, 메모리 120Mi**가 Pod 오버헤드로 반영됨.

***

### Pod 오버헤드 확인

Pod이 승인되면, **RuntimeClass Admission Controller**가 PodSpec을 수정하여 오버헤드를 포함합니다.\
PodSpec에 이미 `overhead` 필드가 정의되어 있으면, 해당 Pod은 **거부됨**.

오버헤드가 적용된 Pod을 확인하려면 다음 명령어를 실행합니다:

```sh
kubectl get pod test-pod -o jsonpath='{.spec.overhead}'
```

출력 결과:

```
map[cpu:250m memory:120Mi]
```

즉, `CPU: 250m`, `Memory: 120Mi`의 오버헤드가 적용되었음을 확인할 수 있습니다.

***

### 스케줄러에서 Pod 오버헤드 반영

스케줄러는 **컨테이너 요청량 + 오버헤드**를 합산하여 적절한 노드를 찾습니다.

```sh
kubectl describe node | grep test-pod -B2
```

출력 예제:

```
Namespace    Name       CPU Requests  CPU Limits   Memory Requests  Memory Limits  AGE
---------    ----       ------------  ----------   ---------------  -------------  ---
default      test-pod   2250m (56%)   2250m (56%)  320Mi (1%)       320Mi (1%)     36m
```

즉, 노드에 반영된 총 요청량:

* `CPU: 2250m`
* `Memory: 320Mi`

***

### Pod Cgroup 제한값 확인

Pod이 노드에 스케줄링되면, kubelet이 해당 Pod을 위한 **Cgroup을 생성**합니다.\
Cgroup의 `memory.limit_in_bytes` 설정값을 확인하면, 오버헤드가 반영된 값을 볼 수 있습니다.

1.  **Pod의 ID 확인** (Pod이 실행 중인 노드에서 실행)

    ```sh
    POD_ID="$(sudo crictl pods --name test-pod -q)"
    ```
2.  **Pod의 Cgroup 경로 확인**

    ```sh
    sudo crictl inspectp -o=json $POD_ID | grep cgroupsPath
    ```

    예제 출력:

    ```
    "cgroupsPath": "/kubepods/podd7f4b509-cf94-4951-9417-d1087c92a5b2/7ccf55aee35dd16aca4189c952d83487297f3cd760f1bbf09620e206e7d0c27a"
    ```

    * Pod의 상위 Cgroup 경로:\
      `/kubepods/podd7f4b509-cf94-4951-9417-d1087c92a5b2`
3.  **Pod Cgroup에서 메모리 제한값 확인**

    ```sh
    cat /sys/fs/cgroup/memory/kubepods/podd7f4b509-cf94-4951-9417-d1087c92a5b2/memory.limit_in_bytes
    ```

    예제 출력:

    ```
    335544320
    ```

    * 이는 `320MiB`(= 320 × 1024 × 1024 = 335544320 bytes)로 예상한 값과 일치함.

***

### 모니터링 (Observability)

Kubernetes에서는 **kube-state-metrics**를 활용하여 Pod 오버헤드를 모니터링할 수 있습니다.\
`kube_pod_overhead_*` 메트릭을 확인하면 Pod 오버헤드 사용 여부 및 워크로드 안정성을 측정할 수 있습니다.

Pod 오버헤드는 리소스 관리의 중요한 요소로, 올바르게 활용하면 **노드 활용도를 최적화**하고 **예상치 못한 리소스 부족 문제를 방지**하는 데 도움이 됩니다.
