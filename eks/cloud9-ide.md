---
description: 'update : 2025-01-25 /10min'
---

# Cloud9 IDE 환경 구성

## CodeServer 설치 <a href="#id-1.vscode" id="id-1.vscode"></a>

### 1. VSCode 서버 설치 <a href="#id-1.vscode" id="id-1.vscode"></a>

**VS Code Server**는 Microsoft의 Visual Studio Code 편집기를 클라우드 환경 또는 원격 서버에서 실행할 수 있도록 설계된 소프트웨어입니다. 이는 개발자가 로컬 머신에 설치하지 않고도 어디서나 웹 브라우저를 통해 VS Code의 기능을 사용할 수 있게 해줍니다. 특히, 자원을 많이 소비하는 작업을 원격 서버에서 처리하거나, 팀이 협업할 때 동일한 개발 환경을 제공하는 데 유용합니다.

**주요 특징**

1. **원격 개발 환경**: VS Code Server를 사용하면 로컬 머신의 성능에 의존하지 않고도 원격 서버의 자원을 활용하여 개발 작업을 수행할 수 있습니다. 이는 특히 대규모 데이터 처리나 컴파일 작업이 필요한 경우 유용합니다.
2. **웹 기반 접근**: 사용자는 웹 브라우저를 통해 어디서나 VS Code Server에 접근할 수 있습니다. 이를 통해 다양한 기기에서 동일한 개발 환경을 유지할 수 있습니다.
3. **플러그인 지원**: 로컬 VS Code와 마찬가지로, 다양한 플러그인과 확장 기능을 설치하여 개발 환경을 확장할 수 있습니다.
4. **보안**: 비밀번호 보호 및 HTTPS 지원을 통해 원격 개발 환경의 보안을 강화할 수 있습니다. 사용자는 비밀번호를 설정하여 무단 접근을 방지할 수 있으며, SSL 인증서를 사용하여 통신을 암호화할 수 있습니다.
5. **협업**: 여러 개발자가 동시에 동일한 프로젝트에 접근하여 작업할 수 있어, 팀 협업에 매우 유리합니다.

**사용 사례**

* **클라우드 개발 환경**: 클라우드 상의 인프라를 활용하여 개발 환경을 구축하고, 이를 통해 다양한 작업을 수행할 수 있습니다.
* **학습 및 교육**: 교육 기관이나 코딩 부트캠프에서 일관된 개발 환경을 제공하여 학습 효율성을 높일 수 있습니다.
* **원격 근무**: 개발자들이 물리적 위치에 관계없이 동일한 개발 환경에서 작업할 수 있도록 지원합니다.

**설치 및 설정**

VS Code Server는 다양한 방법으로 설치할 수 있으며, 일반적으로 다음과 같은 단계를 따릅니다:

1. **서버 환경 준비**: 원격 서버에 VS Code Server를 설치할 준비를 합니다.
2. **다운로드 및 설치**: 공식 GitHub 릴리즈 페이지에서 최신 버전을 다운로드하고 설치합니다.
3. **설정 파일 구성**: `config.yaml` 파일을 생성하여 서버 설정을 구성합니다.
4. **서비스 관리**: `systemd`와 같은 서비스 관리 도구를 사용하여 VS Code Server를 시작하고, 부팅 시 자동으로 시작되도록 설정합니다.

VS Code Server는 개발자들이 원격 환경에서 편리하게 코딩하고 협업할 수 있도록 도와주는 강력한 도구입니다. 이를 통해 개발 프로세스의 유연성과 효율성을 높일 수 있습니다.

VSCode를 실행하기 위해 아래와 같이 AWS 관리콘솔에서 **`"Cloudshell"`** 을 사용해서 구성합니다.

