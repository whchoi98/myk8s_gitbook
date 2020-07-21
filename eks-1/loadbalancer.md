# Loadbalancer 기반 배포

## Loadbalancer 서비스 타입 소개.

앞서 Service 배포의 ClusterIP 타입을 Kubernetes Dashboard 배포를 통해 확인했습니다.

그외에 NodePort 타입으로도 구성이 가능합니다. 아래는 NodePort 타입으로 구성하는 아키텍쳐입니다. 

![](../.gitbook/assets/image%20%288%29.png)

노드 타입 구조의 경우 노드의 IP주소와 Port에 종속되고, 확장성과 유연함에 한계가 있습니다.

대부분은 Loadbalancer 타입과 Ingress 기반과 Service 가 연계되어 확장성과 유연함을 갖는 구조를 제공합니다.

![](../.gitbook/assets/image%20%285%29.png)

## Loadbalancer 서비스 기반 구성

### 1.배포용 yaml 복제.

LAB에서 사용할 App을 복제합니다.

```text
cd ~/environment
git clone https://github.com/brentley/ecsdemo-frontend.git
git clone https://github.com/brentley/ecsdemo-nodejs.git
git clone https://github.com/brentley/ecsdemo-crystal.git
```

정상적으로 복제 이후 Cloud9에서 아래와 같이 확인됩니다.

![](../.gitbook/assets/image%20%2816%29.png)

아래와 같이 새로운 deployment, service를 복사합니다.

```text
  cd ~/environment/
  cp ./ecsdemo-frontend/kubernetes/deployment.yaml ./ecsdemo-frontend/kubernetes/1deployment.yaml
  cp ./ecsdemo-crystal/kubernetes/deployment.yaml ./ecsdemo-crystal/kubernetes/1deployment.yaml
  cp ./ecsdemo-nodejs/kubernetes/deployment.yaml ./ecsdemo-nodejs/1deployment.yaml
```

### 2. 배포용 Node 선택.

앞서 랩을 진행과정에서 eksctl을 통해 배포한 , yaml 파일에는 Worker Node에 Label을 설정하였습니다.

```text
# A simple example of ClusterConfig object:
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig


생략.

nodeGroups:
  - name: ng1-public
    instanceType: m5.xlarge
    desiredCapacity: 3
    minSize: 3
    maxSize: 9
    volumeSize: 30
    volumeType: gp2 
    amiFamily: AmazonLinux2
    labels:
      nodegroup-type: "frontend-workloads"
생략
  - name: ng2-private
    instanceType: m5.xlarge
    desiredCapacity: 3
    privateNetworking: true
    minSize: 3
    maxSize: 9
    volumeSize: 30
    volumeType: gp2 
    amiFamily: AmazonLinux2
    labels:
      nodegroup-type: "backend-workloads"
이하 생략
```

3개의 복제된 1depolyment.yaml 파일에 아래내용을 추가합니다. 이것은 Worker Node중에 Public Subnet에 위치한 Workernode에 App을 배포하는 것입니다.

추가하게 되면 아래와 같은 배포 yaml을 가지게 됩니다. 

예 - ecsdemo-frontend - 1deployment.yaml nodegroup-type: “frontend-workloads”를 지정해서 Application을 배포할 것입니다. git에서 복제한 deployment yaml에 아래 nodeSelector를 선언합니다.

```text
      nodeSelector:
        nodegroup-type: "frontend-workloads"
```

1deployment.yaml은 다음과 같이 구성됩니다.

```text
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecsdemo-crystal
  labels:
    app: ecsdemo-crystal
  namespace: default
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
        - image: 'brentley/ecsdemo-crystal:latest'
          imagePullPolicy: Always
          name: ecsdemo-crystal
          ports:
            - containerPort: 3000
              protocol: TCP
      nodeSelector:
        nodegroup-type: "frontend-workloads"
```

### 3. 어플리케이션 배포와 서비스 구성.

어플리케이션을 배포하고, service를 구성합니다.

