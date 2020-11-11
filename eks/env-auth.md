---
description: 'update : 2020-07-18'
---

# 인증/자격증명 및 환경 구성

## 인증 및 자격증명 환경 구성

### 1.Cloud9을 위한 IAM 역할 생성

**IAM - 역할 - 역할 만들기**

IAM 서비스 대쉬보드 접속 및 역할\(Role\) 생성을 합니다. AWS 서비스에서 IAM을 선택하고, "역할 만들기"를 선택합니다.

![](../.gitbook/assets/image%20%2836%29.png)

### 2.역할\(Role\)을 만듭니다.

EC2 - 다음: 권한

역할 만들기 단계에서 EC2를 선택하고, "다음:권한"을 선택합니다.

![](../.gitbook/assets/image%20%2814%29.png)

권한정책 연결에서 `"AdministratorAccess"`를 선택합니다.

![](../.gitbook/assets/image%20%2829%29.png)

역할 이름 생성 및 정책 연결을 확인합니다.

역할 이름

```text
eksworkshop-admin
```

![](../.gitbook/assets/image%20%2833%29.png)

### 3.Cloud9 권한 설정.

Cloud9 상단 좌측 메뉴에서 EC2 대쉬보드로 접속을 선택하거나, AWS 서비스에서 EC2 대쉬보드로 접속합니다.

![](../.gitbook/assets/image%20%2831%29.png)

EC2 대쉬보드의 인스턴스에는 이미 생성된 Cloud9 EC2 인스턴스가 보입니다.

"작업"-"인스턴스 설정"-"IAM 역할 연결/바꾸기"를 선택합니다.

![](../.gitbook/assets/image%20%2824%29.png)

앞서 생성한 IAM 역할 **"eksworkshop-admin"**을 선택합니다.

![](../.gitbook/assets/image%20%2812%29.png)

정상적으로 Cloud9 IDE의 IAM 역할이 정상적으로 연결되었는지 확인합니다.

![](../.gitbook/assets/image%20%2826%29.png)

### 4. 기존자격 증명 파일 제거

Cloud9의 기존 자격증명과 임시 자격 증명등을 비활성화 합니다.

Cloud9에서 설정 환경을 아래와 같이 선택합니다.

![](../.gitbook/assets/image%20%284%29.png)

Cloud9 설정환경에서 "AWS managed temporary credential"을 비활성합니다.

![](../.gitbook/assets/image%20%2815%29.png)

임시 자격증명이 설치되지 않도록 기존 자격 증명 파일을 제거합니다.

```text
rm -vf ${HOME}/.aws/credentials
```

### 5. Cloud9 IDE 역할 점검

Cloud9 이 올바른 IAM 역할을 사용하고 있는지 확인합니다.

```text
aws sts get-caller-identity --query Arn | grep eksworkshop-admin -q && echo "IAM role valid" || echo "IAM role NOT valid"

```

실제 Role의 Arn은 아래 명령을 통해 확인 할 수 있습니다.

```text
aws sts get-caller-identity

```

### 6. Shell 환경변수 저장

Account ID, Region 정보 등을 환경변수와 프로파일에 저장해 두고, EKSworkshop 에서 사용합니다.

```text
export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
echo $ACCOUNT_ID
echo $AWS_REGION

```

bash\_profile에 저장합니다.

```text
echo "export ACCOUNT_ID=${ACCOUNT_ID}" | tee -a ~/.bash_profile
echo "export AWS_REGION=${AWS_REGION}" | tee -a ~/.bash_profile
aws configure set default.region ${AWS_REGION}
aws configure --profile default list

```

출력결과 예제는 아래와 같습니다.

```text
whchoi98:~ $ aws configure --profile default list
      Name                    Value             Type    Location
      ----                    -----             ----    --------
   profile                  default           manual    --profile
access_key     ****************AHEW         iam-role    
secret_key     ****************8czE         iam-role    
    region           ap-northeast-2      config-file    ~/.aws/config
```

## SSH 키 생성

eksworkshop에서 사용될 키를 생성합니다.

### 1.SSH Key 생성

Cloud9에서 ssh key를 생성합니다.

```text
ssh-keygen
```

아래는 예제입니다.key 이름을 eksworkshop으로 생성하였습니다.

```text
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

### 2.ssh key 전송

생성된 SSH key를  Key 페어로 전송합니다. 앞서 eksworkshop.pub 로 public key가 생성되었습니다.

```text
aws ec2 import-key-pair --key-name "eksworkshop" --public-key-material file://~/environment/eksworkshop.pub

```

OpenSSH public key format 에러가 발생할 경우 아래와 같은 명령으로 Key 전송합니다.

```text
aws ec2 import-key-pair --key-name "eksworkshop" --public-key-material file://~/environment/eksworkshop.pub

