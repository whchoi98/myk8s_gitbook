---
description: 'Update: 2021-04-04 / 30min'
---

# NodePort 기반 배포

Kubernetes에서는 Pod의 전면에서 Pod로 트래픽이 들어오는 트래픽을 전달하는 service 자원이 제공됩니다. 해당 Service 자원은 Pod의 IP 주소와 관계 없이 Pod의 Label Selector를 보고 트래픽을 전달하는 역할을 담당합니다.

Service의 종류는 아래와 같습니다.

* Cluster IP - Service 자원의 기본 타입이며 Kubernetes 내부에서만 접근 가
* NodePort - 로컬 호스트의 특정 포트를 Serivce의 특정 포트와 연결
* Loadbalancer - AWS CLB, NLB 등과 같은 로드밸런서가 노드 전면에서 처리하는 방식

## Nodeport 기반 Service 구성 

아래 그림에서 처럼 Service의 기본은 CLUSTER-IP 방식입니다. 외부로 노출되지 않으며, Service에는  Pod Container의 포트를 기술해 줍니다.

![Cluster IP &#xD0C0;&#xC785; &#xAE30;&#xBC18; &#xC11C;&#xBE44;&#xC2A4;](../.gitbook/assets/image%20%28175%29.png)

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

![](../.gitbook/assets/image%20%28173%29.png)

Security Group에서 TCP 30080를 허용합니다.

![](../.gitbook/assets/image%20%28179%29.png)

이제 해당 인스턴스의 공인 IP로 브라우저를 통해서 접근해서 서비스를 확인해 봅니다.

```text
node공인ip주소:30080
```

아래와 같은 결과를 확인할 수 있습니다.

![](../.gitbook/assets/image%20%28174%29.png)

이제 Pod를 3개로 늘려서 서비스를 확인해 봅니다.

```text
kubectl -n nodeport-test scale deployment ecsdemo-frontend --replicas=3
kubectl -n nodeport-test get pods

```

![](../.gitbook/assets/image%20%28178%29.png)

{% hint style="info" %}
NodePort 30080을 하나의 노드에서만 Security Group으로 허용했는데도, 서비스 분산이 이뤄집니다. 이것은 특정 Node로 Nodeport로 트래픽이 인입하고, 내부에서는 Service를 통해서 Label Selector를 통해서 부하 분산이 이뤄지고 있는 것입니다.
{% endhint %}

## CoreDNS와 Service

### 1.CoreDNS와 Service 역할 확인을 위한 App배포 

아래와 같이 새로운 Namespace와 Pod를 생성합니다.

```text
cd ~/environment/myeks/network-test
kubectl create namespace network-test
kubectl -n network-test apply -f test-deployment.yaml
kubectl -n network-test get pods

```

정상적으로 Pods가 생성되었는지 확인합니다.

```text
whchoi98:~/environment/myeks/network-test (master) $ kubectl -n network-test get pod
NAME                          READY   STATUS    RESTARTS   AGE
alpine-app-6d8d6bb647-lbp7v   1/1     Running   0          27s
alpine-app-6d8d6bb647-mwzbp   1/1     Running   0          27s
alpine-app-6d8d6bb647-rwbts   1/1     Running   0          27s
```

한개의 Pod로 접속해 봅니다.

```text
kubectl -n network-test exec -it alpine-app-6d8d6bb647-lbp7v -- bash

```

ip a 와 /etc/resolve.conf를 조회해 봅니다.

```text
ip a
cat /etc/resolve.conf
```

다음과 같이 출력됩니다.

```text
bash-5.0# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if17: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP group default 
    link/ether ba:bc:1a:b1:e7:d8 brd ff:ff:ff:ff:ff:ff link-netnsid 0
## Pod의 IP입니다.
    inet 10.11.37.106/32 scope global eth0
       valid_lft forever preferred_lft forever
bash-5.0# cat /etc/resolv.conf
##coredns 주소입니다. 
nameserver 172.20.0.10
## FQDN 정책이며,실제 내부에서 사용하는 Host 명입니다.
search network-test.svc.cluster.local svc.cluster.local cluster.local ap-northeast-2.compute.internal
options ndots:5
```

한개의 Pod에 더 연결해 보고 동일하게 비교해 봅니다.

```text
kubectl -n network-test exec -it alpine-app-6d8d6bb647-mwzbp -- bash

```



```text
bash-5.0# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if21: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP group default 
    link/ether 32:3c:aa:a0:0b:6d brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.11.19.91/32 scope global eth0
       valid_lft forever preferred_lft forever
bash-5.0# cat /etc/resolv.conf 
nameserver 172.20.0.10
search network-test.svc.cluster.local svc.cluster.local cluster.local ap-northeast-2.compute.internal
options ndots:5
```

AWS VPC CNI 구성은 Pod 생성할 때 마다 ENI를 생성하므로, Pod간 IP 직접 통신이 가능합니다.

```text
bash-5.0# ping 10.11.37.106
PING 10.11.37.106 (10.11.37.106) 56(84) bytes of data.
64 bytes from 10.11.37.106: icmp_seq=1 ttl=253 time=1.15 ms
64 bytes from 10.11.37.106: icmp_seq=2 ttl=253 time=1.13 ms
64 bytes from 10.11.37.106: icmp_seq=3 ttl=253 time=1.12 ms

```

이제 상호간의 Pod 이름으로 ping을 사용해 봅니다.

```text
bash-5.0# ping alpine-app-6d8d6bb647-lbp7v
ping: alpine-app-6d8d6bb647-lbp7v: Name does not resolve

```



```text
kubectl apply -f test-service.yaml

```

