---
description: 'update : 2021-10-18 / 2h'
---

# Loadbalancer 기반 배포

## Loadbalancer 서비스 타입 소개.

Loadbalancer 기반의 서비스 타입은 현재 CLB (Classic Load Balancer)와 NLB(Network Load Balancer) 2가지 타입을 지원하고 있습니다. 모두 Port 기반의 LB를 제공하고 있으며, Kubernetes 의 Node와 Service 전면에서 서비스를 제공합니다.

## CLB Loadbalancer 서비스 기반 구성

### 1.CLB 기반 ServiceType

Service Type 필드를 LoadBalancer로 설정하여 프로브저닝합니다. Cloud Service Provider의 기본 로드밸런서 타입을 사용하게 되며, AWS의 경우에는 CLB를 사용합니다. CLB는 내부 또는 외부 로드밸런서로 지정이 가능합니다.

![](<../.gitbook/assets/image (221) (1).png>)

### 2. CLB Service Type 트래픽 흐름

Traffic 흐름은 다음과 같습니다.

* 외부 사용자는 CLB DNS A Record:Port로 접근
* CLB는 NodePort로 LB 처리 (NodePort는 임의로 할당 됩니다.)
* NodePort는 ClusterIP로 Forwarding되고 IPTable에 의해 분산 처리 됩니다.

![](<../.gitbook/assets/image (222) (1).png>)

### 3. CLB Service 시험

CLB Loadbalance Service Type 을 시험하기 위해 아래와 같이 namespace와  pod,service를 배포합니다

```
## clb-test-01 namespace를 생성하고, pod, service를 배포 
kubectl create namespace clb-test-01
kubectl -n clb-test-01 apply -f ~/environment/myeks/network-test/clb-test-01.yaml
kubectl -n clb-test-01 apply -f ~/environment/myeks/network-test/clb-test-01-service.yaml

```

정상적으로 배포되었는지 아래 Command로 확인합니다

```
## clb-test-01 namespace의 pod 확인 
kubectl -n clb-test-01 get pod -o wide

## clb-test-01 namespace의 service 확인
kubectl -n clb-test-01 get service -o wide

```

다음과 같은 결과를 얻을 수 있습니다

```
$ kubectl -n clb-test-01 get pod -o wide
NAME                           READY   STATUS    RESTARTS   AGE     IP             NODE                                              NOMINATED NODE   READINESS GATES
clb-test-01-66f4b975ff-4ljrm   1/1     Running   0          2m55s   10.11.45.100   ip-10-11-35-116.ap-northeast-2.compute.internal   <none>           <none>
clb-test-01-66f4b975ff-ldttb   1/1     Running   0          2m55s   10.11.29.252   ip-10-11-21-111.ap-northeast-2.compute.internal   <none>           <none>
clb-test-01-66f4b975ff-xb2rs   1/1     Running   0          2m55s   10.11.7.225    ip-10-11-3-68.ap-northeast-2.compute.internal     <none>           <none>
$ kubectl -n clb-test-01 get service -o wide
NAME              TYPE           CLUSTER-IP       EXTERNAL-IP                                                                  PORT(S)          AGE    SELECTOR
clb-test-01-svc   LoadBalancer   172.20.214.188   a8505cbace07d4e15842c4403e55f27b-54739774.ap-northeast-2.elb.amazonaws.com   8080:32139/TCP   3m8s   app=clb-test-01
```

아래와 같이 구성됩니다 . nodeport는 별도의 지정이 없으면 생성할때 자동으로 지정됩니다

![](<../.gitbook/assets/image (217).png>)

아래와 같이 배포된 pod에 접속을 편리하게 하기 위해 Cloud9 IDE terminal Shell에 등록 합니다.

```
echo "export ClbTestPod03=clb-test-01-66f4b975ff-4ljrm" | tee -a ~/.bash_profile
echo "export ClbTestPod02=clb-test-01-66f4b975ff-ldttb" | tee -a ~/.bash_profile
echo "export ClbTestPod01=clb-test-01-66f4b975ff-xb2rs" | tee -a ~/.bash_profile
source ~/.bash_profile
```

ClbTestPod01에 접속해서 아래와 같이 확인해 봅니다

```
kubectl -n clb-test-01 exec -it $ClbTestPod01 -- /bin/sh
nslookup {cluster-ip}
tcpdump -i eth0 dst port 80

```

