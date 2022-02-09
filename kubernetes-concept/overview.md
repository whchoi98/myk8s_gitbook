---
description: '참조 원문 : https://kubernetes.io/'
---

# Overview

### 개념

이 단원에서는 쿠버네티스 시스템을 구성하는 요소와 [클러스터](https://kubernetes.io/ko/docs/reference/glossary/?all=true#term-cluster)를 표현하는데 사용되는 추상적 개념에 대해 배우고 쿠버네티스가 작동하는 방식에 대해 보다 깊이 이해할 수 있습니다.

### 개요 <a href="#undefined" id="undefined"></a>

쿠버네티스를 사용하려면, 쿠버네티스 API 오브젝트(객체)로 클러스터에 대해 사용자가 원하는 상태를 기술해야 합니다. 원하는 상태의 의미는 어떤 애플리케이션이나 워크로드를 구동시키려고 하는지, 어떤 컨테이너 이미지를 쓰는지, 복제를 원하는 수는 몇 개인지, 어떤 네트워크와 디스크 자원을 쓸 수 있도록 할 것인지 등을 의미합니다.

사용자가 원하는 상태를 설명하는 방법은 쿠버네티스 API를 사용해서 오브젝트(객체)를 만드는 것인데, 일반적으로 `kubectl`이라는 커맨드라인 인터페이스를 사용합니다.  또한 클러스터와 상호 작용하고 원하는 상태를 설정하거나 수정하기 위해서 쿠버네티스 API를 직접 사용할 수도 있습니다.

원하는 상태를 설정하면, 쿠버네티스 컨트롤 플레인(제어부)은 Pod Lifecycle Event Generator ([PLEG](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/pod-lifecycle-event-generator.md))를 통해 클러스터의 현재 상태를 원하는 상태와 일치시킵니다. 그렇게 함으로써, 쿠버네티스가 컨테이너를 시작, 재시작하거나, 주어진 애플리케이션의 복제 수를 확장 하는 등의 다양한 작업을 자동으로 수행합니다. 쿠버네티스 컨트롤 플레인은 클러스터에서 실행 중인 프로세스의 집합(collection)으로 구성됩니다.&#x20;

* **쿠버네티스 마스터**는 클러스터 내 마스터 노드로 지정된 노드 내에서 구동되는 세 개의 프로세스 집합입니다. \
  해당 프로세스는 [kube-apiserver](https://kubernetes.io/docs/admin/kube-apiserver/), [kube-controller-manager](https://kubernetes.io/docs/admin/kube-controller-manager/) 및 [kube-scheduler](https://kubernetes.io/docs/admin/kube-scheduler/) 입니다.
* 클러스터 내 마스터 노드가 아닌 각각의 노드는 다음 두 개의 프로세스를 구동시킵니다.( 일반적으로 워커 노드라고 이야기 합니다.)
  * [**kubelet**](https://kubernetes.io/docs/admin/kubelet/) **-** 쿠버네티스 마스터와 통신하는 프로세스
  * [**kube-proxy**](https://kubernetes.io/docs/admin/kube-proxy/) **-** 각 노드의 쿠버네티스 네트워킹 서비스를 반영하는 네트워크 프록시

### 쿠버네티스 오브젝트 <a href="#undefined" id="undefined"></a>

쿠버네티스는 시스템의 상태를 나타내는 추상 개념을 다수 포함하고 있습니다.  컨테이너화되어 배포된 애플리케이션과 워크로드, 이에 관련된 네트워크와 디스크 자원, 그 밖에 클러스터가 무엇을 하고 있는지에 대한 정보가 이에 해당됩니다. 이런 추상적 개념은 쿠버네티스 API 내에 오브젝트로 표현됩니다. 보다 자세한 내용은 [쿠버네티스 오브젝트 이해하기](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/kubernetes-objects/#kubernetes-objects) 문서를 참조하시기 바랍니다.

기초적인 쿠버네티스 오브젝트에는 다음과 같은 것들이 있다.

* [파드](https://kubernetes.io/ko/docs/concepts/workloads/pods/pod-overview/)
* [서비스](https://kubernetes.io/ko/docs/concepts/services-networking/service/)
* [볼륨](https://kubernetes.io/ko/docs/concepts/storage/volumes/)
* [네임스페이스(Namespace)](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/namespaces/)

또한, 쿠버네티스에는 기본 오브젝트를 기반으로, 부가 기능 및 편의 기능을 제공하는 [컨트롤러](https://kubernetes.io/ko/docs/concepts/architecture/controller/)에 의존하는 보다 높은 수준의 추상적 개념도 포함되어 있습니다. 아래와 같은 것들이 해당됩니다.

* [디플로이먼트(Deployment)](https://kubernetes.io/ko/docs/concepts/workloads/controllers/deployment/)
* [데몬셋(DaemonSet)](https://kubernetes.io/ko/docs/concepts/workloads/controllers/daemonset/)
* [스테이트풀셋(StatefulSet)](https://kubernetes.io/ko/docs/concepts/workloads/controllers/statefulset/)
* [리플리카셋(ReplicaSet)](https://kubernetes.io/ko/docs/concepts/workloads/controllers/replicaset/)
* [잡(Job)](https://kubernetes.io/ko/docs/concepts/workloads/controllers/jobs-run-to-completion/)

### 쿠버네티스 컨트롤 플레인 <a href="#undefined" id="undefined"></a>

쿠버네티스 마스터, kubelet 프로세스와 같은 쿠버네티스 컨트롤 플레인의 다양한 구성 요소는 쿠버네티스가 클러스터와 통신하는 방식을 담당합니다.  컨트롤 플레인은 시스템 내 모든 쿠버네티스 오브젝트의 레코드를 유지하면서, 오브젝트의 상태를 관리하는 제어 루프를 지속적으로 구동시킵니다. 컨트롤 플레인의 제어 루프는 클러스터 내 변경이 발생하면 언제라도 응답하고 시스템 내 모든 오브젝트의 실제 상태가 사용자가 원하는 상태와 일치시키기 위한 일을 담당합니다.

예를 들어, 쿠버네티스 API를 사용해서 디플로이먼트를 만들 때에는, 원하 상태를 시스템에 신규로 입력해야 합니다. 이를 통해 쿠버네티스 컨트롤 플레인이 오브젝트 생성을 기록하고, 사용자 명령로 필요한 애플리케이션을 시작시키고 클러스터 노드에 스케줄링합니다.  이 과정을 통해서 클러스터의 실제 상태가 원하는 상태와 일치하게 됩니다.

### 쿠버네티스 마스터

클러스터에 대해 원하는 상태를 유지는 역할을 담당합니다. `kubectl` 커맨드라인 인터페이스 를  통해서  쿠버네티스  마스터와   워커노드는 상호간에  통신하게 됩니다.

{% hint style="warning" %}
"마스터"는 클러스터 상태를 관리하는 프로세스의 집합입니다 . 주로 모든 프로세스는 클러스터 내 단일 노드에서 구동되며, 이 노드가 바로 마스터입니다. 이러한 마스터는 가용성과 중복을 위해 복제될 수 있습니다.&#x20;
{% endhint %}

### 쿠버네티스 노드 (워커노드)

클러스터 내 노드는 애플리케이션과 클라우드 워크플로우를 구동시키는 머신(VM, 물리 서버 등)입니다.  앞서 소개한 쿠버네티스 마스터는 각각의 노드를 관리합니다.&#x20;
