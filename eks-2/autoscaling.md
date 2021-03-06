---
description: 'Update : 2020-11-11'
---

# 스케쥴링 - AutoScaling 구성



![](../.gitbook/assets/image%20%2857%29.png)

## HPA 구성 - Pod 레벨 확장성 

HPA\(Horizontal Pod Autoscaler\)는 CPU 사용량 \(또는 [사용자 정의 메트릭](https://git.k8s.io/community/contributors/design-proposals/instrumentation/custom-metrics-api.md), 아니면 다른 애플리케이션 지원 메트릭\)을 모니터하여 ReplicationController, Deployment, ReplicaSet 또는 StatefulSet의 파드 개수를 자동으로 스케일합니다.Horizontal Pod Autoscaler는 크기를 조정할 수 없는 오브젝트\(예: 데몬셋\(DaemonSet\)\)에는 적용되지 않습니다.

HPA\(Horizontal Pod Autoscaler\)는 쿠버네티스 API 리소스 및 컨트롤러로 구현됩니. 리소스는 컨트롤러의 동작을 결정합니다. 컨트롤러는 모니터링 평균 CPU 사용률이 사용자가 지정한 대상과 일치하도록 ReplicationController, Deployment, ReplicaSet 개수를 주기적으로 조정합니다 .

### 1.Metric Server 설치

아래와 같 metric-server 설치를 진행합니다. 

```text
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

```

metric API 상태를 확인합니다.

```text
kubectl get apiservice v1beta1.metrics.k8s.io -o yaml

```

metric API가 아래와 같은 결과가 출력되면 정상입니다.

```text
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

```text
kubectl top pod -n nlb-test

```

### 2. APP 설치 및 HPA 리소스 생성.

이제 HPA 기반의 Scaling을 확인하기 위해 테스트용 PHP Web App 배포를 합니다.

```text
kubectl create namespace metric-test
kubectl -n metric-test create deployment php-apache --image=us.gcr.io/k8s-artifacts-prod/hpa-example
kubectl -n metric-test set resources deploy php-apache --requests=cpu=200m
kubectl -n metric-test expose deploy php-apache --port 80
kubectl -n metric-test get pods -l app=php-apache

```

생성된 컨테이너가 CPU 50% 초과하면 확장되도록 설정합니다. 최소 Pod 1개, 최대 10개까지 replica를 선언합니다.

```text
kubectl -n metric-test autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10

```

아래와 같은 출력 결과 예제를 확인할 수 있습니다.

```text
kubectl -n metric-test get hpa
```

```text
whchoi98:~/environment $ kubectl -n metric-test get hpa
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         10        1          86s
```

### 3. 부하생성과 HPA기반의 AutoScale

이제 load generator를 생성해서 Trigger Event를 발생해 봅니다. busybox를 배포하고 Shell로 접속합니다.

Cloud9 IDE에서 Terminal을 한개 더 오픈합하고, 아래 명령을 실행합니다.

```text
kubectl -n metric-test run -i --tty load-generator --image=busybox /bin/sh

```

앞서 생성한 PHP Web 컨테이너로 무한 접속하도록 합니다.

```text
while true; do wget -q -O - http://php-apache; done

```

이제 자동확장이 되는지 확인합니다.

```text
kubectl -n metric-test get hpa -w

```

아래와 같은 출력 결과를 확인 할 수 있습니다.

```text
whchoi98:~/environment $ kubectl -n metric-test get hpa -w
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         10        1          23m
php-apache   Deployment/php-apache   483%/50%   1         10        1          23m
php-apache   Deployment/php-apache   483%/50%   1         10        4          23m
php-apache   Deployment/php-apache   483%/50%   1         10        8          23m
```

Cloud9 IDE 에서 터미널을 한개 더 열고 K9s 를 실행하면 더 상세하게 볼 수 있습니다.

```text
k9s -n metrics

```

![](../.gitbook/assets/image%20%2855%29.png)

앞서 Shell을 실행했던 Cloud9 IDE 창에서 Shell을 종료합니다. 그리고 다시 Replica가 줄어드는지를 확인합니다.

```text
whchoi98:~/environment $ kubectl -n metrics get hpa -w
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   46%/50%   1         10        10         10m
php-apache   Deployment/php-apache   0%/50%    1         10        10         11m
php-apache   Deployment/php-apache   0%/50%    1         10        10         16m
php-apache   Deployment/php-apache   0%/50%    1         10        1          16m
```

![](../.gitbook/assets/image%20%2862%29.png)

## CA 구성과 클러스터 확장 - Node 레벨 확장

AWS를 위한 Cluster Autoscaler는 Auto Scaling Group과 연계하여 제공합니다. 사용자는 아래와 같은 4가지 배포 옵션을 선택할 수 있습니다.

* 1개의 Auto Scaling Group
* 다중 Auto Scaling Group
* 자동 검색
* 컨트롤 플레인 노드 설정

### 1.CA \(Cluster Autoscaler\) 다운로드 

CA 배포를 위한 매니페스트 파일을 다운으로 하고, 새롭게 생성한 디렉토리에 복사합니다.

```text
mkdir ~/environment/cluster-autoscaler
cd ~/environment/cluster-autoscaler
wget https://eksworkshop.com/beginner/080_scaling/deploy_ca.files/cluster_autoscaler.yml

```

앞서 다운로드 받은 매니페스트 파일의 내용 중에서 CA \(Cluster Autoscaler\) version 값을 , 현재 EKS version에 맞추어서 변경합니다.

```text
export K8S_VERSION=$(kubectl version --short | grep 'Server Version:' | sed 's/[^0-9.]*\([0-9.]*\).*/\1/' | cut -d. -f1,2)
export AUTOSCALER_VERSION=$(curl -s "https://api.github.com/repos/kubernetes/autoscaler/releases" | grep '"tag_name":' | sed -s 's/.*-\([0-9][0-9\.]*\).*/\1/' | grep -m1 ${K8S_VERSION})

```

저장된 값을 확인해 봅니다.

```text
echo $K8S_VERSION
echo $AUTOSCALER_VERSION

```

```text
cd ~/environment/cluster-autoscaler
echo "$(envsubst < cluster_autoscaler.yml)" > cluster_autoscaler.yml

```

### 2.ASG \(Auto Scaling Group\) 구성

CA\(Cluster Autoscaler\)가 제어할 ASG\(AutoScaling Group\)의 이름을 구성합니다. Worker Node를 찾아서 저장해 둡니다. \([Public Subnet Worker Node Group 링크](https://console.aws.amazon.com/ec2/autoscaling/home?#AutoScalingGroups:id=eksctl-eksworkshop-eksctl-nodegroup-0-NodeGroup-SQG8QDVSR73G;view=details;filter=eksworkshop)\) 

**EC2 대시보드 - Auto Scaling**

![](../.gitbook/assets/image%20%28187%29.png)

```text
eksctl-eksworkshop-nodegroup-ng-public-01-NodeGroup-ZCRPGHX2NH7Q

```

ASG Group의 최소, 최대 사이즈를 확인합니다. \(min = 3, max =**6**\)

### 3.CA\(Cluster AutoScaler\) 구성

Cloud9 IDE에서 다운로드 받은 매니페스트 파일\(cluster\_autoscaler.yml\)파일에서 앞서 복사 해 둔 Auto Scaling Group 이름을 --node flag 부분에 변경하고 저장합니다.

```text
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --nodes=3:9:eksctl-eksworkshop-nodegroup-ng1-public-NodeGroup-1OKGC9A5SPGB1
```

인라인 정책을 구성하고 Public Worker Node의 EC2 인스턴스 프로파일에 추가합니다. 아래 그림에서 처럼 설정되어 있어야 합니다.

![](../.gitbook/assets/image%20%28192%29.png)

![](../.gitbook/assets/image%20%2854%29.png)

![](../.gitbook/assets/image%20%2858%29.png)

![](../.gitbook/assets/image%20%2861%29.png)

인라인 정책이 Public Worker Node의 EC2 인스턴스 프로파일에 없다면 아래와 같이 추가합니다. \(이미 eksctl을 배포할 때 추가되었기 때문에 아래는 생략해도 됩니다.\)

먼저 StackName을 확인합니다.

```text
eksctl get nodegroup --cluster eksworkshop -o json | jq -r '.[].StackName'
```

아래와 같은 값을 복사합니다. \(--stack-name\)

```text
eksctl-eksworkshop-nodegroup-ng-public-01
```

StackName의 값을 아래 aws cli에 넣고, IAM Role Name 값을 확인합니다.

```text
aws cloudformation describe-stack-resources --stack-name eksctl-eksworkshop-nodegroup-ng-public-01 | jq -r '.StackResources[] | select(.ResourceType=="AWS::IAM::Role") | .PhysicalResourceId'
```

아래와 같은 값을 복사해 둡니다. \(--role-name\)

```text
eksctl-eksworkshop-nodegroup-ng-p-NodeInstanceRole-S7BXB5A9FVBG

```

정책 아래와 같이 json파일을 만들고 추가합니다.

```text
mkdir ~/environment/asg_policy
cat <<EoF > ~/environment/asg_policy/k8s-asg-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "autoscaling:DescribeTags"
      ],
      "Resource": "*"
    }
  ]
}
EoF
aws iam put-role-policy --role-name eksctl-eksworkshop-nodegroup-ng-p-NodeInstanceRole-S7BXB5A9FVBG --policy-name ASG-Policy-For-Worker --policy-document file://~/environment/asg_policy/k8s-asg-policy.json
```

정상적으로 추가되었다면, 아래와 같이 확인 됩니다.

![](../.gitbook/assets/image%20%2864%29.png)

aws cli를 통해서도 확인 할 수 있습니다.

```text
aws iam get-role-policy --role-name eksctl-eksworkshop-nodegroup-ng-p-NodeInstanceRole-S7BXB5A9FVBG --policy-name ASG-Policy-For-Worker

