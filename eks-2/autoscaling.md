# AutoScaling 구성



![](../.gitbook/assets/image%20%2854%29.png)

HPA 구성

1.Metric Server 설치



```text
kubectl create namespace metrics
helm install metrics-server \
    stable/metrics-server \
    --version 2.9.0 \
    --namespace metrics
```

