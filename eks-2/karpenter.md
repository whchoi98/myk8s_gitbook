---
description: 'Update: 2022-04-30'
---

# 스케쥴링-Karpenter

## Karpenter 소개&#x20;

![](<../.gitbook/assets/image (229).png>)

Karpenter는 AWS로 구축된 유연한 오픈 소스의 고성능 Kubernetes 클러스터 오토스케일러입니다. 애플리케이션 로드의 변화에 대응하여 적절한 크기의 컴퓨팅 리소스를 신속하게 실행함으로써 애플리케이션 가용성과 클러스터 효율성을 개선할 수 있습니다. 또한 Karpenter는 애플리케이션의 요구 사항을 충족하는 컴퓨팅 리소스를 적시에 제공하며, 앞으로 클러스터의 컴퓨팅 리소스 공간을 자동으로 최적화하여 비용을 절감하고 성능을 개선 할수 있습니다.&#x20;

Karpenter 이전에는 Kubernetes 사용자가 [Amazon EC2 Auto Scaling 그룹](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html)과 [Kubernetes Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)를 사용하는 애플리케이션을 지원하기 위해 클러스터의 컴퓨팅 파워를 동적으로 조정해야 했습니다. EKS를 사용하는 많은 고객들이 Kubernetes Cluster Autoscaler를 사용하여 클러스터 Auto Scaling을 구성하기가 어렵고 구성할 수 있는 범위가 제한적인 것에 대해 개선을 요구했습니다&#x20;

Karpenter가 클러스터에 설치되면 Karpenter는 예약되지 않은 포드의 전체 리소스 요청을 관찰하고 새 노드를 시작하고 종료하는 결정을 내림으로써 예약 대기 시간과 인프라 비용을 줄입니다. 이를 위해 Karpenter는 Kubernetes 클러스터 내의 이벤트를 관찰한 다음 Amazon EC2와 같은 기본 클라우드 공급자의 컴퓨팅 서비스로 명령을 전송합니다.

![](<../.gitbook/assets/image (236).png>)

Karpenter는 [Apache License 2.0](https://github.com/awslabs/karpenter/blob/main/LICENSE)을 통해 라이선스가 부여되는 오픈 소스 프로젝트입니다. 모든 주요 클라우드 공급업체 및 온프레미스 환경을 포함하여, 모든 환경에서 실행되는 모든 Kubernetes 클러스터와 함께 작동하도록 설계되었습니다.&#x20;



## Karpenter 설치

### 1.환경 설정



### 2.Karpenter 설치



### 3.Provisioner 구성