Cloud9 IDE Terminal에서 CLB External IP:8080 으로 접속합니다

```
kubectl -n clb-test-01 get service
NAME              TYPE           CLUSTER-IP       EXTERNAL-IP                                                                  PORT(S)          AGE
clb-test-01-svc   LoadBalancer   172.20.214.188   a8505cbace07d4e15842c4403e55f27b-54739774.ap-northeast-2.elb.amazonaws.com   8080:32139/TCP   3h16m

curl a8505cbace07d4e15842c4403e55f27b-54739774.ap-northeast-2.elb.amazonaws.com:8080
```

Node에서 iptable에 설정된 NAT Table, Loadbalancing 구성을 확인해 봅니다.

```

aws ssm start-session --target $NGPublic01
sudo -s
iptables -t nat -L --line-number | more
```

CLB에서는 아래와 같은 다양한 Annotation을 추가하여 CLB의 속성 또는 AWS 자원을 연결해서 사용할 수 있습니다

```
metadata:
      name: my-service
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "60"
        # 로드 밸런서가 연결을 닫기 전에, 유휴 상태(연결을 통해 전송 된 데이터가 없음)의 연결을 허용하는 초단위 시간

        service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
        # 로드 밸런서에 교차-영역(cross-zone) 로드 밸런싱을 사용할 지 여부를 지정

        service.beta.kubernetes.io/aws-load-balancer-additional-resource-tags: "environment=prod,owner=devops"
        # 쉼표로 구분된 key-value 목록은 ELB에
        # 추가 태그로 기록됨

        service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold: ""
        # 백엔드가 정상인 것으로 간주되는데 필요한 연속적인
        # 헬스 체크 성공 횟수이다. 기본값은 2이며, 2와 10 사이여야 한다.

        service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold: "3"
        # 백엔드가 비정상인 것으로 간주되는데 필요한
        # 헬스 체크 실패 횟수이다. 기본값은 6이며, 2와 10 사이여야 한다.

        service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "20"
        # 개별 인스턴스의 상태 점검 사이의
        # 대략적인 간격 (초 단위). 기본값은 10이며, 5와 300 사이여야 한다.

        service.beta.kubernetes.io/aws-load-balancer-healthcheck-timeout: "5"
        # 헬스 체크 실패를 의미하는 무 응답의 총 시간 (초 단위)
        # 이 값은 service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval
        # 값 보다 작아야한다. 기본값은 5이며, 2와 60 사이여야 한다.

        service.beta.kubernetes.io/aws-load-balancer-security-groups: "sg-53fae93f"
        # 생성된 ELB에 설정할 기존 보안 그룹(security group) 목록.
        # service.beta.kubernetes.io/aws-load-balancer-extra-security-groups 어노테이션과 달리, 이는 이전에 ELB에 할당된 다른 모든 보안 그룹을 대체하며,
        # '해당 ELB를 위한 고유 보안 그룹 생성'을 오버라이드한다.
        # 목록의 첫 번째 보안 그룹 ID는 인바운드 트래픽(서비스 트래픽과 헬스 체크)이 워커 노드로 향하도록 하는 규칙으로 사용된다.
        # 여러 ELB가 하나의 보안 그룹 ID와 연결되면, 1줄의 허가 규칙만이 워커 노드 보안 그룹에 추가된다.
        # 즉, 만약 여러 ELB 중 하나를 지우면, 1줄의 허가 규칙이 삭제되어, 같은 보안 그룹 ID와 연결된 모든 ELB에 대한 접속이 막힌다.
        # 적절하게 사용되지 않으면 이는 다수의 서비스가 중단되는 상황을 유발할 수 있다.

        service.beta.kubernetes.io/aws-load-balancer-extra-security-groups: "sg-53fae93f,sg-42efd82e"
        # 생성된 ELB에 추가할 추가 보안 그룹 목록
        # 이 방법을 사용하면 이전에 생성된 고유 보안 그룹이 그대로 유지되므로, 각 ELB가 고유 보안 그룹 ID와 그에 매칭되는 허가 규칙 라인을 소유하여
        # 트래픽(서비스 트래픽과 헬스 체크)이 워커 노드로 향할 수 있도록 한다. 여기에 기재되는 보안 그룹은 여러 서비스 간 공유될 수 있다.

        service.beta.kubernetes.io/aws-load-balancer-target-node-labels: "ingress-gw,gw-name=public-api"
        # 로드 밸런서의 대상 노드를 선택하는 데
        # 사용되는 키-값 쌍의 쉼표로 구분된 목록
```

