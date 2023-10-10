---
description: 'Update : 2023-09-21'
---

# Argo 기반 CD

## Argo CD Overview

애플리케이션 정의, 구성 및 환경은 선언적이고 버전이 제어되어야 합니다. 애플리케이션 배포 및 수명 주기 관리는 자동화되고 감사 가능하며 이해하기 쉬워야 합니다.Argo CD 는 원하는 애플리케이션 상태를 정의하기 위한 정보 소스로 Git 리포지토리를 사용 하는 **GitOps 패턴을 따릅니다.** Kubernetes 매니페스트는 여러 가지 방법으로 지정할 수 있습니다.

* [kustomize](https://kustomize.io/) applications
* [helm](https://helm.sh/) charts
* [jsonnet](https://jsonnet.org/) files
* Plain directory of YAML/json manifests
* 구성 관리 플러그인으로 구성된 모든 맞춤형 구성 관리 도구

Argo CD는 지정된 대상 환경에서 원하는 애플리케이션 상태의 배포를 자동화합니다. 애플리케이션 배포는 분기, 태그에 대한 업데이트를 추적하거나 Git 커밋에서 매니페스트의 특정 버전에 고정할 수 있습니다.



Argo CD는 아래와 같은 아키텍쳐로 구성됩니다.

<figure><img src="../.gitbook/assets/image (125).png" alt=""><figcaption></figcaption></figure>

Argo CD는 실행 중인 애플리케이션을 지속적으로 모니터링하고 현재 원하는 대상 상태(Git 저장소에 지정된 대로)와 비교하는 kubernetes 컨트롤러와 같은 형태로 구현됩니다. Argo CD는 라이브 상태를 원하는 대상 상태로 다시 자동 또는 수동으로 동기화하는 기능을 제공하면서 , 현재 상태와 원하는 상태의 차이점을 리포팅하고 시각화합니다. Git 리포지토리에서 원하는 대상 상태에 대한 모든 수정 사항은 지정된 대상 환경에 자동으로 적용되고 반영될 수 있습니다.



Argo CD는 **3가지 컴포넌트**로 구성되어 있습니다.

* 실행 중인 애플리케이션을 **지속적으로 모니터링**
* **현재 라이브 상태를 원하는 대상 상태(Git 저장소에 지정된 대로)와 주기적으로 비교**
  * 라이브 상태가 대상 상태와 다른 배포된 애플리케이션은 **OutOfSync**로 처리
* 차이점을 리포팅 및 **UI 기반으로 시각화**
* 라이브 상태를 원하는 대상 상태로 자동 또는 수동으로 다시 동기화 하는 기능을 제공
* **API Server**
  * API 서버는 Web UI, CLI 및 외부에서 사용할 수 있도록 gRPC와 REST API를 제공
* **Repository Server**
  * Application manifest 파일을 가지고 있는 Git 저장소의 로컬 캐시를 유지 관리
  * Git 저장소에 저장된 manifest 파일을 생성하는 역할
* **Application Controller**
  * 애플리케이션 컨트롤러는 실행중인 응용 프로그램을 지속적으로 모니터링
  * live state와 target state를 비교해서 UI로 리포팅 (OutOfSync)



## Argo CD 설치

### 1.Argo CD 설치

Argo 프로젝트에서 제공하는 매니페스트를 사용하여 설치를 합니다.

```
#namespace - argocd
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.7/manifests/install.yaml

```

### 2.Argo CD CLI 설치

Argo CD CLI를 설치합니다. API 서버와 Interactive하게 운영하려면 CLI를 배포해야 합니다.

```
sudo curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.7/argocd-linux-amd64
sudo chmod +x /usr/local/bin/argocd

```

## ArgoCD 구성



Argo CD가 배포되었으므로 이제 argocd-server를 구성한 다음 로그인해야 합니다.

### 3.Argo CD Expose

기본적으로 argocd-server는 외부에 노출되지 않습니다. 여기에서는 편의상 로드 밸런서를 사용하여 외부에 노출 시킵니다. LoadBalancer가 생성될 때까지 2\~3분 정도 소요됩니다.

```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

```

아래 명령을 통해서 Argo CD 서버의 주소를 변수에 저장해 둡니다.

```
export ARGOCD_SERVER=`kubectl get svc argocd-server -n argocd -o json | jq --raw-output '.status.loadBalancer.ingress[0].hostname'`
echo $ARGOCD_SERVER

```

4.Argo CD 로그인

초기 로그인 암호는 ArgoCD API 서버의 포드 이름으로 자동 생성됩니다.

```
export ARGO_PWD=`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`
echo $ARGO_PWD

```

Admin이 로그인 id 이고, 자동 생성된 암호를 이용해서 로그인 합니다.

```
argocd login $ARGOCD_SERVER --username admin --password $ARGO_PWD --insecure

```

아래와 같이 정상적으로 콘솔에서 로그인 된 것을 확인합니다.

```
$ argocd login $ARGOCD_SERVER --username admin --password $ARGO_PWD --insecure
'admin:login' logged in successfully
Context 'a81830f78562145b3bca329b3a7699cb-1011047444.ap-northeast-2.elb.amazonaws.com' updated

```

## Application 배포

이제 ArgoCD가 완전히 배포되었으므로 , 간단한 애플리케이션(ecsdemo-nodejs)을 배포할 것입니다.

### 4.Application Fork

아래 Repo에서 사용자의 Github으로 Application Fork를 수행합니다.

```
https://github.com/brentley/ecsdemo-nodejs.git 
```

<figure><img src="../.gitbook/assets/image (112).png" alt=""><figcaption></figcaption></figure>

사용자의 Github에 Fork된 Repo로 이동하고, Repo를 복사해 둡니다.

<figure><img src="../.gitbook/assets/image (336).png" alt=""><figcaption></figcaption></figure>

복사한 URL은 아래와 같은 형식입니다.

```
https://github.com/{github-userid}/ecsdemo-nodejs.git
```

이 URL은 Application을 ArgoCD로 구성할 때 필요합니다.

5.Application 생성

클러스터 컨텍스트를 사용하여 ArgoCD CLI와 연결합니다.

```
CONTEXT_NAME=`kubectl config view -o jsonpath='{.current-context}'`
argocd cluster add $CONTEXT_NAME

```

아래와 같이 출력됩니다.

```
$ argocd cluster add $CONTEXT_NAME
WARNING: This will create a service account `argocd-manager` on the cluster referenced by context `i-0fd091734070b9f6f@eksworkshop.ap-northeast-2.eksctl.io` with full cluster level privileges. Do you want to continue [y/N]? y
INFO[0003] ServiceAccount "argocd-manager" created in namespace "kube-system" 
INFO[0003] ClusterRole "argocd-manager-role" created    
INFO[0003] ClusterRoleBinding "argocd-manager-role-binding" created 
Cluster 'https://095783B7DF46A1EA53103B1201579207.gr7.ap-northeast-2.eks.amazonaws.com' added
```

애플리케이션을 구성하고 사용자의 Fork에 연결합니다.

```
kubectl create namespace ecsdemo-nodejs
argocd app create ecsdemo-nodejs --repo https://github.com/{GITHUB_USERNAME}/ecsdemo-nodejs.git --path kubernetes --dest-server https://kubernetes.default.svc --dest-namespace ecsdemo-nodejs

```

이제 애플리케이션이 설정되었으므로 배포된 애플리케이션 상태를 argo cd cli로 확인합니다.

```
argocd app get ecsdemo-nodejs

```

아래와 같이 출력됩니다.

```
$ argocd app get ecsdemo-nodejs
Name:               ecsdemo-nodejs
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          ecsdemo-nodejs
URL:                https://a81830f78562145b3bca329b3a7699cb-1011047444.ap-northeast-2.elb.amazonaws.com/applications/ecsdemo-nodejs
Repo:               https://github.com/whchoi98/ecsdemo-nodejs.git
Target:             
Path:               kubernetes
SyncWindow:         Sync Allowed
Sync Policy:        <none>
Sync Status:        OutOfSync from  (c63f795)
Health Status:      Missing

GROUP  KIND        NAMESPACE       NAME            STATUS     HEALTH   HOOK  MESSAGE
       Service     ecsdemo-nodejs  ecsdemo-nodejs  OutOfSync  Missing        
apps   Deployment  default         ecsdemo-nodejs  OutOfSync  Missing        

```

UI에서도 동일한 값을 확인 할 수 있습니다.

<figure><img src="../.gitbook/assets/image (119).png" alt=""><figcaption></figcaption></figure>

애플리케이션이 아직 배포되지 않았기 때문에 애플리케이션이 OutOfSync 상태에 있음을 알 수 있습니다. 이제 애플리케이션을 동기화합니다.

```
argocd app sync ecsdemo-nodejs

```

2\~3분 이후 Synced 된 상태와 Heathy 상태로 전환된 것을 확인 할 수 있습니다.

```
$ argocd app sync ecsdemo-nodejs
TIMESTAMP                  GROUP        KIND   NAMESPACE                      NAME    STATUS   HEALTH        HOOK  MESSAGE
2023-01-31T00:05:33+00:00            Service  ecsdemo-nodejs        ecsdemo-nodejs    Synced  Healthy              
2023-01-31T00:05:33+00:00   apps  Deployment     default            ecsdemo-nodejs    Synced  Healthy              
2023-01-31T00:05:34+00:00            Service  ecsdemo-nodejs        ecsdemo-nodejs    Synced  Healthy              service/ecsdemo-nodejs unchanged
2023-01-31T00:05:34+00:00   apps  Deployment     default            ecsdemo-nodejs    Synced  Healthy              deployment.apps/ecsdemo-nodejs unchanged

Name:               ecsdemo-nodejs
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          ecsdemo-nodejs
URL:                https://a81830f78562145b3bca329b3a7699cb-1011047444.ap-northeast-2.elb.amazonaws.com/applications/ecsdemo-nodejs
Repo:               https://github.com/whchoi98/ecsdemo-nodejs.git
Target:             
Path:               kubernetes
SyncWindow:         Sync Allowed
Sync Policy:        <none>
Sync Status:        Synced to  (c63f795)
Health Status:      Healthy

Operation:          Sync
Sync Revision:      c63f79569c15ac032241ea98479801f061e75aac
Phase:              Succeeded
Start:              2023-01-31 00:05:33 +0000 UTC
Finished:           2023-01-31 00:05:33 +0000 UTC
Duration:           0s
Message:            successfully synced (all tasks run)

GROUP  KIND        NAMESPACE       NAME            STATUS  HEALTH   HOOK  MESSAGE
       Service     ecsdemo-nodejs  ecsdemo-nodejs  Synced  Healthy        service/ecsdemo-nodejs unchanged
apps   Deployment  default         ecsdemo-nodejs  Synced  Healthy        deployment.apps/ecsdemo-nodejs unchanged
```

<figure><img src="../.gitbook/assets/image (110).png" alt=""><figcaption></figcaption></figure>

## Application Update&#x20;

Application이 ArgoCD에 배포되었습니다. 이제 애플리케이션과 동기화된 github 저장소를 업데이트 해 봅니다.



### 5.Application Update

사용자의 Github 포크 저장소로 이동해서 , Replica를 변경해 봅니다.

```
https://github.com/{github-userid}/ecsdemo-nodejs/blob/master/kubernetes/deployment.yaml
```

<figure><img src="../.gitbook/assets/image (326).png" alt=""><figcaption></figcaption></figure>

변경후 _**`Commit Chaged`**_ 를 선택합니다.

이제 다시 Argo CD에서 상태를 확인해 봅니다.

<figure><img src="../.gitbook/assets/image (88).png" alt=""><figcaption></figcaption></figure>

Argo CD의 UI에서 주요 메뉴들을 선택해서 실제 어떻게 Report 되는지를 살펴봅니다. \
