---
description: 'update : 2022-02-12 / 2h'
---

# Loadbalancer 기반 배포

## Loadbalancer 서비스 타입 소개.

Loadbalancer 기반의 서비스 타입은 현재 CLB (Classic Load Balancer)와 NLB(Network Load Balancer) 2가지 타입을 지원하고 있습니다. 모두 Port 기반의 LB를 제공하고 있으며, Kubernetes 의 Node와 Service 전면에서 서비스를 제공합니다.

## CLB Loadbalancer 서비스 기반 구성

### 1.CLB 기반 ServiceType

Service Type 필드를 LoadBalancer로 설정하여 프로브저닝합니다. Cloud Service Provider의 기본 로드밸런서 타입을 사용하게 되며, AWS의 경우에는 CLB를 사용합니다. CLB는 내부 또는 외부 로드밸런서로 지정이 가능합니다.

### 2. CLB Service Type 트래픽 흐름

Traffic 흐름은 다음과 같습니다.

* 외부 사용자는 CLB DNS A Record:Port로 접근
* CLB는 NodePort로 LB 처리 (NodePort는 임의로 할당 됩니다.)
* NodePort는 ClusterIP로 Forwarding되고 IPTable에 의해 분산 처리 됩니다.

![](<../.gitbook/assets/image (358).png>)

### 3. CLB Service 시험

CLB Loadbalance Service Type 을 시험하기 위해 아래와 같이 namespace와  pod,service를 배포합니다.&#x20;

정상적으로 배포되었는지 아래 Command로 확인합니다.&#x20;

```
## clb-test-01 namespace를 생성하고, pod, service를 배포 
kubectl apply -f ~/environment/myeks/network-test/clb-test-01-deployment.yaml 
## clb-test-01 namespace의 pod 확인 
kubectl -n clb-test-01 get pod -o wide
## clb-test-01 namespace의 service 확인
kubectl -n clb-test-01 get service -o wide

```

정상적으로 배포되었는지 아래 Command로 확인합니다.

```
## clb-test-01 namespace의 pod 확인 
kubectl -n clb-test-01 get pod -o wide

## clb-test-01 namespace의 service 확인
kubectl -n clb-test-01 get service -o wide

```

다음과 같은 결과를 얻을 수 있습니다

```
kubectl -n clb-test-01 get pod -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP             NODE                                              NOMINATED NODE   READINESS GATES
clb-test-01-5bbc58fd87-29k6r   1/1     Running   0          32s   10.11.42.35    ip-10-11-44-207.ap-northeast-2.compute.internal   <none>           <none>
clb-test-01-5bbc58fd87-f56bv   1/1     Running   0          32s   10.11.20.234   ip-10-11-26-22.ap-northeast-2.compute.internal    <none>           <none>
clb-test-01-5bbc58fd87-fr49z   1/1     Running   0          32s   10.11.2.14     ip-10-11-1-131.ap-northeast-2.compute.internal    <none>           <none>
kubectl -n clb-test-01 get service -o wide
NAME              TYPE           CLUSTER-IP    EXTERNAL-IP                                                                    PORT(S)          AGE   SELECTOR
clb-test-01-svc   LoadBalancer   172.20.81.6   aa4ad3c7607774968a7f13c1940554c1-1190922948.ap-northeast-2.elb.amazonaws.com   8080:31906/TCP   32s   app=clb-test-01
```

아래와 같이 구성됩니다 . nodeport는 별도의 지정이 없으면 생성할때 자동으로 지정됩니다.&#x20;

![](<../.gitbook/assets/image (344).png>)

아래와 같이 배포된 pod에 접속을 편리하게 하기 위해 Cloud9 IDE terminal Shell에 등록 합니다. (Option)

