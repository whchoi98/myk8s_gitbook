# kubectl Command

## kubectl 주요 명령 

### 주요 자원 상태 확인 

```text
# 주요 자원 확인 
kubectl get all
kubectl get all --all-namespaces
kubectl get namespace
kubectl get nodes
kubectl get pods
kubectl get svc
kubectl get ingress
kubectl get daemosets
kubectl get namespace,nodes,pods,svc,ingress

#Detials
kubectl get nodes -o wide
kubectl get pods -o wide
kubectl get svc -o wide
kubectl get ingress -o wide
kubectl get daemosets -o wide

#특정 namespaces
kubectl -n "namespace-name" get nodes,pods,svc,ingress

# 상세 내용 확인 
kubectl get all --all-namespace -o wide
```

### 생성 및 배포 

```text
kubectl apply -f "manifest file path"
kubectl -n "namespaces" apply -f "manifest file path"
kubectl scale deployment "pod_name" --replicas=3
kubectl -n "namespaces" scale deployment "pod_name" --replicas=3

```

### ELB 주소 확인

```text
kubectl get svc -n "namespace name" "pod name" --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}"
```

### 배포 상태 확인

```text
kubectl describe deployment "app"
kubectl get deployment "app"
kubectl get deployment "app" -o wide
kubectl -n "namespace-name" get deployment "app"
kubectl -n "namespace-name" describe deployment "app"
```

### 삭제

```text
kubectl delete -f "manifest file path"
kubectl -n "namespaces" delete -f "manifest file path"
```

### log확인

```text
kubectl logs -n "namespace" "pods name"
```

### 인증 

```text
kubectl -n kube-system edit configmaps aws-auth
kubectl edit clusterrole cluster-admin
```

