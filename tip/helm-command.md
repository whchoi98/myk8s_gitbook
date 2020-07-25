# helm command

## Install , 환경 구성 . 

### Helm 설치

```text
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
chmod 700 get_helm.sh
./get_helm.sh
```

### Helm 명령어 자동 완성

```text
helm completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
source <(helm completion bash)
```

### Helm version check

```text
helm version --short
```

## Repo구성 및 App 설치. 

### Helm reop 구성

```text
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### Helm repo에 검색

```text
helm search repo "app name"
helm search repo bitnami/nginx
```

### Helm repo 기반 설치

```text
helm install eksworkshop-nginx bitnami/nginix
```

### Helm 로 설치된 List 확인

```text
helm list 
```

### Helm을 통한 App 삭제 

```text
helm uninstall "App Name"
```

## Chart 구성 

### Helm Chart 생성

```text
helm create "Chart Name"
```

### Helm Chart 배포 전에 Dry run 하기

```text
helm install --debug --dry-run "Chart name"
```

### Helm Chart 배포

```text
helm install "Chart Name" "Chart Path"

```

### Helm 변경 후 Upgrade

```text
helm upgrade "Chart Name" ~/environment/eksdemo
```

### helm Chart 상태 및 history

```text
helm status "Chart Name"
helm history "Chart Name"

```

### Helm Rolling Back

```text
helm rollback "Chart Name" "Revison Number"

```

### Helm  배포 App 삭제

```text
helm uninstall "배포APP"
```

### Helm ChartMuseum 설치

```text
curl -LO https://s3.amazonaws.com/chartmuseum/release/latest/bin/linux/amd64/chartmuseum
chmod +x ./chartmuseum
```



