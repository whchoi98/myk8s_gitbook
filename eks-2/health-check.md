# Health Check 구성

## 컨테이너 프로브 소개

[프로브](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#probe-v1-core)는 컨테이너에서 [kubelet](https://kubernetes.io/docs/admin/kubelet/)에 의해 주기적으로 수행되는 진단\(diagnostic\) 도구입니. 진단을 수행하기 위해서, kubelet은 컨테이너에 의해서 구현된 [핸들러](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#handler-v1-core)를 호출합니다.

핸들러는 다음과 같이 세가지 타입이 있습니다.

* [ExecAction](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#execaction-v1-core) 은 컨테이너 내에서 지정된 명령어를 실행합니. 명령어가 상태 코드 0으로 종료되면 진단이 성공한 것으로 판단합니다.
* [TCPSocketAction](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#tcpsocketaction-v1-core) 은 지정된 포트에서 컨테이너의 IP주소에 대해 TCP 검사를 수행합니. 포트가 활성화되어 있다면 진단이 성공한 것으로 간주합니다.
* [HTTPGetAction](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#httpgetaction-v1-core) 은 지정한 포트 및 경로에서 컨테이너의 IP주소에 대한 HTTP Get 요청을 수행합니. 응답의 상태 코드가 200보다 크고 400보다 작으면 진단이 성공한 것으로 간주합니다.

각 probe는 다음 세 가지 결과 중 하나를 가집니.

* Success: 컨테이너가 진단을 통과함.
* Failure: 컨테이너가 진단에 실패함.
* Unknown: 진단 자체가 실패하였으므로 아무런 액션도 수행되면 안됨.

kubelet은 실행 중인 컨테이너들에 대해서 선택적으로 세 가지 종류의 프로브를 수행하고 그에 반응할 수 있습니.

* `livenessProbe`: 컨테이너가 동작 중인지 여부를 나타냅니. 만약 활성 프로브\(liveness probe\)에 실패한다면, kubelet은 컨테이너를 죽이고, 해당 컨테이너는 [재시작 정책](https://kubernetes.io/ko/docs/concepts/workloads/pods/pod-lifecycle/#%EC%9E%AC%EC%8B%9C%EC%9E%91-%EC%A0%95%EC%B1%85)의 대상이 됩니. 만약 컨테이너가 활성 프로브를 제공하지 않는 경우, 기본 상태는 `Success`입니다.
* `readinessProbe`: 컨테이너가 요청을 처리할 준비가 되었는지 여부를 나타냅니. 만약 `readinessProbe`

   실패한다면, 엔드포인트 컨트롤러는 파드에 연관된 모든 서비스들의 엔드포인트에서 파드의 IP주소를 제거합니다. 

  `readinessProbe`의 초기 지연 이전의 기본 상태는 `Failure`입니다. 만약 컨테이너가 `readinessProbe`

  를 지원하지 않는다면, 기본 상태는 `Success`입니다.

* `startupProbe`: 컨테이너 내의 애플리케이션이 시작되었는지를 나타냅니다. `startupProbe`가 주어진 경우, 성공할 때 까지 다른 나머지 프로브는 활성화 되지 않습니. 만약 `startupProbe`실패하면, kubelet이 컨테이너를 죽이고, 컨테이너는 [재시작 정책](https://kubernetes.io/ko/docs/concepts/workloads/pods/pod-lifecycle/#%EC%9E%AC%EC%8B%9C%EC%9E%91-%EC%A0%95%EC%B1%85)에 따라 처리됩니. 컨테이너에 스타트업 프로브가 없는 경우, 기본 상태는 `Success`입니다.

Liveness Probe 구성

1.Liveness Probe를 위한 매니페스트 구성



```text
mkdir -p ~/environment/healthchecks
```

kubelet은 periodSeconds 필드를 사용하여 컨테이너를 상태를 확인합니다. 이 경우 kubelet은 5 초마다 liveness probe를 통해 확인합니다. initialDelaySeconds 필드는 첫 번째 Probe를 수행하기 전에 5 초 동안 기다려야한다는 것을 kubelet에 알리는 데 사용됩니다. 프로브를 수행하기 위해 kubelet은 이 포드를 호스팅 하는 서버에 HTTP GET 요청을 전송하고 서버 / health의 핸들러가 성공 코드를 반환하면 컨테이너가 정상으로 간주됩니다. 핸들러가 실패 코드를 반환하면 kubelet은 컨테이너를 종료하고 다시 시작합니다. 

```text
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

```text
kubectl create namespace healthchecks
kubectl apply -f ~/environment/healthchecks/liveness-app.yaml
kubectl -n healthchecks get pod liveness-app
```

출력 결과 예제

```text
whchoi98:~ $ kubectl -n healthchecks get pod liveness-app
NAME           READY   STATUS    RESTARTS   AGE
liveness-app   1/1     Running   0          14s
```



Readiness Probe 구성





  


