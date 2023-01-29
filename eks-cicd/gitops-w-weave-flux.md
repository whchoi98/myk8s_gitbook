# WEAVE Flux 기반 GitOps (TBD)

## WEAVE Flux 기반 GitOps 구성

### 1.사전 준비

AWS CodePipeline과 AWS CodeBuild 를 모두 사용합니다. AWS 자격 증명 및 액세스 관리(IAM)Docker 이미지 빌드 파이프라인을 생성하기 위한 Role을 구성해야합니다.&#x20;

IAM Role을 생성하고 CodeBuild 단계에서 kubectl을 통해 EKS 클러스터를 위한 인라인 정책을 추가합니다. \
또한 S3버킷과 Role을 생성합니다.

아래와 같이 Cloud9 콘솔에서 내용을 입력합니다.

```
aws s3 mb s3://eksworkshop-${ACCOUNT_ID}-codepipeline-artifacts
cd ~/environment
wget https://eksworkshop.com/intermediate/260_weave_flux/iam.files/cpAssumeRolePolicyDocument.json
aws iam create-role --role-name eksworkshop-CodePipelineServiceRole --assume-role-policy-document file://cpAssumeRolePolicyDocument.json 
wget https://eksworkshop.com/intermediate/260_weave_flux/iam.files/cpPolicyDocument.json
aws iam put-role-policy --role-name eksworkshop-CodePipelineServiceRole --policy-name codepipeline-access --policy-document file://cpPolicyDocument.json
wget https://eksworkshop.com/intermediate/260_weave_flux/iam.files/cbAssumeRolePolicyDocument.json
aws iam create-role --role-name eksworkshop-CodeBuildServiceRole --assume-role-policy-document file://cbAssumeRolePolicyDocument.json 
wget https://eksworkshop.com/intermediate/260_weave_flux/iam.files/cbPolicyDocument.json
aws iam put-role-policy --role-name eksworkshop-CodeBuildServiceRole --policy-name codebuild-access --policy-document file://cbPolicyDocument.json

```



### 2.Github 구성

2개의 GitHub 리포지토리를 만듭니다. 하나는 Docker 이미지 빌드를 트리거하는 샘플 애플리케이션에 사용됩니다.&#x20;

다른 하나는 Weave Flux가 클러스터에 배포하는 Kubernetes 매니페스트를 보관하는 데 사용됩니다. 이는 Kubernetes에 푸시하는 다른 CD 도구와 비교하여 Pull 기반 방법입니다.&#x20;

아래와 같이 개인 Github 계정에서 샘플 애플리케이션 리포지토리를 생성합니다. \
리포지토리 이름, 설명으로 양식을 채우고 아래와 같이 README로 리포지토리 초기화를 확인하고 리포지토리 생성을 클릭합니다.

<figure><img src="../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

위의 단계와 동일하게 Kubernetes 매니페스트 리포지토리를 만듭니다. 아래와 같이 양식을 작성하고 저장소 생성을 클릭합니다.

<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

다음 단계는 CodePipeline이 GitHub에서 콜백을 수신할 수 있도록 개인 액세스 토큰을 생성하는 것입니다. 생성된 액세스 토큰은 재사용할 수 있으므로 이 단계는 처음 사용하는 경우이거나 새 키를 생성해야 하는 경우에만 필요합니다. GitHub에서 새 개인 액세스 페이지를 엽니다.
