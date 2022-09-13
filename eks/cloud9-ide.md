---
description: 'update : 2022-05-07 /15min'
---

# Cloud9 IDE 환경 구성

## Cloud9 환경 구성

### Cloud9 소개

AWS Cloud9은 브라우저만으로 코드를 작성, 실행 및 디버깅할 수 있는 클라우드 기반 IDE(통합 개발 환경)입니다. 코드 편집기, 디버거 및 터미널이 포함되어 있습니다. Cloud9은 JavaScript, Python, PHP를 비롯하여 널리 사용되는 프로그래밍 언어를 위한 필수 도구가 사전에 패키징되어 제공되므로, 새로운 프로젝트를 시작하기 위해 파일을 설치하거나 개발 머신을 구성할 필요가 없습니다. Cloud9 IDE는 클라우드 기반이므로, 인터넷이 연결된 머신을 사용하여 사무실, 집 또는 어디서든 프로젝트 작업을 할 수 있습니다. 또한, Cloud9은 서버리스 애플리케이션을 개발할 수 있는 원활한 환경을 제공하므로 손쉽게 서버리스 애플리케이션의 리소스를 정의하고, 디버깅하고, 로컬 실행과 원격 실행 간에 전환할 수 있습니다. Cloud9에서는 개발 환경을 팀과 신속하게 공유할 수 있으므로 프로그램을 연결하고 서로의 입력 값을 실시간으로 추적할 수 있습니다.&#x20;

### 1. Cloud9 IDE 환경 설정

AWS 서비스에서 Cloud9을 선택하고, `"Environments"`를 설정합니다.

Cloud9 의 이름과 Description을 설정합니다.

