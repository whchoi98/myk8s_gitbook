---
description: 'update : 2023-05-06 / 15min'
---

# 인증/자격증명 및 환경 구성

## 인증 및 자격증명 환경 구성

### 1.Cloud9을 위한 IAM 역할 생성

**IAM - 역할 - 역할 만들기**

IAM 서비스 대쉬보드 접속 및 역할(Role) 생성을 합니다. AWS 서비스에서 IAM을 선택하고, "역할 만들기"를 선택합니다.

<figure><img src="../.gitbook/assets/image (61).png" alt=""><figcaption></figcaption></figure>

### 2.역할(Role)을 만듭니다.

EC2 - 다음: 권한

역할 만들기 단계에서 EC2를 선택하고, "다음:권한"을 선택합니다.

<figure><img src="../.gitbook/assets/image (423).png" alt=""><figcaption></figcaption></figure>

권한정책 연결에서 `"AdministratorAccess"`를 선택합니다.

```
AdministratorAccess
```

![](<../.gitbook/assets/image (476).png>)

역할 이름 생성 및 정책 연결을 확인합니다.

역할 이름을 아래와 같이 선언합니다.

```
eksworkshop-admin
```

![](<../.gitbook/assets/image (65).png>)

### 3.Cloud9 권한 설정.

Cloud9 상단 좌측 메뉴에서 EC2 대쉬보드로 접속을 선택하거나, AWS 서비스에서 EC2 대쉬보드로 접속합니다.

![](<../.gitbook/assets/image (347).png>)

EC2 대쉬보드의 인스턴스에는 이미 생성된 Cloud9 EC2 인스턴스가 보입니다.

"작업"-"보안"-"IAM 역할 연결/바꾸기"를 선택합니다.&#x20;

![](<../.gitbook/assets/eks\_cloud9\_iam\_role\_change (1).png>)

앞서 생성한 IAM 역할 **"eksworkshop-admin"**을 선택합니다.&#x20;

![](<../.gitbook/assets/image (38).png>)

정상적으로 Cloud9 IDE의 IAM 역할이 정상적으로 연결되었는지 확인합니다.

![](<../.gitbook/assets/image (455).png>)

### 4. 기존자격 증명 파일 제거

Cloud9의 기존 자격증명과 임시 자격 증명등을 비활성화 합니다.

Cloud9 콘솔에서 아래와 같이 명령을 입력합니다.

```
aws cloud9 update-environment  --environment-id $C9_PID --managed-credentials-action DISABLE

```

아래와 같이 Cloud9 UI 환경에서도 설정할 수 있습니다. 위의 Terminal 에서의 입력된 명령이 정상적으로 처리되었다면, 추가 작업은 필요없습니다.

Cloud9에서 설정 환경을 아래와 같이 선택합니다.

![](<../.gitbook/assets/image (68).png>)

Cloud9 설정환경에서 "AWS managed temporary credential"을 비활성합니다.

<figure><img src="../.gitbook/assets/image (419).png" alt=""><figcaption></figcaption></figure>

### 5. Cloud9 IDE 역할 점검

Cloud9 이 올바른 IAM 역할을 사용하고 있는지 확인합니다. \
(앞서 선언한 IAM Role 이름을 "eksworkshop-admin"으로 선언하지 않은 경우에는 다른 이름으로 변경합니다.)

```
# 임시 자격증명을 사용하지 않도록 기존 자격 증명 파일을 제거합니다.
rm -vf ${HOME}/.aws/credentials
# Cloud9 이 올바른 IAM 역할을 사용하고 있는지 확인합니다. 
aws sts get-caller-identity --region ap-northeast-2 --query Arn | grep eksworkshop-admin -q && echo "IAM role valid" || echo "IAM role NOT valid"
# 실제 Role을 확인해 봅니다.
aws sts get-caller-identity --region ap-northeast-2

```

### 6. Shell 환경변수 저장

Account ID, Region 정보 등을 환경변수와 프로파일에 저장해 두고, EKSworkshop 에서 사용합니다.

```
# Account , Region 정보를 AWS Cli로 추출합니다.
export ACCOUNT_ID=$(aws sts get-caller-identity --region ap-northeast-2 --output text --query Account)
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
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

## SSH 키 생성

eksworkshop에서 사용될 키를 생성합니다.

### 7.SSH Key 생성

Cloud9에서 ssh key를 생성합니다.

```
cd ~/environment/
ssh-keygen
```

아래는 예제입니다.key 이름을 eksworkshop으로 생성하였습니다.

```
ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/ec2-user/.ssh/id_rsa): eksworkshop
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in eksworkshop.
Your public key has been saved in eksworkshop.pub.
The key fingerprint is:
SHA256:zAcvK4NFMkIaB4Vh507xv6LsYX/GMhXdb50TBZQasYE ec2-user@ip-172-31-41-2
The key's randomart image is:
+---[RSA 2048]----+
|+*+o       .oooo |
|o=o o     E .o. .|
|. .oo.. o . .o  .|
|  o. +.+ + ..  . |
|   .  ..S o . . o|
|     o ..+   o + |
|   o..=..   .   .|
|  o +o.*         |
|  .+ .=          |
+----[SHA256]——+
```

Cloud9 IDE Terminal 에서 이후 생성된 Workernode 또는 EC2에 접속하기 위해 pem key의 권한 설정을 해 둡니다.

```
cd ~/environment/
mv ./eksworkshop ./eksworkshop.pem
chmod 400 ./eksworkshop.pem

