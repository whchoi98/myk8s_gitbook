---
description: 'update : 2025.01.29'
---

# 테인트와 톨러레이션

## 테인트(Taints) 및 톨러레이션(Tolerations)

노드 어피니티(Node affinity)는 특정 노드에 대한 파드를 유인하는 속성으로, 이는 선택적(preference)일 수도 있고 필수적(hard requirement)일 수도 있습니다. 반면, 테인트(Taint)는 그 반대로, 특정 파드가 특정 노드에서 실행되지 않도록 하는 기능입니다.

톨러레이션(Toleration)은 파드에 적용됩니다. 톨러레이션은 테인트가 있는 노드에 파드를 스케줄링할 수 있도록 허용하지만, 반드시 스케줄링이 보장되는 것은 아닙니다. 쿠버네티스 스케줄러는 다른 조건들도 함께 고려합니다.

테인트와 톨러레이션은 협력하여 부적절한 노드에 파드가 스케줄링되지 않도록 합니다. 하나 이상의 테인트가 노드에 적용되면, 해당 테인트를 허용하지 않는 파드는 해당 노드에 스케줄링되지 않습니다.

***

### 개념

노드에 테인트를 추가하려면 `kubectl taint` 명령어를 사용합니다. 예를 들어:

```sh
kubectl taint nodes node1 key1=value1:NoSchedule
```

위 명령어는 `node1`에 `key1=value1`의 테인트를 추가하며, `NoSchedule` 효과를 적용합니다. 이는 해당 테인트를 허용하는 톨러레이션이 없는 한, 파드가 `node1`에 스케줄링되지 않도록 합니다.

이 테인트를 제거하려면 다음 명령어를 실행합니다:

```sh
kubectl taint nodes node1 key1=value1:NoSchedule-
```

***

### 톨러레이션(Toleration)

파드의 `PodSpec`에서 톨러레이션을 지정할 수 있습니다. 아래 두 가지 톨러레이션 예시는 위에서 설정한 테인트와 매칭되며, 따라서 해당 톨러레이션을 가진 파드는 `node1`에 스케줄링될 수 있습니다.

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
```

```yaml
tolerations:
- key: "key1"
  operator: "Exists"
  effect: "NoSchedule"
```

#### 파드 예제

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  tolerations:
  - key: "example-key"
    operator: "Exists"
    effect: "NoSchedule"
```

기본적으로 `operator`의 기본값은 `Equal`입니다.

***

### 톨러레이션과 테인트 매칭 조건

톨러레이션은 다음 조건을 만족하면 테인트와 매칭됩니다:

* 키(`key`)와 효과(`effect`)가 동일하며:
  * `operator`가 `Exists`인 경우 (이때 `value`는 필요 없음)
  * `operator`가 `Equal`이고 `value`도 동일한 경우

#### 특수한 경우

1. `key`가 비어 있고 `operator`가 `Exists`인 경우 → 모든 키, 값, 효과와 매칭 (즉, 모든 테인트를 허용)
2. `effect`가 비어 있는 경우 → 특정 키와 일치하는 모든 효과와 매칭

***

### 테인트 효과(Effect)

테인트 효과는 다음 세 가지 유형이 있습니다:

| 효과                 | 설명                                             |
| ------------------ | ---------------------------------------------- |
| `NoExecute`        | 이미 실행 중인 파드에 영향을 주며, 테인트를 허용하지 않는 파드는 즉시 축출됨   |
| `NoSchedule`       | 새로운 파드가 스케줄링되지 않도록 방지하지만, 기존 실행 중인 파드는 영향받지 않음 |
| `PreferNoSchedule` | 강제적이지 않지만, 해당 노드에 파드를 스케줄링하지 않도록 시도함           |

예제:

```sh
kubectl taint nodes node1 key1=value1:NoSchedule
kubectl taint nodes node1 key1=value1:NoExecute
kubectl taint nodes node1 key2=value2:NoSchedule
```

파드가 다음과 같은 톨러레이션을 가지면:

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
```

위 파드는 `key1=value1` 테인트와 매칭되므로 해당 노드에서 계속 실행될 수 있지만, `key2=value2:NoSchedule` 테인트와 매칭되지 않으므로 해당 노드에 스케줄링되지 않습니다.

***

### `NoExecute` 테인트 및 `tolerationSeconds`

`NoExecute` 효과를 가진 테인트가 추가되면, 파드는 다음과 같이 행동합니다:

* 테인트를 허용하지 않는 경우 → 즉시 축출됨
* `tolerationSeconds`를 지정하지 않으면 → 계속 실행됨
* `tolerationSeconds`가 지정된 경우 → 설정된 시간 후 축출됨

예제:

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600
```

