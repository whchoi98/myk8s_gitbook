---
description: 'Update : 2021-04-08'
---

# Prometheus-Grafana

## Prometheus 소개

[Prometheus](https://github.com/prometheus) 는 원래 [SoundCloud에](https://soundcloud.com/) 구축 된 오픈 소스 시스템 모니터링 및 alerting tool kit 입니다. 2012 년에 시작된 이래로 많은 회사와 조직에서 Prometheus를 채택했으며이 프로젝트에는 매우 활발한 개발자 및 사용자 [커뮤니티가](https://prometheus.io/community) 있습니다.  프로젝트의 거버넌스 구조를 명확하게하기 위해 Prometheus는 2016 년 [Kubernetes](https://kubernetes.io/) 에 이어 두 번째 호스팅 프로젝트로 [Cloud Native Computing Foundation](https://cncf.io/) 에 합류했습니다 .

### 1.주요 특징

* 메트릭 이름 및 key -value pair로 식별 된 시계열 데이터가 포함 된 다차원 [데이터 모델](https://prometheus.io/docs/concepts/data_model/)
* 다차원 데이터 모델 활용 하는 [유연한 쿼리 언어 ](https://prometheus.io/docs/prometheus/latest/querying/basics/)PromQL 사
* 분산 스토리지에 의존성이 없음.
* HTTP pull model을 통해 시계열 수집.
* Intermediary G.W를 통해 [푸시 시계열](https://prometheus.io/docs/instrumenting/pushing/) 지원
* 서비스 디스커버 또는 정적 구성을 통해 타켓을 식별.
* 다양한 모드의 그래프 및 대시 보드 지원
* 대부분의 주요언어가 Go로 작성.

### 2.주요 구성 요소 및 아키텍쳐

* [Prometheus ](https://github.com/prometheus/prometheus): 시계열 데이터를 스크랩하고 저장 하는 주요 [서버](https://github.com/prometheus/prometheus)
* [클라이언트 라이브러리](https://prometheus.io/docs/instrumenting/clientlibs/) : 계측용 프로그램 코드
* [푸쉬 게이트웨이](https://github.com/prometheus/pushgateway) : shot lived job 지원
* HAProxy, StatsD, Graphite 등과 같은 서비스를위한 특수 목적의 [export](https://prometheus.io/docs/instrumenting/exporters/)
* [alertmanager](https://github.com/prometheus/alertmanager) : alert 제공. 
* 기타 다양한 지원 도구들.

## Prometheus 구성

### 1.Prometheus 설치. 

앞서 [Helm Chart](../eks-2/helm.md#1-helm)를 설치하였습니다. Helm Chart를 활용해서 설치합니다.

```text
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

```

아래와 같이 prometheus namespace를 만들고, helm을 통해 설치합니다. 

```text
kubectl create namespace prometheus
helm upgrade -i prometheus prometheus-community/prometheus \
    --namespace prometheus \
    --set alertmanager.persistentVolume.storageClass="gp2",server.persistentVolume.storageClass="gp2"
```

아래와 같은 결과를 얻을 수 있습니다.

```text
$ helm install prometheus stable/prometheus \
>     --namespace prometheus \
>     --set alertmanager.persistentVolume.storageClass="gp3" \
>     --set server.persistentVolume.storageClass="gp3"
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

### 2.Prometheus 실행 및 접속. 

Cloud9 터미널에서 백그라운드로 실행시킵니다.

```text
kubectl port-forward -n prometheus deploy/prometheus-server 8080:9090 &

```

외부에서 접속되지 않으므로, Cloud9 IDE에서 Preview를 통해 Proxy로 접속합니다. \([K8s Dashboard ](k8s-dashboard.md)접속 형태와 유사합니다.\)

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

## Grafana 구성

### 1.Grafana 설치 

Grafana를 사용하게 되면 시계열 메트릭 데이터를 질의, 시각화, Alert을 이해하는 데 사용할 수 있습니다.

#### Grafana Demo 공식 사이트 - [https://play.grafana.org/](https://play.grafana.org/)

먼저 Grafana설치를 위해서 몇가지 변수를 정의하고, 설치를 위한 매니페스트 파일을 작성합니다.

```text
mkdir ~/environment/grafana
kubectl create namespace grafana
cat <<EoF > ~/environment/grafana/grafana.yaml
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-server.prometheus.svc.cluster.local
      access: proxy
      isDefault: true
EoF
```

helm chart를 통해 설치를 위해, repo 검색을 합니다.

```text
helm search repo grafana

```

아래와 같은 결과를 얻을 수 있습니다.

```text
whchoi98:~/environment/grafana $ helm search repo grafana
NAME            CHART VERSION   APP VERSION     DESCRIPTION                                       
bitnami/grafana 3.1.2           7.1.0           Grafana is an open source, feature rich metrics...
stable/grafana  5.4.1           7.0.5           The leading tool for querying and visualizing t...
```

이제 helm 을 통해 grafana를 설치합니다. 설치시에 옵션을 통해 prometheus 설치와 동일하게 storage type을 설정하고, 생성한 매니페스트를 불러옵니다.  또한 외부에서 접속을 위해서 Loadbalacer type을 지정합니다.

```text
cd ~/environment/grafana/
helm install grafana stable/grafana \
    --namespace grafana \
    --set persistence.storageClassName="gp2" \
    --set persistence.enabled=true \
    --set adminPassword='1234Qwer' \
    --values grafana.yaml \
    --set service.type=LoadBalancer

```

아래와 같이 helm을 통해 설치된 결과를 확인 할 수 있습니다. 

```text
whchoi98:~/environment/grafana $ helm install grafana stable/grafana \
>     --namespace grafana \
>     --set persistence.storageClassName="gp2" \
>     --set persistence.enabled=true \
>     --set adminPassword='1234Qwer' \
>     --values grafana.yaml \
>     --set service.type=LoadBalancer
NAME: grafana
LAST DEPLOYED: Thu Jul 23 10:39:13 2020
NAMESPACE: grafana
STATUS: deployed
REVISION: 1
NOTES:
1. Get your 'admin' user password by running:

   kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

2. The Grafana server can be accessed via port 80 on the following DNS name from within your cluster:

   grafana.grafana.svc.cluster.local

   Get the Grafana URL to visit by running these commands in the same shell:
NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        You can watch the status of by running 'kubectl get svc --namespace grafana -w grafana'
     export SERVICE_IP=$(kubectl get svc --namespace grafana grafana -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
     http://$SERVICE_IP:80

3. Login with the password from step 1 and the username: admin
```

 다음 명령을 통해 , grafana 설치를 확인합니다.

```text
kubectl -n grafana get all
```

출력결과 예시는 다음과 같습니다.

```text
whchoi98:~/environment/grafana $ kubectl -n grafana get all
NAME                           READY   STATUS    RESTARTS   AGE
pod/grafana-6744db7855-nr5b5   1/1     Running   0          6m44s

NAME              TYPE           CLUSTER-IP      EXTERNAL-IP                                                                    PORT(S)        AGE
service/grafana   LoadBalancer   172.20.23.245   a555fefad0ed8493fb4a9ec240318103-2014381236.ap-northeast-2.elb.amazonaws.com   80:31156/TCP   6m44s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grafana   1/1     1            1           6m44s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/grafana-6744db7855   1         1         1       6m44s
```

service를 확인합니다.

```text
kubectl get svc -n grafana grafana
```

### 2.Grafana 접속 확인. 

출력 결과에서 제공되는 Loadbalancer 주소를 브라우져에서 입력합니다.

```text
whchoi98:~/environment/grafana $ kubectl -n grafana get svc
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP                                                                    PORT(S)        AGE
grafana   LoadBalancer   172.20.23.245   a555fefad0ed8493fb4a9ec240318103-2014381236.ap-northeast-2.elb.amazonaws.com   80:31156/TCP   3h12m
```

앞서 grafana.yaml에서 입력한 admin의 패스워드를 입력합니다.

```text
--set adminPassword='1234Qwer'
```

![](../.gitbook/assets/image%20%2882%29.png)

## DashBoard 구성

### 1.Cluster Monitoring

이제 생성된 Grafana 에서 배포된 Cluster들에 대해서 모니터링을 합니다.

좌측 상단 메뉴의 "+" 를 선택하고 Import를 선택합니다.

![](../.gitbook/assets/image%20%2873%29.png)

Import 값을 "3119"를 선택하고, Load를 선택합니다.

![](../.gitbook/assets/image%20%2886%29.png)

DataSource를 Prometheus를 선택합니다.

![](../.gitbook/assets/image%20%2875%29.png)

아래와 같이 다양한 Cluster 내부의 정보를 확인 할 수 있습니다.

![](../.gitbook/assets/image%20%2884%29.png)

### 2.Pod Monitoring 구성

이제 생성된 Grafana 에서 배포된 Pod들에 대해서 모니터링을 합니다.

좌측 상단 메뉴의 "+" 를 선택하고 Import를 선택합니다.

![](../.gitbook/assets/image%20%2873%29.png)

Import 값을 "6417"를 선택하고, Load를 선택합니다.

![](../.gitbook/assets/image%20%2883%29.png)

Name : Kubernetes Pods Monitoring , Change uid, 데이터 소스 : Prometheus 를 선택하고, Import를 선택합니다.

![](../.gitbook/assets/image%20%2876%29.png)

Pod들 중심으로 모니터링을 할 수 있습니다.

![](../.gitbook/assets/image%20%2870%29.png)

### 3. Grafana Labs Dashboard 활용하기

위에서 활용한 것 처럼, 이미 템플릿으로 생성한 ID를 가져와서 구성해 봅니다.

[Grafana Labs Dashboard](https://grafana.com/grafana/dashboards)에 접속합니다. \([https://grafana.com/grafana/dashboards](https://grafana.com/grafana/dashboards)\)

아래와 같이 Filter를 통해서 유용한 Dashboard를 가져옵니다. \(Name : kubernetes, Data Source : Prometheus\)

![](../.gitbook/assets/image%20%2869%29.png)

* Kubernetes Deployment Statefulset Daemonset metrics

```text
8588
```

![](../.gitbook/assets/image%20%2872%29.png)

* Kubernetes Cluster

```text
7249
```

![](../.gitbook/assets/image%20%2888%29.png)

* 1 Kubernetes cluster overview

```text
11802
```

![](../.gitbook/assets/image%20%2881%29.png)

Kubernetes / Networking / Pod

```text
12661
```

![](../.gitbook/assets/image%20%2874%29.png)

