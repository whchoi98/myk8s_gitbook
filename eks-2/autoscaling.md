---
description: 'Update : 2025-04-08'
---

# 스케쥴링 - AutoScaling 구성



![](<../.gitbook/assets/image (438).png>)

## HPA 구성 - Pod 레벨 확장성&#x20;

HPA (Horizontal Pod Autoscaler) 는 Kubernetes에서 워크로드의 CPU 사용률, 사용자 정의 메트릭, 또는 애플리케이션 지표를 기준으로 파드 수를 자동으로 조정하는 메커니즘입니다.

HPA는 다음과 같은 오브젝트의 파드 개수를 자동으로 스케일합니다:

* Deployment
* ReplicaSet
* StatefulSet
* ReplicationController

HPA는 Kubernetes API 리소스와 이를 관리하는 컨트롤러로 구성됩니다:

• HPA 리소스는 어떤 대상 워크로드를 얼마나, 어떤 기준으로 스케일링할지를 정의합니다.

• 컨트롤러는 해당 리소스를 주기적으로 감시하면서 메트릭을 수집하고, 설정된 타겟과 비교하여 파드 개수를 조정합니다.\


예를 들어, 평균 CPU 사용률이 사용자가 정의한 목표값(예: 70%)보다 높으면 HPA는 파드를 확장하고, 낮으면 축소합니다.\


> 💡 단, DaemonSet과 같이 크기를 조정할 수 없는 오브젝트에는 HPA가 적용되지 않습니다.

메트릭 소스는 어디일까요?

HPA가 사용하는 지표는 다음 경로로 수집됩니다:

<table data-header-hidden><thead><tr><th width="172.32421875"></th><th></th><th></th></tr></thead><tbody><tr><td>메트릭 타입</td><td>데이터 소스</td><td>설명</td></tr><tr><td>CPU / 메모리 (기본)</td><td>metrics-server</td><td>클러스터 내 각 파드의 리소스 사용량을 수집합니다. 기본적으로 CPU는 averageUtilization 기준으로, 메모리는 averageValue 기준으로 평가됩니다.</td></tr><tr><td>External Metrics</td><td>외부 서비스 <br>(예: CloudWatch, Datadog)</td><td>외부 시스템의 메트릭을 가져올 수 있으며, 주로 Prometheus Adapter 등을 통해 연동합니다.</td></tr><tr><td>Custom Metrics</td><td>Prometheus + Custom Metrics API</td><td>애플리케이션 수준 지표(예: 큐 길이, 요청 수 등)를 사용하기 위해 Prometheus Adapter 등과 함께 사용됩니다.</td></tr></tbody></table>

> HPA가 제대로 작동하려면 클러스터에 metrics-server가 설치되어 있어야 합니다. Custom / External 메트릭을 사용하려면 별도의 어댑터(Prometheus Adapter 등)를 설치해야 합니다.

📌 요약

• HPA는 워크로드의 리소스 사용량을 기반으로 파드 수를 자동 조절합니다.

• 기본적으로 CPU/메모리는 metrics-server로부터 수집됩니다.

• Prometheus + Custom Metrics API를 활용하면 사용자 정의 지표 기반의 오토스케일링도 구현할 수 있습니다.

• 리소스와 컨트롤러 구조로 구성되며, 설정된 목표와 실제 지표를 비교해 동적으로 스케일합니다.

### 1.Metric Server 설치

아래와 같 metric-server 설치를 진행합니다.&#x20;

```
eksctl delete addon --name metrics-server --cluster ${EKSCLUSTER_NAME}
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

```

metric API 상태를 확인합니다.

```
kubectl get apiservice v1beta1.metrics.k8s.io -o yaml

```

metric API가 아래와 같은 결과가 출력되면 정상입니다.

```
kubectl get apiservice v1beta1.metrics.k8s.io -o yaml
#생략
status:
  conditions:
  - lastTransitionTime: "2021-04-06T10:06:38Z"
    message: all checks passed
    reason: Passed
    status: "True"
    type: Available
  
```

metric server가 정상적으로 설치가 완료되면 아래와 같이 리소스 모니터링을 확인 할 수 있습니다.

```
kubectl top pod --all-namespaces
```

### 2. APP 설치 및 HPA 리소스 생성.

이제 HPA 기반의 Scaling을 확인하기 위해 테스트용 PHP Web App 배포를 합니다.

