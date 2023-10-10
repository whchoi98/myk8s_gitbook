# X-Ray기반 추적

## X-Ray 소개

AWS X-Ray는 개발자가 마이크로 서비스 아키텍처를 사용해 구축된 애플리케이션과 같은 프로덕션 분산 애플리케이션을 분석하고 디버그하는 데 도움이 됩니다. X-Ray를 사용해 자신이 개발한 애플리케이션과 기본 서비스가 성능 문제와 오류의 근본 원인 식별과 문제 해결을 올바로 수행하는지 파악할 수 있습니다. X-Ray는 요청이 애플리케이션을 통과함에 따라 요청에 대한 엔드 투 엔드 뷰를 제공하고 애플리케이션의 기본 구성 요소를 맵으로 보여줍니다. X-Ray를 사용하여 간단한 3-티어 애플리케이션에서부터 수천 개의 서비스로 구성된 복잡한 마이크로 서비스 애플리케이션에 이르기까지 개발 중인 애플리케이션과 프로덕션에 적용된 애플리케이션 모두 분석할 수 있습니다.

## IAM 역할(Role) 구성

X-Ray 데몬셋이 서비스 되기 위해서는, Kubernetes 서비스 어카운트와 IAM 역할과 정책이 있어야 합니다.

### 1.서비스 어카운트 생성.

X-Ray를 위해 서비스 어카운트를 생성합니다.

```
eksctl create iamserviceaccount --name xray-daemon --namespace default --cluster ${ekscluster_name} --attach-policy-arn arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess --approve --override-existing-serviceaccounts

```

### 2. 서비스 어카운트에 Label생성.

서비스 어카운트에 Label 생성을 합니다.

```
kubectl label serviceaccount xray-daemon app=xray-daemon

```

## X-Ray 데몬셋 배포

EKS Cluster에 X-Ray 데몬셋을 배포합니다. X-Ray 데몬셋은 EKS Cluster 의 각 Worker Node에 배포합니다.&#x20;

AWS X-Ray SDK는 마이크로서비스 어플리케이션을 분석하는 데 사용되며, 데몬셋은 xray-service.default:2000을 가리키도록 구성될 것입니다. 다음은 Go를 위한 X-Ray SDK에 대한 구성방법의 예입니다. 이 랩에서는 필수 구성조건이 아닙니다.

```
func init() {
	xray.Configure(xray.Config{
		DaemonAddr:     "xray-service.default:2000",
		LogLevel:       "info",
	})
}
```

X-Ray 데몬세트를 배포합니다.&#x20;

