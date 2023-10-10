---
description: 'update : 2022-09-10 / 15min'
---

# Cloudformation 구성

## Cloudformation 소개

AWS CloudFormation에서는 클라우드 환경에서 AWS 및 타사 애플리케이션 리소스를 모델링하고 프로비저닝할 수 있도록 공용 언어를 제공합니다. AWS CloudFormation을 사용하면 프로그래밍 언어 또는 간단한 텍스트 파일을 사용하여 자동화되고 안전한 방식으로 모든 지역과 계정에 걸쳐 애플리케이션에 필요한 모든 리소스를 모델링 및 프로비저닝할 수 있습니다. 이를 통해 AWS 및 타사 리소스의 단일 소스를 제공합니다.

## Cloudformation 기반의 VPC 구성

{% hint style="info" %}
생성되는 VPC에서는 NAT Gateway가 3개 사용됩니다. NAT Gateway는 3개의 EIP를 사용합니다. 사용 중인 계정에 EIP 할당 숫자는 최대 5개 입니다. EIP가 3개 이상 여유가 있어야 배포가 가능합니다.&#x20;
{% endhint %}

### 1.VPC yaml 다운로드

eksworkshop에서 사용할 다양한 yaml file을 git에서 내려받습니다. Cloud9에서 아래와 같이 실행합니다.

```
cd ~/environment
git clone https://github.com/whchoi98/myeks
git clone https://github.com/whchoi98/useful-shell

```

### 2. Stack 생성

아래 aws cli를 통해서 Cloudformation을 실행하여 , VPC를 구성합니다.&#x20;

```
cd ./myeks/
aws cloudformation deploy \
  --stack-name "eksworkshop" \
  --template-file "EKSVPC3AZ.yml" \
  --capabilities CAPABILITY_NAMED_IAM 
  
```

### 3. Stack 완료 확인

Cloudformation Stack 이 완료된 것을 확인합니다.

![](<../.gitbook/assets/image (370).png>)

### 4.Output 정보 확인

Cloudformation을 통해 생성된 VPC의 자원들을 기반으로, eksctl 을 사용해서 EKS Cluster를 사용할 것입니다.

이때 필요한 것이, VPC id, Subnet id 입니다. 출력이 되면 subnet id는 public subnet 01,02,03, private subnet 01,02,03으로 출력됩니다.&#x20;

이 값을 확인해 봅니다. (다음 단원에서 aws cli 통해서 Cloud9 인스턴스 홈 디렉토리에 결과값을 txt 파일로 저장할 것입니다.)

![](<../.gitbook/assets/image (144).png>)

구성이 완료되면 다음과 같은 VPC가 구성됩니다.

![](<../.gitbook/assets/image (416).png>)

