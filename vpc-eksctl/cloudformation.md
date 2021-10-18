---
description: 'update : 2021-09-24 / 20min'
---

# Cloudformation 구성

## Cloudformation 소개

AWS CloudFormation에서는 클라우드 환경에서 AWS 및 타사 애플리케이션 리소스를 모델링하고 프로비저닝할 수 있도록 공용 언어를 제공합니다. AWS CloudFormation을 사용하면 프로그래밍 언어 또는 간단한 텍스트 파일을 사용하여 자동화되고 안전한 방식으로 모든 지역과 계정에 걸쳐 애플리케이션에 필요한 모든 리소스를 모델링 및 프로비저닝할 수 있습니다. 이를 통해 AWS 및 타사 리소스의 단일 소스를 제공합니다.

## Cloudformation 기반의 VPC 구성

### 1.VPC yaml 다운로드

eksworkshop에서 사용할 다양한 yaml file을 git에서 내려받습니다. Cloud9에서 아래와 같이 실행합니다.

```
cd ~/environment
git clone https://github.com/whchoi98/myeks

```

### 2. Stack 생성

Cloud9에서 앞서 다운로드 받은 git에서 EKSVPC3AZ.yml 파일을 로컬 PC에 다운로드 합니다. EKS 환경 구성을 위한 VPC 구성용 Cloudformation 입니다.

![](<../.gitbook/assets/image (198).png>)

Cloud9에서 직접 file을 업로드하기 위해서는 아래와 같이 S3를 활용할 수도 있습니다.

```
#S3 Bucket 생성합니다. 
#Bucket name은 고유해야 합니다.
aws s3 mb s3://{bucket-name}
#생성한 S3 Bucket으로 파일을 모두 복사해 둡니다.
cd ./myeks/

# Cloud9에서 변경되는 파일을 S3와 동기화 합니다. 
aws s3 sync ./ s3://{bucket-name}

##option - copy를 통해 사용해도 가능합니다.
aws s3 cp ./ s3://{bucket-name} --recursive

## LAB에서 사용할 Object접근을 허용합니다.
aws s3api put-object-acl --bucket {bucket name} --key EKSVPC3AZ.yml --acl public-read  
```

AWS 서비스 - Cloudformation 을 선택합니다. 

"스택생성" - "새 리소스 사용"을 선택합니다.

![](<../.gitbook/assets/image (41).png>)

앞서 다운로드 받은 "**EKSVPC3AZ.yml**" 파일을 업로드 합니다.

![](<../.gitbook/assets/image (221) (1).png>)

{% hint style="info" %}
yml 파일을 로컬 PC에서 업로드 불가능한 경우에는 , S3 URL을 선택하고 업로드 합니다.
{% endhint %}

S3 URL은 생성한 버킷 이름과 리전 주소, Object 로 생성되어 있습니다.

```
https://{bucket_name}.s3.ap-northeast-2.amazonaws.com/EKSVPC3AZ.yml
```

![](<../.gitbook/assets/image (220) (1).png>)



### 3. Stack 상세 정보 구성

Cloudformation Stack 상세 정보 구성을 합니다.

![](<../.gitbook/assets/image (163).png>)

![](<../.gitbook/assets/image (23).png>)

![](<../.gitbook/assets/image (7).png>)

생성이 완료되었는지 확인합니다.

![](<../.gitbook/assets/image (32).png>)

### 4.Output 정보 확인

Cloudformation을 통해 생성된 VPC의 자원들을 기반으로, eksctl 을 사용해서 EKS Cluster를 사용할 것입니다.

이때 필요한 것이, VPC id, Subnet id 입니다. 출력이 되면 subnet id는 public subnet 01,02,03, private subnet 01,02,03으로 출력됩니다. 

이 값을 확인해 봅니다. (다음 단원에서 aws cli 통해서 Cloud9 인스턴스 홈 디렉토리에 결과값을 txt 파일로 저장할 것입니다.)

![](<../.gitbook/assets/image (160).png>)

구성이 완료되면 다음과 같은 VPC가 구성됩니다.

![](<../.gitbook/assets/image (221).png>)

## Cloudformation 기반 VPC 구성 요약

#### 1.EKS workshop을 위한 File Download

```
cd ~/environment
git clone https://github.com/whchoi98/myeks

```

#### 2. Cloudformation 기반 배포

**`AWS 서비스 - Cloudformation`**

**`스택생성 - 새 리소스 사용`**

**`스택생성 - 템플릿 파일 업로드`**

**`파일선택 - EKSVPCxAZ.yml 선택`**

**`출력정보 확인`**