```

aws cli를 통한 출력 결과 예제입니다.

```text
whchoi98:~/environment/cluster-autoscaler $ aws iam get-role-policy --role-name eksctl-eksworkshop-nodegroup-ng-p-NodeInstanceRole-S7BXB5A9FVBG --policy-name ASG-Policy-For-Worker
{
    "RoleName": "eksctl-eksworkshop-nodegroup-ng1-NodeInstanceRole-1970I5BJYVPFS",
    "PolicyName": "ASG-Policy-For-Worker",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "autoscaling:DescribeAutoScalingGroups",
                    "autoscaling:DescribeAutoScalingInstances",
                    "autoscaling:SetDesiredCapacity",
                    "autoscaling:TerminateInstanceInAutoScalingGroup",
                    "autoscaling:DescribeTags"
                ],
                "Resource": "*"
            }
        ]
    }
}
```

이제 Cluster Auto Scaler를 배포합니다.

```text
kubectl apply -f ~/environment/cluster-autoscaler/cluster_autoscaler.yml
```

### 4.CA로 Cluster 확장하기.

nginx 샘플 App을 배포하기 위해 매니페스트 파일을 작성하고, 배포합니다.

```text
cat <<EoF> ~/environment/cluster-autoscaler/nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-to-scaleout
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
        nodegroup-type: "frontend-workloads"
