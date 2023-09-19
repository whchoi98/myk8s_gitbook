---
description: 'Update : 2023-09-19'
---

# Assign

## Assign 소개.&#x20;

Pod의 효과적인 할당을 위한 기술 전략과 작동 방식 및 다양한 방법들에 대해 소개합니다.필요에 따라 특정 노드에서만 실행하거나 특정 노드에서 실행하는 것에 우선 순위를 두도록 제한할 수 있습니다. 스케쥴러가 자동으로 적절한 배치를 수행하므로 일반적으로 이러한 제한들은 필요하지 않지만, 더 세밀한 제어가 필요한 경우가 프로덕션에서 발생할 수 있습니다. &#x20;

이번 실습에서는 이러한 요구 조건에 맞추어 Pod에 대한 할당을 중점적으로 다룹니다.

## NodeSelector

### 1.소개.

NodeSelector는 가장 간단한 노드 선택 방법입니다. NodeSelector는 PodSpec 필드에서 선택이 가능하며,  Key-value 페어로 지정하게 됩니다. Pod가 Node에 실행 될 수 있으려면, Node에 표시된 Key-Value 형태의 Lable이 존재해야 합니다. 이것은 다중으로 지정할 수도 있습니다.

```
"metadata": {
  "labels": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

### 2.Node 의 Label 확인.

이미 yaml로 정의된 EKS LAB을 eksctl로 배포시에, 각 Worker Node들에 Lable이 정의 되어 있었습니다.

```
cat ~/environment/myeks/eksworkshop-cluster-3az.yaml
```

whchoi-cluster.yaml의 내용을 살펴 봅니다.

```
#생략

nodeGroups:
  - name: ng1-public
    instanceType: m5.xlarge
    desiredCapacity: 3
    minSize: 3
    maxSize: 9
    volumeSize: 30
    volumeType: gp2 
    amiFamily: AmazonLinux2
#
#Public-worker-node에 대한 label을 정의
#
    labels:
      nodegroup-type: "frontend-workloads"
    ssh: 
        publicKeyPath: "/home/ec2-user/environment/eksworkshop.pub"
        allow: true
    iam:
      attachPolicyARNs:
      withAddonPolicies:
        autoScaler: true
        cloudWatch: true
        ebs: true
        fsx: true
        efs: true

  - name: ng2-private
    instanceType: m5.xlarge
    desiredCapacity: 3
    privateNetworking: true
    minSize: 3
    maxSize: 9
    volumeSize: 30
    volumeType: gp2 
    amiFamily: AmazonLinux2
#
#Private-worker-node에 대한 label을 정의
#
    labels:
      nodegroup-type: "backend-workloads"
    ssh: 
        publicKeyPath: "/home/ec2-user/environment/eksworkshop.pub"
        allow: true
    iam:
      withAddonPolicies:
        autoScaler: true
        cloudWatch: true
        ebs: true
        fsx: true
        efs: true

#이하 생략
```

현재 설정되어 있는 Node, Pods의 label은 아래 명령을 통해 확인이 가능합니다.

```
kubectl get node --show-labels -o wide

```

사전에 정의해 둔 "nodegroup-type=frontend-workloads" 의 노드들을 출력해 봅니다.

```
kubectl get nodes -l nodegroup-type=frontend-workloads

```

아래와 같은 출력 결과를 볼 수 있습니다.

```
whchoi98:~ $ kubectl get nodes -l nodegroup-type=backend-workloads
NAME                                              STATUS   ROLES    AGE   VERSION
ip-10-11-16-31.ap-northeast-2.compute.internal    Ready    <none>   90m   v1.16.12-eks-904af05
ip-10-11-55-30.ap-northeast-2.compute.internal    Ready    <none>   88m   v1.16.12-eks-904af05
ip-10-11-90-240.ap-northeast-2.compute.internal   Ready    <none>   85m   v1.16.12-eks-904af05
```

### 3.새로운 Label 정의 .

한 개의 Node Name을 선택하고, label을 지정합니다. (예. assigntest=01)

```
kubectl label nodes ip-10-11-16-31.ap-northeast-2.compute.internal disktype=ssd
```

정상적으로 라벨링 되었는지 확인합니다.

```
kubectl get nodes -l disktype=ssd
```

### 4. 특정 레이블의 노드에 Pod 배포.

새로운 디렉토리와 namespace를 만들고 nginx 배포를 위한 매니페스트 파일을 간단하게 작성합니다. 해당 Pod는 생성될때 앞서 label에서 정의된 disktype=ssd 가  추가된 worker node로 배포됩니다.

```
mkdir ~/environment/nodeselector
kubectl create namespace nodeselector 
cat <<EoF > ~/environment/nodeselector/pod-nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
  namespace: nodeselector
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
EoF

