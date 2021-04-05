# Cluster IP 기반 배포

## 개요 

Kubernetes에서는 Pod의 전면에서 Pod로 트래픽이 들어오는 트래픽을 전달하는 service 자원이 제공됩니다. 해당 Service 자원은 Pod의 IP 주소와 관계 없이 Pod의 Label Selector를 보고 트래픽을 전달하는 역할을 담당합니다.

Service의 종류는 아래와 같습니다.

* Cluster IP - Service 자원의 기본 타입이며 Kubernetes 내부에서만 접근 가
* NodePort - 로컬 호스트의 특정 포트를 Serivce의 특정 포트와 연결
* Loadbalancer - AWS CLB, NLB 등과 같은 로드밸런서가 노드 전면에서 처리하는 방식

아래 그림에서 처럼 Service의 기본은 CLUSTER-IP 방식입니다. 외부로 노출되지 않으며, Service에는  Pod Container의 포트를 기술해 줍니다.

![Cluster IP &#xD0C0;&#xC785; &#xAE30;&#xBC18; &#xC11C;&#xBE44;&#xC2A4;](../.gitbook/assets/image%20%28176%29.png)

## CoreDNS와 Service

아래와 같은 구성을 만들고, Cluster IP의 동작에 대해 이해할 수 있습니다.

![](../.gitbook/assets/image%20%28170%29.png)

### 1.CoreDNS와 Service 역할 확인을 위한 App배포 

아래와 같이 새로운 Namespace와 Pod를 생성합니다.

```text
cd ~/environment/myeks/network-test
kubectl create namespace network-test
kubectl -n network-test apply -f test-deployment-01.yaml
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

이제 Service를 배포합니다.

```text
kubectl apply -f test-service-01.yaml

```

다시 앞서 생성한 Container에서 service의 nameservice를 조회해 봅니다.

```text
nslookup
alpine-app-svc-01

```

이제 한개의 cluster 서비스를 더 배포해 봅니다.

```text
cd ~/environment/myeks/network-test
kubectl apply -f test-deployment-02.yaml 
kubectl apply -f test-service-02.yaml

```

Container 에서 curl 을 통해 자신이 속한 service와 다른 service로 연결해 봅니다.

```text
curl alpine-app-svc-02:8080
curl alpine-app-svc-01:8080

```

