# Loki

## Overview



Observability/Monitoring 분야에서 로그는 개발자와 시스템 관리자가 애플리케이션, 인프라 내부에서 발생하는 이벤트를 이해하는 데 매우 중요한 역할을 합니다.

Grafana Loki는 로그를 쉽고 비용 효율적으로 저장하고 쿼리할 수 있도록 설계된 로그 집계 시스템입니다.

Loki는 매우 대중적으로 사용되는 모니터링 및 Alert 용 도구인 Grafana에서 파생된 수평 확장가능하고 가용성이 매우 높은 멀티 태넌트 기반의 로그 집계 시스템입니다.

Prometheus와 동일하게 Loki는 작동이 간단하고 비용 효율적으로 설계 되었습니다. 다른 많은 로그 집계 시스템과는 달리 Loki는 로그 내용을 인덱싱하지 않고, 대신 로그 스트림에 대한 레이블 세트를 인덱싱하도록 설계 되었습니다.

## 주요 기능

다른 로그 시스템들과 비교해서 Loki는 다음과 같은 주요 장점을 제공합니다.

* 로그에 전체 텍스트 인덱싱이 없음 - Loki는 압축된 구조화 되어 있지 않은 로그를 저장하고 메타데이터만 인덱싱 하는 방식입니다. 이러한 방식으로 인해 Loki는 로그의 전체 내용을 인덱싱하는 방식에 비해서 운영이 간단하고 실행 비용이 저렴합니다.
* 레이블 기반의 인덱싱 및 그룹화 : Loki는 Prometheus 와 동일한 레이블을 사용해서 로그 스트림을 인덱싱하고 그룹화 합니다. 이를 통해서  Prometheus에서 이미 사용하고 있는 것과 동일한 레이블을 사용해서 메트릭과 로그간에 원할하게 전환이 가능합니다.
* Kubernetes Pod 로그에 이상적 구조 : Loki는 Pod 레이블과 같은 메타데이터를 자동으로 스크랩하고 인덱싱하기 때문에 Kubernetes 포드로그를 저장하는데 적합한 구조입니다.
* Grafana 기본 지원 : Loki는 Version 6.0 부터 Grafana에서 기본 지원을 제공합니다.

<figure><img src="../.gitbook/assets/image (195).png" alt=""><figcaption></figcaption></figure>

## Loki 기반 로깅 스택의 구성 요소

* Promtail - 로그 수집, Loki 전송을 담당하는 에이전트
* Loki - 로그를 저장하고 쿼리를 처리하는 메인 서버.
* Grafana - 로그 쿼리 및 표시를 위한 플랫폼

<figure><img src="../.gitbook/assets/image (129).png" alt=""><figcaption></figcaption></figure>

Loki는 Prometheus 와 유사하지만 로그에 대한 다차원 레이블 기반 접근방식을 사용해서 인덱싱하고 종속성이 없는 운영하기 쉬운 단일 바이너리 시스템을 제공합니다. Loki와 Prometheus 의 주요 차이점은 무엇에 집중하고 있는지에 대한 차이에 있습니다.

Loki는 로그에 집중하고,  Prometheus는 metric에 집중하고 있습니다. Loki는 Push를 통해 로그를 전달하는 반면, Prometheus는 Pull 메카니즘을 사용하고 있습니다.



1\. Promtail로 로그 가져오기

Promtail은 Loki를 위해 특별히 설계된 로그 수집기입니다. Prometheus와 동일한 서비스 검색 메커니즘을 사용하고 Loki로 수집하기 전에 로그에 레이블 지정, 변환 및 필터링을 위한 유사한 기능을 공유합니다.

Promtail은 Kubernetes 클러스터 내에서 사이드카 또는 DaemonSet으로 배포하거나 독립 실행형 에이전트로 실행하여 다른 소스에서 로그를 수집할 수 있습니다. 에이전트는 로그에서 타임스탬프 및 레이블과 같은 메타데이터를 추출하여 구조화된 형식으로 Loki에 보냅니다.

2\. Loki에 로그 저장

Loki는 다른 로그 집계 시스템과 다르게 로그 텍스트를 인덱싱하지 않습니다. 대신 로그 항목은 스트림으로 그룹화되고 라벨로 인덱싱됩니다. 이 접근 방식은 Loki가 수신한 후 밀리초 내에 로그 라인을 쿼리할 수 있으므로 비용을 줄이고 효율성을 높입니다.

Loki의 스토리지 아키텍처는 Prometheus의 시계열 데이터베이스(TSDB)와 동일한 개념을 기반으로 하며, 로그는 청크로 구성되고 Amazon S3 와 같은 객체 스토리지에 저장됩니다. 이 디자인은 수평적 확장성과 고가용성을 가능하게 합니다.

3\. LogQL을 사용하여 로그 탐색

Loki는 로그를 효율적으로 탐색할 수 있는 LogQL이라는 강력한 쿼리 언어를 도입했습니다. LogQL은 Prometheus의 PromQL과 유사하며 Grafana 내에서 직접 LogQL 쿼리를 실행하여 다른 데이터 소스와 함께 로그를 시각화하여 통합 관찰 환경을 생성할 수 있습니다. CLI를 선호하는 사람들을 위해 Loki는 터미널에서 LogQL 쿼리를 실행할 수 있는 LogCLI도 제공합니다.

LogQL은 필터링, 패턴 일치 및 집계를 지원하므로 대량의 로그를 쉽게 검색하고 귀중한 통찰력을 추출할 수 있습니다. 예를 들어 특정 로그 메시지의 발생 횟수를 계산하거나 로그의 평균 응답 시간을 계산하거나 특정 레이블을 기반으로 로그를 그룹화할 수 있습니다.

