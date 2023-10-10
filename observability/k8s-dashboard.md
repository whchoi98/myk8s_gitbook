---
description: 'update : 2020-11-15 / 20min'
---

# K8s Dashboard 배포

## K8s Dashboard 소개

대시보드는 웹 기반 쿠버네티스 유저 인터페이스입니다. 대시보드를 통해 컨테이너화 된 애플리케이션을 쿠버네티스 클러스터에 배포할 수 있고, 컨테이너화 된 애플리케이션을 트러블슈팅 할 수 있으며, 클러스터 리소스들을 관리할 수 있습니. 대시보드를 통해 클러스터에서 동작중인 애플리케이션의 정보를 볼 수 있고, 개별적인 쿠버네티스 리소스들을(예를 들면 디플로이먼트, 잡, 데몬셋 등) 생성하거나 수정할 수 있습니다. 예를 들면, 디플로이먼트를 스케일하거나, 롤링 업데이트를 초기화하거나, 파드를 재시작하거나 또는 배포 마법사를 이용해 새로운 애플리케이션을 배포할 수 있습니다.

또한 대시보드는 클러스터 내 쿠버네티스 리소스들의 상태와 발생하는 모든 에러 정보를 제공합니다.

## K8s Dashboard 구성

### 1.Kubernetes 지표 서버 구성

kubernetes 지표 서버를 다음 명령을 사용하여 배포합니다.

```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

```

또는 git을 통해 복제 해 둔 yaml 을 실행합니다.

```
cd ~/environment/myeks/k8s_dashboard
kubectl apply -f components.yaml

```

다음 명령을 사용하여 metrics-server 배포에서 원하는 수의 포드를 실행하고 있는지 확인합니다.

```
kubectl get deployment metrics-server -n kube-system

```

출력 결과 예시

```
~/environment $ kubectl get deployment metrics-server -n kube-system
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
metrics-server   1/1     1            1           57s
```

### 2.Kubernetes 대시보드 배포

다음 명령을 사용하여 Kubernetes 대시보드를 배포합니다.

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.4.0/aio/deploy/recommended.yaml

```

또는 git을 통해 복제 해 둔 yaml 을 실행합니다.

```
cd ~/environment/myeks/k8s_dashboard
kubectl apply -f recommended.yaml

```

이제 배포된 dashboard와 metric을 확인합니다.&#x20;

```
kubectl -n kubernetes-dashboard get svc -o wide

```

```
whchoi98:~/environment $ kubectl -n kubernetes-dashboard get svc -o wide                                                    
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE   SELECTOR
dashboard-metrics-scraper   ClusterIP   172.20.230.33    <none>        8000/TCP   28h   k8s-app=dashboard-metrics-scraper
kubernetes-dashboard        ClusterIP   172.20.122.109   <none>        443/TCP    28h   k8s-app=kubernetes-dashboard
```

### 3.권한 설정

기본적으로 Kubernetes 대시보드 사용자의 권한은 제한되어 있습니다. eks-admin 서비스 계정 및 클러스터 역할 바인딩을 생성하고, 이를 사용하여 관리자 수준 권한으로 대시보드에 보안 연결할 수 있습니다.&#x20;

아래 텍스트가 포함된 **"eks-admin-service-account.yaml"** 이라는 파일을 생성합니다. 이 매니페스트는 eks-admin이라는 서비스 계정 및 클러스터 역할 바인딩을 정의합니다. \
(\~/home/ec2-user/environment/myeks/k8s\_dashboard/eks-admin-service-account.yaml 에 이미 정의되어 있습니다. 앞서 Chapter에서 git clone을 수행하였다면 생략합니다. )

```
cat <<EoF > ~/environment/myeks/k8s_dashboard/eks-admin-service-account.yaml
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
EoF
```

서비스 계정 및 클러스터 역할 바인딩을 클러스터에 적용합니다.

```
kubectl apply -f ~/environment/myeks/k8s_dashboard/eks-admin-service-account.yaml

