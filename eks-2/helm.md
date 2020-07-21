# Helm 구성

## Helm 소개

헬름은 [쿠버네티스](https://kubernetes.io/)용 소프트웨어를 검색하거나, 공유하고 사용하기에 가장 좋은 방법입니다. 쿠버네티스 차트를 관리하기 위한 도구이며, 차트는 사전 구성된 쿠버네티스 리소스의 패키지입니다.

주요 3가지의 개념이 있습니다.

**차트**는 헬름 패키지입니. 이 패키지에는 쿠버네티스 클러스터 내에서 애플리게이션, 도구, 서비스를 구동하는데 필요한 모들 리소스 정의가 포함되어 있습니다. 쿠버네티스에서의 Homebrew 포뮬러, Apt dpkg, YUM RPM 파일과 같은 것으로 생각할 수 있습니다.

**저장소**는 차트를 모아두고 공유하는 장소입다. 이것은 마치 Perl의 [CPAN 아카이브](https://www.cpan.org/)나 [페도라 패키지 데이터베이스](https://admin.fedoraproject.org/pkgdb/)와 같은데, 쿠버네티스 패키지용이라고 보면 됩니다.

**릴리스**는 쿠버네티스 클러스터에서 구동되는 차트의 인스턴스입다. 일반적으로 하나의 차트는 동일한 클러스터내에 여러 번 설치될 수 있다. 설치될 때마다, 새로운 _release_ 가 생성됩니다.

헬름은 쿠버네티스 내부에 \_charts\_를 설치하고, 각 설치에 대해 새로운 \_release\_를 생성하고, 새로운 차트를 찾기 위해 헬름 차트 \_repositories\_를 검색할 수 있습니다.

![&#xCC38;&#xC870; - https://devopscube.com/install-configure-helm-kubernetes/](../.gitbook/assets/image%20%2847%29.png)

## Helm 설치와 간단한 배포

아래와 같은 순서대로 구성합니다.

1. **Helm3 최신 버전 설치**
2. **Chart Repo 구성**
3. **Helm 명령어 자동완성 구성**
4. **Helm 을 통한 nginx 배포 및 확인.**
5. **Helm 을 통한 nginx 삭제.**

### 1.Helm 설치

헬름은 헬름 최신 버전을 자동으로 가져와서 [로컬에 설치](https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3)하는 인스톨러 스크립트를 제공합니다. 이 스크립트를 받아서 로컬에서 실행할 수 있습니다.

```text
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
chmod 700 get_helm.sh
./get_helm.sh
```

정상적으로 설치되었는지 확인합니다.

```text
helm version --short
```

 출력 결과 예시

```text
whchoi98:~/environment $ helm version --short
v3.2.4+g0ad800e
```

### 2. 차트 Repository 구성

Stable한 저장소를 다운로드하여 아래와 같이 구성합니다.

```text
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm repo update
```

설치 한 이후에는 설치 가능한 차트를 아래와 같은 명령으로 검색할 수 있습니다.

```text
helm search repo stable
```

### 3. Helm 명령어 자동 완성 구성

Helm 명령에 대한 Bash 자동완성을 구성합니다.

```text
helm completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
source <(helm completion bash)
```

### 4. **Helm 을 통한 nginx 배포 및 확인.**

Helm Chart를 통한 간단한 nginx 배포를 위해 repo에서 nginx를 검색합니다.

```text
helm search repo nginx
```

출력 결과 예시

```text
whchoi98:~/environment $ helm search repo nginx
NAME                            CHART VERSION   APP VERSION     DESCRIPTION                                       
stable/nginx-ingress            1.41.1          v0.34.1         An nginx Ingress controller that uses ConfigMap...
stable/nginx-ldapauth-proxy     0.1.4           1.13.5          nginx proxy with ldapauth                         
stable/nginx-lego               0.3.1                           Chart for nginx-ingress-controller and kube-lego  
stable/gcloud-endpoints         0.1.2           1               DEPRECATED Develop, deploy, protect and monitor...
```

간단한 웹서비스 배포를 위해 nginx를 추가해서 구동해 봅니다. 배포도구로 인기 있는 bitnami repo를 추가해서 nginx를 설치해 봅니다.

```text
helm repo add bitnami https://charts.bitnami.com/bitnami
```

다시 nginx를 검색해 봅니다.

```text
helm search repo bitnami/nginx
```

출력결과 예시

```text
whchoi98:~/environment $ helm search repo bitnami/nginx NAME CHART VERSION APP VERSION DESCRIPTION
bitnami/nginx 6.0.2 1.19.1 Chart for the nginx server
bitnami/nginx-ingress-controller 5.3.25 0.33.0 Chart for the nginx Ingress controller
```

helm install 명령을 통해 nginx를 설치해 봅니다.

```text
helm install eksworkshop-nginx bitnami/nginix
```

아래와 같은 결과를 얻을 수 있습니다.

```text
whchoi98:~/environment $ helm install eksworkshop-nginx bitnami/nginx
NAME: eksworkshop-nginx
LAST DEPLOYED: Tue Jul 21 15:01:26 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Get the NGINX URL:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace default -w eksworkshop-nginx'

  export SERVICE_IP=$(kubectl get svc --namespace default eksworkshop-nginx --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
  echo "NGINX URL: http://$SERVICE_IP/"
```

Pod와 서비스 배포를 확인합니다.

```text
whchoi98:~/environment $ kubectl get svc eksworkshop-nginx
NAME                TYPE           CLUSTER-IP       EXTERNAL-IP                                                                   PORT(S)                      AGE
eksworkshop-nginx   LoadBalancer   172.20.225.235   a580840c2d2f24533a7fe36836e99a93-149103174.ap-northeast-2.elb.amazonaws.com   80:31437/TCP,443:32083/TCP   79s
```

ELB 주소를 확인합니다.

```text
export SERVICE_IP=$(kubectl get svc --namespace default eksworkshop-nginx --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
echo "NGINX URL: http://$SERVICE_IP/"
```

해당 주소로 접속해 봅니다.

![](../.gitbook/assets/image%20%2846%29.png)

EC2 대시보드에서 ELB가 정상적으로 생성된 것을 확인 할 수 있습니다.

![](../.gitbook/assets/image%20%2845%29.png)

설치된 helm list를 확인합니다.

```text
helm list 
```

출력 예시

```text
whchoi98:~/environment $ helm list 
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
eksworkshop-nginx       default         1               2020-07-21 15:01:26.441053601 +0000 UTC deployed        nginx-6.0.2     1.19.1     
```

아래 명령을 통해 배포된 내용을 확인합니다.

```text
kubectl describe deployments.apps eksworkshop-nginx
```

출력 결과 예시

```text
whchoi98:~/environment $ kubectl describe deployments.apps eksworkshop-nginx 
Name:                   eksworkshop-nginx
Namespace:              default
CreationTimestamp:      Tue, 21 Jul 2020 15:01:26 +0000
Labels:                 app.kubernetes.io/instance=eksworkshop-nginx
                        app.kubernetes.io/managed-by=Helm
                        app.kubernetes.io/name=nginx
                        helm.sh/chart=nginx-6.0.2
Annotations:            deployment.kubernetes.io/revision: 1
                        meta.helm.sh/release-name: eksworkshop-nginx
                        meta.helm.sh/release-namespace: default
Selector:               app.kubernetes.io/instance=eksworkshop-nginx,app.kubernetes.io/name=nginx
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app.kubernetes.io/instance=eksworkshop-nginx
           app.kubernetes.io/managed-by=Helm
           app.kubernetes.io/name=nginx
           helm.sh/chart=nginx-6.0.2
  Containers:
   nginx:
    Image:        docker.io/bitnami/nginx:1.19.1-debian-10-r0
    Port:         8080/TCP
    Host Port:    0/TCP
    Liveness:     tcp-socket :http delay=30s timeout=5s period=10s #success=1 #failure=6
    Readiness:    tcp-socket :http delay=5s timeout=3s period=5s #success=1 #failure=3
    Environment:  <none>
    Mounts:
      /opt/bitnami/nginx/conf/server_blocks from nginx-server-block-paths (rw)
  Volumes:
   nginx-server-block-paths:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      eksworkshop-nginx-server-block
    Optional:  false
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   eksworkshop-nginx-79bfdcd875 (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  21m   deployment-controller  Scaled up replica set eksworkshop-nginx-79bfdcd875 to 1
```

### 5. Helm을 통한 nginx 삭제

아래 명령을 통해 생성된 Helm을 삭제합니다.

```text
helm uninstall eksworkshop-nginx
```

정상적으로 삭제되었는지 확인합니다.

```text
helm list
kubectl get service eksworkshop-nginx
```

{% hint style="info" %}
ELB 삭제 시간으로 3분 정도 소요됩니다.
{% endhint %}

## Helm을 이용한 Microservice 배포.

### 1.Chart 만들기

helmdemo 라는 Chart를 생성합니다.

```text
cd ~/environment/
helm create eksdemo
```

Helm Chart를 생성하면, 아래와 같은 디렉토리 구조를 생성합니다.

```text
whchoi98:~/environment/eksdemo $ tree 
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

### 2. Chart 사용자 정의

새로운 Chart구성을 위해 기본 생성된 파일들을 삭제합니다.

```text
rm -rf ~/environment/eksdemo/templates/
rm ~/environment/eksdemo/Chart.yaml
rm ~/environment/eksdemo/values.yaml
```

다음 코드 블록을 실행하여 새로운 chart.yaml 파일을 생성합니다.

```text
cat <<EoF > ~/environment/eksdemo/Chart.yaml
apiVersion: v2
name: eksdemo
description: A Helm chart for EKS Workshop Microservices application
version: 0.1.0
appVersion: 1.0
EoF
```

앞서 microservice 어플리케이션 매니페스트 파일들을 servicename.yaml로 템플릿 디렉토리에 복사합니다.

```text
# 각 템플릿 타입을 위한 서브 폴더 생성 
mkdir -p ~/environment/eksdemo/templates/deployment
mkdir -p ~/environment/eksdemo/templates/service

# frontend manifests 복사 
cp ~/environment/ecsdemo-frontend/kubernetes/deployment.yaml ~/environment/eksdemo/templates/deployment/frontend.yaml
cp ~/environment/ecsdemo-frontend/kubernetes/service.yaml ~/environment/eksdemo/templates/service/frontend.yaml

# crystal manifests 복사 
cp ~/environment/ecsdemo-crystal/kubernetes/deployment.yaml ~/environment/eksdemo/templates/deployment/crystal.yaml
cp ~/environment/ecsdemo-crystal/kubernetes/service.yaml ~/environment/eksdemo/templates/service/crystal.yaml

# nodejs manifests 복사 
cp ~/environment/ecsdemo-nodejs/kubernetes/deployment.yaml ~/environment/eksdemo/templates/deployment/nodejs.yaml
cp ~/environment/ecsdemo-nodejs/kubernetes/service.yaml ~/environment/eksdemo/templates/service/nodejs.yaml
```

아래 파일들을 찾아서, "replicas:1" 값을 변경합니다.

변경할 파일들

```text
~/environment/eksdemo/templates/deployment/frontend.yaml
~/environment/eksdemo/templates/deployment/crystal.yaml
~/environment/eksdemo/templates/deployment/nodejs.yaml
```

![](../.gitbook/assets/image%20%2848%29.png)

Cloud9 IDE 편집기를 이용해서 3개 파일의 "replicas:1" 값을 변경하고, 저장합니다.

```text
replicas: {{ .Values.replicas }}
```

추가로 3개 파일의 "image: 'brentley/ecsdemo-xxx' 을 찾아서, 아래와 같이 변경합니다.

~/environment/eksdemo/templates/deployment/frontend.yaml

```text
- image: {{ .Values.frontend.image }}:{{ .Values.version }}
```

~/environment/eksdemo/templates/deployment/crystal.yaml

```text
- image: {{ .Values.crystal.image }}:{{ .Values.version }}
```

~/environment/eksdemo/templates/deployment/nodejs.yaml

```text
- image: {{ .Values.nodejs.image }}:{{ .Values.version }}
```

새로운 namespace에 생성하기 위해서 deployment, service 매니페스트 파일에 namespace를 추가합니다.

변경대상 파일.

```text
~/environment/eksdemo/templates/deployment/frontend.yaml
~/environment/eksdemo/templates/deployment/crystal.yaml
~/environment/eksdemo/templates/deployment/nodejs.yaml
~/environment/eksdemo/templates/service/frontend.yaml
~/environment/eksdemo/templates/service/crystal.yaml
~/environment/eksdemo/templates/service/nodejs.yaml
```

추가 구문

```text
  namespace: helm-chart-demo
```

변경 이후 파일 내용

```text
metadata:
  name: ecsdemo-xxxx
  labels:
    app: ecsdemo-xxxx
  namespace: helm-chart-demo
```

이제 values.yaml을 생성하고, Values 들에 대한 값을 정의합니다.

```text
cat <<EoF > ~/environment/eksdemo/values.yaml
# Default values for eksdemo.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

# Release-wide Values
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

### 3. Chart 배포

Helm chart는 실제 배포하지 않고 **"--dry-run"** 플래그를 사용하여, 랜더링 된 템플릿을 빌드하고 출력할 수 있습니다.

```text
helm install --debug --dry-run workshop ~/environment/eksdemo
```

출력 결과 예제

```text
whchoi98:~/environment $ helm install --debug --dry-run workshop ~/environment/eksdemo
install.go:159: [debug] Original chart version: ""
install.go:176: [debug] CHART PATH: /home/ec2-user/environment/eksdemo

NAME: workshop
LAST DEPLOYED: Tue Jul 21 17:19:19 2020
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
nodejs:
  image: brentley/ecsdemo-nodejs
replicas: 3
version: latest

HOOKS:
MANIFEST:
---
# Source: eksdemo/templates/service/crystal.yaml
apiVersion: v1
kind: Service
metadata:
  name: ecsdemo-crystal
  namespace: helm-chart-demo
spec:
  selector:
    app: ecsdemo-crystal
  ports:
   -  protocol: TCP
      port: 80
      targetPort: 3000
---
# Source: eksdemo/templates/service/frontend.yaml
apiVersion: v1
kind: Service
metadata:
  name: ecsdemo-frontend
  namespace: helm-chart-demo
spec:
  selector:
    app: ecsdemo-frontend
  type: LoadBalancer
  ports:
   -  protocol: TCP
      port: 80
      targetPort: 3000
---
# Source: eksdemo/templates/service/nodejs.yaml
apiVersion: v1
kind: Service
metadata:
  name: ecsdemo-nodejs
  namespace: helm-chart-demo
spec:
  selector:
    app: ecsdemo-nodejs
  ports:
   -  protocol: TCP
      port: 80
      targetPort: 3000
---
# Source: eksdemo/templates/deployment/crystal.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecsdemo-crystal
  labels:
    app: ecsdemo-crystal
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
---
# Source: eksdemo/templates/deployment/frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecsdemo-frontend
  labels:
    app: ecsdemo-frontend
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
            - name: CRYSTAL_URL
              value: 'http://ecsdemo-crystal.default.svc.cluster.local/crystal'
            - name: NODEJS_URL
              value: 'http://ecsdemo-nodejs.default.svc.cluster.local/'
---
# Source: eksdemo/templates/deployment/nodejs.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecsdemo-nodejs
  labels:
    app: ecsdemo-nodejs
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
```

템플릿이 정상적으로 렌더링되고, 빌드 출력이 이뤄졌으므로 차트를 배포합니다.

```text
kubectl create namespace helm-chart-demo
helm install helmdemo ~/environment/eksdemo
```

출력 결과 예시

```text
whchoi98:~/environment $ helm install helmdemo ~/environment/eksdemo
NAME: helmdemo
LAST DEPLOYED: Tue Jul 21 17:27:53 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

정상적으로 배포되었는 지 확인해 봅니다.

```text
kubectl -n helm-chart-demo get svc,pod,deploy -o wide
```

출력 결과 예시

```text
whchoi98:~/environment $ kubectl -n helm-chart-demo get svc,pod,deploy -o wide
NAME                       TYPE           CLUSTER-IP       EXTERNAL-IP                                                                   PORT(S)        AGE     SELECTOR
service/ecsdemo-crystal    ClusterIP      172.20.167.203   <none>                                                                        80/TCP         2m44s   app=ecsdemo-crystal
service/ecsdemo-frontend   LoadBalancer   172.20.195.10    a6ad6b49efa9a426d86ce1779cee31f6-759353285.ap-northeast-2.elb.amazonaws.com   80:31738/TCP   2m44s   app=ecsdemo-frontend
service/ecsdemo-nodejs     ClusterIP      172.20.44.202    <none>                                                                        80/TCP         2m45s   app=ecsdemo-nodejs

NAME                                    READY   STATUS    RESTARTS   AGE     IP              NODE                                               NOMINATED NODE   READINESS GATES
pod/ecsdemo-crystal-6d5f6f4b47-78lcr    1/1     Running   0          2m44s   10.11.148.146   ip-10-11-146-170.ap-northeast-2.compute.internal   <none>           <none>
pod/ecsdemo-crystal-6d5f6f4b47-9dn47    1/1     Running   0          2m44s   10.11.120.83    ip-10-11-114-132.ap-northeast-2.compute.internal   <none>           <none>
pod/ecsdemo-crystal-6d5f6f4b47-k26nr    1/1     Running   0          2m44s   10.11.165.194   ip-10-11-189-67.ap-northeast-2.compute.internal    <none>           <none>
pod/ecsdemo-frontend-7f5fddd684-7mlg2   1/1     Running   0          2m44s   10.11.72.249    ip-10-11-69-28.ap-northeast-2.compute.internal     <none>           <none>
pod/ecsdemo-frontend-7f5fddd684-dcvzf   1/1     Running   0          2m44s   10.11.97.62     ip-10-11-114-132.ap-northeast-2.compute.internal   <none>           <none>
pod/ecsdemo-frontend-7f5fddd684-s9nfh   1/1     Running   0          2m44s   10.11.56.6      ip-10-11-53-186.ap-northeast-2.compute.internal    <none>           <none>
pod/ecsdemo-nodejs-7dd8987798-7rx6g     1/1     Running   0          2m44s   10.11.99.190    ip-10-11-114-132.ap-northeast-2.compute.internal   <none>           <none>
pod/ecsdemo-nodejs-7dd8987798-9cdxv     1/1     Running   0          2m44s   10.11.183.26    ip-10-11-189-67.ap-northeast-2.compute.internal    <none>           <none>
pod/ecsdemo-nodejs-7dd8987798-bn9dd     1/1     Running   0          2m44s   10.11.136.126   ip-10-11-146-170.ap-northeast-2.compute.internal   <none>           <none>

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS         IMAGES                             SELECTOR
deployment.apps/ecsdemo-crystal    3/3     3            3           2m44s   ecsdemo-crystal    brentley/ecsdemo-crystal:latest    app=ecsdemo-crystal
deployment.apps/ecsdemo-frontend   3/3     3            3           2m44s   ecsdemo-frontend   brentley/ecsdemo-frontend:latest   app=ecsdemo-frontend
deployment.apps/ecsdemo-nodejs     3/3     3            3           2m44s   ecsdemo-nodejs     brentley/ecsdemo-nodejs:latest     app=ecsdemo-nodejs
```

### 4. 서비스 확인

아래 명령을 통해 service loadbalancer\(ELB\) 의 DNS name을 확인합니다.

```text
kubectl -n helm-chart-demo get svc ecsdemo-frontend
```

출력 결과 예시

```text
whchoi98:~/environment $ kubectl -n helm-chart-demo get svc ecsdemo-frontend
NAME               TYPE           CLUSTER-IP      EXTERNAL-IP                                                                   PORT(S)        AGE
ecsdemo-frontend   LoadBalancer   172.20.195.10   a6ad6b49efa9a426d86ce1779cee31f6-759353285.ap-northeast-2.elb.amazonaws.com   80:31738/TCP   6m3s
```

정상적으로 서비스에 접속되는 것을 확인 할 수 있습니다.

![](../.gitbook/assets/image%20%2849%29.png)

helm list에도 정상적으로 등록되어 있는지 확인합니다.

```text
helm list 
```

출력 결과 예

```text
whchoi98:~/environment $ helm list 
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
helmdemo        default         1               2020-07-21 17:27:53.828241377 +0000 UTC deployed        eksdemo-0.1.0   1          
```

### 5. Rolling Back 구성

배포 중에 이슈가 발생하더라도, Helm을 사용하면 손쉽게 이전 배포 버전으로 Rolling back 할 수 있습니다. 앞서 출력결과에서 "Revision" 값이 "1"인 것을 확인했습니다. 이제 helm Chart를 변경하고 업데이트 한 이후 , 다시 Rollback 하는 과정을 수행하겠습니다.

먼저 helm Chart의 Value를 변경합니다. values.yaml을 다시 열고, Cloud9 IDE 편집기에서 nodejs.image 값을 아래와 같이 수정합니다.

```text
brentley / ecsdemo-nodejs-non-existing
```

  
  