EoF
kubectl apply -f ~/environment/cluster-autoscaler/nginx.yaml

```

정상적으로 배포되었는지 확인해 봅니다.

```text
kubectl get deployment/nginx-to-scaleout nodegroup-type=frontend-workloads
```

아래와 같은 출력 결과를 확인 해 볼 수 있습니다.

```text
whchoi98:~/environment/cluster-autoscaler $ kubectl get deployment/nginx-to-scaleout
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
nginx-to-scaleout   1/1     1            1           11s
```

아래 명령을 통해 Replicaset을 50개로 변경합니다.

```text
kubectl scale --replicas=50 deployment/nginx-to-scaleout

```

포드가 증가하는 것을 아래 명령을 통해 확인합니다. 또는 k9s 명령이 실행되고 있는 터미널에서 확인합니다.

```text
kubectl get pods -o wide --watch

```

Pod가 Replicaset이 발생하면서, Pod 상태가 Pending 이 발생하면 EC2 worker node가 추가되고 있는 것입니다.

```text
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

아래와 같이 EC2 대시보드에서 인스턴스들이 추가 됩니다.

![](../.gitbook/assets/image%20%2860%29.png)

이제 아래와 같이 생성되었던 자원을 삭제합니다.

```text
aws iam delete-role-policy --role-name eksctl-eksworkshop-nodegroup-ng1-NodeInstanceRole-1970I5BJYVPFS --policy-name ASG-Policy-For-Worker
kubectl delete -f ~/environment/cluster-autoscaler/cluster_autoscaler.yml
kubectl delete -f ~/environment/cluster-autoscaler/nginx.yaml
kubectl delete hpa,svc php-apache
kubectl delete deployment php-apache load-generator
rm -rf ~/environment/cluster-autoscaler
helm -n metrics uninstall my-metric-server
kubectl delete ns metrics
```



