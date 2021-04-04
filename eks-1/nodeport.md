---
description: 2021-04-04 / 20min
---

# NodePort 기반 배포



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
  type: LoadBalancer
  ports:
   -  protocol: TCP
      port: 80
      targetPort: 3000
```

