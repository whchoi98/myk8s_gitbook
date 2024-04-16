---
description: 'Update : 2024-04-16'
---

# Github Action & ArgoCD 기반 CI/CD

이번 실습에서는 Github Action과 ArgoCD를 활용하여 CI/CD 파이프라인을 아래와 같이 구성하는 실습을 진행하도록 하겠습니다.

<figure><img src="../.gitbook/assets/Untitled (3).png" alt=""><figcaption></figcaption></figure>

### Continuous Integration with Github Action

1.  두 개의 git repo; application, k8s manifest 생성

    실습을 위해서는 다음 두 개의 Github 레파지토리가 필요합니다.

    * _**front-app-repo**_: 애플리케이션 소스가 위치한 레파지토리
    * _**k8s-manifest-repo**_: K8S 관련 메니페스트가 위치한 레파지토리
2.  `front-app-repo` 라는 이름으로 애플리케이션 레파지토리를 생성합니다.

    <figure><img src="../.gitbook/assets/Screenshot 2024-04-16 at 10.15.32 PM.png" alt=""><figcaption></figcaption></figure>
3.  `k8s-manifest-repo` 라는 이름으로 github 레파지토리를 생성 합니다. 이 레파지토리는 kubernetes manifests 들을 저장하는 레파지토리입니다.

    <figure><img src="../.gitbook/assets/Screenshot 2024-04-16 at 5.24.26 PM (1).png" alt=""><figcaption></figcaption></figure>
4. User profile > Settings > Developer settings > Personal access tokens 탭의 Tokens (classic)을 클릭합니다. 그리고 우측 상단에 위치한 Generate new token을 선택 합니다. Note 에 test-eks-workshop 라고 입력 하고 Select scopes 에서 repo, workflow 를 선택 합니다. 그리고 화면 아래에서 Generate token 을 클릭 합니다.

<figure><img src="../.gitbook/assets/Screenshot 2024-04-16 at 3.49.35 PM.png" alt=""><figcaption></figcaption></figure>

그리고 화면에 출력되는 token 값을 복사한 후 기록해둡니다.

<figure><img src="../.gitbook/assets/Untitled (3) (1).png" alt=""><figcaption></figcaption></figure>

5. Cloud9 터미널로 돌아가 환경변수가 제대로 설정되어있는지 확인하고, 설정되어있지 않다면 실습 환경에 맞게 설정합니다.

```sh
echo $AWS_REGION
echo $ACCOUNT_ID
```

5. `demo-frontend` 라는 이름의 ECR repository를 생성합니다.

```sh
aws ecr create-repository \\
--repository-name demo-frontend \\
--image-scanning-configuration scanOnPush=true \\
--region ${AWS_REGION}
```

7. 실습용 소스코드를 clone한 후, 생성된 `amazon-eks-frontend` 디렉토리의 .git 파일을 삭제하고 초기화합니다.

```sh
cd /home/ec2-user/environment
git clone <https://github.com/joozero/amazon-eks-frontend.git>
cd ~/environment/amazon-eks-frontend
rm -rf .git
git init
```

8. 스크립트의 **your-github-username**를 실습자의 환경에 맞게 수정하여 환경변수를 설정합니다.

```jsx
export GITHUB_USERNAME=your-github-username
```

9. 위에서 생성한 `front-app-repo` 에 애플리케이션 소스코드를 push합니다.

```jsx
cd ~/environment/amazon-eks-frontend
git init
git add .
git commit -m "first commit"
git branch -M main
git remote add origin <https://github.com/$GITHUB_USERNAME/front-app-repo.git>
git push -u origin main
```

이 과정이 완료되면 github에서 소스코드가 잘 업로드 되었는지 확인 합니다.

{% hint style="info" %}
push 과정에서 필요한 **username**은 **github username**, **password**는 위에서 복사한 **token** 값을 입력합니다.
{% endhint %}

<figure><img src="../.gitbook/assets/Screenshot 2024-04-16 at 5.40.30 PM.png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
만약 push 과정에서, username, password 를 매번 넣어야 하는 상황이 번거롭다면 아래와 같이 cache 설정을 통해 지정 된 시간(기본 15분) 동안 cache 기반으로 로그인 가능 합니다. 이 때 아래의 USERNAME 및 EMAIL 값은 실습자의 환경에 맞게 수정하여 입력합니다.
{% endhint %}