## CLB Application 배포

다음과 같은 구성을 통해서 CLB 서비스를 구현해 봅니다.&#x20;

![](<../.gitbook/assets/image (221).png>)

* namespace : clb-test
* eksdemo-frontend service type : LoadbBlancer
* eksdemo-crystal service type: Cluster-IP&#x20;
* eksdemo-nodejs service type: Cluster-IP&#x20;

### 4. FrontEnd 어플리케이션 배포와 서비스 구성.

기본 Loadbalacer 구성을 위해 새로운 Namespace를 생성합니다.

```
kubectl create namespace clb-test

```

어플리케이션을 배포하고, service를 구성합니다.

```
#eksdemo frontend clb depolyment apply
kubectl -n clb-test apply -f ~/environment/eksdemo-frontend/kubernetes/clb_deployment.yaml
#eksdemo frontend clb service apply
kubectl -n clb-test apply -f ~/environment/eksdemo-frontend/kubernetes/clb_service.yaml

```

정상적으로 Pod가 배포되었는지 아래 명령을 통해서 확인해 봅니다.

```
kubectl -n clb-test get deployments ecsdemo-frontend -o wide
kubectl -n clb-test get service ecsdemo-frontend -o wide 

```

Replica를 3개로 늘려서 LB가 FrontEnd에서 정상적으로 이뤄지는 지 확인합니다.

```
kubectl -n clb-test scale deployment ecsdemo-frontend --replicas=3
kubectl -n clb-test get pod -o wide
```

아래 출력되는 결과의 EXTERNAL-IP를 복사해서 브라우져 창에서 실행해 봅니다.

```
kubectl -n clb-test get service -o wide                                                           
NAME               TYPE           CLUSTER-IP     EXTERNAL-IP                                                                   PORT(S)        AGE     SELECTOR
ecsdemo-frontend   LoadBalancer   172.20.37.78   afd75bf8c69c04c3aacf6cfbdefe1c4f-884593752.ap-northeast-2.elb.amazonaws.com   80:31380/TCP   5m45s   app=ecsdemo-frontend
```

출력결과 예시

![](<../.gitbook/assets/image (151).png>)

앞서 설치해 둔 K9s 유틸리티를 통해서 , 현재 배포된 Pod들의 상태를 확인해 봅니다.

```
k9s -A

```

### 5. BackEnd 어플리케이션 배포

Backend 어플리케이션 Nodejs와 Crystal을 배포합니다. 이 2개의 어플리케이션들은 Private Subnet에 배포할 것입니다. 이 구성은 앞서 이미 Yaml 파일의 Deployment에서 nodeSelector로 지정하였습니다.

```
#eksdemo nodejs clb depolyment apply
#eksdemo nodejs clb service apply
kubectl -n clb-test apply -f ~/environment/eksdemo-crystal/kubernetes/clb_deployment.yaml
kubectl -n clb-test apply -f ~/environment/eksdemo-crystal/kubernetes/clb_service.yaml

#eksdemo crystal clb depolyment apply
#ecsdemo crystal clb service apply
kubectl -n clb-test apply -f ~/environment/eksdemo-nodejs/kubernetes/clb_deployment.yaml
kubectl -n clb-test apply -f ~/environment/eksdemo-nodejs/kubernetes/clb_service.yaml

```

정상적으로 Pod가 배포되었는지 아래 명령을 통해서 확인해 봅니다.

```
kubectl -n clb-test get deployments ecsdemo-nodejs -o wide
kubectl -n clb-test get service ecsdemo-nodejs -o wide 
kubectl -n clb-test get deployments ecsdemo-crystal -o wide
kubectl -n clb-test get service ecsdemo-crystal -o wide 

```

Replica를 3개로 늘려서 Service Type이 없는 경우, BackEnd에서 정상적으로 이뤄지는 지 확인합니다.

```
kubectl -n clb-test scale deployment ecsdemo-nodejs --replicas=3
kubectl -n clb-test scale deployment ecsdemo-crystal --replicas=3

```

![](<../.gitbook/assets/image (153).png>)

k9s 를 통해 Pod의 구성을 확인합니다.

