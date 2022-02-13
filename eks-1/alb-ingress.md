---
description: 'update : 2021-10-20 / 30min'
---

# AWS Load Balancer Controller

## Ingress 아키텍쳐&#x20;

### 1.소개

Ingress는 앞서 소개된 Loadbalancer 방식과 다르게 URL 패스에 대한 설정을 담당하는 자원입니다. 외부에서 요청하는 HTTP에 대한 트래픽 처리를 지원하게 됩니다. (eg. 도메인 기반 라우팅) 사용자들이 외부에서 접근이 가능한 URL을 제공하여 , 사용자의 접근성을 편리하게 제공합니다.

Ingress는 Ingress Controller가 존재하고, Ingress 에 정의된 트래픽 라우팅 규칙에 따라 라우팅을 처리합니다.

### 2. Ingress Controller

Ingress 는 반드시 Ingress Controller가 존재 해야하며, 외부에서 내부로 요청되는 트래픽을 읽고 서비스로 전달하는 역할을 합니다. 다른 컨트롤러와 다르게 목적에 맞게 수동으로 설치해야 합니다.

AWS EKS 환경에서는 AWS Load Balancer Controller 를 별도로 설치하고, Ingress는 ALB를 통해 구성됩니다

* NGINX Ingress Controller
* HA Proxy
* **AWS Load Balancer Controller (이전 이름 : ALB Ingress Controller)**
* Kong
* traefik

![](<../.gitbook/assets/image (218).png>)

### 3. ALB Ingress & AWS Load Balancer Controller 트래픽 흐름

* 외부 사용자는 ALB DNS A Record:Port 번호로 접근합니다
* ALB는 각 노드로 로드밸런싱 합니다
* Kube-API에 의해 업데이트 된 정보를 가지고 ALB에서 Loadbalancing 처리를 합니다.&#x20;

![](<../.gitbook/assets/image (222).png>)

## AWS ALB Ingress 개요.

AWS 로드 밸런서 컨트롤러는 Kubernetes 클러스터의 AWS Elastic Load Balancer를 관리합니다. AWS ALB Ingress Controller"로 알려졌으며 "AWS Load Balancer Controller"로 브랜드를 변경했습니다.

수신 리소스는 ALB를 구성하여 HTTP 또는 HTTPS 트래픽을 클러스터 내 다른 포드로 라우팅합니다. ALB 수신 컨트롤러는 Amazon EKS 클러스터에서 실행 중인 프로덕션 워크로드에서 지원됩니다.

## AWS 로드 밸런서 컨트롤러 작동 방식 <a href="#how-aws-load-balancer-controller-works" id="how-aws-load-balancer-controller-works"></a>

### 4. 인그레스 생성

다음 다이어그램은 이 컨트롤러가 생성하는 AWS 구성 요소를 자세히 설명합니다. 또한 수신 트래픽이 ALB에서 Kubernetes 클러스터로 이동하는 경로를 보여줍니다.

