---
description: 'Update : 2023-09-19'
---

# 고가용성 Health Check 구성

## 컨테이너 프로브 소개

[프로브](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#probe-v1-core)는 컨테이너에서 [kubelet](https://kubernetes.io/docs/admin/kubelet/)에 의해 주기적으로 수행되는 진단(diagnostic) 도구입니. 진단을 수행하기 위해서, kubelet은 컨테이너에 의해서 구현된 [핸들러](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#handler-v1-core)를 호출합니다.

핸들러는 다음과 같이 세가지 타입이 있습니다.

* [ExecAction](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#execaction-v1-core) 은 컨테이너 내에서 지정된 명령어를 실행합니다 . 명령어가 상태 코드 0으로 종료되면 진단이 성공한 것으로 판단합니다.
* [TCPSocketAction](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#tcpsocketaction-v1-core) 은 지정된 포트에서 컨테이너의 IP주소에 대해 TCP 검사를 수행합니다 . 포트가 활성화되어 있다면 진단이 성공한 것으로 간주합니다.
* [HTTPGetAction](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#httpgetaction-v1-core) 은 지정한 포트 및 경로에서 컨테이너의 IP주소에 대한 HTTP Get 요청을 수행합니다 . 응답의 상태 코드가 200보다 크고 400보다 작으면 진단이 성공한 것으로 간주합니다.

각 probe는 다음 세 가지 결과 중 하나를 가집니다.&#x20;

* Success: 컨테이너가 진단을 통과함.
* Failure: 컨테이너가 진단에 실패함.
* Unknown: 진단 자체가 실패하였으므로 아무런 액션도 수행되면 안됨.

kubelet은 실행 중인 컨테이너들에 대해서 선택적으로 세 가지 종류의 프로브를 수행하고 그에 반응할 수 있습니다 .

* `livenessProbe`: 컨테이너가 동작 중인지 여부를 나타냅니다. 만약 활성 프로브(liveness probe)에 실패한다면, kubelet은 컨테이너를 죽이고, 해당 컨테이너는 [재시작 정책](https://kubernetes.io/ko/docs/concepts/workloads/pods/pod-lifecycle/#%EC%9E%AC%EC%8B%9C%EC%9E%91-%EC%A0%95%EC%B1%85)의 대상이 됩니다. \
  컨테이너가 활성 프로브를 제공하지 않는 경우, 기본 상태는 `Success`입니다.
*   `readinessProbe`: 컨테이너가 요청을 처리할 준비가 되었는지 여부를 나타냅니다. \
    `readinessProbe`실패한다면, 엔드포인트 컨트롤러는 파드에 연관된 모든 서비스들의 엔드포인트에서 파드의 IP주소를 제거합니다.&#x20;

    `readinessProbe`의 초기 지연 이전의 기본 상태는 `Failure`입니다. 컨테이너가 `readinessProbe`를 지원하지 않는다면, 기본 상태는 `Success`입니다.
* `startupProbe`: 컨테이너 내의 애플리케이션이 시작되었는지를 나타냅니다. `startupProbe`가 주어진 경우, 성공할 때 까지 다른 나머지 프로브는 활성화 되지 않습니다. \
  `startupProbe`실패하면, kubelet이 컨테이너를 죽이고, 컨테이너는 [재시작 정책](https://kubernetes.io/ko/docs/concepts/workloads/pods/pod-lifecycle/#%EC%9E%AC%EC%8B%9C%EC%9E%91-%EC%A0%95%EC%B1%85)에 따라 처리됩니. 컨테이너에 스타트업 프로브가 없는 경우, 기본 상태는 `Success`입니다.

## Liveness Probe 구성

아래 그림에서 1번의 구성과 같은 컨셉으로을 이번 LAB에서 구성할 것입니다.

![참조 - https://kubernetes.io/blog/2018/10/01/health-checking-grpc-servers-on-kubernetes/](<../.gitbook/assets/image (299).png>)

### 1.Liveness Probe 구성을 포함한 App 배포

Liveness Probe와 Readiness Probe 구성을 위한 디렉토리를 생성합니다.

```
mkdir -p ~/environment/healthchecks
```

kubelet은 periodSeconds 필드를 사용하여 컨테이너를 상태를 확인합니다. 이 경우 kubelet은 5 초마다 liveness probe를 통해 확인합니다. initialDelaySeconds 필드는 첫 번째 Probe를 수행하기 전에 5 초 동안 기다려야한다는 것을 kubelet에 알리는 데 사용됩니다. 프로브를 수행하기 위해 kubelet은 이 포드를 호스팅 하는 서버에 HTTP GET 요청을 전송하고 서버 / health의 핸들러가 성공 코드를 반환하면 컨테이너가 정상으로 간주됩니다. 핸들러가 실패 코드를 반환하면 kubelet은 컨테이너를 종료하고 다시 시작합니다.&#x20;

```
cat <<EoF > ~/environment/healthchecks/liveness-app.yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-app
  namespace: healthchecks
spec:
  containers:
  - name: liveness
    image: brentley/ecsdemo-nodejs
    livenessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 5
      periodSeconds: 5
EoF

```

생성된 매니페스트를 사용하여 포드를 생성합니다.

```
kubectl create namespace healthchecks
kubectl apply -f ~/environment/healthchecks/liveness-app.yaml
kubectl -n healthchecks get pod liveness-app

```

출력 결과 예제

```
whchoi98:~ $ kubectl -n healthchecks get pod liveness-app
NAME           READY   STATUS    RESTARTS   AGE
liveness-app   1/1     Running   0          14s
```

아래 명령을 통해서 liveness-app의 이벤트를 확인 할 수 있습니다.

```
kubectl -n healthchecks describe pods liveness-app
```

출력 결과 예제&#x20;

{% hint style="info" %}
Event 출력항목을 확인해 봅니다.
{% endhint %}

```
whchoi98:~ $ kubectl -n healthchecks describe pods liveness-app
Name:         liveness-app
Namespace:    healthchecks
Priority:     0
Node:         ip-10-11-189-67.ap-northeast-2.compute.internal/10.11.189.67
Start Time:   Wed, 22 Jul 2020 02:47:44 +0000
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"name":"liveness-app","namespace":"healthchecks"},"spec":{"containers":[{"ima...
              kubernetes.io/psp: eks.privileged
Status:       Running
IP:           10.11.162.172
IPs:
  IP:  10.11.162.172
Containers:
  liveness:
    Container ID:   docker://650f39a9c57c38661e97c1bdddd0338751d6df9d78e6baea9b2767c7a888d58a
    Image:          brentley/ecsdemo-nodejs
    Image ID:       docker-pullable://brentley/ecsdemo-nodejs@sha256:81b3ef18eb896bf0737593f49cf813e46b03bda5eaff975dd95105b2b7b12ded
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Wed, 22 Jul 2020 02:47:47 +0000
    Ready:          True
    Restart Count:  0
    Liveness:       http-get http://:3000/health delay=5s timeout=1s period=5s #success=1 #failure=3
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-v76s5 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-v76s5:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-v76s5
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age   From                                                      Message
  ----    ------     ----  ----                                                      -------
  Normal  Scheduled  14m   default-scheduler                                         Successfully assigned healthchecks/liveness-app to ip-10-11-189-67.ap-northeast-2.compute.internal
  Normal  Pulling    14m   kubelet, ip-10-11-189-67.ap-northeast-2.compute.internal  Pulling image "brentley/ecsdemo-nodejs"
  Normal  Pulled     14m   kubelet, ip-10-11-189-67.ap-northeast-2.compute.internal  Successfully pulled image "brentley/ecsdemo-nodejs"
  Normal  Created    14m   kubelet, ip-10-11-189-67.ap-northeast-2.compute.internal  Created container liveness
  Normal  Started    14m   kubelet, ip-10-11-189-67.ap-northeast-2.compute.internal  Started container liveness
```

### 2. App 장애 발생.

이제 Docker 런타임에서 nodejs 애플리케이션 프로그램에 대한 kill 신호를 보내서 장애를 발생시킵니다.

```
kubectl -n healthchecks exec -it liveness-app -- /bin/kill -s SIGUSR1 1
```

아래 명령을 통해서 liveness-app의 이벤트의 변화를 확인해 봅니다.

```
kubectl -n healthchecks describe pods liveness-app
```

출력 결과 예시

```
생략
Events:
  Type     Reason     Age                    From                                                      Message
  ----     ------     ----                   ----                                                      -------
  Normal   Scheduled  29m                    default-scheduler                                         Successfully assigned healthchecks/liveness-app to ip-10-11-189-67.ap-northeast-2.compute.internal
  Warning  Unhealthy  7m38s (x3 over 7m48s)  kubelet, ip-10-11-189-67.ap-northeast-2.compute.internal  Liveness probe failed: Get http://10.11.162.172:3000/health: net/http: request canceled (Client.Timeout exceeded while awaiting headers)
  Normal   Killing    7m38s                  kubelet, ip-10-11-189-67.ap-northeast-2.compute.internal  Container liveness failed liveness probe, will be restarted
  Normal   Pulling    7m8s (x2 over 29m)     kubelet, ip-10-11-189-67.ap-northeast-2.compute.internal  Pulling image "brentley/ecsdemo-nodejs"
  Normal   Pulled     7m5s (x2 over 29m)     kubelet, ip-10-11-189-67.ap-northeast-2.compute.internal  Successfully pulled image "brentley/ecsdemo-nodejs"
  Normal   Created    7m5s (x2 over 29m)     kubelet, ip-10-11-189-67.ap-northeast-2.compute.internal  Created container liveness
  Normal   Started    7m5s (x2 over 29m)     kubelet, ip-10-11-189-67.ap-northeast-2.compute.internal  Started container liveness
```

5초 간격으로 지속적으로 Probe를 체크하고 있기 때문에, 로그에서도 확인 할 수 있습니다.

```
kubectl -n healthchecks logs liveness-app --tail 10
```

출력 결과 예시&#x20;

{% hint style="info" %}
5초 마다 Probe를 시도하는 것을 로그에서 확인할 수 있습니다.
{% endhint %}

```
whchoi98:~ $ kubectl -n healthchecks logs liveness-app --tail 10
::ffff:10.11.189.67 - - [22/Jul/2020:03:17:41 +0000] "GET /health HTTP/1.1" 200 17 "-" "kube-probe/1.16+"
::ffff:10.11.189.67 - - [22/Jul/2020:03:17:46 +0000] "GET /health HTTP/1.1" 200 17 "-" "kube-probe/1.16+"
::ffff:10.11.189.67 - - [22/Jul/2020:03:17:51 +0000] "GET /health HTTP/1.1" 200 17 "-" "kube-probe/1.16+"
::ffff:10.11.189.67 - - [22/Jul/2020:03:17:56 +0000] "GET /health HTTP/1.1" 200 17 "-" "kube-probe/1.16+"
::ffff:10.11.189.67 - - [22/Jul/2020:03:18:01 +0000] "GET /health HTTP/1.1" 200 17 "-" "kube-probe/1.16+"
::ffff:10.11.189.67 - - [22/Jul/2020:03:18:06 +0000] "GET /health HTTP/1.1" 200 17 "-" "kube-probe/1.16+"
::ffff:10.11.189.67 - - [22/Jul/2020:03:18:11 +0000] "GET /health HTTP/1.1" 200 17 "-" "kube-probe/1.16+"
::ffff:10.11.189.67 - - [22/Jul/2020:03:18:16 +0000] "GET /health HTTP/1.1" 200 17 "-" "kube-probe/1.16+"
::ffff:10.11.189.67 - - [22/Jul/2020:03:18:21 +0000] "GET /health HTTP/1.1" 200 17 "-" "kube-probe/1.16+"
::ffff:10.11.189.67 - - [22/Jul/2020:03:18:26 +0000] "GET /health HTTP/1.1" 200 17 "-" "kube-probe/1.16+"
```

## Readiness Probe 구성

### 1.Readiness Probe 구성을 포함하는 App 배포.

kubelet은 periodSeconds 필드를 사용하여 컨테이너를 상태를 확인합니다. 이 경우 kubelet은 3 초마다 readiness probe를 통해 확인합니다. initialDelaySeconds 필드는 첫 번째 Probe를 수행하기 전에 5 초 동안 기다려야한다는 것을 kubelet에 알리는 데 사용됩니다. 프로브를 수행하기 위해 kubelet은 /tmp/health 디렉토리가 존재하는 지를 확인합니다. 만약 없다면 해당 Pod에는 App을 배포하지 않습니다.

```
cat <<EoF > ~/environment/healthchecks/readiness-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: readiness-deployment
  namespace: healthchecks
spec:
  replicas: 3
  selector:
    matchLabels:
      app: readiness-deployment
  template:
    metadata:
      labels:
        app: readiness-deployment
    spec:
      containers:
      - name: readiness-deployment
        image: alpine
        command: ["sh", "-c", "touch /tmp/healthy && sleep 86400"]
        readinessProbe:
          exec:
            command:
            - cat
            - /tmp/healthy
          initialDelaySeconds: 5
          periodSeconds: 3
EoF
```

생성된 매니페스트를 사용하여 포드를 생성합니다.

```
kubectl -n healthchecks apply -f ~/environment/healthchecks/readiness-deployment.yaml

```

아래 명령을 통해 App배포와 Replica 상태를 확인해 봅니다.

```
kubectl -n healthchecks get pods -l app=readiness-deployment

```

```
kubectl -n healthchecks describe deployment readiness-deployment | grep Replicas:
```

아래와 같이 모두 정상적으로 수행되는 것을 확인할 수 있습니다.

```
whchoi98:~/environment/healthchecks $ kubectl -n healthchecks get pods -l app=readiness-deployment
NAME                                   READY   STATUS    RESTARTS   AGE
readiness-deployment-589b548d5-qhw6k   1/1     Running   0          103s
readiness-deployment-589b548d5-xbj47   1/1     Running   0          103s
readiness-deployment-589b548d5-xnmcm   1/1     Running   0          103s
whchoi98:~/environment/healthchecks $ kubectl -n healthchecks describe deployment readiness-deployment | grep Replicas:
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
```

### 2. App 장애 발생.

앞서 매니페스트 파일 내부에 포함된 Readiness Probe에는 반드시 `tmp/healthy` 디렉토리가 존재해야 Readiness Probe가 구동되도록 되어 있습니다. 해당 디렉토리를 삭제 시켜서 Readiness Fail을 발생시킵니다.

```
kubectl -n healthchecks exec -it  {pod-name} -- rm /tmp/healthy
```

아래 명령을 통해 App배포와 Replica 상태를 확인해 봅니다.

```
kubectl -n healthchecks get pods -l app=readiness-deployment

```

```
kubectl -n healthchecks describe deployment readiness-deployment | grep Replicas:
```

아래와 같이 출력 결과를 확인 할 수 있습니다.

```
# pod readiness-deployment-589b548d5-xnmcm가 정상작동하지 않습니다.
~/environment/healthchecks $ kubectl -n healthchecks get pods -l app=readiness-deployment
NAME                                   READY   STATUS    RESTARTS   AGE
readiness-deployment-589b548d5-qhw6k   1/1     Running   0          9m52s
readiness-deployment-589b548d5-xbj47   1/1     Running   0          9m52s
readiness-deployment-589b548d5-xnmcm   0/1     Running   0          9m52s

#Replicas에서 1개의 Pod가 가용하지 않음을 확인 할 수 있습니다.
~/environment/healthchecks $ kubectl -n healthchecks describe deployment readiness-deployment | grep Replicas:
Replicas:               3 desired | 3 updated | 3 total | 2 available | 1 unavailable
```

{% hint style="info" %}
1개의 Pod에서 컨테이너가 /tmp/healthy 디렉토리가 발견되지 않습니다. 따라서 더이상 Replica에 가용한 Pod가 아닙니다.
{% endhint %}

### 3. App 복구를 통한 Readiness Probe 확인.

앞서 Pod에서 삭제한 디렉토리를 다시 생성합니다.

```
kubectl -n healthchecks exec -it {pod-name} -- touch /tmp/healthy
```

디렉토리를 생성한 후 Pod의 상태와 Replica 가용 숫자를 확인해 봅니다.

```
kubectl -n healthchecks get pods -l app=readiness-deployment
kubectl -n healthchecks describe deployment readiness-deployment | grep Replicas:
```

아래와 같은 결과를 얻을 수 있습니다.

```
whchoi98:~/environment/healthchecks $ kubectl -n healthchecks get pods -l app=readiness-deployment
NAME                                   READY   STATUS    RESTARTS   AGE
readiness-deployment-589b548d5-qhw6k   1/1     Running   0          21m
readiness-deployment-589b548d5-xbj47   1/1     Running   0          21m
readiness-deployment-589b548d5-xnmcm   1/1     Running   0          21m
whchoi98:~/environment/healthchecks $ kubectl -n healthchecks describe deployment readiness-deployment | grep Replicas:
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
```









\
