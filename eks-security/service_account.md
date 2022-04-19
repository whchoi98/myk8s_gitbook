---
description: 'Update : 2022-04-19'
---

# IRSA (IAM Roles for Service Account)

## IRSA (IAM Roles for Service Account)

Kubernetes 버전 1.12에서는 OIDC JSON 웹 토큰인 새로운 ProjectedServiceAccountToken 기능에 대한 지원이 추가되었습니다. Amazon EKS는 이제 ProjectedServiceAccountToken JSON Web Token에 대한 OIDC 엔드포인트를 호스팅하므로 IAM과 같은 외부 시스템이 Kubernetes에서 발급한 OIDC 토큰을 확인하고 수락할 수 있습니다. 이러한 IRSA는 OpenID Connector(OIDC) Identity Provider와 STS를 활용하여 쿠버네티스의 Service Account에 IAM Role을 연계하도록 합니다.

OIDC URL은 EKS 클러스터를 생성하면 자동으로 생성되고, 쿠버네티스 관리자는 이 OIDC를 통해 Identity Provider와 Pod에서 필요한 IAM Role을 만들어주고 Service Account에 announce를 해주면 컨트롤 플레인에서 이를 참조하여 OIDC 인증을 거친 뒤에 JWT를 발급하여 이를 통해 파드의 env에 AWS\_ROLE\_ARN, AWS\_WEB\_IDENTITY\_TOKEN\_FILE 설정을 하여 파드가 임시 IAM 권한을 획득할 수 있게 해줍니다.

OIDC Federation Access를 기반으로 STS(Secure Token Service)를 사용해서 IAM 역할을 수임할 수 있으며, OIDC 공급자를 통한 인증을 활성화하고 JWT(JSON 웹 토큰)를 수신하여 IAM Role을 수행할 있습니다.

새 자격 증명 공급자 "sts:AssumeRoleWithWebIdentity"



## 사전 준비

### 1.OIDC 확인.

EKS 클러스터에는 연결되어 있는 OpenID Connect 발급자 URL이 있으며 이는 IAM OIDC 공급자를 구성할 때 사용됩니다.  아래 명령으로  OIDC Provider URL을 확인합니다.&#x20;

```
aws eks describe-cluster --name ${ekscluster_name} --query cluster.identity.oidc.issuer --output text
 
```

OIDC 가 생성되어 있지 않다면, 아래와 같이 생성합니다. (Ingress Controller 를 생성하는 단계를 수행하였다면, OIDC는 이미 생성되었습니다.)

```
eksctl utils associate-iam-oidc-provider --cluster ${ekscluster_name} --approve

```

## &#x20;IRSA 생성

### 2.IRSA 생성

포드에 부여할 권한을 지정하는 IAM 정책을 생성합니다.

모든 S3 버킷에 대한 get, list 을 허용하는 "AmazonS3ReadOnlyAccess"라는 AWS 관리형 정책을 사용합니다.

"AmazonS3ReadOnlyAccess" 정책에 대한 ARN을 확인합니다.&#x20;

```
aws iam list-policies --query 'Policies[?PolicyName==`AmazonS3ReadOnlyAccess`].Arn'

```

아래와 같은 결과가 출력될 것입니다.&#x20;

```
arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

S3에 대한 읽기 전용 액세스 권한이 있는 서비스 계정에 바인딩된 IAM 역할을 생성합니다.

```
eksctl create iamserviceaccount \
    --name iam-test \
    --namespace default \
    --cluster ${ekscluster_name} \
    --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
    --approve \
    --override-existing-serviceaccounts

```

### 3.Service Account에 대한 IAM Role 연결

이전 단계에서 클러스터의 **`iam-test`** 라는 서비스 계정과 연결된 IAM 역할을 생성했습니다.

먼저 서비스 계정 **`iam-test`**가 생성되었는지 확인합니다.

```
kubectl get serviceaccounts iam-test

