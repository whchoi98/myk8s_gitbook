---
description: 'Update : 2023-09-21'
---

# 이미지 보안

## ECR Advanced Scanner

## 1. ECR 환경 설정

Container Image 보안 관련 실습을 진행하기에 앞서 ECR 에서 제공하는 Image Scanning 기능과 관련하여 환경 설정을 진행합니다. 아래의 명령을 실행하여 ECR 의 Scanning 과 관련할 설정을 변경하도록 합니다.&#x20;

아래의 설정은 ECR Repository 에 Image 가 Push 되면 Enhanced Scanning Mode 를 사용하도록하며 Scanning 의 대상이 되는 Image 를 Filter(prod)로 제한하는 내용을 담고 있습니다.

```
aws ecr put-registry-scanning-configuration \
--scan-type ENHANCED \
--rules '[{"repositoryFilters" : [{"filter":"prod","filterType" : "WILDCARD"}],"scanFrequency" : "CONTINUOUS_SCAN"}]' \
--region $AWS_REGION

```

AWS Management Console 의 [ECR 메뉴 ](https://ap-northeast-2.console.aws.amazon.com/ecr/private-registry/edit-scanning?region=ap-northeast-2)에서 아래와 같이 설정이 반영되어 있는 것을 확인합니다.



<figure><img src="../.gitbook/assets/image (315).png" alt=""><figcaption></figcaption></figure>

## 2. ECR 생성

ECR 환경 설정이 정상적으로 이뤄졌다면 이제 Container Image 와 관련된 각종 실습을 진행하기 위하여 ECR Repository 를 생성하도록 하겠습니다.

아래의 명령을 이용하여 3개의 ECR Repository 를 생성하도록 하겠습니다. 각 Repository 는 Production, Sandbox, Shared Image 용 Repository 로 각각 사용될 예정입니다.

```
aws ecr create-repository --repository-name eks-security-prod
aws ecr create-repository --repository-name eks-security-sandbox
aws ecr create-repository --repository-name eks-security-shared

```

\
ECR Repository 가 정상적으로 생성되었는지 확인합니다.

<figure><img src="../.gitbook/assets/image (89).png" alt=""><figcaption></figcaption></figure>

## 3. Enhanced Scanning

단계 1에서 설정한 ECR Registry 의 Enhanced Scanning 기능을 확인하기 위하여 Container Image 를 Push 하여 해당 Container Image 의 취약점을 진단하는지 확인하는 과정을 실습합니다. 먼저, ECR Repository 에 Push 할 Container Image 를 Cloud9 으로 Pull 합니다. 아래의 명령을 이용하여 Log4j 취약점을 포함하고 있는 Container Image 를 Pull 한 후 ECR Repository 에 Push 하기 위하여 Tag 를 설정하도록 합니다.

{% hint style="warning" %}
주의 !!! 실습에 사용한 Container Image 는 취약점을 포함하고 있으므로 절대 실제 운영환경에서는 사용하면 안됩니다.
{% endhint %}

```
docker pull public.ecr.aws/docker/library/neo4j:4.4.0
docker tag public.ecr.aws/docker/library/neo4j:4.4.0 $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/eks-security-prod
docker tag public.ecr.aws/docker/library/neo4j:4.4.0 $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/eks-security-sandbox

```

Container Image 가 정상적으로 다운로드되었다면 ECR Repository 에 Push 하기 위하여 아래와 같이 AWS CLI 를 이용하여 Login 하도록 합니다.

```
aws ecr get-login-password --region $AWS_REGION | \
docker login --username AWS --password-stdin \
$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

```

정상적으로 로그인이 되면 아래와 같은 결과를 확인 할 수 있습니다.

```
## ECR Login 예제입니다. ##
$ aws ecr get-login-password --region $AWS_REGION | \
> docker login --username AWS --password-stdin \
> $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
WARNING! Your password will be stored unencrypted in /home/ec2-user/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded

```

ECR Registry 에 Login 이 성공하였다면 아래의 명령을 이용하여 조금 전 다운로드한 Container Image 를 두 개의 Repository 에 Push 하도록 하겠습니다.

```
docker push $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/eks-security-prod
docker push $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/eks-security-sandbox

```

Cloud9 콘솔에서 Image 가 정상적으로 Push 되었다면 다음 단계를 진행합니다.



## 4. 스캔 결과 확인

ECR Repository 에 Image 가 Push 되면 ECR 자체적으로 취약점 Scanning 을 시작합니다. 그리고 그 결과는 각 Image 가 Push 된 Repository 를 통해 확인할 수 있습니다 [AWS Management Console 의 ECR 메뉴 ](https://ap-northeast-2.console.aws.amazon.com/ecr/repositories?region=ap-northeast-2) 에 접속하여 현재 생성되어 있는 3개의 Repository 중 "eks-security-prod" 와 "eks-security-sandbox" Repository 에 각각 조금 전 Push 한 Container Image 가 Upload 되어 있는 것을 확인합니다.

"eks-security-sandbox" Repository 의 경우 아래와 같이 "Scanning" 이 꺼져있는 것을 확인할 수 있습니다. 즉, "eks-security-sandbox" Repository 에 Push 되는 Container Image 들은 취약점 Scanning 이 적용되지 않는다는 것을 알 수 있습니다.

<figure><img src="../.gitbook/assets/image (334).png" alt=""><figcaption></figcaption></figure>

"eks-security-prod" Repository 의 경우에는 아래와 같이 "See Findings" 링크가 있는 것을 확인할 수 있습니다. 해당 링크를 클릭하여 탐지된 취약점 리스트를 확인하도록 하겠습니다.

<figure><img src="../.gitbook/assets/image (100).png" alt=""><figcaption></figcaption></figure>

정상적인 경우 아래와 같이 Enhanced Scanning 기능을 통해 해당 Container Image 에 포함되어 있는 "Log4j" 취약점을 포함한 다양한 취약점들이 탐지된 것을 확인할 수 있습니다.\


<figure><img src="../.gitbook/assets/image (120).png" alt=""><figcaption></figcaption></figure>

ECR Repository 의 Enhanced Scanning 은 Amazon Inspector 와 통합되어 있는 기능으로 ECR Repository 뿐만 아니라 [AWS Management Console 의 Inspector 메뉴 ](https://ap-northeast-2.console.aws.amazon.com/inspector/v2/home?region=ap-northeast-2#/dashboard) 에서도 확인이 가능합니다. [Amazon Inspector 메뉴 ](https://ap-northeast-2.console.aws.amazon.com/inspector/v2/home?region=ap-northeast-2#/dashboard)에 접속하여 아래와 같이 "eks-security-prod" Repository 에 대한 취약점 정보를 확인하도록 합니다.\


<figure><img src="../.gitbook/assets/image (126).png" alt=""><figcaption></figcaption></figure>
