---
description: 'Update: 2022-04-12 / 40min'
---

# NodePort 기반 배포

## Overview

nodeport 타입의 service는 Node(EC2인스턴스)의 포트를 통해서 서비스로 전달하는 방식입니다. 특정 노드로 유입 시킨 이후에 Service에서 로드밸런싱을 사용할 수 있습니다.하지만 최초 모든 트래픽이 하나의 노드로 집중되기 때문에 이에 대한 설계 고려가 필요합니다.

## Nodeport 이해

nodeport type의 service는 클러스터에서 실행되는 서비스를 Node의 포트를 외부에 노출 시켜서 사용하는 방식입니다. NodePort로 노드그룹의 각 워커 노드의 공인 IP(ENI)에서 서비스를 노출 시키게 됩니다.

이 NodeIP:NodePort는 30000\~32767번 포트를 사용하게 되며, NodePort 서비스가 라우팅 되는 ClusterIP 서비스가 자동 생성됩니다.

### 1.Nodeport 동작 방식&#x20;

* 외부 사용자는 Node(EC2)의 공인 IP/Port로 접근하게 됩니다.&#x20;
* 공인 IP/Port로 접근한 트래픽은 Node(EC2)의 IPTable 규칙에 의해 Cluster IP/Port로 이동합니다.&#x20;
* IPTable 규칙에 의해 PoD 분산하게 됩니다.

![](<../.gitbook/assets/image (343).png>)

아래와 같이 새로운 Namespace와 Pod를 생성합니다.

```
cd ~/environment/myeks/network-test
kubectl create namespace node-test-01
kubectl -n node-test-01 apply -f node-test-01.yaml
kubectl -n node-test-01 get pods
kubectl -n node-test-01 get pods -o wide

```

생성한 pod를 확인합니다.&#x20;

```
kubectl -n node-test-01 get pods -o wide
NAME                            READY   STATUS    RESTARTS   AGE   IP            NODE                                             NOMINATED NODE   READINESS GATES
node-test-01-869b8d5f87-fpww5   1/1     Running   0          19s   10.11.16.79   ip-10-11-28-53.ap-northeast-2.compute.internal   <none>           <none>
node-test-01-869b8d5f87-pp7p9   1/1     Running   0          19s   10.11.41.25   ip-10-11-40-87.ap-northeast-2.compute.internal   <none>           <none>
node-test-01-869b8d5f87-x6mf9   1/1     Running   0          19s   10.11.11.70   ip-10-11-3-14.ap-northeast-2.compute.internal    <none>           <none>
```

shell 연결을 편리하게 접속하기 위해 아래와 같이 cloud9 terminal 의 bash profile에 등록합니다.

```
export NodePort_Test_Pod01=$(kubectl -n node-test-01 get pod -o wide | awk 'NR==2' | awk '/node-test-01/{print $1 } ')
export NodePort_Test_Pod02=$(kubectl -n node-test-01 get pod -o wide | awk 'NR==3' | awk '/node-test-01/{print $1 } ')
export NodePort_Test_Pod03=$(kubectl -n node-test-01 get pod -o wide | awk 'NR==4' | awk '/node-test-01/{print $1 } ')
echo "export NodePort_Test_Pod01=${NodePort_Test_Pod01}" | tee -a ~/.bash_profile
echo "export NodePort_Test_Pod02=${NodePort_Test_Pod02}" | tee -a ~/.bash_profile
echo "export NodePort_Test_Pod03=${NodePort_Test_Pod03}" | tee -a ~/.bash_profile
source ~/.bash_profile

```

### **2.NodePort Service 시험**&#x20;

Nodeport service를 배포합니다.

```
cd ~/environment/myeks/network-test
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
node-test-01-svc   NodePort   172.20.177.168   <none>        8080:30080/TCP   37s   app=node-test-01
```

아래와 같은 구성이 배포되었습니다. 외부에 Node IP:30080 으로 노출되어 있으며, Cluster 8080으로 Forwarding됩니다. 이후 Iptable에 의해 Pod들로 80 Port로 로드밸런싱됩니다.

![](<../.gitbook/assets/image (369).png>)

pod shell로 접속해서 Service A Record를 확인해 봅니다.

```
kubectl -n node-test-01 exec -it $NodePort_Test_Pod01 -- /bin/sh
/# nslookup 172.20.177.168
```

Yaml 파일에 정의된 Service의 NodePort는 EKS Node에서 허용되지 않은 서비스 포트입니다. 허용하기 위해 Security Group을 추가합니다.

Public-SG 라는 이름으로 Security Group을 생성합니다.&#x20;

* TCP 30080-30090 허용

<figure><img src="../.gitbook/assets/image (250).png" alt=""><figcaption></figcaption></figure>

아래와 같이 Security Group이 생성됩니다.

![](<../.gitbook/assets/image (463).png>)

![](<../.gitbook/assets/image (36).png>)