```
export Clb_Test_Pod01=$(kubectl -n clb-test-01 get pod -o wide | awk 'NR==2' | awk '/clb-test-01/{print $1 } ')
export Clb_Test_Pod02=$(kubectl -n clb-test-01 get pod -o wide | awk 'NR==3' | awk '/clb-test-01/{print $1 } ')
export Clb_Test_Pod03=$(kubectl -n clb-test-01 get pod -o wide | awk 'NR==4' | awk '/clb-test-01/{print $1 } ')
echo "export Clb_Test_Pod01=${Clb_Test_Pod01}" | tee -a ~/.bash_profile
echo "export Clb_Test_Pod02=${Clb_Test_Pod02}" | tee -a ~/.bash_profile
echo "export Clb_Test_Pod03=${Clb_Test_Pod03}" | tee -a ~/.bash_profile
source ~/.bash_profile

```

ClbTestPod01에 접속해서 아래와 같이 확인해 봅니다. K9s에서 접속해도 됩니다.

```
kubectl -n clb-test-01 exec -it $Clb_Test_Pod01 -- /bin/sh
nslookup {cluster-ip}
# Container tcpdump
tcpdump -i eth0 dst port 80 | grep "HTTP: GET"

```

Cloud9 IDE Terminal에서 CLB External IP:8080 으로 접속합니다.

```
## clb-test-01-svc external hostname 변수 등록
kubectl -n clb-test-01 get svc clb-test-01-svc --output jsonpath='{.status.loadBalancer.ingress[*].hostname}'
export clb_test_01_svc_name=$(kubectl -n clb-test-01 get svc clb-test-01-svc --output jsonpath='{.status.loadBalancer.ingress[*].hostname}')
echo "export clb_test_01_svc_name=${clb_test_01_svc_name}" | tee -a ~/.bash_profile
source ~/.bash_profile

## clb-test-01-svc external hostname 으로 접속
curl $clb_test_01_svc_name:8080
```

Node에서 iptable에 설정된 NAT Table, Loadbalancing 구성을 확인해 봅니다.

```
aws ssm start-session --target $ng_public01_id
sudo -s
iptables -t nat -nvL --line-number | more
iptables -t nat -nvL --line-number | grep clb-test-01-svc
iptables -t nat -nL KUBE-SERVICES
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

* namespace : clb-test
* eksdemo-frontend service type : LoadbBlancer
* eksdemo-crystal service type: Cluster-IP&#x20;
* eksdemo-nodejs service type: Cluster-IP&#x20;

![](<../.gitbook/assets/image (59).png>)

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

CLB 접속 URL 주소는 아래 결과로 출력할 수 있습니다.&#x20;

```
kubectl -n clb-test get svc ecsdemo-frontend | tail -n 1 | awk '{ print "CLB-FRONTEND URL = http://"$4 }' 

```

출력결과 예시

![](<../.gitbook/assets/image (462).png>)

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

![](<../.gitbook/assets/image (292).png>)

k9s 를 통해 Pod의 구성을 확인합니다.

{% hint style="info" %}
LAB 을 진행하면서, Pod의 배포 상황을 계속 모니터링하기 위해서 Cloud9 에서 Terminal을 하나 더 열고 K9s를 실행 시켜 두는 것이 좋습니다.
{% endhint %}

![](<../.gitbook/assets/image (53).png>)

### 6. Loadbalancer 확인.

이제 서비스 타입을 확인하기 위해서 EC2 대시보드에서 Loadbalancer를 확인합니다.

CLB의 DNS Name을 복사해서 Web Browser에서 입력합니다.

![](<../.gitbook/assets/image (160).png>)

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

### 7. Super Mario 어플리케이션 배포하기

CLB 로드밸런서를 사용하는 간단한 게임 앱을 배포해 봅니다.

```
kubectl create namespace mario
kubectl apply -f ~/environment/myeks/sample/super_mario.yml 

```

정상적으로 배포되었는지 확인해 봅니다.&#x20;

```
kubectl -n mario get pods,svc

```

아래와 같이 배포된 것을 확인 할 수 있습니다.

```
$ kubectl -n mario get pods,svc
NAME                         READY   STATUS              RESTARTS   AGE
pod/mario-7f947cb549-mxzk8   0/1     ContainerCreating   0          9s