{% hint style="info" %}
LAB 을 진행하면서, Pod의 배포 상황을 계속 모니터링하기 위해서 Cloud9 에서 Terminal을 하나 더 열고 K9s를 실행 시켜 두는 것이 좋습니다.
{% endhint %}

![](<../.gitbook/assets/image (155).png>)

### 6. Loadbalancer 확인.

이제 서비스 타입을 확인하기 위해서 EC2 대시보드에서 Loadbalancer를 확인합니다.

CLB의 DNS Name을 복사해서 Web Browser에서 입력합니다.

![](<../.gitbook/assets/image (147).png>)

{% hint style="info" %}
service 매니페스트에서 Service Type을 LoadBalancer로 지정하면, Default로 Classic LB가 구성됩니다. 또한 별도로 Service Type을 지정하지 않으면, ClusterIP로 지정됩니다.
{% endhint %}

아래 kubectl 명령을 통해 service type을 확인해 봅니다.

```
 kubectl -n clb-test get service -o wide
```

```
whchoi98:~/environment $ kubectl -n clb-test get service -o wide
NAME               TYPE           CLUSTER-IP       EXTERNAL-IP                                                                  PORT(S)        AGE    SELECTOR
ecsdemo-crystal    ClusterIP      172.20.180.230   <none>                                                                       80/TCP         27m    app=ecsdemo-crystal
ecsdemo-frontend   LoadBalancer   172.20.213.219   a6531bc45d323472d869946b9bfac449-46256153.ap-northeast-2.elb.amazonaws.com   80:31699/TCP   108m   app=ecsdemo-frontend
ecsdemo-nodejs     ClusterIP      172.20.181.252   <none>                                                                       80/TCP         22m    app=ecsdemo-nodejs
```

## NLB Loadbalancer 서비스 기반 구성

### 7.NLB 기반 Service Type

Service Type 필드를 LoadBalancer로 설정하여 프로브저닝합니다. CLB와 다르게 반드시 annotation을 통해 NLB를 지정해야 합니다. NLB도 내부 또는 외부 로드밸런서로 지정이 가능합니다. 또한 NLB는 외부의 IP를 PoD까지 그대로 전달 할 수 있습니다

* NLB Annotation&#x20;

```
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "instance"
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
```

### 8.NLB Service Type 트래픽 흐름

Traffic 흐름은 다음과 같습니다.

* 외부 사용자는 NLB DNS A Record:Port로 접근합니다.&#x20;
* NLB는 NodePort로 LB 처리 (NodePort는 임의로 할당 됩니다.)
* NodePort는 ClusterIP로 Forwarding되고 IPTable에 의해 분산 처리 됩니다.

![](<../.gitbook/assets/image (225).png>)

### 9. NLB Service 시험

NLB Loadbalance Service Type 을 시험하기 위해 아래와 같이 namespace와  pod,service를 배포합니다.

```
## nlb-test-01 namespace를 생성하고, pod, service를 배포
kubectl create namespace nlb-test-01
kubectl -n nlb-test-01 apply -f ~/environment/myeks/network-test/nlb-test-01.yaml
kubectl -n nlb-test-01 apply -f ~/environment/myeks/network-test/nlb-test-01-service.yaml

```

정상적으로 배포되었는지 아래 Command로 확인합니다.&#x20;

```
kubectl -n nlb-test-01 get pod -o wide
kubectl -n nlb-test-01 get service -o wide
```

다음과 같은 결과를 얻을 수 있습니다.&#x20;

```
kubectl -n nlb-test-01 get pod -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP             NODE                                              NOMINATED NODE   READINESS GATES
nlb-test-01-7c5cf9bd5-chpdb   1/1     Running   0          50s   10.11.10.19    ip-10-11-3-68.ap-northeast-2.compute.internal     <none>           <none>
nlb-test-01-7c5cf9bd5-dfdjm   1/1     Running   0          50s   10.11.32.190   ip-10-11-35-116.ap-northeast-2.compute.internal   <none>           <none>
nlb-test-01-7c5cf9bd5-zp494   1/1     Running   0          50s   10.11.20.119   ip-10-11-21-111.ap-northeast-2.compute.internal   <none>           <none>

kubectl -n nlb-test-01 get service -o wide
NAME              TYPE           CLUSTER-IP       EXTERNAL-IP                                                                          PORT(S)          AGE   SELECTOR
nlb-test-01-svc   LoadBalancer   172.20.167.205   aee3bccea7e554a008b3257942202ee1-89daeb3e502cc5fc.elb.ap-northeast-2.amazonaws.com   8080:30360/TCP   19s   app=nlb-test-01
```

