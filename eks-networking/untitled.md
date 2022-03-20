---
description: 'Update : 2021-09-27'
---

# Multus

## Multus 소개&#x20;

[Multus](https://github.com/Intel-Corp/multus-cni)는 쿠버네티스의 CRD 기반 네트워크 오브젝트를 사용하여 쿠버네티스에서 멀티 네트워킹 기능을 지원하는 멀티 CNI 플러그인입니다.

Multus는 CNI 명세를 구현하는 모든 [레퍼런스 플러그인](https://github.com/containernetworking/plugins) 및 3rd Party 플러그인을 지원합니. 또한, Multus는 쿠버네티스의 클라우드 네이티브 애플리케이션과 NFV 기반 애플리케이션을 통해 쿠버네티스의 [SRIOV](https://github.com/hustcat/sriov-cni), [DPDK](https://github.com/Intel-Corp/sriov-cni), [OVS-DPDK 및 VPP](https://github.com/intel/vhost-user-net-plugin) 워크로드를 지원합니다.

Multus는 Pod에 멀 네트워크 인터페이스를 첨부할 수 있는 Kubernetes용 오픈 소스 CNI 플러그인입니다. Multus는 메타 플러그인을 기반으로 Pod에 첨부된 다중 네트워크 인터페이스를 작동하는 추가 CNI 플러그인을 지원합니다. 다중 인터페이스를 포함한 Pod가 일반적으로 필요한 사용 사례에는 Kubernetes에 대한 5G 및 스트리밍 네크워크 등이 있습니다. EKS가 지원하는 Multus를 사용하여 이러한 환경에 걸쳐 어드밴스드 네트워킹을 활성화함으로써 사용자에게 고품질 콘텐츠를 전달하는 컨테이너화 네트워크 기능을 실행할 수 있습니다.

## Multus 구성을 위한 사전 준비

### Task1. Cloud9 구성

AWS 서비스에서 Cloud9을 선택하고, `"Environments"`를 설정합니다.

Cloud9 의 이름과 Description을 설정합니다.

![](<../.gitbook/assets/image (211).png>)

인스턴스 타입과 운영체제, 그리고 절전모드 환경을 선택합니다. 절전모드 환경은 기본 30분입니다. 아래와 같이 변경합니다.

* Instance type : m5.large
* cost-saving : Never

![](<../.gitbook/assets/image (210).png>)

### Task2. SSH Key 구성 및 패키지 설치

[EKS 환경구성](../eks/)에서 적용한 방식과 동일하게 적용합니다. 먼저 Cloud9 IDE 터미널에서 ssh key를 생성합니다.

```
#ssh key를 생성합니다.
cd ~/envvironment
ssh-keygen

```

key는 "eksworkshop"으로 구성합니다.

```
eksworkshop

```

pem 을 설정하고, 권한을 부여합니다.

```
cd ~/envvironment
mv ./eksworkshop ./eksworkshop.pem
chmod 400 ./eksworkshop.pem

```

아래와 같이 aws cli v2.x 로 업그레이드 하고, kubelet 을 설치합니다.

```
#AWS CLI를 2.0으로 업그레이드합니다.
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

#AWS CLI 자동완성을 설치합니다.
which aws_completer
export PATH=/usr/local/bin:$PATH
source ~/.bash_profile
complete -C '/usr/local/bin/aws_completer' aws

# EKS 1.21.2 기반 설치
cd ~
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.21.2/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc

# Kubectl 자동완성 설치
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

```

Task3. Multus Git Clone

```
#multus 구성을 위한 git clone을 실행합니다.
cd ~/envvironment
git clone https://github.com/aws-samples/eks-install-guide-for-multus.git

# US-WEST-2에 S3 Bucket을 생성합니다. Bucket Name은 고유해야 합니다.
cd ~/envvironment
export bucket_name=whchoimultus
aws s3 mb s3://$bucket_name

# 생성된 Bucket에 git을 업로드 합니다.
cd ~/envvironment
aws s3 sync ./eks-install-guide-for-multus s3://$bucket_name
aws s3 ls s3://$bucket_name/cfn/templates/infra/
aws s3 ls s3://$bucket_name/cfn/templates/nodegroup/

# object가 외부에서 접근할 수 있도록 , Read 권한을 부여합니다.
# aws s3api put-object-acl --bucket {bucket name} --key cfn/templates/infra/eks-infra.yaml --acl public-read  
# aws s3api put-object-acl --bucket {bucket name} --key cfn/templates/nodegroup/eks-nodegroup-multus.yaml --acl public-read
```

## Cloudfomration 기반 배포

### Task4. EKS Infra 배포



![](<../.gitbook/assets/image (227).png>)

**`CloudFormation - 스택 - 스택생성`**  을 선택합니다. 앞서 복사해 둔 eks-infra.yaml 의 Object URL을 Cloudformation S3 URL에 입력하고, 스택을 배포합니다.

![](<../.gitbook/assets/image (226).png>)

![](<../.gitbook/assets/image (225).png>)

![](<../.gitbook/assets/image (219).png>)

* Stack Name : eks-multus-cluster
* Availability Zone : ap-northeast-2a, ap-northeast-2b
* PublicSubnetAz1Cidr : 10.0.0.0/24
* PublicSubnetAz2Cidr: 10.0.1.0/24
* PrivateSubnetAz1Cidr: 10.0.2.0/24
* PrivateSubnetAz2Cidr: 10.0.3.0/24
* MultusSubnet1Az1Cidr: 10.0.4.0/24
* MultusSubnet1Az2Cidr: 10.0.5.0/24
* MultusSubnet2Az1Cidr: 10.0.6.0/24
* MultusSubnet2Az2Cidr: 10.0.7.0/24
* Bastion Keyname : eksworkshop
*

### Task5. EKS multus nodegroup 배포

EKS nodegroup을 배포하기 위해 , Lambda function을 S3에 업로드합니다. Lambda function은 앞서 multus git에서 다운로드 하였습니다.

```
# US-WEST-2에 S3 Bucket을 생성합니다. Bucket Name은 고유해야 합니다.
aws s3 mb s3://{bucket name} --region us-west-2

# 생성된 Bucket에 lambda function을 업로드 합니다.
cd ~/envvironment
aws s3 cp  ~/environment/eks-install-guide-for-multus/cfn/templates/nodegroup/lambda_function.zip s3://whchoi-multus-lambda  

# object가 외부에서 접근할 수 있도록 , Read 권한을 부여합니다.
aws s3api put-object-acl --bucket {bucket name} --key lambda_function.zip --acl public-read  

```

S3에 업로드한 EKS Nodegroup용 Cloudformation Stack yaml 파일의 Object URL을 복사해 둡니다.

![](<../.gitbook/assets/image (207).png>)

Cloudformation 에서 새로운 Stack을 배포합니다.

**`CloudFormation - 스택 - 스택생성`**  을 선택합니다. 앞서 복사해 둔 eks-nodegroup-multus.yaml 의 Object URL을 Cloudformation S3 URL에 입력하고, 스택을 배포합니다.

![](<../.gitbook/assets/image (209).png>)

![](<../.gitbook/assets/image (212).png>)

* Stack Name : ng1
* Cluster Name : eks-multus-cluster
* ClusterControlPlaneSecurityGroup - eks-multus-cluster-EksControlSecurityGroup-xxxx
* NodeGroupName : ng1
* Min/Desired/MaxSize : 1&#x20;
* KeyName : eksworkshop
* VpcId : vpc-eks-multus-cluster
* Subnets : privateAz1-eks-multus-cluster (main primary K8s networking network 입니다.)
* MultusSubnets : multus1Az1 , Multus2Az1
* MultusSecurityGroups : multus-Sg-eks-multus-cluster
* LambdaS3Bucket : 앞서 생성한 Bucket Name (예. whchoi-multus-lambda)
* LambdaS3Key : lambda\_function.zip&#x20;

Bastion Host 접

![](<../.gitbook/assets/image (218).png>)



![](<../.gitbook/assets/image (224).png>)

Multus&#x20;



\
