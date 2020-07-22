# AutoScaling 구성



![](../.gitbook/assets/image%20%2854%29.png)

## HPA 구성

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

### 2. HPA기반의 APP 확장

이제 

PHP Web App 배포를 합니다.

```text
kubectl run php-apache --image=us.gcr.io/k8s-artifacts-prod/hpa-example --requests=cpu=200m --expose --port=80
```

HPA Resource 생성

컨테이너가 CPU 50% 초과하면 확장되도록 설정합니다.

```text
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

아래와 같은 출력 결과 예제를 확인할 수 있습니다.

```text
whchoi98:~/environment $ kubectl get hpa
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         10        1          86s
```

이제 load generator를 생성해서 Trigger Event를 발생해 봅니다.

```text
kubectl run -i --tty load-generator --image=busybox /bin/sh
```

