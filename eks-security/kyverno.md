---
description: 'Update : 2023-10-10'
---

# Kyverno

## Kyverno 소개



쿠버네티스 클러스터에서 수행되는 작업의 범위 또는 허용 여부를 결정하기 위해 정의된 규칙을 Policy라고 합니다. Kyverno는 정책을 관리하고 쿠버네티스 클러스터에 적용하는 도구 입니다. Kyverno를 통해 이미지의 특정 Tag 제한, Host의 포트 사용 제한, 사용할 수 있는 이미지 레지스트리 제한을 설정할수 있습니다. 또한 적용할 수 있는 정책들은 매우 다양합니다.

OPA 과 같은 정책도구들이 있지만, 정책적용을 위해 별도의 언어를 학습해야하는 러닝커브가 존재합니다.

Kyverno는 쿠버네티스 YAML을 그대로 사용하기 때문에 사용자 친화적으로 사용이 가능하며, 가장 널리 사용하는 방법 중 하나입니다.



Kyverno(그리스어 "통치")는 Kubernetes를 위해 특별히 설계된 정책 엔진입니다.

* Kubernetes 리소스로서의 정책(새로운 언어를 배울 필요가 없습니다!)
* 모든 리소스의 유효성 검사, 변형, 생성 또는 정리(제거)
* 소프트웨어 공급망 보안을 위한 컨테이너 이미지 확인
* 이미지 메타데이터 검사
* Label Selector와 와일드카드를 사용하여 리소스 일치
* 오버레이를 사용하여 유효성을 검사하고 변경합니다(예: Kustomize!)
* 네임스페이스 전체에서 구성 동기화
* 를  a사m용. 하 여 비준수 리소스를 차단하거나 정책 위반을 보고합니다.
* 셀프 서비스 보고서(독점 감사 로그 없음!)
* 셀프 서비스 정책 예외
* 클러스터에 적용하기 전에 CI/CD 파이프라인에서 Kyverno CLI를 사용하여 정책을 테스트하고 리소스를 검증합니다.
* `git`및 같은 친숙한 도구를 사용하여 코드로 정책을 관리합니다.



<figure><img src="../.gitbook/assets/image (500).png" alt=""><figcaption></figcaption></figure>

## Kyverno 설치

Helm을 기반으로 Kyverno를 설치합니다.

```
# kyverno 레포지토리 추가
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update

```

Kyverno를 설치합니다.

```
helm install kyverno kyverno/kyverno -n kyverno --create-namespace --set replicaCount=3

```

