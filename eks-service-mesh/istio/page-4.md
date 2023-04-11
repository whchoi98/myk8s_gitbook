---
description: 'Update : 2022-09-24'
---

# istio 설치와 구성

## Istio 설치/구성&#x20;

### 1.istio Client 구성

istio 설치를 위해 istio binary를 Cloud9에 다운로드 받습니다. kubectl binary를 Cloud9, 로컬PC에 설치하는 것과 동일합니다

Cloud9에서 아래와 같이 실행시킵니다. istio version 1.13.5 기준으로 설치하는 예제입니다.&#x20;

```
# istio version 1.14.4 기준
echo 'export ISTIO_VERSION="1.14.4"' | tee -a ~/.bash_profile
source ~/.bash_profile

cd ~/environment
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=${ISTIO_VERSION} TARGET_ARCH=x86_64 sh -

```

{% hint style="info" %}
istio 버전은 분기에 한번씩 마이너 릴리즈를 제공하고 있습니다. Kubernetes 버전과 API 지원 종속성을 가지기 때문에 확인해야 합니다.  아래 링크를 통해서 확인이 가능합니다

[https://istio.io/latest/docs/releases/supported-releases/#supported-releases/](https://istio.io/latest/docs/releases/supported-releases/#supported-releases/)
{% endhint %}

다운로드 받은 Binary안에는 istioctl이 포함되어 있습니다.

```
cd ${HOME}/environment/istio-${ISTIO_VERSION}
export PATH=$PWD/bin:$PATH
sudo cp -v bin/istioctl /usr/local/bin/

```

정상적으로 설치되었는지 아래 명령을 통해 확인합니다.

```
istioctl version

```

아래와 같은 결과를 확인할 수 있습니다

```
$ istioctl version
no running Istio pods in "istio-system"
1.14.4
```

### 2. Istio Profile 구성

Istio는 Profile이라는 4가지 설정이 있고 여기서 필요한 모든 것을 정의하도록 되어 있습니다

* default :  IstioOperator API의 기본 설정에 따라 구성 요소를 활성화합니다. 이 프로필은 프로덕션 배포 및 멀티클러스터 메시의 기본 클러스터에 권장됩니다. istioctl profile dump 명령을 실행하여 기본 설정을 표시할 수 있습니다.
* demo: 적절한 리소스 요구 사항으로 Istio 기능을 보여주도록 설계된 구성입니다. Bookinfo 응용 프로그램 및 관련 작업을 실행하는 데 적합합니다. 이것은 Quick start와 함께 설치되는 구성입니다. 이 프로필은 높은 수준의 추적 및 액세스 로깅을 가능하게 하므로 성능 테스트에 적합하지 않습니다.
* minimal: : 기본 프로필과 동일하지만 Control Plane 만 설치됩니다. 이를 통해 별도의 프로필을 사용하여 컨트롤 플레인 및 데이터 플레인 구성 요소(예: Gateway)를 구성할 수 있습니다.
* external: 다중 클러스터 메시의 기본 클러스터에 있는 Control Plane 또는 외부 Control Plane에서 관리하는 원격 클러스터를 구성하는 데 사용됩니다.
* empty : 아무 것도 배포하지 않습니다. 이것은 사용자 지정 구성을 위한 기본 프로필로 유용할 수 있습니다.
* perview : perview 프로필에는 실험적인 기능이 포함되어 있습니다. 이는 Istio에 제공되는 새로운 기능을 경험하기 위한 것입니다. 안정성, 보안 및 성능이 보장되지 않습니다. 사용에 따른 위험은 사용자가 감수해야 합니다.

![](<../../.gitbook/assets/image (232) (1).png>)

각 프로파일들은 diff 명령을 통해서 비교가 가능합니다.&#x20;

```
# default 와 demo profile을 비교 
istioctl profile diff default demo
```

istio demo 구성 프로파일을 설치하고 확인해 봅니다.&#x20;

```
istioctl install --set profile=demo -y
kubectl -n istio-system get pod,svc

```

아래와 같은 결과가 출력됩니다.&#x20;

```
$ kubectl -n istio-system get pod,svc
NAME                                        READY   STATUS    RESTARTS   AGE
pod/istio-egressgateway-649b7b8dd5-cl4wx    1/1     Running   0          14s
pod/istio-ingressgateway-75758855d7-bjxfj   1/1     Running   0          15s
pod/istiod-59bd9654d7-qsfhz                 1/1     Running   0          27s

NAME                           TYPE           CLUSTER-IP       EXTERNAL-IP                                                                  PORT(S)                                                                      AGE
service/istio-egressgateway    ClusterIP      172.20.201.1     <none>                                                                       80/TCP,443/TCP                                                               14s
service/istio-ingressgateway   LoadBalancer   172.20.101.189   ab31b810421fa465c9c6fd8f82f4ad3e-23812066.ap-northeast-2.elb.amazonaws.com   15021:32050/TCP,80:30601/TCP,443:30861/TCP,31400:30474/TCP,15443:31660/TCP   14s
service/istiod                 ClusterIP      172.20.212.253   <none>                                                                       15010/TCP,15012/TCP,443/TCP,15014/TCP 
```

아래 명령을 통해 현재 구성된 프로파일을 확인해 볼 수 있습니다

```
istioctl profile dump demo

```

### 3. istio operator 구성&#x20;

istio설정을 file로 관리하는 기능을 제공하고 있으며, 이것을 istio Operator라고 부릅니다.&#x20;

```
istioctl operator init
kubectl -n istio-operator get all

```

아래와 같이 설치가 구성됩니다.&#x20;

```
$ istioctl operator init
Installing operator controller in namespace: istio-operator using image: docker.io/istio/operator:1.13.4
Operator controller will watch namespaces: istio-system
✔ Istio operator installed                                                                                                                                                                                                     
✔ Installation complete
```

이제 istio를 삭제하고, yaml로 istio demo profile을 설치해 봅니다.&#x20;

```
#istio 삭제 
kubectl delete namespaces istio-system

```

아래와 같이 istio operator로 kubectl을 이용해서 istio demo profile을 설치해 봅니다.

```
istioctl operator init
kubectl create namespace istio-system
kubectl apply -f - <<EOF
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: example-istiocontrolplane
spec:
  profile: demo
EOF

```

### 4. istio injection 구성 이해&#x20;

Istio의 모든 기능을 활용하려면 Pod에서 Istio 사이드카 프록시를 실행해야 합니다.

&#x20;Istio 사이드카를 포드에 삽입하는 두 가지 방법은 포드의 네임스페이스에서 자동 Istio 사이드카 삽입을 활성화하거나 istioctl 명령으로 수동으로 사용하는 방법이 있습니다.&#x20;

포드의 네임스페이스에서 활성화된 경우 istio proxy 자동 삽입은 승인 컨트롤러를 사용하여 포드 생성 시 프록시 구성을 주입합니다. 수동 주입은 프록시 구성을 추가하여 배포와 같은 구성을 직접 수정합니다.

어떤 것을 사용해야 할지 잘 모르겠다면 자동 주입을 권장합니다.

Istio에서 제공하는  Webhook admission Controller를 사용하여 사이드카를 적용 가능한 Kubernetes 포드에 자동으로 추가할 수 있습니다. 네임스페이스에 istio-injection=enabled 레이블을 설정하고 Injection Webhook이 활성화되면 해당 네임스페이스에서 생성되는 모든 새로운 Pod에는 자동으로 사이드카가 추가됩니다.

![](<../../.gitbook/assets/image (239) (1) (1) (1).png>)

수동 주입과 달리 자동 주입은 Pod 수준에서 발생합니다.&#x20;

```
#default namespace 에 자동 injection
kubectl label namespace default istio-injection=enabled --overwrite

```

아래와 같이 Pod Deployment의 metadata에 설정할 수도 있습니다.&#x20;

```
  template:
    metadata:
      labels:
        app: istio-test01-app
        sidecar.istio.io/inject: "false"
```

아래 예제를 통해 Pod를 배포해 봅니다.&#x20;

```
kubectl apply -f ${HOME}/environment/istio-${ISTIO_VERSION}/samples//sleep/sleep.yaml
kubectl get pods

```

아래 처럼 Istio proxy는 주입되지 않고, Pod에 1개의 Container만 구동합니다.&#x20;

```
$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
sleep-557747455f-hz28q   1/1     Running   0          9s

```

아래 예제를 통해 Istio proxy를 주입해 봅니다.&#x20;

```
kubectl label namespace default istio-injection=enabled --overwrite
kubectl delete pod -l app=sleep
kubectl get pod -l app=sleep
kubectl get pods

```

아래에서 처럼 Pod에 Proxy가 주입되었습니다. 1개의 Pod안에 2개의 Container를 확인할 수 있습니다.&#x20;

```
$ kubectl get pod -l app=sleep
NAME                     READY   STATUS    RESTARTS   AGE
sleep-557747455f-gncrp   2/2     Running   0          72s
```

아래에서 처럼 새로운 Proxy를 확인할 수 있습니다.&#x20;

```
$ kubectl logs -f sleep-557747455f-gncrp istio-proxy | grep "Envoy"
2022-06-20T13:39:02.629088Z     info    Envoy command: [-c etc/istio/proxy/envoy-rev0.json --restart-epoch 0 --drain-time-s 45 --drain-strategy immediate --parent-shutdown-time-s 60 --local-address-ip-version v4 --file-flush-interval-msec 1000 --disable-hot-restart --log-format %Y-%m-%dT%T.%fZ %l      envoy %n        %v -l warning --component-log-level misc:error --concurrency 2]
2022-06-20T13:39:05.217811Z     info    Envoy proxy is ready
```

Pod에 주입해서도 Apply 할 수 있습니다.&#x20;

```
  template:
    metadata:
      labels:
        app: istio-test01-app
        sidecar.istio.io/inject: "true"	
```

default namespace를 default로 변경합니다. &#x20;

```
kubectl label namespace default istio-injection-
kubectl delete pod -l app=sleep
kubectl get pod

```



## Bookinfo Sample App

istio 공식 사이트에서는 Sample App을 제공합니다.

![](<../../.gitbook/assets/image (227) (1).png>)

### 5. Sample App 설치

istio binary를 설치하고 나면 samples 디렉토리에 bookinfo 앱이 포함되어 있습니다. 이 앱을 설치해 봅니다.&#x20;

```
kubectl create namespace bookinfo
kubectl label namespace bookinfo istio-injection=enabled
kubectl get namespace -L istio-injection
kubectl get ns bookinfo --show-labels
kubectl -n bookinfo apply \
  -f ${HOME}/environment/istio-${ISTIO_VERSION}/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl -n bookinfo get pod,svc
  
```

아래와 같은 Pod와 Service들이 배포 됩니다.&#x20;

```
$ kubectl -n bookinfo get pod,svc
NAME                                  READY   STATUS    RESTARTS   AGE
pod/details-v1-7d88846999-5rw28       2/2     Running   0          7m56s
pod/productpage-v1-7795568889-zwzgx   2/2     Running   0          7m56s
pod/ratings-v1-754f9c4975-lmhl7       2/2     Running   0          7m56s
pod/reviews-v1-55b668fc65-6np7d       2/2     Running   0          7m56s
pod/reviews-v2-858f99c99-rvxvz        2/2     Running   0          7m56s
pod/reviews-v3-7886dd86b9-j4fk7       2/2     Running   0          7m56s

NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/details       ClusterIP   172.20.32.154   <none>        9080/TCP   7m56s
service/productpage   ClusterIP   172.20.5.101    <none>        9080/TCP   7m56s
service/ratings       ClusterIP   172.20.14.146   <none>        9080/TCP   7m56s
service/reviews       ClusterIP   172.20.134.44   <none>        9080/TCP   7m56s
```

이제 Gateway와 Virtual Service를 배포합니다.&#x20;

```
kubectl -n bookinfo \
 apply -f ${HOME}/environment/istio-${ISTIO_VERSION}/samples/bookinfo/networking/bookinfo-gateway.yaml

kubectl -n bookinfo get gateways,virtualservices
kubectl -n istio-system get services

```

아래와 같이 출력됩니다.&#x20;

```
$ kubectl -n bookinfo get gateways,virtualservices
NAME                                           AGE
gateway.networking.istio.io/bookinfo-gateway   4m26s

NAME                                          GATEWAYS               HOSTS   AGE
virtualservice.networking.istio.io/bookinfo   ["bookinfo-gateway"]   ["*"]   4m27s

$ kubectl -n istio-system get services
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP                                                                  PORT(S)                                                                      AGE
istio-egressgateway    ClusterIP      172.20.9.165   <none>                                                                       80/TCP,443/TCP                                                               47m
istio-ingressgateway   LoadBalancer   172.20.26.89   a36f3d5b98fc24cb29851355830f685e-55560905.ap-northeast-2.elb.amazonaws.com   15021:31074/TCP,80:32168/TCP,443:31397/TCP,31400:30472/TCP,15443:31709/TCP   47m
istiod                 ClusterIP      172.20.69.47   <none>                                                                       15010/TCP,15012/TCP,443/TCP,15014/TCP                                        47m
```

접속할 Product Page URL을 확인합니다.&#x20;

```
export GATEWAY_URL=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "http://${GATEWAY_URL}/productpage"

```

![](<../../.gitbook/assets/image (240) (1).png>)

EC2-로드밸런싱-로드밸런서에서 확인이 가능합니다.&#x20;

![](<../../.gitbook/assets/image (228) (1).png>)

### 6. istio flow 파악

아래에서 ingress gateway의 실제 Endpoint를 확인합니다.&#x20;

```
 kubectl -n istio-system get pods -o wide
 
```

아래 처럼 결과를 확인 할 수 있습니다.&#x20;

```
## 10.11.88.41 주소가 istio-ingressgateway 입니다. 
$ kubectl -n istio-system get pods -o wide
NAME                                    READY   STATUS    RESTARTS   AGE    IP             NODE                                            NOMINATED NODE   READINESS GATES
istio-egressgateway-859f895667-79rd6    1/1     Running   0          108m   10.11.89.163   ip-10-11-90-2.ap-northeast-2.compute.internal   <none>           <none>
istio-ingressgateway-6565cc5469-frp4w   1/1     Running   0          108m   10.11.88.41    ip-10-11-90-2.ap-northeast-2.compute.internal   <none>           <none>
istiod-54588fb7bf-286r7                 1/1     Running   0          108m   10.11.95.4     ip-10-11-90-2.ap-northeast-2.compute.internal   <none>           <none>
```

이제 워커 노드에서 경로를 확인해 봅니다.  워커노드를 SSH 또는 Session Manager로 접속해 봅니다.&#x20;

```
# NodePort확인을 위해서 EKS WorkerNode에 접속
# 80 port 확인
iptables -t nat -nvL KUBE-SERVICES | grep 80
#istio-gateway endpoint
iptables -t nat -nvL KUBE-SVC-xxxxxxxxx

```

아래와 같이 결과를 확인 합니다.

* CLB - NodePort - IstioIngress Gateway

IPTable에서 구성된 Proxy로 연결되지 않고 직접 Istio-Ingress Gateway로 포워딩 됩니다.&#x20;

```
# 각 노드들은 Kubernetes Service로 유입되는 80포트를 IPTable의 체인으로 보냅니다. 
[root@ip-10-11-12-72 bin]# iptables -t nat -nvL KUBE-SERVICES | grep dpt:80
     0     0 KUBE-SVC-G6D3V5KS3PXPUEDS  tcp  --  *      *       0.0.0.0/0            172.20.26.89         /* istio-system/istio-ingressgateway:http2 cluster IP */ tcp dpt:80

# 해당 체인은 Endpoint로 포워딩 합니다
[root@ip-10-11-12-72 bin]# iptables -t nat -nvL KUBE-SVC-G6D3V5KS3PXPUEDS
Chain KUBE-SVC-G6D3V5KS3PXPUEDS (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SEP-HL5XEYN4N2XIE3Z4  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* istio-system/istio-ingressgateway:http2 */

해당 Endpoint는 Ingress Gateway 입니다. 
[root@ip-10-11-12-72 bin]#  iptables -t nat -nvL KUBE-SEP-HL5XEYN4N2XIE3Z4
Chain KUBE-SEP-HL5XEYN4N2XIE3Z4 (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.11.88.41          0.0.0.0/0            /* istio-system/istio-ingressgateway:http2 */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* istio-system/istio-ingressgateway:http2 */ tcp to:10.11.88.41:8080
```

아래에서 Curl을 통해 실제 Istio Proxy/Envoy가 응답 하지는 지 확인해 봅니다.&#x20;

```
curl -I XGET http://${GATEWAY_URL}/productpage -v

```

아래 처럼 Pod내의 istio-envoy Container가 응답하는 것을 확인할 수 있습니다.&#x20;

```
$ curl -I XGET http://${GATEWAY_URL}/productpage -v
* Could not resolve host: XGET
* Closing connection 0
curl: (6) Could not resolve host: XGET
*   Trying 15.164.69.90:80...
* Connected to a36f3d5b98fc24cb29851355830f685e-55560905.ap-northeast-2.elb.amazonaws.com (15.164.69.90) port 80 (#1)
> HEAD /productpage HTTP/1.1
> Host: a36f3d5b98fc24cb29851355830f685e-55560905.ap-northeast-2.elb.amazonaws.com
> User-Agent: curl/7.79.1
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
< content-type: text/html; charset=utf-8
content-type: text/html; charset=utf-8
< content-length: 5290
content-length: 5290
< server: istio-envoy
server: istio-envoy
< date: Mon, 20 Jun 2022 15:27:24 GMT
date: Mon, 20 Jun 2022 15:27:24 GMT
< x-envoy-upstream-service-time: 32
x-envoy-upstream-service-time: 32

< 
* Connection #1 to host a36f3d5b98fc24cb29851355830f685e-55560905.ap-northeast-2.elb.amazonaws.com left intact
```

