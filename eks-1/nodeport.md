---
description: 2021-04-04 / 20min
---

# NodePort 기반 배포

Kubernetes에서는 Pod의 전면에서 Pod로 트래픽이 들어오는 트래픽을 전달하는 service 자원이 제공됩니다. 해당 Service 자원은 Pod의 IP 주소와 관계 없이 Pod의 Label Selector를 보고 트래픽을 전달하는 역할을 담당합니다.

Service의 종류는 아래와 같습니다.

* Cluster IP - Service 자원의 기본 타입이며 Kubernetes 내부에서만 접근 가
* NodePort - 로컬 호스트의 특정 포트를 Serivce의 특정 포트와 연결
* Loadbalancer - AWS CLB, NLB 등과 같은 로드밸런서가 노드 전면에서 처리하는 방식

## Nodeport 기반 Service 구성 

아래 그림에서 처럼 Service의 기본은 CLUSTER-IP 방식입니다. 외부로 노출되지 않으며, Service에는  Pod Container의 포트를 기술해 줍니다.

![Cluster IP &#xD0C0;&#xC785; &#xAE30;&#xBC18; &#xC11C;&#xBE44;&#xC2A4;](../.gitbook/assets/image%20%28173%29.png)

NodePort 타입기반의 Service는 Node에서 Port를 외부에 노출 시키고 , 해당 포트로 유입되는 트래픽을 Service로 전달하고  Pod Container의 포트로 전달합니다.

![NodePort &#xD0C0;&#xC785; &#xAE30;&#xBC18;&#xC758; &#xC11C;&#xBE44;&#xC2A4;](../.gitbook/assets/image%20%28170%29.png)

### 1.배포용 yaml 복제.

NodePort 타입의 서비스 구성을 위해서 LAB에서 사용할 App을 복제합니다.

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
 cp ./ecsdemo-frontend/kubernetes/deployment.yaml ./ecsdemo-frontend/kubernetes/nodeport_deployment.yaml
 cp ./ecsdemo-frontend/kubernetes/service.yaml ./ecsdemo-frontend/kubernetes/nodeport_service.yaml
 cp ./ecsdemo-crystal/kubernetes/deployment.yaml ./ecsdemo-crystal/kubernetes/nodeport_deployment.yaml
 cp ./ecsdemo-crystal/kubernetes/service.yaml ./ecsdemo-crystal/kubernetes/nodeport_service.yaml
 cp ./ecsdemo-nodejs/kubernetes/deployment.yaml ./ecsdemo-nodejs/kubernetes/nodeport_deployment.yaml
 cp ./ecsdemo-nodejs/kubernetes/service.yaml ./ecsdemo-nodejs/kubernetes/nodeport_service.yaml
 
```

### 2. Yaml 변경

~/environment/ecsdemo-frontend/kubernetes/ecsdemo-frontend/nodeport\_deployment.yaml 파일은은 다음과 같이 변경합니다.

```text
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecsdemo-frontend
  labels:
    app: ecsdemo-frontend
#name space change 
  namespace: nodeport-test
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

~/environment/ecsdemo-frontend/kubernetes/ecsdemo-frontend/nodeport\_service.yaml은 다음과 같이 변경합니다.

```text
apiVersion: v1
kind: Service
metadata:
  name: ecsdemo-frontend
#name space change 
  namespace: nodeport-test
spec:
  selector:
    app: ecsdemo-frontend
#Service Type change
  type: NodePort
  ports:
   -  protocol: TCP
      nodePort: 30080
      port: 80
      targetPort: 3000
```

## Application 배포 

### 1.namespace 생성 

이제 Namespace를 먼저 생성합니다.

```text
kubectl create namespace nodeport-test

```

### 2. Application , Service 배포

nodeport 용 어플리케이션과 서비스를 배포합니다.

```text
cd ~/environment/ecsdemo-frontend/kubernetes/
kubectl apply -f nodeport_deployment.yaml
kubectl apply -f nodeport_service.yaml

```

정상적으로 배포되었는지 확인해 봅니다.

```text
kubectl -n nodeport-test get service

```

아래와 같은 결과를 볼 수 있습니다.

```text
whchoi98:~/environment/ecsdemo-frontend/kubernetes (main) $ kubectl -n nodeport-test get service
NAME               TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
ecsdemo-frontend   NodePort   172.20.164.40   <none>        80:30080/TCP   20h
```

### 3. 서비스 확인



