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

Sample Repo 포크

리포지토리를 수정하고 빌드를 트리거할 수 있도록 샘플 Kubernetes 서비스를 분기합니다.&#x20;

GitHub에 로그인하고 샘플 서비스를 자신의 계정으로 분기합니다.

먼저 각자의 Github 계정으로 먼저 로그인하고, 아래 Repo로 이동합니다.&#x20;

Sample repo&#x20;

```
https://github.com/rnzsgh/eks-workshop-sample-api-service-go

```

아래와 같이 이동한 Repo에서 Fork 를 수행합니다.

![](<../.gitbook/assets/image (216).png>)

자신의 계정으로 분기된 Repo는 아래와 같습니다.&#x20;

![](<../.gitbook/assets/image (223).png>)

CodePipeline이 GitHub에서 Callback을 수신하려면 개인 액세스 토큰을 생성해야 합니다.

일단 생성되면 액세스 토큰을 보안 영역에 저장하고 재사용할 수 있으므로 이 단계는 처음 실행하는 동안이나 새 키를 생성해야 할 때만 필요합니다.&#x20;

개인 Github 계정에서 아와 같이 "settings"를 선택합니다.&#x20;



![](<../.gitbook/assets/image (225).png>)

Profile 메뉴 하단의 "Developer settings"를 선택합니다.&#x20;

![](<../.gitbook/assets/image (231).png>)

![](<../.gitbook/assets/image (221).png>)

