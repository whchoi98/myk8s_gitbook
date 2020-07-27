# shell

환경 변수 저장 관 - 이 랩에서 자주 사용되는 환경 변수들.

```text
export CLUSTER_NAME=eksworkshop
export AWS_REGION=ap-northeast-2
export ACCOUNT_ID="account-id"
export MASTER_ARN="MASTER_ARN"
export PUB_ROLE_NAME="PUB_ROLE_NAME"
export PRI_ROLE_NAME="PRI_ROLE_NAME"
export ALB_INGRESS_VERSION="v1.1.8"
export EBS_CNI_POLICY_NAME="Amazon_EBS_CSI_Driver"
export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
export MASTER_ARN=$(aws kms describe-key --key-id alias/eksworkshop --query KeyMetadata.Arn --output text)
```

bash\_profile에 저장

```text
echo "export CLUSTER_NAME=$CLUSTER_NAME" | tee -a ~/.bash_profile
echo "export AWS_REGION=$AWS_REGION" | tee -a ~/.bash_profile
echo "export ACCOUNT_ID=$ACCOUNT_ID" | tee -a ~/.bash_profile
echo "export MASTER_ARN=$MASTER_ARN" | tee -a ~/.bash_profile
echo "export PUB_ROLE_NAME=$PUB_ROLE_NAME" | tee -a ~/.bash_profile
echo "export PRI_ROLE_NAME=$PRI_ROLE_NAME" | tee -a ~/.bash_profile
echo "export MASTER_ARN=${MASTER_ARN}" | tee -a ~/.bash_profile
echo 'export ALB_INGRESS_VERSION="v1.1.8"' | tee -a ~/.bash_profile
echo 'export EBS_CNI_POLICY_NAME="Amazon_EBS_CSI_Driver"' | tee -a ~/.bash_profile
source ~/.bash_profile

```

code 생성

```text
cat <<EoF > ~/environment/eks-admin-service-account.yaml
...
EoF
```

kubectl에서 ELB 주소 추출

```text
export ELB_SERVICE_URL=$(kubectl get svc -n "namespace name" "pod name" --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
echo "ELB SERVICE URL = $ELB_SERVICE_URL"
```



