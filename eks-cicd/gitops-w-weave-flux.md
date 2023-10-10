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

<figure><img src="../.gitbook/assets/image (108).png" alt=""><figcaption></figcaption></figure>

위의 단계와 동일하게 Kubernetes 매니페스트 리포지토리를 만듭니다. 아래와 같이 양식을 작성하고 저장소 생성을 클릭합니다.

<figure><img src="../.gitbook/assets/image (302).png" alt=""><figcaption></figcaption></figure>

CodePipeline이 GitHub에서 Callback을 수신하려면 개인 액세스 토큰을 생성해야 합니다.

일단 생성되면 액세스 토큰을 보안 영역에 저장하고 재사용할 수 있으므로 이 단계는 처음 실행하는 동안이나 새 키를 생성해야 할 때만 필요합니다.&#x20;

개인 Github 계정에서 "settings"를 선택합니다.&#x20;

Profile 메뉴 하단의 "Developer settings"를 선택합니다.&#x20;



<figure><img src="../.gitbook/assets/image (332).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (235).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (325).png" alt=""><figcaption></figcaption></figure>

personal access token 생성을 확인하고, 복사해 둡니다.&#x20;

<figure><img src="../.gitbook/assets/image (98).png" alt=""><figcaption></figcaption></figure>

### 3.WEAVE FLUX 설치

Helm을 사용하여 클러스터에 Weave Flux를 설치하고 GitHub 리포지토리와 상호 연계 할 수 있습니다.

FLUX CRD(Custom Resource Definition)를 설치합니다.

```
kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/master/deploy/crds.yaml

```

github User Id 정보에 대해서, 사전에 변수로 저장해 둡니다.

```
export githubid={github userid}
echo "export githubid=${githubid}"| tee -a ~/.bash_profile
source ~/.bash_profile

```

flux라는 이름의 Namespace를 만들고, Helm을 통해서 Flux와 Helm-Operator를 설치 합니다.

```
helm repo add fluxcd https://charts.fluxcd.io
helm repo update
kubectl create namespace flux
helm upgrade -i flux fluxcd/flux \
--set git.url=git@github.com:${githubid}/k8s-config \
--set git.branch=main \
--namespace flux

helm upgrade -i helm-operator fluxcd/helm-operator \
--set helm.versions=v3 \
--set git.ssh.secretName=flux-git-deploy \
--set git.branch=main \
--namespace flux

```

정상적으로 FluxPod와 Helm Operator가 설치되었는지 확인합니다.

```
kubectl get pods -n flux

```

아래와 같이 확인 할 수 있습니다.

```
$ kubectl get pods -n flux
NAME                              READY   STATUS    RESTARTS   AGE
flux-6886cd9cc6-2wwxf             1/1     Running   0          4m44s
flux-memcached-74f9d695cd-sg7ls   1/1     Running   0          4m44s
helm-operator-56b9f8b5c5-68kmd    1/1     Running   0          4m42s
```

GitHub에 Write 접근을 허용하는 SSH 키를 얻기 위해 fluxctl를 설치합니다. 이를 통해 Flux는 GitHub의 구성을 클러스터에 배포된 구성과 동기화 상태로 유지할 수 있습니다.

```
sudo wget -O /usr/local/bin/fluxctl $(curl https://api.github.com/repos/fluxcd/flux/releases/latest | jq -r ".assets[] | select(.name | test(\"linux_amd64\")) | .browser_download_url")
sudo chmod 755 /usr/local/bin/fluxctl

fluxctl version
fluxctl identity --k8s-fwd-ns flux

```

결과에 출력된 키를 복사하고 GitHub 리포지토리에 배포 키로 추가합니다.

