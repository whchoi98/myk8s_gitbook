---
description: 'Update: 2021-05-21 / 30min'
---

# NodePort 기반 배포

## Overview

nodeport 타입의 service는 Node(EC2인스턴스)의 포트를 통해서 서비스로 전달하는 방식입니다. 특정 노드로 유입 시킨 이후에 Service에서 로드밸런싱을 사용할 수 있습니다.하지만 최초 모든 트래픽이 하나의 노드로 집중되기 때문에 이에 대한 설계 고려가 필요합니다.

## Nodeport 이해

nodeport type의 service는 클러스터에서 실행되는 서비스를 Node의 포트를 외부에 노출 시켜서 사용하는 방식입니다. NodePort로 노드그룹의 각 워커 노드의 공인 IP(ENI)에서 서비스를 노출 시키게 됩니다.

이 NodeIP:NodePort는 30000\~32767번 포트를 사용하게 되며, NodePort 서비스가 라우팅 되는 ClusterIP 서비스가 자동 생성됩니다.

### Nodeport 동작 방식 

* 외부 사용자는 Node(EC2)의 공인 IP/Port로 접근하게 됩니다. 
* Node(EC2)는 IPTable 규칙에 의해 Cluster IP/Port로 이동합니다. 
* IPTable 규칙에 의해 PoD 분산하게 됩니다.

![](<../.gitbook/assets/image (219).png>)

아래와 같이 새로운 Namespace와 Pod를 생성합니다.

```
cd ~/environment/myeks/network-test
kubectl create namespace node-test-01
kubectl -n node-test-01 apply -f node-test-01.yaml
kubectl -n node-test-01 get pods
kubectl -n node-test-01 get pods -o wide

```

생성한 pod를 확인합니다. 

```
kubectl -n node-test-01 get pods -o wide
NAME                            READY   STATUS    RESTARTS   AGE   IP             NODE                                              NOMINATED NODE   READINESS GATES
node-test-01-869b8d5f87-4w5g5   0/1     Running   0          7s    10.11.33.224   ip-10-11-35-116.ap-northeast-2.compute.internal   <none>           <none>
node-test-01-869b8d5f87-6rbhz   0/1     Running   0          7s    10.11.16.91    ip-10-11-21-111.ap-northeast-2.compute.internal   <none>           <none>
node-test-01-869b8d5f87-rbwqz   0/1     Running   0          7s    10.11.11.5     ip-10-11-3-68.ap-northeast-2.compute.internal     <none>           <none>
```

shell 연결을 편리하게 접속하기 위해 아래와 같이 cloud9 terminal 의 bash profile에 등록합니다.

```
echo "export NodeTestPod03=node-test-01-869b8d5f87-4w5g5" | tee -a ~/.bash_profile
echo "export NodeTestPod02=node-test-01-869b8d5f87-6rbhz" | tee -a ~/.bash_profile
echo "export NodeTestPod01=node-test-01-869b8d5f87-rbwqz" | tee -a ~/.bash_profile
source ~/.bash_profile

```

### **NodePort Service 시험 **

Nodeport service를 배포합니다.

```
kubectl -n node-test-01 apply -f node-test-01-service.yaml
kubectl -n node-test-01 get services -o wide
```

Nodeport service yaml은 아래와 같이 구성되어 있습니다.

```
apiVersion: v1
kind: Service
metadata:
  name: node-test-01-svc
  namespace: node-test-01
spec:
  selector:
    app: node-test-01
  ports:
  type: NodePort
  ports:
   -  protocol: TCP
      nodePort: 30080
      port: 8080
      targetPort: 80
```

nodePort Service가 정상적으로 배포되었는지 확인합니다.

```
kubectl -n node-test-01 get services -o wide
NAME               TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE   SELECTOR
node-test-01-svc   NodePort   172.20.138.254   <none>        8080:30080/TCP   37s   app=node-test-01
```

아래와 같은 구성이 배포되었습니다. 외부에 Node IP:30080 으로 노출되어 있으며, Cluster 8080으로 Forwarding됩니다. 이후 Iptable에 의해 Pod들로 80 Port로 로드밸런싱됩니다.

![](<../.gitbook/assets/image (221) (1).png>)

pod shell로 접속해서 Service A Record를 확인해 봅니다.

```
kubectl -n node-test-01 exec -it $NodeTestPod01 -- /bin/sh
/# nslookup 172.20.138.254
```

