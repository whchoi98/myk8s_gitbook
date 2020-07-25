# kubectl Command

kubectl 주요 명령 

\#주요 자원 상태 확

```text
# 주요 자원 확
kubectl get all
kubectl get all --all-namespaces
kubectl get namespace
kubectl get nodes
kubectl get pods
kubectl get svc
kubectl get ingress
kubectl get daemosets
kubectl get namespace,nodes,pods,svc,ingress

#특정 namespaces
kubectl -n "namespace-name" get nodes,pods,svc,ingress

# 상세 내용 확
kubectl get all --all-namespace -o wide
```

\#배포 상태 확인

```text
kubectl describe deployment "app"
kubectl -n "namespace-name" describe deployment "app"
```