![](https://firebasestorage.googleapis.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F-MB-iH\_e37nRZq7Rkf7x%2Fuploads%2F277HscdTA0yGf6A28s9S%2Ffile.png?alt=media)

### 2. Cloud9 인스턴스 타입과 플랫폼 설정

인스턴스 타입과 운영체제, 그리고 절전모드 환경을 선택합니다. 절전모드 환경은 기본 30분입니다.

![](<../.gitbook/assets/image (144).png>)

{% hint style="info" %}
Cloud9 하단의 설정 메뉴 중에 Network Setting은 변경하지 않으면, 자동으로 VPC Default로 설정되며 Cloud9 인스턴스는 해당 Default VPC의 public subnet에 자동으로 설치됩니다.
{% endhint %}

### 3. Cloud9 터미널 접속

Cloud9 터미널에 접속하여, EKS Workshop 터미널  IDE 환경을 살펴봅니다.

![](<../.gitbook/assets/image (10).png>)

### 4. Cloud9 Volume 증설

Cloud9은 생성될 때 기본 10GB의 EBS 볼륨이 생성됩니다. 아래 Script를 실행해서 Cloud9의 볼륨을 100GB로 늘려 줍니다.

```
pip3 install --user --upgrade boto3
export instance_id=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
python -c "import boto3
import os
from botocore.exceptions import ClientError 
ec2 = boto3.client('ec2')
volume_info = ec2.describe_volumes(
    Filters=[
        {
            'Name': 'attachment.instance-id',
            'Values': [
                os.getenv('instance_id')
            ]
        }
    ]
)
volume_id = volume_info['Volumes'][0]['VolumeId']
try:
    resize = ec2.modify_volume(    
            VolumeId=volume_id,    
            Size=100
    )
    print(resize)
except ClientError as e:
    if e.response['Error']['Code'] == 'InvalidParameterValue':
        print('ERROR MESSAGE: {}'.format(e))"
if [ $? -eq 0 ]; then
    sudo reboot
fi



```

## AWS CLI 설치

### 5.AWS CLI 버전 확인과 업그레이드

{% hint style="info" %}
AWS 명령줄 인터페이스(CLI)는 AWS 서비스를 관리하는 통합 도구입니다. 도구 하나만 다운로드하여 구성하면 여러 AWS 서비스를 명령줄에서 제어하고 스크립트를 통해 자동화할 수 있습니다.
{% endhint %}

Cloud9 IDE는 이미 AWS CLI가 설치되어 있습니다. 하지만 기본 1.x 버전이 설치되어 있습니다.

```
$ aws --version
aws-cli/1.19.39 Python/2.7.18 Linux/4.14.225-169.362.amzn2.x86_64 botocore/1.20.39
```

아래 명령을 통해 CLI를 2.0으로 업그레이드합니다.

```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

```

정상적으로 업그레이드 되었는지 확인합니다.

```
source ~/.bashrc
aws --version

```

aws cli 자동완성을 설치 합니다.

```
which aws_completer
export PATH=/usr/local/bin:$PATH
source ~/.bash_profile
complete -C '/usr/local/bin/aws_completer' aws

```

### 6. AWS Session Manager Plugin 설치

Cloud9 Terminal에 Hands On Lab에서 Session Manager 를 통해 EKS Worker Node 접속을 위해 아래와 같이 설치합니다.

```
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/linux_64bit/session-manager-plugin.rpm" -o "session-manager-plugin.rpm"
sudo sudo yum install -y session-manager-plugin.rpm

```

## Kubectl 설치

### 7.Kubectl 소개

쿠버네티스 커맨드 라인 도구인 [kubectl](https://kubernetes.io/docs/user-guide/kubectl/)을 사용하면, 쿠버네티스 클러스터에 대해 명령을 실행할 수 있습니. kubectl을 사용하여 애플리케이션을 배포하고, 클러스터 리소스를 검사 및 관리하며 로그를 볼 수 있다습니다. kubectl 작업의 전체 목록에 대해서는, [kubectl 개요](https://kubernetes.io/ko/docs/reference/kubectl/overview/)를 참고합니다.

### 8.kubectl 바이너리 다운로드

EKS를 위한 kubectl 바이너리를 다운로드합니다. 아래 kubectl version 가운데 1개를 다운로드 받습니다.

(참조 - [https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html))

{% hint style="info" %}
Kubernetes 버전 1.23 출시부터 공식적으로 Amazon EKS AMI에는 containerd가 유일한 런타임으로 포함됩니다. Kubernetes 버전 1.18–1.21은 Docker를 기본 런타임으로 사용합니다.
{% endhint %}

**EKS 1.19.15 기반 설치 (Option)**

```
cd ~
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.19.15/bin/linux/amd64/kubectl

```

**EKS 1.20.7 기반 설치 (Option)**

```
cd ~
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.20.7/bin/linux/amd64/kubectl

```

**EKS 1.21.5 기반 설치 (2022.05 기준 추천)**

```
cd ~
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.21.5/bin/linux/amd64/kubectl

```

#### EKS 1.22.6 (Option)

```
cd ~
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.22.6/bin/linux/amd64/kubectl

```

#### Kubectl 버전 다운로드 (Linux 기준)

```
# 최신버전 다운로드
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
# 특정 버전을 다운로드하려면, $(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt) 명령 부분을 특정 버전으로 바꾼다.
```

#### :dart: 추가 참조 URL - [https://docs.aws.amazon.com/ko\_kr/eks/latest/userguide/install-kubectl.html](https://docs.aws.amazon.com/ko\_kr/eks/latest/userguide/install-kubectl.html)

### 9. 실행권한을 적용 및 구성&#x20;

kubectl 설치 후, 바이너리에 실행권한을 적용합니다.

```
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl

```

### 10. kubectl 설치 확인 &#x20;

```
kubectl version --short --client

```

출력결과 예제 (1.20.7 예시)

```
Client Version: v1.21.5

```

kubectl 자동완성을 설치합니다.&#x20;

{% hint style="info" %}
kubectl은 Bash 및 Zsh에 대한 자동 완성 지원을 제공하므로 입력을 위한 타이핑을 많이 절약할 수 있습니다.
{% endhint %}

```
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

```



## 기타 유틸리티 설치

### 11.GNU gettext,jq,bash 자동완성, moreutil 설치

```
sudo yum -y install jq gettext bash-completion moreutils

```

### 12.jq 구성

```
for command in kubectl jq envsubst aws
  do
    which $command &>/dev/null && echo "$command in path" || echo "$command NOT FOUND"
  done
  
```

{% hint style="info" %}
**jq**는 커맨드라인에서 JSON을 조작할 수 있는 도구입니다. 프로그래밍 언어는 아니지만 JSON 데이터를 다루기 위한 다양한 기능들을 제공합니다. kubectl의 결과들 중에서 복잡한 중첩 JSON구조  내에서 키를 찾을 때 유용합니다.
{% endhint %}

### 13.K9s 설치

K9s는 쿠버네티스 클러스터와 상호작용을 통해 직관적인 UI 터미널을 제공합니다. 이 도구를 통해서 쿠버네티스 자원들을 쉽게 탐색하고 관리할 수 있도록 도움을 줍니다.(참조 - [https://github.com/derailed/k9s](https://github.com/derailed/k9s))

K9s 설치를 위해 Linux brew 를 설치합니다.&#x20;

해당 패키지를 설치하기 위해 필수 패키지를 먼저 설치합니다.

```
sudo yum groupinstall -y 'Development Tools' && sudo yum install curl file git ruby which

```

Linux Brew를 설치합니다.

```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/Linuxbrew/install/master/install.sh)"

#Press Control-D to install to /home/ec2-user/.linuxbrew
```

경로를 설정합니다.

```
echo 'eval "$(/home/ec2-user/.linuxbrew/bin/brew shellenv)"' >> /home/ec2-user/.bash_profile
eval "$(/home/ec2-user/.linuxbrew/bin/brew shellenv)"

```

Brew 기반의 K9s를 설치합니다.

```
brew install derailed/k9s/k9s

```

설치가 완료되면 정상 작동하는지 확인합니다.

```
k9s

```

### 14.Kube krew 설치 (option)

{% hint style="info" %}
kube krew는 Mac OS brew, CentOS yum, Ubuntu apt 처럼 Kube에 관련된 좋은 유틸리티를 제공하고 있습니다.
{% endhint %}

Kube krew를 설치합니다.

```
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)

```

Kube krew경로를 설정합니다.

```
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
source ~/.bashrc

```

### 15.Kubectx 설치 (Option)

kubectx는 다중의 Kubecluster 가 존재할 때 전환이 쉽도록 도와주는 훌륭한 도구입니다. kubectx를 설치합니다.

```
kubectl krew install ctx
## 앞서 설치된 brew를 통해서 설치도 가능합니다
## brew install kubectx
```

### 16.Kubens 설치 (Option)

kubens는 여러개의 namespace를 전환이 쉽도록 도와주는 도구 입니다. kubens를 설치합니다.

```
kubectl krew install ns

```

### 17.Kubetree 설치 (Option)

kubetree는 linux의 tree처럼 kube의  파일구조를 확인하는 데 유용한 도구입니다.

```
kubectl krew install tree

```

### 18. Kube PS1 설치 (Option)

kube-ps1은 Prompt 상에 현재 Cluster와 Namespace 위치를 알려줍니다.&#x20;

```
brew install kube-ps1
source "$(brew --prefix)/opt/kube-ps1/share/kube-ps1.sh"
PS1='$(kube_ps1)'$PS1

```

## **EKS 환경 구성 요약**

1.AWS Cloud 9 인스턴스 활성화 및 콘솔 접속

2.AWS CLI 2.0 업그레이드 및 자동완성 설치

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

# Session Manager Plugin 설치 
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/linux_64bit/session-manager-plugin.rpm" -o "session-manager-plugin.rpm"
sudo sudo yum install -y session-manager-plugin.rpm

```

3\. Cloud9에 Kubectl 설치 (1.21.5기준)

```
# EKS 1.21.5 기반 설치
cd ~
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.21.5/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc

# Kubectl 자동완성 설치
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

```

4\. 기타 유틸리티 설치

```
#GNU gettext,jq,bash 자동완성, moreutil 설치
sudo yum -y install jq gettext bash-completion moreutils tree

#yq 구성
for command in kubectl jq envsubst aws
  do
    which $command &>/dev/null && echo "$command in path" || echo "$command NOT FOUND"
  done
  
#K9s 설치
sudo yum groupinstall -y 'Development Tools' && sudo yum install curl file git ruby which
sh -c "$(curl -fsSL https://raw.githubusercontent.com/Linuxbrew/install/master/install.sh)"
echo 'eval "$(/home/ec2-user/.linuxbrew/bin/brew shellenv)"' >> /home/ec2-user/.bash_profile
eval "$(/home/ec2-user/.linuxbrew/bin/brew shellenv)"
brew install derailed/k9s/k9s

#Kube krew 설치
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
source ~/.bashrc

#kubectx 설치
kubectl krew install ctx

#kubens 설치
kubectl krew install ns

#kubetree 설치
kubectl krew install tree

```