```

오류가 발생할 경우에는 다음과 같은 방법으로 key 페어를 등록합니다.아래 명령의 Public key값을 복사합니다.

```text
cat ~/environment/eksworkshop.pub 
```

아래 ec2 대쉬보드에서 키를 등록합니다.

![](../.gitbook/assets/image%20%2891%29.png)

![](../.gitbook/assets/image%20%2892%29.png)

생성된 key를 "AWS 서비스" - "EC2 대시보드" 에서 확인합니다.

![](../.gitbook/assets/image%20%2819%29.png)

## CMK  생성

### KMS 소개

AWS KMS\(Key Management Service\)를 사용하면 손쉽게 암호화 키를 생성 및 관리하고 다양한 AWS 서비스와 애플리케이션에서의 사용을 제어할 수 있습니다. AWS KMS는 FIPS 140-2에 따라 검증되었거나 검증 과정에 있는 하드웨어 보안 모듈을 사용하여 키를 보호하는 안전하고 복원력 있는 서비스입니다. 또한, AWS KMS는 AWS CloudTrail과도 통합되어 모든 키 사용에 관한 로그를 제공함으로써 각종 규제 및 규정 준수 요구 사항을 충족할 수 있게 지원합니다. 

![](../.gitbook/assets/image%20%281%29.png)

EKS에서는 K8s와 Key를 통한 인증이 많이 일어납니다. 안전한 관리를 위해 Option으로 구성할 수 있습니다.

### 1.CMK 생성

K8s Secret 암호화를 할 때, EKS 클러스터에서 사용할 CMK를 생성합니다.

```text
aws kms create-alias --alias-name alias/eksworkshop --target-key-id $(aws kms create-key --query KeyMetadata.Arn --output text)

```

정상적으로 Key가 생성되었는지 **`AWS 관리 콘솔 - KMS - 고객관리형 키`**에서 확인합니다.

![](../.gitbook/assets/image%20%28142%29.png)

### 2. CMK ARN 변수 저장

CMK의 ARN을 $MASTER\_ARN에 입력해 둡니다. 

```text
export MASTER_ARN=$(aws kms describe-key --key-id alias/eksworkshop --query KeyMetadata.Arn --output text)
```

MASTER\_ARN에 입력된 값을 조회해 봅니다.

```text
 echo $MASTER_ARN
```

출력 결과 예제 

```text
arn:aws:kms:ap-northeast-2:xxxxxxx:key/xxxxxxx
```

이 랩에서 KMS Key를 쉽게 참조 할 수 있도록 MASTER 환경변수를 bash\_profile에 저장합니다.

```text
echo "export MASTER_ARN=${MASTER_ARN}" | tee -a ~/.bash_profile
```

출력 결과 예제

```text
whchoi98:~/environment $ echo $MASTER_ARN
arn:aws:kms:ap-northeast-2:909121566064:key/9a0c5a6c-be81-4463-90e4-e3b1252d96fc
whchoi98:~/environment $ echo "export MASTER_ARN=${MASTER_ARN}" | tee -a ~/.bash_profile
export MASTER_ARN=arn:aws:kms:ap-northeast-2:909121566064:key/9a0c5a6c-be81-4463-90e4-e3b1252d96fc
```

인증/자격증명 및 환경 구성 요약

1.Cloud9를 위한 IAM 역할\(Role\) 생성 및 IAM 역할 연경

**`AWS 서비스 - IAM - 역할 - 역할 만들기`**

**`역할만들기 - AWS서비스 - EC2선택`**

**`역할만들기 - 정책이름 - AdministratorAccess 선택`**

**`역할만들기 - 역할이름 - eksworkshop-admin 생성`**

**`AWS 서비스 - EC2 - 인스턴스 - "Cloud9 인스턴스 선택" - 작업 - 보안 - IAM 역할 수정`**

**`IAM 역할 수정 - IAM 역할 - "eksworkshop-admin 선택"`**

**`Cloud9 콘솔 - Preference - AWS Settings - Credentials - AWS managed temporary credentials 비활성`**

```text
#임시 자격 증명 삭제
rm -vf ${HOME}/.aws/credentials

```

2.Cloud 9 또는 기타장소에서 Key 생성

```text
#key 생성 , Key name - eksworkshop
ssh-keygen

```

3. Public Key를 AWS 키페어 집합소에 전송

```text
aws ec2 import-key-pair --key-name "eksworkshop" --public-key-material file://~/environment/eksworkshop.pub
```

4. KMS 기반 key 생성

```text
#KMS Key 생성
aws kms create-alias --alias-name alias/eksworkshop --target-key-id $(aws kms create-key --query KeyMetadata.Arn --output text)

#CMK의 ARN을 $MASTER_ARN에 입력
export MASTER_ARN=$(aws kms describe-key --key-id alias/eksworkshop --query KeyMetadata.Arn --output text)

#KMS Key를 쉽게 참조 할 수 있도록 MASTER 환경변수를 bash_profile에 저장
echo "export MASTER_ARN=${MASTER_ARN}" | tee -a ~/.bash_profile

```