```

아래와 같은 결과를 확인 할 수 있습니다.

```
$ kubectl apply -f eks-admin-service-account.yaml
serviceaccount/eks-admin created
clusterrolebinding.rbac.authorization.k8s.io/eks-admin created
```

### 4. Kubernetes 대쉬보드 접속

클러스터에 Kubernetes 대시보드가 배포되었고, 클러스터를 보고 제어하는 데 사용할 수 있는 관리자 서비스 계정을 가졌기 때문에 이제 이 서비스 계정을 사용하여 대시보드에 연결할 수 있습니다.

Kubernetes 대시보드에 연결하기 위해서, eks-admin 서비스 계정에 대한 인증 토큰을 조회합니다. 출력에서  값을 복사합니다. 이 토큰을 사용하여 대시보드에 연결합니다.

```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')

```

출력 결과 예시

```
Name:         eks-admin-token-psdqk
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: eks-admin
              kubernetes.io/service-account.uid: 7e9b6218-bcb6-4262-9bf4-be55abdfc078

Type:  kubernetes.io/service-account-token

Data
====
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IkVJZ3JvMjV6Y1NqWTlRM2VLN0E0WXBUR19tbFMyLUpNa01XbTdOT1ROdzQifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJla3MtYWRtaW4tdG9rZW4tcHNkcWsiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZWtzLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiN2U5YjYyMTgtYmNiNi00MjYyLTliZjQtYmU1NWFiZGZjMDc4Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmVrcy1hZG1pbiJ9.knFP6lhXP47_XHoGlaKKWT6rgCopjLUOfEvQEHy_85gjOtHTcAviUUWCSynyW1Mvbs0Iif6e8QrP4dYTj_tWNrVuz6w_lwdRrBFjFn_WhtjUngRFPhi2gChgzkJGkWjHdn_nlBhBXUny6UjBxpc178cJqQelHR1fY_ITJ8EZ0EPo_LzDd4ezdcpvJjTO68p4FkMPkI968DZKGO6oIKQV7A_7mYar77hRQmq1K8iFtSKcs0eFSJxLf5RzvFHc0YH4YrKn7GuRX83TxwvmlbAqbQ-T74ktHmJThbp-0vWHftsl_1VmK11QrZAqcrj1PrRRn5H2tEIg-qBVnNbAYPq9Dg
ca.crt:     1066 bytes
namespace:  11 bytes
```

Kubernetes 대쉬보드 접속을 위해 kubectl proxy를 시작합니다.

```
kubectl proxy --port=8080 --address=0.0.0.0 --disable-filter=true &

```

{% hint style="success" %}
kubernetes dashboard는 ClusterIP 타입으로 서비스 배포됩니다. ClusterIP구성으로 외부에 노출되지 않습니다. 따라서 Kubectl proxy 설정이 필요합니다.
{% endhint %}

외부에서 접속되지 않으므로, Cloud9 IDE에서 Preview를 통해 Proxy로 접속합니다.

Cloud9의 상단 메뉴 Preview - Preview Running Application을 선택합니다. 메뉴에서 보이지 않는 경우 Tools - Preview - Preview Running Application을 선택합니다.

![](<../.gitbook/assets/image (397).png>)

생선된 Preview 브라우져에서 새로운 윈도우를 선택합니다.

![](<../.gitbook/assets/image (268).png>)

생성된 윈도우에서 아래 Path를 URL뒤에 복사해서 뒤에 붙여 넣습니다.

```
api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

앞서 생성한 eks-admin 토큰 값을 Kubernettes Dashboard에 입력합니다.

![](<../.gitbook/assets/image (283).png>)

정상적으로 생성된 것을 확인합니다.

![](<../.gitbook/assets/image (252).png>)

앞서 생성해 두었던 K9s로 어떤 namespace에 생성되었는지 살펴보고, 어떤 PoD들이 생성되었는지 확인해 봅니다.

```
k9s -A

```

### 5.Kubernetes 대쉬보드 삭제 (Option)

랩을 진행하면서 K8s Dashboard에서 Pod 현황을 살펴보고 싶다면, 삭제 하지 않습니다. 다음 Lab과 연관성이 없습니다.

Proxy를 먼저 삭제합니다.jobs의 결과에서 Proxy 번호를 확인하고 kill을 통해서 삭제합니다.

```
jobs
```

1번에 background process가 동작 중이라면 , 다음과 같이 삭제합니다.

```
kill %1
```

설치된 Dashboard를 삭제합니다.

```
kubectl delete -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.3/aio/deploy/recommended.yaml
```

저장된 변수를 삭제합니다.

```
unset DASHBOARD_VERSION
```

