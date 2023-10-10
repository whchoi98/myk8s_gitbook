---
description: 'Update : 2023-09-21'
---

# RBAC

## Overview

{% hint style="info" %}
AWS 보안인증과 역할 ,정책 관리등과 K8s는 매우 유사합니다. 아래 AWS IAM 에 대해 참조합니다.

[https://whchoi98.gitbook.io/aws-iam/iam-role](https://whchoi98.gitbook.io/aws-iam/iam-role)
{% endhint %}

Role-based access control (RBAC)은 기업 내에서 개별 사용자의 역할에 따라 컴퓨터 또는 네트워크 리소스에 대한 액세스를 규제하는 방법입니다.

RBAC의 핵심적인 구성요소는 다음과 같습니다.

#### Entity

Group, User 또는 Service Account (특정 작업을 실행하고 이를 수행하는 데 권한이 필요한 애플리케이션을 나타내는 ID).

**Resource**

엔티티가 특정 작업을 사용하여 액세스하려는 포드 , 서비스 등.

**Role**

엔티티가 다양한 리소스에 대해 수행 할 수있는 작업에 대한 규칙을 정의하는 데 사용.

**Role Binding**

Role과 Subject의 연결을 정의하며, 두 가지 유형의 역할 (Role, ClusterRole)과 각 바인딩 (RoleBinding, ClusterRoleBinding)이 있습니다. ( 네임 스페이스 또는 클러스터 전체의 인증을 구분)

**NameSpace**

가상 클러스터를 네임스페이스라고 하며, 네임스페이스는 여러 개의 팀이나, 프로젝트에 걸쳐서 많은 사용자가 있는 환경에서 사용

## RBAC 시험용 환경 구성

### 1. Pod 구성&#x20;

Pod 생성

```
kubectl create namespace rbac-test
kubectl create deploy nginx --image=nginx -n rbac-test

```

생성한 Pod 확인을 합니다.

```
kubectl get all -n rbac-test

```

Cloud9 터미널에서 rbac-user 라는 새로운 User 자격증명을 생성하고 저장합니다.&#x20;

```
aws iam create-user --user-name rbac-user
aws iam create-access-key --user-name rbac-user | tee /tmp/create_output.json

```

생성한 사용자를 IAM 콘솔에서 확인해 봅니다.

![](<../.gitbook/assets/image (265).png>)

Cluster를 생성한 Admin(Cloud9 EC2)과 새로운 rbac-user 간에 쉽게 전환할 수 있도록 아래 처럼 shell을 작성해 둡니다.

```
mkdir ~/environment/rbac
cd ~/environment/rbac
cat << EoF > ~/environment/rbac/rbacuser_creds.sh
export AWS_SECRET_ACCESS_KEY=$(jq -r .AccessKey.SecretAccessKey /tmp/create_output.json)
export AWS_ACCESS_KEY_ID=$(jq -r .AccessKey.AccessKeyId /tmp/create_output.json)
EoF
chmod 755 ~/environment/rbac/rbacuser_creds.sh

```

###

### 2. IAM 사용자 Mapping

rbac-user라는 k8s 사용자를 정의하고 해당 IAM 사용자에 매핑합니다. 다음을 실행하여 기존 ConfigMap을 가져오고 aws-auth.yaml 이라는 파일에 저장합니다. 기본 configmap도 저장해 둡니다.

```
kubectl get configmap -n kube-system aws-auth -o yaml > ~/environment/rbac/backup-aws-auth.yaml
kubectl get configmap -n kube-system aws-auth -o yaml > ~/environment/rbac/aws-auth.yaml

```

IAM 사용자 "rbac-user" 매핑을 기존 configMap에 추가합니다.

```
eksctl create iamidentitymapping \
  --cluster ${ekscluster_name} \
  --arn arn:aws:iam::${ACCOUNT_ID}:user/rbac-user \
  --username rbac-user

```

생성한 aws-auth.yml을 확인 합니다.

```
kubectl get configmap -n kube-system aws-auth -o yaml

```

아래와 같이 새롭게 추가된 것을 확인 할 수 있습니다.

```
  mapUsers: |
    - userarn: arn:aws:iam::300861432382:user/rbac-user
      username: rbac-user
```



### 3. 신규 사용자 테스트

지금까지 EKS Cluster 운영은 관리자로 클러스터에 액세스했습니다. 이제 새로 생성 된 rbac-user로 클러스터에 액세스하면 어떻게되는지 살펴 보겠습니다.

다음 명령을 실행하여 기존 권한을 "master\_sts.txt"로 확인해 보고, rbac-user로 권한을 변경합니다.

기존 sts 정보와 새로운 sts 정보를 비교해 봅니다.

```
aws sts get-caller-identity > ~/environment/rbac/master_sts.txt
cd ~/environment/rbac
. rbacuser_creds.sh
aws sts get-caller-identity > ~/environment/rbac/rbacuser_sts.txt

```

```
#master sts
{
    "UserId": "AROAUMDF5EI7KFXSND2T5:i-0ee8ebb7ae4b880de",
    "Account": "300861432382",
    "Arn": "arn:aws:sts::300861432382:assumed-role/eksworkshop-admin/i-0ee8ebb7ae4b880de"
}

#rabcuser sts
{
    "UserId": "AIDAUMDF5EI7CPY5HK32C",
    "Account": "300861432382",
    "Arn": "arn:aws:iam::300861432382:user/rbac-user"
}

```

rbac-user의 컨텍스트에서 호출을하고 있으므로 , 아래 명령을 실행하면 kubectl 명령 권한에 문제가 발생합니다.

```
kubectl get pods -n rbac-test

```

아래와 같은 메세지로 에러가 발생합니다.

```
$ kubectl get pods -n rbac-test
Error from server (Forbidden): pods is forbidden: User "rbac-user" cannot list resource "pods" in API group "" in the namespace "rbac-test"
```

## Role & Binding&#x20;

### 1.Role/RoleBinding&#x20;

새 사용자 rbac-user가 있지만 아직 어떤 역할에도 바인딩되지 않았습니다. 그렇게하려면 기본 관리자 사용자로 다시 전환해야합니다. 아래에서 처럼 kubectl API 조회가 되지 않습니다.

rbac-user로 정의하는 환경 변수를 설정 해제하려면 아래 명령을 실행합니다.&#x20;

```
unset AWS_SECRET_ACCESS_KEY
unset AWS_ACCESS_KEY_ID
# 환경변수 설정이 해제되고 MASTER 권한으로 변경되었는지 확인합니다.
kubectl -n rbac-test get pods

```

다시 admin 사용자이고 더 이상 rbac-user가 아님을 확인하려면 앞서 저장해둔 master\_sts.txt 파일 결과와 비교해 봅니다.

```
aws sts get-caller-identity

```

관리자권한 모드로 전환된 것을 확인할 수 있습니다.

```
$ aws sts get-caller-identity
{
    "UserId": "AROAUMDF5EI7KFXSND2T5:i-0ee8ebb7ae4b880de",
    "Account": "300861432382",
    "Arn": "arn:aws:sts::300861432382:assumed-role/eksworkshop-admin/i-0ee8ebb7ae4b880de"
}
```

이제 다시 관리자 모드로 전환되었으므로 모든 kubectl 조회가 가능하지만, "rbac-user"에게  해당 네임 스페이스에 대해서만 pod-reader라는 Role을 만들어 봅니다.  pod-reader의 Role은 rbac-test namespace에 대한 조회와 deploy등의 권한을 가지게 됩니다.

```
kubectl -n rbac-test get pods
```

아래에서 Role을 생성합다.

```
cat << EoF > ~/environment/rbac/rbacuser-role.yaml
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

```
cat << EoF > ~/environment/rbac/rbacuser-role-binding.yaml
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

```
kubectl apply -f ~/environment/rbac/rbacuser-role.yaml
kubectl apply -f ~/environment/rbac/rbacuser-role-binding.yaml

```

### 2.Role/RoleBinding 시험&#x20;

이제 사용자, 역할 및 RoleBinding이 정의되었으므로 rbac-user로 다시 전환하고 테스트 해 봅니다.

rbac-user로 다시 전환하려면 rbac-user 환경 변수를 제공하는 다음 명령을 실행하고 해당 변수가 사용되었는지 확인합니다.

```
cd ~/environment/rbac/
. rbacuser_creds.sh
aws sts get-caller-identity

```

아래와 같이 rabc user로 권한이 전환된 것을 확인할 수 있습니다.

```
$ aws sts get-caller-identity
{
    "UserId": "AIDAUMDF5EI7CPY5HK32C",
    "Account": "300861432382",
    "Arn": "arn:aws:iam::300861432382:user/rbac-user"
}
```

rbac-user로서 다음을 실행하여 rbac 네임 스페이스에서 Pod 정보를 조회해 봅니다.

```
kubectl get pods -n rbac-test

```

```
whchoi:~/environment $ kubectl get pods -n rbac-test
NAME                     READY   STATUS    RESTARTS   AGE
nginx-6799fc88d8-lt6kc   1/1     Running   0          15m
```

rbac-user로 다음을 실행해서 권한이 제대로 바인딩되었는지 확인해 봅니다.

```
kubectl get pods -n kube-system

```

```
whchoi:~/environment $ kubectl get pods -n kube-system
Error from server (Forbidden): pods is forbidden: User "rbac-user" cannot list resource "pods" in API group "" in the namespace "kube-system"
```

{% hint style="info" %}
&#x20;rbac-user 에게는 pod-reader 권한만 주었기 때문에 get all은 에러가 발생합니다.
{% endhint %}

다시 master 권한으로 복귀합니다.

```
unset AWS_SECRET_ACCESS_KEY
unset AWS_ACCESS_KEY_ID
kubectl delete namespace rbac-test

```

아래 명령을 실행해서 정상적으로 Admin 권한으로 복귀되었는지 확인합니다.

```
aws sts get-caller-identity

```

아래와 같은 결과를 확인합니다.

```
$ aws sts get-caller-identity
{
    "UserId": "AROAUMDF5EI7KFXSND2T5:i-0ee8ebb7ae4b880de",
    "Account": "300861432382",
    "Arn": "arn:aws:sts::300861432382:assumed-role/eksworkshop-admin/i-0ee8ebb7ae4b880de"
}
```

rbac-user 에 대한 모든 정보와 configMap을 삭제하려면 아래를 수행합니다.삭제 하지 않더라도 랩 수행에는 이슈가 없습니다.

```
cd ~/environment/rbac
rm rbacuser_creds.sh
rm rbacuser-role.yaml
rm rbacuser-role-binding.yaml
aws iam delete-access-key --user-name=rbac-user --access-key-id=$(jq -r .AccessKey.AccessKeyId /tmp/create_output.json)
aws iam delete-user --user-name rbac-user
rm /tmp/create_output.json

```

기존 aws-auth.yaml 파일을 편집하여 기존 configMap에서 **`rbac-user`** 매핑을 제거합니다.

```
eksctl delete iamidentitymapping \
  --cluster ${ekscluster_name} \
  --arn arn:aws:iam::${ACCOUNT_ID}:user/rbac-user

```