```

생성한 매니페스트 파일을 통해 nginx app을 배포합니다.

```
kubectl -n nodeselector apply -f ~/environment/nodeselector/pod-nginx.yaml

```

정상적으로 배포되었는지 확인합니다. 앞서 지정한 worker node로 배포되어야 합니다.

```
kubectl get nodes --show-labels | grep disktype
kubectl -n nodeselector get pods -o wide

```

아래는 출력 결과 예시입니다.

```
$ kubectl get nodes --show-labels | grep disktype
ip-10-11-16-31.ap-northeast-2.compute.internal     Ready    <none>   120m   v1.16.12-eks-904af05   alpha.eksctl.io/cluster-name=eksworkshop,alpha.eksctl.io/instance-id=i-0ca4ddf4adea01038,alpha.eksctl.io/nodegroup-name=ng1-public,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=m5.xlarge,beta.kubernetes.io/os=linux,disktype=ssd,failure-domain.beta.kubernetes.io/region=ap-northeast-2,failure-domain.beta.kubernetes.io/zone=ap-northeast-2a,kubernetes.io/arch=amd64,kubernetes.io/hostname=ip-10-11-16-31.ap-northeast-2.compute.internal,kubernetes.io/os=linux,nodegroup-type=frontend-workloads

$ kubectl -n nodeselector get pods -o wide
NAME    READY   STATUS    RESTARTS   AGE     IP            NODE                                             NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          4m18s   10.11.28.19   ip-10-11-16-31.ap-northeast-2.compute.internal   <none>           <none>
```

## Affinity 와 Anti-Affinity

### 1.소개.

`nodeSelector` 는 파드를 특정 label이 있는 노드로 배하는 매우 간단한 방법을 제공합니. 하지만 Affinity/Anti-Affinity 기능은 조금 더 유연하게 파드를 노드에 배치하는 방법을 제공합니다. 주요 개선 사항은 다음과 같습니다 .

1. Affinity/Anti-Affinity 언어가 더 다양하게 표현할 수 있습니다 . 언어는 논리 연산자인 AND 연산으로 작성된 정확한 매칭 항목 이외에 더 많은 매칭 규칙을 제공합니다 .
2. 규칙이 엄격한 요구 사항이 아니라 "required-hard affinity / preferred - soft affinity" 규칙을 나타낼 수 있기에 스케줄러가 규칙을 만족할 수 없더라도, 파드가 계속 유연하게 스케줄 되도록 합니다.
3. 노드 자체에 label을 붙이기보다는 노드(또는 다른 토폴로지 도메인)에서 실행 중인 다른 파드의 label을 제한할 수 있습니다. 이를 통해 어떤 파드가 함께 위치할 수 있는지와 없는지에 대한 규칙을 적용할 수 있습니다 .

Affinity 기능은 "노드 Affinity" 와 "파드 Affinity/Anti-Affinity" 두 종류의 Affinity로 구성됩니다. "노드 Affinity" 는 기존 `nodeSelector` 와 비슷하지만(그러나 위에서 나열된 첫째와 두 번째 이점이 있습다.), "파드 Affinity/Anti-Affinity"  위에서 나열된 세번째 항목에 설명된 대로 노드 label이 아닌 파드 label에 대해 제한되고 위에서 나열된 첫 번째와 두 번째 속성을 가집니다 .&#x20;

### 2.Node Affinity

앞서 소개한 [nodeSelector](assign.md#nodeselector) 와 유사합니다. 노드 label을 기반으로 다양한 조건들을 명시할 수 있다는 점에서 차이가 있습니다.

현재 `requiredDuringSchedulingIgnoredDuringExecution` 와 `preferredDuringSchedulingIgnoredDuringExecution` 로 부르는 두 가지 종류의 노드 어피니티가 있습니다 .

&#x20;`requiredDuringSchedulingIgnoredDuringExecution`  파드가 노드에 스케줄 되도록 반드시 규칙을 만해야 하는 것(`nodeSelector` 와 같으나 보다 상세 구문을 사용해서)을 지정하고, `preferredDuringSchedulingIgnoredDuringExecution` 는 스케줄러가 시도하려고는 하지만, _preferences_ 를 지정한다는 점에서 이를 각각 "엄격함(hard)" 과 "유연함(soft)" 으로 생각할 수 있습니다.&#x20;

이름의 "IgnoredDuringExecution" 부분은 `nodeSelector` 작동 방식과 유사하게 노드의 레이블이 런타임 중에 변경되어 파드의 Affinity 규칙이 더 이상 충족되지 않아도  파드가 여전히 그 노드에서 동작한다는 의미입니다 .&#x20;

향후에는 `referredDuringSchedulingIgnoredDuringExecution` 와 같은 `requiredDuringSchedulingIgnoredDuringExecution` 를 제공할 계획입니다 .

따라서 `requiredDuringSchedulingIgnoredDuringExecution` 의 예로는 "인텔 CPU가 있는 노드에서만 파드 실행"이 될 수 있고, `preferredDuringSchedulingIgnoredDuringExecution` 의 예로는 "장애 조치 영역 XYZ에 파드 집합을 실행하려고 하지만, 불가능하다면 다른 곳에서 일부를 실행하도록 허용"이 있을 것이다.

아래와 같이 pod 배포를 위한 새로운 매니페스트 파일을 작성합니다.

Node Affinity는 다음과 같은 구문의 의미를 가지고 있습니다.

* required - 반드시 포함되어야 함.
* preferred - 우선하지만 필수는 아님.
* Ingnored - Runtime 중에 Node Label 변경시 무시
* DuringScheduling 뒤에 오는 required - Runtime 중에 NodeLabel 변경시 즉시 재기동.

```
mkdir ~/environment/affinity
cat <<EoF > ~/environment/affinity/pod-with-node-affinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: azname
            operator: In
            values:
            - ap-northeast-2a
            - ap-northeast-2b
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: us.gcr.io/k8s-artifacts-prod/pause:2.0
EoF

