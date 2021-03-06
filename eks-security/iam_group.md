# IAM 그룹 기반 관리



* k8sAdmin role - EKS Cluster 관리자 역할 . 
* k8sDev role - Developer Namespace에 대한 액세스 권한.
* k8sInteg role - integration Namespace에 대한 액세스 권한.

아래 역할은 EKS 클러스터 내에서 인증하는 데만 사용되기 때문에 AWS 권한이 필요하지 않습니다. EKS 클러스터에 액세스하기 위해 일부 IAM 그룹이 , 이러 역할을 맡도록 허용하는 데만 사용합니다.

```text
POLICY=$(echo -n '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"AWS":"arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ':root"},"Action":"sts:AssumeRole","Condition":{}}]}')

echo ACCOUNT_ID=$ACCOUNT_ID
echo POLICY=$POLICY

aws iam create-role \
  --role-name k8sAdmin \
  --description "Kubernetes administrator role (for AWS IAM Authenticator for Kubernetes)." \
  --assume-role-policy-document "$POLICY" \
  --output text \
  --query 'Role.Arn'

aws iam create-role \
  --role-name k8sDev \
  --description "Kubernetes developer role (for AWS IAM Authenticator for Kubernetes)." \
  --assume-role-policy-document "$POLICY" \
  --output text \
  --query 'Role.Arn'
  
aws iam create-role \
  --role-name k8sInteg \
  --description "Kubernetes role for integration namespace in quick cluster." \
  --assume-role-policy-document "$POLICY" \
  --output text \
  --query 'Role.Arn'

```

## **IAM Group 생성** 

### **1.k8sAdmin IAM 그룹 생성**

**k8sAdmin 그룹은 k8sAdmin IAM Role을 위임 받게 됩니다.**

```text
aws iam create-group --group-name k8sAdmin

```

```text
ADMIN_GROUP_POLICY=$(echo -n '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAssumeOrganizationAccountRole",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ':role/k8sAdmin"
    }
  ]
}')
echo ADMIN_GROUP_POLICY=$ADMIN_GROUP_POLICY

aws iam put-group-policy \
--group-name k8sAdmin \
--policy-name k8sAdmin-policy \
--policy-document "$ADMIN_GROUP_POLICY"

```

### **2.k8sDev IAM 그룹 생성**

\*\*\*\*

```text
aws iam create-group --group-name k8sDev

```



```text
DEV_GROUP_POLICY=$(echo -n '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAssumeOrganizationAccountRole",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ':role/k8sDev"
    }
  ]
}')
echo DEV_GROUP_POLICY=$DEV_GROUP_POLICY

aws iam put-group-policy \
--group-name k8sDev \
--policy-name k8sDev-policy \
--policy-document "$DEV_GROUP_POLICY"

```

### **3. k8sInteg IAM 그룹 생성**

```text
aws iam create-group --group-name k8sInteg

```



```text
INTEG_GROUP_POLICY=$(echo -n '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAssumeOrganizationAccountRole",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ':role/k8sInteg"
    }
  ]
}')
echo INTEG_GROUP_POLICY=$INTEG_GROUP_POLICY

aws iam put-group-policy \
--group-name k8sInteg \
--policy-name k8sInteg-policy \
--policy-document "$INTEG_GROUP_POLICY"

```

t생성된 3개의 그룹을 확인해 봅니다.

```text
aws iam list-groups

```

## IAM User 생성

시나리오를 테스트하기 위해 생성 한 각 그룹에 대해 각각 하나씩 3 명의 User를 생성합니다.

```text
aws iam create-user --user-name AdminUser
aws iam create-user --user-name DevUser
aws iam create-user --user-name IntUser

```

연결된 그룹에 사용자를 추가합니다.

```text
aws iam add-user-to-group --group-name k8sAdmin --user-name AdminUser
aws iam add-user-to-group --group-name k8sDev --user-name DevUser
aws iam add-user-to-group --group-name k8sInteg --user-name IntUser

```

사용자가 그룹에 올바르게 추가되었는지 확인해 봅니다.

```text
aws iam get-group --group-name k8sAdmin
aws iam get-group --group-name k8sDev
aws iam get-group --group-name k8sInteg

```

Access Key에 대한 

```text
aws iam create-access-key --user-name PaulAdmin | tee /tmp/AdminUser.json
aws iam create-access-key --user-name JeanDev | tee /tmp/DevUser.json
aws iam create-access-key --user-name PierreInteg | tee /tmp/IntUser.json

```

## RBAC 구성

### 1.Namespace 만들기 

2개의 네임스페이스를 만듭니다.

* k8sDev group - namespace development
* k8sInteg - namespace integration

```text
kubectl create namespace integration
kubectl create namespace development

```

### **2. 네임 스페이스에 대한 액세스 구성**

kubernetes 사용자 DevUser 에게 development namespace 전체 액세스 권한을 제공 하는 kubernetes `role`및 `rolebinding`을 만듭니다.

