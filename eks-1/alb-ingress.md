# ALB Ingress 배포

[Kubernetes용 AWS ALB 수신 컨트롤러](https://github.com/kubernetes-sigs/aws-alb-ingress-controller)는 `kubernetes.io/ingress.class: alb` 주석과 클러스터에 수신 리소스가 생성될 때마다 Application Load Balancer\(ALB\) 및 필수 지원 AWS 리소스가 생성되도록 트리거하는 컨트롤러입니다. 수신 리소스는 ALB를 구성하여 HTTP 또는 HTTPS 트래픽을 클러스터 내 다른 포드로 라우팅합니다. ALB 수신 컨트롤러는 Amazon EKS 클러스터에서 실행 중인 프로덕션 워크로드에서 지원됩니다.



1.IAM Policy 생성

eksctl을 사용하여 ALB Ingress Controller를 위한 IAM Policy를 생성합니다.

> 참조 URL - [https://eksctl.io/usage/iamserviceaccounts/](https://eksctl.io/usage/iamserviceaccounts/)

```text
eksctl utils associate-iam-oidc-provider --cluster=eksworkshop --approve
```

출력 결과 예시

```text
whchoi98:~/environment $ eksctl utils associate-iam-oidc-provider --cluster=eksworkshop --approve
[ℹ]  eksctl version 0.23.0
[ℹ]  using region ap-northeast-2
[ℹ]  will create IAM Open ID Connect provider for cluster "eksworkshop" in "ap-northeast-2"
[✔]  created IAM Open ID Connect provider for cluster "eksworkshop" in "ap-northeast-2"
```

이전 랩에서 수행한 $ALB\_INGRESS\_VERSION 변수에 값이 정상적으로 입력되었는지 확인합니다.

```text
echo $ALB_INGRESS_VERSION
```

출력 결과 예시

```text
whchoi98:~/environment $ echo $ALB_INGRESS_VERSION
v1.1.8
```

2. RBAC 역할 생성과 바인딩

ALB Ingress 컨트롤러에 필요한 관련 RBAC 역할을 생성하고 바인딩합니다.

```text
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/${ALB_INGRESS_VERSION}/docs/examples/rbac-role.yaml
```

출력 결과 예시

```text
whchoi98:~/environment $ kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/${ALB_INGRESS_VERSION}/docs/examples/rbac-role.yaml
clusterrole.rbac.authorization.k8s.io/alb-ingress-controller created
clusterrolebinding.rbac.authorization.k8s.io/alb-ingress-controller created
serviceaccount/alb-ingress-controller created
```

3.ALBIngressControllerIAMPolicy 정책 생

`ALBIngressControllerIAMPolicy`라는 IAM 정책을 만듭니다.

```text
aws iam create-policy \
   --policy-name ALBIngressControllerIAMPolicy \
   --policy-document https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/${ALB_INGRESS_VERSION}/docs/examples/iam-policy.json
```

PolicyARN 변수에 생성된 PolicyARN 값을 저장합니다.

```text
export PolicyARN=$(aws iam list-policies --query 'Policies[?PolicyName==`ALBIngressControllerIAMPolicy`].Arn' --output text)
```

PolicyARN 변수에 저장된 값을 확인합니다.

```text
whchoi98:~/environment $ echo $PolicyARN 
arn:aws:iam::909121566064:policy/ALBIngressControllerIAMPolicy
```