이 경우, 파드는 해당 테인트가 추가된 후 3600초 동안만 해당 노드에서 실행되며, 이후 축출됩니다.

***

### 활용 사례

1. **전용 노드 (Dedicated Nodes)**
   * 특정 사용자 그룹만 사용할 수 있도록 특정 노드를 예약
   *   예제:

       ```sh
       kubectl taint nodes nodename dedicated=groupName:NoSchedule
       ```
   * 파드에 해당 톨러레이션을 추가하여 해당 노드에서만 실행되도록 설정
2. **특수 하드웨어가 있는 노드 (Nodes with Special Hardware)**
   * GPU 또는 기타 특수 하드웨어가 있는 노드에 특정 파드만 실행되도록 설정
   *   예제:

       ```sh
       kubectl taint nodes nodename special=true:NoSchedule
       ```
3. **테인트 기반 축출 (Taint Based Evictions)**
   * 노드 문제 발생 시 파드를 자동 축출
   *   예제:

       ```yaml
       tolerations:
       - key: "node.kubernetes.io/unreachable"
         operator: "Exists"
         effect: "NoExecute"
         tolerationSeconds: 6000
       ```

***

### 자동 테인트 및 톨러레이션

쿠버네티스 노드 컨트롤러는 특정 조건이 충족되면 자동으로 테인트를 추가합니다.

| 테인트                                              | 설명                                |
| ------------------------------------------------ | --------------------------------- |
| `node.kubernetes.io/not-ready`                   | 노드가 준비되지 않음 (`Ready=False`)       |
| `node.kubernetes.io/unreachable`                 | 노드 컨트롤러에서 접근 불가 (`Ready=Unknown`) |
| `node.kubernetes.io/memory-pressure`             | 메모리 부족                            |
| `node.kubernetes.io/disk-pressure`               | 디스크 부족                            |
| `node.kubernetes.io/pid-pressure`                | PID 부족                            |
| `node.kubernetes.io/network-unavailable`         | 네트워크 사용 불가                        |
| `node.kubernetes.io/unschedulable`               | 노드가 스케줄링 불가능                      |
| `node.cloudprovider.kubernetes.io/uninitialized` | 클라우드 프로바이더 초기화 전                  |

#### 기본적으로 추가되는 톨러레이션

* `node.kubernetes.io/not-ready` 및 `node.kubernetes.io/unreachable`에 대해 `tolerationSeconds=300`이 자동 추가됨
* `DaemonSet` 파드는 위 두 테인트를 영구적으로 허용하여 축출되지 않음

***

### 결론

테인트와 톨러레이션을 활용하면 클러스터에서 특정 노드에 대한 파드 배치를 제어할 수 있습니다. 특정 노드를 특정 워크로드에만 사용하거나, 노드 장애 시 파드를 자동 축출하는 등 다양한 활용이 가능합니다.

***

## **Taints & Tolerations 실습 (특정 노드 그룹만 적용)**

이 실습에서는 **특정 노드 그룹**에 **Taints & Tolerations**을 적용하고, 일부 그룹에서는 예외를 설정하여 다르게 동작하는 것을 실습합니다.

***

### **목표**

1. **테인트(taint) 적용 대상: 배포시에 이미 Label 이 적용되어 있습니다.**
   * `frontend-workloads`
   * `backend-workloads`
   * `managed-frontend-workloads`
2. **테인트(taint) 적용 제외 대상:**
   * `managed-backend-workloads` (테인트 없이 모든 파드 허용)
3. **테인트(taint)가 적용된 노드에서만 실행 가능한 파드 배포**
4. **테인트(taint)가 없는 `managed-backend-workloads` 노드에서 모든 파드 허용 확인**
5. **실습 종료 후 테인트 제거**

***

### **특정 노드 그룹에 테인트 적용**

각 노드 그룹의 **첫 번째 노드**에 자동으로 테인트를 적용합니다.\
단, `managed-backend-workloads`는 테인트를 적용하지 않습니다.