Pod가 배포된 Node를 AWS 관리콘솔 - EC2 대시보드에서 선택합니다. 해당 EC2 대시보드에서 인스턴스를 선택합니다.

Public-SG 라는 Security Group을 생성하고, 해당 인스턴스에 적용합니다.

![](<../.gitbook/assets/image (223).png>)

Public-SG 라는 이름으로 Security Group을 생성합니다. 

* TCP 30080-30090 허용
* HTTP, HTTPS, ICMP, SSH 허용 

![](<../.gitbook/assets/image (225) (1).png>)

아래와 같이 Security Group이 생성됩니다.

![](<../.gitbook/assets/image (217).png>)

![](<../.gitbook/assets/image (218).png>)

아래와 같이 EC2 Public IP 주소와 Nodeport로 웹 브라우져에서 접속해 봅니다.

![](<../.gitbook/assets/image (221).png>)

## Nodeport 기반 Service 구성 

이제 실제 웹서비스를 배포해 봅니다.

* namespace : nodeport-test
* ecsdemo-frontend service type : nodePort

### 배포용 yaml 복제.

NodePort 타입의 서비스 구성을 위해서 LAB에서 사용할 App을 Cloud9에서 복제합니다.

```
cd ~/environment
git clone https://github.com/whchoi98/eksdemo-frontend.git
git clone https://github.com/whchoi98/eksdemo-nodejs.git
git clone https://github.com/whchoi98/eksdemo-crystal.git

```

## Application 배포 

### 1.namespace 생성 

이제 Namespace를 먼저 생성합니다.

```
kubectl create namespace nodeport-test

```

### 2. Application , Service 배포

nodeport 용 어플리케이션과 서비스를 배포합니다.

```
cd ~/environment/eksdemo-frontend/kubernetes/
kubectl apply -f nodeport_deployment.yaml
kubectl apply -f nodeport_service.yaml

```

정상적으로 배포되었는지 확인해 봅니다.

```
kubectl -n nodeport-test get service

```

아래와 같은 결과를 볼 수 있습니다.

```
 kubectl -n nodeport-test get service
NAME               TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
ecsdemo-frontend   NodePort   172.20.210.120   <none>        80:30081/TCP   10s
```

### 3. 서비스 확인

정상적으로 배포되었는지 확인합니다.

```
kubectl -n nodeport-test get pods

```

```
$ kubectl -n nodeport-test get pods
NAME                               READY   STATUS    RESTARTS   AGE
ecsdemo-frontend-cc96ff7df-ftnt8   1/1     Running   0          44s
```

Output wide 옵션을 통해서 실제 Pod가 배포된 Node를 확인합니다.

```
kubectl -n nodeport-test get pods ecsdemo-frontend-cc96ff7df-ftnt8 -o wide

```

아래 Node 이름을 확인 할 수 있습니다.

```
kubectl -n nodeport-test get pods ecsdemo-frontend-cc96ff7df-ftnt8 -o wide
NAME                               READY   STATUS    RESTARTS   AGE     IP             NODE                                              NOMINATED NODE   READINESS GATES
ecsdemo-frontend-cc96ff7df-ftnt8   1/1     Running   0          2m12s   10.11.36.234   ip-10-11-35-116.ap-northeast-2.compute.internal   <none>           <none>
```

Node 이름을 아래와 같이 상세하게 확인 할 수 있습니다.

```
kubectl -n nodeport-test get pods ecsdemo-frontend-746f9ff7-bpffv -o wide
kubectl get nodes -o wide

```

이제 해당 인스턴스의 공인 IP로 브라우저를 통해서 접근해서 서비스를 확인해 봅니다.

```
node공인ip주소:30081
```

아래와 같은 결과를 확인할 수 있습니다.

![](<../.gitbook/assets/image (225).png>)

이제 Pod를 3개로 늘려서 서비스를 확인해 봅니다.

```
kubectl -n nodeport-test scale deployment ecsdemo-frontend --replicas=3
kubectl -n nodeport-test get pods

```

{% hint style="info" %}
NodePort 30080을 하나의 노드에서만 Security Group으로 허용했는데도, 서비스 분산이 이뤄집니다. 이것은 특정 Node로 Nodeport로 트래픽이 인입하고, 내부에서는 Service를 통해서 부하 분산이 이뤄지고 있는 것입니다.
{% endhint %}
