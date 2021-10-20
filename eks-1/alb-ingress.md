---
description: 'update : 2021-04-08 / 30min'
---

# ALB Ingress 배포

## Ingress

### 소개

Ingress는 앞서 소개된 Loadbalancer 방식과 다르게 URL 패스에 대한 설정을 담당하는 자원입니다. 외부에서 요청하는 HTTP에 대한 트래픽 처리를 지원하게 됩니다. (eg. 도메인 기반 라우팅) 사용자들이 외부에서 접근이 가능한 URL을 제공하여 , 사용자의 접근성을 편리하게 제공합니다.

Ingress는 Ingress Controller가 존재하고, Ingress 에 정의된 트래픽 라우팅 규칙에 따라 라우팅을 처리합니다.

### Ingress Controller

Ingress 는 반드시 Ingress Controller가 존재해야하며, 외부에서 내부로 요청되는 트래픽을 읽고 서비스로 전달하는 역할을 합니다. 다른 컨트롤러와 다르게 목적에 맞게 수동으로 설치해야 합니다.

* NGINX Ingress Controller
* HA Proxy
* AWS ALB Ingress Controller
* Kong
* traefik

## AWS ALB Ingress 개요.

[Kubernetes용 AWS ALB 수신 컨트롤러](https://github.com/kubernetes-sigs/aws-alb-ingress-controller)는 `kubernetes.io/ingress.class: alb` 주석과 클러스터에 수신 리소스가 생성될 때마다 Application Load Balancer(ALB) 및 필수 지원 AWS 리소스가 생성되도록 트리거하는 컨트롤러입니다.&#x20;

수신 리소스는 ALB를 구성하여 HTTP 또는 HTTPS 트래픽을 클러스터 내 다른 포드로 라우팅합니다. ALB 수신 컨트롤러는 Amazon EKS 클러스터에서 실행 중인 프로덕션 워크로드에서 지원됩니다.

## AWS ALB Ingress 구성

아래와 같은 구성 단계로 ALB Ingress를 구성합니다.

1. IAM OIDC 공급자 생성
2. AWS Loadbalancer 컨트롤러에 대한 IAM 정책 다운로드.
3. AWSLoadBalancerControllerIAMPolicy 이름의 IAM 정책 생성.
4. AWS Load Balancer 컨트롤러에 대한 IAM역할 및 ServiceAccount 생성
5. EKS Cluster에 컨트롤러 추가 &#x20;

![참조 - https://aws.amazon.com/ko/blogs/opensource/kubernetes-ingress-aws-alb-ingress-controller/](<../.gitbook/assets/image (21).png>)

Reference - [https://github.com/kubernetes-sigs/aws-load-balancer-controller](https://github.com/kubernetes-sigs/aws-load-balancer-controller) , [https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases](https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases),\
[https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/)

### 1.IAM OIDC Provider 생성

IAM OIDC Provider는 기본으로 활성화되어 있지 않습니다. eksctl을 사용하여 IAM OIDC Provider를 생성합니다.

```
eksctl utils associate-iam-oidc-provider \
    --region ${AWS_REGION} \
    --cluster eksworkshop \
    --approve
    
```

다음과 같이 IAM 서비스 메뉴에서 생성된 OIDC를 확인 할 수 있습니다.

![](<../.gitbook/assets/image (197).png>)

### 2. AWS Load Balancer 컨트롤러에 대한 IAM 정책 다운로드&#x20;

ALB Load Balancer 컨트롤러에 대한 IAM정책을 다운로드 받습니다. (이미 앞서 git에서 받은 폴더에 포함되어 있습니다.)

```
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.2/docs/install/iam_policy.json

```

### 3. AWSLoadBalancerControllerIAMPolicy IAM 정책 생성.

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

### 4. AWS Ingress Controller IAM 역할 및 Service Account 생성

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

### 5. 인증서 관리자 설치

아래와 같이 Cert Manager (인증서 관리자)를 설치합니다.

```
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.2/cert-manager.yaml

```

### 6. AWS ALB Loadbalancer Controller Pod 설치

Helm 기반 또는 mainfest 파일을 통해 ALB Loadbalancer Controller Pod를 설치합니다. 여기에서는 Yaml을 통해 직접 설치해 봅니다.

```
wget https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.2/docs/install/v2_1_2_full.yaml

```

이미 git을 통해 사전에 다운로드 받아 두었습니다. 직접 실행합니다.

```
cd ~/environment/myeks/alb-controller
kubectl apply -f v2_1_3_full.yaml
```

## ALB Ingress Controller 기반 ALB 확인

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