```sh
kubectl taint nodes -l nodegroup-type=frontend-workloads key=value:NoSchedule
kubectl taint nodes -l nodegroup-type=backend-workloads key=value:NoSchedule
kubectl taint nodes -l nodegroup-type=managed-frontend-workloads key=value:NoSchedule
```

**테인트 적용 확인:**

```sh
kubectl get nodes -o jsonpath="{range .items[*]}{.metadata.name}{': '}{.spec.taints}{'\n'}{end}"

```

출력 예시:

```
ip-10-11-11-82.ap-northeast-2.compute.internal: [{"effect":"NoSchedule","key":"key","value":"value"}]
ip-10-11-13-244.ap-northeast-2.compute.internal: [{"effect":"NoSchedule","key":"key","value":"value"}]
ip-10-11-17-104.ap-northeast-2.compute.internal: [{"effect":"NoSchedule","key":"key","value":"value"}]
ip-10-11-24-137.ap-northeast-2.compute.internal: [{"effect":"NoSchedule","key":"key","value":"value"}]
ip-10-11-38-129.ap-northeast-2.compute.internal: [{"effect":"NoSchedule","key":"key","value":"value"}]
ip-10-11-47-81.ap-northeast-2.compute.internal: [{"effect":"NoSchedule","key":"key","value":"value"}]
ip-10-11-53-225.ap-northeast-2.compute.internal: [{"effect":"NoSchedule","key":"key","value":"value"}]
ip-10-11-56-142.ap-northeast-2.compute.internal: 
ip-10-11-69-201.ap-northeast-2.compute.internal: 
ip-10-11-74-202.ap-northeast-2.compute.internal: [{"effect":"NoSchedule","key":"key","value":"value"}]
ip-10-11-81-255.ap-northeast-2.compute.internal: 
ip-10-11-83-245.ap-northeast-2.compute.internal: [{"effect":"NoSchedule","key":"key","value":"value"}]
```

`managed-backend-workloads`에는 테인트가 적용되지 않았습니다.

***

### **톨러레이션이 없는 일반 파드 배포 (테인트 적용 노드에서 실패, 적용 안 된 노드에서 성공)**

톨러레이션이 없는 일반 파드를 배포하여 **테인트가 적용된 노드에는 배포되지 않고, `managed-backend-workloads`에서는 배포되는지** 확인합니다.

#### `nginx-no-toleration.yaml`

```yaml
mkdir -p ~/environment/taint-toleration
cat <<EoF > ~/environment/taint-toleration/nginx-no-toleration.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-no-toleration
  namespace: taint-toleration
spec:
  containers:
  - name: nginx
    image: nginx
EoF
kubectl create namespace taint-toleration
kubectl apply -f ~/environment/taint-toleration/nginx-no-toleration.yaml

```

**상태 확인 및 비교:**

```sh
kubectl -n taint-toleration get pods -o wide
kubectl get nodes -o jsonpath="{range .items[*]}{.metadata.name}{': '}{.spec.taints}{'\n'}{end}"

```

출력 예시:

```
NAME                  READY   STATUS    RESTARTS   AGE   IP             NODE                                              NOMINATED NODE   READINESS GATES
nginx-no-toleration   1/1     Running   0          35s   10.11.93.240   ip-10-11-81-255.ap-northeast-2.compute.internal   <none>           <none>
```

톨러레이션이 없는 파드는 `managed-backend-workloads` 노드에서만 실행되는 것을 확인 할 수 있습니다.

아래 명령으로 실습에 적용한 namespace를 삭제 하고, taint를 제거합니다.

```
kubectl delete namespaces taint-toleration
kubectl taint nodes -l nodegroup-type=frontend-workloads key-
kubectl taint nodes -l nodegroup-type=backend-workloads key-
kubectl taint nodes -l nodegroup-type=managed-frontend-workloads key-
```

***

### **톨러레이션이 포함된 파드 배포 (테인트 적용된 노드에서 성공)**

이제 **톨러레이션이 포함된 파드**를 배포하여, 해당 노드 그룹의 첫번째 노드에서 실행되는지 확인합니다.

<pre><code><strong>
</strong><strong>kubectl taint nodes $(kubectl get nodes -l nodegroup-type=managed-frontend-workloads -o jsonpath='{.items[0].metadata.name}') frontend-only=true:NoSchedule
</strong>kubectl taint nodes $(kubectl get nodes -l nodegroup-type=managed-backend-workloads -o jsonpath='{.items[0].metadata.name}') backend-only=true:NoSchedule

