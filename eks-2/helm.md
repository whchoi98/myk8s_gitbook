# Helm

## Helm 소개

헬름은 [쿠버네티스](https://kubernetes.io/)용 소프트웨어를 검색하거나, 공유하고 사용하기에 가장 좋은 방법입니다. 쿠버네티스 차트를 관리하기 위한 도구이며, 차트는 사전 구성된 쿠버네티스 리소스의 패키지입니다.

주요 3가지의 개념이 있습니다.

**차트**는 헬름 패키지입니. 이 패키지에는 쿠버네티스 클러스터 내에서 애플리게이션, 도구, 서비스를 구동하는데 필요한 모들 리소스 정의가 포함되어 있습니다. 쿠버네티스에서의 Homebrew 포뮬러, Apt dpkg, YUM RPM 파일과 같은 것으로 생각할 수 있습니다.

**저장소**는 차트를 모아두고 공유하는 장소입다. 이것은 마치 Perl의 [CPAN 아카이브](https://www.cpan.org/)나 [페도라 패키지 데이터베이스](https://admin.fedoraproject.org/pkgdb/)와 같은데, 쿠버네티스 패키지용이라고 보면 됩니다.

**릴리스**는 쿠버네티스 클러스터에서 구동되는 차트의 인스턴스입다. 일반적으로 하나의 차트는 동일한 클러스터내에 여러 번 설치될 수 있다. 설치될 때마다, 새로운 _release_ 가 생성됩니다.

헬름은 쿠버네티스 내부에 \_charts\_를 설치하고, 각 설치에 대해 새로운 \_release\_를 생성하고, 새로운 차트를 찾기 위해 헬름 차트 \_repositories\_를 검색할 수 있습니다.

## Helm 설치와 간단한 배포

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

### 4. Helm Chart를 통한 간단한 웹 배포

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

## Helm을 이용한 Microservice 배포.



  
  


