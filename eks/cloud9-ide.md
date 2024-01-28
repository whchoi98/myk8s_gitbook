---
description: 'update : 2023-05-06 /15min'
---

# Cloud9 IDE 환경 구성

## Cloud9 환경 구성

### Cloud9 소개

AWS Cloud9은 브라우저만으로 코드를 작성, 실행 및 디버깅할 수 있는 클라우드 기반 IDE(통합 개발 환경)입니다. 코드 편집기, 디버거 및 터미널이 포함되어 있습니다. Cloud9은 JavaScript, Python, PHP를 비롯하여 널리 사용되는 프로그래밍 언어를 위한 필수 도구가 사전에 패키징되어 제공되므로, 새로운 프로젝트를 시작하기 위해 파일을 설치하거나 개발 머신을 구성할 필요가 없습니다. Cloud9 IDE는 클라우드 기반이므로, 인터넷이 연결된 머신을 사용하여 사무실, 집 또는 어디서든 프로젝트 작업을 할 수 있습니다. 또한, Cloud9은 서버리스 애플리케이션을 개발할 수 있는 원활한 환경을 제공하므로 손쉽게 서버리스 애플리케이션의 리소스를 정의하고, 디버깅하고, 로컬 실행과 원격 실행 간에 전환할 수 있습니다. Cloud9에서는 개발 환경을 팀과 신속하게 공유할 수 있으므로 프로그램을 연결하고 서로의 입력 값을 실시간으로 추적할 수 있습니다.&#x20;

### 1.Cloudshell 기반 환경 준비

Cloud9을 실행하기 위해 아래와 같이 AWS 관리콘솔에서 **`"Cloudshell"`** 을 사용해서 구성합니다.

<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

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
  이 LAB에서는 kubectl 바이너리 버전은 1.25.12 설치합니다.&#x20;

```
# 최신버전 다운로드
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
# 특정 버전을 다운로드하려면, $(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt) 명령 부분을 특정 버전으로 변경합니다.
```

:dart: 추가 참조 URL - [https://docs.aws.amazon.com/ko\_kr/eks/latest/userguide/install-kubectl.html](https://docs.aws.amazon.com/ko\_kr/eks/latest/userguide/install-kubectl.html)

* eksctl - `eksctl`은 관리형 Kubernetes 서비스 인 EKS에서 클러스터를 생성하기위한 간단한 CLI 도구입니다. Go로 작성되었으며 CloudFormation을 사용하며 [Weaveworks](https://www.weave.works/) 가 작성했으며 단 하나의 명령으로 몇 분 안에 기본 클러스터를 만듭니다. 이것은 EKS를 구성하기 위한 도구 이며,  AWS 관리콘솔에서 제공하는 EKS UI, CDK, Terraform, Rancher 등 다양한 도구로도 구성이 가능합니다.
* K9s - K9s는 쿠버네티스 클러스터와 상호작용을 통해 직관적인 UI 터미널을 제공합니다. 이 도구를 통해서 쿠버네티스 자원들을 쉽게 탐색하고 관리할 수 있도록 도움을 줍니다.(참조 - [https://github.com/derailed/k9s](https://github.com/derailed/k9s))
* jq - **jq**는 커맨드라인에서 JSON을 조작할 수 있는 도구입니다. 프로그래밍 언어는 아니지만 JSON 데이터를 다루기 위한 다양한 기능들을 제공합니다. kubectl, aws cli의 결과들 중에서 복잡한 중첩 JSON구조  내에서 키를 찾을 때 유용합니다.
* kube krew - kube krew는 Mac OS brew, CentOS yum, Ubuntu apt 처럼 Kube에 관련된 좋은 유틸리티를 제공하고 있습니다.
* kube ctx - kubectx는 다중의 Kubecluster 가 존재할 때 전환이 쉽도록 도와주는 훌륭한 도구입니다. kubectx를 설치합니다.

## **EKS 환경 구성 요약**

1.AWS Cloud 9 인스턴스 활성화 및 콘솔 접속

2.AWS CLI 2.0 업그레이드 및 자동완성 설치

3\. Cloud9에 Kubectl 설치 (1.21.5기준)

4\. 기타 유틸리티 설치