```
kubectl create namespace metric-test
kubectl -n metric-test apply -f https://k8s.io/examples/application/php-apache.yaml

```

생성된 컨테이너가 CPU 50% 초과하면 확장되도록 설정합니다. 최소 Pod 1개, 최대 10개까지 replica를 선언합니다.

```
kubectl -n metric-test autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10

```

아래와 같은 출력 결과 예제를 확인할 수 있습니다.

```
kubectl -n metric-test get hpa

```

```
whchoi98:~/environment $ kubectl -n metric-test get hpa
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         10        1          86s
```

### 3. 부하생성과 HPA기반의 AutoScale

이제 load generator를 생성해서 Trigger Event를 발생해 봅니다. busybox를 배포하고 Shell로 접속합니다.

Cloud9 IDE에서 Terminal을 한개 더 오픈하고, 아래 명령을 실행합니다.

```
# 부하 생성을 유지하면서 나머지 스텝을 수행할 수 있도록,
# 다음의 명령을 별도의 터미널에서 실행한다.
# 앞서 생성한 PHP Web 컨테이너로 무한 접속하도록 합니다.
kubectl -n metric-test run -i  --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"

```

이제 자동확장이 되는지 확인합니다.

```
kubectl -n metric-test get hpa -w

```

아래와 같은 출력 결과를 확인 할 수 있습니다.

```
kubectl -n metric-test get hpa -w
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         10        1          23m
php-apache   Deployment/php-apache   483%/50%   1         10        1          23m
php-apache   Deployment/php-apache   483%/50%   1         10        4          23m
php-apache   Deployment/php-apache   483%/50%   1         10        8          23m
```

IDE 에서 터미널을 한개 더 열고 K9s 를 실행하면 더 상세하게 볼 수 있습니다.

```
k9s -n metric-test

```

![](<../.gitbook/assets/image (401).png>)

앞서 Shell을 실행했던 DE 창에서 Shell을 종료합니다. 그리고 다시 Replica가 줄어드는지를 확인합니다. 수분 이후에 Pod수는 줄어들게 됩니다.&#x20;

```
whchoi98:~/environment $ kubectl -n metrics get hpa -w
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   46%/50%   1         10        10         10m
php-apache   Deployment/php-apache   0%/50%    1         10        10         11m
php-apache   Deployment/php-apache   0%/50%    1         10        10         16m
php-apache   Deployment/php-apache   0%/50%    1         10        1          16m
```

![](<../.gitbook/assets/image (373).png>)

## CA 구성과 클러스터 확장 - Node 레벨 확장

4.CAS or CA (Cluster AutoScaler) 란?

Kubernetes에서는 워크로드(파드)의 수평 확장(HPA)뿐 아니라, 클러스터 자체의 리소스가 부족할 경우 노드의 수를 자동으로 확장할 수 있는 메커니즘도 제공합니다.바로 그것이 Cluster Autoscaler (CA) 입니다.

Cluster Autoscaler (CA) 는 클러스터에 필요한 리소스를 파악하여 EC2와 같은 노드를 자동으로 추가하거나 제거하는 컨트롤러입니다. CA는 다음과 같은 상황에서 동작합니다:

* 스케줄되지 못한 파드가 존재할 때 → 노드를 추가 (Scale-out)
* 특정 노드가 오랫동안 사용되지 않을 때 → 노드를 제거 (Scale-in)

\
CA는 다음 흐름으로 동작합니다:

1\. 파드 스케줄링 실패 감지: 스케줄러가 특정 파드를 노드에 할당하지 못할 경우, CA는 이를 감지합니다.

2\. 적절한 노드 그룹 선택: 해당 파드를 수용할 수 있는 노드 그룹을 판단합니다.

3\. 노드 추가 요청: 클라우드 프로바이더(EKS의 경우 EC2 Auto Scaling Group 또는 Karpenter)를 통해 노드를 자동으로 생성합니다.

4\. 리소스 낭비 노드 감지: 특정 노드가 장기간 idle 상태이면 파드를 다른 노드로 옮기고 해당 노드를 종료합니다.



AWS를 위한 Cluster Autoscaler (CA) 는 Auto Scaling Group과 연계하여 제공합니다.&#x20;