```

### 8.ssh key 전송

생성된 SSH key를  Key 페어로 전송합니다. 앞서 eksworkshop.pub 로 public key가 생성되었습니다. \
(AWS CLI Version 2.0)

```
cd ~/environment/
# ap-northeast-2 로 전송합니다.
aws ec2 import-key-pair --key-name "eksworkshop" --public-key-material fileb://./eksworkshop.pub --region ap-northeast-2
# ap-northeast-1 으로 전송합니다.
aws ec2 import-key-pair --key-name "eksworkshop" --public-key-material fileb://./eksworkshop.pub --region ap-northeast-1

```

## CMK  생성

### 9.KMS 소개

AWS KMS(Key Management Service)를 사용하면 손쉽게 암호화 키를 생성 및 관리하고 다양한 AWS 서비스와 애플리케이션에서의 사용을 제어할 수 있습니다. AWS KMS는 FIPS 140-2에 따라 검증되었거나 검증 과정에 있는 하드웨어 보안 모듈을 사용하여 키를 보호하는 안전하고 복원력 있는 서비스입니다. 또한, AWS KMS는 AWS CloudTrail과도 통합되어 모든 키 사용에 관한 로그를 제공함으로써 각종 규제 및 규정 준수 요구 사항을 충족할 수 있게 지원합니다.&#x20;

![](<../.gitbook/assets/image (51).png>)

EKS에서는 K8s와 Key를 통한 인증이 많이 일어납니다. 안전한 관리를 위해 Option으로 구성할 수 있습니다.

비밀 암호화(Secrets encryption)를 활성화하면 AWS Key Management Service(KMS) 키를 사용하여 클러스터의 etcd에 저장된 Kubernetes 비밀 봉투 암호화를 제공할 수 있습니다. 이 암호화는 EKS 클러스터의 일부로 etcd에 저장된 모든 데이터(비밀 포함)에 대해 기본적으로 활성화되는 EBS 볼륨 암호화에 추가됩니다.

EKS 클러스터에 비밀 암호화(Secrets encryption)를 사용하면 사용자가 정의하고 관리하는 KMS 키로 Kubernetes 비밀 암호화하여 Kubernetes 애플리케이션에 대한 심층 방어 전략을 배포할 수 있습니다.

### 10.CMK 생성

K8s Secret 암호화를 할 때, EKS 클러스터에서 사용할 CMK(Cusomter Management Key : 사용자 관리형 키)를 생성하고 변수에 저장해 둡니다.

```
# kms 를 생성합니다.
aws kms create-alias --alias-name alias/eksworkshop --target-key-id $(aws kms create-key --query KeyMetadata.Arn --output text)
# kms 값을 환경변수에 저장합니다.
export MASTER_ARN=$(aws kms describe-key --key-id alias/eksworkshop --query KeyMetadata.Arn --output text)
echo "export MASTER_ARN=${MASTER_ARN}" | tee -a ~/.bash_profile
echo $MASTER_ARN

```

정상적으로 Key가 생성되었는지 **`AWS 관리 콘솔 - KMS - 고객관리형 키`**에서 확인합니다.

![](<../.gitbook/assets/image (146).png>)

출력 결과 예제

```
whchoi98:~/environment $ echo $MASTER_ARN
arn:aws:kms:ap-northeast-2:909121566064:key/9a0c5a6c-be81-4463-90e4-e3b1252d96fc
whchoi98:~/environment $ echo "export MASTER_ARN=${MASTER_ARN}" | tee -a ~/.bash_profile
export MASTER_ARN=arn:aws:kms:ap-northeast-2:909121566064:key/9a0c5a6c-be81-4463-90e4-e3b1252d96fc
```

## 인증/자격증명 및 환경 구성 요약

#### **1.Cloud9를 위한 IAM 역할(Role) 생성 및 IAM 역할 연결.**

**`AWS 서비스 - IAM - 역할 - 역할 만들기`**

**`역할만들기 - AWS서비스 - EC2선택`**

**`역할만들기 - 정책이름 - AdministratorAccess 선택`**

**`역할만들기 - 역할이름 - eksworkshop-admin 생성`**

**`AWS 서비스 - EC2 - 인스턴스 - "Cloud9 인스턴스 선택" - 작업 - 보안 - IAM 역할 수정`**

**`IAM 역할 수정 - IAM 역할 - "eksworkshop-admin 선택"`**

**`Cloud9 콘솔 - Preference - AWS Settings - Credentials - AWS managed temporary credentials 비활성`**

**2.Cloud 9 또는 기타장소에서 Key 생성.**

**3. Public Key를 AWS 키페어 집합소에 전송.**

**4. KMS 기반 key 생성.**



