---
description: 'Update : 2022-04-19'
---

# Code Pipeline기반 CI/CD

## CI/CD 소개&#x20;

CI([Continuous integration](https://aws.amazon.com/devops/continuous-integration/) 지속적 통합) 및 CD([continuous delivery](https://aws.amazon.com/devops/continuous-delivery/) 지속적 전달)는 민첩성을 요구하는  조직에 필수적입니다. 팀은 개별 변경을 자주 수행하고 프로그래밍 방식으로 해당 변경 사항을 릴리스하고 중단 없이 업데이트를 제공할 수 있을 때 생산성이 더 높아집니다.\
이 모듈에서는 AWS CodePipeline을 사용하여 CI/CD 파이프라인을 구축합니다. CI/CD 파이프라인은 샘플 Kubernetes 서비스를 배포하고 GitHub 리포지토리를 변경하고 이 변경 사항이 클러스터에 자동으로 전달되는 것을 확인합니다.

## Role 구성

### 1. IAM Role 생성&#x20;

AWS CodePipeline에서는 AWS CodeBuild를 사용하여 샘플 Kubernetes 서비스를 배포합니다. 이를 위해서는 EKS 클러스터와 상호 작용할 수 있는 AWS Identity and Access Management(IAM) 역할이 필요합니다.

이 단계에서는 IAM 역할을 생성하고, kubectl을 통해 EKS 클러스터와 상호 작용하기 위해 CodeBuild 단계에서 사용할 인라인 정책을 추가합니다.

IAM Role을 생성합니다.&#x20;

```
cd ~/environment
TRUST="{ \"Version\": \"2012-10-17\", \"Statement\": [ { \"Effect\": \"Allow\", \"Principal\": { \"AWS\": \"arn:aws:iam::${ACCOUNT_ID}:root\" }, \"Action\": \"sts:AssumeRole\" } ] }"
echo '{ "Version": "2012-10-17", "Statement": [ { "Effect": "Allow", "Action": "eks:Describe*", "Resource": "*" } ] }'
echo '{ "Version": "2012-10-17", "Statement": [ { "Effect": "Allow", "Action": "eks:Describe*", "Resource": "*" } ] }' > /tmp/iam-role-policy
aws iam create-role --role-name EksWorkshopCodeBuildKubectlRole --assume-role-policy-document "$TRUST" --output text --query 'Role.Arn'
aws iam put-role-policy --role-name EksWorkshopCodeBuildKubectlRole --policy-name eks-describe --policy-document file:///tmp/iam-role-policy

```

### 2. aws-auth configmap 적용

ConfigMap에 이 새 Role이 포함되면 파이프라인의 CodeBuild 단계에 있는 kubectl이 IAM 역할을 통해 EKS 클러스터와 상호 작용할 수 있습니다.

aws-auth configmap을 확인해 봅니다.&#x20;

```
kubectl get -n kube-system configmap/aws-auth -o yaml

```

IAM Role이 생성되었으므로 EKS 클러스터에 대한 aws-auth ConfigMap에 Role을 추가합니다.&#x20;

```
ROLE="    - rolearn: arn:aws:iam::${ACCOUNT_ID}:role/EksWorkshopCodeBuildKubectlRole\n      username: build\n      groups:\n        - system:masters"
kubectl get -n kube-system configmap/aws-auth -o yaml | awk "/mapRoles: \|/{print;print \"$ROLE\";next}1" > /tmp/aws-auth-patch.yml
kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"

```

\[참고] 아래와 같은 kubectl 명령으로 직접 수정할 수도 있습니다. (옵션)&#x20;

```
kubectl edit -n kube-system configmap/aws-auth

```

aws-auth configmap을 다시 확인해 보고 Role이 추가 되었는지 확인합니다 .&#x20;

```
kubectl get -n kube-system configmap/aws-auth -o yaml

```

결과 예

```
apiVersion: v1
data:
  mapRoles: |
    - rolearn: arn:aws:iam::011218731119:role/EksWorkshopCodeBuildKubectlRole
      username: build
      groups:
        - system:masters
```

## Repo 구성

### 3.Sample Repo 포크

리포지토리를 수정하고 빌드를 트리거할 수 있도록 샘플 Kubernetes 서비스를 분기합니다.&#x20;

GitHub에 로그인하고 샘플 서비스를 자신의 계정으로 분기합니다.

먼저 각자의 Github 계정으로 먼저 로그인하고, 아래 Repo로 이동합니다.&#x20;

Sample repo&#x20;

```
https://github.com/rnzsgh/eks-workshop-sample-api-service-go

```

아래와 같이 이동한 Repo에서 Fork 를 수행합니다.

![](<../.gitbook/assets/image (264).png>)

<figure><img src="../.gitbook/assets/image (109).png" alt=""><figcaption></figcaption></figure>

자신의 계정으로 분기된 Repo는 아래와 같습니다.&#x20;

![](<../.gitbook/assets/image (458).png>)

### 4. Github access token 구성&#x20;

CodePipeline이 GitHub에서 Callback을 수신하려면 개인 액세스 토큰을 생성해야 합니다.

일단 생성되면 액세스 토큰을 보안 영역에 저장하고 재사용할 수 있으므로 이 단계는 처음 실행하는 동안이나 새 키를 생성해야 할 때만 필요합니다.&#x20;

개인 Github 계정에서 "settings"를 선택합니다.&#x20;

Profile 메뉴 하단의 "Developer settings"를 선택합니다.&#x20;

* Note - **`"eks-workshop"`**
* repo를 선택합니다
* 하단의 Generate token을 선택합니다.&#x20;

<figure><img src="../.gitbook/assets/image (170).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (95).png" alt=""><figcaption></figcaption></figure>

![](<../.gitbook/assets/image (451).png>)

personal access token 생성을 확인하고, 복사해 둡니다.&#x20;

<figure><img src="../.gitbook/assets/image (105).png" alt=""><figcaption></figcaption></figure>

## CodePipeline 구성

### 5. Codepipeline 구성

AWS CloudFormation을 사용하여 AWS CodePipeline을 생성합니다.&#x20;

CloudFormation은 AWS 클라우드 환경의 모든 인프라 리소스를 설명하고 프로비저닝할 수 있는 공통 언어를 제공하는 코드형 인프라(IaC) 도구입니다. CloudFormation을 사용하면 간단한 텍스트 파일을 사용하여 모든 지역 및 계정에서 애플리케이션에 필요한 모든 리소스를 자동화되고 안전한 방식으로 모델링하고 프로비저닝할 수 있습니다.

각 EKS 배포/서비스에는 고유한 CodePipeline이 있어야 하며, 격리된 소스 리포지토리에 있어야 합니다.

이 워크샵과 함께 제공되는 CloudFormation 템플릿을 수정하고 시스템 요구 사항을 충족하여 EKS 클러스터에 새 서비스를 쉽게 적용 할 수 있습니다. 각각의 새로운 서비스에 대해 다음 단계를 반복할 수 있습니다.

**`Cloudformation - 스택생성 - 새 리소스 사용 (표준)`** 을 선택합니다.

아래 S3 URL을 입력합니다.&#x20;

```
https://s3.amazonaws.com/eksworkshop.com/templates/main/ci-cd-codepipeline.cfn.yml

```

![](<../.gitbook/assets/image (243).png>)

* 스택 이름 : eksworkshop-codepipeline
* Username : github 계정
* Access Token : Github Access token 값
* Repository : eks-workshop-sample-api-service-go
* Branch: master
* Codebuilde dockerimage : aws/codebuild/standard:4.0
* IAM kubectl IAM Role : EksWorkshopCodeBuildKubectlRole
* EKS cluster name: eksworkshop

![](<../.gitbook/assets/image (469).png>)

다음 단계를 계속 진행해서 , Cloudformation을 완료합니다.

### 6.Codepipeline 확인

관리 콘솔에서 CodePipeline을 엽니다.&#x20;

**`AWS 관리콘솔 - CodePipeline`**&#x20;

eks-workshop-codepipeline으로 시작하는 CodePipeline이 표시됩니다. 이 링크를 클릭하면 세부 정보를 볼 수 있습니다.

![](<../.gitbook/assets/image (174).png>)

Codepipeline에서 진행중이거나, 완료된 Pipeline을 선택하면 Build 상태를 확인 할 수 있습니다.&#x20;

![](<../.gitbook/assets/image (444).png>)

<figure><img src="../.gitbook/assets/image (104).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (115).png" alt=""><figcaption></figcaption></figure>

5분 정도 이후에, kubectl을 통해서 정상적인 배포가 이뤄졌는지 확인해 봅니다.&#x20;

```
kubectl describe deployment hello-k8s
kubectl describe service hello-k8s
kubectl get services hello-k8s -o wide

```

### 7. 신규 버전 배포&#x20;

Application의 신규 버전을 배포해 봅니다.소스 Github Repo에서 새롭게 변경하고, 배포를 합니다.&#x20;

개인 계정에서 fork된 Github의 소스에서 go 파일을 변경합니다.

```
https://github.com/개인계정/eks-workshop-sample-api-service-go

```

![](<../.gitbook/assets/image (271).png>)

![](<../.gitbook/assets/image (74).png>)



![](<../.gitbook/assets/image (57).png>)

Commit Changes에서 Commit을 합니다.&#x20;

![](<../.gitbook/assets/image (322).png>)

GitHub에서 변경 사항을 수정하고 커밋하면 약 1분 안에 AWS Management Console CodePipeline에서 트리거된 새 빌드가 실행되는 것을 볼 수 있습니다.

![](<../.gitbook/assets/image (276).png>)

Build 상태를 통해서 , 빌드의 상세사항을 확인 할 수 있습니다.&#x20;

![](<../.gitbook/assets/image (166).png>)

kubectl 명령을 통해 Service URL을 확인하고, 업데이트 현황을 확인합니다.

```
kubectl get services hello-k8s -o wide

```

아래는 변경된 메세지가 포함된 결과입니다.&#x20;

![](<../.gitbook/assets/image (345).png>)