```

Service Account에 IAM 역할의 ARN Annotation이 추가되었는지 확인합니다.

```
kubectl describe serviceaccounts iam-test 

```

아래와 같이 출력될 것입니다.&#x20;

```
$ kubectl describe serviceaccounts iam-test 
Name:                iam-test
Namespace:           default
Labels:              app.kubernetes.io/managed-by=eksctl
Annotations:         eks.amazonaws.com/role-arn: arn:aws:iam::011218731119:role/eksctl-eksworkshop-addon-iamserviceaccount-d-Role1-F3SWVH044SEZ
Image pull secrets:  <none>
Mountable secrets:   iam-test-token-tbsl8
Tokens:              iam-test-token-tbsl8
Events:              <none>
```

### 4.IRSA 테스트

새로 생성된 IAM 역할로 두 개의 kubernetes 작업을 실행합니다.

* job-s3.yaml: `aws s3 ls` 명령의 결과를 출력합니다. (이 작업은 정상적으로 출력되어야 합니다.)
* job-ec2.yaml: `aws ec2 describe-instances --region ${AWS_REGION}` 명령의 결과를 출력합니다(이 작업은 실패해야 합니다.)

Service Account가 S3 버킷을 조회할 수 있는지 테스트하기 위해, 아래와 같이 yaml을 생성하고 배포합니다.&#x20;

```
mkdir ~/environment/irsa

cat <<EoF> ~/environment/irsa/job-s3.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: eks-iam-test-s3
spec:
  template:
    metadata:
      labels:
        app: eks-iam-test-s3
    spec:
      serviceAccountName: iam-test
      containers:
      - name: eks-iam-test
        image: amazon/aws-cli:latest
        args: ["s3", "ls"]
      restartPolicy: Never
EoF

kubectl apply -f ~/environment/irsa/job-s3.yaml
kubectl get job -l app=eks-iam-test-s3

```

app=esk-iam-test-s3 어플리케이션 로그를 확인합니다.&#x20;

```
kubectl logs -l app=eks-iam-test-s3

```

S3에 대한 조회를 한 로그를 확인합니다.&#x20;

```
$ kubectl logs -l app=eks-iam-test-s3
2022-04-19 13:53:06 eksworkshop-codepipeline-codepipelineartifactbuck-11n63p4vzrzxb
2022-04-19 10:06:13 skuser01
2022-04-19 12:13:03 skuser01chartmuseum
```

Service-Account가 EC2 인스턴스를 조회할 수 없는지 확인해 봅니다. 먼저 아래와 같이 Pod를 생성합니다.&#x20;

```
cat <<EoF> ~/environment/irsa/job-ec2.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: eks-iam-test-ec2
spec:
  template:
    metadata:
      labels:
        app: eks-iam-test-ec2
    spec:
      serviceAccountName: iam-test
      containers:
      - name: eks-iam-test
        image: amazon/aws-cli:latest
        args: ["ec2", "describe-instances", "--region", "${AWS_REGION}"]
      restartPolicy: Never
  backoffLimit: 0
EoF

kubectl apply -f ~/environment/irsa/job-ec2.yaml
kubectl get job -l app=eks-iam-test-ec2

```

app=esk-iam-test-ec2 어플리케이션 로그를 확인합니다.

```
kubectl logs -l app=eks-iam-test-ec2

```

EC2에 대한 조회를 한 로그를 확인합니다.&#x20;

```
$ kubectl logs -l app=eks-iam-test-ec2
Error from server (BadRequest): container "eks-iam-test" in pod "eks-iam-test-ec2-bnbsn" is waiting to start: ContainerCreating
```

아래 명령을 통해 yaml을 확인해 봅니다.

```
kubectl get pod eks-iam-test-s3-xxxx  debug -o yaml
kubectl get pod eks-iam-test-ec2-xxxx  debug -o yaml
```
