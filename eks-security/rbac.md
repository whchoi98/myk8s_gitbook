# RBAC

## Overview

Role-based access control \(RBAC\)은 기업 내에서 개별 사용자의 역할에 따라 컴퓨터 또는 네트워크 리소스에 대한 액세스를 규제하는 방법입니다.

RBAC의 핵심적인 구성요소는 다음과 같습니다.

#### Entity

Group, User 또는 Service Account \(특정 작업을 실행하고 이를 수행하는 데 권한이 필요한 애플리케이션을 나타내는 ID\).

**Resource**

엔티티가 특정 작업을 사용하여 액세스하려는 포드 , 서비스 등.

**Role**

엔티티가 다양한 리소스에 대해 수행 할 수있는 작업에 대한 규칙을 정의하는 데 사용.

**Role Binding**

Role과 Subject의 연결을 정의하며, 두 가지 유형의 역할 \(Role, ClusterRole\)과 각 바인딩 \(RoleBinding, ClusterRoleBinding\)이 있습니다. \( 네임 스페이스 또는 클러스터 전체의 인증을 구분\)

**NameSpace**

가상 클러스터를 네임스페이스라고 하며, 네임스페이스는 여러 개의 팀이나, 프로젝트에 걸쳐서 많은 사용자가 있는 환경에서 사용

## RBAC 시험용 환경 구성

### 1. Pod 구성 

Pod 생성

```text
kubectl create namespace rbac-test
kubectl create deploy nginx --image=nginx -n rbac-test

```

생성한 Pod 확인을 합니다.

```text
kubectl get all -n rbac-test

```

Cloud9 터미널에서 rbac-user 라는 새로운 User 자격증명을 생성하고 저장합니다. 

```text
aws iam create-user --user-name rbac-user
aws iam create-access-key --user-name rbac-user | tee /tmp/create_output.json

```

Cluster를 생성한 Admin\(Cloud9 EC2\)과 새로운 rback-user 간에 쉽게 전환할 수 있도록 아래 처럼 shell을 작성해 둡니다.

```text
cat << EoF > rbacuser_creds.sh
export AWS_SECRET_ACCESS_KEY=$(jq -r .AccessKey.SecretAccessKey /tmp/create_output.json)
export AWS_ACCESS_KEY_ID=$(jq -r .AccessKey.AccessKeyId /tmp/create_output.json)
EoF

```

### 

### 2. IAM 사용자 Mapping

rbac-user라는 k8s 사용자를 정의하고 해당 IAM 사용자에 매핑합니다. 다음을 실행하여 기존 ConfigMap을 가져오고 aws-auth.yaml 이라는 파일에 저장합니다.

```text
kubectl get configmap -n kube-system aws-auth -o yaml > aws-auth.yaml

```

IAM 사용자 "rbac-user" 매핑을 기존 configMap에 추가합니다.

```text
cat << EoF >> aws-auth.yaml
data:
  mapUsers: |
    - userarn: arn:aws:iam::${ACCOUNT_ID}:user/rbac-user
      username: rbac-user
EoF

```

생성한 aws-auth.yml을 확인 합니다.

```text
cat aws-auth.yaml

```

Configmap을 적용합니다.

```text
kubectl apply -f aws-auth.yaml

```

### 

### 3. 신규 사용자 테스트

지금까지 EKS Cluster 운영은 관리자로 클러스터에 액세스했습니다. 이제 새로 생성 된 rbac-user로 클러스터에 액세스하면 어떻게되는지 살펴 보겠습니다.

다음 명령을 실행하여 rbac-user의 AWS IAM 사용자 환경 변수를 실행해서, 기존 sts 정보와 새로운 sts 정보를 비교해 봅니다.

```text
aws sts get-caller-identity > master_sts.txt
. rbacuser_creds.sh
aws sts get-caller-identity > rbacuser_sts.txt

```

rbac-user의 컨텍스트에서 호출을하고 있으므로 , kubectl 명령 권한에 문제가 발생합니다.

```text
kubectl get pods -n rbac-test

```

user를 생성하는 것만으로는 해당 사용자에게 클러스터의 리소스에 대한 액세스 권한이 부여되지 않습니다. 이를 해결려면 Role 정의한 다음 사용자를 해당 Role 바인딩해야합니다.

## Role & Binding 

### 1.Role/RoleBinding 

새 사용자 rbac-user가 있지만 아직 어떤 역할에도 바인딩되지 않았습니다. 그렇게하려면 기본 관리자 사용자로 다시 전환해야합니다.

rbac-user로 정의하는 환경 변수를 설정 해제하려면 아래 명령을 실행합니.

```text
unset AWS_SECRET_ACCESS_KEY
unset AWS_ACCESS_KEY_ID

```

다시 admin 사용자이고 더 이상 rbac-user가 아님을 확인하려면 앞서 저장해둔 master\_sts.txt 파일 결과와 비교해 봅니다.

```text
aws sts get-caller-identity

```

이제 다시 관리자 모드로 전환되었으므로 모든 kubectl 조회가 가능하지만, "rbac-user"에게  해당 네임 스페이스에 대해서만 pod-reader라는 Role을 만들어 봅니다.  pod-reader의 Role은 rbac-test namespace에 대한 조회와 deploy등의 권한을 가지게 됩니다.

```text
cat << EoF > rbacuser-role.yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: rbac-test
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["list","get","watch"]
- apiGroups: ["extensions","apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
EoF

```

생성된 Role에 이제 Rolebinding에 대한 yaml 파일을 생성합니다.

```text
cat << EoF > rbacuser-role-binding.yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: rbac-test
subjects:
- kind: User
  name: rbac-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EoF

```

이제 생성한 Role과 Rolebinding을 실행해 봅니다.

```text
kubectl apply -f rbacuser-role.yaml
kubectl apply -f rbacuser-role-binding.yaml

```

### 2.Role/RoleBinding 시험 

이제 사용자, 역할 및 RoleBinding이 정의되었으므로 rbac-user로 다시 전환하고 테스트 해 봅니다.

rbac-user로 다시 전환하려면 rbac-user 환경 변수를 제공하는 다음 명령을 실행하고 해당 변수가 사용되었는지 확인합니다.

```text
. rbacuser_creds.sh
aws sts get-caller-identity

```

rbac-user로서 다음을 실행하여 rbac 네임 스페이스에서 Pod 정보를 조회해 봅니다.

```text
kubectl get pods -n rbac-test

```

rbac-user로 다음을 실행해서 권한이 제대로 바인딩되었는지 확인해 봅니다.

```text
kubectl get pods -n kube-system

```



```text
unset AWS_SECRET_ACCESS_KEY
unset AWS_ACCESS_KEY_ID
kubectl delete namespace rbac-test

```

rback-user 에 대한 모든 정보와 configMap을 삭제하려면 아래를 수행합니다.

```text
rm rbacuser_creds.sh
rm rbacuser-role.yaml
rm rbacuser-role-binding.yaml
aws iam delete-access-key --user-name=rbac-user --access-key-id=$(jq -r .AccessKey.AccessKeyId /tmp/create_output.json)
aws iam delete-user --user-name rbac-user
rm /tmp/create_output.json

```

기존 aws-auth.yaml 파일을 편집하여 기존 configMap에서 rbac-user 매핑을 제거합니다.

```text
data:
  mapUsers: |
    []
```

ConfigMap을 적용하고 aws-auth.yaml 파일을 삭제합니다.

```text
kubectl apply -f aws-auth.yaml
rm aws-auth.yaml

```