CA(Cluster AutoScaling)은 Pod가 Pending 상태인지를 모니터링하면서, Pending 상태를 확인하게 되면 CA가 ASG(Auto Scaling Group)의 Desired Capacity의 수치를 변경해서 Worker Node를 변경하는 방법을 사용합니다.

먼저 현재 구성된 완전관리형노드의 최소(minimum), 최대(maximum), 원하는 용량(desired) 를 확인합니다.

```
aws autoscaling \
    describe-auto-scaling-groups \
    --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='eksworkshop']].[AutoScalingGroupName, MinSize, MaxSize,DesiredCapacity]" \
    --output table
    
```

Managed Node의 frontend workernode들의 ASG 용량을 아래와 같이 변경합니다.&#x20;

```
export ASG_NAME=$(aws autoscaling describe-auto-scaling-groups --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='eksworkshop']].AutoScalingGroupName" --output text | awk '{ print $2 }')
aws autoscaling \
    update-auto-scaling-group \
    --auto-scaling-group-name ${ASG_NAME} \
    --min-size 3 \
    --desired-capacity 3 \
    --max-size 6
    
```

정상적으로 변경되었는지 확인해 봅니다.&#x20;

```
aws autoscaling \
    describe-auto-scaling-groups \
    --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='eksworkshop']].[AutoScalingGroupName, MinSize, MaxSize,DesiredCapacity]" \
    --output table
    
```



### 4. Service Account를 위한 IAM Role 구성 (IRSA)

Amazon EKS 클러스터의 서비스 계정(Service Account)와 IAM 역할을 IRSA를 통해서 연결할 수 있습니다. 그러면 이 서비스 계정(Service Account)은 해당 서비스 계정을 사용하는 모든 포드의 컨테이너에 AWS 권한을 제공할 수 있습니다. 이 기능을 사용하면 해당 노드의 CA 포드가 AWS API를 호출할 수 있도록 할 수 있습니다.&#x20;

클러스터의 서비스 계정에 대한 IAM 역할 활성화를 아래 ekstcl  명령을 통해 선언합니다.&#x20;

```
eksctl utils associate-iam-oidc-provider \
    --cluster ${EKSCLUSTER_NAME} \
    --approve

```

CA 포드가 AWS EC2 ASG (Auto Scaling Group)과 상호 작용할 수 있도록  허용하는 서비스 계정(Service Account)에 대한 IAM 정책 생성을 합니다.

```
mkdir ~/environment/cluster-autoscaler

cat <<EoF > ~/environment/cluster-autoscaler/k8s-asg-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeTags",
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup",
                "ec2:DescribeLaunchTemplateVersions"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
EoF

aws iam create-policy   \
  --policy-name k8s-asg-policy \
  --policy-document file://~/environment/cluster-autoscaler/k8s-asg-policy.json

```

마지막으로 **`kube-system`** 네임스페이스에서 **`cluster-autoscaler`** 서비스 계정에 대한 IAM 역할을 생성합니다.

```
eksctl create iamserviceaccount \
    --name cluster-autoscaler \
    --namespace kube-system \
    --cluster ${EKSCLUSTER_NAME}  \
    --attach-policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/k8s-asg-policy" \
    --approve \
    --override-existing-serviceaccounts

```

IAM 역할의 ARN이 있는 서비스 계정에 annotation이 추가되었는지 확인합니다.&#x20;

```
kubectl -n kube-system describe sa cluster-autoscaler

```

아래와 같은 결과가 출력됩니다.&#x20;

```
$ kubectl -n kube-system describe sa cluster-autoscalerName:                cluster-autoscaler
Namespace:           kube-system
Labels:              app.kubernetes.io/managed-by=eksctl
Annotations:         eks.amazonaws.com/role-arn: arn:aws:iam::195829107343:role/eksctl-eksworkshop-addon-iamserviceaccount-k-Role1-12LCPJRCWUCG1
Image pull secrets:  <none>
Mountable secrets:   cluster-autoscaler-token-9l7wf
Tokens:              cluster-autoscaler-token-9l7wf
Events:              <none>
```

### 5.CA (Cluster Autoscaler) Pod 배포 &#x20;

Cluster Autoscaler version을 확인하고, Shell 변수에 입력해 둡니다.&#x20;

K8s 버전과 Autoscaler를 위한 CA version을 동일하게 사용합니다.

