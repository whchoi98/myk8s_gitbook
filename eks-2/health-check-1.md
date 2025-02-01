---
description: 'Update : 2025.01.31'
---

# 고가용성 Health Check 구성 (작업중)

## 라이브니스, 레디니스 및 스타트업 프로브 구성하기

이 페이지에서는 컨테이너에 대한 **라이브니스(Liveness), 레디니스(Readiness) 및 스타트업(Startup) 프로브**를 구성하는 방법을 설명합니다.

### 개요

* **라이브니스 프로브(Liveness Probe)**: 컨테이너를 다시 시작해야 하는 시점을 kubelet이 알 수 있도록 합니다. 예를 들어, 애플리케이션이 실행 중이지만 진행할 수 없는 **데드락(deadlock)** 상태를 감지하여 컨테이너를 다시 시작할 수 있습니다.
* **레디니스 프로브(Readiness Probe)**: 컨테이너가 트래픽을 받을 준비가 되었는지 kubelet이 확인하는 데 사용됩니다. **Pod의 모든 컨테이너가 준비된 상태(Ready)여야 해당 Pod가 서비스의 백엔드로 사용될 수 있습니다**.
* **스타트업 프로브(Startup Probe)**: 애플리케이션이 완전히 시작되었는지 확인합니다. **스타트업 프로브가 설정되면, 성공하기 전까지 라이브니스 및 레디니스 프로브가 실행되지 않도록 보장합니다**.

#### 주의사항

* 라이브니스 프로브는 애플리케이션의 **회복 불가능한 오류를 감지할 수 있도록 신중하게 구성해야 합니다**. 잘못된 설정은 **과도한 컨테이너 재시작, 서비스 다운타임 증가, 남은 Pod의 부하 증가** 등의 문제를 초래할 수 있습니다.
* 라이브니스와 레디니스 프로브의 차이를 이해하고, 애플리케이션에 맞게 적절하게 설정해야 합니다.

***

### 사전 준비

* Kubernetes 클러스터 및 `kubectl` 명령어 사용 환경이 필요합니다.
* 클러스터에 최소한 **제어 플레인 노드가 아닌 두 개 이상의 노드**가 있어야 합니다.
* 클러스터가 없다면 **Minikube**를 사용하거나 아래의 Kubernetes 테스트 환경을 활용할 수 있습니다.
  * [Killercoda](https://www.killercoda.com/)
  * [Play with Kubernetes](https://labs.play-with-k8s.com/)

***

### 라이브니스 프로브 설정하기

#### 1. 실행 명령어를 사용하는 라이브니스 프로브

일부 애플리케이션은 장시간 실행 후 장애 상태에 빠질 수 있으며, **재시작해야만 복구되는 경우**가 있습니다. Kubernetes는 이를 감지하여 해결할 수 있도록 **라이브니스 프로브**를 제공합니다.

다음은 `busybox` 이미지를 사용하여 라이브니스 프로브를 구성하는 예제입니다.

```yaml
mkdir -p ~/environment/health_checks
cat <<EoF > ~/environment/health_checks/liveness-app.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
  namespace: healthchecks
spec:
  containers:
  - name: liveness
    image: registry.k8s.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
EoF
```

**설명**

* **프로브 실행 방식**: `cat /tmp/healthy` 명령어 실행
* **초기 지연 시간**(`initialDelaySeconds`): 5초 후 첫 번째 프로브 실행
* **실행 주기**(`periodSeconds`): 5초마다 실행
* 컨테이너 시작 후 30초 동안 `/tmp/healthy` 파일이 존재 → `cat /tmp/healthy` 성공(코드 0 반환)
* 30초 후 파일 삭제됨 → 실패 시 컨테이너 재시작

**Pod 생성**

```sh
kubectl create namespace healthchecks
kubectl apply -f ~/environment/health_checks/liveness-app.yaml
kubectl -n healthchecks get pod liveness-app

```

***

#### 2. HTTP 요청을 사용하는 라이브니스 프로브

다른 방법으로 **HTTP GET 요청을 사용하는 라이브니스 프로브**도 설정할 수 있습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - name: liveness
    image: registry.k8s.io/e2e-test-images/agnhost:2.40
    args:
    - liveness
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: Custom-Header
          value: Awesome
      initialDelaySeconds: 3
      periodSeconds: 3
```

**설명**

* **HTTP GET 요청을 사용**하여 `/healthz` 경로를 검사
* **포트 8080에서 서비스 실행**
* **HTTP 상태 코드 200\~399 → 성공**, 그 외는 실패로 간주하여 컨테이너 재시작

**Pod 생성**

```sh
kubectl apply -f https://k8s.io/examples/pods/probe/http-liveness.yaml
```

***

#### 3. TCP 소켓을 사용하는 라이브니스 프로브

**TCP 소켓을 여는 방식**으로 라이브니스 프로브를 설정할 수도 있습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: goproxy
  labels:
    app: goproxy
spec:
  containers:
  - name: goproxy
    image: registry.k8s.io/goproxy:0.1
    ports:
    - containerPort: 8080
    readinessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 10
```

**Pod 생성**

```sh
kubectl apply -f https://k8s.io/examples/pods/probe/tcp-liveness-readiness.yaml
```

***

#### 4. gRPC 프로브 설정

Kubernetes v1.27 이상에서는 **gRPC Health Checking Protocol**을 사용할 수 있습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: etcd-with-grpc
spec:
  containers:
  - name: etcd
    image: registry.k8s.io/etcd:3.5.1-0
    command: [ "/usr/local/bin/etcd", "--data-dir", "/var/lib/etcd", "--listen-client-urls", "http://0.0.0.0:2379"]
    ports:
    - containerPort: 2379
    livenessProbe:
      grpc:
        port: 2379
      initialDelaySeconds: 10
```

**Pod 생성**

```sh
kubectl apply -f https://k8s.io/examples/pods/probe/grpc-liveness.yaml
```

***

### 스타트업 프로브 사용하여 느린 컨테이너 보호

스타트업 시간이 긴 애플리케이션이 **라이브니스 프로브에 의해 너무 빨리 종료되지 않도록** 하기 위해 \*\*스타트업 프로브(Startup Probe)\*\*를 사용할 수 있습니다.

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: liveness-port
  failureThreshold: 30
  periodSeconds: 10
```

이 설정은 **최대 300초(30 × 10) 동안 애플리케이션이 완전히 시작되기를 기다립니다**.

***

### 결론

* **라이브니스 프로브**: 애플리케이션이 멈춘 경우 감지하여 컨테이너를 재시작
* **레디니스 프로브**: 애플리케이션이 요청을 받을 준비가 되었는지 확인
* **스타트업 프로브**: 초기 부팅 시간이 긴 애플리케이션 보호

애플리케이션의 특성에 맞게 **적절한 프로브를 조합**하여 Kubernetes 환경에서 안정적인 서비스를 운영할 수 있습니다.
