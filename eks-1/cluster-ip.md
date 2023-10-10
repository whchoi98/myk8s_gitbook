---
description: 'Update: 2022-06-01 / 30min'
---

# Cluster IP 기반 배포

## 개요&#x20;

Kubernetes에서는 Pod의 전면에서 Pod로 들어오는 트래픽을 전달하는 service 자원이 제공됩니다. 해당 Service 자원은 Pod의 IP 주소와 관계 없이 Pod의 Label Selector를 보고 트래픽을 전달하는 역할을 담당합니다.

Service의 종류는 아래와 같습니다.

* Cluster IP - Service 자원의 기본 타입이며 Kubernetes 내부에서만 접근 가능&#x20;
* NodePort - 로컬 호스트의 특정 포트를 Serivce의 특정 포트와 연결
* Loadbalancer - AWS CLB, NLB 등과 같은 로드밸런서가 노드 전면에서 처리하는 방식

아래 그림에서 처럼 Service의 기본은 CLUSTER-IP 방식입니다. 외부로 노출되지 않으며, Service에는  Pod Container의 포트를 기술해 줍니다.

![](<../.gitbook/assets/image (380).png>)

<figure><img src="../.gitbook/assets/image (169).png" alt=""><figcaption></figcaption></figure>

## CoreDNS와 Service

### 1.CoreDNS와 Service 역할 확인을 위한 App배포&#x20;

아래와 같이 새로운 Namespace와 Pod를 생성합니다.

```
cd ~/environment/myeks/network-test
kubectl apply -f cluster-test-01.yaml
kubectl -n cluster-test-01 get pods
kubectl -n cluster-test-01 get pods -o wide

```

정상적으로 Pods가 생성되었는지 확인합니다.

```
kubectl -n cluster-test-01 get pods -o wide
NAME                              READY   STATUS    RESTARTS   AGE   IP             NODE                                             NOMINATED NODE   READINESS GATES
cluster-test-01-b86b9c685-8g7rt   1/1     Running   0          94s   10.11.1.103    ip-10-11-3-14.ap-northeast-2.compute.internal    <none>           <none>
cluster-test-01-b86b9c685-lkm56   1/1     Running   0          95s   10.11.38.204   ip-10-11-40-87.ap-northeast-2.compute.internal   <none>           <none>
cluster-test-01-b86b9c685-vxgxq   1/1     Running   0          94s   10.11.16.103   ip-10-11-28-53.ap-northeast-2.compute.internal   <none>           <none>
```

(Option) shell 연결을 편리하게 접속하기 위해 아래와 같이 cloud9 terminal 의 bash profile에 등록합니다.

```
export ClusterTestPod01=$(kubectl -n cluster-test-01 get pod -o wide | awk 'NR==2' | awk '/cluster/{print $1 } ')
export ClusterTestPod02=$(kubectl -n cluster-test-01 get pod -o wide | awk 'NR==3' | awk '/cluster/{print $1 } ')
export ClusterTestPod03=$(kubectl -n cluster-test-01 get pod -o wide | awk 'NR==4' | awk '/cluster/{print $1 } ')
echo "export ClusterTestPod01=${ClusterTestPod01}" | tee -a ~/.bash_profile
echo "export ClusterTestPod02=${ClusterTestPod02}" | tee -a ~/.bash_profile
echo "export ClusterTestPod03=${ClusterTestPod03}" | tee -a ~/.bash_profile
source ~/.bash_profile

```

K9s로 접속하거나 bash profile에 등록한 컨테이너들로 각각의 Pod로 접속해 봅니다.&#x20;

```
kubectl -n cluster-test-01 exec -it $ClusterTestPod01 -- /bin/sh
```

앞서 설치한 K9s에서 container에 접속해 봅니다.

```
k9s -n cluster-test-01

```

<figure><img src="../.gitbook/assets/image (183).png" alt=""><figcaption></figcaption></figure>

ip a 와 /etc/resolve.conf를 조회해 봅니다.

```
ip a
cat /etc/resolv.conf

```

다음과 같이 출력됩니다.

```
## ClusterTestPod01##
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
##Pod의 IP 주소 입니다.
3: eth0@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP group default 
    link/ether ca:63:b0:58:89:81 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.11.5.202/32 scope global eth0
       valid_lft forever preferred_lft forever
       
# cat /etc/resolv.conf 
## Coredns 주소 입니다.
nameserver 172.20.0.10
## Pod의 A Record 입니다.
search cluster-test-01.svc.cluster.local svc.cluster.local cluster.local ap-northeast-2.compute.internal
options ndots:5
```

K9s로 접속하거나 bash profile에 등록한 다른 Pod에 더 연결해 보고 동일하게 비교해 봅니다.

```
kubectl -n cluster-test-01 exec -it $ClusterTestPod02 -- /bin/sh

```

```
## ClusterTestPod02
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP group default 
    link/ether e6:69:08:ca:10:9f brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.11.26.206/32 scope global eth0
       valid_lft forever preferred_lft forever
# cat /etc/resolv.conf 
nameserver 172.20.0.10
search cluster-test-01.svc.cluster.local svc.cluster.local cluster.local ap-northeast-2.compute.internal
options ndots:5
```

AWS VPC CNI 구성은 Pod 생성할 때 마다 ENI를 생성하므로, Pod간 IP 직접 통신이 가능합니다.

