---
description: 'Update : 2023-01-27'
---

# Argo 기반 CD (TBD)

애플리케이션 정의, 구성 및 환경은 선언적이고 버전이 제어되어야 합니다. 애플리케이션 배포 및 수명 주기 관리는 자동화되고 감사 가능하며 이해하기 쉬워야 합니다.Argo CD 는 원하는 애플리케이션 상태를 정의하기 위한 정보 소스로 Git 리포지토리를 사용 하는 **GitOps 패턴을 따릅니다.** Kubernetes 매니페스트는 여러 가지 방법으로 지정할 수 있습니다.

* [kustomize](https://kustomize.io/) applications
* [helm](https://helm.sh/) charts
* [jsonnet](https://jsonnet.org/) files
* Plain directory of YAML/json manifests
* 구성 관리 플러그인으로 구성된 모든 맞춤형 구성 관리 도구

Argo CD는 지정된 대상 환경에서 원하는 애플리케이션 상태의 배포를 자동화합니다. 애플리케이션 배포는 분기, 태그에 대한 업데이트를 추적하거나 Git 커밋에서 매니페스트의 특정 버전에 고정할 수 있습니다.



Argo CD는 아래와 같은 아키텍쳐로 구성됩니다.

<figure><img src="../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

Argo CD는 실행 중인 애플리케이션을 지속적으로 모니터링하고 현재 원하는 대상 상태(Git 저장소에 지정된 대로)와 비교하는 kubernetes 컨트롤러와 같은 형태로 구현됩니다. Argo CD는 라이브 상태를 원하는 대상 상태로 다시 자동 또는 수동으로 동기화하는 기능을 제공하면서 , 현재 상태와 원하는 상태의 차이점을 리포팅하고 시각화합니다. Git 리포지토리에서 원하는 대상 상태에 대한 모든 수정 사항은 지정된 대상 환경에 자동으로 적용되고 반영될 수 있습니다.



Argo CD는 **3가지 컨포넌트**로 구성되어 있습니다.

* 실행 중인 애플리케이션을 **지속적으로 모니터링**
* **현재 라이브 상태를 원하는 대상 상태(Git 저장소에 지정된 대로)와 주기적으로 비교**
  * 라이브 상태가 대상 상태와 다른 배포된 애플리케이션은 **OutOfSync**로 처리
* 차이점을 리포팅 및 **UI 기반으로 시각화**
* 라이브 상태를 원하는 대상 상태로 자동 또는 수동으로 다시 동기화 하는 기능을 제공
* **API Server**
  * API 서버는 Web UI, CLI 및 외부에서 사용할 수 있도록 gRPC와 REST API를 제공
* Repository Server
  * Application manifest 파일을 가지고 있는 Git 저장소의 로컬 캐시를 유지 관리
  * Git 저장소에 저장된 manifest 파일을 생성하는 역할
* **Application Controller**
  * 애플리케이션 컨트롤러는 실행중인 응용 프로그램을 지속적으로 모니터링
  * live state와 target state를 비교해서 UI로 리포팅 (OutOfSync)



