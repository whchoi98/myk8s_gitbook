---
description: 'Update : 2021-10-21'
---

# EFK기반 로깅

## EFK 소개

![](<../.gitbook/assets/image (279).png>)

Fluent Bit는 다양한 소스에서 데이터 / 로그를 수집하고 통합하여 여러 대상으로 보낼 수 있는 오픈 소스 및 다중 플랫폼 **로그 프로세서 및 Forwarder** 입니다. Docker 및 Kubernetes 환경 과 완벽하게 호환 됩니다.Fluent Bit는 **C** 로 작성되었으며 약 30 개의 확장을 지원하는 플러그 가능 아키텍처가 있습니다. 빠르고 가벼우 며 TLS를 통한 네트워크 운영에 필요한 보안을 제공합니다. (참조 - [https://fluentbit.io](https://fluentbit.io/))

## Fluent Bit 설치환경 구성 .

### 1.권한 설정

Amazon EKS 클러스터의 서비스 계정에 대한 IAM 역할을 사용하면, IAM 역할을 Kubernetes 서비스 계정과 연결할 수 있습니다. 그런 다음이 서비스 계정은 해당 서비스 계정을 사용하는 모든 포드의 컨테이너에 AWS 권한을 제공 할 수 있습니다. 이 기능을 사용하면 해당 노드의 포드가 AWS API를 호출 할 수 있도록 더 이상 노드 IAM 역할에 대한 확장 권한을 제공 할 필요가 없습니다.

#### Cluster에서 서비스 계정에 대한 IAM 역할을 활성화 합니다. (이전 LAB에서 이미 수행했을 수도 있습니다.)

```
eksctl utils associate-iam-oidc-provider \
    --cluster eksworkshop \
    --approve
```

### 2.서비스 계정에 대한 IAM 역할 및 정책 생성.&#x20;

서비스 계정에 대한 IAM 역할 및 정책을 생성합니다.

#### 생성에 앞서 주요 환경변수를 저장하고, 정상적으로 저장되었는 지 확인합니다. (이전 LAB에서 이미 수행했을 수도 있습니다.)

```
export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
export ES_DOMAIN_NAME="eksworkshop-logging"
echo $AWS_REGION
echo $ACCOUNT_ID
echo $ES_DOMAIN_NAME
```

#### Fluent Bit 컨테이너가 ElasticSearch 클러스터에 연결하는 데 필요한 권한을 제한하는 IAM 정책을 아래와 같이 생성합니다. 또한 Kubernetes 서비스 계정을 연결하기 위한 IAM역할을 만듭니다.

```
mkdir ~/environment/logging/

cat <<EoF > ~/environment/logging/fluent-bit-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "es:ESHttp*"
            ],
            "Resource": "arn:aws:es:${AWS_REGION}:${ACCOUNT_ID}:domain/${ES_DOMAIN_NAME}"
        }
    ]
}
EoF

aws iam create-policy   \
  --policy-name fluent-bit-policy \
  --policy-document file://~/environment/logging/fluent-bit-policy.json

```

다음과 같이 정상적으로 수행됩니다.

```
$ aws iam create-policy   \
>   --policy-name fluent-bit-policy \
>   --policy-document file://~/environment/logging/fluent-bit-policy.json
{
    "Policy": {
        "PolicyName": "fluent-bit-policy",
        "PolicyId": "ANPAVNGLFJAQVRQJH2LCC",
        "Arn": "arn:aws:iam::371940739105:policy/fluent-bit-policy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2021-10-21T16:20:59+00:00",
        "UpdateDate": "2021-10-21T16:20:59+00:00"
    }
}
```

### 3.IAM 역할 생성.

새로운 namespace를 생성하고, Fluent Bit 서비스 계정에 대한 IAM 역할을 생성합니다.

```
kubectl create namespace logging

eksctl create iamserviceaccount \
    --name fluent-bit \
    --namespace logging \
    --cluster eksworkshop \
    --attach-policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/fluent-bit-policy" \
    --approve \
    --override-existing-serviceaccounts

```

아래와 같은 결과를 확인 할 수 있습니다.

```
$ eksctl create iamserviceaccount \
>     --name fluent-bit \
>     --namespace logging \
>     --cluster eksworkshop \
>     --attach-policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/fluent-bit-policy" \
>     --approve \
>     --override-existing-serviceaccounts
2021-10-21 16:21:07 [ℹ]  eksctl version 0.70.0
2021-10-21 16:21:07 [ℹ]  using region ap-northeast-2
2021-10-21 16:21:08 [ℹ]  1 existing iamserviceaccount(s) (kube-system/aws-load-balancer-controller) will be excluded
2021-10-21 16:21:08 [ℹ]  1 iamserviceaccount (logging/fluent-bit) was included (based on the include/exclude rules)
2021-10-21 16:21:08 [!]  metadata of serviceaccounts that exist in Kubernetes will be updated, as --override-existing-serviceaccounts was set
2021-10-21 16:21:08 [ℹ]  1 task: { 
    2 sequential sub-tasks: { 
        create IAM role for serviceaccount "logging/fluent-bit",
        create serviceaccount "logging/fluent-bit",
    } }2021-10-21 16:21:08 [ℹ]  building iamserviceaccount stack "eksctl-eksworkshop-addon-iamserviceaccount-logging-fluent-bit"
2021-10-21 16:21:08 [ℹ]  deploying stack "eksctl-eksworkshop-addon-iamserviceaccount-logging-fluent-bit"
2021-10-21 16:21:08 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-addon-iamserviceaccount-logging-fluent-bit"
2021-10-21 16:21:25 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-addon-iamserviceaccount-logging-fluent-bit"
2021-10-21 16:21:42 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-addon-iamserviceaccount-logging-fluent-bit"
2021-10-21 16:21:42 [ℹ]  created serviceaccount "logging/fluent-bit"
```

정상적으로 service 계정이 생성되었는지 확인합니다.

```
kubectl -n logging describe serviceaccounts fluent-bit

```

아래와 같은 결과를 확인 할 수 있습니다.

```
$ kubectl -n logging describe serviceaccounts fluent-bit
Name:                fluent-bit
Namespace:           logging
Labels:              app.kubernetes.io/managed-by=eksctl
Annotations:         eks.amazonaws.com/role-arn: arn:aws:iam::371940739105:role/eksctl-eksworkshop-addon-iamserviceaccount-l-Role1-N1MQGT4G5PH4
Image pull secrets:  <none>
Mountable secrets:   fluent-bit-token-rh4bq
Tokens:              fluent-bit-token-rh4bq
Events:              <none>
```



## ElasticSearch Cluster 구성

### 1.사전 환경 구성

빠른 설치를 위해서 ES관련 키워드 몇가지를 변수에 저장해 둡니다.

```
# name of our elasticsearch cluster
export ES_DOMAIN_NAME="eksworkshop-logging"

# Elasticsearch version
export ES_VERSION="7.4"

# kibana admin user
export ES_DOMAIN_USER="eksworkshop"

# kibana admin password
export ES_DOMAIN_PASSWORD="$(openssl rand -base64 12)_Ek1$"

```

{% hint style="info" %}
ES의 패스워드는 one uppercase letter, one lowercase letter, one number,  one special character 를 포함하도록 되어 있습니다. openssl random으로 패스워드를 생성합니다.
{% endhint %}

### 2. ElasticSearch 설치.

```
# Download and update the template using the variables created previously
curl -sS https://www.eksworkshop.com/intermediate/230_logging/deploy.files/es_domain.json \
  | envsubst > ~/environment/logging/es_domain.json

# Create the cluster
aws es create-elasticsearch-domain \
  --cli-input-json  file://~/environment/logging/es_domain.json

```

AWS Console 에서 Elasticsearch를 검색합니다. AWS ES를 배포하게 되면 아래와 같이 "로드 중"으로 도메인 상태가 표기 됩니다.

![](<../.gitbook/assets/image (446).png>)

정상적으로 도메인 상태가 표기되기 까지는 15분 이상 소요됩니다.

![](<../.gitbook/assets/image (435).png>)

{% hint style="danger" %}
ElasticSearch 도메인 상태가 정상일 때까지 , 다음 단계를 수행하지 마십시요.
{% endhint %}

### 3. ElasticSearch Access 구성

앞서 생성한 Fluent Bit ARN이 ElasticSearch의 API를 통해 Backend 역할을 받아 접근 할 수 있도록 아래와 같이 설정합니다.

Endpoint URL은 아래에서 확인이 가능합니다.

![](<../.gitbook/assets/image (208).png>)

```
# Need to retrieve the Fluent Bit Role ARN, ES_Endpoint
export FLUENTBIT_ROLE=$(eksctl get iamserviceaccount --cluster eksworkshop --namespace logging -o json | jq '.[].status.roleARN' -r) 
export ES_ENDPOINT=$(aws es describe-elasticsearch-domain --domain-name ${ES_DOMAIN_NAME} --output text --query "DomainStatus.Endpoint")

# Update the Elasticsearch internal database
curl -sS -u "${ES_DOMAIN_USER}:${ES_DOMAIN_PASSWORD}" \
    -X PATCH \
    https://${ES_ENDPOINT}/_opendistro/_security/api/rolesmapping/all_access?pretty \
    -H 'Content-Type: application/json' \
    -d'
[
  {
    "op": "add", "path": "/backend_roles", "value": ["'${FLUENTBIT_ROLE}'"]
  }
]
'

```

정상적으로 구성되면 아래와 같이 결과가 출력됩니다.

```
{
  "status" : "OK",
  "message" : "'all_access' updated."
}
```

4.Fluent Bit 구성.

fluent Bit 매니페스트 파일을 다운받고, 일부 파일을 수정합니다.

```
cd ~/environment/logging

# get the Elasticsearch Endpoint
export ES_ENDPOINT=$(aws es describe-elasticsearch-domain --domain-name ${ES_DOMAIN_NAME} --output text --query "DomainStatus.Endpoint")

curl -Ss https://www.eksworkshop.com/intermediate/230_logging/deploy.files/fluentbit.yaml \
    | envsubst > ~/environment/logging/fluentbit.yaml

```

fluent Bit 파일을 배포합니다.

```
kubectl apply -f ~/environment/logging/fluentbit.yaml

```

정상적으로 모든 Worker Node에 설치되었는지 확인해 봅니다.

```
kubectl -n logging get pods -o wide

```

정상적으로 설치되었다면, 아래와 같이 모든 노드에 설치되어 있습니다.

```
whchoi98:~/environment/logging $ kubectl -n logging get pods -o wide
NAME               READY   STATUS    RESTARTS   AGE   IP              NODE                                               NOMINATED NODE   READINESS GATES
fluent-bit-8m2m4   1/1     Running   0          64s   10.11.22.53     ip-10-11-16-31.ap-northeast-2.compute.internal     <none>           <none>
fluent-bit-94kp7   1/1     Running   0          63s   10.11.138.76    ip-10-11-146-170.ap-northeast-2.compute.internal   <none>           <none>
fluent-bit-9x5r4   1/1     Running   0          63s   10.11.107.197   ip-10-11-114-132.ap-northeast-2.compute.internal   <none>           <none>
fluent-bit-gxvml   1/1     Running   0          63s   10.11.61.131    ip-10-11-55-30.ap-northeast-2.compute.internal     <none>           <none>
fluent-bit-vxt62   1/1     Running   0          63s   10.11.170.168   ip-10-11-189-67.ap-northeast-2.compute.internal    <none>           <none>
fluent-bit-zn7rs   1/1     Running   0          63s   10.11.79.179    ip-10-11-90-240.ap-northeast-2.compute.internal    <none>           <none>
```

## Kibana 접속.

1.kibana 접속.&#x20;

이제 Kibana에 접속해 봅니다.

```
echo "Kibana URL: https://${ES_ENDPOINT}/_plugin/kibana/
Kibana user: ${ES_DOMAIN_USER}
Kibana password: ${ES_DOMAIN_PASSWORD}"

```

앞서 변수에 저장한 값을 통해 URL, user id, Pwd를 확인하고 , 브라우져에서 접속합니다.

![](<../.gitbook/assets/image (213).png>)

![](<../.gitbook/assets/image (296).png>)

![](<../.gitbook/assets/image (223).png>)

&#x20;index pattern에 아래 값을 입

```
*fluent-bit*
```

![](<../.gitbook/assets/image (364).png>)

Time Filter field name에서 "@timestamp"를 선택하고, "Create Index Pattern"을 선택합니다.

![](<../.gitbook/assets/image (52).png>)

![](<../.gitbook/assets/image (490).png>)