```
## ClusterTestPod01 컨테이너에서 ClusterTestPod02로 Ping Test

/ # ping 10.11.26.206
PING 10.11.29.107 (10.11.29.107) 56(84) bytes of data.
64 bytes from 10.11.29.107: icmp_seq=1 ttl=253 time=0.951 ms
64 bytes from 10.11.29.107: icmp_seq=2 ttl=253 time=0.655 ms

```

## ClusterIP Type

### 2.ClusterType 소개 &#x20;

Kubernetes 에서 Service는 Pod들이 실행 중인 애플리케이션을 네트워크 서비스로 노출하는 추상적인 방법입니다. 다양한 Service 들 중에서 ClusterIP는 Service - Type의 기본 값입니다.

ClusterIP 타입은 내부에서 사용하도록 노출되며 , 외부에 노출되지 않습니다.

* ClusterIP 타입 종류 - UserSpace Proxy 모드 , iptables Proxy 모드, IPVS Proxy 모드.

### 3.ClusterIP Server 시험 &#x20;

아래와 같이 ClusterIP 서비스를 배포합니다.

```
cd ~/environment/myeks/network-test/
kubectl -n cluster-test-01 apply -f cluster-test-01-service.yaml

```

ClusterIP Service에 대한 yaml 파일은 아래와 같습니다.&#x20;

```
apiVersion: v1
kind: Service
metadata:
  name: cluster-test-01-svc
  namespace: cluster-test-01
spec:
  selector:
    app: cluster-test-01
  ports:
   -  protocol: TCP
      port: 8080
      targetPort: 80
```

ClusterIP Service가 정상적으로 배포되었는지 확인합니다.

```
kubectl -n cluster-test-01 get services -o wide

```

아래와 같이 출력됩니다.&#x20;

```
kubectl -n cluster-test-01 get services -o wide
NAME                  TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE     SELECTOR
cluster-test-01-svc   ClusterIP   172.20.4.192   <none>        8080/TCP   3h26m   app=cluster-test-01
```

ClusterIP service의 A Record는

&#x20;**`"ClusterIP Metadata.name"."namesapce".svc.cluster.local.`** 의 형식을 사용하게 됩니다.

![](<../.gitbook/assets/image (77).png>)

ClusterTest01 로 접속후, "cluster-test-01-svc" ClusterIP Service A Record를 확인합니다. curl을 통해서 Loadbalancing이 정상적으로 이뤄지는지 curl을 통해서 확인해 봅니다.

```
kubectl -n cluster-test-01 exec -it $ClusterTestPod01 -- /bin/sh
nslookup {CLUSTER-IP}
```

```
## ClusterTest01에서 ClusterIP Service A Record 확인.
#ClusterTestPod01
# nslookup 172.20.120.4
4.120.20.172.in-addr.arpa       name = cluster-test-01-svc.cluster-test-01.svc.cluster.local.

/ # curl cluster-test-01-svc.cluster-test-01.svc.cluster.local.:8080
Praqma Network MultiTool (with NGINX) - cluster-test-01-6f4dddc749-tq8jj - 10.11.1.72
/ # curl cluster-test-01-svc.cluster-test-01.svc.cluster.local.:8080
Praqma Network MultiTool (with NGINX) - cluster-test-01-6f4dddc749-s8hkp - 10.11.29.107
/ # curl cluster-test-01-svc.cluster-test-01.svc.cluster.local.:8080
Praqma Network MultiTool (with NGINX) - cluster-test-01-6f4dddc749-pfq77 - 10.11.107.227
```

iptable에 설정된 NAT Table, Loadbalancing 구성을 확인해 봅니다.

```
## git clone ###
cd ~/environment/
git clone https://github.com/whchoi98/useful-shell
## Node 정보 확인 ##
kubectl -n cluster-test-01 get pods -o wide

## Node instance id 확인 ##
export ng_public01=10.11.3.14
export ng_public02=10.11.28.53
export ng_public03=10.11.40.87
echo "export ng_public01=${ng_public01}" | tee -a ~/.bash_profile
echo "export ng_public02=${ng_public02}" | tee -a ~/.bash_profile
echo "export ng_public03=${ng_public03}" | tee -a ~/.bash_profile
~/environment/useful-shell/aws_ec2_text.sh | awk '/10.11.3.14/{print $1,$2,$3,$7}'
~/environment/useful-shell/aws_ec2_text.sh | awk '/10.11.28.53/{print $1,$2,$3,$7}'
~/environment/useful-shell/aws_ec2_text.sh | awk '/10.11.40.87/{print $1,$2,$3,$7}'
## 위에서 출력된 instance-id 값을 입력합니다. ##
export ng_public01_id=i-03ab12d2f4e14f7dd
export ng_public02_id=i-0879b78d9d79712f0
export ng_public03_id=i-0fcf7a3a914e6c44b
echo "export ng_public01_id=${ng_public01_id}" | tee -a ~/.bash_profile
echo "export ng_public02_id=${ng_public02_id}" | tee -a ~/.bash_profile
echo "export ng_public03_id=${ng_public03_id}" | tee -a ~/.bash_profile
source ~/.bash_profile

aws ssm start-session --target $ng_public01_id
sudo -s
iptables -t nat -nvL --line-number | more
iptables -t nat -nvL --line-number | grep cluster-test-01-svc
iptables -t nat -nvL KUBE-SERVICES
```