![](https://whchoi98.gitbook.io/~gitbook/image?url=https%3A%2F%2F406222686-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-x-prod.appspot.com%2Fo%2Fspaces%252F4LeNGzsjFqHe9dSZbKSC%252Fuploads%252FImTjHxkRhGp8Llrsi40c%252Fimage.png%3Falt%3Dmedia%26token%3D30ba72e8-eb1a-46cb-a9ed-3c2ec19ed9db\&width=768\&dpr=4\&quality=100\&sign=279540cc\&sv=2)

`아래와 같이 iam user와 패스워드, user를 위한 Policy를 생성해서 연결합니다.`

```
export user_name=user01
export pass_word=1234Qwer
aws iam create-user --user-name ${user_name}
aws iam create-login-profile --user-name ${user_name} --password ${pass_word} --no-password-reset-required
aws iam attach-user-policy --user-name ${user_name} --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

제공된 AWS 계정에 손쉽게 접근하기 위해서 Alisa를 생성합니다. Alias는 고유해야 하므로 , 중복되지 않도록 합니다.

```
aws iam create-account-alias --account-alias ${alias-name}
```

앞서 생성한 Alias로 접속하고, 새로운 User로 인증해서 로그인 합니다.

정상적으로 접속하면, 다시 CloudShell을 사용해서 , VSCode Server를 구성합니다.

```
# 아래 git을 cloudshell에 복제합니다.
git clone https://github.com/whchoi98/ec2_vscode.git
```

### 2.VSCode Server 구성 <a href="#id-2.vscode-server" id="id-2.vscode-server"></a>

CloudShell에서 VSCode 서버를 구성하기 위한 환경 변수를 정의합니다. VSCode Server는 Default VPC, Public Subnet에 설치합니다.

```
source ~/.bash_profile
export AWS_REGION=ap-northeast-2
~/ec2_vscode/defaultvpcid.sh
source ~/.bashrc
```

Cloudshell에서 Cloudformation 을 배포해서 VSCode 서버를 구성합니다.

```
aws cloudformation deploy \
  --template-file "~/ec2_vscode/ec2vscode.yaml" \
  --stack-name=ec2vscodeserver \
  --parameter-overrides \
    InstanceType=t3.xlarge \
    AMIType=AmazonLinux2023 \
    DefaultVPCId=$DEFAULT_VPC_ID \
    PublicSubnetId=$PUBLIC_SUBNET_ID \
  --capabilities CAPABILITY_NAMED_IAM

```

10분 이후 VSCode 서버가 생성됩니다.

cloudshell 에서 아래 Shell을 실행시켜, vscode server의 Public IP를 확인하고, VSCode에 접속합니다.

```
# AWS_REGION 변수를 설정하고 현재 쉘 세션에서 사용 가능하게 합니다.
export AWS_REGION=ap-northeast-2
# bashrc 파일에 AWS_REGION 변수를 영구적으로 추가하여, 모든 새로운 쉘 세션에서도 사용 가능하게 합니다.
echo 'export AWS_REGION=ap-northeast-2' >> ~/.bashrc
~/ec2_vscode/vscode_ip.sh
```

아래는 출력예제 입니다.

```
$ ~/ec2_vscode/vscode_ip.sh 
EC2VSCodeServer = 15.165.36.130
CodeServer Connect = 15.165.36.130:8888
```

해당 VSCodeServer 공인 IP 주소로 접속합니다.

```
EC2SVSCodeServer:8888
```

EC2가 완전하게 배포된 후 3\~5분 뒤에 브라우저에서 EC2VSCodeServer PublicIP:8080으로 접속합니다.

![](https://whchoi98.gitbook.io/~gitbook/image?url=https%3A%2F%2F406222686-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-x-prod.appspot.com%2Fo%2Fspaces%252F4LeNGzsjFqHe9dSZbKSC%252Fuploads%252F9CjCxSocTtf1FsxMF9WT%252Fimage.png%3Falt%3Dmedia%26token%3D9a511f30-7643-48d7-80ca-902b2864ec77\&width=768\&dpr=4\&quality=100\&sign=c9023ed7\&sv=2)

EC2VSCodeServer Terminal에서 아래를 실행합니다.

```
git clone https://github.com/whchoi98/ec2_vscode.git
```

VSCodeServer는 패스워드 설정이 되어 있지 않습니다.

비밀번호 설정을 확인하고, 적절한 비밀번호로 변경한 후 다음 스크립트를 실행합니다.

```
cat ~/ec2_vscode/vscode_pwd.sh

~/ec2_vscode/vscode_pwd.sh
```

패스워드를 입력하고, 다시 브라우저를 재접속합니다.,

이 랩의 모든 콘솔은 VSCode Server 터미널에서 수행합니다.

## AWS CLI 설치

### 3. Code-server에 패키지 설치

아래와 같이 Cloud9 에 필요한 도구들을 설치합니다.

```bash
mkdir ~/environment/
cd ~/environment/
git clone https://github.com/whchoi98/useful-shell.git
~/environment/useful-shell/c9-tool-set.sh

```

다음과 같은 도구들을 설치했습니다.

* AWS CLI 최신 버전
* Session Manager
* jq , gettext, bash-comletion

## 4.기타 도구 설치

아래 Shell 명령을 실행해서, 이 LAB에 필요한 도구들을 설치합니다.

```
~/environment/useful-shell/eks_tools.sh
source ~/.bash_profile

```

* kubectl - 쿠버네티스 커맨드 라인 도구인 [kubectl](https://kubernetes.io/docs/user-guide/kubectl/)을 사용하면, 쿠버네티스 클러스터에 대해 명령을 실행할 수 있습니. kubectl을 사용하여 애플리케이션을 배포하고, 클러스터 리소스를 검사 및 관리하며 로그를 볼 수 있다습니다. kubectl 작업의 전체 목록에 대해서는, [kubectl 개요](https://kubernetes.io/ko/docs/reference/kubectl/overview/)를 참고합니다.\
  (참조 - [https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html))\
  이 LAB에서는 kubectl 바이너리 버전은 1.29.10을 설치합니다.&#x20;

```
# 최신버전 다운로드
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
# 특정 버전을 다운로드하려면, $(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt) 명령 부분을 특정 버전으로 변경합니다.
```

:dart: 추가 참조 URL - [https://docs.aws.amazon.com/ko\_kr/eks/latest/userguide/install-kubectl.html](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/install-kubectl.html)

* eksctl - eksctl은 Amazon EKS (Elastic Kubernetes Service) 클러스터를 손쉽게 생성, 관리, 및 삭제할 수 있도록 설계된 CLI 도구입니다. eksctl은 Kubernetes 클러스터를 AWS에서 구축하고 운영하는 데 필요한 대부분의 작업을 자동화하며, Kubernetes 사용자의 생산성을 크게 향상시켜 줍니다. &#x20;
* K9s - K9s는 Kubernetes 클러스터를 관리하고 모니터링하기 위한 터미널 기반의 UI 도구입니다. Kubernetes 리소스와 상호작용하기 위해 설계되었으며, kubectl 명령어의 대안으로 사용할 수 있는 효율적이고 직관적인 CLI 인터페이스를 제공합니다.(참조 - [https://github.com/derailed/k9s](https://github.com/derailed/k9s))
* jq - jq는 JSON 데이터 처리 및 조작을 위해 설계된 경량 명령줄 도구입니다. JSON 데이터를 파싱, 필터링, 변환, 포맷팅, 그리고 분석하는 데 유용하며, 특히 스크립트나 터미널 기반 워크플로에서 활용도가 높습니다.jq는 단순하면서도 강력한 도구로, 특히 JSON 데이터 처리와 조작이 필요한 모든 상황에서 큰 도움을 줍니다. REST API 응답 분석, 로그 처리, 클라우드 서비스와의 연동 작업 등 다양한 작업에서 꼭 필요한 필수 도구입니다.
* kube krew - Krew는 Kubernetes 커맨드라인 도구인 kubectl의 플러그인을 쉽게 검색, 설치, 관리할 수 있도록 해주는 플러그인 매니저입니다. Krew를 사용하면 Kubernetes 클러스터와 상호작용하기 위해 다양한 플러그인을 간단한 명령어로 설치하고 관리할 수 있습니다.Krew는 Kubernetes 작업의 효율성을 극대화할 수 있는 필수 도구입니다. kubectl 명령어를 더욱 강력하고 확장성 있게 만들어주는 도구이므로, Kubernetes를 사용하는 모든 사용자에게 적극 추천합니다
* kube ctx - kubectl ctx 또는 **kube-ctx**는 Kubernetes 클러스터와 컨텍스트/네임스페이스를 빠르게 전환할 수 있도록 설계된 경량 명령줄 도구입니다. Kubernetes는 다중 클러스터와 컨텍스트를 관리할 수 있는 기능을 제공하며, kube-ctx는 이를 더욱 쉽게 관리할 수 있도록 도와줍니다.

## **EKS 환경 구성 요약**

1.Code Server 인스턴스 활성화 및 콘솔 접속

2.AWS CLI 2.0 업그레이드 및 자동완성 설치

3\. Code Server에 Kubectl 설치 (1.29.10기준)

4\. 기타 유틸리티 설치

***

{% hint style="info" %}
AWS Cloud9 서비스는 2025년 12월 31일자로 서비스 종료될 예정이라는 발표가 있었습니다. 기존 사용자는 해당 날짜 이전까지 Cloud9 IDE에서 작업을 마치고 필요한 데이터를 백업하거나 다른 환경으로 이전해야 합니다.
{% endhint %}

### Cloud9 소개

AWS Cloud9은 브라우저만으로 코드를 작성, 실행 및 디버깅할 수 있는 클라우드 기반 IDE(통합 개발 환경)입니다. 코드 편집기, 디버거 및 터미널이 포함되어 있습니다. Cloud9은 JavaScript, Python, PHP를 비롯하여 널리 사용되는 프로그래밍 언어를 위한 필수 도구가 사전에 패키징되어 제공되므로, 새로운 프로젝트를 시작하기 위해 파일을 설치하거나 개발 머신을 구성할 필요가 없습니다. Cloud9 IDE는 클라우드 기반이므로, 인터넷이 연결된 머신을 사용하여 사무실, 집 또는 어디서든 프로젝트 작업을 할 수 있습니다. 또한, Cloud9은 서버리스 애플리케이션을 개발할 수 있는 원활한 환경을 제공하므로 손쉽게 서버리스 애플리케이션의 리소스를 정의하고, 디버깅하고, 로컬 실행과 원격 실행 간에 전환할 수 있습니다. Cloud9에서는 개발 환경을 팀과 신속하게 공유할 수 있으므로 프로그램을 연결하고 서로의 입력 값을 실시간으로 추적할 수 있습니다.&#x20;

### 1.Cloudshell 기반 환경 준비

Cloud9을 실행하기 위해 아래와 같이 AWS 관리콘솔에서 **`"Cloudshell"`** 을 사용해서 구성합니다.

<figure><img src="../.gitbook/assets/image (2) (1) (1).png" alt=""><figcaption></figcaption></figure>

아래와 같이 iam user와 패스워드, user를 위한 Policy를 생성해서 연결합니다.

```
export user_name=user01
export password=1234Qwer
aws iam create-user --user-name ${user_name}
aws iam create-login-profile --user-name ${user_name} --password ${password} --no-password-reset-required
aws iam attach-user-policy --user-name ${user_name} --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

제공된 AWS 계정에 손쉽게 접근하기  위해서 Alisa를 생성합니다. Alias는 고유해야 하므로 , 중복되지 않도록 합니다.

```
aws iam create-account-alias --account-alias ${alias-name}
```

앞서 생성한 Alias로 접속하고, 새로운 User로 인증해서 로그인 합니다.

### 2. Cloud9 IDE 환경 설정

정상적으로 접속하면, 다시 CloudShell을 사용해서 , Cloud9을 구성합니다.

```
# 아래 git을 cloudshell에 복제합니다.
git clone https://github.com/whchoi98/useful-shell.git

# cloud9 을 생성합니다.2~3분 시간이 소요됩니다.
~/useful-shell/create-cloud9.sh

```



{% hint style="info" %}
Cloud9 하단의 설정 메뉴 중에 Network Setting은 변경하지 않으면, 자동으로 VPC Default로 설정되며 Cloud9 인스턴스는 해당 Default VPC의 public subnet에 자동으로 설치됩니다.
{% endhint %}

### 3. Cloud9 터미널 접속

Cloud9 터미널에 접속하여, EKS Workshop 터미널  IDE 환경을 살펴봅니다.

<figure><img src="../.gitbook/assets/image (429).png" alt=""><figcaption></figcaption></figure>

![](<../.gitbook/assets/image (277).png>)

## AWS CLI 설치

### 4. Cloud9 에 패키지 설치

아래와 같이 Cloud9 에 필요한 도구들을 설치합니다.

```
git clone https://github.com/whchoi98/useful-shell.git
~/environment/useful-shell/c9-tool-set.sh

```

다음과 같은 도구들을 설치했습니다.

* AWS CLI 최신 버전
* Session Manager
* jq , gettext, bash-comletion

## 기타 도구 설치

아래 Shell 명령을 실행해서, 이 LAB에 필요한 도구들을 설치합니다.

```
~/environment/useful-shell/eks_tools.sh
source ~/.bash_profile

```

* kubectl - 쿠버네티스 커맨드 라인 도구인 [kubectl](https://kubernetes.io/docs/user-guide/kubectl/)을 사용하면, 쿠버네티스 클러스터에 대해 명령을 실행할 수 있습니. kubectl을 사용하여 애플리케이션을 배포하고, 클러스터 리소스를 검사 및 관리하며 로그를 볼 수 있다습니다. kubectl 작업의 전체 목록에 대해서는, [kubectl 개요](https://kubernetes.io/ko/docs/reference/kubectl/overview/)를 참고합니다.\
  (참조 - [https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html))\
  이 LAB에서는 kubectl 바이너리 버전은 1.27.13 설치합니다.&#x20;

```
# 최신버전 다운로드
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
# 특정 버전을 다운로드하려면, $(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt) 명령 부분을 특정 버전으로 변경합니다.
```

:dart: 추가 참조 URL - [https://docs.aws.amazon.com/ko\_kr/eks/latest/userguide/install-kubectl.html](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/install-kubectl.html)

* eksctl - `eksctl`은 관리형 Kubernetes 서비스 인 EKS에서 클러스터를 생성하기위한 간단한 CLI 도구입니다. Go로 작성되었으며 CloudFormation을 사용하며 [Weaveworks](https://www.weave.works/) 가 작성했으며 단 하나의 명령으로 몇 분 안에 기본 클러스터를 만듭니다. 이것은 EKS를 구성하기 위한 도구 이며,  AWS 관리콘솔에서 제공하는 EKS UI, CDK, Terraform, Rancher 등 다양한 도구로도 구성이 가능합니다.
* K9s - K9s는 쿠버네티스 클러스터와 상호작용을 통해 직관적인 UI 터미널을 제공합니다. 이 도구를 통해서 쿠버네티스 자원들을 쉽게 탐색하고 관리할 수 있도록 도움을 줍니다.(참조 - [https://github.com/derailed/k9s](https://github.com/derailed/k9s))
* jq - **jq**는 커맨드라인에서 JSON을 조작할 수 있는 도구입니다. 프로그래밍 언어는 아니지만 JSON 데이터를 다루기 위한 다양한 기능들을 제공합니다. kubectl, aws cli의 결과들 중에서 복잡한 중첩 JSON구조  내에서 키를 찾을 때 유용합니다.
* kube krew - kube krew는 Mac OS brew, CentOS yum, Ubuntu apt 처럼 Kube에 관련된 좋은 유틸리티를 제공하고 있습니다.
* kube ctx - kubectx는 다중의 Kubecluster 가 존재할 때 전환이 쉽도록 도와주는 훌륭한 도구입니다. kubectx를 설치합니다.

## **EKS 환경 구성 요약**

1.AWS Cloud 9 인스턴스 활성화 및 콘솔 접속

2.AWS CLI 2.0 업그레이드 및 자동완성 설치

3\. Cloud9에 Kubectl 설치 (1.27.13기준)

4\. 기타 유틸리티 설치