NAME            TYPE           CLUSTER-IP       EXTERNAL-IP                                                                   PORT(S)        AGE
service/mario   LoadBalancer   172.20.103.149   a0557981a407141ab81fb9e0e5a5e02f-169514957.ap-northeast-2.elb.amazonaws.com   80:30459/TCP   9s
```

실제 deployment에 사용된 YAML 을 확인해 봅니다.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mario
  labels:
    app: mario
  namespace: mario
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mario
  template:
    metadata:
      labels:
        app: mario
    spec:
      containers:
      - name: mario
        image: pengbai/docker-supermario
      nodeSelector:
        nodegroup-type: "managed-frontend-workloads"
---
apiVersion: v1
kind: Service
metadata:
  name: mario
  namespace: mario
spec:
  selector:
    app: mario
  ports:
  type: LoadBalancer
  ports:
   -  protocol: TCP
      port: 80
      targetPort: 8080
```

아래 명령을 통해서 게임앱 URL을 확인하고에 브라우저를 통해 접속해 봅니다.

```
kubectl -n mario get svc mario | tail -n 1 | awk '{ print "mario URL = http://"$4 }'

```

3\~4분 뒤에 웹 브라우저를 통해 위 명령에서 실행된 URL을 접속하면 , CLB를 통해서 아래와 같이 게임이 실행됩니다.

<figure><img src="../.gitbook/assets/image (300).png" alt=""><figcaption></figcaption></figure>

* "s" - 게임시작
* 방향키로 각 스테이지 이동
* "s" - 점프
* 방향키  - 마리오 이동&#x20;

## NLB Loadbalancer 서비스 기반 구성

### 8. NLB 기반 Service Type

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

### 9.NLB Service Type 트래픽 흐름

Traffic 흐름은 다음과 같습니다.

* 외부 사용자는 NLB DNS A Record:Port로 접근합니다.&#x20;
* NLB는 NodePort로 LB 처리 (NodePort는 임의로 할당 됩니다.)
* NodePort는 ClusterIP로 Forwarding되고 IPTable에 의해 분산 처리 됩니다.

![](<../.gitbook/assets/image (405).png>)

### 10. NLB Service 시험

NLB Loadbalance Service Type 을 시험하기 위해 아래와 같이 namespace와  pod,service를 배포합니다.

```
## nlb-test-01 namespace를 생성하고, pod, service를 배포
kubectl create namespace nlb-test-01
kubectl -n nlb-test-01 apply -f ~/environment/myeks/network-test/nlb-test-01.yaml
kubectl -n nlb-test-01 apply -f ~/environment/myeks/network-test/nlb-test-01-service.yaml

```

정상적으로 배포되었는지 아래 명령어로 확인합니다.&#x20;

```
kubectl -n nlb-test-01 get pod -o wide
kubectl -n nlb-test-01 get service -o wide

```

다음과 같은 결과를 얻을 수 있습니다.&#x20;

```
kubectl -n nlb-test-01 get pod -o wide
NAME                          READY   STATUS    RESTARTS   AGE     IP             NODE                                             NOMINATED NODE   READINESS GATES
nlb-test-01-7c5cf9bd5-c8qfj   1/1     Running   0          6m15s   10.11.13.246   ip-10-11-10-88.ap-northeast-2.compute.internal   <none>           <none>
nlb-test-01-7c5cf9bd5-d8j76   1/1     Running   0          6m15s   10.11.39.128   ip-10-11-35-39.ap-northeast-2.compute.internal   <none>           <none>
nlb-test-01-7c5cf9bd5-gsxlk   1/1     Running   0          6m15s   10.11.27.219   ip-10-11-30-67.ap-northeast-2.compute.internal   <none>           <none>

kubectl -n nlb-test-01 get service -o wide
NAME              TYPE           CLUSTER-IP     EXTERNAL-IP                                                                          PORT(S)          AGE   SELECTOR
nlb-test-01-svc   LoadBalancer   172.20.221.81   aaa7c67484fd94ea8a2b6bf1fa091017-374347eabaa0875f.elb.ap-northeast-2.amazonaws.com   8080:31965/TCP   28s   app=nlb-test-01
```

