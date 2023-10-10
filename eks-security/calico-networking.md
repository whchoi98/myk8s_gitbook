---
description: 'Update : 2020-11-17'
---

# Calico 네트워크 정책

## Calico 네트워킹 소개

![](<../.gitbook/assets/image (424).png>)

Calico는 컨테이너, 가상 머신 및 기본 호스트 기반 워크로드를 위한 오픈 소스 네트워킹 및 네트워크 보안 솔루션입니다. Calico는 Kubernetes, OpenShift, Docker EE, OpenStack 및 베어 메탈 서비스를 포함한 광범위한 플랫폼을 지원합니다.

Calico는 유연한 네트워킹 기능과 보안 기능을 결합하여 네이티 Linux 커널 성능과 클라우드 네이티브 확장성을 갖춘 솔루션을 제공합니다. Calico 네트워크 정책 집행을 통해 네트워크 세분화 및 테넌트 격리를 구현할 수 있습니다. 이는 테넌트를 각각 격리해야 하는 다중 테넌트 환경에서 또는 개발, 스테이징 및 프로덕션에 별도의 환경을 생성하고자 하는 경우 유용합니다. 네트워크 정책은 네트워크 인,아 규칙을 생성할 수 있다는 점에서 AWS 보안 그룹과 유사하지만, 인스턴스를 보안 그룹에 할당하는 대신 포드의 selector, label등 사용하여 네트워크 정책을 Pod 할당합니다.

{% hint style="danger" %}
주의 !!! Amazon EKS와 함께 Fargate를 사용하는 경우 Calico가 지원되지 않습니다.

이 랩에서는 Calico CNI를 사용하지 않습니다. Calico의 Network Policy만 구성해서 사용합니다.
{% endhint %}