아래와 같이 구성됩니다 . nodeport는 별도의 지정이 없으면 생성할때 자동으로 지정됩니다.

![](<../.gitbook/assets/image (223).png>)

아래와 같이 배포된 pod에 접속을 편리하게 하기 위해 Cloud9 IDE terminal Shell에 등록 합니다.

```
echo "export NlbTestPod03=nlb-test-01-7c5cf9bd5-dfdjm" | tee -a ~/.bash_profile
echo "export NlbTestPod02=nlb-test-01-7c5cf9bd5-zp494" | tee -a ~/.bash_profile
echo "export NlbTestPod01=nlb-test-01-7c5cf9bd5-chpdb" | tee -a ~/.bash_profile
source ~/.bash_profile
```

NlbTestPod01에 접속해서 아래와 같이 확인해 봅니다.&#x20;

```
kubectl -n clb-test-01 exec -it $ClbTestPod01 -- /bin/sh
nslookup {cluster-ip}
tcpdump -i eth0 dst port 80
```

Cloud9 IDE Terminal에서 CLB External IP:8080 으로 접속합니다.&#x20;

```
$ kubectl -n nlb-test-01 get service -o wide
NAME              TYPE           CLUSTER-IP       EXTERNAL-IP                                                                          PORT(S)          AGE   SELECTOR
nlb-test-01-svc   LoadBalancer   172.20.167.205   aee3bccea7e554a008b3257942202ee1-89daeb3e502cc5fc.elb.ap-northeast-2.amazonaws.com   8080:30360/TCP   19s   app=nlb-test-01

curl aee3bccea7e554a008b3257942202ee1-89daeb3e502cc5fc.elb.ap-northeast-2.amazonaws.com:8080
```

Node에서 iptable에 설정된 NAT Table, Loadbalancing 구성을 확인해 봅니다.

```
aws ssm start-session --target $NGPublic01
sudo -s
iptables -t nat -L --line-number | more

```

NLB Service를 삭제하고, 새롭게 배포해 봅니다. nlb-test-01-service.yaml 파일에서 "externalTrafficPolicy: Local"을 활성화 해 봅니다

```
## nlb-test-01-service 삭제
kubectl -n nlb-test-01 delete -f ~/environment/myeks/network-test/nlb-test-01-service.yaml

##  ~/environment/myeks/network-test/nlb-test-01-service.yaml 파일에서 아래 line의 주석처리를 제거 합니다
  externalTrafficPolicy: Local
```

NlbTestPod01에 접속해서 Client IP가 보이는지 확인해 봅니다

## NLB기반 Loadbalancer 배포

다음과 같은 구성을 통해서 NLB 서비스를 구현해 봅니다.&#x20;

![](<../.gitbook/assets/image (183).png>)

* namespace : nlb-test
* ecsdemo-frontend service type : nlb (external)
* ecsdemo-crystal service type: nlb(internal)
* ecsdemo-nodejs service type: nlb(internal)

### 10.FrontEnd 어플리케이션 배포

새로운 namespace를 구성합니다.

```
kubectl create namespace nlb-test
```

어플리케이션을 배포하고, service를 구성합니다.

```
## nlb-test-01 namespace를 생성하고, pod, service를 배포
kubectl create namespace nlb-test
kubectl -n nlb-test apply -f ~/environment/eksdemo-frontend/kubernetes/nlb_deployment.yaml
kubectl -n nlb-test apply -f ~/environment/eksdemo-frontend/kubernetes/nlb_service.yaml

```

정상적으로 Pod가 배포되었는지 아래 명령을 통해서 확인해 봅니다.

```
kubectl -n nlb-test-01 get pod -o wide
kubectl -n nlb-test-01 get service -o wide

```

Replica를 3개로 늘려서 LB가 FrontEnd에서 정상적으로 이뤄지는 지 확인합니다.

```
kubectl -n nlb-test scale deployment ecsdemo-frontend --replicas=3
```

{% hint style="info" %}
NLB를 구성하기 위해서는 annotation을 통한 Labeling이 필요합니다. 아래 내용을 확인하고 목적에 맞게 설정합니다. 배포 파일에 이미 설정되어 있습니다
{% endhint %}

