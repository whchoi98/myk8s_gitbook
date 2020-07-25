# shell

환경 변수 저 - 이 랩에서 자주 사용되는 환경 변수들.

```text
export CLUSTER_NAME=eksworkshop
export AWS_REGION=ap-northeast-2
export ACCOUNT_ID="account-id"
export MASTER_ARN="MASTER_ARN"
export PUB_ROLE_NAME="PUB_ROLE_NAME"
export PRI_ROLE_NAME="PRI_ROLE_NAME"
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
source ~/.bash_profile

```