([https://github.com/kubernetes/autoscaler/releases](https://github.com/kubernetes/autoscaler/releases))&#x20;

```
export K8S_VERSION=$(kubectl version --short | grep 'Server Version:' | sed 's/[^0-9.]*\([0-9.]*\).*/\1/' | cut -d. -f1,2)
#export AUTOSCALER_VERSION="1.26.2"

echo $K8S_VERSION
echo $AUTOSCALER_VERSION
```

```
# ~/.bash_profile 불러오기 (EKS_VERSION 포함)
source ~/.bash_profile

# EKS_VERSION 값 확인
echo "📌 EKS_VERSION is $EKS_VERSION"

# Autoscaler 버전 매핑
case $EKS_VERSION in
  "1.29") AUTOSCALER_VERSION="1.29.1" ;;
  "1.30") AUTOSCALER_VERSION="1.30.4" ;;  
  *) echo "❌ 지원하지 않는 EKS 버전입니다: $EKS_VERSION"; exit 1 ;;
esac

# 환경변수로 export
export AUTOSCALER_VERSION
echo "✅ AUTOSCALER_VERSION set to $AUTOSCALER_VERSION"
```

CA (Cluster AutoScaler) PoD 생성을 위한 mainfest 파일을 생성합니다.&#x20;

```
cat << EOF > ~/environment/cluster-autoscaler/cluster_autoscaler.yml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
  name: cluster-autoscaler
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-autoscaler
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
rules:
  - apiGroups: [""]
    resources: ["events", "endpoints"]
    verbs: ["create", "patch"]
  - apiGroups: [""]
    resources: ["pods/eviction"]
    verbs: ["create"]
  - apiGroups: [""]
    resources: ["pods/status"]
    verbs: ["update"]
  - apiGroups: [""]
    resources: ["endpoints"]
    resourceNames: ["cluster-autoscaler"]
    verbs: ["get", "update"]
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["watch", "list", "get", "update"]
  - apiGroups: [""]
    resources:
      - "namespaces"
      - "pods"
      - "services"
      - "replicationcontrollers"
      - "persistentvolumeclaims"
      - "persistentvolumes"
    verbs: ["watch", "list", "get"]
  - apiGroups: ["extensions"]
    resources: ["replicasets", "daemonsets"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["policy"]
    resources: ["poddisruptionbudgets"]
    verbs: ["watch", "list"]
  - apiGroups: ["apps"]
    resources: ["statefulsets", "replicasets", "daemonsets"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses", "csinodes", "csidrivers", "csistoragecapacities"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["batch", "extensions"]
    resources: ["jobs"]
    verbs: ["get", "list", "watch", "patch"]
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["create"]
  - apiGroups: ["coordination.k8s.io"]
    resourceNames: ["cluster-autoscaler"]
    resources: ["leases"]
    verbs: ["get", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["create","list","watch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["cluster-autoscaler-status", "cluster-autoscaler-priority-expander"]
    verbs: ["delete", "get", "update", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-autoscaler
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-autoscaler
subjects:
  - kind: ServiceAccount
    name: cluster-autoscaler
    namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: cluster-autoscaler
subjects:
  - kind: ServiceAccount
    name: cluster-autoscaler
    namespace: kube-system

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
      annotations:
        prometheus.io/scrape: 'true'
        prometheus.io/port: '8085'
    spec:
      priorityClassName: system-cluster-critical
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        fsGroup: 65534
      serviceAccountName: cluster-autoscaler
      containers:
        - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v${AUTOSCALER_VERSION}
          name: cluster-autoscaler
          resources:
            limits:
              cpu: 100m
              memory: 600Mi
            requests:
              cpu: 100m
              memory: 600Mi
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/${EKSCLUSTER_NAME}
          volumeMounts:
            - name: ssl-certs
              mountPath: /etc/ssl/certs/ca-certificates.crt
              readOnly: true
          imagePullPolicy: "Always"
      volumes:
        - name: ssl-certs
          hostPath:
            path: "/etc/ssl/certs/ca-bundle.crt"
EOF

```

CA Pod를 위한 Manifest 파일은 아래에서 확인 할 수 있습니다. (참조)

```
curl -O https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

```

CA Pod를 배포합니다.&#x20;

```
cd ~/environment/cluster-autoscaler/
kubectl apply -f ./cluster_autoscaler.yml
kubectl -n kube-system get pods -o wide | grep auto
# cluster-autoscaler pod가 정상적으로 배포되었는지 확인

```

CA 포드가 실행 중인 노드를 제거하는 것을 방지하기 위해 다음 명령을 사용하여 **`cluster-autoscaler.kubernetes.io/safe-to-evict`** 주석을 추가합니다.

```
kubectl -n kube-system \
    annotate deployment.apps/cluster-autoscaler \
    cluster-autoscaler.kubernetes.io/safe-to-evict="false"

```

### 6. CA로  Node 확장하기.

nginx 샘플 App을 배포하기 위해 매니페스트 파일을 작성하고, 배포합니다.

```
cat <<EoF> ~/environment/cluster-autoscaler/nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-to-scaleout
  namespace: autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        service: nginx
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx-to-scaleout
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 500m
            memory: 512Mi
      nodeSelector:
        nodegroup-type: "managed-frontend-workloads"
EoF
kubectl create namespace autoscaler
kubectl -n autoscaler apply -f ~/environment/cluster-autoscaler/nginx.yaml

```

정상적으로 배포되었는지 확인해 봅니다.

```
kubectl -n autoscaler  get deployment/nginx-to-scaleout

```

아래와 같은 출력 결과를 확인 해 볼 수 있습니다.

```
kubectl -n autoscaler  get deployment/nginx-to-scaleout
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
nginx-to-scaleout   1/1     1            1           11s
```

아래 명령을 통해 Replicaset을 40개로 변경합니다.

```
kubectl -n autoscaler scale deployment nginx-to-scaleout --replicas=40

```

포드가 증가하는 것을 아래 명령을 통해 확인합니다. 또는 k9s 명령이 실행되고 있는 터미널에서 확인합니다.

```
kubectl -n autoscaler get pods -o wide --watch

```

Pod가 Replicaset이 발생하면서, Pod 상태가 Pending 이 발생하면 EC2 worker node가 추가되고 있는 것입니다.

```
nginx-to-scaleout-5c74d46fd6-qmj9k   0/1     Pending             0          0s      <none>        <none>                                            <none>           <none>
nginx-to-scaleout-5c74d46fd6-9wpt9   0/1     Pending             0          0s      <none>        <none>                                            <none>           <none>
nginx-to-scaleout-5c74d46fd6-m5lck   0/1     Pending             0          0s      <none>        <none>                                            <none>           <none>
nginx-to-scaleout-5c74d46fd6-9smbd   0/1     Pending             0          0s      <none>        <none>                                            <none>           <none>
nginx-to-scaleout-5c74d46fd6-2ltqn   0/1     Pending             0          0s      <none>        <none>                                            <none>           <none>
nginx-to-scaleout-5c74d46fd6-jvnpl   0/1     Pending             0          0s      <none>        <none>                                            <none>           <none>
nginx-to-scaleout-5c74d46fd6-tnvtp   0/1     Pending             0          0s      <none>        <none>                                            <none>           <none>
nginx-to-scaleout-5c74d46fd6-jm5nj   0/1     Pending             0          0s      <none>        <none>                                            <none>           <none>
nginx-to-scaleout-5c74d46fd6-tsrvq   0/1     Pending             0          0s      <none>        <none>                                            <none>           <none>
nginx-to-scaleout-5c74d46fd6-lprfm   0/1     Pending             0          0s      <none>        <none>                                            <none>           <none>
```

CAS Pod 로그를 통해서 ASG에 API Call을 하는 것을 확인할 수 있습니다.

```
kubectl -n kube-system logs -l app=cluster-autoscaler --tail=100 -f
```

• scale-up → 실제 ASG 증설 시도 로그

• unschedulable → Pending 상태인 Pod 인식

• no node group / backoff → 스케일링 실패 사유

• upcoming → 스케일링 예정 상태



아래와 같이 EC2 대시보드에서 Auto Scaling Group에서 Desired 용량이 변경되고  EC 인스턴스들이 추가 됩니다.

**`EC2 - Auto Scaling Group`**&#x20;

![](<../.gitbook/assets/image (141).png>)

**`EC2 - 인스턴스`**&#x20;

<figure><img src="../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

아래와 같이 replica를 1로 원복 시킵니다.

```
kubectl -n autoscaler scale deployment nginx-to-scaleout --replicas=1

```
