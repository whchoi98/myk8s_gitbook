---
description: 'update : 2022-10-03 / 50min'
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

### 3. AWS ALB Ingress 개요.

AWS 로드 밸런서 컨트롤러는 Kubernetes 클러스터의 AWS Elastic Load Balancer를 관리합니다. AWS ALB Ingress Controller"로 알려졌으며 "AWS Load Balancer Controller"로 브랜드를 변경했습니다.

수신 리소스는 ALB를 구성하여 HTTP 또는 HTTPS 트래픽을 클러스터 내 다른 포드로 라우팅합니다. ALB 수신 컨트롤러는 Amazon EKS 클러스터에서 실행 중인 프로덕션 워크로드에서 지원됩니다.

## AWS 로드 밸런서 컨트롤러 기반 구성 <a href="#how-aws-load-balancer-controller-works" id="how-aws-load-balancer-controller-works"></a>

### 4. Ingress 동작 방식

다음 다이어그램은 이 컨트롤러가 생성하는 AWS 구성 요소를 자세히 설명합니다. 또한 수신 트래픽이 ALB에서 Kubernetes 클러스터로 이동하는 경로를 보여줍니다.

1. 컨트롤러는 API 서버의 Ingress 이벤트를 모니터링합니다. 요구 사항을 충족하는 수신 리소스를 찾으면 AWS 리소스 생성을 시작합니다.
2. 새 수신 리소스에 대해 AWS에서 ALB (ELBv2)가 생성됩니다. 이 ALB는 인터넷에 연결되거나 내부에 있을 수 있습니다. Annotation을 사용하여 생성된 서브넷을 지정할 수도 있습니다.
3. Target Group은 수신 리소스에 설명된 각 고유 Kubernetes 서비스에 대해 AWS에서 생성됩니다.
4. 수신 리소스 Annotation에 자세히 설명된 모든 포트에 대해 리스너가 생성됩니다.[ ](http://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html)포트를 지정하지 않으면 적절한 기본값( `80`또는 `443`)이 사용됩니다. Annotation을 통해 인증서를 첨부할 수도 있습니다.
5. 수신 리소스에 지정된 각 경로에 대해 규칙이 생성됩니다[. ](http://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-update-rules.html)이렇게 하면 특정 경로에 대한 트래픽이 적절한 Kubernetes 서비스로 라우팅됩니다.



![참조 - https://aws.amazon.com/ko/blogs/opensource/kubernetes-ingress-aws-alb-ingress-controller/](<../.gitbook/assets/image (207).png>)

Reference - [https://github.com/kubernetes-sigs/aws-load-balancer-controller](https://github.com/kubernetes-sigs/aws-load-balancer-controller) , [https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases](https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases),\
[https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/)

### 5. Ingress Traffic

AWS Load Balancer 컨트롤러는 두 가지 트래픽 모드를 지원합니다.

* 인스턴스 모드
* IP 모드

기본적으로 Instance Mode가 사용되며 , Annotation의  `target-type` 을 통해 모드를 선택할 수 있습니다.

**인스턴스 모드**

수신 트래픽은 ALB에서 시작하여 각 서비스의 NodePort를 통해 Kubernetes 노드에 도달합니다. 이는 수신 리소스에서 참조하는 서비스가 ALB에 도달하기 위해 에서 `type:NodePort`가 있어야 함을 의미합니다.

**IP 모드**

수신 트래픽은 ALB에서 시작하여 Kubernetes Pod에 직접 도달합니다. CNI는 ENI의 Secondary IP 주소를 통해 직접 액세스할 수 있는 POD IP를 지원해야 합니다.

### 6. ALB Ingress & AWS Load Balancer Controller 트래픽 흐름

* 외부 사용자는 ALB DNS A Record:Port 번호로 접근합니다
* ALB는 각 노드로 로드밸런싱 합니다
* Kube-API에 의해 업데이트 된 정보를 가지고 ALB에서 Loadbalancing 처리를 합니다.&#x20;

![](<../.gitbook/assets/image (245).png>)

아래와 같은 구성 단계로 ALB Loadbalancer Controller를 구성합니다.

1. IAM OIDC 공급자 생성
2. AWS Loadbalancer 컨트롤러에 대한 IAM 정책 다운로드.
3. AWSLoadBalancerControllerIAMPolicy 이름의 IAM 정책 생성.
4. AWS Load Balancer 컨트롤러에 대한 IAM역할 및 ServiceAccount 생성
5. EKS Cluster에 컨트롤러 추가 &#x20;

### 7. IAM OIDC Provider 생성

AWS IAM(Identity and Access Management)에서는 OpenID Connect(OIDC)를 사용해 연동 자격 증명을 지원하는 기능을 추가하였습니다. 이 기능을 사용하면 지원되는 자격 증명 공급자를 이용해 AWS API 호출을 인증하고 유효한 OIDC JWT(JSON WebToken)을 수신할 수 있습니다. \
이 토큰을 AWS STS `AssumeRoleWithWebIdentity` API 작업에 전달하고 IAM 임시 역할 자격 증명을 수신할 수 있습니다. 이 자격 증명을 사용하여 AWS 서비스 자원들을 Kubernetes 자원들이 사용할 수 있습니다.&#x20;

IAM OIDC Provider는 기본으로 활성화되어 있지 않습니다. eksctl을 사용하여 IAM OIDC Provider를 생성합니다.

```
source ~/.bash_profile
eksctl utils associate-iam-oidc-provider \
    --region ${AWS_REGION} \
    --cluster ${ekscluster_name} \
    --approve
    
```

다음과 같이 Cloud9 Console 또는 IAM 서비스 메뉴에서 생성된 OIDC를 확인 할 수 있습니다.

```
aws eks describe-cluster --name ${ekscluster_name} --query "cluster.identity.oidc.issuer" --output text
aws iam list-open-id-connect-providers

```

![](<../.gitbook/assets/image (481).png>)

### 8. AWS Load Balancer 컨트롤러에 대한 IAM 정책 다운로드 (생략)

ALB Load Balancer 컨트롤러에 대한 IAM정책을 다운로드 받습니다. (이미 앞서 git에서 받은 폴더에 포함되어 있습니다.)

```
## ALB Load Balancer Controller 의 IAM Policy Download
# cd ~/environment/myeks/alb-controller/
# export ALB_CONTROLLER_VERSION=2.5.2
# curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v${ALB_CONTROLLER_VERSION}/docs/install/iam_policy.json
```

### 9. AWSLoadBalancerControllerIAMPolicy IAM 정책 생성.

AWSLoadBalancerControllerIAMPolicy라는 IAM 정책을 생성합니다.

```
cd ~/environment/myeks/alb-controller
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://./iam_policy_v2.5.2.json

```

아래 처럼 결과가 출력됩니다.

```
{
    "Policy": {
        "PolicyName": "AWSLoadBalancerControllerIAMPolicy",
        "PolicyId": "ANPAQXXZSBRZZZKGZT2F6",
        "Arn": "arn:aws:iam::050989239411:policy/AWSLoadBalancerControllerIAMPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2022-12-18T15:09:27+00:00",
        "UpdateDate": "2022-12-18T15:09:27+00:00"
    }
}
```

아래와 같이 IAM - 정책 메뉴에서 새롭게 생성된 정책을 확인할 수 있습니다.&#x20;

![](<../.gitbook/assets/image (323).png>)

### 10. AWS LoadBalancer Controller IAM 역할 및 Service Account 생성

이 단계에서는 AWS LoadBalancer Controller 에 대한 Service Account를 생성하고, 앞서 생성한 IAM Role을 연결합니다.  2\~3분 정도 시간이 소요 됩니다.&#x20;

```
eksctl create iamserviceaccount \
--cluster=${ekscluster_name} \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--region ${AWS_REGION} \
--approve

```

Kubernetes에 정상적으로 Service Account가 등록되었는지 확인해 봅니다.

```
kubectl get serviceaccounts -n kube-system aws-load-balancer-controller -o yaml

```

아래와 같이 출력 예제를 확인해 볼 수 있습니다.

```
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::050989239411:role/eksctl-eksworkshop-addon-iamserviceaccount-k-Role1-1UV0H179WJOBO
  creationTimestamp: "2022-12-18T15:11:35Z"
  labels:
    app.kubernetes.io/managed-by: eksctl
  name: aws-load-balancer-controller
  namespace: kube-system
  resourceVersion: "43910"
  uid: c92e0386-cd24-43a2-a90a-eac960e27b66
secrets:
- name: aws-load-balancer-controller-token-zlq6v
```

> 참조  URL - [https://eksctl.io/usage/iamserviceaccounts/](https://eksctl.io/usage/iamserviceaccounts/)
>
> Amazon EKS는 클러스터 운영자가 AWS IAM 역할을 Kubernetes 서비스 계정에 매핑 할 수 있도록하는 IRSA[ (IAM Roles for Service Accounts)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) 를 지원합니다 .
>
> 이를 통해 EKS에서 실행되고 다른 AWS 서비스를 사용하는 앱에 대해 세분화 된 권한 관리를 제공합니다. S3, 다른 데이터 서비스 (RDS, MQ, STS, DynamoDB) , AWS ALB Ingress 컨트롤러 또는 ExternalDNS와 같은 Kubernetes 구성 요소를 사용하는 어플리케이션 들이 대표적입니다. IAM OIDC Provider는 기본적으로 활성화되어 있지 않습니다. 따라서 앞선 과정들을 수행해야 합니다

### 11. 인증서 관리자 설치

아래와 같이 Cert Manager (인증서 관리자)를 설치합니다.

```
export CERTMGR_VERSION=1.12.2
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v${CERTMGR_VERSION}/cert-manager.yaml
kubectl -n cert-manager get pods

```

{% hint style="warning" %}
cert-manager pod 3대가 모두 정상적으로 동작되는지 확인하고 , 다음 단계를 진행합니다.&#x20;
{% endhint %}

```
$ kubectl -n cert-manager get pods
NAME                                      READY   STATUS    RESTARTS   AGE
cert-manager-55649d64b4-q4vvg             1/1     Running   0          41s
cert-manager-cainjector-666db4777-zfzl5   1/1     Running   0          41s
cert-manager-webhook-6466bc8f4-zsz4w      1/1     Running   0          41s
```

### 12. AWS Loadbalancer Controller Pod 설치

Helm 기반 또는 manfest 파일을 통해 ALB Loadbalancer Controller Pod를 설치합니다. 여기에서는 Yaml을 통해 직접 설치해 봅니다. (이미 git을 통해서 다운 받았을 경우에는 생략해도 됩니다.)

```
# wget # https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.5.2/v2_5_2_full.yaml

```

AWS Loadbalancer Controller Pod의 Deployment file에 지정된 <mark style="color:red;background-color:red;">**`cluster-name`**</mark> 값을 , 현재 배포한 Cluster name으로 변경합니다. (이미 앞서 git을 통해서 다운 받았을 경우에는 생략해도 됩니다.)

```
### cluster name을 eksworkshop 또는 현재 실행 중인 Cluster name으로 변경합니다.
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

앞서서 Service Account와 IAM Role을 연결하는 작업을 이미 완료했으므로, kind: ServiceAccount 섹션은 삭제하거나 주석처리하는 것이 좋습니다.&#x20;

```
# apiVersion: v1
# kind: ServiceAccount
```

이미 git을 통해 사전에 다운로드 받아 두었습니다. 해당 AWS Loadbalancer Controller Pod의 yaml에는 Cluster Name이 eksworkshop으로 수정되어 있고 kine: ServiceAccount 섹션은 주석처리되어 있습니다. 해당 yaml을 배포합니다. pod가 정상적으로 Running 되는지 확인하고 다음 단계를 진행합니다.&#x20;

```
cd ~/environment/myeks/alb-controller
kubectl apply -f v2_5_2_full.yaml
kubectl -n kube-system get pods | grep balancer

```

### 12.Ingress Annotation&#x20;

kubernetes Ingress 및 Service Object에 Annotation을 추가하여 동작을 상세하게 지정할 수 있습니다

* [ALB Load Balancer Controller Annotation ](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/guide/ingress/annotations/)

```
    annotations:
  ## IngressGroup
    ### Ingress가 속한 그룹 이름을 지정합니다.
    alb.ingress.kubernetes.io/group.name: my-team.awesome-group
    ### IngressGroup 내의 모든 Ingress에 대한 순서를 지정합니다. default 0
    alb.ingress.kubernetes.io/group.order: '10'
  ## Traffic Listening   
    ## ALB가 수신 대기하는 데 사용한 포트를 지정합니다.
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}, {"HTTP": 8080}, {"HTTPS": 8443}]'
    ## SSLRedirect를 활성화하고 리디렉션할 SSL 포트를 지정합니다.단일 Ingress에 정의되면 IngressGroup 내의 모든 Ingress에 영향을 미칩니다.
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    ## ALB 의 IP 주소 유형 을 지정합니다 .
    alb.ingress.kubernetes.io/ip-address-type: ipv4
    ## Outpost의 ALB에 대한 고객 소유 IPv4 주소 풀을 지정합니다.
    alb.ingress.kubernetes.io/customer-owned-ipv4-pool: ipv4pool-coip-xxxxxxxx
  ## Traffic Routing
    ## 로드 밸런서에 사용할 사용자 지정 이름을 지정합니다. 32자보다 긴 이름은 오류로 처리됩니다.    
    alb.ingress.kubernetes.io/load-balancer-name: custom-name
    ## 트래픽을 포드로 라우팅하는 방법을 지정합니다.
    ## instance모드는 서비스에 대해 오픈된 NodePort 의 클러스터 내의 모든 ec2 인스턴스로 트래픽을 라우팅합니다 .
    ## instance모드 를 사용하려면 서비스 유형이 "NodePort" 또는 "LoadBalancer" 가 되어야 합니다.
    alb.ingress.kubernetes.io/target-type: instance
    ## ip모드는 트래픽을 포드 IP로 직접 라우팅합니다.
    ## CNI가 VPC CNI이어야 합니다. ENI의 Secondary IP, 즉 PoD IP를 사용하기 때문입니다.
    ## Sticky Session을 사용하려면 이 모드가 필요합니다.
    alb.ingress.kubernetes.io/target-type: ip
    ### LB 대상 그룹 등록에 포함할 노드를 지정합니다.
    alb.ingress.kubernetes.io/target-node-labels: label1=value1, label2=value2
    ### 트래픽을 포드로 라우팅할 때 사용되는 프로토콜을 지정합니다.
    alb.ingress.kubernetes.io/backend-protocol: HTTPS
    ### 트래픽을 포드로 라우팅하는 데 사용되는 애플리케이션 프로토콜을 지정
    ### HTTP2
    alb.ingress.kubernetes.io/backend-protocol-version: HTTP2
    ### GRPC
    alb.ingress.kubernetes.io/backend-protocol-version: GRPC
    ### ALB가 트래픽을 라우팅할 가용 영역을 지정
    alb.ingress.kubernetes.io/subnets: subnet-xxxx, mySubnet
    ## 리디렉션 작업과 같은 Listener에서 Custom Action 작업을 구성하는 방법을 제공합니다.
    alb.ingress.kubernetes.io/actions.${action-name}
    ## Ingress 사양의 원래 호스트/경로 조건 외에 라우팅 조건을 지정하는 방법을 제공합니다.
    alb.ingress.kubernetes.io/conditions.${conditions-name}
  ## Access Control
    ## LoadBalancer가 인터넷에 연결되는지 여부를 지정합니다.
    alb.ingress.kubernetes.io/scheme: internal
    alb.ingress.kubernetes.io/scheme: internet-facing
    ## LoadBalancer에 액세스할 수 있는 CIDR을 지정합니다.
    ## alb.ingress.kubernetes.io/security-groups지정된 경우 무시됩니다.
    alb.ingress.kubernetes.io/inbound-cidrs: 10.0.0.0/24
    ## LoadBalancer에 연결할 securityGroups를 지정합니다.securityGroups의 이름 또는 ID가 모두 지원
    alb.ingress.kubernetes.io/security-groups: sg-xxxx, nameOfSg1, nameOfSg2
  ## Authentication
    ## 인증은 HTTPS 리스너에 대해서만 지원됩니다.
    ## 대상에 대한 인증 유형을 지정합니다.
    alb.ingress.kubernetes.io/auth-type: cognito
    ## oidc idp 구성을 지정합니다.
    alb.ingress.kubernetes.io/auth-idp-oidc: '{"issuer":"https://example.com","authorizationEndpoint":"https://authorization.example.com","tokenEndpoint":"https://token.example.com","userInfoEndpoint":"https://userinfo.example.com","secretName":"my-k8s-secret"}'
    ## 사용자가 인증되지 않은 경우 동작을 지정합니다.
    alb.ingress.kubernetes.io/auth-on-unauthenticated-request: authenticate
    ## 공백으로 구분된 목록에서 IDP(cognito 또는 oidc)에서 요청할 사용자 클레임 집합을 지정합니다.
    alb.ingress.kubernetes.io/auth-scope: 'email openid'
    ## 세션 정보를 유지하는 데 사용되는 쿠키의 이름을 지정합니다.
    alb.ingress.kubernetes.io/auth-session-cookie: custom-cookie
    ## 인증 세션의 최대 지속 시간을 초 단위로 지정합니다.
    alb.ingress.kubernetes.io/auth-session-timeout: '86400'
  ## Health Check
    ## Target에서 상태 확인을 수행할 때 사용되는 프로토콜을 지정합니다.
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTPS
    ## Target에서 상태 확인을 수행할 때 사용되는 포트를 지정합니다.
    ## 상태 확인 포트를 트래픽 포트로 설정
    alb.ingress.kubernetes.io/healthcheck-port: traffic-port
    ## 상태 확인 포트를 명명된 포트의 NodePort(target-type=instance일 때) 또는 TargetPort(target-type=ip일 때)로 설정합니다.
    alb.ingress.kubernetes.io/healthcheck-port: my-port
    ## 상태 확인 포트를 80/tcp로 설정
    alb.ingress.kubernetes.io/healthcheck-port: '80'
    ## 대상에서 상태 확인을 수행할 때 HTTP 경로를 지정합니다.
    ## HTTP
    alb.ingress.kubernetes.io/healthcheck-path: /ping
    ## GRPC
    alb.ingress.kubernetes.io/healthcheck-path: /package.service/method
    ## 개별 target 의 상태 확인 간격(초)을 지정합니다. default 15
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '10'
    ## Target 에서 응답이 없으면 상태 확인이 실패했음을 의미하는 시간 초과(초)를 지정합니다. default 5
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '8'
    ## 지정된 상태 확인 경로에 대해 상태 확인을 수행할 때 예상해야 하는 HTTP 상태 코드를 지정합니다.
    alb.ingress.kubernetes.io/success-codes: 200-300
    ## 비정상 대상을 정상으로 간주하기 전에 필요한 연속적인 상태 확인 성공을 지정합니다. default 2
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    ## 대상을 비정상으로 간주하기 전에 필요한 연속적인 상태 확인 실패를 지정합니다. default 2
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'
  ## SSL
    ## AWS Certificate Manager 에서 관리하는 하나 이상의 인증서의 ARN을 지정합니다.
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-west-2:xxxxx:certificate/xxxxxxx
    ## 프로토콜과 암호를 제어할 수 있도록 ALB에 할당해야 하는 보안 정책 을 지정합니다 .
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-1-2017-01
  ## Custom attributes
    ## ALB에 적용해야 하는 로드 밸런서 속성 을 지정합니다.
    ## s3에 대한 액세스 로그 활성화
    alb.ingress.kubernetes.io/load-balancer-attributes: access_logs.s3.enabled=true,access_logs.s3.bucket=my-access-log-bucket,access_logs.s3.prefix=my-app
    ## 삭제 방지 활성화
    alb.ingress.kubernetes.io/load-balancer-attributes: deletion_protection.enabled=true
    ## 유효하지 않은 헤더 필드 제거 활성화
    alb.ingress.kubernetes.io/load-balancer-attributes: routing.http.drop_invalid_header_fields.enabled=true
    ## http2 지원 활성화
    alb.ingress.kubernetes.io/load-balancer-attributes: routing.http2.enabled=true
    ## idle_timeout 지연을 600초로 설정
    alb.ingress.kubernetes.io/load-balancer-attributes: idle_timeout.timeout_seconds=600
    ## target-group 에 적용해야 하는 속성을 지정합니다 .
    ## slow start 지속시간을 설정
    alb.ingress.kubernetes.io/target-group-attributes: slow_start.duration_seconds=30
    ## 등록 취소 지연 시간 설정
    alb.ingress.kubernetes.io/target-group-attributes: deregistration_delay.timeout_seconds=30
    ## Stickiness 세션 활성화, alb.ingress.kubernetes.io/target-type 이 반드시 IP 모드이어야 합니다.
    alb.ingress.kubernetes.io/target-group-attributes: stickiness.enabled=true,stickiness.lb_cookie.duration_seconds=60
    alb.ingress.kubernetes.io/target-type: ip
    ## LB 알고리즘을 least outstanding requests 로 설정.
    alb.ingress.kubernetes.io/target-group-attributes: load_balancing.algorithm.type=least_outstanding_requests
  ### Resource Tag
    ## 생성한 AWS 리소스(ALB/TargetGroups/SecurityGroups/Listener/ListenerRule)에 다음 태그를 자동으로 적용합니다.
    elbv2.k8s.aws/cluster: ${clusterName}
    ingress.k8s.aws/stack: ${stackID}
    ingress.k8s.aws/resource: ${resourceID}
    ## 생성된 AWS 리소스에 적용할 추가 태그를 지정
    alb.ingress.kubernetes.io/tags: Environment=dev,Team=test
  ### Add On
    ## alb.ingress.kubernetes.io/waf-acl-idAmzon WAF 웹 ACL의 식별자를 지정합니다.
    ## Regional WAF만 지원
    alb.ingress.kubernetes.io/waf-acl-id: 499e8b99-6671-4614-a86d-adb1810b7fbe
    ## Regional WAFv2 만 지원
    alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:us-west-2:xxxxx:regional/webacl/xxxxxxx/3ab78708-85b0-49d3-b4e1-7a9615a6613b
    ## 로드 밸런서에 대한 AWS Shield Advanced 보호를 켜거나 끕니다.
    alb.ingress.kubernetes.io/shield-advanced-protection: 'true'
```

### 13.ALB Ingress Traffic 흐름 확인



ALB Ingress를 시험하기 위해 아래와 같이 namespace와  pod,service를 배포합니다.

```
## alb-test-01 namespace를 생성하고, pod, service를 배포 
kubectl create namespace alb-ing-01
kubectl -n alb-ing-01 apply -f ~/environment/myeks/ingress/v1.22/alb-ing-01.yaml
kubectl -n alb-ing-01 apply -f ~/environment/myeks/ingress/v1.22/alb-ing-01-ingress.yaml
kubectl -n alb-ing-01 apply -f ~/environment/myeks/ingress/v1.22/alb-ing-01-service.yaml

```

아래와 같은 명령으로 결과를 확인 할 수 있습니다.

```
kubectl -n alb-ing-01 get pod -o wide
kubectl -n alb-ing-01 get service -o wide 
kubectl -n alb-ing-01 get ingress -o wide 

```

아래와 같은 결과를 확인하고 ingress LB의 외부 A Record를 확인합니다.&#x20;

```
kubectl -n alb-ing-01 get ingress alb-ing-01 | tail -n 1 | awk '{ print "ALB-INGRESS URL = http://"$4}' 

```

해당 A Record를 Cloud9 IDE Terminal에서  Curl을 통해 접속하거나 브라우저에서 접속해 봅니다.

```
$ kubectl -n alb-ing-01 get pod -o wide
NAME                          READY   STATUS    RESTARTS   AGE     IP              NODE                                               NOMINATED NODE   READINESS GATES
alb-ing-01-ffd85d89f-5s66x   1/1     Running   0          6m15s   10.11.92.148    ip-10-11-94-22.ap-northeast-2.compute.internal     <none>           <none>
alb-ing-01-ffd85d89f-sl5bd   1/1     Running   0          6m14s   10.11.88.200    ip-10-11-94-22.ap-northeast-2.compute.internal     <none>           <none>
alb-ing-01-ffd85d89f-td4tv   1/1     Running   0          6m14s   10.11.107.214   ip-10-11-108-153.ap-northeast-2.compute.internal   <none>           <none>
$ kubectl -n alb-ing-01 get service -o wide
NAME          TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE   SELECTOR
alb-ing-01   NodePort   172.20.121.174   <none>        8080:30299/TCP   12m   app=alb-test-01
$ kubectl -n alb-ing-01 get ingress -o wide
NAME          CLASS    HOSTS   ADDRESS                                                                      PORTS   AGE
alb-ing-01   <none>   *       k8s-alb-ing-01-alb-ing-01-1aa7c83247-45114489.ap-northeast-2.elb.amazonaws.com   80      13m
```

아래와 같이 배포된 pod에 접속을 편리하게 하기 위해 Cloud9 IDE terminal Shell에 등록 합니다.

```
export alb_ing_01_Pod01=$(kubectl -n alb-ing-01 get pod -o wide | awk 'NR==2' | awk '/alb-ing-01/{print $1 } ')
export alb_ing_01_Pod02=$(kubectl -n alb-ing-01 get pod -o wide | awk 'NR==3' | awk '/alb-ing-01/{print $1 } ')
export alb_ing_01_Pod03=$(kubectl -n alb-ing-01 get pod -o wide | awk 'NR==4' | awk '/alb-ing-01/{print $1 } ')
echo "export alb_ing_01_Pod01=${alb_ing_01_Pod01}" | tee -a ~/.bash_profile
echo "export alb_ing_01_Pod02=${alb_ing_01_Pod02}" | tee -a ~/.bash_profile
echo "export alb_ing_01_Pod03=${alb_ing_01_Pod03}" | tee -a ~/.bash_profile
source ~/.bash_profile

```

alb-ing-01에 접속해서 아래와 같이 확인해 봅니다.

```
#K9으로 접속 할 경우
k9s -n alb-ing-01
# Cloud9 에서 접속할 경
kubectl -n alb-ing-01 exec -it $alb_ing_01_Pod01 -- /bin/sh

```

```
#Container shell terminal

tcpdump -s 0 -A 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'

```

Pod에서 TCPDump로 확인하면 정상적으로 Pakcet이 덤프되고 X-Forwarded를 통해 Source IP를 확인할 수 있습니다.&#x20;

```
## Cloud9 IDE
export alb_ing_01_svc_name=$(kubectl -n alb-ing-01 get ingress alb-ing-01 --output jsonpath='{.status.loadBalancer.ingress[*].hostname}')
echo "export alb_ing_01_svc_name=${alb_ing_01_svc_name}" | tee -a ~/.bash_profile
curl ${alb_ing_01_svc_name}

```

```
### Cotainer tcpdump 예제
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

<figure><img src="../.gitbook/assets/image (307).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (164).png" alt=""><figcaption></figcaption></figure>

![](<../.gitbook/assets/image (498).png>)



ALB Ingress Controller는 Target Group을 IP기반으로 Pod에 직접 배포할 수 있습니다.&#x20;

![](<../.gitbook/assets/image (440).png>)

IP Mode로 아래와 같이 배포해 봅니다.&#x20;

```
## alb-test-02 namespace를 생성하고, pod, service를 배포 
kubectl create namespace alb-ing-02
kubectl -n alb-ing-02 apply -f ~/environment/myeks/ingress/v1.22/alb-ing-02.yaml
kubectl -n alb-ing-02 apply -f ~/environment/myeks/ingress/v1.22/alb-ing-02-ingress.yaml
kubectl -n alb-ing-02 apply -f ~/environment/myeks/ingress/v1.22/alb-ing-02-service.yaml

```

아래와 같이 구성되었습니다.

![](<../.gitbook/assets/image (324).png>)

아래와 같은 명령으로 결과를 확인 할 수 있습니다.

```
kubectl -n alb-ing-02 get pod -o wide
kubectl -n alb-ing-02 get service -o wide 
kubectl -n alb-ing-02 get ingress -o wide 

```

아래와 같이 ALB 구성의 Target Group이 PoD IP로 등록됩니다.&#x20;

<figure><img src="../.gitbook/assets/image (122).png" alt=""><figcaption></figcaption></figure>

아래와 같은 결과를 확인하고 ingress LB의 외부 A Record를 확인합니다.&#x20;

```
kubectl -n alb-ing-02 get ingress alb-ing-02 | tail -n 1 | awk '{ print "ALB-INGRESS URL = http://"$4}' 

```

해당 A Record를 Cloud9 IDE Terminal에서  Curl을 통해 접속하거나 브라우저에서 접속해 봅니다.

```
$ kubectl -n alb-ing-02 get pod -o wide
NAME                         READY   STATUS    RESTARTS   AGE     IP             NODE                                              NOMINATED NODE   READINESS GATES
alb-ing-02-68795df47-7wt4j   1/1     Running   0          6m23s   10.11.55.70    ip-10-11-58-9.ap-northeast-2.compute.internal     <none>           <none>
alb-ing-02-68795df47-kcmps   1/1     Running   0          6m23s   10.11.68.116   ip-10-11-69-251.ap-northeast-2.compute.internal   <none>           <none>
alb-ing-02-68795df47-wp7jr   1/1     Running   0          6m23s   10.11.94.155   ip-10-11-81-171.ap-northeast-2.compute.internal   <none>           <none>
$ kubectl -n alb-ing-02 get ingress -o wide
NAME         CLASS    HOSTS   ADDRESS                                                  PORTS   AGE
alb-ing-02   <none>   *       alb-ing-02-1854312130.ap-northeast-2.elb.amazonaws.com   80      7m14s
```

아래와 같이 배포된 pod에 접속을 편리하게 하기 위해 Cloud9 IDE terminal Shell에 등록 합니다.

```
export alb_ing_02_Pod01=$(kubectl -n alb-ing-02 get pod -o wide | awk 'NR==2' | awk '/alb-ing-02/{print $1 } ')
export alb_ing_02_Pod02=$(kubectl -n alb-ing-02 get pod -o wide | awk 'NR==3' | awk '/alb-ing-02/{print $1 } ')
export alb_ing_02_Pod03=$(kubectl -n alb-ing-02 get pod -o wide | awk 'NR==4' | awk '/alb-ing-02/{print $1 } ')
echo "export alb_ing_02_Pod01=${alb_ing_02_Pod01}" | tee -a ~/.bash_profile
echo "export alb_ing_02_Pod02=${alb_ing_02_Pod02}" | tee -a ~/.bash_profile
echo "export alb_ing_02_Pod03=${alb_ing_02_Pod03}" | tee -a ~/.bash_profile
source ~/.bash_profile

```

alb-ing-01에 접속해서 아래와 같이 확인해 봅니다.

```
kubectl -n alb-ing-02 exec -it $alb_ing_02_Pod01 -- /bin/sh
tcpdump -s 0 -A 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'

```

Pod에서 TCPDump로 확인하면 정상적으로 Pakcet이 덤프되고 X-Forwarded를 통해 Source IP를 확인할 수 있습니다.&#x20;

```
## Cloud9 IDE
export alb_ing_02_svc_name=$(kubectl -n alb-ing-02 get ingress alb-ing-02 --output jsonpath='{.status.loadBalancer.ingress[*].hostname}')
echo "export alb_ing_02_svc_name=${alb_ing_02_svc_name}" | tee -a ~/.bash_profile
curl ${alb_ing_02_svc_name}

```

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
apiVersion: networking.k8s.io/v1
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
          - pathType: ImplementationSpecific
            backend:
              service:
                name: "ecsdemo-frontend"
                port:
                  number: 80
EoF

```

yaml 파일을 배포하고, 서비스를 확인합니다.

```
kubectl create namespace alb-test
kubectl apply -f ~/environment/myeks/alb-controller/alb_front_full.yaml 
kubectl -n alb-test get pods,svc,ingress

```

아래와 같은 결과를 확인 할 수 있습니다.

```
$ kubectl -n alb-test get pods,svc,ingress
NAME                                   READY   STATUS    RESTARTS   AGE
pod/ecsdemo-frontend-6cc7bb877-29rps   1/1     Running   0          101s
pod/ecsdemo-frontend-6cc7bb877-2t8ht   1/1     Running   0          101s
pod/ecsdemo-frontend-6cc7bb877-4cpwn   1/1     Running   0          101s

NAME                       TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/ecsdemo-frontend   NodePort   172.20.127.150   <none>        80:31576/TCP   101s

NAME                                         CLASS    HOSTS   ADDRESS                                                                       PORTS   AGE
ingress.networking.k8s.io/ecsdemo-frontend   <none>   *       k8s-albtest-ecsdemof-ee882fe5b1-2046431973.ap-northeast-2.elb.amazonaws.com   80      61s
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

![](<../.gitbook/assets/image (49).png>)

### 14. 2048 Game 어플리케이션 배포

아래와 같이 새로운 aplication game을 실행합니다.&#x20;

```
kubectl apply -f ~/environment/myeks/alb-controller/2048_full.yaml

```

yaml 파일을 배포하고, 서비스를 확인합니다.

```
kubectl -n game-2048 get pods,svc,ingress

```

아래와 같은 결과를 확인 할 수 있습니다.

```
$ kubectl -n game-2048 get pods,svc,ingress
NAME                                   READY   STATUS    RESTARTS   AGE
pod/deployment-2048-79785cfdff-5w9vw   1/1     Running   0          2m1s
pod/deployment-2048-79785cfdff-6h9wt   1/1     Running   0          2m1s
pod/deployment-2048-79785cfdff-ghrx7   1/1     Running   0          2m1s
pod/deployment-2048-79785cfdff-kq9bd   1/1     Running   0          2m1s
pod/deployment-2048-79785cfdff-rhpmt   1/1     Running   0          2m1s

NAME                   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/service-2048   NodePort   172.20.51.224   <none>        80:31181/TCP   2m1s

NAME                                     CLASS    HOSTS   ADDRESS                                                                        PORTS   AGE
ingress.networking.k8s.io/ingress-2048   <none>   *       k8s-game2048-ingress2-2810c0c2ad-1261616116.ap-northeast-2.elb.amazonaws.com   80      2m1s
```

아래 명령을 통해서 게임앱 URL을 확인하고에 브라우저를 통해 접속해 봅니다.

```
kubectl -n game-2048 get ingress ingress-2048 | tail -n 1 | awk '{ print "game-2048 URL = http://"$4 }'

```

![](<../.gitbook/assets/image (193).png>)