```
#인스턴스 기반 외부 NLB
service.beta.kubernetes.io/aws-load-balancer-type: "nlb"

#인스턴스 기반 내부 NLB
service.beta.kubernetes.io/aws-load-balancer-internal: "true"

#IP 기반 외부 NLB
service.beta.kubernetes.io/aws-load-balancer-type: "nlb-ip"
```

{% hint style="info" %}
NLB를 위해서는 사전에 서브넷에 태그가 지정되어야 합니다. 각 가용 영역에서 퍼블릭 서브넷을 선택하는 대신 외부 로드 밸런서에 대해 이러한 서브넷만 사용해야 한다는 것을 Kubernetes가 알 수 있도록 다음과 같이 퍼블릭 서브넷에 태그를 지정해야 합니다 .March 26, 2020 이후에 `eksctl` 또는 Amazon EKS AWS CloudFormation 템플릿을 사용하여 VPC를 생성하는 경우 서브넷은 생성될 때 적절하게 태그가 지정됩니다.
{% endhint %}

#### 참조 - 외부 로드밸런서를 위한 Public subnet 태그&#x20;

| 키                        | 값   |
| ------------------------ | --- |
| `kubernetes.io/role/elb` | `1` |

#### 내부 로드밸런서를 위한 Private subnet 태그&#x20;

| 키                                 | 값   |
| --------------------------------- | --- |
| `kubernetes.io/role/internal-elb` | `1` |

아래 출력되는 결과의 EXTERNAL-IP를 복사해서 브라우져 창에서 실행해 봅니다.

```
kubectl -n nlb-test get service ecsdemo-frontend -o wide                                                           
NAME               TYPE           CLUSTER-IP     EXTERNAL-IP                                                                          PORT(S)        AGE   SELECTOR
ecsdemo-frontend   LoadBalancer   172.20.42.31   a7400b4751cf74f8e9cf9acb0c22c8b7-674596f9c43ee0e0.elb.ap-northeast-2.amazonaws.com   80:32228/TCP   17m   app=ecsdemo-frontend
```

![](<../.gitbook/assets/image (148).png>)

앞서 설치해 둔 K9s 유틸리티를 통해서 , 현재 배포된 Pod들의 상태를 확인해 봅니다.

```
k9s -A

```

{% hint style="info" %}
우리 LAB에서는 NLB의 Internal과 External을 어떻게 변경했는지 yaml 파일을 다시 확인해 봅니다.
{% endhint %}

### 10.BackEnd 어플리케이션 배포

Backend 어플리케이션 Nodejs와 Crystal을 배포합니다. 이 2개의 어플리케이션들은 Private Subnet에 배포할 것입니다. 이 구성은 앞서 이미 Yaml 파일의 Deployment에서 nodeSelector로 지정하였습니다.

```
#nodejs nlb depolyment,service apply
kubectl -n nlb-test apply -f ~/environment/eksdemo-crystal/kubernetes/nlb_deployment.yaml
kubectl -n nlb-test apply -f ~/environment/eksdemo-crystal/kubernetes/nlb_service.yaml

#crystal nlb depolyment,service apply
kubectl -n nlb-test apply -f ~/environment/eksdemo-crystal/kubernetes/nlb_deployment.yaml
kubectl -n nlb-test apply -f ~/environment/eksdemo-crystal/kubernetes/nlb_service.yaml

```

정상적으로 Pod가 배포되었는지 아래 명령을 통해서 확인해 봅니다.

```
kubectl -n nlb-test get deployments ecsdemo-nodejs -o wide
kubectl -n nlb-test get service ecsdemo-nodejs -o wide 
kubectl -n nlb-test get deployments ecsdemo-crystal -o wide
kubectl -n nlb-test get service ecsdemo-crystal -o wide 

```

Replica를 3개로 늘려서 Service Type이 없는 경우, BackEnd에서 정상적으로 이뤄지는 지 확인합니다.

```
kubectl -n nlb-test scale deployment ecsdemo-nodejs --replicas=3
kubectl -n nlb-test scale deployment ecsdemo-crystal --replicas=3

```

![](<../.gitbook/assets/image (156).png>)

k9s 를 통해 Pod의 구성을 확인합니다.

![](<../.gitbook/assets/image (154).png>)



