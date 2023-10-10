---
description: 'update : 2023-09-19 /1h'
---

# 패키지 관리 - Helm

## Helm 소개

헬름은 [쿠버네티스](https://kubernetes.io/)용 소프트웨어를 검색하거나, 공유하고 사용하기에 가장 좋은 방법입니다. 쿠버네티스 차트를 관리하기 위한 도구이며, 차트는 사전 구성된 쿠버네티스 리소스의 패키지입니다. (리눅스의 apt, yum, pip 등과 같은 플랫폼 패키지 입니다.)

주요 3가지의 개념이 있습니다.

**차트**는 헬름 패키지입니다. 이 패키지에는 쿠버네티스 클러스터 내에서 애플리게이션, 도구, 서비스를 구동하는데 필요한 모들 리소스 정의가 포함되어 있습니다. 쿠버네티스에서의 Homebrew 포뮬러, Apt dpkg, YUM RPM 파일과 같은 것으로 생각할 수 있습니다.

**저장소**는 차트를 모아두고 공유하는 장소입니다. 이것은 마치 Perl의 [CPAN 아카이브](https://www.cpan.org/)나 [페도라 패키지 데이터베이스](https://admin.fedoraproject.org/pkgdb/)와 같은데, 쿠버네티스 패키지용이라고 보면 됩니다.

**릴리스**는 쿠버네티스 클러스터에서 구동되는 차트의 인스턴스입니다. 일반적으로 하나의 차트는 동일한 클러스터내에 여러 번 설치될 수 있다. 설치될 때마다, 새로운 _release_ 가 생성됩니다.

헬름은 쿠버네티스 내부에  charts를 설치하고, 각 설치에 대해 새로운 release를 생성하고, 새로운 차트를 찾기 위해 헬름 차트 repositories를 검색할 수 있습니다.

![참조 - https://devopscube.com/install-configure-helm-kubernetes/](<../.gitbook/assets/image (29).png>)

Helm Chart의 구조는 아래와 같습니다.

* values.yaml : 템플릿에 사용될 변수들 정
* tempates: 설치할 리소스 파일들이 존재하는 디렉토리. 해당 디렉토리 내부에 Deployment , Service 등이 존재.

## Helm 설치와 간단한 배포

아래와 같은 구성으로 LAB을 진행합니다.

![Helm LAB 목표 구성도](<../.gitbook/assets/image (44).png>)

아래와 같은 순서대로 구성합니다.

1. **Cloud9에 Helm3 최신 버전 설치**
2. **Cloud9에 원격 Chart Repo 구성**
3. **Cloud9에 Helm 명령어 자동완성 구성**
4. **Helm 을 통한 nginx 배포 및 확인.**
5. **Helm 을 통한 nginx 삭제.**

### 1.Helm 설치

헬름은 헬름 최신 버전을 자동으로 가져와서 [로컬에 설치](https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3)하는 인스톨러 스크립트를 제공합니다. 이 스크립트를 받아서 로컬에서 실행할 수 있습니다.&#x20;

```
cd ~/environment
curl -L https://git.io/get_helm.sh | bash -s -- --version v3.8.2

```

정상적으로 설치되었는지 확인합니다.

```
helm version --short

```

&#x20;출력 결과 예시

```
$ helm version --short
v3.8.2+g6e3701e

```

### 2. 차트 Repository 구성

Stable한 저장소를 다운로드하여 아래와 같이 구성합니다.

```
## helm repo 추가 
helm repo add stable https://charts.helm.sh/stable
helm repo update

```

설치 한 이후에는 설치 가능한 차트를 아래와 같은 명령으로 검색할 수 있습니다.

```
helm search repo stable

```

### 3. Helm 명령어 자동 완성 구성

Helm 명령에 대한 Bash 자동완성을 구성합니다.

```
## helem 자동완성 
helm completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
source <(helm completion bash)

```

### 4. **Helm 을 통한 nginx 배포 및 확인.**

Helm Chart를 통한 간단한 nginx 배포를 위해 repo에서 nginx를 검색합니다.

```
helm search repo nginx

```

출력 결과 예시

```
helm search repo nginx
NAME                            CHART VERSION   APP VERSION     DESCRIPTION                                       
stable/nginx-ingress            1.41.3          v0.34.1         DEPRECATED! An nginx Ingress controller that us...
stable/nginx-ldapauth-proxy     0.1.6           1.13.5          DEPRECATED - nginx proxy with ldapauth            
stable/nginx-lego               0.3.1                           Chart for nginx-ingress-controller and kube-lego  
stable/gcloud-endpoints         0.1.2           1               DEPRECATED Develop, deploy, protect and monitor...
```

간단한 웹서비스 배포를 위해 nginx를 추가해서 구동해 봅니다. 배포도구로 인기 있는 bitnami repo를 추가해서 nginx를 설치해 봅니다.

```
helm repo add bitnami https://charts.bitnami.com/bitnami

```

다시 nginx를 검색해 봅니다.

```
helm search repo bitnami/nginx
```

출력결과 예시

```
helm search repo bitnami/nginx
NAME                                    CHART VERSION   APP VERSION     DESCRIPTION                           
bitnami/nginx                           9.5.8           1.21.3          Chart for the nginx server            
bitnami/nginx-ingress-controller        8.0.9           1.0.4           Chart for the nginx Ingress controller
```

helm install 명령을 통해 nginx를 설치해 봅니다.

```
kubectl create namespace helm-test
helm install helm-nginx bitnami/nginx \
--namespace helm-test \
--set service.type=LoadBalancer \
--set service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-scheme"="internet-facing"

```

아래와 같은 결과를 얻을 수 있습니다.

```
$ helm install helm-nginx bitnami/nginx --namespace helm-test
NAME: helm-nginx
LAST DEPLOYED: Wed Oct 20 11:37:15 2021
NAMESPACE: helm-test
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
** Please be patient while the chart is being deployed **

NGINX can be accessed through the following DNS name from within your cluster:

    helm-nginx.helm-test.svc.cluster.local (port 80)

To access NGINX from outside the cluster, follow the steps below:

1. Get the NGINX URL by running these commands:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace helm-test -w helm-nginx'

    export SERVICE_PORT=$(kubectl get --namespace helm-test -o jsonpath="{.spec.ports[0].port}" services helm-nginx)
    export SERVICE_IP=$(kubectl get svc --namespace helm-test helm-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    echo "http://${SERVICE_IP}:${SERVICE_PORT}"
```

Pod와 서비스 배포를 확인합니다.

```
kubectl -n helm-test get service
```

```
$ kubectl -n helm-test get service
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP                                                                   PORT(S)        AGE
helm-nginx   LoadBalancer   172.20.141.34   adeb84a799b4046689bec6ed2b120c51-272707121.ap-northeast-2.elb.amazonaws.com   80:32303/TCP   50s
```

ELB 주소를 확인합니다.

```
export SERVICE_IP=$(kubectl get svc --namespace helm-test helm-nginx --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
echo "NGINX URL: http://$SERVICE_IP/"

```

해당 주소로 접속해 봅니다.

![](<../.gitbook/assets/image (355).png>)

EC2 대시보드에서 ELB가 정상적으로 생성된 것을 확인 할 수 있습니다.

![](<../.gitbook/assets/image (384).png>)

설치된 helm list를 확인합니다.

```
helm list -n helm-test

```

출력 예시

```
$ helm list -n helm-test
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
helm-nginx      helm-test       1               2021-10-20 11:37:15.991941399 +0000 UTC deployed        nginx-9.5.8     1.21.3   
```

아래 명령을 통해 배포된 내용을 확인합니다.

```
kubectl describe deployments.apps helm-nginx -n helm-test

```

출력 결과 예시

```
$ kubectl describe deployments.apps helm-nginx -n helm-test
Name:                   helm-nginx
Namespace:              helm-test
CreationTimestamp:      Wed, 20 Oct 2021 11:37:16 +0000
Labels:                 app.kubernetes.io/instance=helm-nginx
                        app.kubernetes.io/managed-by=Helm
                        app.kubernetes.io/name=nginx
                        helm.sh/chart=nginx-9.5.8
Annotations:            deployment.kubernetes.io/revision: 1
                        meta.helm.sh/release-name: helm-nginx
                        meta.helm.sh/release-namespace: helm-test
Selector:               app.kubernetes.io/instance=helm-nginx,app.kubernetes.io/name=nginx
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:           app.kubernetes.io/instance=helm-nginx
                    app.kubernetes.io/managed-by=Helm
                    app.kubernetes.io/name=nginx
                    helm.sh/chart=nginx-9.5.8
  Service Account:  default
  Containers:
   nginx:
    Image:      docker.io/bitnami/nginx:1.21.3-debian-10-r29
    Port:       8080/TCP
    Host Port:  0/TCP
    Liveness:   tcp-socket :http delay=0s timeout=5s period=10s #success=1 #failure=6
    Readiness:  tcp-socket :http delay=5s timeout=3s period=5s #success=1 #failure=3
    Environment:
      BITNAMI_DEBUG:  false
    Mounts:           <none>
  Volumes:
   nginx-server-block-paths:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      helm-nginx-server-block
    Optional:  false
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   helm-nginx-6c4788c7cf (1/1 replicas created)
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  2m49s  deployment-controller  Scaled up replica set helm-nginx-6c4788c7cf to 1
```

### 5. Helm을 통한 nginx 삭제

아래 명령을 통해 생성된 Helm을 삭제합니다.

```
helm uninstall helm-nginx -n helm-test

```

정상적으로 삭제되었는지 확인합니다.

```
helm list -n helm-test
kubectl -n helm-test get service 

```

{% hint style="info" %}
ELB 삭제 시간으로 3분 정도 소요됩니다.
{% endhint %}

## Helm을 이용한 Microservice 배포.

아래와 같은 구성으로 LAB을 진행합니다.

![ Helm Chart 기반의 App배포](<../.gitbook/assets/image (387).png>)

### 6.Chart 만들기

helmdemo 라는 Chart를 생성합니다.

```
cd ~/environment/
helm create helm-chart-demo
sudo yum -y install tree

```

Helm Chart를 생성하면, 아래와 같은 디렉토리 구조를 생성되어 있습니다.&#x20;

```
cd ~/environment/helm-chart-demo
~/environment/helm-chart-demo $ sudo yum -y install tree
~/environment/helm-chart-demo $ tree 
.
├── charts
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt
│   ├── serviceaccount.yaml
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```

* `deployment.yaml`: Kubernetes 배포를 만들기위한 기본 목록
* `_helpers.tpl`: 차트 전체에서 재사용 할 수있는 템플릿 도우미를 배치 할 수있는 장소
* `ingress.yaml`: service를 위한 Kubernetes ingress object 생성을 위한 기본 목록
* `NOTES.txt`: 차트의“도움말 텍스트”. 사용자가 Helm Install을 실행할 때 이 정보가 표시됩니다.
* `serviceaccount.yaml`: 서비스 계정을 만들기위한 기본 목록입니다.
* `service.yaml`: Depolyment 를 위한 서비스 엔드 포인트 작성을 기본 목록
* `tests/`: 차트 테스트가 포함 된 폴더

### 7. Chart 사용자 정의

새로운 Chart구성을 위해 기본 생성된 파일들을 삭제합니다.

```
rm -rf ~/environment/helm-chart-demo/templates/
rm ~/environment/helm-chart-demo/Chart.yaml
rm ~/environment/helm-chart-demo/values.yaml

```

다음 코드 블록을 실행하여 새로운 chart.yaml 파일을 생성합니다.

```
cat <<EoF > ~/environment/helm-chart-demo/Chart.yaml
apiVersion: v2
name: helm-chart-demo
description: A Helm chart for EKS Workshop Microservices application
version: 0.1.0
appVersion: 1.0
EoF

```

앞서 git에서 복제한 파일들 중에서 helm-chart-demo 폴더의 파일을 복사합니다.

```
# 각 템플릿 타입을 위한 서브 폴더 생성 
mkdir -p ~/environment/helm-chart-demo/templates/deployment
mkdir -p ~/environment/helm-chart-demo/templates/service

# helm chart pod deploy용 manifest 파일 복사 

cp ~/environment/myeks/helm-chart-demo/ecsdemo-frontend-deployment.yaml ~/environment/helm-chart-demo/templates/deployment/frontend.yaml
cp ~/environment/myeks/helm-chart-demo/ecsdemo-crystal-deployment.yaml ~/environment/helm-chart-demo/templates/deployment/crystal.yaml
cp ~/environment/myeks/helm-chart-demo/ecsdemo-nodejs-deployment.yaml ~/environment/helm-chart-demo/templates/deployment/nodejs.yaml

# helm chart pod service용 manifest 파일 복사 
cp ~/environment/myeks/helm-chart-demo/ecsdemo-frontend-service.yaml ~/environment/helm-chart-demo/templates/service/frontend.yaml
cp ~/environment/myeks/helm-chart-demo/ecsdemo-crystal-service.yaml ~/environment/helm-chart-demo/templates/service/crystal.yaml
cp ~/environment/myeks/helm-chart-demo/ecsdemo-nodejs-service.yaml ~/environment/helm-chart-demo/templates/service/nodejs.yaml

```

아래 deploy용 yaml manifest 파일은 replica와 image 값이 helm chart의 value를 참조하도록 선언되어 있습니다. 각 파일에서 확인해 봅니다. (ecsdemo-frontend-deployment.yaml, ecsdemo-crystal-deployment.yaml, ecsdemo-nodejs-deployment.yaml)

```
metadata:
  name: ecsdemo-xxxx
  labels:
    app: ecsdemo-xxxx
  namespace: helm-chart-demo
replicas: {{ .Values.replicas }}
- image: {{ .Values.frontend.image }}:{{ .Values.version }}

```



이제 values.yaml을 생성하고, Values 들에 대한 값을 정의합니다.

```
cat <<EoF > ~/environment/helm-chart-demo/values.yaml
# Default values for eksdemo.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

# Release-wide Values
namespace: 'helm-chart-demo'
replicas: 3
version: 'latest'

# Service Specific Values
nodejs:
  image: brentley/ecsdemo-nodejs
crystal:
  image: brentley/ecsdemo-crystal
frontend:
  image: brentley/ecsdemo-frontend
EoF

```

helmdemo에 사용할 namespace를 생성합니다.

```
kubectl create namespace helm-chart-demo

```

### 8. Chart 배포

Helm chart는 실제 배포하지 않고 **"--dry-run"** 플래그를 사용하여, 랜더링 된 템플릿을 빌드하고 출력할 수 있습니다.

```
helm install --debug --dry-run workshop ~/environment/helm-chart-demo

```

출력 결과 예제

```
$ helm install --debug --dry-run workshop ~/environment/helm-chart-demo
install.go:178: [debug] Original chart version: ""
install.go:199: [debug] CHART PATH: /home/ec2-user/environment/helm-chart-demo

NAME: workshop
LAST DEPLOYED: Wed Oct 20 13:36:24 2021
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
crystal:
  image: brentley/ecsdemo-crystal
frontend:
  image: brentley/ecsdemo-frontend
namespace: helm-chart-demo
nodejs:
  image: brentley/ecsdemo-nodejs
replicas: 3
version: latest

HOOKS:
MANIFEST:
---
# Source: helm-chart-demo/templates/service/crystal.yaml
apiVersion: v1
kind: Service
metadata:
  name: ecsdemo-crystal
#name space change 
  namespace: helm-chart-demo
spec:
  selector:
    app: ecsdemo-crystal
  ports:
   -  protocol: TCP
      port: 80
      targetPort: 3000
---
# Source: helm-chart-demo/templates/service/frontend.yaml
apiVersion: v1
kind: Service
metadata:
  name: ecsdemo-frontend
#name space change 
  namespace: helm-chart-demo
spec:
  selector:
    app: ecsdemo-frontend
#Service Type change
  type: LoadBalancer
  ports:
   -  protocol: TCP
      port: 80
      targetPort: 3000
---
# Source: helm-chart-demo/templates/service/nodejs.yaml
apiVersion: v1
kind: Service
metadata:
  name: ecsdemo-nodejs
#name space change 
  namespace: helm-chart-demo
spec:
  selector:
    app: ecsdemo-nodejs
  ports:
   -  protocol: TCP
      port: 80
      targetPort: 3000
---
# Source: helm-chart-demo/templates/deployment/crystal.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecsdemo-crystal
  labels:
    app: ecsdemo-crystal
#name space change 
  namespace: helm-chart-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ecsdemo-crystal
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: ecsdemo-crystal
    spec:
      containers:
      - image: brentley/ecsdemo-crystal:latest
        imagePullPolicy: Always
        name: ecsdemo-crystal
        ports:
        - containerPort: 3000
          protocol: TCP
#add nodeSelector
      nodeSelector:
        nodegroup-type: "backend-workloads"
---
# Source: helm-chart-demo/templates/deployment/frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecsdemo-frontend
  labels:
    app: ecsdemo-frontend
#name space change 
  namespace: helm-chart-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ecsdemo-frontend
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: ecsdemo-frontend
    spec:
      containers:
      - image: brentley/ecsdemo-frontend:latest
        imagePullPolicy: Always
        name: ecsdemo-frontend
        ports:
        - containerPort: 3000
          protocol: TCP
        env:
#Container URL change.
        - name: CRYSTAL_URL
          value: "http://ecsdemo-crystal.clb-test.svc.cluster.local/crystal"
        - name: NODEJS_URL
          value: "http://ecsdemo-nodejs.clb-test.svc.cluster.local/"
#add nodeSelector
      nodeSelector:
        nodegroup-type: "backend-workloads"
---
# Source: helm-chart-demo/templates/deployment/nodejs.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecsdemo-nodejs
  labels:
    app: ecsdemo-nodejs
#name space change 
  namespace: helm-chart-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ecsdemo-nodejs
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: ecsdemo-nodejs
    spec:
      containers:
      - image: brentley/ecsdemo-nodejs:latest
        imagePullPolicy: Always
        name: ecsdemo-nodejs
        ports:
        - containerPort: 3000
          protocol: TCP
#add nodeSelector
      nodeSelector:
        nodegroup-type: "backend-workloads"
```

템플릿이 정상적으로 렌더링되고, 빌드 출력이 이뤄졌으므로 차트를 배포합니다.

```
helm -n helm-chart-demo install rollingback-app ~/environment/helm-chart-demo/

```

출력 결과 예시

```
$ helm -n helm-chart-demo install rollingback-app ~/environment/helm-chart-demo/
NAME: rollingback-app
LAST DEPLOYED: Wed Oct 20 13:36:51 2021
NAMESPACE: helm-chart-demo
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

정상적으로 배포되었는 지 확인해 봅니다.

```
kubectl -n helm-chart-demo get svc,pod,deploy -o wide

```

출력 결과 예시

```
$ kubectl -n helm-chart-demo get svc,pod,deploy -o wide
NAME                       TYPE           CLUSTER-IP       EXTERNAL-IP                                                                    PORT(S)        AGE   SELECTOR
service/ecsdemo-crystal    ClusterIP      172.20.24.250    <none>                                                                         80/TCP         11s   app=ecsdemo-crystal
service/ecsdemo-frontend   LoadBalancer   172.20.185.71    a79963b6877134227a0c6e5760d57bd5-1823214377.ap-northeast-2.elb.amazonaws.com   80:31678/TCP   11s   app=ecsdemo-frontend
service/ecsdemo-nodejs     ClusterIP      172.20.254.165   <none>                                                                         80/TCP         11s   app=ecsdemo-nodejs

NAME                                    READY   STATUS    RESTARTS   AGE   IP              NODE                                               NOMINATED NODE   READINESS GATES
pod/ecsdemo-crystal-86b65cdbfd-dsh8j    1/1     Running   0          11s   10.11.94.60     ip-10-11-94-22.ap-northeast-2.compute.internal     <none>           <none>
pod/ecsdemo-crystal-86b65cdbfd-szx4f    1/1     Running   0          11s   10.11.106.164   ip-10-11-108-153.ap-northeast-2.compute.internal   <none>           <none>
pod/ecsdemo-crystal-86b65cdbfd-tw9b6    1/1     Running   0          11s   10.11.72.5      ip-10-11-64-123.ap-northeast-2.compute.internal    <none>           <none>
pod/ecsdemo-frontend-576d7899b5-5pkjm   1/1     Running   0          11s   10.11.104.156   ip-10-11-108-153.ap-northeast-2.compute.internal   <none>           <none>
pod/ecsdemo-frontend-576d7899b5-98tf6   1/1     Running   0          11s   10.11.83.133    ip-10-11-94-22.ap-northeast-2.compute.internal     <none>           <none>
pod/ecsdemo-frontend-576d7899b5-trqbh   1/1     Running   0          11s   10.11.77.146    ip-10-11-64-123.ap-northeast-2.compute.internal    <none>           <none>
pod/ecsdemo-nodejs-64c4fd7579-2hdr7     1/1     Running   0          11s   10.11.78.121    ip-10-11-64-123.ap-northeast-2.compute.internal    <none>           <none>
pod/ecsdemo-nodejs-64c4fd7579-jmktc     1/1     Running   0          11s   10.11.83.38     ip-10-11-94-22.ap-northeast-2.compute.internal     <none>           <none>
pod/ecsdemo-nodejs-64c4fd7579-vqdsd     1/1     Running   0          11s   10.11.98.185    ip-10-11-108-153.ap-northeast-2.compute.internal   <none>           <none>

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS         IMAGES                             SELECTOR
deployment.apps/ecsdemo-crystal    3/3     3            3           11s   ecsdemo-crystal    brentley/ecsdemo-crystal:latest    app=ecsdemo-crystal
deployment.apps/ecsdemo-frontend   3/3     3            3           11s   ecsdemo-frontend   brentley/ecsdemo-frontend:latest   app=ecsdemo-frontend
deployment.apps/ecsdemo-nodejs     3/3     3            3           11s   ecsdemo-nodejs     brentley/ecsdemo-nodejs:latest     app=ecsdemo-nodejs
```

### 9. 서비스 확인

아래 명령을 통해 service loadbalancer(ELB) 의 DNS name을 확인합니다.

```
kubectl -n helm-chart-demo get svc ecsdemo-frontend

```

출력 결과 예시

```
$ kubectl -n helm-chart-demo get svc ecsdemo-frontend
NAME               TYPE           CLUSTER-IP      EXTERNAL-IP                                                                    PORT(S)        AGE
ecsdemo-frontend   LoadBalancer   172.20.185.71   a79963b6877134227a0c6e5760d57bd5-1823214377.ap-northeast-2.elb.amazonaws.com   80:31678/TCP   3m30s
```

정상적으로 서비스에 접속되는 것을 확인 할 수 있습니다.

![](<../.gitbook/assets/image (391).png>)

helm list에도 정상적으로 등록되어 있는지 확인합니다.

```
helm list -n helm-chart-demo

```

출력 결과 예

```
$ helm list -n helm-chart-demo
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
rollingback-app helm-chart-demo 1               2021-10-20 13:36:51.965888237 +0000 UTC deployed        helm-chart-demo-0.1.0   1          
```

### 10. Rolling Back 구성

배포 중에 이슈가 발생하더라도, Helm을 사용하면 손쉽게 이전 배포 버전으로 Rolling back 할 수 있습니다. 앞서 출력결과에서 "Revision" 값이 "1"인 것을 확인했습니다. 이제 helm Chart를 변경하고 업데이트 한 이후 , 다시 Rollback 하는 과정을 수행하겠습니다.

먼저 Helm Chart로 배포한 내용을 확인합니다. REVISON:1 으로 구성된 것을 확인 할 수 있습니다.

```
helm history -n helm-chart-demo rollingback-app

```

```
$ helm history -n helm-chart-demo rollingback-app
REVISION        UPDATED                         STATUS          CHART                   APP VERSION     DESCRIPTION     
1               Wed Oct 20 13:36:51 2021        deployed        helm-chart-demo-0.1.0   1               Install complete
```

Rolling Back 시험을 위해, 먼저 helm Chart의 Value를 변경합니다. \~/environment/helm-chart-demo/values.yaml 파을 다시 열고, Cloud9 IDE 편집기에서 replicat의 값을 아래와 같이 수정합니다.

```
replicas: 1

```

변경 후 File 내용 확인 (values.yaml)

```
# Default values for eksdemo.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

# Release-wide Values
namespace: 'helm-chart-demo'
## replicas를 3에서 1로 변경합니다
replicas: 1
version: 'latest'

```

"values.yaml" 파일이 모두 수정되었으면, helm을 upgrade 합니다.

```
helm upgrade -n helm-chart-demo rollingback-app ~/environment/helm-chart-demo/

```

출력 결과 예시

```
$ helm upgrade -n helm-chart-demo rollingback-app ~/environment/helm-chart-demo/
Release "rollingback-app" has been upgraded. Happy Helming!
NAME: rollingback-app
LAST DEPLOYED: Wed Oct 20 13:45:18 2021
NAMESPACE: helm-chart-demo
STATUS: deployed
REVISION: 2
TEST SUITE: None
```

{% hint style="warning" %}
helm chart로 배포한 Revision이 "2"로 변경되었습니다.
{% endhint %}

정상적으로 배포되었는 지 확인합니다.

```
kubectl -n helm-chart-demo get pods

```

아래와 같이 pod 의 숫자가 1개로 줄었습니다.

```
$ kubectl -n helm-chart-demo get pods
NAME                                READY   STATUS    RESTARTS   AGE
ecsdemo-crystal-86b65cdbfd-szx4f    1/1     Running   0          9m10s
ecsdemo-frontend-576d7899b5-5pkjm   1/1     Running   0          9m10s
ecsdemo-nodejs-64c4fd7579-jmktc     1/1     Running   0          9m10s
```

문제 해결을 위해 Rollback을 수행합니다.

먼저 현재 helm Chart를 통해 배포된 상태를 확인합니다.

```
helm status -n helm-chart-demo rollingback-app 

```

아래와 같은 결과를 확인 할 수 있습니다.최종 배포시간과 Revision 값을 확인합니다.

```
$ helm status -n helm-chart-demo rollingback-app 
NAME: rollingback-app
LAST DEPLOYED: Wed Oct 20 13:45:18 2021
NAMESPACE: helm-chart-demo
STATUS: deployed
REVISION: 2
TEST SUITE: None
```

helm 배포 히스토리를 확인합니다.

```
helm history -n helm-chart-demo rollingback-app

```

아래와 같은 결과를 확인 할 수 있습니다. Revison을 확인하고, 정상적으로 수행되었던 Revision 값으로 Rollback 할 것입니다.

```
$ helm history -n helm-chart-demo rollingback-app
REVISION        UPDATED                         STATUS          CHART                   APP VERSION     DESCRIPTION     
1               Wed Oct 20 13:36:51 2021        superseded      helm-chart-demo-0.1.0   1               Install complete
2               Wed Oct 20 13:45:18 2021        deployed        helm-chart-demo-0.1.0   1               Upgrade complete
```

Rollback 명령을 통해 정상 배포 버전으로 Rolling Back  합니다.

```
helm rollback -n helm-chart-demo rollingback-app 1

```

이제 pod들이 다시 3개로 확장되었는지 확인해 봅니다.

```
kubectl -n helm-chart-demo get pods

```

아래와 같이 정상적으로 Pod가 배포됩니다.

```
$ kubectl -n helm-chart-demo get pods
NAME                                READY   STATUS    RESTARTS   AGE
ecsdemo-crystal-86b65cdbfd-bfskh    1/1     Running   0          25s
ecsdemo-crystal-86b65cdbfd-hm796    1/1     Running   0          25s
ecsdemo-crystal-86b65cdbfd-szx4f    1/1     Running   0          11m
ecsdemo-frontend-576d7899b5-5pkjm   1/1     Running   0          11m
ecsdemo-frontend-576d7899b5-db7jq   1/1     Running   0          25s
ecsdemo-frontend-576d7899b5-j7gqf   1/1     Running   0          25s
ecsdemo-nodejs-64c4fd7579-8jgpr     1/1     Running   0          25s
ecsdemo-nodejs-64c4fd7579-jmktc     1/1     Running   0          11m
ecsdemo-nodejs-64c4fd7579-pznvw     1/1     Running   0          25s
```

helm history를 통해 Revision을 확인해 봅니다.

```
helm history -n helm-chart-demo rollingback-app

```

d아래와 같이 REVISON 3은 DESCRIPTION 에 자동으로 Rollback 되었고, REVISION 1번으로 복귀하였음을 알려줍니다.

```
$ helm history -n helm-chart-demo rollingback-app
REVISION        UPDATED                         STATUS          CHART                   APP VERSION     DESCRIPTION     
1               Wed Oct 20 13:36:51 2021        superseded      helm-chart-demo-0.1.0   1               Install complete
2               Wed Oct 20 13:45:18 2021        superseded      helm-chart-demo-0.1.0   1               Upgrade complete
3               Wed Oct 20 13:47:51 2021        deployed        helm-chart-demo-0.1.0   1               Rollback to 1   
```

\~/environment/helm-chart-demo/values.yaml 파을 다시 열고, Cloud9 IDE 편집기에서 replicat의 값을 아래와 같이 수정합니다.해 value.yaml 파일 다시 정상적으로 수정합니다.

```
replicas: 3

```

생성했던 helmdemo를 삭제합니다. 아래 ChartMuseum 구성과 배포를 위해서 반드시 삭제합니다.

```
helm uninstall -n helm-chart-demo rollingback-app

```

## ChartMuseum 구성과 배포.

ChartMuseum은 Amazon S3,Google Cloud Storage, , Microsoft Azure Blob Storage, Alibaba Cloud OSS Storage, Openstack Object Storage, Oracle Cloud Infrastructure Object를 포함한 클라우드 스토리지 백엔드를 지원하는 Go (Golang)로 작성된 오픈 소스 Helm Chart Repository 서버입니다.&#x20;

이 랩에서는 ChartMuseum을 구성하여, S3에 Helm Chart를 위한 로컬 레포지토리를 만들어 보겠습니다.

### 1.ChartMuseum 설치.

먼저 ChartMuseum을 설치합니다.

```
cd ~/environment
curl -LO https://s3.amazonaws.com/chartmuseum/release/latest/bin/linux/amd64/chartmuseum
chmod +x ./chartmuseum

```

Cloud9 IDE를 이용해서 Chartmuseum을 구동합니다. 스토리지 저장소는 S3를 사용합니다.\
사전에 s3 bucket을 AWS CLI 또는 Console 에서 생성합니다.

S3를 CLI로 생성합니다.&#x20;

```
##S3 Bucket 생성합니다. 
##Bucket name은 고유해야 합니다.
export s3chartmuseum=usernamechartmuseum
echo "export s3chartmuseum=${s3chartmuseum}" | tee -a ~/.bash_profile
aws s3 mb s3://${s3chartmuseum}

```

\[참고 - AWS Console] AWS 서비스 콘솔에서 S3를 생성합니다

AWS 서비스 - S3&#x20;

![](<../.gitbook/assets/image (266).png>)

버킷이름은 고유해야 합니다.

![](<../.gitbook/assets/image (186).png>)

dkfo아래와 같이 Cloud9  IDE에서 aws cli로 실행하여, S3 버킷을&#x20;

정상적으로 생성되었는지 확인합니다.

```
aws s3 ls | grep 'chartmuseum'

```

```
whchoi98:~/environment $ aws s3 ls | grep 'chartmuseum'
2021-04-06 08:14:49 whchoi-chartmuseum-2021-04-06
```

아래 명령어를 통해서 debug하고 정상적으로 동작하는지 확인합니다. **`--storage-amazon-bucket="생성한 버킷 이름"`** 을 입력합니다.

```
./chartmuseum --debug --port=8888 --storage="amazon" --storage-amazon-bucket=whchoi-chartmuseum-2021-04-06 --storage-amazon-prefix="" --storage-amazon-region="ap-northeast-2" &
./chartmuseum --debug --port=8888 --storage="amazon" --storage-amazon-bucket=${s3chartmuseum} --storage-amazon-prefix="" --storage-amazon-region="ap-northeast-2" &      
```

Cloud9 IDE의 Security Group에서 Chartmuseum으로 사용될 서비스 포트를 오픈해 주고, Cloud9의 공인 IP를 확인합니다.&#x20;

* EC2 - 인스턴스 - Cloud9 인스턴스 선택 - 퍼블릭 IPv4 주소&#x20;

![](<../.gitbook/assets/image (46).png>)

* EC2 - 인스턴스 -Cloud9 인스턴스 선택 - 보안 - 보안 그룹  선택

![](<../.gitbook/assets/image (70).png>)

* 인바운드 규칙 편집 선택&#x20;

![](<../.gitbook/assets/image (337).png>)

* TCP 8888 포트 추가 / 소스 Anywhere-IPv4&#x20;

![](<../.gitbook/assets/image (210).png>)

helm repo list에 정상적으로 등록되었는지 확인합니다.

```
helm repo list

```

이제 외부에서 정상적으로 서비스가 접속되는 지 확인해 봅니다.

Cloud9 IDE 콘솔에서 Public IP를 확인해 봅니다

```
curl http://169.254.169.254/latest/meta-data/public-ipv4

```

![](<../.gitbook/assets/image (340).png>)

### 2. Chart 패키징 및 업로드.

Helm Client (Cloud9 IDE)에 저장소를 추가해 봅니다.

```
helm repo add chartmuseum http://localhost:8888

```

출력 결과 예시

```
whchoi98:~/environment $ helm repo add chartmuseum http://localhost:8888
2021-04-06T08:19:36.368Z        DEBUG   [3] Incoming request: /index.yaml       {"reqID": "572624a5-6e8e-4566-a071-1828c7aafeb7"}
2021-04-06T08:19:36.368Z        DEBUG   [3] Entry found in cache store  {"repo": "", "reqID": "572624a5-6e8e-4566-a071-1828c7aafeb7"}
2021-04-06T08:19:36.368Z        DEBUG   [3] Fetching chart list from storage    {"repo": "", "reqID": "572624a5-6e8e-4566-a071-1828c7aafeb7"}
2021-04-06T08:19:36.424Z        DEBUG   [3] No change detected between cache and storage        {"repo": "", "reqID": "572624a5-6e8e-4566-a071-1828c7aafeb7"}
2021-04-06T08:19:36.424Z        INFO    [3] Request served      {"path": "/index.yaml", "comment": "", "clientIP": "127.0.0.1", "method": "GET", "statusCode": 200, "latency": "56.032676ms", "reqID": "572624a5-6e8e-4566-a071-1828c7aafeb7"}
"chartmuseum" has been added to your repositories
```

이제 앞서 생성한 eksdemo helm chart 패키징 합니다.&#x20;

```
cd ~/environment/helm-chart-demo/
helm package ./ 

```

출력 결과 예시

```
~/environment/helm-chart-demo $ helm package ./ 
Successfully packaged chart and saved it to: /home/ec2-user/environment/helm-chart-demo/eksdemo-0.1.0.tgz
```

eksdemo heml chart가 정상적으로 패키징 되었는지 확인합니다.

```
~/environment/helm-chart-demo $ tree
.
├── charts
├── Chart.yaml
├── eksdemo-0.1.0.tgz
├── templates
│   ├── deployment
│   │   ├── crystal.yaml
│   │   ├── frontend.yaml
│   │   └── nodejs.yaml
│   └── service
│       ├── crystal.yaml
│       ├── frontend.yaml
│       └── nodejs.yaml
└── values.yaml
```

### 3. Chartmuseum 으로 부터 배포

Chartmuseum에 패키징을 업로드 합니다.

```
curl --data-binary "@helm-chart-demo-0.1.0.tgz" http://localhost:8888/api/charts

```

출력결과 예제

```
whchoi98:~/environment/helm-chart-demo $ curl --data-binary "@helm-chart-demo-0.1.0.tgz" http://localhost:8888/api/charts
2021-04-06T08:44:51.077Z        DEBUG   [16] Incoming request: /api/charts      {"reqID": "7d175c21-50e7-43b2-9e81-5922ede8982d"}
2021-04-06T08:44:51.108Z        DEBUG   [16] Adding package to storage  {"package": "helm-chart-demo-0.1.0.tgz", "reqID": "7d175c21-50e7-43b2-9e81-5922ede8982d"}
2021-04-06T08:44:51.137Z        INFO    [16] Request served     {"path": "/api/charts", "comment": "", "clientIP": "127.0.0.1", "method": "POST", "statusCode": 201, "latency": "60.471257ms", "reqID": "7d175c21-50e7-43b2-9e81-5922ede8982d"}
{"saved":true}
```

S3에 정상적으로 Chartmuseum이 배포되었는지 확인합니다.

```
aws s3 ls s3://${s3chartmuseumt}

```

출력 결과 예제

```
aws s3 ls s3://${s3chartmuseumt}
2022-02-14 14:28:03       1400 helm-chart-demo-0.1.0.tgz
```

이제 등록된 Repo를 업데이트하고, Chartmuseum 로컬 레포지토리를 검색해 봅니다.

```
helm repo update
helm search repo chartmuseum

```

출력결과 예제

```
whchoi98:~/environment/helm-chart-demo $ helm search repo chartmuseum
NAME                            CHART VERSION   APP VERSION     DESCRIPTION                                       
stable/chartmuseum              2.14.2          0.12.0          DEPRECATED Host your own Helm Chart Repository    
chartmuseum/helm-chart-demo     0.1.0           1               A Helm chart for EKS Workshop Microservices app...
```

등록된 Chartmuseum 로컬 레포지토리에서 패키지를 배포합니다.

```
helm install chartmuseum/helm-chart-demo --generate-name

```

출력 결과 예시

```
helm install chartmuseum/helm-chart-demo --generate-name
2021-10-20T14:07:42.333Z        DEBUG   [6] Incoming request: /charts/helm-chart-demo-0.1.0.tgz {"reqID": "bf9836df-470a-4894-998c-a8b37b3ae8e9"}
2021-10-20T14:07:42.372Z        INFO    [6] Request served      {"path": "/charts/helm-chart-demo-0.1.0.tgz", "comment": "", "clientIP": "127.0.0.1", "method": "GET", "statusCode": 200, "latency": "39.261297ms", "reqID": "bf9836df-470a-4894-998c-a8b37b3ae8e9"}
NAME: helm-chart-demo-1634738862
LAST DEPLOYED: Wed Oct 20 14:07:43 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

정상적으로 배포되었는지 확인합니다.

```
kubectl -n helm-chart-demo get svc
```

출력 결과 예시

```
$ kubectl -n helm-chart-demo get svc
NAME               TYPE           CLUSTER-IP       EXTERNAL-IP                                                                   PORT(S)        AGE
ecsdemo-crystal    ClusterIP      172.20.60.158    <none>                                                                        80/TCP         116s
ecsdemo-frontend   LoadBalancer   172.20.169.42    a738ceddaa67142e9af23d09bd39df2e-322542006.ap-northeast-2.elb.amazonaws.com   80:31103/TCP   116s
ecsdemo-nodejs     ClusterIP      172.20.238.125   <none>                                                                           80/TCP         29s                                                                     80/TCP         102s
```

ELB DNS 레코드로 접속해 봅니다.

![](<../.gitbook/assets/image (274).png>)

{% hint style="info" %}
Helm Chartmuseum은 이제 AWS ECR과도 연동이 가능해 졌습니다. [https://docs.aws.amazon.com/ko\_kr/AmazonECR/latest/userguide/push-oci-artifact.html](https://docs.aws.amazon.com/ko\_kr/AmazonECR/latest/userguide/push-oci-artifact.html)
{% endhint %}

