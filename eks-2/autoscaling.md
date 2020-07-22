# AutoScaling 구성



![](../.gitbook/assets/image%20%2855%29.png)

## HPA 구성

Horizontal Pod Autoscaler는 CPU 사용량 \(또는 [사용자 정의 메트릭](https://git.k8s.io/community/contributors/design-proposals/instrumentation/custom-metrics-api.md), 아니면 다른 애플리케이션 지원 메트릭\)을 모니터하여 ReplicationController, Deployment, ReplicaSet 또는 StatefulSet의 파드 개수를 자동으로 스케일합니. Horizontal Pod Autoscaler는 크기를 조정할 수 없는 오브젝트\(예: 데몬셋\(DaemonSet\)\)에는 적용되지 않습니다.

Horizontal Pod Autoscaler는 쿠버네티스 API 리소스 및 컨트롤러로 구현됩니. 리소스는 컨트롤러의 동작을 결정합니다. 컨트롤러는 모니터링 평균 CPU 사용률이 사용자가 지정한 대상과 일치하도록 레ReplicationController, Deployment, ReplicaSet 개수를 주기적으로 조정합니.

### 1.Metric Server 설치

Namespace 생성, helm Chart를 통해 metric-server 설치를 진행합니다. bitnami를 통해서 metric-server를 설치하

```text
kubectl create namespace metrics
helm install my-metric-server bitnami/metrics-server --namespace metrics
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
  - lastTransitionTime: "2020-07-22T07:12:11Z"
    message: all checks passed
    reason: Passed
    status: "True"
    type: Available
```

### 2. APP 설치 및 HPA 리소스 생성.

이제 HPA 기반의 Scaling을 확인하기 위해 테스트용 PHP Web App 배포를 합니다.

```text
kubectl -n metrics run php-apache --image=us.gcr.io/k8s-artifacts-prod/hpa-example --requests=cpu=200m --expose --port=80생성된 컨테이너가 CPU 50% 초과하면 확장되도록 설정합니다.
```

```text
kubectl -n metrics autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

아래와 같은 출력 결과 예제를 확인할 수 있습니다.

```text
whchoi98:~/environment $ kubectl -n metrics get hpa
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         10        1          86s
```

이제 load generator를 생성해서 Trigger Event를 발생해 봅니다. busybox를 배포하고 Shell로 접속합니다.

Cloud9 IDE에서 Terminal을 한개 더 오픈합하고, 아래 명령을 실행합니다.

```text
kubectl -n metrics run -i --tty load-generator --image=busybox /bin/sh
```

앞서 생성한 PHP Web 컨테이너로 무한 접속하도록 합니다.

```text
while true; do wget -q -O - http://php-apache; done
```

이제 자동확장이 되는지 확인합니다.

```text
kubectl -n metrics get hpa -w
```

아래와 같은 출력 결과를 확인 할 수 있습니다.

```text
whchoi98:~/environment $ kubectl -n metrics get hpa -w
NAME         REFERENCE               TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   248%/50%   1         10        8          6m1s
php-apache   Deployment/php-apache   248%/50%   1         10        10         6m16s
php-apache   Deployment/php-apache   56%/50%    1         10        10         6m32s
php-apache   Deployment/php-apache   54%/50%    1         10        10         7m32s
php-apache   Deployment/php-apache   47%/50%    1         10        10         8m33s
```

Cloud9 IDE 에서 터미널을 한개 더 열고 K9s 를 실행하면 더 상세하게 볼 수 있습니다.

```text
k9s -n metrics
```

![](../.gitbook/assets/image%20%2854%29.png)

앞서 Shell을 실행했던 Cloud9 IDE 창에서 Shell을 종료합니다. 그리고 다시 Replica가 줄어드는지를 확인합니다.

```text
whchoi98:~/environment $ kubectl -n metrics get hpa -w
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   46%/50%   1         10        10         10m
php-apache   Deployment/php-apache   0%/50%    1         10        10         11m
php-apache   Deployment/php-apache   0%/50%    1         10        10         16m
php-apache   Deployment/php-apache   0%/50%    1         10        1          16m
```

![](../.gitbook/assets/image%20%2856%29.png)







