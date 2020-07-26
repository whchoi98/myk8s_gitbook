# aws cli

## aws cli 설치 및 업그레이드.

### 1.리눅스 \(Fedora 기준\)

aws cli 2 설치

```text
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

```

aws cli 1에서 cli version 2로 업그레이드

```text
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

```

명령어 자동 완성

```text
which aws_completer
export PATH=/usr/local/bin:$PATH
source ~/.bash_profile
complete -C '/usr/local/bin/aws_completer' aws

```

### 

## 인증 및 계정 관련 aws cli . 

### account id 출력 

```text
aws sts get-caller-identity --output text --query Account
curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.accountId'
```

### Key 전송

```text
aws ec2 import-key-pair --key-name "public key name" --public-key-material file://"key path"
```

### IAM 정책 생성

```text
aws iam create-policy \
   --policy-name ALBIngressControllerIAMPolicy \
   --policy-document https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/${ALB_INGRESS_VERSION}/docs/examples/iam-policy.json
```

## 기본 정보 출력.

### Arn 출력

```text
aws sts get-caller-identity --output text --query Arn
```

### Instance Region 정보

```text
curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region'
```

### Instance AZ 정보

```text
curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.availabilityZone'
```

### VPC ID  출력

```text
aws ec2 describe-vpcs --filters Name=tag:Name,Values=eksworkshop | jq -r '.Vpcs[].VpcId'

```

### Subnet 출력

```text
aws ec2 describe-subnets  --filters "Name=cidr-block,Values=10.11.*" --query 'Subnets[*].[CidrBlock,SubnetId,AvailabilityZone]' --output table
```

