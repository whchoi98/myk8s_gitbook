# 클라우드 컨트롤러 매니저

## 클라우드 컨트롤러 매니저

**FEATURE STATE:** `Kubernetes v1.11 [beta]`

클라우드 인프라스트럭쳐 기술을 통해 퍼블릭, 프라이빗 그리고 하이브리드 클라우드에서 쿠버네티스를 실행할 수 있다. 쿠버네티스는 컴포넌트간의 긴밀한 결합 없이 자동화된 API 기반의 인프라스트럭쳐를 신뢰한다.

클라우드 컨트롤러 매니저는 클라우드별 컨트롤 로직을 포함하는 쿠버네티스 [컨트롤 플레인](https://kubernetes.io/ko/docs/reference/glossary/?all=true#term-control-plane) 컴포넌트이다. 클라우트 컨트롤러 매니저를 통해 클러스터를 클라우드 공급자의 API에 연결하고, 해당 클라우드 플랫폼과 상호 작용하는 컴포넌트와 클러스터와 상호 작용하는 컴포넌트를 분리할 수 있다.

쿠버네티스와 기본 클라우드 인프라스터럭처 간의 상호 운용성 로직을 분리함으로써, cloud-controller-manager 컴포넌트는 클라우드 공급자가 주요 쿠버네티스 프로젝트와 다른 속도로 기능들을 릴리스할 수 있도록 한다.

클라우드 컨트롤러 매니저는 다양한 클라우드 공급자가 자신의 플랫폼에 쿠버네티스를 통합할 수 있도록 하는 플러그인 메커니즘을 사용해서 구성된다.

## 디자인

![&#xCFE0;&#xBC84;&#xB124;&#xD2F0;&#xC2A4; &#xCEF4;&#xD3EC;&#xB10C;&#xD2B8;](https://d33wubrfki0l68.cloudfront.net/7016517375d10c702489167e704dcb99e570df85/7bb53/images/docs/components-of-kubernetes.png)

클라우드 컨트롤러 매니저는 컨트롤 플레인에서 복제된 프로세스의 집합으로 실행된다\(일반적으로, 파드의 컨테이너\). 각 클라우드 컨트롤러 매니저는 단일 프로세스에 여러 [컨트롤러](https://kubernetes.io/ko/docs/concepts/architecture/controller/)를 구현한다.

> **참고:** 또한 사용자는 클라우드 컨트롤러 매니저를 컨트롤 플레인의 일부가 아닌 쿠버네티스 [애드온](https://kubernetes.io/docs/concepts/cluster-administration/addons/)으로 실행할 수도 있다.

## 클라우드 컨트롤러 매니저의 기능

클라우드 컨틀롤러 매니저의 내부 컨트롤러에는 다음 컨트롤러들이 포함된다.

### 노드 컨트롤러

노드 컨트롤러는 클라우드 인프라스트럭처에 새 서버가 생성될 때 [노드](https://kubernetes.io/ko/docs/concepts/architecture/nodes/) 오브젝트를 생성하는 역할을 한다. 노드 컨트롤러는 클라우드 공급자의 사용자 테넌시 내에서 실행되는 호스트에 대한 정보를 가져온다. 노드 컨트롤러는 다음 기능들을 수행한다.

1. 컨트롤러가 클라우드 공급자 API를 통해 찾아내는 각 서버에 대해 노드 오브젝트를 초기화한다.
2. 클라우드 관련 정보\(예를 들어, 노드가 배포되는 지역과 사용 가능한 리소스\(CPU, 메모리 등\)\)를 사용해서 노드 오브젝트에 어노테이션과 레이블을 작성한다.
3. 노드의 호스트 이름과 네트워크 주소를 가져온다.
4. 노드의 상태를 확인한다. 노드가 응답하지 않는 경우, 이 컨트롤러는 사용자가 이용하는 클라우드 공급자의 API를 통해 서버가 비활성화됨 / 삭제됨 / 종료됨인지 확인한다. 노드가 클라우드에서 삭제된 경우, 컨트롤러는 사용자의 쿠버네티스 클러스터에서 노드 오브젝트를 삭제한다.

일부 클라우드 공급자의 구현에서는 이를 노드 컨트롤러와 별도의 노드 라이프사이클 컨트롤러로 분리한다.

### 라우트 컨트롤러

라우트 컨트롤러는 사용자의 쿠버네티스 클러스터의 다른 노드에 있는 각각의 컨테이너가 서로 통신할 수 있도록 클라우드에서 라우트를 적절히 구성해야 한다.

클라우드 공급자에 따라 라우트 컨트롤러는 파드 네트워크 IP 주소 블록을 할당할 수도 있다.

### 서비스 컨트롤러

[서비스](https://kubernetes.io/docs/concepts/services-networking/service/) 는 관리형 로드 밸런서, IP 주소, 네트워크 패킷 필터링 그리고 대상 상태 확인과 같은 클라우드 인프라스트럭처 컴포넌트와 통합된다. 서비스 컨트롤러는 사용자의 클라우드 공급자 API와 상호 작용해서 필요한 서비스 리소스를 선언할 때 로드 밸런서와 기타 인프라스트럭처 컴포넌트를 설정한다.

## 인가

이 섹션에서는 클라우드 컨트롤러 매니저가 작업을 수행하기 위해 다양한 API 오브젝트에 필요한 접근 권한을 세분화한다.

### 노드 컨트롤러

노드 컨트롤러는 노드 오브젝트에서만 작동한다. 노드 오브젝트를 읽고, 수정하려면 전체 접근 권한이 필요하다.

`v1/Node`:

* Get
* List
* Create
* Update
* Patch
* Watch
* Delete

### 라우트 컨트롤러

라우트 컨트롤러가 노드 오브젝트의 생성을 수신하고 적절하게 라우트를 구성한다. 노드 오브젝트에 대한 접근 권한이 필요하다.

`v1/Node`:

* Get

### 서비스 컨트롤러

서비스 컨트롤러는 서비스 오브젝트 생성, 업데이트 그리고 삭제 이벤트를 수신한 다음 해당 서비스에 대한 엔드포인트를 적절하게 구성한다.

서비스에 접근하려면, 목록과 감시 접근 권한이 필요하다. 서비스를 업데이트하려면, 패치와 업데이트 접근 권한이 필요하다.

서비스에 대한 엔드포인트 리소스를 설정하려면 생성, 목록, 가져오기, 감시 그리고 업데이트에 대한 접근 권한이 필요하다.

`v1/Service`:

* List
* Get
* Watch
* Patch
* Update

### 그 외의 것들

클라우드 컨트롤러 매니저의 핵심 구현을 위해 이벤트 오브젝트를 생성하고, 안전한 작동을 보장하기 위해 서비스어카운트\(ServiceAccounts\)를 생성해야 한다.

`v1/Event`:

* Create
* Patch
* Update

`v1/ServiceAccount`:

* Create

클라우드 컨트롤러 매니저의 [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) 클러스터롤\(ClusterRole\)은 다음과 같다.

```text
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cloud-controller-manager
rules:
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch
  - update
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - '*'
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - list
  - patch
  - update
  - watch
- apiGroups:
  - ""
  resources:
  - serviceaccounts
  verbs:
  - create
- apiGroups:
  - ""
  resources:
  - persistentvolumes
  verbs:
  - get
  - list
  - update
  - watch
- apiGroups:
  - ""
  resources:
  - endpoints
  verbs:
  - create
  - get
  - list
  - watch
  - update
```

###  <a id="&#xB2E4;&#xC74C;-&#xB0B4;&#xC6A9;"></a>

