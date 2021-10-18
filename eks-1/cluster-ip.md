---
description: 'Update: 2021-05-22'
---

# Cluster IP 기반 배포

## 개요 

Kubernetes에서는 Pod의 전면에서 Pod로 들어오는 트래픽을 전달하는 service 자원이 제공됩니다. 해당 Service 자원은 Pod의 IP 주소와 관계 없이 Pod의 Label Selector를 보고 트래픽을 전달하는 역할을 담당합니다.

Service의 종류는 아래와 같습니다.

* Cluster IP - Service 자원의 기본 타입이며 Kubernetes 내부에서만 접근 가능 
* NodePort - 로컬 호스트의 특정 포트를 Serivce의 특정 포트와 연결
* Loadbalancer - AWS CLB, NLB 등과 같은 로드밸런서가 노드 전면에서 처리하는 방식

아래 그림에서 처럼 Service의 기본은 CLUSTER-IP 방식입니다. 외부로 노출되지 않으며, Service에는  Pod Container의 포트를 기술해 줍니다.

![Cluster IP 타입 기반 서비스](<../.gitbook/assets/image (179).png>)

## CoreDNS와 Service

아래와 같은 구성을 만들고, Cluster IP의 동작에 대해 이해할 수 있습니다.

![](<../.gitbook/assets/image (219).png>)

![](<../.gitbook/assets/image (173).png>)

### 1.CoreDNS와 Service 역할 확인을 위한 App배포 

아래와 같이 새로운 Namespace와 Pod를 생성합니다.

```
cd ~/environment/myeks/network-test
kubectl create namespace cluster-test-01
kubectl -n cluster-test-01 apply -f cluster-test-01.yaml
kubectl -n cluster-test-01 get pods
kubectl -n cluster-test-01 get pods -o wide

```

정상적으로 Pods가 생성되었는지 확인합니다.

```
kubectl -n cluster-test-01 get pods
NAME                               READY   STATUS    RESTARTS   AGE
cluster-test-01-6f4dddc749-pfq77   1/1     Running   0          36s
cluster-test-01-6f4dddc749-s8hkp   1/1     Running   0          36s
cluster-test-01-6f4dddc749-tq8jj   1/1     Running   0          36s

kubectl -n cluster-test-01 get pods -o wide
NAME                               READY   STATUS    RESTARTS   AGE     IP              NODE                                               NOMINATED NODE   READINESS GATES
cluster-test-01-6f4dddc749-pfq77   1/1     Running   0          7m35s   10.11.107.227   ip-10-11-108-153.ap-northeast-2.compute.internal   <none>           <none>
cluster-test-01-6f4dddc749-s8hkp   1/1     Running   0          7m35s   10.11.29.107    ip-10-11-21-111.ap-northeast-2.compute.internal    <none>           <none>
cluster-test-01-6f4dddc749-tq8jj   1/1     Running   0          7m35s   10.11.1.72      ip-10-11-3-68.ap-northeast-2.compute.internal      <none>           <none>
```

shell 연결을 편리하게 접속하기 위해 아래와 같이 cloud9 terminal 의 bash profile에 등록합니다.

```
echo "export ClusterTestPod03=cluster-test-01-6f4dddc749-pfq77" | tee -a ~/.bash_profile
echo "export ClusterTestPod02=cluster-test-01-6f4dddc749-s8hkp" | tee -a ~/.bash_profile
echo "export ClusterTestPod01=cluster-test-01-6f4dddc749-tq8jj" | tee -a ~/.bash_profile
source ~/.bash_profile
```

각각의 Pod로 접속해 봅니다.

```
kubectl -n cluster-test-01 exec -it $ClusterTestPod01 -- /bin/sh
```

ip a 와 /etc/resolve.conf를 조회해 봅니다.

```
ip a
cat /etc/resolv.conf
```

다음과 같이 출력됩니다.

```
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
##Pod의 IP 주소 입니다.
3: eth0@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP group default 
    link/ether ca:63:b0:58:89:81 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.11.1.72/32 scope global eth0
       valid_lft forever preferred_lft forever
       
# cat /etc/resolv.conf 
## Coredns 주소 입니다.
nameserver 172.20.0.10
## Pod의 A Record 입니다.
search cluster-test-01.svc.cluster.local svc.cluster.local cluster.local ap-northeast-2.compute.internal
options ndots:5
```

한개의 Pod에 더 연결해 보고 동일하게 비교해 봅니다.

```
kubectl -n cluster-test-01 exec -it $ClusterTestPod02 -- /bin/sh

```

```
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP group default 
    link/ether e6:69:08:ca:10:9f brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.11.29.107/32 scope global eth0
       valid_lft forever preferred_lft forever
# cat /etc/resolv.conf 
nameserver 172.20.0.10
search cluster-test-01.svc.cluster.local svc.cluster.local cluster.local ap-northeast-2.compute.internal
options ndots:5
```

AWS VPC CNI 구성은 Pod 생성할 때 마다 ENI를 생성하므로, Pod간 IP 직접 통신이 가능합니다.

```
## ClusterTestPod01 컨테이너에서 ClusterTestPod02로 Ping Test

/ # ping 10.11.29.107
PING 10.11.29.107 (10.11.29.107) 56(84) bytes of data.
64 bytes from 10.11.29.107: icmp_seq=1 ttl=253 time=0.951 ms
64 bytes from 10.11.29.107: icmp_seq=2 ttl=253 time=0.655 ms

```

이제 Service를 배포합니다.

```
kubectl apply -f ~/environment/myeks/network-test/test-service-01.yaml
```

다시 앞서 생성한 Container에서 service의 nameservice를 조회해 봅니다.

```
nslookup
alpine-app-svc-01

```

이제 한개의 cluster 서비스를 더 배포해 봅니다.

```
cd ~/environment/myeks/network-test
kubectl apply -f test-deployment-02.yaml 
kubectl apply -f test-service-02.yaml

```

Container 에서 curl 을 통해 자신이 속한 service와 다른 service로 연결해 봅니다.

```
curl alpine-app-svc-02:8080
curl alpine-app-svc-01:8080

```

