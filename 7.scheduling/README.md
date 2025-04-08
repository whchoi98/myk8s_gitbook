---
description: 'update : 2025.04.08'
---

# 7.Scheduling

<figure><img src="../.gitbook/assets/image (517).png" alt=""><figcaption></figcaption></figure>

🚀 Amazon EKS 환경에서의 워크로드 확장 방식\



Amazon EKS에서는 워크로드 확장을 어플리케이션 레벨과 데이터 플레인(인프라) 레벨로 구분하여 유연하게 관리할 수 있습니다.

***

### 1. 어플리케이션 확장 기법

<table data-header-hidden><thead><tr><th></th><th width="328.453125"></th><th width="106.6328125"></th><th></th></tr></thead><tbody><tr><td>기법</td><td>설명</td><td>확장 대상</td><td>트리거</td></tr><tr><td>HPA <br>(Horizontal Pod Autoscaler)</td><td>파드 수를 수평적으로 증가시켜 트래픽에 대응</td><td>파드</td><td>CPU, 메모리 등 메트릭</td></tr><tr><td>VPA <br>(Vertical Pod Autoscaler)</td><td>파드 자체의 자원(CPU, 메모리)을 증가시켜 성능 향상</td><td>파드</td><td>리소스 사용률</td></tr><tr><td>KEDA <br>(Event-driven Autoscaler)</td><td>이벤트 기반으로 파드 수를 확장. 외부 시스템과 연동 가능</td><td>파드</td><td>메시지 큐, <br>스케줄러 등 이벤트</td></tr></tbody></table>

> 📌 HPA와 VPA는 metrics-server 또는 custom metrics API가 필요하며, KEDA는 이벤트 소스 연동이 필요합니다.

***

### 2. 데이터 플레인 확장 기법

| 기법                       | 설명                                                  | 확장 대상  | 트리거               |
| ------------------------ | --------------------------------------------------- | ------ | ----------------- |
| Cluster Autoscaler (CAS) | Auto Scaling Group 기반 노드 확장. Pending Pod을 감지해 노드 증가 | EC2 노드 | Pending Pod       |
| Karpenter                | 필요 시점에 적합한 EC2 인스턴스를 직접 생성. 빠르고 유연한 확장              | EC2 노드 | Unschedulable Pod |

> ⚡ Karpenter는 ASG에 의존하지 않고, 원하는 인스턴스 타입을 선택하여 즉시 생성하므로 비용 최적화와 확장 속도에 유리합니다.

***

### 3. 전체 아키텍처 흐름

* HPA/VPA/KEDA → 파드 확장 요청 발생
* CAS/Karpenter → 파드가 스케줄링되지 못할 경우 노드 확장 수행
* 각각의 Autoscaler가 서로 보완적으로 작동하며, 효율적인 워크로드 운영 가능

***

### 4. 선택 가이드

| 상황                      | 추천 기법              |
| ----------------------- | ------------------ |
| 트래픽에 따라 파드 수 조절         | HPA                |
| 머신러닝/배치 작업 등 리소스 집약형 파드 | VPA                |
| 큐 기반 처리, 이벤트 트리거        | KEDA               |
| 안정적이고 관리형 ASG 기반 확장     | Cluster Autoscaler |
| 빠르고 비용 최적화된 노드 확장       | Karpenter          |