아래와 같이 구성됩니다 . nodeport는 별도의 지정이 없으면 생성할때 자동으로 지정됩니다.

![](<../.gitbook/assets/image (201).png>)

아래와 같이 배포된 pod에 접속을 편리하게 하기 위해 Cloud9 IDE terminal Shell에 등록 합니다.

```
export Nlb_Test_01_Pod01=$(kubectl -n nlb-test-01 get pod -o wide | awk 'NR==2' | awk '/nlb-test-01/{print $1 } ')
export Nlb_Test_01_Pod02=$(kubectl -n nlb-test-01 get pod -o wide | awk 'NR==3' | awk '/nlb-test-01/{print $1 } ')
export Nlb_Test_01_Pod03=$(kubectl -n nlb-test-01 get pod -o wide | awk 'NR==4' | awk '/nlb-test-01/{print $1 } ')
echo "export Nlb_Test_01_Pod01=${Nlb_Test_01_Pod01}" | tee -a ~/.bash_profile
echo "export Nlb_Test_01_Pod02=${Nlb_Test_01_Pod02}" | tee -a ~/.bash_profile
echo "export Nlb_Test_01_Pod03=${Nlb_Test_01_Pod03}" | tee -a ~/.bash_profile
source ~/.bash_profile

```

NlbTestPod01에 접속해서 아래와 같이 확인해 봅니다.&#x20;

```
#Nlb_Test_01_Pod01 Container 접속
kubectl -n nlb-test-01 exec -it $Nlb_Test_01_Pod01 -- /bin/sh
#Nlb_Test_01_Pod01 Container 에서 HTTP 접속 확인
tcpdump -i eth0 dst port 80 | grep "HTTP: GET"
```

Cloud9 IDE Terminal에서 NLB External IP:8080 으로 접속합니다.&#x20;

```
## nlb-test-01-svc external hostname 변수 등록
export nlb_test_01_svc_name=$(kubectl -n nlb-test-01 get svc nlb-test-01-svc --output jsonpath='{.status.loadBalancer.ingress[*].hostname}')
echo "export nlb_test_01_svc_name=${nlb_test_01_svc_name}" | tee -a ~/.bash_profile
source ~/.bash_profile

## nlb-test-01-svc external hostname 으로 접속
curl $nlb_test_01_svc_name:8080

```

Node에서 iptable에 설정된 NAT Table, Loadbalancing 구성을 확인해 봅니다.

```
aws ssm start-session --target $ng_public01_id
sudo -s
iptables -t nat -nvL --line-number | more
iptables -t nat -nvL --line-number | grep nlb-test-01-svc
iptables -t nat -nL KUBE-SERVICES

```

NLB는 "externalTrafficPolicy: Local"을 지원합니다. 외부의 소스 IP를 그대로 보존하여, Node로 유입된 Traffic을 Node 내의 PoD로 전달합니다.&#x20;

![](<../.gitbook/assets/image (353).png>)

![](<../.gitbook/assets/image (349).png>)

아래와 같이 새롭게 서비스와  PoD를 배포하고 확인해 봅니다.&#x20;

```
## nlb-test-02 namespace 생성 
kubectl create namespace nlb-test-02
## nlb-test-02 namespace에 pod, service 생성 
kubectl -n nlb-test-02 apply -f ~/environment/myeks/network-test/nlb-test-02.yaml
kubectl -n nlb-test-02 apply -f ~/environment/myeks/network-test/nlb-test-02-service.yaml

## nlb-test-02 pod, service 확인 
kubectl -n nlb-test-02 get pod -o wide
kubectl -n nlb-test-02 get service -o wide

```

아래와 같이 pod에 접속을 편리하게 하기 위해 Cloud9 IDE terminal Shell에 등록합니다. &#x20;

