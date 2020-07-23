# Prometheus-Grapana

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

![&#xCC38;&#xC870; - https://prometheus.io/docs/introduction/overview/](../.gitbook/assets/image%20%2866%29.png)

## 2. Prometheus 구성



```text
helm search repo stable/prometheus
```



```text

```

## 3. Grafana 구성



## 4. DashBoard 구성





