# Prometheus-Grafana

## 1.Prometheus 소개

[Prometheus](https://github.com/prometheus) 는 원래 [SoundCloud에](https://soundcloud.com/) 구축 된 오픈 소스 시스템 모니터링 및 alerting tool kit 입니다. 2012 년에 시작된 이래로 많은 회사와 조직에서 Prometheus를 채택했으며이 프로젝트에는 매우 활발한 개발자 및 사용자 [커뮤니티가](https://prometheus.io/community) 있습니다.  프로젝트의 거버넌스 구조를 명확하게하기 위해 Prometheus는 2016 년 [Kubernetes](https://kubernetes.io/) 에 이어 두 번째 호스팅 프로젝트로 [Cloud Native Computing Foundation](https://cncf.io/) 에 합류했습니다 .

주요 특징

* 메트릭 이름 및 key -value pair로 식별 된 시계열 데이터가 포함 된 다차원 [데이터 모델](https://prometheus.io/docs/concepts/data_model/)
* 다차원 데이터 모델 활용 하는 [유연한 쿼리 언어 ](https://prometheus.io/docs/prometheus/latest/querying/basics/)PromQL 사
* 분산 스토리지에 의존성이 없음.
* HTTP pull model을 통해 시계열 수집.
* Intermediary G.W를 통해 [푸시 시계열](https://prometheus.io/docs/instrumenting/pushing/) 지원
* 서비스 디스커버 또는 정적 구성을 통해 타켓을 식별.
* 다양한 모드의 그래프 및 대시 보드 지원
* 대부분의 주요언어가 Go로 작성.

주요 구성 요소 및 아키텍쳐

* [Prometheus ](https://github.com/prometheus/prometheus): 시계열 데이터를 스크랩하고 저장 하는 주요 [서버](https://github.com/prometheus/prometheus)
* [클라이언트 라이브러리](https://prometheus.io/docs/instrumenting/clientlibs/) : 계측용 프로그램 코드
* [푸쉬 게이트웨이](https://github.com/prometheus/pushgateway) : shot lived job 지원
* HAProxy, StatsD, Graphite 등과 같은 서비스를위한 특수 목적의 [export](https://prometheus.io/docs/instrumenting/exporters/)
* [alertmanager](https://github.com/prometheus/alertmanager) : alert 제공. 
* 기타 다양한 지원 도구들.

![&#xCC38;&#xC870; - https://prometheus.io/docs/introduction/overview/](../.gitbook/assets/image%20%2867%29.png)

## 2. Prometheus 구성

앞서 [Helm Chart](helm.md#1-helm)를 설치하였습니다. Helm Chart를 활용해서 설치합니다.

```text
helm search repo stable/prometheus
```

아래와 같이 prometheus namespace를 만들고, helm을 통해 설치합니다. 앞서 랩에서 이미 Worker node의 스토리지 타입은 gp2 타입으로 배포되었습니다.

```text
kubectl create namespace prometheus
helm install prometheus stable/prometheus \
    --namespace prometheus \
    --set alertmanager.persistentVolume.storageClass="gp2" \
    --set server.persistentVolume.storageClass="gp2"
```

아래와 같은 결과를 얻을 수 있습니다.

```text
whchoi98:~ $ helm install prometheus stable/prometheus \
>     --namespace prometheus \
>     --set alertmanager.persistentVolume.storageClass="gp2" \
>     --set server.persistentVolume.storageClass="gp2"
NAME: prometheus
LAST DEPLOYED: Thu Jul 23 08:19:32 2020
NAMESPACE: prometheus
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The Prometheus server can be accessed via port 80 on the following DNS name from within your cluster:
prometheus-server.prometheus.svc.cluster.local


Get the Prometheus server URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace prometheus -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace prometheus port-forward $POD_NAME 9090

```

이제 Prometheus가 정상적으로 배포되었는지 확인합니다.

```text
kubectl -n prometheus get all 
```

앞서 소개한 주요 컴포넌트들이 Pod형태로 배포된 것을 확인 할 수 있습니다. 앞서 아키텍쳐 도식도와 비교해서 확인해 봅니다.

```text
whchoi98:~ $ kubectl -n prometheus get all 
NAME                                                 READY   STATUS    RESTARTS   AGE
pod/prometheus-alertmanager-6d6469d9bb-mqlk9         2/2     Running   0          6m45s
pod/prometheus-kube-state-metrics-6df5d44568-66rm8   1/1     Running   0          6m45s
pod/prometheus-node-exporter-287wp                   1/1     Running   0          6m45s
pod/prometheus-node-exporter-7t82k                   1/1     Running   0          6m45s
pod/prometheus-node-exporter-n5gcw                   1/1     Running   0          6m45s
pod/prometheus-node-exporter-spc56                   1/1     Running   0          6m45s
pod/prometheus-node-exporter-w5l5d                   1/1     Running   0          6m45s
pod/prometheus-node-exporter-xcxgj                   1/1     Running   0          6m45s
pod/prometheus-pushgateway-6d65c95d5-xwvhh           1/1     Running   0          6m45s
pod/prometheus-server-869d4b999b-6cbtk               2/2     Running   0          6m45s

NAME                                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/prometheus-alertmanager         ClusterIP   172.20.182.229   <none>        80/TCP     6m45s
service/prometheus-kube-state-metrics   ClusterIP   172.20.121.112   <none>        8080/TCP   6m45s
service/prometheus-node-exporter        ClusterIP   None             <none>        9100/TCP   6m45s
service/prometheus-pushgateway          ClusterIP   172.20.197.145   <none>        9091/TCP   6m45s
service/prometheus-server               ClusterIP   172.20.98.8      <none>        80/TCP     6m45s

NAME                                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/prometheus-node-exporter   6         6         6       6            6           <none>          6m45s

NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prometheus-alertmanager         1/1     1            1           6m45s
deployment.apps/prometheus-kube-state-metrics   1/1     1            1           6m45s
deployment.apps/prometheus-pushgateway          1/1     1            1           6m45s
deployment.apps/prometheus-server               1/1     1            1           6m45s

NAME                                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/prometheus-alertmanager-6d6469d9bb         1         1         1       6m45s
replicaset.apps/prometheus-kube-state-metrics-6df5d44568   1         1         1       6m45s
replicaset.apps/prometheus-pushgateway-6d65c95d5           1         1         1       6m45s
replicaset.apps/prometheus-server-869d4b999b               1         1         1       6m45s
```

Cloud9 터미널에서 백그라운드로 실행시킵니다.

```text
kubectl port-forward -n prometheus deploy/prometheus-server 8080:9090 &

```

외부에서 접속되지 않으므로, Cloud9 IDE에서 Preview를 통해 Proxy로 접속합니다. \([K8s Dashboard ](../eks-1/k8s-dashboard.md)접속 형태와 유사합니다.\)

Cloud9의 상단 메뉴 Preview - Preview Running Application을 선택합니다. 메뉴에서 보이지 않는 경우 Tools - Preview - Preview Running Application을 선택합니다.

![](../.gitbook/assets/image%20%2816%29.png)

생선된 Preview 브라우져에서 새로운 윈도우를 선택합니다.

![](../.gitbook/assets/image%20%282%29.png)

전체 화면 창에서 아래와 같이 마지막에 URL을 추가합니다.

```text
/targets
```

아래와 같은 결과를 브라우져에서 확인할 수 있습니다.Status 메뉴에서 다양한 결과를 확인 할 수 있습니다.

![](../.gitbook/assets/image%20%2866%29.png)

## 3. Grafana 구성

Grafana를 사용하게 되면 시계열 메트릭 데이터를 질의, 시각화, Alert을 이해하는 데 사용할 수 있습니다.

#### Grafana Demo 공식 사이트 - [https://play.grafana.org/](https://play.grafana.org/)



```text
mkdir ~/environment/
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



## 4. DashBoard 구성







