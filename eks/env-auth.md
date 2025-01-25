---
description: 'update : 2024-01-28 / 10min'
---

# 인증/자격증명 및 환경 구성

## 인증 및 자격증명 환경 구성

### 1. Cloud9 IDE 역할 점검

Cloud9 이 올바른 IAM 역할을 사용하고 있는지 확인합니다. \
(앞서 선언한 IAM Role 이름을 "c9\_role"으로 선언하지 않은 경우에는 다른 이름으로 변경합니다.)

```
# Cloud9 이 올바른 IAM 역할을 사용하고 있는지 확인합니다. 
aws sts get-caller-identity --region ap-northeast-2 --query Arn | grep c9_role -q && echo "IAM role valid" || echo "IAM role NOT valid"
# 실제 Role을 확인해 봅니다.
aws sts get-caller-identity --region ap-northeast-2

```

### 2. Shell 환경변수 저장

Account ID, Region 정보 등을 환경변수와 프로파일에 저장해 두고, EKSworkshop 에서 사용합니다.

```
# Account , Region 정보를 AWS Cli로 추출합니다.
export ACCOUNT_ID=$(aws sts get-caller-identity --region ap-northeast-2 --output text --query Account)
export AWS_REGION=ap-northeast-2
echo $ACCOUNT_ID
echo $AWS_REGION
# bash_profile에 Account 정보, Region 정보를 저장합니다.
echo "export ACCOUNT_ID=${ACCOUNT_ID}" | tee -a ~/.bash_profile
echo "export AWS_REGION=${AWS_REGION}" | tee -a ~/.bash_profile
aws configure set default.region ${AWS_REGION}
aws configure --profile default list

```

출력결과 예제는 아래와 같습니다.

```
whchoi98:~ $ aws configure --profile default list
      Name                    Value             Type    Location
      ----                    -----             ----    --------
   profile                  default           manual    --profile
access_key     ****************OF5M         iam-role    
secret_key     ****************sDz7         iam-role    
    region           ap-northeast-2              env    ['AWS_REGION', 'AWS_DEFAULT_REGION']
```

## CMK  생성

### 3.KMS 소개

AWS KMS(Key Management Service)를 사용하면 손쉽게 암호화 키를 생성 및 관리하고 다양한 AWS 서비스와 애플리케이션에서의 사용을 제어할 수 있습니다. AWS KMS는 FIPS 140-2에 따라 검증되었거나 검증 과정에 있는 하드웨어 보안 모듈을 사용하여 키를 보호하는 안전하고 복원력 있는 서비스입니다. 또한, AWS KMS는 AWS CloudTrail과도 통합되어 모든 키 사용에 관한 로그를 제공함으로써 각종 규제 및 규정 준수 요구 사항을 충족할 수 있게 지원합니다.&#x20;

![](<../.gitbook/assets/image (51).png>)

EKS에서는 K8s와 Key를 통한 인증이 많이 일어납니다. 안전한 관리를 위해 Option으로 구성할 수 있습니다.

비밀 암호화(Secrets encryption)를 활성화하면 AWS Key Management Service(KMS) 키를 사용하여 클러스터의 etcd에 저장된 Kubernetes 비밀 봉투 암호화를 제공할 수 있습니다. 이 암호화는 EKS 클러스터의 일부로 etcd에 저장된 모든 데이터(비밀 포함)에 대해 기본적으로 활성화되는 EBS 볼륨 암호화에 추가됩니다.

EKS 클러스터에 비밀 암호화(Secrets encryption)를 사용하면 사용자가 정의하고 관리하는 KMS 키로 Kubernetes 비밀 암호화하여 Kubernetes 애플리케이션에 대한 심층 방어 전략을 배포할 수 있습니다.

### 4.CMK 생성

K8s Secret 암호화를 할 때, EKS 클러스터에서 사용할 CMK(Cusomter Management Key : 사용자 관리형 키)를 생성하고 변수에 저장해 둡니다.

```
# kms 를 생성합니다.
aws kms create-alias --alias-name alias/eksworkshop --target-key-id $(aws kms create-key --query KeyMetadata.Arn --output text)
# kms 값을 환경변수에 저장합니다.
export MASTER_ARN=$(aws kms describe-key --key-id alias/eksworkshop --query KeyMetadata.Arn --output text)
echo "export MASTER_ARN=${MASTER_ARN}" | tee -a ~/.bash_profile
echo $MASTER_ARN

```

정상적으로 Key가 생성되었는지 **`AWS 관리 콘솔 - KMS - 고객관리형 키`**&#xC5D0;서 확인합니다.

![](<../.gitbook/assets/image (146).png>)

출력 결과 예제

```
whchoi98:~/environment $ echo $MASTER_ARN
arn:aws:kms:ap-northeast-2:909121566064:key/9a0c5a6c-be81-4463-90e4-e3b1252d96fc
whchoi98:~/environment $ echo "export MASTER_ARN=${MASTER_ARN}" | tee -a ~/.bash_profile
export MASTER_ARN=arn:aws:kms:ap-northeast-2:909121566064:key/9a0c5a6c-be81-4463-90e4-e3b1252d96fc
```