```
## pod에 접속을 편리하게 하기 위해 Cloud9 IDE terminal Shell에 등록 
export Nlb_Test_02_Pod01=$(kubectl -n nlb-test-02 get pod -o wide | awk 'NR==2' | awk '/nlb-test-02/{print $1 } ')
export Nlb_Test_02_Pod02=$(kubectl -n nlb-test-02 get pod -o wide | awk 'NR==3' | awk '/nlb-test-02/{print $1 } ')
export Nlb_Test_02_Pod03=$(kubectl -n nlb-test-02 get pod -o wide | awk 'NR==4' | awk '/nlb-test-02/{print $1 } ')
echo "export Nlb_Test_Pod01=${Nlb_Test_02_Pod01}" | tee -a ~/.bash_profile
echo "export Nlb_Test_Pod02=${Nlb_Test_02_Pod02}" | tee -a ~/.bash_profile
echo "export Nlb_Test_Pod03=${Nlb_Test_02_Pod03}" | tee -a ~/.bash_profile
source ~/.bash_profile

```

NlbTestPod에 접속해서 외부의 Client IP가 보이는지 확인해 봅니다.&#x20;

```
### Cloud9 Terminal IP 를 확인합니다. 
curl http://169.254.169.254/latest/meta-data/public-ipv4

## nlb-test-02-svc external hostname 변수 등록
export nlb_test_02_svc_name=$(kubectl -n nlb-test-02 get svc nlb-test-02-svc --output jsonpath='{.status.loadBalancer.ingress[*].hostname}')
echo "export nlb_test_02_svc_name=${nlb_test_02_svc_name}" | tee -a ~/.bash_profile
source ~/.bash_profile

### Cloud9 에서 NLB External IP로 계속 접속 해 봅니다.
while true; do curl ${nlb_test_02_svc_name}:8080; sleep 1; done

### nlb-test-02 namespace의 Pod1로 접속합니다.

kubectl -n nlb-test-02 exec -it ${Nlb_Test_02_Pod01} -- /bin/sh                                   

/ # tcpdump -i eth0 src {cloud9_terminal_ip} and dst port 80 | grep "HTTP: GET"
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
17:03:37.925845 IP ec2-13-125-172-173.ap-northeast-2.compute.amazonaws.com.56206 > nlb-test-02-789d59867-hxnl8.80: Flags [P.], seq 0:151, ack 1, win 211, options [nop,nop,TS val 2984805554 ecr 1007839403], length 151: HTTP: GET / HTTP/1.1
17:03:38.937357 IP ec2-13-125-172-173.ap-northeast-2.compute.amazonaws.com.49756 > nlb-test-02-789d59867-hxnl8.80: Flags [P.], seq 0:151, ack 1, win 211, options [nop,nop,TS val 1993855085 ecr 1007840414], length 151: HTTP: GET / HTTP/1.1

```

Node에서 iptable에 설정된 NAT Table, Loadbalancing 구성을 확인해 봅니다.

```
aws ssm start-session --target $ng_public01_id
sudo -s
iptables -t nat -L --line-number | more
iptables -t nat -L --line-number | grep nlb-test-02-svc
iptables -t nat -nL KUBE-SERVICES

```

## NLB기반 Loadbalancer 배포

다음과 같은 구성을 통해서 NLB 서비스를 구현해 봅니다.&#x20;

![](<../.gitbook/assets/image (417).png>)

* namespace : nlb-test
* ecsdemo-frontend service type : nlb (external)
* ecsdemo-crystal service type: nlb(internal)
* ecsdemo-nodejs service type: nlb(internal)

### 10.FrontEnd 어플리케이션 배포

어플리케이션을 배포하고, service를 구성합니다.

```
## nlb-test namespace를 생성하고, pod, service를 배포
kubectl create namespace nlb-test
kubectl -n nlb-test apply -f ~/environment/eksdemo-frontend/kubernetes/nlb_deployment.yaml
kubectl -n nlb-test apply -f ~/environment/eksdemo-frontend/kubernetes/nlb_service.yaml

```

정상적으로 Pod가 배포되었는지 아래 명령을 통해서 확인해 봅니다.

