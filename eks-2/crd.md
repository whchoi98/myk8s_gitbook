---
description: CRD
---

# CRD

## 소개

Custom Resource 란?

_커스텀 리소스_ 는 쿠버네티스 API의 익스텐션입니다. _리소스_ 는 [쿠버네티스 API](https://kubernetes.io/ko/docs/reference/using-api/api-overview/)에서 특정 종류의 [API 오브젝트](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/kubernetes-objects/) 모음을 저장하는 엔드포인트입니. 예를 들어 _파드_ 리소스에는 파드 오브젝트 모음이 포함되어 있습니다.

_커스텀 리소스_ 는 쿠버네티스 API의 익스텐션으로, 기본 쿠버네티스 설치에서 반드시 사용할 수 있는 것은 아니다. 이는 특정 쿠버네티스 설치에 수정이 가해졌음을 나타냅니. 그러나 많은 핵심 쿠버네티스 기능은 이제 커스텀 리소스를 사용하여 구축되어, 쿠버네티스를 더욱 모듈화합니다.

동적 등록을 통해 실행 중인 클러스터에서 커스텀 리소스가 나타나거나 사라질 수 있으며 클러스터 관리자는 클러스터 자체와 독립적으로 커스텀 리소스를 업데이트 할 수 있습니. 커스텀 리소스가 설치되면 사용자는 _파드_ 와 같은 빌트인 리소스와 마찬가지로 [kubectl](https://kubernetes.io/ko/docs/reference/kubectl/overview/)을 사용하여 해당 오브젝트를 생성하고 접근할 수 있습니다.





