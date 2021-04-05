---
description: 'Update: 2021-04-04 / 30min'
---

# NodePort 기반 배포

## Overview

nodeport 타입의 service는 Node\(EC2인스턴스\)의 포트를 통해서 서비스로 전달하는 방식입니다. 특정 노드로 유입 시킨 이후에 Service에서 로드밸런싱을 사용할 수 있습니다.하지만 최초 모든 트래픽이 하나의 노드로 집중되기 때문에 이에 대한 설계 고려가 필요합니다.

## Nodeport 기반 Service 구성 

![Cluster IP &#xD0C0;&#xC785; &#xAE30;&#xBC18; &#xC11C;&#xBE44;&#xC2A4;](../.gitbook/assets/image%20%28179%29.png)

NodePort 타입기반의 Service는 Node에서 Port를 외부에 노출 시키고 , 해당 포트로 유입되는 트래픽을 Service로 전달하고  Pod Container의 포트로 전달합니다.

![NodePort &#xD0C0;&#xC785; &#xAE30;&#xBC18;&#xC758; &#xC11C;&#xBE44;&#xC2A4;](../.gitbook/assets/image%20%28174%29.png)

* namespace : nodeport-test
* ecsdemo-frontend service type : nodePort

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

정상적으로 배포되었는지 확인합니다.

```text
kubectl -n nodeport-test get pods

```

```text
whchoi98:~/environment/ecsdemo-frontend/kubernetes (main) $ kubectl -n nodeport-test get pods 
NAME                              READY   STATUS    RESTARTS   AGE
ecsdemo-frontend-746f9ff7-bpffv   1/1     Running   0          86s
```

Output wide 옵션을 통해서 실제 Pod가 배포된 Node를 확인합니다.

```text
kubectl -n nodeport-test get pods ecsdemo-frontend-746f9ff7-bpffv -o wide

```

아래 Node 이름을 확인 할 수 있습니다.

```text
whchoi98:~/environment/ecsdemo-frontend/kubernetes (main) $ kubectl -n nodeport-test get pods ecsdemo-frontend-746f9ff7-bpffv -o wide
NAME                              READY   STATUS    RESTARTS   AGE     IP             NODE                                             NOMINATED NODE   READINESS GATES
ecsdemo-frontend-746f9ff7-bpffv   1/1     Running   0          2m26s   10.11.18.181   ip-10-11-22-15.ap-northeast-2.compute.internal   <none>           <none>
```

Node 이름을 아래와 같이 상세하게 확인 할 수 있습니다.

```text
kubectl -n nodeport-test get pods ecsdemo-frontend-746f9ff7-bpffv -o wide
kubectl get nodes -o wide

```

Pod가 배포된 Node를 AWS 관리콘솔 - EC2 대시보드에서 선택합니다. 해당 EC2 대시보드에서 인스턴스를 선택합니다.

Public-SG 라는 Security Group을 생성하고, 해당 인스턴스에 적용합니다.

![](../.gitbook/assets/image%20%28177%29.png)

Security Group에서 TCP 30080를 허용합니다.

![](../.gitbook/assets/image%20%28184%29.png)

이제 해당 인스턴스의 공인 IP로 브라우저를 통해서 접근해서 서비스를 확인해 봅니다.

```text
node공인ip주소:30080
```

아래와 같은 결과를 확인할 수 있습니다.

![](../.gitbook/assets/image%20%28178%29.png)

이제 Pod를 3개로 늘려서 서비스를 확인해 봅니다.

```text
kubectl -n nodeport-test scale deployment ecsdemo-frontend --replicas=3
kubectl -n nodeport-test get pods

```

![](../.gitbook/assets/image%20%28182%29.png)

{% hint style="info" %}
NodePort 30080을 하나의 노드에서만 Security Group으로 허용했는데도, 서비스 분산이 이뤄집니다. 이것은 특정 Node로 Nodeport로 트래픽이 인입하고, 내부에서는 Service를 통해서 Label Selector를 통해서 부하 분산이 이뤄지고 있는 것입니다.
{% endhint %}