```
$ fluxctl identity --k8s-fwd-ns flux
ssh-rsa xxxxxxxyc2EAAAADAQABAAABgQC/LPcWqaF4gixC7ZTNhDj8Mwr8F3l4gfVjXJOV7dDwRP5m8kYc4gtV5MzT5p0mlrJwJHcyWpoqya4dIFN35uW1raKGwV27zJNBasjYJjQBusQJ6o2C2oXO0prPpjfMhFkvCw0EyzsLn399zqshTG7jNInCVSYvFet4JB+yAOVBq3qdkMYvdjJzjlpJ5foIAxq3/SgiiOHjfp12WG9U2jjLpitXISXyF3l9A4kPgO6ntMSX0WFmFP0ZFsp97XsA5XNC+RTTgRqYHXY3xO6+EUqk2jadzt1y2I5sfnvc/kCX2fNkxgblhJic/zDB0xkznWEh7LC0wINEDfYIQyHDxH3HZpELenelwZrfYixCP8R9Jg9l+Hje3eGt5592CH+R+pUFsF1LoujaxWJnmXpxNmmD9AZALxJXxchQtHQcbKGtu6tio4H7Vl5DF5A/ahc0eK4PuQJa2P/pZdIQLgXrpEJnpjztVwBbxxxxxxxxxx= root@flux-6886cd9cc6-2wwxf
```

GitHub 쓰기 접근을 허용하는 SSH 키를 취득하기 우해 fluxctl을 설치합니다. 이를 통해 Flux는 GitHub의 구성을 클러스터에 배포된 구성과 동기화 된 상태로 유지 시킬 수 있습니다.

개인 Github Repo에서 앞서 생성한 k8s-config reop로 이동하고, settings에서 Deploy keys를 선택합니다.



* Github k8s-config repo - settings - Deploy Keys 선택

<figure><img src="../.gitbook/assets/image (99).png" alt=""><figcaption></figcaption></figure>

Add deploy key를 선택합니다.

* Add deploy key 선택

<figure><img src="../.gitbook/assets/image (93).png" alt=""><figcaption></figcaption></figure>

앞서 복사해 둔 fluxctl identity --k8s-fwd-ns flux 값을 붙여 넣고, Add Key를 선택합니다.

<figure><img src="../.gitbook/assets/image (91).png" alt=""><figcaption></figcaption></figure>

아래와 같이 Deploy Keys에 추가 됩니다.

<figure><img src="../.gitbook/assets/image (113).png" alt=""><figcaption></figcaption></figure>

### 4.CODEPIPELINE으로 이미지 생성

이제 AWS CloudFormation을 사용하여 AWS CodePipeline을 생성합니다.

이 파이프라인은 GitHub 소스 리포지토리(eks-example)에서 Docker 이미지를 빌드하는 데 사용됩니다. 이것은 이미지를 배포하지는 않습니다. Weave Flux가 처리하게 됩니다.

Cloudformation yaml 은 아래 링크를 실행합니다.

```
https://s3.amazonaws.com/eksworkshop.com/templates/main/weave_flux_pipeline.cfn.yml
```

<figure><img src="../.gitbook/assets/image (311).png" alt=""><figcaption></figcaption></figure>

Stack의 세부정보를 아래와 같이 입력합니다.

* Stack Name : image-codepipeline
* username : github username
* Access token: github token (앞서 Github token을 복사해 두었습니다.)
* repository : eks-example (default - 앞서 설정한 Repo 이름)
* Branch : main

<figure><img src="../.gitbook/assets/image (87).png" alt=""><figcaption></figcaption></figure>

Cloudformation이 완료 되면 아래와 같이 확인할 수 있습니다.

<figure><img src="../.gitbook/assets/image (96).png" alt=""><figcaption></figcaption></figure>

Codepipeline으로 이동해서 진행상황을 살펴 봅니다.

<figure><img src="../.gitbook/assets/image (73).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (94).png" alt=""><figcaption></figcaption></figure>

현재 리포지토리에 코드가 없기 때문에 이미지 빌드가 실패했을 것입니다.&#x20;

<figure><img src="../.gitbook/assets/image (225).png" alt=""><figcaption></figcaption></figure>

GitHub 리포지토리(eks-example)에 샘플 애플리케이션을 추가합니다. GitHub 사용자 이름을 대체하는 리포지토리를 복제합니다.

```
source ~/.bash_profile
cd ~/environment
git clone https://github.com/${githubid}/eks-example.git
cd eks-example

```

