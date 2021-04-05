---
description: 'update : 2020-04-04 / 1h 30min'
---

# Loadbalancer 기반 배포

## Loadbalancer 서비스 타입 소개.

Loadbalancer 기반의 서비스 타입은 현재 CLB \(Classic Load Balancer\)와 NLB\(Network Load Balancer\) 2가지 타입을 지원하고 있습니다. 모두 Port 기반의 LB를 제공하고 있으며, Kubernetes 의 Node와 Service 전면에서 서비스를 제공합니다.

## CLB Loadbalancer 서비스 기반 구성

다음과 같은 구성을 통해서 CLB 서비스를 구현해 봅니다. 

![CLB &#xAE30;&#xBC18; LAB &#xAD6C;&#xC131;&#xB3C4;](../.gitbook/assets/image%20%28171%29.png)

* namespace : clb-test
* ecsdemo-frontend service type : LoadbBlancer
* ecsdemo-crystal service type: Cluster-IP \(Default\)
* ecsdemo-nodejs service type: Cluster-IP \(Default\)

### 1.배포용 yaml 복제.

LAB에서 사용할 App을 복제합니다.

```text
cd ~/environment
git clone https://github.com/brentley/ecsdemo-frontend.git
git clone https://github.com/brentley/ecsdemo-nodejs.git
git clone https://github.com/brentley/ecsdemo-crystal.git

```

정상적으로 복제 이후 Cloud9에서 아래와 같이 확인됩니다.

![](../.gitbook/assets/image%20%2818%29.png)

아래와 같이 새로운 deployment, service를 복사합니다.

```text
 cd ~/environment/
 cp ./ecsdemo-frontend/kubernetes/deployment.yaml ./ecsdemo-frontend/kubernetes/clb_deployment.yaml
 cp ./ecsdemo-frontend/kubernetes/service.yaml ./ecsdemo-frontend/kubernetes/clb_service.yaml
 cp ./ecsdemo-crystal/kubernetes/deployment.yaml ./ecsdemo-crystal/kubernetes/clb_deployment.yaml
 cp ./ecsdemo-crystal/kubernetes/service.yaml ./ecsdemo-crystal/kubernetes/clb_service.yaml
 cp ./ecsdemo-nodejs/kubernetes/deployment.yaml ./ecsdemo-nodejs/kubernetes/clb_deployment.yaml
 cp ./ecsdemo-nodejs/kubernetes/service.yaml ./ecsdemo-nodejs/kubernetes/clb_service.yaml
 
```

### 2. Yaml 변경

~/environment/ecsdemo-frontend/kubernetes/ecsdemo-frontend/clb\_deployment.yaml 파일은은 다음과 같이 변경합니다.

```text
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecsdemo-frontend
  labels:
    app: ecsdemo-frontend
#name space change 
  namespace: clb-test
spec:
  replicas: 1
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
#Container URL change.
        - name: CRYSTAL_URL
          value: "http://ecsdemo-crystal.clb-test.svc.cluster.local/crystal"
        - name: NODEJS_URL
          value: "http://ecsdemo-nodejs.clb-test.svc.cluster.local/"
#add nodeSelector
      nodeSelector:
        nodegroup-type: "frontend-workloads"
```

~/environment/ecsdemo-frontend/kubernetes/ecsdemo-frontend/clb\_service.yaml은 다음과 같이 변경합니다.

```text
apiVersion: v1
kind: Service
metadata:
  name: ecsdemo-frontend
#name space change 
  namespace: clb-test
spec:
  selector:
    app: ecsdemo-frontend
#Service Type change
  type: LoadBalancer
  ports:
   -  protocol: TCP
      port: 80
      targetPort: 3000
```

~/environment/ecsdemo-frontend/kubernetes/ecsdemo-crystal/clb\_deployment.yaml은 다음과 같이 변경합니다.

```text
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecsdemo-crystal
  labels:
    app: ecsdemo-crystal
#name space change 
  namespace: clb-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ecsdemo-crystal
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: ecsdemo-crystal
    spec:
      containers:
      - image: brentley/ecsdemo-crystal:latest
        imagePullPolicy: Always
        name: ecsdemo-crystal
        ports:
        - containerPort: 3000
          protocol: TCP
#add nodeSelector
      nodeSelector:
        nodegroup-type: "backend-workloads"

```

~/environment/ecsdemo-frontend/kubernetes/ecsdemo-crystal/clb\_service.yaml은 다음과 같이 변경합니다.

```text
apiVersion: v1
kind: Service
metadata:
  name: ecsdemo-crystal
#name space change 
  namespace: clb-test
spec:
  selector:
    app: ecsdemo-crystal
  ports:
   -  protocol: TCP
      port: 80
      targetPort: 3000
```

~/environment/ecsdemo-frontend/kubernetes/ecsdemo-nodejs/clb\_deployment.yaml은 다음과 같이 변경합니다.

```text
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecsdemo-nodejs
  labels:
    app: ecsdemo-nodejs
#name space change 
  namespace: clb-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ecsdemo-nodejs
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: ecsdemo-nodejs
    spec:
      containers:
      - image: brentley/ecsdemo-nodejs:latest
        imagePullPolicy: Always
        name: ecsdemo-nodejs
        ports:
        - containerPort: 3000
          protocol: TCP
#add nodeSelector
      nodeSelector:
        nodegroup-type: "backend-workloads"
```

~/environment/ecsdemo-frontend/kubernetes/ecsdemo-nodejs/clb\_service.yaml은 다음과 같이 변경합니다.

```text
apiVersion: v1
kind: Service
metadata:
  name: ecsdemo-nodejs
#name space change 
  namespace: clb-test
spec:
  selector:
    app: ecsdemo-nodejs
  ports:
   -  protocol: TCP
      port: 80
      targetPort: 3000

```

### 3. FrontEnd 어플리케이션 배포와 서비스 구성.

기본 Loadbalacer 구성을 위해 새로운 Namespace를 생성합니다.

```text
kubectl create namespace clb-test

```

어플리케이션을 배포하고, service를 구성합니다.

```text
#ecsdemo frontend clb depolyment apply
kubectl apply -f ./ecsdemo-frontend/kubernetes/clb_deployment.yaml
#ecsdemo frontend clb service apply
kubectl apply -f ./ecsdemo-frontend/kubernetes/clb_service.yaml

```

정상적으로 Pod가 배포되었는지 아래 명령을 통해서 확인해 봅니다.

```text
kubectl -n clb-test get deployments ecsdemo-frontend -o wide
kubectl -n clb-test get service ecsdemo-frontend -o wide 

```

Replica를 3개로 늘려서 LB가 FrontEnd에서 정상적으로 이뤄지는 지 확인합니다.

```text
kubectl -n clb-test scale deployment ecsdemo-frontend --replicas=3

```

아래 출력되는 결과의 EXTERNAL-IP를 복사해서 브라우져 창에서 실행해 봅니다.

```text
kubectl -n clb-test get service ecsdemo-frontend -o wide                                                           
NAME               TYPE           CLUSTER-IP     EXTERNAL-IP                                                                   PORT(S)        AGE     SELECTOR
ecsdemo-frontend   LoadBalancer   172.20.37.78   afd75bf8c69c04c3aacf6cfbdefe1c4f-884593752.ap-northeast-2.elb.amazonaws.com   80:31380/TCP   5m45s   app=ecsdemo-frontend
```

출력결과 예시

![](../.gitbook/assets/image%20%28151%29.png)

앞서 설치해 둔 K9s 유틸리티를 통해서 , 현재 배포된 Pod들의 상태를 확인해 봅니다.

```text
k9s -A

```

### 4. BackEnd 어플리케이션 배포

Backend 어플리케이션 Nodejs와 Crystal을 배포합니다. 이 2개의 어플리케이션들은 Private Subnet에 배포할 것입니다. 이 구성은 앞서 이미 Yaml 파일의 Deployment에서 nodeSelector로 지정하였습니다.

```text
#ecsdemo nodejs clb depolyment apply
kubectl apply -f ./ecsdemo-nodejs/kubernetes/clb_deployment.yaml
#ecsdemo nodejs clb service apply
kubectl apply -f ./ecsdemo-nodejs/kubernetes/clb_service.yaml

#ecsdemo crystal clb depolyment apply
kubectl apply -f ./ecsdemo-crystal/kubernetes/clb_deployment.yaml
#ecsdemo crystal clb service apply
kubectl apply -f ./ecsdemo-crystal/kubernetes/clb_service.yaml 

```

정상적으로 Pod가 배포되었는지 아래 명령을 통해서 확인해 봅니다.

```text
kubectl -n clb-test get deployments ecsdemo-nodejs -o wide
kubectl -n clb-test get service ecsdemo-nodejs -o wide 
kubectl -n clb-test get deployments ecsdemo-crystal -o wide
kubectl -n clb-test get service ecsdemo-crystal -o wide 

```

Replica를 3개로 늘려서 Service Type이 없는 경우, BackEnd에서 정상적으로 이뤄지는 지 확인합니다.

```text
kubectl -n clb-test scale deployment ecsdemo-nodejs --replicas=3
kubectl -n clb-test scale deployment ecsdemo-crystal --replicas=3

```

![](../.gitbook/assets/image%20%28153%29.png)

k9s 를 통해 Pod의 구성을 확인합니다.

{% hint style="info" %}
LAB 을 진행하면서, Pod의 배포 상황을 계속 모니터링하기 위해서 Cloud9 에서 Terminal을 하나 더 열고 K9s를 실행 시켜 두는 것이 좋습니다.
{% endhint %}

![](../.gitbook/assets/image%20%28155%29.png)

### 5. Loadbalancer 확인.

이제 서비스 타입을 확인하기 위해서 EC2 대시보드에서 Loadbalancer를 확인합니다.

CLB의 DNS Name을 복사해서 Web Browser에서 입력합니다.

![](../.gitbook/assets/image%20%28147%29.png)

{% hint style="info" %}
service 매니페스트에서 Service Type을 LoadBalancer로 지정하면, Default로 Classic LB가 구성됩니다. 또한 별도로 Service Type을 지정하지 않으면, ClusterIP로 지정됩니다.
{% endhint %}

아래 kubectl 명령을 통해 service type을 확인해 봅니다.

```text
 kubectl -n clb-test get service -o wide
```

```text
whchoi98:~/environment $ kubectl -n clb-test get service -o wide
NAME               TYPE           CLUSTER-IP       EXTERNAL-IP                                                                  PORT(S)        AGE    SELECTOR
ecsdemo-crystal    ClusterIP      172.20.180.230   <none>                                                                       80/TCP         27m    app=ecsdemo-crystal
ecsdemo-frontend   LoadBalancer   172.20.213.219   a6531bc45d323472d869946b9bfac449-46256153.ap-northeast-2.elb.amazonaws.com   80:31699/TCP   108m   app=ecsdemo-frontend
ecsdemo-nodejs     ClusterIP      172.20.181.252   <none>                                                                       80/TCP         22m    app=ecsdemo-nodejs
```

## NLB기반 Loadbalancer 서비스 구성.

다음과 같은 구성을 통해서 NLB 서비스를 구현해 봅니다. 

![](../.gitbook/assets/image%20%28182%29.png)

* namespace : nlb-test
* ecsdemo-frontend service type : nlb \(external\)
* ecsdemo-crystal service type: nlb\(internal\)
* ecsdemo-nodejs service type: nlb\(internal\)

### 1.배포용 yaml 복제

아래와 같이 NLB-service.yaml 를 각 App별로 생성하여 구성합니다.

```text
  cd ~/environment/
  cp ./ecsdemo-frontend/kubernetes/deployment.yaml ./ecsdemo-frontend/kubernetes/nlb_deployment.yaml
  cp ./ecsdemo-crystal/kubernetes/deployment.yaml ./ecsdemo-crystal/kubernetes/nlb_deployment.yaml
  cp ./ecsdemo-nodejs/kubernetes/deployment.yaml ./ecsdemo-nodejs/kubernetes/nlb_deployment.yaml
  cp ./ecsdemo-frontend/kubernetes/service.yaml ./ecsdemo-frontend/kubernetes/nlb_service.yaml
  cp ./ecsdemo-crystal/kubernetes/service.yaml ./ecsdemo-crystal/kubernetes/nlb_service.yaml
  cp ./ecsdemo-nodejs/kubernetes/service.yaml ./ecsdemo-nodejs/kubernetes/nlb_service.yaml
  
```

### 2.Yaml 변경

NLB 구성을 위해 복사한 Yaml 파일을 다음과 같이 변경합니다.

ecsdemo-frontend nlb\_deployment.yaml

```text
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecsdemo-frontend
  labels:
    app: ecsdemo-frontend
#name space change 
  namespace: nlb-test
spec:
  replicas: 1
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
#Container URL change.
        env:
        - name: CRYSTAL_URL
          value: "http://ecsdemo-crystal.nlb-test.svc.cluster.local/crystal"
        - name: NODEJS_URL
          value: "http://ecsdemo-nodejs.nlb-test.svc.cluster.local/"
#add nodeSelector
      nodeSelector:
        nodegroup-type: "backend-workloads"
```

ecsdemo-frontend nlb\_service.yaml

```text
apiVersion: v1
kind: Service
metadata:
  name: ecsdemo-frontend
#name space change 
  namespace: nlb-test
#add annotations for External nlb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  selector:
    app: ecsdemo-frontend
  type: LoadBalancer
  ports:
   -  protocol: TCP
      port: 80
      targetPort: 3000
```

ecsdemo-nodejs nlb\_deployment.yaml

```text
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecsdemo-nodejs
  labels:
    app: ecsdemo-nodejs
#name space change 
  namespace: nlb-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ecsdemo-nodejs
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: ecsdemo-nodejs
    spec:
      containers:
      - image: brentley/ecsdemo-nodejs:latest
        imagePullPolicy: Always
        name: ecsdemo-nodejs
        ports:
        - containerPort: 3000
          protocol: TCP
#add nodeSelector
      nodeSelector:
        nodegroup-type: "backend-workloads"
```

ecsdemo-nodejs nlb\_service.yaml

```text
apiVersion: v1
kind: Service
metadata:
  name: ecsdemo-nodejs
#name space change 
  namespace: nlb-test
#add annotations for Internal nlb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
spec:
  selector:
    app: ecsdemo-nodejs
#type LoadBalancer
  type: LoadBalancer
  ports:
   -  protocol: TCP
      port: 80
      targetPort: 3000


```

ecsdemo-crystal nlb\_deployment.yaml

```text
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecsdemo-crystal
  labels:
    app: ecsdemo-crystal
#name space change 
  namespace: nlb-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ecsdemo-crystal
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: ecsdemo-crystal
    spec:
      containers:
      - image: brentley/ecsdemo-crystal:latest
        imagePullPolicy: Always
        name: ecsdemo-crystal
        ports:
        - containerPort: 3000
          protocol: TCP
#add nodeSelector
      nodeSelector:
        nodegroup-type: "backend-workloads"
```

ecsdemo-crystal nlb\_service.yaml

```text
apiVersion: v1
kind: Service
metadata:
  name: ecsdemo-crystal
#name space change 
  namespace: nlb-test
#add annotations for Internal nlb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
spec:
  selector:
    app: ecsdemo-crystal
#type LoadBalancer
  type: LoadBalancer
  ports:
   -  protocol: TCP
      port: 80
      targetPort: 3000

```

### 3.FrontEnd 어플리케이션 배포

새로운 namespace를 구성합니다.

```text
kubectl create namespace nlb-test
```

어플리케이션을 배포하고, service를 구성합니다.

```text
#ecsdemo frontend nlb depolyment apply
kubectl apply -f ./ecsdemo-frontend/kubernetes/nlb_deployment.yaml
#ecsdemo frontend nlb service apply
kubectl apply -f ./ecsdemo-frontend/kubernetes/nlb_service.yaml

```

정상적으로 Pod가 배포되었는지 아래 명령을 통해서 확인해 봅니다.

```text
kubectl -n nlb-test get deployments ecsdemo-frontend -o wide
kubectl -n nlb-test get service ecsdemo-frontend -o wide 

```

Replica를 3개로 늘려서 LB가 FrontEnd에서 정상적으로 이뤄지는 지 확인합니다.

```text
kubectl -n nlb-test scale deployment ecsdemo-frontend --replicas=3
```

{% hint style="info" %}
NLB를 구성하기 위해서는 annotation을 통한 Labeling이 필요합니다. 아래 내용을 확인하고 목적에 맞게 설정합니다. 
{% endhint %}

```text
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

#### 참조 - 외부 로드밸런서를 위한 Public subnet 태그 

| 키 | 값 |
| :--- | :--- |
| `kubernetes.io/role/elb` | `1` |

#### 내부 로드밸런서를 위한 Private subnet 태그 

| 키 | 값 |
| :--- | :--- |
| `kubernetes.io/role/internal-elb` | `1` |

아래 출력되는 결과의 EXTERNAL-IP를 복사해서 브라우져 창에서 실행해 봅니다.

```text
kubectl -n nlb-test get service ecsdemo-frontend -o wide                                                           
NAME               TYPE           CLUSTER-IP     EXTERNAL-IP                                                                          PORT(S)        AGE   SELECTOR
ecsdemo-frontend   LoadBalancer   172.20.42.31   a7400b4751cf74f8e9cf9acb0c22c8b7-674596f9c43ee0e0.elb.ap-northeast-2.amazonaws.com   80:32228/TCP   17m   app=ecsdemo-frontend
```

![](../.gitbook/assets/image%20%28148%29.png)

앞서 설치해 둔 K9s 유틸리티를 통해서 , 현재 배포된 Pod들의 상태를 확인해 봅니다.

```text
k9s -A

```

{% hint style="info" %}
우리 LAB에서는 NLB의 Internal과 External을 어떻게 변경했는지 yaml 파일을 다시 확인해 봅니다.
{% endhint %}

### 4.BackEnd 어플리케이션 배포

Backend 어플리케이션 Nodejs와 Crystal을 배포합니다. 이 2개의 어플리케이션들은 Private Subnet에 배포할 것입니다. 이 구성은 앞서 이미 Yaml 파일의 Deployment에서 nodeSelector로 지정하였습니다.

```text
#ecsdemo nodejs nlb depolyment apply
kubectl apply -f ./ecsdemo-nodejs/kubernetes/nlb_deployment.yaml
#ecsdemo nodejs nlb service apply
kubectl apply -f ./ecsdemo-nodejs/kubernetes/nlb_service.yaml

#ecsdemo crystal nlb depolyment apply
kubectl apply -f ./ecsdemo-crystal/kubernetes/nlb_deployment.yaml
#ecsdemo crystal nlb service apply
kubectl apply -f ./ecsdemo-crystal/kubernetes/nlb_service.yaml 

```

정상적으로 Pod가 배포되었는지 아래 명령을 통해서 확인해 봅니다.

```text
kubectl -n nlb-test get deployments ecsdemo-nodejs -o wide
kubectl -n nlb-test get service ecsdemo-nodejs -o wide 
kubectl -n nlb-test get deployments ecsdemo-crystal -o wide
kubectl -n nlb-test get service ecsdemo-crystal -o wide 

```

Replica를 3개로 늘려서 Service Type이 없는 경우, BackEnd에서 정상적으로 이뤄지는 지 확인합니다.

```text
kubectl -n nlb-test scale deployment ecsdemo-nodejs --replicas=3
kubectl -n nlb-test scale deployment ecsdemo-crystal --replicas=3

```

![](../.gitbook/assets/image%20%28156%29.png)

k9s 를 통해 Pod의 구성을 확인합니다.

![](../.gitbook/assets/image%20%28154%29.png)