```jsx
git config --global user.name USERNAME
git config --global user.email EMAIL
git config credential.helper store
git config --global credential.helper 'cache --timeout TIME YOU WANT'
```

10. 애플리케이션을 빌드하고, docker 이미지를 생성한 후, 이를 ECR 에 push 하는 과정은 Github Action을 통해 이루어 집니다. 이 과정에서 사용할 IAM User를 least privilege 를 준수하여 생성 합니다. 먼저 IAM User를 생성합니다.

```sh
aws iam create-user --user-name github-action
```

11. IAM User에 연결할 ECR Policy를 생성합니다.

```sh
cd ~/environment
cat <<EOF> ecr-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowPush",
            "Effect": "Allow",
            "Action": [
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage",
                "ecr:BatchCheckLayerAvailability",
                "ecr:PutImage",
                "ecr:InitiateLayerUpload",
                "ecr:UploadLayerPart",
                "ecr:CompleteLayerUpload"
            ],
            "Resource": "arn:aws:ecr:${AWS_REGION}:${ACCOUNT_ID}:repository/demo-frontend"
        },
        {
            "Sid": "GetAuthorizationToken",
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken"
            ],
            "Resource": "*"
        }
    ]
}
EOF

```

12. 만들어진 파일을 통해 IAM policy를 생성 합니다. 이때 policy 이름으로 `ecr-policy` 를 사용 합니다.

```jsx
aws iam create-policy --policy-name ecr-policy --policy-document file://ecr-policy.json
```

13. 생성한 ecr-policy를 새로 생성한 IAM user 에게 할당 합니다.

```jsx
aws iam attach-user-policy --user-name github-action --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/ecr-policy
```

14. 이번 실습에서는 Github Action이 빌드된 애플리케이션을 Docker Image로 만들어 ECR로 push 합니다. 이 과정에서 AWS credential 을 사용 합니다. 이를 위해 앞서 `github-action`이라는 IAM User를 생성 했습니다. 이제 아래 명령어를 실행하여 해당 User의 Access Key, Secret Key를 생성하도록 하겠습니다.

```jsx
aws iam create-access-key --user-name github-action
```

아래와 같은 출력 결과 중 `"SecretAccessKey"`, `"AccessKeyId"`값을 따로 기록해둡니다. 이 값은 향후에 사용 합니다.

```jsx
{
  "AccessKey": {
    "UserName": "github-action",
    "Status": "Active",
    "CreateDate": "2021-07-29T08:41:04Z",
    "SecretAccessKey": "***",
    "AccessKeyId": "***"
  }
}

```

15. Github의 `front-app-repo` 레파지토리로 돌아가 **Settings > Secrets and variables** 를 선택 한 후, 화면 우측의 **New repository secret** 을 클릭 합니다.

    <figure><img src="../.gitbook/assets/Screenshot 2024-04-16 at 3.36.39 PM (2).png" alt=""><figcaption></figcaption></figure>
16. 아래 화면 같이 **Name**에는 `ACTION_TOKEN` **Value** 에는 앞서 복사한 Github의 personal access token 값을 넣은 후 **Add secret** 을 클릭 합니다.

    <figure><img src="../.gitbook/assets/Untitled (3) (3).png" alt=""><figcaption></figcaption></figure>
17. 다음은 마찬가지 절차로, 앞서 생성 후 기록/저장 해둔 IAM User 인 **`github-action`** 의 `AccessKeyId` 와 `SecretAccessKey`의 값을 Secret 에 저장 합니다. 이때 `AccessKeyId` 와 `SecretAccessKey`의 각각의 Name 값은 `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` 로 합니다.

    <figure><img src="../.gitbook/assets/Untitled (4).png" alt=""><figcaption></figcaption></figure>

    <figure><img src="../.gitbook/assets/Untitled (5) (1).png" alt=""><figcaption></figcaption></figure>
18. 이제 Github Action에서 사용할 빌드 스크립트를 생성하도록 하겠습니다. 먼저 .github 및 workflows 디렉터리를 생성합니다.

    ```jsx
    cd ~/environment/amazon-eks-frontend
    mkdir -p ./.github/workflows
    ```