1. 컨트롤러는 API 서버의 Ingress 이벤트를 모니터링합니다. 요구 사항을 충족하는 수신 리소스를 찾으면 AWS 리소스 생성을 시작합니다.
2. 새 수신 리소스에 대해 AWS에서 ALB (ELBv2)가 생성됩니다. 이 ALB는 인터넷에 연결되거나 내부에 있을 수 있습니다. Annotation을 사용하여 생성된 서브넷을 지정할 수도 있습니다.
3. Target Group은 수신 리소스에 설명된 각 고유 Kubernetes 서비스에 대해 AWS에서 생성됩니다.
4. 수신 리소스 Annotation에 자세히 설명된 모든 포트에 대해 리스너가 생성됩니다.[ ](http://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html)포트를 지정하지 않으면 적절한 기본값( `80`또는 `443`)이 사용됩니다. Annotation을 통해 인증서를 첨부할 수도 있습니다.
5. 수신 리소스에 지정된 각 경로에 대해 규칙이 생성됩니다[. ](http://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-update-rules.html)이렇게 하면 특정 경로에 대한 트래픽이 적절한 Kubernetes 서비스로 라우팅됩니다.



![참조 - https://aws.amazon.com/ko/blogs/opensource/kubernetes-ingress-aws-alb-ingress-controller/](<../.gitbook/assets/image (21).png>)

Reference - [https://github.com/kubernetes-sigs/aws-load-balancer-controller](https://github.com/kubernetes-sigs/aws-load-balancer-controller) , [https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases](https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases),\
[https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/)

### 5. 인그레스 트래픽

AWS Load Balancer 컨트롤러는 두 가지 트래픽 모드를 지원합니다.

* 인스턴스 모드
* IP 모드

기본적으로 Instance Mode가 사용되며 , Annotation `Instance mode`을 통해 모드를 선택할 수 있습니다 .`alb.ingress.kubernetes.io/target-type`

**인스턴스 모드**

수신 트래픽은 ALB에서 시작하여 각 서비스의 NodePort를 통해 Kubernetes 노드에 도달합니다. 이는 수신 리소스에서 참조하는 서비스가 ALB에 도달하기 위해 에서 `type:NodePort`가 있어야함을 의미합니다.

**IP 모드**

수신 트래픽은 ALB에서 시작하여 Kubernetes Pod에 직접 도달합니다. CNI는 ENI의 Secondary IP 주소를 통해 직접 액세스할 수 있는 POD IP를 지원해야 합니다 .

## AWS 로드밸런서 컨트롤러 구성

아래와 같은 구성 단계로 ALB Ingress를 구성합니다.

1. IAM OIDC 공급자 생성
2. AWS Loadbalancer 컨트롤러에 대한 IAM 정책 다운로드.
3. AWSLoadBalancerControllerIAMPolicy 이름의 IAM 정책 생성.
4. AWS Load Balancer 컨트롤러에 대한 IAM역할 및 ServiceAccount 생성
5. EKS Cluster에 컨트롤러 추가 &#x20;

### 4.IAM OIDC Provider 생성

IAM OIDC Provider는 기본으로 활성화되어 있지 않습니다. eksctl을 사용하여 IAM OIDC Provider를 생성합니다.

```
eksctl utils associate-iam-oidc-provider \
    --region ${AWS_REGION} \
    --cluster eksworkshop \
    --approve
    
```

다음과 같이 IAM 서비스 메뉴에서 생성된 OIDC를 확인 할 수 있습니다.

![](<../.gitbook/assets/image (197).png>)

### 5. AWS Load Balancer 컨트롤러에 대한 IAM 정책 다운로드&#x20;

ALB Load Balancer 컨트롤러에 대한 IAM정책을 다운로드 받습니다. (이미 앞서 git에서 받은 폴더에 포함되어 있습니다.)

```
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.2/docs/install/iam_policy.json

```

### 6. AWSLoadBalancerControllerIAMPolicy IAM 정책 생성.

AWSLoadBalancerControllerIAMPolicy라는 IAM 정책을 생성합니다.

```
cd ~/environment/myeks/alb-controller
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://./iam-policy.json
```

아래 처럼 결과가 출력됩니다.

```
{
    "Policy": {
        "PolicyName": "AWSLoadBalancerControllerIAMPolicy",
        "PolicyId": "ANPA5MSDOOD6OR6VVGXZ4",
        "Arn": "arn:aws:iam::920338198780:policy/AWSLoadBalancerControllerIAMPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2021-04-07T15:58:25+00:00",
        "UpdateDate": "2021-04-07T15:58:25+00:00"
    }
}
```

아래와 같이 IAM - 정책 메뉴에서 새롭게 생성된 정책을 확인할 수 있습니다.&#x20;

![](<../.gitbook/assets/image (196).png>)

### 7. AWS Ingress Controller IAM 역할 및 Service Account 생성

이 단계에서는 AWS Ingress Controller 에 대한 IAM Roel , Service Account를 생성하고, 3번 단계에서 출력되었던 AWS Account ID를 복사해서 사용해아합니다. 앞서 "${ACCOUNT\_ID}에 저장해 두었습니다.

```
eksctl create iamserviceaccount \
--cluster=eksworkshop \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--approve

```

Kubernetes에 정상적으로 Service Account가 등록되었는지 확인해 봅니다.

```
kubectl get serviceaccounts -n kube-system aws-load-balancer-controller -o yaml

```

아래와 같이 출력 예제를 확인해 볼 수 있습니다.

```
whchoi:~/environment/myeks/alb-controller (master) $ kubectl get serviceaccounts -n kube-system aws-load-balancer-controller -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::920338198780:role/eksctl-eksworkshop-addon-iamserviceaccount-k-Role1-OV9MERC8QPAV
  creationTimestamp: "2021-04-07T16:17:17Z"
  labels:
    app.kubernetes.io/managed-by: eksctl
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:eks.amazonaws.com/role-arn: {}
        f:labels:
          .: {}
          f:app.kubernetes.io/managed-by: {}
    manager: eksctl
    operation: Update
    time: "2021-04-07T16:17:17Z"
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:secrets:
        .: {}
        k:{"name":"aws-load-balancer-controller-token-hr5rh"}:
          .: {}
          f:name: {}
    manager: kube-controller-manager
    operation: Update
    time: "2021-04-07T16:17:17Z"
  name: aws-load-balancer-controller
  namespace: kube-system
  resourceVersion: "42507"
  selfLink: /api/v1/namespaces/kube-system/serviceaccounts/aws-load-balancer-controller
  uid: 673de39f-a336-4d01-8fd1-8045b410e02a
secrets:
- name: aws-load-balancer-controller-token-hr5rh
```

> 참조  URL - [https://eksctl.io/usage/iamserviceaccounts/](https://eksctl.io/usage/iamserviceaccounts/)
>
> Amazon EKS는 클러스터 운영자가 AWS IAM 역할을 Kubernetes 서비스 계정에 매핑 할 수 있도록하는 [ISA (IAM Roles for Service Accounts)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) 를 지원합니다 .
>
> 이를 통해 EKS에서 실행되고 다른 AWS 서비스를 사용하는 앱에 대해 세분화 된 권한 관리를 제공합니다. S3, 다른 데이터 서비스 (RDS, MQ, STS, DynamoDB) , AWS ALB Ingress 컨트롤러 또는 ExternalDNS와 같은 Kubernetes 구성 요소를 사용하는 어플리케이션 들이 대표적입니다.IAM OIDC Provider는 기본적으로 활성화되어 있지 않습니다.

### 8. 인증서 관리자 설치

아래와 같이 Cert Manager (인증서 관리자)를 설치합니다.

```
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.2/cert-manager.yaml

```

### 9. AWS ALB Loadbalancer Controller Pod 설치

Helm 기반 또는 mainfest 파일을 통해 ALB Loadbalancer Controller Pod를 설치합니다. 여기에서는 Yaml을 통해 직접 설치해 봅니다.

```
wget https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.2/docs/install/v2_1_2_full.yaml

```

이미 git을 통해 사전에 다운로드 받아 두었습니다. 직접 실행합니다.

```
cd ~/environment/myeks/alb-controller
kubectl apply -f v2_1_3_full.yaml
```

## ALB Ingress 시험

ALB Ingress를 시험하기 위해 아래와 같이 namespace와  pod,service를 배포합니다.

```
## alb-test-01 namespace를 생성하고, pod, service를 배포 
kubectl create namespace alb-test-01
kubectl -n alb-test-01 apply -f ~/environment/myeks/network-test/alb-test-01.yaml
kubectl -n alb-test-01 apply -f ~/environment/myeks/network-test/alb-test-01-ingress.yaml
kubectl -n alb-test-01 apply -f ~/environment/myeks/network-test/alb-test-01-service.yaml

```

아래와 같은 명령으로 결과를 확인 할 수 있습니다.

```
kubectl -n alb-test-01 get pod -o wide
kubectl -n alb-test-01 get service -o wide 
kubectl -n alb-test-01 get ingress -o wide 

```

dk아래와 같은 결과를 확인하고 ingress LB의 외부 A Record를 확인합니다. 해당 A Record를 Cloud9 IDE Terminal에서  Curl을 통해 접속하거나 브라우저에서 접속해 봅니다

```
$ kubectl -n alb-test-01 get pod -o wide
NAME                          READY   STATUS    RESTARTS   AGE     IP              NODE                                               NOMINATED NODE   READINESS GATES
alb-test-01-ffd85d89f-5s66x   1/1     Running   0          6m15s   10.11.92.148    ip-10-11-94-22.ap-northeast-2.compute.internal     <none>           <none>
alb-test-01-ffd85d89f-sl5bd   1/1     Running   0          6m14s   10.11.88.200    ip-10-11-94-22.ap-northeast-2.compute.internal     <none>           <none>
alb-test-01-ffd85d89f-td4tv   1/1     Running   0          6m14s   10.11.107.214   ip-10-11-108-153.ap-northeast-2.compute.internal   <none>           <none>
$ kubectl -n alb-test-01 get service -o wide
NAME          TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE   SELECTOR
alb-test-01   NodePort   172.20.121.174   <none>        8080:30299/TCP   12m   app=alb-test-01
$ kubectl -n alb-test-01 get ingress -o wide
NAME          CLASS    HOSTS   ADDRESS                                                                      PORTS   AGE
alb-test-01   <none>   *       k8s-albtest0-albtest0-1aa7c83247-45114489.ap-northeast-2.elb.amazonaws.com   80      13m
```

아래와 같이 배포된 pod에 접속을 편리하게 하기 위해 Cloud9 IDE terminal Shell에 등록 합니다.

```
echo "export AlbTestPod03=alb-test-01-ffd85d89f-td4tv" | tee -a ~/.bash_profile
echo "export AlbTestPod02=alb-test-01-ffd85d89f-5s66x" | tee -a ~/.bash_profile
echo "export AlbTestPod01=alb-test-01-ffd85d89f-sl5bd" | tee -a ~/.bash_profile
source ~/.bash_profile

```

AlbTestPod01에 접속해서 아래와 같이 확인해 봅니다.

```
kubectl -n alb-test-01 exec -it $ClbTestPod01 -- /bin/sh
tcpdump -s 0 -A 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'

```

Pod에서 TCPDump로 확인하면 정상적으로 Pakcet이 덤프되고 X-Forwarded를 통해 Source IP를 확인할 수 있습니다

```
tcpdump -s 0 -A 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'

11:23:59.800248 IP ip-10-11-29-154.ap-northeast-2.compute.internal.62078 > alb-test-01-ffd85d89f-sl5bd.80: Flags [P.], seq 638:1276, ack 180, win 110, options [nop,nop,TS val 355718059 ecr 1502835583], length 638: HTTP: GET / HTTP/1.1
E...j.@.....
...
.X..~.Pj>...x{....nY......
.3..Y.s.GET / HTTP/1.1
X-Forwarded-For: 122.40.8.88
X-Forwarded-Proto: http
X-Forwarded-Port: 80
Host: k8s-albtest0-albtest0-1aa7c83247-45114489.ap-northeast-2.elb.amazonaws.com
X-Amzn-Trace-Id: Root=1-616ffc4f-26f121442cec9b1b3ac7fdbb
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:92.0) Gecko/20100101 Firefox/92.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: ko-KR,ko;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
Upgrade-Insecure-Requests: 1
If-Modified-Since: Wed, 20 Oct 2021 11:07:33 GMT
If-None-Match: "616ff875-53"
Cache-Control: max-age=0
```

아래와 같이 ALB Ingress가 구성되었습니다.&#x20;

![](<../.gitbook/assets/image (226).png>)

## ALB Ingress Controller 기반 Application 배포

아래와 같이 새로운 manifest 파일을 생성합니다. (이미 git을 통해 해당 폴더에 다운로드 받았습니다. 생략 할 수 있습니다. )

```
cat <<EoF > ~/environment/myeks/alb-controller/alb_front_full.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: alb-test
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecsdemo-frontend
  labels:
    app: ecsdemo-frontend
  namespace: alb-test
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
          value: "http://ecsdemo-crystal.alb-test.svc.cluster.local/crystal"
        - name: NODEJS_URL
          value: "http://ecsdemo-nodejs.alb-test.svc.cluster.local/"
#add nodeSelector
      nodeSelector:
        nodegroup-type: "backend-workloads"
---
apiVersion: v1
kind: Service
metadata:
  name: ecsdemo-frontend
  namespace: alb-test
spec:
  selector:
    app: ecsdemo-frontend
  type: NodePort
  ports:
   -  protocol: TCP
      port: 80
      targetPort: 3000
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ecsdemo-frontend
  namespace: alb-test
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - http:
        paths:
          - path: /*
            backend:
              serviceName: "ecsdemo-frontend"
              servicePort: 80
EoF

```

yaml 파일을 배포하고, 서비스를 확인합니다.

```
kubectl create namespace alb-test
kubectl apply -f ~/environment/myeks/alb-controller/alb_front_full.yaml 
kubectl -n alb-test get all

```

아래와 같은 결과를 확인 할 수 있습니다.

```
whchoi:~/environment/myeks/alb-controller (master) $ kubectl -n alb-test get all
NAME                                   READY   STATUS    RESTARTS   AGE
pod/ecsdemo-frontend-6cc7bb877-tjsrk   1/1     Running   0          48s
pod/ecsdemo-frontend-6cc7bb877-tknk7   1/1     Running   0          48s
pod/ecsdemo-frontend-6cc7bb877-xgfxj   1/1     Running   0          48s

NAME                       TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
service/ecsdemo-frontend   NodePort   172.20.63.91   <none>        80:32203/TCP   48s

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ecsdemo-frontend   3/3     3            3           48s

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/ecsdemo-frontend-6cc7bb877   3         3         3       48s

NAME                                                               SERVICE-NAME       SERVICE-PORT   TARGET-TYPE   AGE
targetgroupbinding.elbv2.k8s.aws/k8s-albtest-ecsdemof-b28228de87   ecsdemo-frontend   80             ip            46s
```

ingress 배포 현황을 살펴보고, ADDRESS를 복사해서 브라우저에서 확인해 봅니다.

```
kubectl -n alb-test get ingress -o wide

```

&#x20;아래와 같이 결과를 확인 할 수 있습니다.

```
whchoi:~/environment/myeks/alb-controller (master) $ kubectl -n alb-test get ingress -o wide
Warning: extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
NAME               CLASS    HOSTS   ADDRESS                                                                       PORTS   AGE
ecsdemo-frontend   <none>   *       k8s-albtest-ecsdemof-ee882fe5b1-1732413705.ap-northeast-2.elb.amazonaws.com   80      2m34s
```

![](<../.gitbook/assets/image (195).png>)