```

{% hint style="info" %}
해당 Affinity의 NodeAffinity는 node의 label의 Key=azname, value=ap-northeast-2a,ap-northeast-2b가 있는 곳에만 배치하겠다는 의미입니다. 또한 해당 기준을 충족하는 노드 중에서 Key=another-node-label-key value=another-node-label-value를 선호합니다.
{% endhint %}

예제에서 operator:In 연산자를 사용하였습니다.  In, NotIn, Exists, DoesNotExist, Gt, Lt 등 다양한 연산자를 사용할 수 있습니다. nodeAffinity에 대한 아래 몇가지 내용을 알아 둘 필요가 있습니다.

* 2개 이상의 matchExpressions가 선언되어 있는 경우에는 , 모든 조건이 만족해야 Pod가 배포됩니다. 또한 matchExpressions 내부의 선언된 조건이 여러개 있는 경우에도 모든 조건을 만족해야 Pod가 배포됩니다.
* key는 고유의 값 한개를 선언하지만, value는 예제 처럼 여러개를 선언할 수 있습니다. (value는 하나의 조건만 만족해도 됩니다.)
* preferred와 required는 create,apply 시점에만 반영됩니다.&#x20;

이제 아래와 같이 새로운 namespace를 만들고 , 적용해 봅니다.

```
kubectl create namespace affinity
kubectl -n affinity apply -f ~/environment/affinity/pod-with-node-affinity.yaml

```

아래 명령을 통해 Pod의 배포 상태를 확인합니다.

```
kubectl -n affinity get pods -o wide

```

Node의 Label 이 azname=ap-northeast-2a, 2b를 가지고 있는 것이 아직 없기 때문에 pending 상태입니다.

```
whchoi98:~ $ kubectl -n affinity get pods -o wide
NAME                 READY   STATUS              RESTARTS   AGE    IP       NODE                                             NOMINATED NODE   READINESS GATES
with-node-affinity   0/1     pendingng               0     2m4s   <none>   ip-10-11-16-31.ap-northeast-2.compute.internal   <none>           <none>
```

이제 node에 nodAffinity를 위한 label을 지정합니다. 노드 어피니티는 PodSpec의 `affinity` 필드의 `nodeAffinity` 필드에서 지정됩니다.&#x20;

```
kubectl label nodes ip-10-11-16-31.ap-northeast-2.compute.internal azname=ap-northeast-2a
```

이제 정상적으로 pod가 생성되는 지 확인합니다.

```
$ kubectl -n affinity get pods -o wide
NAME                 READY   STATUS    RESTARTS   AGE     IP            NODE                                             NOMINATED NODE   READINESS GATES
with-node-affinity   1/1     Running   0          2m10s   10.11.24.52   ip-10-11-16-31.ap-northeast-2.compute.internal   <none>           <none>
```



### 3.Pod Affinity&#x20;

PodAffinity를 활용하면 ReplicaSets, StatefulSets, Deployments 등과 함께 좀 더 상세하고 유용하게 사용할 수 있습니다. 예를 들면 동일 노드에 목적에 따라 함께 사용될 Pod들을 함께 배치 할 수 있습니다.

이번 랩에서는 아래에서 처럼 Pod를 배포하려고 합니다.

{% hint style="success" %}
worker node 1 - Webserver1 - RedisCache1

worker node2 - Webserver2 - RedisCache2

worker node3 - Webserver3 - RedisCache3
{% endhint %}

앞서 생성한 namespace에 아래와 같은 RedisCache 매니페스트 파일을 생성합니다.

```
cat <<EoF > ~/environment/affinity/redis-with-node-affinity.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
  namespace: affinity