19. front-app Repo의 소스코드를 checkout 하고, build 한 다음, docker container 로 만들어 ECR 로 push 하는 과정을 담고 있는 github action build 스크립트를 작성 합니다. 이 때 `$IMAGE_TAG` 값은 빌드 마다 랜덤한 값으로 만들어 이미지에 부착하여 ECR로 push 합니다.

    ```jsx
    cd ~/environment/amazon-eks-frontend/.github/workflows
    cat > build.yaml <<EOF

    name: Build Front

    on:
      push:
        branches: [ main ]

    jobs:
      build:
        runs-on: ubuntu-latest
        steps:
          - name: Checkout source code
            uses: actions/checkout@v2

          - name: Check Node v
            run: node -v

          - name: Build front
            run: |
              npm install
              npm run build

          - name: Configure AWS credentials
            uses: aws-actions/configure-aws-credentials@v1
            with:
              aws-access-key-id: \\${{ secrets.AWS_ACCESS_KEY_ID }}
              aws-secret-access-key: \\${{ secrets.AWS_SECRET_ACCESS_KEY }}
              aws-region: $AWS_REGION

          - name: Login to Amazon ECR
            id: login-ecr
            uses: aws-actions/amazon-ecr-login@v1

          - name: Get image tag(verion)
            id: image
            run: |
              VERSION=\\$(echo \\${{ github.sha }} | cut -c1-8)
              echo VERSION=\\$VERSION
              echo "::set-output name=version::\\$VERSION"

          - name: Build, tag, and push image to Amazon ECR
            id: image-info
            env:
              ECR_REGISTRY: \\${{ steps.login-ecr.outputs.registry }}
              ECR_REPOSITORY: demo-frontend
              IMAGE_TAG: \\${{ steps.image.outputs.version }}
            run: |
              echo "::set-output name=ecr_repository::\\$ECR_REPOSITORY"
              echo "::set-output name=image_tag::\\$IMAGE_TAG"
              docker build -t \\$ECR_REGISTRY/\\$ECR_REPOSITORY:\\$IMAGE_TAG .
              docker push \\$ECR_REGISTRY/\\$ECR_REPOSITORY:\\$IMAGE_TAG

    EOF

    ```
20. 이제 변경한 코드를 `front-app-repo` 로 push 하여 github action workflow를 동작 시킵니다. 위에서 작성한 build.yaml 을 기반으로 github action이 동작 합니다.

    ```jsx
    cd ~/environment/amazon-eks-frontend
    git add .
    git commit -m "Add github action build script"
    git push origin main
    ```
21. Github의 `front-app-repo` 로 돌아가 변경사항이 push 되었는지 확인하고, Actions 탭의 workflow가 다음과 같이 동작 하는지 확인 합니다.

    <figure><img src="../.gitbook/assets/Untitled (3) (4).png" alt=""><figcaption></figcaption></figure>

    <figure><img src="../.gitbook/assets/Untitled (6).png" alt=""><figcaption></figcaption></figure>
22. 정상적으로 빌드가 완료 되었다면 애플리케이션 `demo-frontend` ECR 레파지토리로 돌아가, 새로운 `$IMAGE_TAG`를 갖는 이미지가 push 되었는지 확인 합니다. sha 값의 일부가 포함된 Image Tag로 push 된 이미지가 확인 됩니다.

    <figure><img src="../.gitbook/assets/Screenshot 2024-04-16 at 10.40.04 PM.png" alt=""><figcaption></figcaption></figure>

### Continuous Deployment with ArgoCD

1. 본 실습 에서는 Kustomize 를 활용해 쿠버네티스 Deployment 리소스에 동일한 label, metadata 값을 주도록 하고, 애플리케이션 소스코드의 새로운 변경 사항 발생에 따른 새로운 Image Tag를 Deployment 리소스에 적용하도록 하겠습니다. 먼저 Kustomize 사용을 위해 kubernetes manifest 디렉토리를 구조화하도록 하겠습니다.

