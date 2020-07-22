---
description: 'update : 2020-07-20'
---

# K8s Dashboard 배포

## K8s Dashboard 소개

대시보드는 웹 기반 쿠버네티스 유저 인터페이스입니다. 대시보드를 통해 컨테이너화 된 애플리케이션을 쿠버네티스 클러스터에 배포할 수 있고, 컨테이너화 된 애플리케이션을 트러블슈팅 할 수 있으며, 클러스터 리소스들을 관리할 수 있습니. 대시보드를 통해 클러스터에서 동작중인 애플리케이션의 정보를 볼 수 있고, 개별적인 쿠버네티스 리소스들을\(예를 들면 디플로이먼트, 잡, 데몬셋 등\) 생성하거나 수정할 수 있습니다. 예를 들면, 디플로이먼트를 스케일하거나, 롤링 업데이트를 초기화하거나, 파드를 재시작하거나 또는 배포 마법사를 이용해 새로운 애플리케이션을 배포할 수 있습니다.

또한 대시보드는 클러스터 내 쿠버네티스 리소스들의 상태와 발생하는 모든 에러 정보를 제공합니다.

## K8s Dashboard 구성

### 1.Kubernetes 지표 서버 구성

kubernetes 지표 서버를 다음 명령을 사용하여 배포합니다.

```text
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml
```

다음 명령을 사용하여 metrics-server 배포에서 원하는 수의 포드를 실행하고 있는지 확인합니다.

```text
kubectl get deployment metrics-server -n kube-system
```

출력 결과 예시

```text
whchoi98:~/environment $ kubectl get deployment metrics-server -n kube-system
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
metrics-server   1/1     1            1           57s
```

### 2.Kubernetes 대시보드 배포

다음 명령을 사용하여 Kubernetes 대시보드를 배포합니다.

```text
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.3/aio/deploy/recommended.yaml
```

배포된 서비스들을 확인합니다.

```text
kubectl -n kubernetes-dashboard get svc -o wide
```

```text
whchoi98:~/environment $ kubectl -n kubernetes-dashboard get svc -o wide                                                    
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE   SELECTOR
dashboard-metrics-scraper   ClusterIP   172.20.230.33    <none>        8000/TCP   28h   k8s-app=dashboard-metrics-scraper
kubernetes-dashboard        ClusterIP   172.20.122.109   <none>        443/TCP    28h   k8s-app=kubernetes-dashboard
```

### 3.권한 설정

기본적으로 Kubernetes 대시보드 사용자의 권한은 제한되어 있습니다. eks-admin 서비스 계정 및 클러스터 역할 바인딩을 생성하고, 이를 사용하여 관리자 수준 권한으로 대시보드에 보안 연결할 수 있습니다. 

아래 텍스트가 포함된 **"eks-admin-service-account.yaml"** 이라는 파일을 생성합니다. 이 매니페스트는 eks-admin이라는 서비스 계정 및 클러스터 역할 바인딩을 정의합니다.

```text
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eks-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: eks-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: eks-admin
  namespace: kube-system
```

서비스 계정 및 클러스터 역할 바인딩을 클러스터에 적용합니다.

```text
kubectl apply -f eks-admin-service-account.yaml
```

### 4. Kubernetes 대쉬보드 접속

클러스터에 Kubernetes 대시보드가 배포되었고, 클러스터를 보고 제어하는 데 사용할 수 있는 관리자 서비스 계정을 가졌기 때문에 이제 이 서비스 계정을 사용하여 대시보드에 연결할 수 있습니다.

Kubernetes 대시보드에 연결하기 위해서, eks-admin 서비스 계정에 대한 인증 토큰을 조회합니다. 출력에서  값을 복사합니다. 이 토큰을 사용하여 대시보드에 연결합니다.

```text
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
```

출력 결과 예시

```text
Name:         eks-admin-token-s4rbb
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: eks-admin
              kubernetes.io/service-account.uid: cd162bc8-004f-401f-97d3-8ff6fc118257

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token: <authentication_token>
```

Kubernetes 대쉬보드 접속을 위해 kubectl proxy를 시작합니다.

```text
kubectl proxy --port=8080 --address=0.0.0.0 --disable-filter=true &
```

{% hint style="success" %}
kubernetes dashboard는 ClusterIP 타입으로 서비스 배포됩니다. ClusterIP구성으로 외부에 노출되지 않습니다. 따라서 Kubectl proxy 설정이 필요합니다.
{% endhint %}

아래 그림은 Kubernetes 대쉬보드의 접속 방식에 대한 도식입니다.

![](../.gitbook/assets/image%20%2828%29.png)

외부에서 접속되지 않으므로, Cloud9 IDE에서 Preview를 통해 Proxy로 접속합니다.

Cloud9의 상단 메뉴 Preview - Preview Running Application을 선택합니다. 메뉴에서 보이지 않는 경우 Tools - Preview - Preview Running Application을 선택합니다.

![](../.gitbook/assets/image%20%2816%29.png)

생선된 Preview 브라우져에서 새로운 윈도우를 선택합니다.

![](../.gitbook/assets/image%20%282%29.png)

생성된 윈도우에서 아래 Path를 복사해서 뒤에 붙여 넣습니다.

```text
/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

앞서 생성한 eks-admin 토큰 값을 Kubernettes Dashboard에 입력합니다.

![](../.gitbook/assets/image%20%2827%29.png)

정상적으로 생성된 것을 확인합니다.

![](../.gitbook/assets/image%20%2820%29.png)

### 5.Kubernetes 대쉬보드 삭제

Proxy를 먼저 삭제합니다.jobs의 결과에서 Proxy 번호를 확인하고 kill을 통해서 삭제합니다.

```text
jobs
```

1번에 background process가 동작 중이라면 , 다음과 같이 삭제합니다.

```text
kill %1
```

설치된 Dashboard를 삭제합니다.

```text
kubectl delete -f https://raw.githubusercontent.com/kubernetes/dashboard/${DASHBOARD_VERSION}/aio/deploy/recommended.yaml
```

저장된 변수를 삭제합니다.

```text
unset DASHBOARD_VERSION
```

