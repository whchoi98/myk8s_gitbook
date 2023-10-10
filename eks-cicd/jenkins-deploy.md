---
description: 'Update : 2023-09-20'
---

# Jenkins 기반 CI

### Jenkins 소개  <a href="#ci-cd" id="ci-cd"></a>

[Jenkins](https://jenkins.io/)는 Java로 작성된 오픈 소스 지속적 통합 도구로서, 소프트웨어 개발을 위한 사용자 지정 통합 서비스를 제공합니다. 많은 개발 팀에서 사용하는 서버 기반 시스템입니다. 소프트웨어 개발 수명 주기(SDLC)를 가속화하고자 한다면 Jenkins를 사용해야 합니다. Jenkins를 사용하면 다양한 환경으로 빌드, 배포 및 테스트를 통합하고 개발 팀이 대기하는 시간을 줄일 수 있습니다. 그뿐만 아니라 지속적으로 통합할 수 있으므로 Jenkins는 빠른 반복 주기를 사용하는 민첩한 방법론과 데브옵스에 매우 적합합니다. AWS는 Jenkins와 같은 애플리케이션을 실행하는 데 적합한 안정적이고 확장 가능하며 안전한 인프라 리소스를 제공합니다. AWS 컴퓨팅에서 Jenkins를 실행함으로써 사용한 만큼만 비용을 지불하고 특정 요구 사항에 맞춰 용량을 확장하거나 축소할 수 있습니다.

### Prerequisite <a href="#role" id="role"></a>

해당 모듈은 [Code Pipeline 기반 CI/CD 의 Repo 구성 절차](https://whchoi98.gitbook.io/k8s/eks-cicd/cicd-w-codepipeline#repo)가 완료 되어 있다는 것을 전제로 합니다.

### Elastic Container Registry 생성 <a href="#role" id="role"></a>

ECR 콘솔로 이동 합니다.

<figure><img src="../.gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>

새로운 리포지토리를 생성합니다.

* Name : jenkins-ci-test

<figure><img src="../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (11).png" alt=""><figcaption></figcaption></figure>

푸시 명령 보기 버튼을 클릭하여 나오는 명령어를 복사 해놓습니다.

<figure><img src="../.gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>

### Jenkins 설치 <a href="#role" id="role"></a>

#### Jenkins Node 생성 <a href="#2.-aws-auth-configmap" id="2.-aws-auth-configmap"></a>

* Node Name : eksworkshop-jenkins-01-Node
* Amazon Linux 2
* 보안그룹
  * Inbound
    * HTTP
    * TCP 8080&#x20;
* IAM 역할 : eksworkshop-admin
* Subnet - eksworkshop-PublicSubnet01 / 퍼블릭 IP 할당



#### Jenkins Node 세팅 <a href="#2.-aws-auth-configmap" id="2.-aws-auth-configmap"></a>

SSM을 사용해 Jenkins Node에 연결합니다. 아래의 명령어들을 실행합니다.

```sh
# add jenkins repo
sudo yum update –y
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum upgrade

# install java (amazon linux 2)
sudo yum upgrade -y
sudo amazon-linux-extras install java-openjdk11 -y

# install git
sudo yum install git -y

# install docker
sudo yum install docker -y
sudo systemctl start docker
sudo chmod 777 /var/run/docker.sock

# install jenkins
sudo yum install jenkins -y

# enable & start the jenkins service
sudo systemctl enable jenkins
sudo systemctl start jenkins

# check jenkins status
systemctl status jenkins
```

initialAdminPassword 확인하여 Copy 합니다.

```sh
cat /var/lib/jenkins/secrets/initialAdminPassword
```

eksworkshop-jenkins-01-Node 의 퍼블릭 IPv4 DNS 로 접속해 Jenkins Getting Started 페이지로 이동합니다.

<figure><img src="../.gitbook/assets/image (24).png" alt=""><figcaption></figcaption></figure>

이전 절차에서 복사해둔 initialAdminPassword 를 붙여넣습니다.

<figure><img src="../.gitbook/assets/image (23).png" alt=""><figcaption></figcaption></figure>

Install Suggested Plugins 클릭합니다.



새로운 관리자 계정을 생성 합니다.

<figure><img src="../.gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>

Jenkins URL을 설정 합니다. 해당 실습에서는 Jenkins Node 의 IPv4 도메인으로 설정합니다.

<figure><img src="../.gitbook/assets/image (27).png" alt=""><figcaption></figcaption></figure>

### Job 생성하기

새로운 Job 을 생성 합니다.

<figure><img src="../.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

jenkins-ci-test 이름의 Freestyle project를 생성 합니다.

<figure><img src="../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

Configure > Source Code Management 에서 Git 선택 후 Repository URL 입력 합니다.



Jenkins Credential Provider 에서 [Code Pipeline 기반 CI/CD 의 Repo 구성 절차](https://whchoi98.gitbook.io/k8s/eks-cicd/cicd-w-codepipeline#repo) 해두었던 Token 값을 Password에 입력합니다. Username은 GitHub 아이디를 입력합니다.

<figure><img src="../.gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

Build Triggers 를 Github hook trigger for GITScm polling 으로 설정 합니다.

<figure><img src="../.gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>

Build Steps의 Command에 ECR 절차에서 복사해두었던 푸시 명령어를 아래와 같이 복사합니다.

```
aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin 103787587884.dkr.ecr.ap-northeast-2.amazonaws.com
IMAGE_NAME="103787587884.dkr.ecr.ap-northeast-2.amazonaws.com/jenkins-ci-test:$BUILD_NUMBER"
docker build --tag $IMAGE_NAME .
docker push $IMAGE_NAME
```

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

Build Now로 Build가 잘 동작하는지 테스트 해봅니다.

<figure><img src="../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

Job에 의해 ECR 에 image가 push 되었는지 확인

<figure><img src="../.gitbook/assets/image (15).png" alt=""><figcaption></figcaption></figure>

### Webhook 연동하기

GitHub Repository에 Webhook을 설정합니다.

* Payload URL : `젠킨스주소/github-webhook/`

<figure><img src="../.gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>

GitHub 에디터 기능을 통해 소스코드(main.go)를 수정 합니다.

<figure><img src="../.gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (17).png" alt=""><figcaption></figcaption></figure>

Git Repository 에 commit 후 trigger에 의해 빌드가 되었는지 확인

<figure><img src="../.gitbook/assets/image (20).png" alt=""><figcaption></figcaption></figure>

### Kubectl 을 통한 배포과정 추가하기

Jenkins Node에서 kubectl 을 통해 EKS Cluster에 새롭게 빌드 된 이미지를 배포하기

Jenkins Node에 aws cli 업그레이드, kubectl 설치 및 kubeconfig 설정

```sh
# AWS CLI Upgrade
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
source ~/.bashrc
aws --version
# AWS CLI 자동완성 설치 
which aws_completer
export PATH=/usr/local/bin:$PATH
source ~/.bash_profile
complete -C '/usr/local/bin/aws_completer' aws

```

```sh
cd ~
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.23.17/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/bin/kubectl

```

```sh
aws eks update-kubeconfig --region ap-northeast-2 --name eksworkshop

# 확인
kubectl get nodes
```



#### Jenkins Web UI에서 Job 수정

Configure > Build Steps 에서 Add build step 클릭 후 Execute shell 클릭

<figure><img src="../.gitbook/assets/image (21).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

```
sed -i "s/CONTAINER_IMAGE/103787587884.dkr.ecr.ap-northeast-2.amazonaws.com\/jenkins-ci-test:$BUILD_NUMBER/" hello-k8s.yml
cat hello-k8s.yml
aws eks update-kubeconfig --region ap-northeast-2 --name eksworkshop
kubectl apply -f hello-k8s.yml
kubectl get all --selector=app=hello-k8s

```

github repository로 돌아가 다시 소스코드 수정 후 commit.

<figure><img src="../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

이미지 빌드 확인

<figure><img src="../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>



배포가 잘 적용 되었는지 확인합니다.

```
kubectl get all --selector=app=hello-k8s
```

```
TeamRole:~/environment/myeks (master) $ kubectl get all --selector=app=hello-k8s
NAME                             READY   STATUS    RESTARTS   AGE
pod/hello-k8s-56c474d7f5-48dfj   1/1     Running   0          47s
pod/hello-k8s-56c474d7f5-nwmvc   1/1     Running   0          47s
pod/hello-k8s-56c474d7f5-xmx7h   1/1     Running   0          47s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-k8s-56c474d7f5   3         3         3       47s
```
