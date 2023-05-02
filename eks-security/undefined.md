# 이미지 보안

## ECR Advanced Scanner

## 1. ECR 환경 설정

Container Image 보안 관련 실습을 진행하기에 앞서 ECR 에서 제공하는 Image Scanning 기능과 관련하여 환경 설정을 진행하도록 하겠습니다. 아래의 명령을 실행하여 ECR 의 Scanning 과 관련할 설정을 변경하도록 합니다. 아래의 설정은 ECR Repository 에 Image 가 Push 되면 Enhanced Scanning Mode 를 사용하도록하며 Scanning 의 대상이 되는 Image 를 Filter(prod)로 제한하는 내용을 담고 있습니다.

```
aws ecr put-registry-scanning-configuration \
--scan-type ENHANCED \
--rules '[{"repositoryFilters" : [{"filter":"prod","filterType" : "WILDCARD"}],"scanFrequency" : "CONTINUOUS_SCAN"}]' \
--region $AWS_REGION

```

AWS Management Console 의 [ECR 메뉴 ](https://ap-northeast-2.console.aws.amazon.com/ecr/private-registry/edit-scanning?region=ap-northeast-2)에서 아래와 같이 설정이 반영되어 있는 것을 확인합니다.



<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

### 2. ECR 생성

ECR 환경 설정이 정상적으로 이뤄졌다면 이제 Container Image 와 관련된 각종 실습을 진행하기 위하여 ECR Repository 를 생성하도록 하겠습니다.

아래의 명령을 이용하여 3개의 ECR Repository 를 생성하도록 하겠습니다. 각 Repository 는 Production, Sandbox, Shared Image 용 Repository 로 각각 사용될 예정입니다.

```
aws ecr create-repository --repository-name eks-security-prod
aws ecr create-repository --repository-name eks-security-sandbox
aws ecr create-repository --repository-name eks-security-shared

```

\


