# ALB Ingress 배포

## ALB Ingress 개요.

[Kubernetes용 AWS ALB 수신 컨트롤러](https://github.com/kubernetes-sigs/aws-alb-ingress-controller)는 `kubernetes.io/ingress.class: alb` 주석과 클러스터에 수신 리소스가 생성될 때마다 Application Load Balancer\(ALB\) 및 필수 지원 AWS 리소스가 생성되도록 트리거하는 컨트롤러입니다. 수신 리소스는 ALB를 구성하여 HTTP 또는 HTTPS 트래픽을 클러스터 내 다른 포드로 라우팅합니다. ALB 수신 컨트롤러는 Amazon EKS 클러스터에서 실행 중인 프로덕션 워크로드에서 지원됩니다.

## ALB Ingress 구성

1. **ALB Ingress Controller 를 위한 IAM Policy 생성**
2. **RABAC 역할 생성과 바인딩**
3. **ALBIngress Controller IAM Policy 정책 부여**
4. **ALB Ingress 컨트롤러 포드에 권한 부여.**
5. **ALB Ingress Controller 포드 배포**
6. **샘플 App/namespace/Pod/Service 배포**
7. \*\*\*\*

### 1.IAM Policy 생성

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

### 2. RBAC 역할 생성과 바인딩

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

### 3.ALBIngressControllerIAMPolicy 정책 생성.

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

### 4.ALB Ingress 컨트롤러 포드에 권한 부여.

eksctl을 사용하여 AWS ALB Ingress 컨르톨러를 실행하는 포드에 대해 Kubernetes 서비스 계정과 IAM 역할을 생성합니다. \(3분 정도 소요됩니다.\)

```text
eksctl create iamserviceaccount --cluster=eksworkshop --namespace=kube-system --name=alb-ingress-controller --attach-policy-arn=$PolicyARN --override-existing-serviceaccounts --approve
```

출력 결과 예시

```text
whchoi98:~/environment $ eksctl create iamserviceaccount --cluster=eksworkshop --namespace=kube-system --name=alb-ingress-controller --attach-policy-arn=$PolicyARN --override-existing-serviceaccounts --approve
[ℹ]  eksctl version 0.23.0
[ℹ]  using region ap-northeast-2
[ℹ]  1 iamserviceaccount (kube-system/alb-ingress-controller) was included (based on the include/exclude rules)
[!]  metadata of serviceaccounts that exist in Kubernetes will be updated, as --override-existing-serviceaccounts was set
[ℹ]  1 task: { 2 sequential sub-tasks: { create IAM role for serviceaccount "kube-system/alb-ingress-controller", create serviceaccount "kube-system/alb-ingress-controller" } }
[ℹ]  building iamserviceaccount stack "eksctl-eksworkshop-addon-iamserviceaccount-kube-system-alb-ingress-controller"
[ℹ]  deploying stack "eksctl-eksworkshop-addon-iamserviceaccount-kube-system-alb-ingress-controller"
[ℹ]  serviceaccount "kube-system/alb-ingress-controller" already exists
[ℹ]  updated serviceaccount "kube-system/alb-ingress-controller"
```

### 5.ALB Ingress Controller 포드를 배포

AWS ALB Ingress 컨트롤러를 배포합니다.

```text
curl -sS "https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/${ALB_INGRESS_VERSION}/docs/examples/alb-ingress-controller.yaml" \
    | sed 's/# - --cluster-name=devCluster/- --cluster-name=eksworkshop/g' \
    | kubectl apply -f -
```

{% hint style="warning" %}
sample yaml에는 Cluster name이 devCluster로 되어 있으므로, 이것을 생성되어 있는 Cluster name = eksworkshop으로 대체합니다.
{% endhint %}

alb-ingress-controller.yaml 소스 참조

```text
# Application Load Balancer (ALB) Ingress Controller Deployment Manifest.
# This manifest details sensible defaults for deploying an ALB Ingress Controller.
# GitHub: https://github.com/kubernetes-sigs/aws-alb-ingress-controller
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: alb-ingress-controller
  name: alb-ingress-controller
  # Namespace the ALB Ingress Controller should run in. Does not impact which
  # namespaces it's able to resolve ingress resource for. For limiting ingress
  # namespace scope, see --watch-namespace.
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: alb-ingress-controller
  template:
    metadata:
      labels:
        app.kubernetes.io/name: alb-ingress-controller
    spec:
      containers:
        - name: alb-ingress-controller
          args:
            # Limit the namespace where this ALB Ingress Controller deployment will
            # resolve ingress resources. If left commented, all namespaces are used.
            # - --watch-namespace=your-k8s-namespace

            # Setting the ingress-class flag below ensures that only ingress resources with the
            # annotation kubernetes.io/ingress.class: "alb" are respected by the controller. You may
            # choose any class you'd like for this controller to respect.
            - --ingress-class=alb

            # REQUIRED
            # Name of your cluster. Used when naming resources created
            # by the ALB Ingress Controller, providing distinction between
            # clusters.
            # - --cluster-name=devCluster

            # AWS VPC ID this ingress controller will use to create AWS resources.
            # If unspecified, it will be discovered from ec2metadata.
            # - --aws-vpc-id=vpc-xxxxxx

            # AWS region this ingress controller will operate in.
            # If unspecified, it will be discovered from ec2metadata.
            # List of regions: http://docs.aws.amazon.com/general/latest/gr/rande.html#vpc_region
            # - --aws-region=us-west-1

            # Enables logging on all outbound requests sent to the AWS API.
            # If logging is desired, set to true.
            # - --aws-api-debug

            # Maximum number of times to retry the aws calls.
            # defaults to 10.
            # - --aws-max-retries=10
          env:
            # AWS key id for authenticating with the AWS API.
            # This is only here for examples. It's recommended you instead use
            # a project like kube2iam for granting access.
            # - name: AWS_ACCESS_KEY_ID
            #   value: KEYVALUE

            # AWS key secret for authenticating with the AWS API.
            # This is only here for examples. It's recommended you instead use
            # a project like kube2iam for granting access.
            # - name: AWS_SECRET_ACCESS_KEY
            #   value: SECRETVALUE
          # Repository location of the ALB Ingress Controller.
          image: docker.io/amazon/aws-alb-ingress-controller:v1.1.8
      serviceAccountName: alb-ingress-controller
```

배포가 완료되고 컨트롤러가 정상적으로 시작되었는지 확인합니다.

```text
kubectl logs -n kube-system $(kubectl get po -n kube-system | egrep -o alb-ingress[a-zA-Z0-9-]+)
```

아래와 같은 출력결과물을 얻었다면 성공적으로 배포된 것입니다.

```text
whchoi98:~/environment $ kubectl logs -n kube-system $(kubectl get po -n kube-system | egrep -o alb-ingress[a-zA-Z0-9-]+)
-------------------------------------------------------------------------------
AWS ALB Ingress controller
  Release:    v1.1.8
  Build:      git-ec387ad1
  Repository: https://github.com/kubernetes-sigs/aws-alb-ingress-controller.git
-------------------------------------------------------------------------------
```



### 6.namespace/App/Pod/Service 배포.

샘플 어플리케이션을 배포해 보겠습니다. 2048 게임 App을 Kubernetes Cluster에 넣고 Ingress 리소스를 사용하여 트래픽을 노출해 봅니다.

```text
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/${ALB_INGRESS_VERSION}/docs/examples/2048/2048-namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/${ALB_INGRESS_VERSION}/docs/examples/2048/2048-deployment.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/${ALB_INGRESS_VERSION}/docs/examples/2048/2048-service.yaml
```

2048-namespace.yaml 소스 참조.

```text
apiVersion: v1
kind: Namespace
metadata:
  name: "2048-game"
```

2048-deployment.yaml 소스 참조.

```text
apiVersion: apps/v1
kind: Deployment
metadata:
  name: "2048-deployment"
  namespace: "2048-game"
spec:
  selector:
    matchLabels:
      app: "2048"
  replicas: 5
  template:
    metadata:
      labels:
        app: "2048"
    spec:
      containers:
      - image: alexwhen/docker-2048
        imagePullPolicy: Always
        name: "2048"
        ports:
        - containerPort: 80
```

2048-service.yaml 소스 참조.

```text
apiVersion: v1
kind: Service
metadata:
  name: "service-2048"
  namespace: "2048-game"
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: NodePort
  selector:
    app: "2048"
```



### 7. ALB Ingress 배포

2048게임 App을 위한 ingress 리소스를 배포합니다.

```text
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/${ALB_INGRESS_VERSION}/docs/examples/2048/2048-ingress.yaml
```

2048-ingress.yaml 소스 참조.

```text
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: "2048-ingress"
  namespace: "2048-game"
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
  labels:
    app: 2048-ingress
spec:
  rules:
    - http:
        paths:
          - path: /*
            backend:
              serviceName: "service-2048"
              servicePort: 80
```

생성된 리소스를 확인합니다.

```text
kubectl get ingress/2048-ingress -n 2048-game
```

출력 결과 예시

```text
whchoi98:~/environment $ kubectl get ingress/2048-ingress -n 2048-game
NAME           HOSTS   ADDRESS                                                                       PORTS   AGE
2048-ingress   *       546056ac-2048game-2048ingr-6fa0-1286541160.ap-northeast-2.elb.amazonaws.com   80      12m
```