```
kubectl -n nlb-test get pod -o wide
kubectl -n nlb-test get service -o wide

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
kubectl -n nlb-test get svc ecsdemo-frontend | tail -n 1 | awk '{ print "NLB-TEST URL = http://"$4 }'

```

```
NAME               TYPE           CLUSTER-IP      EXTERNAL-IP                                                                          PORT(S)        AGE   SELECTOR
ecsdemo-frontend   LoadBalancer   172.20.55.163   a68e1e3f279654af99a680bff29f6685-43ae3919758c9316.elb.ap-northeast-2.amazonaws.com   80:31784/TCP   55s   app=ecsdemo-frontend
```

![](<../.gitbook/assets/image (456).png>)

앞서 설치해 둔 K9s 유틸리티를 통해서 , 현재 배포된 Pod들의 상태를 확인해 봅니다.

```
k9s -A

```

{% hint style="info" %}
우리 LAB에서는 NLB의 Internal과 External을 어떻게 변경했는지 yaml 파일을 다시 확인해 봅니다.
{% endhint %}

### 12.BackEnd 어플리케이션 배포

Backend 어플리케이션 Nodejs와 Crystal을 배포합니다. 이 2개의 어플리케이션들은 Private Subnet에 배포할 것입니다. 이 구성은 앞서 이미 Yaml 파일의 Deployment에서 nodeSelector로 지정하였습니다.

```
#nodejs nlb depolyment,service apply
kubectl -n nlb-test apply -f ~/environment/eksdemo-nodejs/kubernetes/nlb_deployment.yaml
kubectl -n nlb-test apply -f ~/environment/eksdemo-nodejs/kubernetes/nlb_service.yaml

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

![](<../.gitbook/assets/image (327).png>)

k9s 를 통해 Pod의 구성을 확인합니다.

![](<../.gitbook/assets/image (55).png>)

### 13. Tetris 어플리케이션 배포

NLB 로드밸런서를 사용하는 간단한 게임 앱을 배포해 봅니다.

```
kubectl create namespace tetris
kubectl apply -f ~/environment/myeks/sample/tetris.yml 

```

정상적으로 배포되었는지 확인해 봅니다.&#x20;

```
kubectl -n tetris get pods,svc

```

아래와 같이 배포된 것을 확인 할 수 있습니다.

```
$ kubectl -n tetris get pods,svc
NAME                          READY   STATUS    RESTARTS   AGE
pod/tetris-67f8b848f4-m5xt2   1/1     Running   0          66s
pod/tetris-67f8b848f4-nn99k   1/1     Running   0          66s
pod/tetris-67f8b848f4-pt4tl   1/1     Running   0          66s

NAME             TYPE           CLUSTER-IP       EXTERNAL-IP                                                                          PORT(S)        AGE
service/tetris   LoadBalancer   172.20.174.151   a2cc703e290b6440a9eb69a6f483f1e7-322e917eecf3bd07.elb.ap-northeast-2.amazonaws.com   80:32418/TCP   66s
```

실제 deployment에 사용된 YAML 을 확인해 봅니다.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tetris
  labels:
    app: tetris
  namespace: tetris
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tetris
  template:
    metadata:
      labels:
        app: tetris
    spec:
      containers:
      - name: tetris
        image: bsord/tetris
      nodeSelector:
        nodegroup-type: "managed-frontend-workloads"
---
apiVersion: v1
kind: Service
metadata:
  name: tetris
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
  namespace: tetris
spec:
  externalTrafficPolicy: Local
  selector:
    app: tetris
  type: LoadBalancer
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
```

아래 명령을 통해서 게임앱 URL을 확인하고에 브라우저를 통해 접속해 봅니다.

```
kubectl -n tetris get svc tetris | tail -n 1 | awk '{ print "tetris-game URL = http://"$4 }'

```

브라우저를 통해 접속하면 아래와 같이 게임이 실행됩니다.

NLB는 배포시간이 5분 정도 소요 됩니다.&#x20;

<figure><img src="../.gitbook/assets/image (318).png" alt=""><figcaption></figcaption></figure>