{% hint style="info" %}
Kustomize 는 쿠버네티스 manifest 를 사용자 입맛에 맞도록 정의 하는데 도움을 주는 도구 입니다. 여기서 "사용자 입맛에 맞도록 정의"에 포함 되는 경우는, 모든 리소스에 동일한 네임스페이스를 정의 하거나, 동일한 네임 접두사/접미사를 준다거나, 동일한 label 을 주거나, 이미지 태그 값을 변경 하는 것들이 있을 수 있습니다. Kustomize 에 관한 자세한 내용은 [다음 ](https://kustomize.io/)을 참고 하세요.
{% endhint %}

```jsx
cd ~/environment
git clone https://github.com/$GITHUB_USERNAME/k8s-manifest-repo.git
mkdir -p ./k8s-manifest-repo/base
mkdir -p ./k8s-manifest-repo/overlays/dev
```

결과적으로 만들어지는 디렉토리 구조는 다음과 같습니다. _`k8s-manifest-repo`_ 디렉토리 아래에 _`base`_, _`overlays/dev`_ 구조가 생깁니다.

* _`base`_ : kubernetes manifest 원본파일들을 저장할 디렉토리 입니다. 이 안에 위치한 manifest 들은 _`overlays`_ 아래에 위치한 **kustomize.yaml** 파일에 담긴 **사용자 지정 설정** 내용에 따라 변경 됩니다.
* _`overlays`_ : **사용자 입맛에 맞는** 설정 값을 저장할 디렉토리 입니다. 이 설정은 **kustomize.yaml** 에 담습니다. 이 하위에 있는 _`dev`_ 디렉토리는 실습을 위해 만든 것으로, 개발 환경에 적용할 설정 파일을 모아 두기 위함 입니다.

2. kubernetes manifest 파일을 생성하도록 하겠습니다. 이 때, 이미지 값에는 **demo-frontend** 리포지토리 URI 값이 잘 들어갔는지 확인합니다.

```jsx
cd ~/environment/k8s-manifest-repo/base

cat <<EOF> frontend-deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-frontend
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo-frontend
  template:
    metadata:
      labels:
        app: demo-frontend
    spec:
      containers:
        - name: demo-frontend
          image: $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/demo-frontend:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 80
EOF

```

```jsx
cat <<EOF> frontend-service.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: demo-frontend
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: "/"
spec:
  selector:
    app: demo-frontend
  type: NodePort
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
EOF

```

3. 이번 실습에서는 위에서 생성한 frontend-deployment.yaml 과 frontend-service.yaml 파일을 kustomize 를 통해 배포 시점에 의도한 값(e.g. Image Tag)을 반영해보도록 하겠습니다. 아래는 반영될 값 입니다.

* **`metadata.labels`:** `"env: dev"`을 frontend-deployment.yaml, frontend-service.yaml 에 일괄 반영 합니다.
* **`spec.selector`** : `"select.app: frontend-fargate"` 를 frontend-deployment.yaml, frontend-service.yaml 에 일괄 반영 합니다.
* **`spec.template.spec.containers.image`** : `"image: "` 값을 새롭게 변경된 Image Tag 정보로 업데이트 합니다.

아래와 같이 kustomize.yaml 파일을 만듭니다. 이 파일을 통해 kustomize을 통해 관리/변경할 kubernetes manifest 대상을 정의합니다.

```jsx
cd ~/environment/k8s-manifest-repo/base
cat <<EOF> kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - frontend-deployment.yaml
  - frontend-service.yaml
EOF
```

4. 이제 위에서 정의한 kubernetes manifest 대상에 어떤 값들을 입맛에 맞게 적용할지 를 결정 하기 위한 파일을 만듭니다. 먼저 fronted-deployment.yaml 을 위한 설정 값을 정의 하겠습니다.

```jsx
cd ~/environment/k8s-manifest-repo/overlays/dev
cat <<EOF> front-deployment-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-frontend
  namespace: default
  labels:
    env: dev
spec:
  selector:
    matchLabels:
      app: frontend-fargate
  template:
    metadata:
      labels:
        app: frontend-fargate
EOF

```

5. 다음은 frontend-service.yaml 을 위한 설정 값을 정의 하겠습니다.

```jsx
cd ~/environment/k8s-manifest-repo/overlays/dev
cat <<EOF> front-service-patch.yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-frontend
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: "/"
  labels:
    env: dev
spec:
  selector:
    app: frontend-fargate
EOF

```

6. 마지막으로 위에서 설정 한 파일들(값)을 사용하고 애플리케이션 빌드에 따라 만들어진 새로운 **Image Tag** 를 사용하겠다고 정의 하겠습니다. 구체적으로는, `name` 에 지정된 image는 `newName`의 image와 `newTag`의 값으로 사용 하겠다는 의미 입니다. 이를 활용해 `newTag` 값을 변경해 새로운 배포가 이루어질 때 마다 이를 kubernetes 클러스터에 반영 할 수 있습니다. 아래 명령을 실행 합니다.

```jsx
cd ~/environment/k8s-manifest-repo/overlays/dev
cat <<EOF> kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
images:
- name: ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/demo-frontend
  newName: ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/demo-frontend
  newTag: abcdefg
resources:
- ../../base
patchesStrategicMerge:
- front-deployment-patch.yaml
- front-service-patch.yaml
EOF

```

이렇게 구성하면 -patch.yaml 파일에 정의한 내용들은 배포 과정에서 kustomize 에 의해 자동으로 kubernetes manifest 에 반영 됩니다.

7. 생성한 kubernetes manifests 및 kustomize 관련 파일들을 `k8s-manifest-repo` 에 push합니다.

```jsx
cd ~/environment/k8s-manifest-repo/
git add .
git commit -m "first commit"
git push -u origin main

```

8. 다음을 실행 하여 ArgoCD를 eks cluster 에 설치 합니다.

```jsx
kubectl create namespace argocd
kubectl apply -n argocd -f <https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml>
```

9. ArgoCD 는 GUI 뿐만 아니라 CLI도 제공 합니다. 아래를 실행하여 ArgoCD CLI 를 설치 합니다.

```jsx
cd ~/environment
VERSION=$(curl --silent "<https://api.github.com/repos/argoproj/argo-cd/releases/latest>" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\\1/')

sudo curl --silent --location -o /usr/local/bin/argocd <https://github.com/argoproj/argo-cd/releases/download/$VERSION/argocd-linux-amd64>

sudo chmod +x /usr/local/bin/argocd

```

10. ArgoCD 서버는 기본적으로 퍼블릭하게 노출되지 않습니다. 여기에서는 편의상 로드 밸런서를 사용하여 외부에 노출 시킵니다. LoadBalancer가 생성될 때까지 2\~3분 정도 소요됩니다.

```jsx
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

11. 아래 명령을 통해서 Argo CD 서버의 주소를 변수에 저장해 둡니다.

```jsx
export ARGOCD_SERVER=`kubectl get svc argocd-server -n argocd -o json | jq --raw-output .status.loadBalancer.ingress[0].hostname`
echo $ARGOCD_SERVER
```

12. ArgoCD의 기본 Username은 `admin` 이고, 아래 명령어를 실행하여 `admin` 유저의 초기 password를 확인할 수 있습니다.

```jsx
ARGO_PWD=`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`
echo $ARGO_PWD
```

13. 브라우저를 열고 `echo $ARGO_SERVER` 의 결과값을 URL에 입력합니다. 그리고 Username `admin` 을 입력하고 Password 는 `echo $ARGO_PWD` 의 결과값을 입력 합니다.



    <figure><img src="../.gitbook/assets/Untitled (3) (6).png" alt=""><figcaption></figcaption></figure>

Cloud9 터미널에서 다음 명령어를 실행하여 CLI에서도 로그인을 진행할 수 있습니다.

```jsx
argocd login $ARGOCD_SERVER --username admin --password $ARGO_PWD --insecure
```

위의 명령어를 실행하면 다음과 같이 CLI 환경에서 로그인이 된 것을 확인할 수 있습니다.

```jsx
$ argocd login $ARGOCD_SERVER --username admin --password $ARGO_PWD --insecure
'admin:login' logged in successfully
Context 'a81830f78562145b3bca329b3a7699cb-1011047444.ap-northeast-2.elb.amazonaws.com' updated

```

14. 클러스터 컨텍스트를 사용하여 ArgoCD CLI와 연결합니다.

```jsx
CONTEXT_NAME=`kubectl config view -o jsonpath='{.current-context}'`
argocd cluster add $CONTEXT_NAME -y
```

위 명령어 실행 결과는 아래와 같습니다.

```jsx
$ argocd cluster add $CONTEXT_NAME -y
WARNING: This will create a service account `argocd-manager` on the cluster referenced by context `i-0fd091734070b9f6f@eksworkshop.ap-northeast-2.eksctl.io` with full cluster level privileges. Do you want to continue [y/N]? y
INFO[0003] ServiceAccount "argocd-manager" created in namespace "kube-system" 
INFO[0003] ClusterRole "argocd-manager-role" created    
INFO[0003] ClusterRoleBinding "argocd-manager-role-binding" created 
Cluster '<https://095783B7DF46A1EA53103B1201579207.gr7.ap-northeast-2.eks.amazonaws.com>' added
```

15. ArgoCD의 애플리케이션을 구성하고 `k8s-manifest-repo` 에 연결합니다.

```jsx
argocd app create eksworkshop-cd-pipeline --repo <https://github.com/${GITHUB_USERNAME}/k8s-manifest-repo.git> --path overlays/dev --dest-server <https://kubernetes.default.svc> --dest-namespace default --revision main
```

16. 이제 애플리케이션이 설정되었으므로 배포된 애플리케이션 상태를 argo cd cli로 확인합니다.

```jsx
argocd app get eksworkshop-cd-pipeline
```

다음과 같이 출력됩니다.

```jsx
$ argocd app get eksworkshop-cd-pipeline
Name:               argocd/eksworkshop-cd-pipeline
Project:            default
Server:             <https://kubernetes.default.svc>
Namespace:          default
URL:                <https://a3c68405e125146c990c107c40f198d1-556207836.ap-northeast-2.elb.amazonaws.com/applications/eksworkshop-cd-pipeline>
Repo:               <https://github.com/jinnypark9393/k8s-manifest-repo.git>
Target:             main
Path:               overlays/dev
SyncWindow:         Sync Allowed
Sync Policy:        <none>
Sync Status:        OutOfSync from main (0f4f2ab)
Health Status:      Missing

GROUP  KIND        NAMESPACE  NAME           STATUS     HEALTH   HOOK  MESSAGE
       Service     default    demo-frontend  OutOfSync  Missing        
apps   Deployment  default    demo-frontend  OutOfSync  Missing        
```

UI에서도 동일한 값을 확인할 수 있습니다.

<figure><img src="../.gitbook/assets/Screenshot 2024-04-16 at 9.53.51 PM.png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/Screenshot 2024-04-16 at 9.54.29 PM.png" alt=""><figcaption></figcaption></figure>

17. 앞서 생성한 github action build 스크립트에 kustomize를 이용하여 컨테이너 image tag 정보를 업데이트 한 후 _**k8s-manifest-repo**_에 commit/push 하는 단계를 추가 해야 합니다. 추가된 단계가 정상적으로 동작 하면, **ArgoCD**가 _**k8s-manifest-repo**_를 지켜 보고 있다가 새로운 변경 사항이 발생 되었음을 알아채고, **kustomize build** 작업을 수행하여 새로운 **kubernetes manifest** (\*새로운 image tag를 포함한)를 eks 클러스터에 배포 합니다.

```jsx
cd ~/environment/amazon-eks-frontend/.github/workflows
cat <<EOF>> build.yaml

      - name: Setup Kustomize
        uses: imranismail/setup-kustomize@v1

      - name: Checkout kustomize repository
        uses: actions/checkout@v2
        with:
          repository: $GITHUB_USERNAME/k8s-manifest-repo
          ref: main
          token: \\${{ secrets.ACTION_TOKEN }}
          path: k8s-manifest-repo

      - name: Update Kubernetes resources
        run: |
          echo \\${{ steps.login-ecr.outputs.registry }}
          echo \\${{ steps.image-info.outputs.ecr_repository }}
          echo \\${{ steps.image-info.outputs.image_tag }}
          cd k8s-manifest-repo/overlays/dev/
          kustomize edit set image \\${{ steps.login-ecr.outputs.registry}}/\\${{ steps.image-info.outputs.ecr_repository }}=\\${{ steps.login-ecr.outputs.registry}}/\\${{ steps.image-info.outputs.ecr_repository }}:\\${{ steps.image-info.outputs.image_tag }}
          cat kustomization.yaml

      - name: Commit files
        run: |
          cd k8s-manifest-repo
          git config --global user.email "github-actions@github.com"
          git config --global user.name "github-actions"
          git commit -am "Update image tag"
          git push -u origin main

EOF

```

새로운 Image Tag가 정상 반영 되었는지 살펴 보겠습니다. _**k8s-manifest-repo**_의 commit history를 통해 변경된 Image Tag 정보를 확인 합니다.

<figure><img src="../.gitbook/assets/Screenshot 2024-04-16 at 10.11.07 PM (2).png" alt=""><figcaption></figcaption></figure>

그리고 이 값이 ArgoCD에 의해 새롭게 배포된 `frontend-deployment`가 사용하는 Image Tag 값과 같은지 확인 합니다. ArgoCD 메뉴에서 **Applications > eksworkshop-cd-pipeline >** 이동 하여 다이어그램에서 `demo-frontend-`로 시작하는 **pod**을 클릭하면 상세 정보 확인이 가능합니다.

<figure><img src="../.gitbook/assets/Screenshot 2024-04-16 at 10.11.29 PM.png" alt=""><figcaption></figcaption></figure>