해당 LAB은 Project Calico 를 참조합니다. [https://docs.projectcalico.org/security/kubernetes-policy](https://docs.projectcalico.org/security/kubernetes-policy)

## EKS에 Calico 설치하기

[`aws/amazon-vpc-cni-k8s` GitHub 프로젝트](https://github.com/aws/amazon-vpc-cni-k8s)에서 Calico 매니페스트를 적용합니다. 이 매니페스트는 `kube-system` 네임스페이스에 데몬 세트를 생성합니다.

```
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/release-1.7/config/v1.7/calico.yaml
```

calico 매니페스트 파일의 주요 내용을 살펴 봅니다.&#x20;

```
more calico.yaml
```

정상적으로 설치되었는 지 확인합니다.

```
kubectl get daemonsets -n kube-system

```

아래와 같이 결과를 확인 할 수 있습니다. 모든 노드에 분산 설치되기 위해 Daemonset으로 설치 되었습니다.

```
whchoi98:~/environment/myeks (master) $ kubectl get daemonsets -n kube-system                                                                           
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
aws-node      6         6         6       6            6           <none>                        8d
calico-node   6         6         6       6            6           beta.kubernetes.io/os=linux   10m
kube-proxy    6         6         6       6            6           <none>             
```

## Calico 기반의 네트워크 정책 구성.

### 1.데모 앱 배포.&#x20;

이 랩에서는 EKS Cluster에 새로운 Namespace, front-end, back-end, Client, UI 서비스 등을 만들고, 상호간의 네트워크 통신을 허용 또는 제어하는 네트워크 정책을 생성합니다. 또한 각 서비스간에 사용 가능한 송/수신 경로를 보여주는 UI를 포함하고 있습니다.

*   name space : stars

    * Pod : frontend , backend 각 1개
    * replicationcontroller : frontend, backend 1개에 설정
    * service : frontend, backend에 ClusterIP type으로 설정


* name space : management-ui
  * Pod : management-ui
  * replicationcontroller : management-ui 1개에 설정
  * service : management-ui에 LoadBalcer Type으로 설정.
* name space : client
  * Pod: client-sjnjk
  * replicationcontroller : client에 1개 설정
  * service : client에 ClusterIP type으로 설정

```
cd ~/environment/myeks
git pull origin master
mkdir ~/environment/calico_resources
cd ~/environment/calico_resources
kubectl apply -f ~/environment/myeks/calico_demo/namespace.yaml
kubectl apply -f ~/environment/myeks/calico_demo/management-ui.yaml
kubectl apply -f ~/environment/myeks/calico_demo/backend.yaml
kubectl apply -f ~/environment/myeks/calico_demo/frontend.yaml
kubectl apply -f ~/environment/myeks/calico_demo/client.yaml 
kubectl -n stars get all -o wide
kubectl -n management-ui get all -o wide
kubectl -n client get all -o wide 
```

아래와 같은 출력 결과를 확인 할 수 있습니다.

```
whchoi98:~/environment/calico_resources $ kubectl -n stars get all -o wide
NAME                 READY   STATUS    RESTARTS   AGE     IP             NODE                                               NOMINATED NODE   READINESS GATES
pod/backend-4z6z5    1/1     Running   0          9m58s   10.11.123.18   ip-10-11-114-132.ap-northeast-2.compute.internal   <none>           <none>
pod/frontend-rvcf7   1/1     Running   0          9m53s   10.11.107.25   ip-10-11-114-132.ap-northeast-2.compute.internal   <none>           <none>

NAME                             DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES                     SELECTOR
replicationcontroller/backend    1         1         1       9m58s   backend      calico/star-probe:v0.1.0   role=backend
replicationcontroller/frontend   1         1         1       9m53s   frontend     calico/star-probe:v0.1.0   role=frontend

NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE     SELECTOR
service/backend    ClusterIP   172.20.176.57    <none>        6379/TCP   9m58s   role=backend
service/frontend   ClusterIP   172.20.145.253   <none>        80/TCP     9m53s   role=frontend
whchoi98:~/environment/calico_resources $ kubectl -n management-ui get all -o wide
NAME                      READY   STATUS    RESTARTS   AGE   IP              NODE                                              NOMINATED NODE   READINESS GATES
pod/management-ui-75xht   1/1     Running   0          12m   10.11.172.124   ip-10-11-189-67.ap-northeast-2.compute.internal   <none>           <none>

NAME                                  DESIRED   CURRENT   READY   AGE   CONTAINERS      IMAGES                       SELECTOR
replicationcontroller/management-ui   1         1         1       12m   management-ui   calico/star-collect:v0.1.0   role=management-ui

NAME                    TYPE           CLUSTER-IP     EXTERNAL-IP                                                                    PORT(S)        AGE   SELECTOR
service/management-ui   LoadBalancer   172.20.18.17   a927c1a56c9a144aba431cdb58b9c5a7-1577995596.ap-northeast-2.elb.amazonaws.com   80:32184/TCP   12m   role=management-ui
whchoi98:~/environment/calico_resources
whchoi98:~/environment/calico_resources $ kubectl -n client get all -o wide                                                                                  
NAME               READY   STATUS    RESTARTS   AGE   IP             NODE                                               NOMINATED NODE   READINESS GATES
pod/client-sjnjk   1/1     Running   0          14m   10.11.122.81   ip-10-11-114-132.ap-northeast-2.compute.internal   <none>           <none>

NAME                           DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES                     SELECTOR
replicationcontroller/client   1         1         1       14m   client       calico/star-probe:v0.1.0   role=client

NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE   SELECTOR
service/client   ClusterIP   172.20.86.231   <none>        9000/TCP   14m   role=client
```

각 매니페스트 파일을 참조하십시요

```
ind: Namespace
apiVersion: v1
metadata:
  name: stars
```

frontend.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: frontend 
  namespace: stars
spec:
  ports:
  - port: 80 
    targetPort: 80 
  selector:
    role: frontend 
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: frontend 
  namespace: stars
spec:
  replicas: 1
  template:
    metadata:
      labels:
        role: frontend 
    spec:
      containers:
      - name: frontend 
        image: calico/star-probe:v0.1.0
        imagePullPolicy: Always
        command:
        - probe
        - --http-port=80
        - --urls=http://frontend.stars:80/status,http://backend.stars:6379/status,http://client.client:9000/status
        ports:
        - containerPort: 80 
```

backend.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: backend 
  namespace: stars
spec:
  ports:
  - port: 6379
    targetPort: 6379 
  selector:
    role: backend 
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: backend 
  namespace: stars
spec:
  replicas: 1
  template:
    metadata:
      labels:
        role: backend 
    spec:
      containers:
      - name: backend 
        image: calico/star-probe:v0.1.0
        imagePullPolicy: Always
        command:
        - probe
        - --http-port=6379
        - --urls=http://frontend.stars:80/status,http://backend.stars:6379/status,http://client.client:9000/status
        ports:
        - containerPort: 6379 
```

management-ui.yaml 파일

```
apiVersion: v1
kind: Namespace
metadata:
  name: management-ui 
  labels:
    role: management-ui 
---
apiVersion: v1
kind: Service
metadata:
  name: management-ui 
  namespace: management-ui 
spec:
  type: LoadBalancer
  ports:
  - port: 80 
    targetPort: 9001
  selector:
    role: management-ui 
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: management-ui 
  namespace: management-ui 
spec:
  replicas: 1
  template:
    metadata:
      labels:
        role: management-ui 
    spec:
      containers:
      - name: management-ui 
        image: calico/star-collect:v0.1.0
        imagePullPolicy: Always
        ports:
        - containerPort: 9001
```

client yaml 파일

```
kind: Namespace
apiVersion: v1
metadata:
  name: client
  labels:
    role: client
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: client 
  namespace: client
spec:
  replicas: 1
  template:
    metadata:
      labels:
        role: client 
    spec:
      containers:
      - name: client 
        image: calico/star-probe:v0.1.0
        imagePullPolicy: Always
        command:
        - probe
        - --urls=http://frontend.stars:80/status,http://backend.stars:6379/status
        ports:
        - containerPort: 9000 
---
apiVersion: v1
kind: Service
metadata:
  name: client
  namespace: client
spec:
  ports:
  - port: 9000 
    targetPort: 9000
  selector:
    role: client 
```

### 2. management-ui에 접속

management UI Pod는 External IP로 LB Service를 제공하고 있습니다. 아래와 같은 명령을 통해서 External IP를 확인합니다.

```
export ELB_SERVICE_URL=$(kubectl get svc -n management-ui management-ui --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
echo "ELB SERVICE URL = $ELB_SERVICE_URL"
```

출력 결과는 아래와 같습니다.

```
whchoi98:~/environment/calico_resources $ echo "ELB SERVICE URL = $ELB_SERVICE_URL"
ELB SERVICE URL = a927c1a56c9a144aba431cdb58b9c5a7-1577995596.ap-northeast-2.elb.amazonaws.com
```

해당 웹 사이트는 Client App , Front end App, Back end App 간의 트래픽 허용 상태를 제공해 줍니다.

![](<../.gitbook/assets/image (50).png>)

### 3.네트워크 정책 적용

Network Policy를 적용해서 Pod간의 제어를 확인해 봅니다.

먼저 stars, client namespace 에 deny 정책을 업데이트 합니다.

```
cd ~/environment/myeks/calico_demo/
kubectl apply -n stars -f default-deny.yaml
kubectl apply -n client -f default-deny.yaml
```

default-deny.yaml을 확인해 봅니다.

```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: default-deny
spec:
  podSelector:
    matchLabels: {}
```

다시 아래 management-ui 로 접속해 봅니다.

```
whchoi98:~/environment/calico_resources $ echo "ELB SERVICE URL = $ELB_SERVICE_URL"
ELB SERVICE URL = a927c1a56c9a144aba431cdb58b9c5a7-1577995596.ap-northeast-2.elb.amazonaws.com
```

ELB 주소로 접속하면, All deny로 출력되는 결과가 없습니다.

![](<../.gitbook/assets/image (84).png>)

다시 정책을 허용합니다.

```
kubectl apply -f allow-ui.yaml
kubectl apply -f allow-ui-client.yaml

```

아래에서 처럼 이제 management-ui로는 접속이 가능합니다. 하지만 Frontend, Backend, Client Pod간에는 통신되지 않습니다.

![](<../.gitbook/assets/image (63).png>)

아래 매니페스트 파일을 확인해 봅니다.

namespace : stars 의 frontend, backend pod는 management-ui에 연결이 가능하도록 합니다.

```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: stars
  name: allow-ui 
spec:
  podSelector:
    matchLabels: {}
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              role: management-ui 
```

namespace : client 의 client pod는 management-ui에 연결이 가능하도록 합니다.

```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: client 
  name: allow-ui 
spec:
  podSelector:
    matchLabels: {}
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              role: management-ui 
```

이제 Client에서 Front end로 유입되는 트래픽을 허용합니다.

```
kubectl apply -f frontend-policy.yaml
```

아래에서 매니페스트 파일을 통해 상세 내용을 확인 할 수 있습니다.

```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: stars
  name: frontend-policy
spec:
  podSelector:
    matchLabels:
      role: frontend
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              role: client
      ports:
        - protocol: TCP
          port: 80
```

다시 management-ui 로 접속해 보면 트래픽 허용되는 것을 확인 할 수 있습니다.

![](<../.gitbook/assets/image (459).png>)

이제 frontend에서 backend로 유입되는 트래픽을 허용합니다.

```
kubectl apply -f backend-policy.yaml
```

아래에서 매니페스트 파일을 통해 상세 내용을 확인 할 수 있습니다.

```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: stars
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      role: backend
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - protocol: TCP
          port: 6379
```

다시 management-ui 로 접속해 보면 트래픽 허용되는 것을 확인 할 수 있습니다.

![](<../.gitbook/assets/image (168).png>)







&#x20;

&#x20;

