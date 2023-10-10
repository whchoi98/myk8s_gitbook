# istio 모니터링

### 1. 모니터링 도구 소개&#x20;

Istio binary에는 기본 모니터링 도구로 Prometheus, Grafana, Jaeger, Kiali가 포함되어 있습니다.&#x20;

Jaeger는 오픈 소스 종단 간 분산 추적 시스템으로 사용자가 복잡한 분산 시스템을 모니터링하고 문제를 해결할 수 있도록 합니다. Jaeger는 분산 트랜잭션 모니터링, 성능 및 대기 시간 최적화, 근본 원인 및 서비스 종속성 분석 등의 문제를 해결합니다. ([https://www.jaegertracing.io/](https://www.jaegertracing.io/))

Kiali는 Istio 기반 서비스 메시용 관리 콘솔입니다. 대시보드, Observability을 제공하고 강력한 구성 및 검증 기능으로 메시를 운영할 수 있습니다. 트래픽 토폴로지를 유추하여 서비스 메시의 구조를 표시하고 메시의 상태를 표시합니다. Kiali는 상세한 메트릭, 강력한 유효성 검사, Grafana 액세스 및 Jaeger와의 분산 추적을 위한 강력한 통합을 제공합니다. 공식 사이트를 방문하여 제공하는 기능을 볼 수 있습니다. ([https://kiali.io/documentation/latest/features/](https://kiali.io/documentation/latest/features/))

### 2.설치&#x20;

아래와 같이 설치를 구성합니다.

```
kubectl apply -f ${HOME}/environment/istio-${ISTIO_VERSION}/samples/addons
kubectl -n istio-system get pods
kubectl -n istio-system get deploy jaeger kiali prometheus grafana

```

```
$ kubectl -n istio-system get pods
NAME                                    READY   STATUS    RESTARTS   AGE
grafana-6c5dc6df7c-fxfp9                1/1     Running   0          2m59s
istio-egressgateway-859f895667-79rd6    1/1     Running   0          4h35m
istio-ingressgateway-6565cc5469-frp4w   1/1     Running   0          4h35m
istiod-54588fb7bf-286r7                 1/1     Running   0          4h36m
jaeger-9dd685668-gtjmf                  1/1     Running   0          2m59s
kiali-699f98c497-2t6vj                  1/1     Running   0          2m58s
prometheus-699b7cc575-9dzdz             2/2     Running   0          2m58s
```

### 3. 트래픽 생성&#x20;

cloud9의 터미널을 하나를 새로 생성하고 , 새로운 탭에서 다음 명령을 사용하여 트래픽을 메시로 보냅니다.

```
export GATEWAY_URL=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
watch --interval 1 curl -s -I -XGET "http://${GATEWAY_URL}/productpage"

```

Kiali를 시작하여 애플리케이션 추적 및 메트릭을 시각화합니다.

### 4. Kiali&#x20;

Cloud9에서 새 터미널 탭을 열고 다음 명령을 실행하여 kiali 대시보드를 시작합니다.&#x20;

```
kubectl -n istio-system port-forward \
$(kubectl -n istio-system get pod -l app=kiali -o jsonpath='{.items[0].metadata.name}') 8080:20001

```

Kiali 대시보드를 열기 위해, 아래 처럼 Cloud9 환경에서 "Preview Running Application"을 클릭합니다.&#x20;

![](<../../.gitbook/assets/image (399).png>)

새로운 창을 확대해서, 실행 시킵니다.&#x20;

![](<../../.gitbook/assets/image (60).png>)

아래와 같이 확인 할 수 있습니다.&#x20;

* Graph

![](<../../.gitbook/assets/image (361).png>)

* Workloads

![](<../../.gitbook/assets/image (382).png>)

![](<../../.gitbook/assets/image (400).png>)

5\. Grafana Dashboard 런칭

현재 Cloud9 IDE는 실행 중인 여러 애플리케이션의 미리보기를 지원하지 않습니다. Grafana를 실행하기 전에 Kiali 리스너를 중지해야합니다.&#x20;

Grafana를 사용하여 애플리케이션 메트릭을 시각화합니다. 다음 명령을 실행하여 새 터미널 탭을 열고 Grafana용 포트 포워딩을 설정합니다.

```
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 8080:3000

```

Grafana 대시보드를 열기 위해, 아래 처럼 Cloud9 환경에서 "Preview Running Application"을 클릭합니다.&#x20;

![](<../../.gitbook/assets/image (434).png>)