Pod가 배포된 Node를 AWS 관리콘솔 - EC2 대시보드에서 선택합니다. 해당 EC2 대시보드에서 인스턴스를 선택합니다. 이 랩에서는 "eksworkshop-managed-ng-public-01-Node"에 배포됩니다.

생성한 Public-SG 라는 Security Group을 해당 인스턴스에 적용합니다.

```
eksworkshop-managed-ng-public-01-Node
```

<figure><img src="../.gitbook/assets/image (135).png" alt=""><figcaption></figcaption></figure>

eksworkshop-managed-ng-public-01-Node 들의 EIP를 확인합니다.

```
aws ec2 describe-instances --filters 'Name=tag:Name,Values=eksworkshop-managed-ng-public-01-Node' | jq -r '.Reservations[].Instances[].PublicIpAddress'

```

아래와 같이 EC2 Public IP 주소와 Nodeport로 접속해 봅니다.

```
curl http://52.79.206.201:30080
Praqma Network MultiTool (with NGINX) - node-test-01-869b8d5f87-qbskb - 10.11.38.70
curl http://3.38.195.240:30080
Praqma Network MultiTool (with NGINX) - node-test-01-869b8d5f87-646t7 - 10.11.11.11
curl http://13.125.40.125:30080
Praqma Network MultiTool (with NGINX) - node-test-01-869b8d5f87-qbskb - 10.11.38.70

```

{% hint style="info" %}
왜 고르게 로드밸런싱이 안될까요?

NodePort로 인입된 후 ClusterIP에서 다시 LB되기 때문입니다.&#x20;
{% endhint %}

Node에서 iptable에 설정된 NAT Table, Loadbalancing 구성을 확인해 봅니다.

```
aws ssm start-session --target $ng_public01_id
sudo -s
iptables -t nat -L --line-number | more
iptables -t nat -L --line-number | grep node-test-01-svc

```

## Nodeport 기반 Service 구성&#x20;

이제 실제 웹서비스를 배포해 봅니다.

* namespace : nodeport-test
* ecsdemo-frontend service type : nodePort

### 3.배포용 yaml 복제.

NodePort 타입의 서비스 구성을 위해서 LAB에서 사용할 App을 Cloud9에서 복제합니다.

```
cd ~/environment
git clone https://github.com/whchoi98/eksdemo-frontend.git
git clone https://github.com/whchoi98/eksdemo-nodejs.git
git clone https://github.com/whchoi98/eksdemo-crystal.git

```

## Application 배포&#x20;

### 4.namespace 생성&#x20;

이제 Namespace를 먼저 생성합니다.

```
kubectl create namespace nodeport-test

```

### 5. Application , Service 배포

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

### 6. 서비스 확인

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

이제 해당 인스턴스의 공인 IP로 브라우저를 통해서 접근해서 서비스를 확인해 봅니다. Node의 IP 주소는 EC2 서비스 대시 보드에서 확인 할 수 있습니다.

<figure><img src="../.gitbook/assets/image (175).png" alt=""><figcaption></figcaption></figure>

eksworkshop-managed-ng-public-01-node 들의 EIP를 확인합니다.

```
~/environment/useful-shell/aws_ec2_text.sh | awk '/eksworkshop-managed-ng-public-01-Node/{print $1,$2,$6,$7,$8}' 
```

```
~/environment/useful-$ ~/environment/useful-shell/aws_ec2_text.sh | awk '/eksworkshop-managed-ng-public-01-Node/{print $1,$2,$6,$7,$8}'                                                    
eksworkshop-managed-ng-public-01-Node ap-northeast-2a running 10.11.13.125 3.38.193.206
eksworkshop-managed-ng-public-01-Node ap-northeast-2b running 10.11.26.163 3.34.195.59
eksworkshop-managed-ng-public-01-Node ap-northeast-2c running 10.11.41.213 52.78.36.141
```

eksworkshop-ng-public-01-Node 의 IP 주소를 확인하고 , 브라우저에서 아래와 같이 주소를 입력합니다.

```
node공인ip주소:30081
```

아래와 같은 결과를 확인할 수 있습니다. Pod를 1개 배포했기 때문에 1개의 Pod로 라우팅 되는 것을 확인할 수 있습니다.

![](<../.gitbook/assets/image (453).png>)

이제 Pod를 3개로 늘려서 서비스를 확인해 봅니다.

```
kubectl -n nodeport-test scale deployment ecsdemo-frontend --replicas=3
kubectl -n nodeport-test get pods

```

Pod 3개로 브라우저에서 정상적으로 서비스 되는지 확인해 봅니다.

{% hint style="info" %}
NodePort 30080\~30081을 하나의 노드에서만 Security Group으로 허용했는데도, 서비스 분산이 이뤄집니다. 이것은 특정 Node로 Nodeport로 트래픽이 인입하고, 내부에서는 Service를 통해서 부하 분산이 이뤄지고 있는 것입니다.
{% endhint %}
