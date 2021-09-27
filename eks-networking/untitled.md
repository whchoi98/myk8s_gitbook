---
description: 'Update : 2021-09-27'
---

# Multus

## Multus 소개 

[Multus](https://github.com/Intel-Corp/multus-cni)는 쿠버네티스의 CRD 기반 네트워크 오브젝트를 사용하여 쿠버네티스에서 멀티 네트워킹 기능을 지원하는 멀티 CNI 플러그인입니다.

Multus는 CNI 명세를 구현하는 모든 [레퍼런스 플러그인](https://github.com/containernetworking/plugins) 및 3rd Party 플러그인을 지원합니. 또한, Multus는 쿠버네티스의 클라우드 네이티브 애플리케이션과 NFV 기반 애플리케이션을 통해 쿠버네티스의 [SRIOV](https://github.com/hustcat/sriov-cni), [DPDK](https://github.com/Intel-Corp/sriov-cni), [OVS-DPDK 및 VPP](https://github.com/intel/vhost-user-net-plugin) 워크로드를 지원합니다.

Multus는 Pod에 멀 네트워크 인터페이스를 첨부할 수 있는 Kubernetes용 오픈 소스 CNI 플러그인입니다. Multus는 메타 플러그인을 기반으로 Pod에 첨부된 다중 네트워크 인터페이스를 작동하는 추가 CNI 플러그인을 지원합니다. 다중 인터페이스를 포함한 Pod가 일반적으로 필요한 사용 사례에는 Kubernetes에 대한 5G 및 스트리밍 네크워크 등이 있습니다. EKS가 지원하는 Multus를 사용하여 이러한 환경에 걸쳐 어드밴스드 네트워킹을 활성화함으로써 사용자에게 고품질 콘텐츠를 전달하는 컨테이너화 네트워크 기능을 실행할 수 있습니다.

랩에서는 아래와 같이 US-WEST-2 \(오레곤\) 리전에서 EKS 기반으로  Multus를 적용하는 방안을 소개 합니다. 

## Multus 구성을 위한 사전 준비

### Task1. Cloud9 구성

AWS 서비스에서 Cloud9을 선택하고, `"Environments"`를 설정합니다.

Cloud9 의 이름과 Description을 설정합니다.

![](../.gitbook/assets/image%20%28209%29.png)

인스턴스 타입과 운영체제, 그리고 절전모드 환경을 선택합니다. 절전모드 환경은 기본 30분입니다. 아래와 같이 변경합니다.

* Instance type : m5.large
* cost-saving : Never

![](../.gitbook/assets/image%20%28208%29.png)

### Task2. SSH Key 구성 및 패키지 설치

[EKS 환경구성](../eks/)에서 적용한 방식과 동일하게 적용합니다. 먼저 Cloud9 IDE 터미널에서 ssh key를 생성합니다.

```text
#ssh key를 생성합니다.
cd ~/envvironment
ssh-keygen

```

key는 "eksworkshop"으로 구성합니다.

```text
eksworkshop

```

pem 을 설정하고, 권한을 부여합니다.

```text
cd ~/envvironment
mv ./eksworkshop ./eksworkshop.pem
chmod 400 ./eksworkshop.pem

```

아래와 같이 aws cli v2.x 로 업그레이드 하고, kubelet 을 설치합니다.

```text
#AWS CLI를 2.0으로 업그레이드합니다.
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

#AWS CLI 자동완성을 설치합니다.
which aws_completer
export PATH=/usr/local/bin:$PATH
source ~/.bash_profile
complete -C '/usr/local/bin/aws_completer' aws

# EKS 1.19.6 기반 설치
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

```text
#multus 구성을 위한 git clone을 실행합니다.
cd ~/envvironment
git clone https://github.com/aws-samples/eks-install-guide-for-multus.git

# US-WEST-2에 S3 Bucket을 생성합니다. Bucket Name은 고유해야 합니다.
cd ~/envvironment
aws s3 mb s3://{bucket name} --region us-west-2

# 생성된 Bucket에 git을 업로드 합니다.
cd ~/envvironment
aws s3 sync ./eks-install-guide-for-multus s3://{bucket name}
aws s3 ls s3://{bucket name}/cfn/templates/infra/
aws s3 ls s3://{bucket name}/cfn/templates/nodegroup/

# object가 외부에서 접근할 수 있도록 , Read 권한을 부여합니다.
aws s3api put-object-acl --bucket {bucket name} --key cfn/templates/infra/eks-infra.yaml --acl public-read  
aws s3api put-object-acl --bucket {bucket name} --key cfn/templates/nodegroup/eks-nodegroup-multus.yaml --acl public-read
```

Cloudfomration 기반 배포

![](../.gitbook/assets/image%20%28205%29.png)

![](../.gitbook/assets/image%20%28204%29.png)

![](../.gitbook/assets/image%20%28207%29.png)

* Stack Name : eks-multus-cluster
* Availability Zone : us-west-2a, us-west-2b
* Bastion Keyname : eksworkshop

Task4. EKS Infra 배포



Task5. EKS multus nodegroup 배포

Multus 



  


