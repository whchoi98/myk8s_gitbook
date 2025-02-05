# 컨테이너 소개

## 컨테이너 개요

컨테이너는 런타임에 필요한 종속성과 애플리케이션의 컴파일 된 코드를 패키징 하는 기술이다. 실행되는 각각의 컨테이너는 반복해서 사용 가능하다. 종속성이 포함된 표준화를 통해 컨테이너가 실행되는 환경과 무관하게 항상 동일하게 동작한다.

컨테이너는 기본 호스트 인프라 환경에서 애플리케이션의 실행환경을 분리한다. 따라서 다양한 클라우드 환경이나 운영체제에서 쉽게 배포 할 수 있다.

### 컨테이너 이미지 <a href="#undefined" id="undefined"></a>

[컨테이너 이미지](https://kubernetes.io/ko/docs/concepts/containers/images/) 는 즉시 실행할 수 있는 소프트웨어 패키지이며, 애플리케이션을 실행하는데 필요한 모든 것 (필요한 코드와 런타임, 애플리케이션 및 시스템 라이브러리 등의 모든 필수 설정에 대한 기본값) 을 포함한다.

원칙적으로, 컨테이너는 변경되지 않는다. 이미 구동 중인 컨테이너의 코드를 변경할 수 없다. 컨테이너화 된 애플리케이션이 있고 그 애플리케이션을 변경하려는 경우, 변경사항을 포함하여 만든 새로운 이미지를 통해 컨테이너를 다시 생성해야 한다.

### 컨테이너 런타임 <a href="#undefined" id="undefined"></a>

컨테이너 런타임은 컨테이너 실행을 담당하는 소프트웨어이다.

쿠버네티스는 여러 컨테이너 런타임을 지원한다. [도커(Docker)](https://docs.docker.com/engine/), [containerd](https://containerd.io/docs/), [CRI-O](https://cri-o.io/#what-is-cri-o) 그리고 [Kubernetes CRI (컨테이너 런타임 인터페이스)](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/container-runtime-interface.md)를 구현한 모든 소프트웨어.