```text
cat << EOF | kubectl apply -f - -n development
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: dev-role
rules:
  - apiGroups:
      - ""
      - "apps"
      - "batch"
      - "extensions"
    resources:
      - "configmaps"
      - "cronjobs"
      - "deployments"
      - "events"
      - "ingresses"
      - "jobs"
      - "pods"
      - "pods/attach"
      - "pods/exec"
      - "pods/log"
      - "pods/portforward"
      - "secrets"
      - "services"
    verbs:
      - "create"
      - "delete"
      - "describe"
      - "get"
      - "list"
      - "patch"
      - "update"
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: dev-role-binding
subjects:
- kind: User
  name: dev-user
roleRef:
  kind: Role
  name: dev-role
  apiGroup: rbac.authorization.k8s.io
EOF

```



```text
cat << EOF | kubectl apply -f - -n integration
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: integ-role
rules:
  - apiGroups:
      - ""
      - "apps"
      - "batch"
      - "extensions"
    resources:
      - "configmaps"
      - "cronjobs"
      - "deployments"
      - "events"
      - "ingresses"
      - "jobs"
      - "pods"
      - "pods/attach"
      - "pods/exec"
      - "pods/log"
      - "pods/portforward"
      - "secrets"
      - "services"
    verbs:
      - "create"
      - "delete"
      - "describe"
      - "get"
      - "list"
      - "patch"
      - "update"
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: integ-role-binding
subjects:
- kind: User
  name: integ-user
roleRef:
  kind: Role
  name: integ-role
  apiGroup: rbac.authorization.k8s.io
EOF

```

## K8s Role Access 구성

이전에 정의한 IAM 역할에 대한 액세스 권한을 EKS 클러스터에 **부여** 하려면 특정 **mapRoles** 를 `aws-auth`ConfigMap에 추가해야합니다. IAM 사용자를 직접 지정하는 대신 Role을 사용하여 클러스터에 액세스 할 때의 장점은 관리가 더 쉽다는 것입니다. 사용자를 추가하거나 제거 할 때마다 ConfigMap을 업데이트 할 필요가 없습니. IAM 그룹에서 사용자를 제거하고 IAM 그룹에 연결된 IAM 역할을 허용하도록 ConfigMap을 구성하기 만 하면 됩니다.

#### IAM 역할을 허용하도록 aws-auth ConfigMap 업데이트 <a id="update-the-aws-auth-configmap-to-allow-our-iam-roles"></a>

arn 그룹을 허용하거나 삭제하려면 kube-system 네임 스페이스 의 **aws-auth** ConfigMap을 편집해야합니다. 이 파일은 IAM 역할과 k8S RBAC 권한을 매핑합니다. 수동으로 편집 할 수 있습니다.

```text
eksctl create iamidentitymapping \
  --cluster eksworkshop-eksctl \
  --arn arn:aws:iam::${ACCOUNT_ID}:role/k8sDev \
  --username dev-user

eksctl create iamidentitymapping \
  --cluster eksworkshop-eksctl \
  --arn arn:aws:iam::${ACCOUNT_ID}:role/k8sInteg \
  --username integ-user

eksctl create iamidentitymapping \
  --cluster eksworkshop-eksctl \
  --arn arn:aws:iam::${ACCOUNT_ID}:role/k8sAdmin \
  --username admin \
  --group system:masters

```

삭제 할 때도 사용할 수 있습니다.

```text
eksctl delete iamidentitymapping --cluster eksworkshop-eksctlv --arn arn:aws:iam::xxxxxxxxxx:role/k8sDev --username dev-user

```

### 구성 확인 

구성된 confimap을 확인해 봅니다.

```text
kubectl get cm -n kube-system aws-auth -o yaml

```

eksctl을 활용하여 클러스터에서 관리되는 모든 ID 목록을 가져올 수 있습니다.

```text
eksctl get iamidentitymapping --cluster eksworkshop-eksctl

```

## EKS 액세스 시험

~/.aws/config를 통해서 전환이 가능합니다.

```text
if [ ! -d ~/.aws ]; then
  mkdir ~/.aws
fi

cat << EoF >> ~/.aws/config
[profile admin]
role_arn=arn:aws:iam::${ACCOUNT_ID}:role/k8sAdmin
source_profile=eksAdmin

[profile dev]
role_arn=arn:aws:iam::${ACCOUNT_ID}:role/k8sDev
source_profile=eksDev

[profile integ]
role_arn=arn:aws:iam::${ACCOUNT_ID}:role/k8sInteg
source_profile=eksInteg

EoF

```

~/.aws/credentials

```text
cat << EoF >> ~/.aws/credentials

[eksAdmin]
aws_access_key_id=$(jq -r .AccessKey.AccessKeyId /tmp/PaulAdmin.json)
aws_secret_access_key=$(jq -r .AccessKey.SecretAccessKey /tmp/PaulAdmin.json)

[eksDev]
aws_access_key_id=$(jq -r .AccessKey.AccessKeyId /tmp/JeanDev.json)
aws_secret_access_key=$(jq -r .AccessKey.SecretAccessKey /tmp/JeanDev.json)

[eksInteg]
aws_access_key_id=$(jq -r .AccessKey.AccessKeyId /tmp/PierreInteg.json)
aws_secret_access_key=$(jq -r .AccessKey.SecretAccessKey /tmp/PierreInteg.json)

EoF

```



```text
aws sts get-caller-identity --profile dev

```