```text
kubectl apply -f ./ecsdemo-nodejs/kubernetes/1deployment.yaml 
kubectl apply -f ./ecsdemo-nodejs/kubernetes/service.yaml 
kubectl apply -f ./ecsdemo-crystal/kubernetes/1deployment.yaml
kubectl apply -f ./ecsdemo-crystal/kubernetes/service.yaml 
kubectl apply -f ./ecsdemo-frontend/kubernetes/1deployment.yaml
kubectl apply -f ./ecsdemo-frontend/kubernetes/service.yaml
```

정상적으로 Pod가 배포되었는지 아래 명령을 통해서 확인해 봅니다.

```text
kubectl get deployment ecsdemo-crystal  -o wide
kubectl get deployment ecsdemo-nodejs  -o wide
kubectl get service ecsdemo-frontend -o wide
```

출력결과 예시

```text
whchoi98:~/environment $ kubectl get deployment ecsdemo-crystal  -o wide
NAME              READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS        IMAGES                            SELECTOR
ecsdemo-crystal   1/1     1            1           6m52s   ecsdemo-crystal   brentley/ecsdemo-crystal:latest   app=ecsdemo-crystal
whchoi98:~/environment $ kubectl get deployment ecsdemo-nodejs  -o wide                                                                                                                        
NAME             READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS       IMAGES                           SELECTOR
ecsdemo-nodejs   1/1     1            1           7m17s   ecsdemo-nodejs   brentley/ecsdemo-nodejs:latest   app=ecsdemo-nodejs
whchoi98:~/environment $ kubectl get service ecsdemo-frontend -o wide
NAME               TYPE           CLUSTER-IP      EXTERNAL-IP                                                                    PORT(S)        AGE     SELECTOR
ecsdemo-frontend   LoadBalancer   172.20.222.76   a4082a6b75ef24b608a0a6705a2ca35a-1509938170.ap-northeast-2.elb.amazonaws.com   80:30547/TCP   3m39s   app=ecsdemo-frontend
```

### 4. Replicas 구성

3개 이상의 Replica를 구성하여, Loadbalancer가 정상적으로 동작하는 지 확인합니다.

```text
kubectl scale deployment ecsdemo-frontend --replicas=3
kubectl scale deployment ecsdemo-nodejs --replicas=3
kubectl scale deployment ecsdemo-crystal --replicas=3
```

정상적으로 배포되었는지 확인합니다.

```text
kubectl get deployment ecsdemo-nodejs  -o wide
kubectl get deployment ecsdemo-crystal  -o wide
kubectl get deployment ecsdemo-frontend  -o wide
```

출력 결과 예시

```text
whchoi98:~/environment $ kubectl get deployment ecsdemo-nodejs  -o wide                                                                                                                
NAME             READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS       IMAGES                           SELECTOR
ecsdemo-nodejs   3/3     3            3           14m   ecsdemo-nodejs   brentley/ecsdemo-nodejs:latest   app=ecsdemo-nodejs
whchoi98:~/environment $ kubectl get deployment ecsdemo-crystal  -o wide                                                                                                                       
NAME              READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS        IMAGES                            SELECTOR
ecsdemo-crystal   3/3     3            3           14m   ecsdemo-crystal   brentley/ecsdemo-crystal:latest   app=ecsdemo-crystal
whchoi98:~/environment $ kubectl get deployment ecsdemo-frontend  -o wide                                                                                                                      
NAME               READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS         IMAGES                             SELECTOR
ecsdemo-frontend   3/3     3            3           13m   ecsdemo-frontend   brentley/ecsdemo-frontend:latest   app=ecsdemo-frontend
```

k9s 를 통해 Pod의 구성을 확인합니다.

```text
k9s
```

{% hint style="info" %}
LAB 을 진행하면서, Pod의 배포 상황을 계속 모니터링하기 위해서 Cloud9 에서 Terminal을 하나 더 열고 K9s를 실행 시켜 두는 것이 좋습니다.
{% endhint %}

이제 서비스 타입을 확인하기 위해서 EC2 대시보드에서 Loadbalancer를 확인합니다.

![](../.gitbook/assets/image.png)