> 설치시 에러가 발생한다면, [https://github.com/whchoi98/myeks.git](https://github.com/whchoi98/myeks.git) 에서 xray-k8s-daemonset.yaml 파일을 내려 받아서 배포합니다.

```
kubectl create -f https://eksworkshop.com/intermediate/245_x-ray/daemonset.files/xray-k8s-daemonset.yaml

```

X-Ray 데몬셋의 상태를 확인합니다.

```
kubectl describe daemonset xray-daemon
```

아래와 같은 결과를 확인 할 수 있습니다.

```
whchoi98:~/environment $ kubectl describe daemonset xray-daemon
Name:           xray-daemon
Selector:       app=xray-daemon
Node-Selector:  <none>
Labels:         <none>
Annotations:    deprecated.daemonset.template.generation: 1
Desired Number of Nodes Scheduled: 6
Current Number of Nodes Scheduled: 6
Number of Nodes Scheduled with Up-to-date Pods: 6
Number of Nodes Scheduled with Available Pods: 0
Number of Nodes Misscheduled: 0
Pods Status:  0 Running / 6 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:           app=xray-daemon
  Service Account:  xray-daemon
  Containers:
   xray-daemon:
    Image:      trevorrobertsjr/eks-workshop-x-ray-daemon:02d13ce10add55081c68b6b76a19b7dfeea00dad
    Port:       2000/UDP
    Host Port:  2000/UDP
    Command:
      /usr/bin/xray
      -c
      /aws/xray/config.yaml
    Limits:
      memory:     24Mi
    Environment:  <none>
    Mounts:
      /aws/xray from config-volume (ro)
  Volumes:
   config-volume:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      xray-config
    Optional:  false
Events:
  Type    Reason            Age   From                  Message
  ----    ------            ----  ----                  -------
  Normal  SuccessfulCreate  5s    daemonset-controller  Created pod: xray-daemon-k7lll
  Normal  SuccessfulCreate  5s    daemonset-controller  Created pod: xray-daemon-k2xpt
  Normal  SuccessfulCreate  5s    daemonset-controller  Created pod: xray-daemon-5l4jp
  Normal  SuccessfulCreate  5s    daemonset-controller  Created pod: xray-daemon-nmh64
  Normal  SuccessfulCreate  5s    daemonset-controller  Created pod: xray-daemon-kb2x9
  Normal  SuccessfulCreate  5s    daemonset-controller  Created pod: xray-daemon-hp8c6
```

X-Ray에 대한 데몬 Pod의 로그는 아래와 같은 명령으로 확인 할 수 있습니다.

```
kubectl logs -l app=xray-daemon
```

## 어플리케이션 배포

### 3.어플리케이션 배포.&#x20;

Front-end & Back-end용 어플리케이션을 배포합니다. X-Ray 는 Go,Python,Node.js, Ruby. .NET, Java용 SDK를 지원합니다.

아래와 같이 어플리케이션을 배포합니다.

```
kubectl apply -f https://eksworkshop.com/intermediate/245_x-ray/sample-front.files/x-ray-sample-front-k8s.yml

kubectl apply -f https://eksworkshop.com/intermediate/245_x-ray/sample-back.files/x-ray-sample-back-k8s.yml

```

정상적으로 배포되었는지를 확인해 봅니다.

```
kubectl describe deployments x-ray-sample-front-k8s x-ray-sample-back-k8s

```

아래와 같은 결과가 출력된 것을 확인 할 수 있습니다.

```
whchoi98:~/environment $ kubectl describe deployments x-ray-sample-front-k8s x-ray-sample-back-k8s
Name:                   x-ray-sample-front-k8s
Namespace:              default
CreationTimestamp:      Sat, 25 Jul 2020 13:04:58 +0000
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
                        kubectl.kubernetes.io/last-applied-configuration:
                          {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"name":"x-ray-sample-front-k8s","namespace":"default"},"sp c":{"r...
Selector:               app=x-ray-sample-front-k8s,tier=frontend
Replicas:               3 desired | 3 updated | 3 total | 2 available | 1 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  2 max unavailable, 2 max surge
Pod Template:
  Labels:  app=x-ray-sample-front-k8s
           tier=frontend
  Containers:
   x-ray-sample-front-k8s:
    Image:        rnzdocker1/eks-workshop-x-ray-sample-front:ba01b042766edb5fc794733b0af28d92d99b63dd
    Port:         8080/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    ReplicaSetUpdated
OldReplicaSets:  <none>
NewReplicaSet:   x-ray-sample-front-k8s-f845ffd7b (3/3 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  19s   deployment-controller  Scaled up replica set x-ray-sample-front-k8s-f845ffd7b to 3


Name:                   x-ray-sample-back-k8s
Namespace:              default
CreationTimestamp:      Sat, 25 Jul 2020 13:05:00 +0000
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
                        kubectl.kubernetes.io/last-applied-configuration:
                          {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"name":"x-ray-sample-back-k8s","namespace":"default"},"spec":{"re...
Selector:               app=x-ray-sample-back-k8s,tier=backend
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  2 max unavailable, 2 max surge
Pod Template:
  Labels:  app=x-ray-sample-back-k8s
           tier=backend
  Containers:
   x-ray-sample-back-k8s:
    Image:        rnzdocker1/eks-workshop-x-ray-sample-back:ba01b042766edb5fc794733b0af28d92d99b63dd
    Port:         8080/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   x-ray-sample-back-k8s-65b7f4c46d (3/3 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  17s   deployment-controller  Scaled up replica set x-ray-sample-back-k8s-65b7f4c46d to 3
```

### 4.Front-End 접속.&#x20;

접속을 위해서 , ELB DNS 레코드를 확인합니다.

```
kubectl get service x-ray-sample-front-k8s -o wide
export XRAY_APP_SERVICE_URL=$(kubectl get svc x-ray-sample-front-k8s --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
echo "X-RAY MSA Front-end APP URL = $XRAY_APP_SERVICE_URL"

```

다음과 같은 결과를 얻을 수 있습니다.

```
X-RAY MSA Front-end APP URL = aefb69fa01e574f1a949c6a18c423ee2-1078192765.ap-northeast-2.elb.amazonaws.com

```

브라우저에 정상적으로 접속되는 지 확인합니다.

{% hint style="warning" %}
ELB 배포 시간으로 3분 정도 시간이 소요 될 수 있습니다.
{% endhint %}

front-end App은 1초에 한번씩 백엔드로 요청하게 됩니다.

![](<../.gitbook/assets/image (179).png>)

## X-Ray 콘솔 확인

X-RAY 서비스 대시보드에서 "서비스 맵"을 선택합니다.

아래와 같이 Client-FrontEnd App- BackEnd App의 연결구성과 어플리케이션 간 응답시간, 분당 트랜잭션 수를 확인 할 수 있습니다.

![](<../.gitbook/assets/image (229).png>)



![](<../.gitbook/assets/image (28).png>)

Analyze trace를 선택하게 되면 , 상세 분석을 할 수 있습니다. 테이블 구성 옵션을 필터하면 HTTP Code 및 Client IP 등을 상세하게 분석할 수 있습니다.

![](<../.gitbook/assets/image (433).png>)

![](<../.gitbook/assets/image (301).png>)

트레이스를 선택하면, 각 트랜잭션 당 더욱 상세한 내용을 살펴 볼 수 있습니다. 트레이스 목록에서 1개를 선택해서 클릭합니다.

![](<../.gitbook/assets/image (372).png>)

Time line 및 소스 데이터를 상세하게 분석 할 수 있습니다.

![](<../.gitbook/assets/image (408).png>)