기본 README 파일, 소스 디렉토리를 만들고 샘플 nginx 구성(hello.conf), 홈페이지(index.html) 및 Dockerfile을 다운로드합니다.

```
echo "# eks-example" > README.md
mkdir src
wget -O src/hello.conf https://raw.githubusercontent.com/aws-samples/eks-workshop/main/content/intermediate/260_weave_flux/app.files/hello.conf
wget -O src/index.html https://raw.githubusercontent.com/aws-samples/eks-workshop/main/content/intermediate/260_weave_flux/app.files/index.html
wget https://raw.githubusercontent.com/aws-samples/eks-workshop/main/content/intermediate/260_weave_flux/app.files/Dockerfile

```

이제 간단한 Hello World 앱이 있으므로 변경 사항을 커밋하여 이미지 빌드 파이프라인을 시작합니다.

```
git add .
git commit -am "Initial commit"
git push 

```

아래와 같이 CodePipeline에서 다시 진행되는 것을 확인해 볼 수 있습니다.

<figure><img src="../.gitbook/assets/image (306).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (90).png" alt=""><figcaption></figcaption></figure>

ECR로 이동해서 정상적으로 Private Repo에 등록되었는지도 확인해 봅니다.

<figure><img src="../.gitbook/assets/image (312).png" alt=""><figcaption></figcaption></figure>

### 5.Manifest 파일 배포

이제 Weave Flux를 사용하여 Hello World 애플리케이션을 Amazon EKS 클러스터에 배포할 준비가 되었습니다. 이를 위해 GitHub 구성 리포지토리(k8s-config)를 복제한 다음 배포할 Kubernetes 매니페스트를 커밋합니다.

```
cd ~/environment
git clone https://github.com/${githubid}/k8s-config.git     
cd k8s-config
mkdir charts namespaces releases workloads

```

네임스페이스 Kubernetes 매니페스트를 만듭니다.

```
cat << EOF > ~/environment/k8s-config/namespaces/eks-example.yaml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    name: eks-example
  name: eks-example
EOF

```

Deploy를 위한 yaml manifest 파일을 만듭니다.

```
cat << EOF > ~/environment/k8s-config/workloads/eks-example-dep.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eks-example
  namespace: eks-example
  labels:
    app: eks-example
  annotations:
    # Container Image Automated Updates
    flux.weave.works/automated: "true"
    # do not apply this manifest on the cluster
    #flux.weave.works/ignore: "true"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: eks-example
  template:
    metadata:
      labels:
        app: eks-example
    spec:
      containers:
      - name: eks-example
        image: ${ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/eks-example:fluxtest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /
            port: http
        readinessProbe:
          httpGet:
            path: /
            port: http
EOF

```

위의 Deployment 에서 Annotation을 확인 할 수 있습니다.

* flux.weave.works/automated는 컨테이너 이미지가 자동으로 업데이트되어야 하는지 여부를 Flux에 알려줍니다.
* flux.weave.works/ignore는 주석 처리되어 있지만 Flux에게 배포를 일시적으로 무시하도록 지시하는 데 사용할 수 있습니다.

마지막으로 로드 밸런서를 생성할 수 있도록 service yaml을 생성합니다.

```
cat << EOF > ~/environment/k8s-config/workloads/eks-example-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: eks-example
  namespace: eks-example
  labels:
    app: eks-example
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app: eks-example
EOF

```

이제 변경 사항을 커밋하고 Repo에 Push합니다.

```
git add . 
git commit -am "eks-example-deployment"
git push 

```

Flux 포드의 로그를 확인 해 봅니다.

5분마다 k8s-config 저장소에서 구성을 가져옵니다.&#x20;

```
kubectl get pods -n flux
kubectl logs flux-6886cd9cc6-2wwxf  -n flux

```

service 주소를 확인합니다.

```
kubectl -n eks-example get svc
kubectl -n eks-example get svc eks-example | tail -n 1 | awk '{ print "eks-example URL = http://"$4 }'

```

eks-example 소스 코드를 변경하고 새 변경 사항을 Push 해 봅니다.

