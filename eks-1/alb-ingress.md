---
description: 'update : 2020-11-11 / 30min'
---

# ALB Ingress 배포

## Ingress

### 소개

Ingress는 앞서 소개된 Loadbalancer 방식과 다르게 URL 패스에 대한 설정을 담당하는 자원입니다. 외부에서 요청하는 HTTP에 대한 트래픽 처리를 지원하게 됩니다. \(eg. 도메인 기반 라우팅\) 사용자들이 외부에서 접근이 가능한 URL을 제공하여 , 사용자의 접근성을 편리하게 제공합니다.

Ingress는 Ingress Controller가 존재하고, Ingress 에 정의된 트래픽 라우팅 규칙에 따라 라우팅을 처리합니다.

### Ingress Controller

Ingress 는 반드시 Ingress Controller가 존재해야하며, 외부에서 내부로 요청되는 트래픽을 읽고 서비스로 전달하는 역할을 합니다. 다른 컨트롤러와 다르게 목적에 맞게 수동으로 설치해야 합니다.

* NGINX Ingress Controller
* HA Proxy
* AWS ALB Ingress Controller
* Kong
* traefik

## AWS ALB Ingress 개요.

[Kubernetes용 AWS ALB 수신 컨트롤러](https://github.com/kubernetes-sigs/aws-alb-ingress-controller)는 `kubernetes.io/ingress.class: alb` 주석과 클러스터에 수신 리소스가 생성될 때마다 Application Load Balancer\(ALB\) 및 필수 지원 AWS 리소스가 생성되도록 트리거하는 컨트롤러입니다. 

수신 리소스는 ALB를 구성하여 HTTP 또는 HTTPS 트래픽을 클러스터 내 다른 포드로 라우팅합니다. ALB 수신 컨트롤러는 Amazon EKS 클러스터에서 실행 중인 프로덕션 워크로드에서 지원됩니다.

## AWS ALB Ingress 구성

아래와 같은 구성 단계로 ALB Ingress를 구성합니다.

1. IAM OIDC 공급자 생성
2. AWS Loadbalancer 컨트롤러에 대한 IAM 정책 다운로드.
3. AWSLoadBalancerControllerIAMPolicy 이름의 IAM 정책 생성.
4. AWS Load Balancer 컨트롤러에 대한 IAM역할 및 ServiceAccount 생성
5. EKS Cluster에 컨트롤러 추가  

![&#xCC38;&#xC870; - https://aws.amazon.com/ko/blogs/opensource/kubernetes-ingress-aws-alb-ingress-controller/](../.gitbook/assets/image%20%2821%29.png)

Reference - [https://github.com/kubernetes-sigs/aws-load-balancer-controller](https://github.com/kubernetes-sigs/aws-load-balancer-controller) , [https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases](https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases),  
[https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/)

### 1.IAM OIDC Provider 생성

IAM OIDC Provider는 기본으로 활성화되어 있지 않습니다. eksctl을 사용하여 IAM OIDC Provider를 생성합니다.

```text
eksctl utils associate-iam-oidc-provider \
    --region ${AWS_REGION} \
    --cluster eksworkshop \
    --approve
    
```

다음과 같이 IAM 서비스 메뉴에서 생성된 OIDC를 확인 할 수 있습니다.

![](../.gitbook/assets/image%20%28196%29.png)

### 2. AWS Load Balancer 컨트롤러에 대한 IAM 정책 다운로드 

ALB Load Balancer 컨트롤러에 대한 IAM정책을 다운로드 받습니다. \(이미 앞서 git에서 받은 폴더에 포함되어 있습니다.\)

```text
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.2/docs/install/iam_policy.json

```

### 3. AWSLoadBalancerControllerIAMPolicy IAM 정책 생성.

AWSLoadBalancerControllerIAMPolicy라는 IAM 정책을 생성합니다.

```text
cd ~/environment/myeks/alb-controller
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://./iam-policy.json
```

아래 처럼 결과가 출력됩니다.

```text
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

아래와 같이 IAM - 정책 메뉴에서 새롭게 생성된 정책을 확인할 수 있습니다. 

![](../.gitbook/assets/image%20%28195%29.png)

### 4. AWS Ingress Controller IAM 역할 및 Service Account 생성

이 단계에서는 AWS Ingress Controller 에 대한 IAM Roel , Service Account를 생성하고, 3번 단계에서 출력되었던 AWS Account ID를 복사해서 사용해아합니다.

```text
eksctl create iamserviceaccount \
--cluster=<cluster-name> \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--approve
```

> 참조  URL - [https://eksctl.io/usage/iamserviceaccounts/](https://eksctl.io/usage/iamserviceaccounts/)
>
> Amazon EKS는 클러스터 운영자가 AWS IAM 역할을 Kubernetes 서비스 계정에 매핑 할 수 있도록하는 [ISA \(IAM Roles for Service Accounts\)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) 를 지원합니다 .
>
> 이를 통해 EKS에서 실행되고 다른 AWS 서비스를 사용하는 앱에 대해 세분화 된 권한 관리를 제공합니다. S3, 다른 데이터 서비스 \(RDS, MQ, STS, DynamoDB\) , AWS ALB Ingress 컨트롤러 또는 ExternalDNS와 같은 Kubernetes 구성 요소를 사용하는 어플리케이션 들이 대표적입니다.IAM OIDC Provider는 기본적으로 활성화되어 있지 않습니다.

### 5. EKS Cluster에 컨트롤러 추가

Helm 또는 YAML 기반의 Menifes를 통해 클러스터에 컨트롤러를 추가합니다. 여기에서는 YAML 기반의 설치를 소개합니다.

* 먼저 Cert Manager를 설치합니다.

Kubernetes 1.16 이상에서는 아래의 명령을 실행합니다.

```text
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.2/cert-manager.yaml

```

Kubernetes 1.16 이하 버전에서는 아래 명령을 수행합니다. 이 랩에서는 이 명령을 사용하지 않습니다.

```text
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.2/cert-manager-legacy.yaml

```

* YAML 기반의 컨트롤러 추가

아래 명령을 통해 최신의 컨트롤러를 추가합니다. 이마 앞서 git을 실행하였다면 폴더에 포함되어 있습니다.

```text
cd ~/environment/myeks/alb-controller 
wget https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.2/docs/install/v2_1_2_full.yaml

```

~/environment/myeks/alb-controller/v2\_1\_2\_full.yaml 파일을 아래와 같이 수정합니다. cluster-name에 현재 적용할 EKS Cluster 이름을 입력합니다. 랩에서는 "eksworkshop" 입니다. git을 통해 받은 파일이라면 이미 수정되어 있습니다.

```text
apiVersion: apps/v1
kind: Deployment
. . . 
name: aws-load-balancer-controller
namespace: kube-system
spec:
    . . . 
    template:
        spec:
            containers:
                - args:
                    - --cluster-name=<INSERT_CLUSTER_NAME>
```

컨트롤러를 생성합니다.

```text
cd ~/environment/myeks/alb-controller
kubectl apply -f v2_1_2_full.yaml

```

아래 명령을 통해 로그를 확인 할 수 있습니다.

```text
kubectl logs -n kube-system $(kubectl get po -n kube-system | egrep -o alb-ingress[a-zA-Z0-9-]+)
```

### 6.namespace/App/Pod/Service 배포.

샘플 어플리케이션을 배포해 보겠습니다. 2048 게임 App을 Kubernetes Cluster에 넣고 Ingress 리소스를 사용하여 트래픽을 노출해 봅니다.

```text
cd ~/environment/myeks/alb-controller/
kubectl apply -f 2048_full.yaml

```

### 7.서비스 확인

ALB 생성된 것을 확인합니다.

![](../.gitbook/assets/image%20%28168%29.png)

브라우저를 열고 2040 앱 실행을 확인합니다.

![](../.gitbook/assets/image%20%2837%29.png)

k9s를 통해 App배포를 확인합니다.

```text
k9s -n 2048-game
k9s -A
```

![](../.gitbook/assets/image%20%2834%29.png)

다음 kubectl 명령으로 구성된 정보를 모두 확인 할 수 있습니다.

```text
kubectl -n kube-system get pods | grep "controller"
kubectl get ingresses.extensions -n game-2048
kubectl get svc -o wide -n game-2048
kubectl get pod -o wide -n game-2048 

```

출력 결과 예제

```text
whchoi98:~/environment/myeks/alb-controller (master) $ kubectl -n kube-system get pods | grep "controller"
aws-load-balancer-controller-86598b5d8d-8w2p2   1/1     Running   0          22h
whchoi98:~/environment/myeks/alb-controller (master) $ kubectl get ingresses.extensions -n game-2048
Warning: extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
NAME           CLASS    HOSTS   ADDRESS                                                                      PORTS   AGE
ingress-2048   <none>   *       k8s-game2048-ingress2-2810c0c2ad-98799391.ap-northeast-2.elb.amazonaws.com   80      22h
whchoi98:~/environment/myeks/alb-controller (master) $ kubectl get svc -o wide -n game-2048
NAME           TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE   SELECTOR
service-2048   NodePort   172.20.5.167   <none>        80:31574/TCP   22h   app.kubernetes.io/name=app-2048
whchoi98:~/environment/myeks/alb-controller (master) $ kubectl get pod -o wide -n game-2048 
NAME                               READY   STATUS    RESTARTS   AGE   IP              NODE                                               NOMINATED NODE   READINESS GATES
deployment-2048-79785cfdff-5mf6r   1/1     Running   0          22h   10.11.102.117   ip-10-11-110-220.ap-northeast-2.compute.internal   <none>           <none>
deployment-2048-79785cfdff-7l8t6   1/1     Running   0          22h   10.11.74.40     ip-10-11-79-51.ap-northeast-2.compute.internal     <none>           <none>
deployment-2048-79785cfdff-7mlmf   1/1     Running   0          22h   10.11.2.77      ip-10-11-1-70.ap-northeast-2.compute.internal      <none>           <none>
deployment-2048-79785cfdff-t67vn   1/1     Running   0          22h   10.11.93.140    ip-10-11-86-68.ap-northeast-2.compute.internal     <none>           <none>
deployment-2048-79785cfdff-wz69h   1/1     Running   0          22h   10.11.24.255    ip-10-11-26-173.ap-northeast-2.compute.internal    <none>           <none>
```