4\. 로그에 대한 경고

Loki를 사용하면 들어오는 로그 데이터를 지속적으로 평가하도록 경고 규칙을 설정할 수 있습니다. 규칙이 트리거되면 Loki는 결과 알림을 Prometheus Alertmanager로 보냅니다. Prometheus Alertmanager는 중복 제거, 그룹화 및 알림을 적절한 팀 구성원에게 전달합니다. Loki를 Alertmanager와 통합하면 기존 알림 인프라를 활용하고 적임자가 로그의 중요한 이벤트에 대한 알림을 적시에 받을 수 있습니다.



Grafana Loki는 단순성, 비용 효율성 및 Prometheus 및 Grafana와의 원활한 통합을 제공하는 강력한 로그 집계 시스템입니다. 레이블 기반 인덱싱에 집중하고 전체 텍스트 인덱싱을 피함으로써 Loki는 로그 저장 및 쿼리를 위한 운영하기 쉽고 확장 가능한 솔루션을 제공합니다. Kubernetes Pod 로그로 작업하든 다른 소스의 로그로 작업하든 Loki는 로그 데이터를 탐색, 시각화 및 경고하기 위한 강력한 도구 세트를 제공합니다.

요약하면 Grafana Loki는 설정 및 운영이 쉽고 확장성이 뛰어나며 비용 효율적인 로그 집계 시스템을 찾는 조직에 탁월한 선택입니다. Grafana의 기본 지원, Kubernetes와의 호환성, 강력한 LogQL 쿼리 언어는 모든 관찰 가능성 및 모니터링 스택에 가치 있는 추가 기능을 제공합니다.&#x20;

## 설치 및 구성

### &#x20;1. Granfan 설치

EKS Workshop에서 다루었던 Grafana 설치를 수행합니다. 이미 설치되었다면 생략합니다.

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

### 2.Loki-Stack 구성 및 설치

아래 명령을 실행해서 Loki-Stack을 구성합니다.

```
helm show values grafana/loki-stack > ~/environment/loki-stack-values.yaml

```

아래와 같이 구성 변경합니다.

```
test_pod:
  image: bats/bats:v1.8.2
  pullPolicy: IfNotPresent

loki:
  enabled: true
  isDefault: true
  url: http://{{(include "loki.serviceName" .)}}:{{ .Values.loki.service.port }}
  readinessProbe:
    httpGet:
      path: /ready
      port: http-metrics
    initialDelaySeconds: 45
  livenessProbe:
    httpGet:
      path: /ready
      port: http-metrics
    initialDelaySeconds: 45
  datasource:
    jsonData: {}
    uid: ""


promtail:
  enabled: true
  config:
    logLevel: info
    serverPort: 3101
    clients:
      - url: http://{{ .Release.Name }}:3100/loki/api/v1/push


```

Loki가 설치될 namespace를 생성하고, Loki를 설치합니다.

```
kubectl create namespace loki
helm install loki-stack grafana/loki-stack --values ~/environment/loki-stack-values.yaml -n loki
kubectl -n loki get pods

```

설치가 아래와 같이 정상적으로 완료되었는지 확인 합니다. (수분 정도가 소요됩니다.)

```
user01:~/environment $ kubectl -n loki get pods
NAME                        READY   STATUS    RESTARTS   AGE
loki-stack-0                1/1     Running   0          103s
loki-stack-promtail-4zvqw   1/1     Running   0          103s
loki-stack-promtail-5j2qb   1/1     Running   0          103s
loki-stack-promtail-5t8jc   1/1     Running   0          103s
loki-stack-promtail-92dfv   1/1     Running   0          103s
loki-stack-promtail-98pbn   1/1     Running   0          103s
loki-stack-promtail-bq95s   1/1     Running   0          103s
loki-stack-promtail-k4wrd   1/1     Running   0          103s
loki-stack-promtail-m282p   1/1     Running   0          103s
loki-stack-promtail-pjf7x   1/1     Running   0          103s
loki-stack-promtail-s6ml4   1/1     Running   0          103s
loki-stack-promtail-sdbz7   1/1     Running   0          103s
loki-stack-promtail-zvwtp   1/1     Running   0          103s
```

3.서비스 확인

Loki 가 구성된 Service를 확인합니다.

```
kubectl -n loki get svc

```

아래와 같은 결과를 확인할 수 있습니다.

```
$ kubectl -n loki get svc
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
loki-stack              ClusterIP   172.20.134.15   <none>        3100/TCP   11m
loki-stack-headless     ClusterIP   None            <none>        3100/TCP   11m
loki-stack-memberlist   ClusterIP   None            <none>        7946/TCP   11m
```

loki-stack의 ClusterIP가 Garafana의 DataSource에 해당됩니다.

앞의 랩을 그대로 수행하였다면 아래와 같은 주소가 될 것입니다. 값을 복사해 둡니다.

```
http://loki-stack.loki.svc.cluster.local:3100
```

3.DataSource 연결

앞서 설치한 Grafana에 접속합니다.

아래와 같이 DataSource 메뉴를 선택하고 "Add New DataSource" 를 선택합니다.

<figure><img src="../.gitbook/assets/image (165).png" alt=""><figcaption></figcaption></figure>

DataSource 중에 "Loki"를 선택합니다.

<figure><img src="../.gitbook/assets/image (159).png" alt=""><figcaption></figcaption></figure>

Settings 메뉴에서 HTTP URL에 앞서 복사한 주소를 입력합니다.

"Save\&Test" 를 선택하고 완료합니다.

<figure><img src="../.gitbook/assets/image (140).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (153).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (142).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (187).png" alt=""><figcaption></figcaption></figure>