</code></pre>

테인트 적용:

```
kubectl get nodes -o jsonpath="{range .items[*]}{.metadata.name}{': '}{.spec.taints}{'\n'}{end}"

```

적용결과 예시:

```
ip-10-11-11-82.ap-northeast-2.compute.internal: [{"effect":"NoSchedule","key":"frontend-only","value":"true"}]
ip-10-11-13-244.ap-northeast-2.compute.internal: 
ip-10-11-17-104.ap-northeast-2.compute.internal: 
ip-10-11-24-137.ap-northeast-2.compute.internal: 
ip-10-11-38-129.ap-northeast-2.compute.internal: 
ip-10-11-47-81.ap-northeast-2.compute.internal: 
ip-10-11-53-225.ap-northeast-2.compute.internal: 
ip-10-11-56-142.ap-northeast-2.compute.internal: [{"effect":"NoSchedule","key":"backend-only","value":"true"}]
ip-10-11-69-201.ap-northeast-2.compute.internal: 
ip-10-11-74-202.ap-northeast-2.compute.internal: 
ip-10-11-81-255.ap-northeast-2.compute.internal: 
ip-10-11-83-245.ap-northeast-2.compute.internal: 
```

**톨러레이션이 포함된 파드를 배포합니다.(managed-frontend-toleration, managed-backend-toleration)**

#### managed-frontend-toleration`.yaml`

```yaml
cat <<EoF > ~/environment/taint-toleration/managed-frontend-toleration.yaml
apiVersion: v1
kind: Pod
metadata:
  name: managed-frontend-toleration
  namespace: taint-toleration
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: "frontend-only"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: nodegroup-type
            operator: In
            values:
            - managed-frontend-workloads
  tolerations:
  - key: "frontend-only"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
EoF
kubectl create namespace taint-toleration
kubectl apply -f ~/environment/taint-toleration/managed-frontend-toleration.yaml

```

#### `nginx-backend-toleration.yaml`

```yaml
cat <<EoF > ~/environment/taint-toleration/managed-backend-toleration.yaml
apiVersion: v1
kind: Pod
metadata:
  name: managed-backend-toleration
  namespace: taint-toleration
spec:
  containers:
  - name: nginx
    image: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: nodegroup-type
            operator: In
            values:
            - managed-backend-workloads
  tolerations:
  - key: "backend-only"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
EoF
kubectl apply -f ~/environment/taint-toleration/managed-backend-toleration.yaml

```

**상태 확인:**

```sh
kubectl -n taint-toleration get pods -o wide
kubectl get nodes -o jsonpath="{range .items[*]}{.metadata.name}{': '}{.spec.taints}{'\n'}{end}"

```

출력 예시:

```
$ kubectl -n taint-toleration get pods -o wide
kubectl get nodes -o jsonpath="{range .items[*]}{.metadata.name}{': '}{.spec.taints}{'\n'}{end}"
NAME                          READY   STATUS    RESTARTS   AGE   IP            NODE                                              NOMINATED NODE   READINESS GATES
managed-backend-toleration    1/1     Running   0          7s    10.11.89.7    ip-10-11-81-255.ap-northeast-2.compute.internal   <none>           <none>
managed-frontend-toleration   1/1     Running   0          63s   10.11.2.193   ip-10-11-11-82.ap-northeast-2.compute.internal    <none>           <none>
ip-10-11-11-82.ap-northeast-2.compute.internal: [{"effect":"NoSchedule","key":"frontend-only","value":"true"}]
ip-10-11-56-142.ap-northeast-2.compute.internal: [{"effect":"NoSchedule","key":"backend-only","value":"true"}]
```

***

### **테인트 제거하기**

실습 종료 후 **테인트를 제거**하려면 다음 명령어를 실행하세요.

```sh
kubectl taint nodes $(kubectl get nodes -l nodegroup-type=managed-frontend-workloads -o jsonpath='{.items[0].metadata.name}') frontend-only=true:NoSchedule-
kubectl taint nodes $(kubectl get nodes -l nodegroup-type=managed-backend-workloads -o jsonpath='{.items[0].metadata.name}') backend-only=true:NoSchedule-

```

**제거 확인**

```sh
kubectl -n taint-toleration get nodes -o jsonpath="{range .items[*]}{.metadata.name}{': '}{.spec.taints}{'\n'}{end}"
```

***

