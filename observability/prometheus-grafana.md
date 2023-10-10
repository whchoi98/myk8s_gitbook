---
description: 'Update : 2023-07-23'
---

# Prometheus-Grafana

## Prometheus 소개

[Prometheus](https://github.com/prometheus) 는 원래 [SoundCloud에](https://soundcloud.com/) 구축 된 오픈 소스 시스템 모니터링 및 alerting tool kit 입니다. 2012 년에 시작된 이래로 많은 회사와 조직에서 Prometheus를 채택했으며이 프로젝트에는 매우 활발한 개발자 및 사용자 [커뮤니티가](https://prometheus.io/community) 있습니다.  프로젝트의 거버넌스 구조를 명확하게하기 위해 Prometheus는 2016 년 [Kubernetes](https://kubernetes.io/) 에 이어 두 번째 호스팅 프로젝트로 [Cloud Native Computing Foundation](https://cncf.io/) 에 합류했습니다 .

### 1.주요 특징

* 메트릭 이름 및 key -value pair로 식별 된 시계열 데이터가 포함 된 다차원 [데이터 모델](https://prometheus.io/docs/concepts/data\_model/)
* 다차원 데이터 모델 활용 하는 [유연한 쿼리 언어 ](https://prometheus.io/docs/prometheus/latest/querying/basics/)PromQL 사
* 분산 스토리지에 의존성이 없음.
* HTTP pull model을 통해 시계열 수집.
* Intermediary G.W를 통해 [푸시 시계열](https://prometheus.io/docs/instrumenting/pushing/) 지원
* 서비스 디스커버 또는 정적 구성을 통해 타켓을 식별.
* 다양한 모드의 그래프 및 대시 보드 지원
* 대부분의 주요언어가 Go로 작성.

### 2.주요 구성 요소 및 아키텍쳐

* [Prometheus ](https://github.com/prometheus/prometheus): 시계열 데이터를 스크랩하고 저장 하는 주요 [서버](https://github.com/prometheus/prometheus)

오픈소스 모니터링 툴로 지표 수집을 통한 모니터링이 주요 기능입니다.**지표(Metric)**를 수집하여 모니터링 할 수 있고, 기본적으로 **Pull 방식**으로 데이터를 수집하는데, 이 말은 모니터링 대상이 되는 자원이 지표정보를 프로메테우스로 보내는 것이 아니라, 프로메테우스가 주기적으로 모니터링 대상에서 지표를 읽어온다는 뜻입니다.

Pull 방식으로 지표정보를 읽어올때는 각 서버에 설치된 Exporter를 통해서 정보를 읽어오며, 배치나 스케쥴 작업의 경우에는 필요한 경우에만 떠 있다가 작업이 끝나면 사라지기 때문에 Pull 방식으로 데이터 수집이 어워서, 그럴 경우 Push방식을 사용하는 Push gateway를 통해 지표정보를 받아오는 경우도 있습니다 .

오토스케일링이 많이 사용되는 클라우드 환경이나 쿠버네티스 클러스터에서는 모니터링 대상의 IP가 동적으로 변경되기 때문에 이를 일일이 설정파일에 넣는데 한계가 있기때문에, 이러한 문제를 해결하기 위해 프로메테우스는 DNS나 Consul, etcd와 같은 다양한 서비스 디스커버리 서비스와 연동을 통해 모니터링 목록을 가지고 모니터링을 수행합니다.

* [클라이언트 라이브러리](https://prometheus.io/docs/instrumenting/clientlibs/) : 계측용 프로그램 코드
* [푸쉬 게이트웨이](https://github.com/prometheus/pushgateway) : shot lived job 지원
* HAProxy, StatsD, Graphite 등과 같은 서비스를위한 특수 목적의 [export](https://prometheus.io/docs/instrumenting/exporters/)
* [alertmanager](https://github.com/prometheus/alertmanager) : alert 제공. 프로메테우스로부터 alert를 전달받아 이를 적절한 포맷으로 가공하여 notify 해주는 역할 수행.
* node-exporter : 모니터링 대상이 프로메테우스의 데이터 포맷을 지원하지 않는 경우에는 별도의 에이전트를 설치해야 지표를 얻어올 수 있는데 이 에이전트를 Exporter라고 합니다 . 쿠버네티스 컨테이너 모니터링을 진행할 경우 node-exporter를 사용합니다 .&#x20;

<figure><img src="../.gitbook/assets/image (107).png" alt=""><figcaption><p>[출처 - <a href="https://prometheus.io/docs/introduction/overview/">https://prometheus.io/docs/introduction/overview/</a>]</p></figcaption></figure>

## Prometheus 구성

### 3.Prometheus 설치.&#x20;

앞서 [Helm Chart](../eks-2/helm.md#1-helm)를 설치하였습니다. Helm Chart를 활용해서 설치합니다.

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

```

Prometheus와 Grafana를 설치할 Storage Class를 구성합니다.

```
mkdir -p ~/environment/ebs_csi/
cat <<EOF> ~/environment/ebs_csi/ebs_obs_sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-obs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
EOF

kubectl apply -f ~/environment/ebs_csi/ebs_obs_sc.yaml

```

아래와 같이 prometheus namespace를 만들고, helm을 통해 설치합니다.&#x20;

```
kubectl create namespace prometheus
helm upgrade -i prometheus prometheus-community/prometheus \
    --namespace prometheus \
    --set alertmanager.persistentVolume.storageClass="ebs-obs-sc",server.persistentVolume.storageClass="ebs-obs-sc"
```

아래와 같은 결과를 얻을 수 있습니다.

```
$ helm upgrade -i prometheus prometheus-community/prometheus \
>     --namespace prometheus \
>     --set alertmanager.persistentVolume.storageClass="gp2",server.persistentVolume.storageClass="gp2"
Release "prometheus" does not exist. Installing it now.
NAME: prometheus
LAST DEPLOYED: Thu Oct 21 15:37:08 2021
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


The Prometheus alertmanager can be accessed via port 80 on the following DNS name from within your cluster:
prometheus-alertmanager.prometheus.svc.cluster.local


Get the Alertmanager URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace prometheus -l "app=prometheus,component=alertmanager" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace prometheus port-forward $POD_NAME 9093
#################################################################################
######   WARNING: Pod Security Policy has been moved to a global property.  #####
######            use .Values.podSecurityPolicy.enabled with pod-based      #####
######            annotations                                               #####
######            (e.g. .Values.nodeExporter.podSecurityPolicy.annotations) #####
#################################################################################


The Prometheus PushGateway can be accessed via port 9091 on the following DNS name from within your cluster:
prometheus-pushgateway.prometheus.svc.cluster.local


Get the PushGateway URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace prometheus -l "app=prometheus,component=pushgateway" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace prometheus port-forward $POD_NAME 9091

For more information on running Prometheus, visit:
https://prometheus.io/
```

이제 Prometheus가 정상적으로 배포되었는지 확인합니다.

```
kubectl -n prometheus get all

```

앞서 소개한 주요 컴포넌트들이 Pod형태로 배포된 것을 확인 할 수 있습니다. 앞서 아키텍쳐 도식도와 비교해서 확인해 봅니다. Pod 들이 완전하게 배포될때 까지 수분이 걸립니다. &#x20;

```
$ kubectl -n prometheus get all
NAME                                                 READY   STATUS    RESTARTS   AGE
pod/prometheus-alertmanager-8784c78d9-fvr2t          2/2     Running   0          89s
pod/prometheus-kube-state-metrics-569d7854c4-kxx8k   1/1     Running   0          89s
pod/prometheus-node-exporter-7kt42                   1/1     Running   0          89s
pod/prometheus-node-exporter-cp8lp                   1/1     Running   0          89s
pod/prometheus-node-exporter-crsvj                   1/1     Running   0          89s
pod/prometheus-node-exporter-h9q9c                   1/1     Running   0          89s
pod/prometheus-node-exporter-j2rjd                   1/1     Running   0          89s
pod/prometheus-node-exporter-jt6mc                   1/1     Running   0          89s
pod/prometheus-node-exporter-krtbs                   1/1     Running   0          89s
pod/prometheus-node-exporter-mlpw6                   1/1     Running   0          89s
pod/prometheus-node-exporter-pd6pj                   1/1     Running   0          89s
pod/prometheus-node-exporter-rrd6g                   1/1     Running   0          89s
pod/prometheus-node-exporter-vqv6b                   1/1     Running   0          89s
pod/prometheus-node-exporter-zjfb9                   1/1     Running   0          89s
pod/prometheus-pushgateway-5c79b789d9-n58pk          1/1     Running   0          89s
pod/prometheus-server-f8d9475d7-4kxth                2/2     Running   0          89s

NAME                                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/prometheus-alertmanager         ClusterIP   172.20.34.60     <none>        80/TCP     89s
service/prometheus-kube-state-metrics   ClusterIP   172.20.208.234   <none>        8080/TCP   89s
service/prometheus-node-exporter        ClusterIP   None             <none>        9100/TCP   89s
service/prometheus-pushgateway          ClusterIP   172.20.170.184   <none>        9091/TCP   89s
service/prometheus-server               ClusterIP   172.20.210.188   <none>        80/TCP     89s

NAME                                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/prometheus-node-exporter   12        12        12      12           12          <none>          89s

NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prometheus-alertmanager         1/1     1            1           89s
deployment.apps/prometheus-kube-state-metrics   1/1     1            1           89s
deployment.apps/prometheus-pushgateway          1/1     1            1           89s
deployment.apps/prometheus-server               1/1     1            1           89s

NAME                                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/prometheus-alertmanager-8784c78d9          1         1         1       89s
replicaset.apps/prometheus-kube-state-metrics-569d7854c4   1         1         1       89s
replicaset.apps/prometheus-pushgateway-5c79b789d9          1         1         1       89s
replicaset.apps/prometheus-server-f8d9475d7                1         1         1       89s
```

### 4.Prometheus 실행 및 접속.&#x20;

Cloud9 터미널에서 백그라운드로 실행시킵니다.

```
kubectl port-forward -n prometheus deploy/prometheus-server 8081:9090 &

```

외부에서 접속되지 않으므로, Cloud9 IDE에서 Preview를 통해 Proxy로 접속합니다. ([K8s Dashboard ](k8s-dashboard.md)접속 형태와 유사합니다.)

Cloud9의 상단 메뉴 Preview - Preview Running Application을 선택합니다. 메뉴에서 보이지 않는 경우 Tools - Preview - Preview Running Application을 선택합니다.

![](<../.gitbook/assets/image (397).png>)

생선된 Preview 브라우져에서 새로운 윈도우를 선택합니다.

![](<../.gitbook/assets/image (268).png>)

전체 화면 창에서 아래와 같이 마지막에 URL을 추가합니다.

```
:8081/targets
```

아래와 같은 결과를 브라우져에서 확인할 수 있습니다.Status 메뉴에서 다양한 결과를 확인 할 수 있습니다.

![](<../.gitbook/assets/image (362).png>)

## Grafana 구성

### 5.Grafana 설치&#x20;

Grafana를 사용하게 되면 시계열 메트릭 데이터를 질의, 시각화, Alert을 이해하는 데 사용할 수 있습니다.

#### Grafana Demo 공식 사이트 - [https://play.grafana.org/](https://play.grafana.org/)

먼저 Grafana설치를 위해서 몇가지 변수를 정의하고, 설치를 위한 매니페스트 파일을 작성합니다.

```
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

helm repo에 grafana를 등록 합니다.

```
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

```

이제 helm 을 통해 grafana를 설치합니다. 설치시에 옵션을 통해 prometheus 설치와 동일하게 storage type을 설정하고, 생성한 매니페스트를 불러옵니다.  또한 외부에서 접속을 위해서 Loadbalacer type을 지정합니다.

```
helm install grafana grafana/grafana \
    --namespace grafana \
    --set persistence.storageClassName="ebs-obs-sc" \
    --set persistence.enabled=true \
    --set adminPassword='1234Qwer' \
    --values ${HOME}/environment/grafana/grafana.yaml \
    --set service.type=LoadBalancer \
    --set service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"="external" \
    --set service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-scheme"="internet-facing"

```

아래와 같이 helm을 통해 설치된 결과를 확인 할 수 있습니다.&#x20;

```
NAME: grafana
LAST DEPLOYED: Sun Jun 19 03:59:13 2022
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

&#x20;다음 명령을 통해 , grafana 설치를 확인합니다.

```
kubectl -n grafana get all

```

출력결과 예시는 다음과 같습니다.

```
$ kubectl -n grafana get all
NAME                           READY   STATUS    RESTARTS   AGE
pod/grafana-69f4d986f6-j4z4l   1/1     Running   0          48s

NAME              TYPE           CLUSTER-IP      EXTERNAL-IP                                                                   PORT(S)        AGE
service/grafana   LoadBalancer   172.20.217.27   a66440ff27d1e43e8ba9e8a8f02a6893-192665941.ap-northeast-2.elb.amazonaws.com   80:32297/TCP   48s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grafana   1/1     1            1           48s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/grafana-69f4d986f6   1         1         1       48s
```

service를 확인합니다.

```
kubectl get svc -n grafana grafana

```

### 6.Grafana 접속 확인.&#x20;

출력 결과에서 제공되는 Loadbalancer 주소를 브라우져에서 입력합니다.

```
whchoi98:~/environment/grafana $ kubectl -n grafana get svc
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP                                                                    PORT(S)        AGE
grafana   LoadBalancer   172.20.23.245   a555fefad0ed8493fb4a9ec240318103-2014381236.ap-northeast-2.elb.amazonaws.com   80:31156/TCP   3h12m
```

아래 명령을 통해서 Grafana Service 주소를 확인합니다.&#x20;

```
kubectl -n grafana get svc grafana  | tail -n 1 | awk '{ print "grafana URL = http://"$4 }'

```



앞서 grafana.yaml에서 입력한 admin의 패스워드를 입력합니다.

```
--set adminPassword='1234Qwer'
```

![](<../.gitbook/assets/image (491).png>)

## DashBoard 구성

### 7.Cluster Monitoring

이제 생성된 Grafana 에서 배포된 Cluster들에 대해서 모니터링을 합니다.

우측 상단 메뉴의 "+" 를 선택하고 Import dashboard를 선택합니다.

<figure><img src="../.gitbook/assets/image (437).png" alt=""><figcaption></figcaption></figure>

Import 값을 "3119"를 선택하고, Load를 선택합니다.

![](<../.gitbook/assets/image (181).png>)

DataSource를 Prometheus를 선택합니다.

![](<../.gitbook/assets/image (495).png>)

아래와 같이 다양한 Cluster 내부의 정보를 확인 할 수 있습니다.

![](<../.gitbook/assets/image (190).png>)

### 8.Pod Monitoring 구성

이제 생성된 Grafana 에서 배포된 Pod들에 대해서 모니터링을 합니다.

좌측 상단 메뉴의 "+" 를 선택하고 Import를 선택합니다.

![](<../.gitbook/assets/image (482).png>)

Import 값을 "6417"를 선택하고, Load를 선택합니다.

![](<../.gitbook/assets/image (284).png>)

Name : Kubernetes Pods Monitoring , Change uid, 데이터 소스 : Prometheus 를 선택하고, Import를 선택합니다.

![](<../.gitbook/assets/image (375).png>)

Pod들 중심으로 모니터링을 할 수 있습니다.

![](<../.gitbook/assets/image (231).png>)

### 9. Grafana Labs Dashboard 활용하기

위에서 활용한 것 처럼, 이미 템플릿으로 생성한 ID를 가져와서 구성해 봅니다.

[Grafana Labs Dashboard](https://grafana.com/grafana/dashboards)에 접속합니다. ([https://grafana.com/grafana/dashboards](https://grafana.com/grafana/dashboards))

아래와 같이 Filter를 통해서 유용한 Dashboard를 가져옵니다. (Name : kubernetes, Data Source : Prometheus)

![](<../.gitbook/assets/image (227).png>)

* Kubernetes Deployment Statefulset Daemonset metrics

```
8588
```

![](<../.gitbook/assets/image (134).png>)

* Kubernetes Cluster

```
7249
```

![](<../.gitbook/assets/image (220).png>)

* 1 Kubernetes cluster overview

```
11802
```

![](<../.gitbook/assets/image (224).png>)

Kubernetes / Networking / Pod

```
12661
```

![](<../.gitbook/assets/image (486).png>)

kubernetes 한국어 대쉬보드

```
13770
```

![](<../.gitbook/assets/image (248).png>)