spec:
  selector:
    matchLabels:
      app: store
  replicas: 3
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: redis-server
        image: redis:3.2-alpine
EoF

```

매니페스트 파일에는 다음과 같은 정보들을 담고 있습니다.

{% hint style="success" %}
* **App : redis-cache**
* **replica : 3개**
* **label : app=store**
* **podAntiAffinity : app=store**
{% endhint %}

{% hint style="info" %}
Redis-Cache를 배포할 때 label이 app입니다. 그런데 podAntiAffinity가 app이므로, Redis-cache가 없는 노드에 배포될 것입니다. 3개의 WorkerNode에 분산해서 설치하겠다는 의미입니다.
{% endhint %}

앞서 생성한 namespace에 아래와 같은 nginx WebServer 매니페스트 파일을 생성합니다.

```
cat <<EoF > ~/environment/affinity/web-with-node-affinity.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
  namespace: affinity
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 3
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname"
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
        image: nginx:1.12-alpine
EoF

```

&#x20;매니페스트 파일에는 다음과 같은 정보들을 담고 있습니다.

{% hint style="success" %}
* App : web-server (alpine nginx 1.12)
* replica : 3개
* label : app=web-store
* podAntiAffinity : app=web-store
* podAffinity: app=store
{% endhint %}

{% hint style="info" %}
Web-Server를 배포할 때 label이 web-store입니다. 그런데 podAntiAffinity가 web-store이므로, web-server가 없는 노드에 배포될 것입니다. 하지만 만약 Node가 많다면, Redis-Cache에 없는 노드에 설치될 수도 있기 때문에, podAffinity가 app이므로 Redis-Cache에 있는 노드에만 설치됩니다.
{% endhint %}

이와 같은 배포전략은 여러개의 앱을 요구 조건에 따라 특정 노드에 묶어서 배포할 때, 매우 효과적이며 강력한 배포전략 기술입니다.

이제 만들어진 매니페스트 파일을 배포합니다.

```
kubectl apply -f ~/environment/affinity/redis-with-node-affinity.yaml
kubectl apply -f ~/environment/affinity/web-with-node-affinity.yaml

```

정상적으로 배포되었는지 확인해 봅니다.

```
kubectl -n affinity get pods -o wide

```

정상적으로 배포되었다면 아래와 같은 결과를 볼 수 있습니다. Web-Server와 Redis-Cache 서버는 모두 분산되어 배포되었습니다.

```
whchoi98:~ $ kubectl -n affinity get pods -o wide
NAME                           READY   STATUS    RESTARTS   AGE    IP              NODE                                               NOMINATED NODE   READINESS GATES
redis-cache-6bc7d5b59d-c2cmr   1/1     Running   0          18s    10.11.94.33     ip-10-11-90-240.ap-northeast-2.compute.internal    <none>           <none>
redis-cache-6bc7d5b59d-d729w   1/1     Running   0          18s    10.11.120.83    ip-10-11-114-132.ap-northeast-2.compute.internal   <none>           <none>
redis-cache-6bc7d5b59d-pt9vc   1/1     Running   0          18s    10.11.150.121   ip-10-11-146-170.ap-northeast-2.compute.internal   <none>           <none>
web-server-655bf8bdf4-7pvsr    1/1     Running   0          18s    10.11.73.255    ip-10-11-90-240.ap-northeast-2.compute.internal    <none>           <none>
web-server-655bf8bdf4-ghgm8    1/1     Running   0          18s    10.11.111.44    ip-10-11-114-132.ap-northeast-2.compute.internal   <none>           <none>
web-server-655bf8bdf4-rmzkk    1/1     Running   0          18s    10.11.153.163   ip-10-11-146-170.ap-northeast-2.compute.internal   <none>           <none>
with-node-affinity             1/1     Running   0          173m   10.11.24.52     ip-10-11-16-31.ap-northeast-2.compute.internal     <none>           <none>
```

