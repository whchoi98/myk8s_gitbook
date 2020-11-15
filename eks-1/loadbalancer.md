---
description: 'update : 2020-11-11'
---

# Loadbalancer 기반 배포

## Loadbalancer 서비스 타입 소개.

앞서 Service 배포의 ClusterIP 타입을 Kubernetes Dashboard 배포를 통해 확인했습니다.

그외에 NodePort 타입으로도 구성이 가능합니다. 아래는 NodePort 타입으로 구성하는 아키텍쳐입니다. 

![](../.gitbook/assets/image%20%288%29.png)

노드 타입 구조의 경우 노드의 IP주소와 Port에 종속되고, 확장성과 유연함에 한계가 있습니다. 하지만 NodePort를 사용해도 대부분의 포트 기반의 서비스를 구성할 수 있습니다.

대부분은 Loadbalancer 타입과 Ingress 기반과 Service 가 연계되어 확장성과 유연함을 갖는 구조를 제공합니다.

![](../.gitbook/assets/image%20%285%29.png)

## 기본 Loadbalancer 서비스 기반 구성

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

ecsdemo-frontend clb\_deployment.yaml은 다음과 같이 구성됩니다.

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

ecsdemo-frontend clb\_service.yaml은 다음과 같이 구성됩니다.

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

ecsdemo-crystal clb\_deployment.yaml은 다음과 같이 구성됩니다.

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

ecsdemo-crystal clb\_service.yaml은 다음과 같이 구성됩니다.

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

ecsdemo-nodejs clb\_deployment.yaml은 다음과 같이 구성됩니다.

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

ecsdemo-nodejs clb\_service.yaml은 다음과 같이 구성됩니다.

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

## NLB기반 Loadbalancer 서비스 구성.

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

NLB 구성을 위해 복사한 Yaml 파일을 변경합니다.

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
        nodegroup-type: "frontend-workloads"
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

ecsdemo-nodejs nlb\_service.yaml

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

#### 외부 로드밸런서를 위한 Public subnet 태그 

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

### 





