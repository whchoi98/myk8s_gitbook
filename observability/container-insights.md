---
description: 'Update : 2021-04-08'
---

# Container Insights

## 소개

CloudWatch Container Insights를 사용해 컨테이너식 애플리케이션 및 마이크로서비스의 지표 및 로그를 수집하고 집계하며 요약할 수 있습니다. Container Insights는 Amazon EC2의 Amazon Elastic Container Service(Amazon ECS), Amazon Elastic Kubernetes Service(Amazon EKS) 및 Kubernetes 플랫폼에서 사용할 수 있고 Amazon ECS 지원은 Fargate에 대한 지원을 포함합니다.

이 지표에는 CPU, 메모리, 디스크, 네트워크 같은 리소스 사용률이 포함되어 있습니다. 또한 Container Insights는 컨테이너 재시작 오류 같은 진단 정보를 제공하여 문제를 격리하고 신속하게 해결할 수 있도록 도와줍니다. Container Insights에서 수집한 지표에 대해 CloudWatch 경보를 설정할 수도 있습니다.

Container Insights에서 수집한 지표는 CloudWatch 자동 대시보드에서 사용할 수 있습니다. CloudWatch Logs Insights를 사용해 컨테이너 성능 및 로그를 분석하고 이와 관련해 발생하는 문제를 해결할 수 있습니다.

운영 데이터는 _성능 로그 이벤트_로서 수집됩니다. 이들은 정형 JSON 스키마를 사용하는 항목이기 때문에 카디널리티가 높은 데이터를 적정 규모로 수집 및 저장할 수 있습니다. 이러한 데이터를 토대로 CloudWatch는 집계 지표를 클러스터, 노드, Pod, 작업 및 서비스 수준에서 CloudWatch 지표로 생성합니다.

Container Insights에서 수집된 지표는 사용자 지정 지표로 청구됩니다. CloudWatch 요금에 대한 자세한 내용은 [Amazon CloudWatch 요금](https://aws.amazon.com/cloudwatch/pricing/)을 참조하십시오.

Amazon EKS 및 Kubernetes에서 Container Insights는 컨테이너화된 버전의 CloudWatch 에이전트를 사용하여 클러스터에서 실행 중인 모든 컨테이너를 검색합니다. 이어서 성능 스택의 모든 계층에서 성능 데이터를 수집합니다.

랩에서는 아래와 같은 도구들을 사용하여, 랩을 구성합니다.

* **Helm** : 클러스터에 Wordpress를 설치합니다. (PHP/Nginix 구성을 하셔도 무방합니다.)
* **CloudWatch Container Insights** : 클러스터에서 로그 및 지표를 수집합니다.
* **Siege** : Wordpress 및 EKS Cluster를 로드 테스트합니다.
* **CloudWatch Container Insights 대시 보드** : 컨테이너 성능 및 로드에 대 시각화합니다.
* **CloudWatch 지표** : 워드 프레스 포드에 과부하가 걸렸을 때 알람을 설정합니다.

## Pod 배포 .

### 1.WordPress/DB 설치

[Helm Chart](../eks-2/helm.md#4-helm-nginx)에서 등록된 Bitnami Repo에서 설치합니다.&#x20;

```
helm repo list

```

Bitnami Repo가 등록되어 있지 않다면 아래와 같이 먼저 등록을 합니다.

```
helm repo add bitnami https://charts.bitnami.com/bitnami

```

아래와 같이 새로운 namespace를 만들고, helm을 통해 설치를 시작합니다.

```
kubectl create namespace wordpress
helm -n wordpress install my-wordpress bitnami/wordpress

```

아래와 같이 배포 상태를 확인해 봅니다.

```
kubectl -n wordpress rollout status deployment my-wordpress 
kubectl -n wordpress get all

```

Helm Chart를 통해 실행된, 내용을 살펴 봅니다.

```
kubectl -n wordpress describe deployments my-wordpress

```

Wordpress 노출 주소를 확인하기 위해, Service External-IP를 확인해 봅니다.

```
kubectl -n wordpress get svc

```

아래와 같은 출력 결과를 확인 할 수 있습니다.

```
whchoi98:~/environment $ kubectl -n wordpress get svc
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP                                                                    PORT(S)                      AGE
my-wordpress           LoadBalancer   172.20.162.6   ac4026294517a4cd69df77ce5ebc4547-1003101175.ap-northeast-2.elb.amazonaws.com   80:31492/TCP,443:32711/TCP   8m3s
my-wordpress-mariadb   ClusterIP      172.20.3.221   <none>   
```

### 2.WordPress Page  접속 확인.&#x20;

External IP (ELB 주소)로 접속합니다.

![](<../.gitbook/assets/image (320).png>)

아래와 같이 URL에 대해 변수에 저장합니다.

```
# Wordpress URL
export SERVICE_URL=$(kubectl get svc -n wordpress my-wordpress --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
# Wordpress admin URL
export ADMIN_URL="http://$SERVICE_URL/admin"
# Wordpress password
export ADMIN_PASSWORD=$(kubectl get secret --namespace wordpress my-wordpress -o jsonpath="{.data.wordpress-password}" | base64 --decode)

echo "Admin URL: http://$SERVICE_URL/admin
Username: user
Password: $ADMIN_PASSWORD
"

```

Admin URL에 접속하고 출력된 결과를 입력합니다.

![](<../.gitbook/assets/image (43).png>)

![](<../.gitbook/assets/image (139).png>)

## Container Insight 설치.

### 1.Worker Node IAM 역할에 정책 추가.&#x20;

Cloudwatch를 통해 수집하려면, EKS Cluster에 Cloudwatch agent를 설치해야 합니다. 이를 위해 worker node에 필요한 IAM 역할에 필요한 정책을 추가해야 합니다. &#x20;

Role을 확인하기 위해 아래 과정을 수행합니다.

```
STACK_NAME=$(eksctl get nodegroup --cluster eksworkshop -o json | jq -r '.[].StackName')
echo $STACK_NAME
```

Role이 Private/Public 2개의 노드입니다. 그중 public node의 role을 "--stack-name" 뒤의 값에 입력합니다. 해당 값을 사용자 bash profile에 입력하고, role 이름을 다시 확인합니다.

```
PUB_ROLE_NAME=$(aws cloudformation describe-stack-resources --stack-name eksctl-eksworkshop-nodegroup-ng-public-01 | jq -r '.StackResources[] | select(.ResourceType=="AWS::IAM::Role") | .PhysicalResourceId')
MGMD_PUB_ROLE_NAME=$(aws cloudformation describe-stack-resources --stack-name eksctl-eksworkshop-nodegroup-managed-ng-public-01 | jq -r '.StackResources[] | select(.ResourceType=="AWS::IAM::Role") | .PhysicalResourceId')

echo "export PUB_ROLE_NAME=${PUB_ROLE_NAME}" | tee -a ~/.bash_profile
echo "export MGMD_PUB_ROLE_NAME=${PUB_ROLE_NAME}" | tee -a ~/.bash_profile

test -n "$PUB_ROLE_NAME" && echo PUB_ROLE_NAME is "$PUB_ROLE_NAME" || echo PUB_ROLE_NAME is not set
test -n "$MGMD_PUB_ROLE_NAME" && echo MGMD_PUB_ROLE_NAME is "$MGMD_PUB_ROLE_NAME" || echo MGMD_PUB_ROLE_NAME is not set

PRI_ROLE_NAME=$(aws cloudformation describe-stack-resources --stack-name eksctl-eksworkshop-nodegroup-ng-private-01 | jq -r '.StackResources[] | select(.ResourceType=="AWS::IAM::Role") | .PhysicalResourceId')
MGMD_PRI_ROLE_NAME=$(aws cloudformation describe-stack-resources --stack-name eksctl-eksworkshop-nodegroup-managed-ng-private-01 | jq -r '.StackResources[] | select(.ResourceType=="AWS::IAM::Role") | .PhysicalResourceId')

echo "export PRI_ROLE_NAME=${PRI_ROLE_NAME}" | tee -a ~/.bash_profile
echo "export MGMD_PRO_ROLE_NAME=${MGMD_PRI_ROLE_NAME}" | tee -a ~/.bash_profile

test -n "$PRI_ROLE_NAME" && echo PRI_ROLE_NAME is "$PRI_ROLE_NAME" || echo PRI_ROLE_NAME is not set
test -n "$MGMD_PRI_ROLE_NAME" && echo PRI_ROLE_NAME is "$MGMD_PRI_ROLE_NAME" || echo MGMD_PRI_ROLE_NAME is not set

```

정책(Policy)를 Worker node 의 IAM 역할(Role)에 연결합니다.

```
aws iam attach-role-policy \
  --role-name $PUB_ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

aws iam attach-role-policy \
  --role-name $PRI_ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

aws iam attach-role-policy \
  --role-name $MGMD_PUB_ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

aws iam attach-role-policy \
  --role-name $MGMD_PRI_ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
```

위의 과정을 IAM 대시보드에서 수행해도 됩니다.

정상적으로 연결되었는지 확인합니다.

```
aws iam list-attached-role-policies --role-name $PUB_ROLE_NAME | grep CloudWatchAgentServerPolicy || echo 'Policy not found'
aws iam list-attached-role-policies --role-name $PRI_ROLE_NAME | grep CloudWatchAgentServerPolicy || echo 'Policy not found'
aws iam list-attached-role-policies --role-name $MGMD_PUB_ROLE_NAME | grep CloudWatchAgentServerPolicy || echo 'Policy not found'
aws iam list-attached-role-policies --role-name $MGMD_PRI_ROLE_NAME | grep CloudWatchAgentServerPolicy || echo 'Policy not found'
```

아래와 같은 결과를 확인 할 수 있습니다.

```
whchoi98:~/environment $ aws iam list-attached-role-policies --role-name $PUB_ROLE_NAME | grep CloudWatchAgentServerPolicy || echo 'Policy not found'
            "PolicyName": "CloudWatchAgentServerPolicy",
            "PolicyArn": "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
```

![](<../.gitbook/assets/image (138).png>)

### 2. Container Insight 설치

다음과 같은 내용을 배포하고 구성합니다.

* "Amazon-Cloudwatch" namespace를 만듭니다.
* 2개의 DaemonSet을 만듭니다. (Cloudwatch-agent, fluentd)
  * Cloudwatch-agent : Metric을 Cloudwatch로 전송
  * fluentd : log를 Cloudwatch에 전송.
* 2개 DaemonSet을 위한 보안 정책적용. (SecurityAccount, ClusterRole, ClusterRoleBinding)
* 2개 DaemonSet을 위한 ConfigMap

```
curl -s https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluentd-quickstart.yaml | sed "s/{{cluster_name}}/eksworkshop/;s/{{region_name}}/${AWS_REGION}/" | kubectl apply -f -
kubectl -n amazon-cloudwatch get daemonsets

```

아래와 같은 결과를 확인 할 수 있습니다.

```
whchoi:~/environment/myeks (master) $ kubectl -n amazon-cloudwatch get daemonsets
NAME                 DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
cloudwatch-agent     12        12        12      12           12          <none>          27s
fluentd-cloudwatch   12        12        12      12           12          <none>          27s
```

CloudWatch 대쉬보드 - Container Insight에 접속해서 각 주요 기능을 살펴 봅니다.

Cloudwatch - Container Insight - Resource

![](<../.gitbook/assets/image (261).png>)

Container Insight - container map

![](<../.gitbook/assets/image (286).png>)

![](<../.gitbook/assets/image (204).png>)

Container Insight - Performance Dashboards

![](<../.gitbook/assets/image (460).png>)



## 3.Load Test 구성.

siege 도구를 통해 HTTP 부하를 테스트 합니다.&#x20;

{% hint style="info" %}
Siege([https://www.joedog.org/siege-home/](https://www.joedog.org/siege-home/))는 HTTP 부하 테스트와 벤치마킹 유틸리티 입니다. 한번에 복수개의 URL 로 테스트가 가능하고, Basic 인증을 지원하며 HTTP와 HTTPS 프로토콜로 테스트가 가능하는 등 ApacheBench 의 제한을 어느정도 해소 해 줍니다. 또 ApacheBench 와 비슷한 인터페이스를 가지고 있어 다른 툴과의 연계가 자유로운 편입니다. 하지만 Thread 기반으로 구현되어 있어 동시성에 제한이 있으며, 미미할 수 있지만 Context Switching 등으로 인하여 성능에도 영향이 있습니다.
{% endhint %}

아래와 같이 Cloud9 IDE에 설치합니다. 외부 다른 리눅스나 mac os에서 설치해도 됩니다.

```
sudo yum -y install siege

```

아래 결과를 통해 표기된 웹페이지로 URL로 부하를 줍니다.

```
kubectl -n wordpress get svc my-wordpress

```

또는 아래 결과를 통해 표기된 웹페이지 URL을 변수에 저장하고 , 사용합니다.

```
export my_wordpress_url=$(kubectl -n wordpress get svc my-wordpress -o jsonpath="{.status.loadBalancer.ingress[].hostname}")

```

출력 결과는 아래와 같습니다. EXTERNAL-IP 주소입니다.

```
whchoi98:~ $ kubectl -n wordpress get svc my-wordpress
NAME           TYPE           CLUSTER-IP     EXTERNAL-IP                                                                    PORT(S)                      AGE
my-wordpress   LoadBalancer   172.20.162.6   ac4026294517a4cd69df77ce5ebc4547-1003101175.ap-northeast-2.elb.amazonaws.com   80:31492/TCP,443:32711/TCP   138m

whchoi98:~ $ export my_wordpress_url=$(kubectl -n wordpress get svc my-wordpress -o jsonpath="{.status.loadBalancer.ingress[].hostname}")                                          
whchoi98:~ $ echo $my_wordpress_url 
ac4026294517a4cd69df77ce5ebc4547-1003101175.ap-northeast-2.elb.amazonaws.com
```

siege는 아래와 같은 옵션으로 부하를 줄 수 있습니다.

> &#x20;\-c : 동시 접속 유 , -t: 부하 시간 -r : 반복 횟수 , -d: 요청시 Delay time , -d : 요청결과 출력, -q: 실시간 수행 출력을 표시 하지 않습니다.

아래와 같이 Load를 발생 시킵니다.

```
siege -c 200 -i http://$my_wordpress_url --time=30M -q 
```



## 메트릭과 로그 수집.

### 1.Metic 모니터링.&#x20;

CloudWatch - Container Insights 에서 다양한 지표를 확인해 봅니다.

![](<../.gitbook/assets/image (411).png>)

![](<../.gitbook/assets/image (209).png>)

my-wordpress pod의 성능 지표를 5분 단위로 확인해 봅니다.

![](<../.gitbook/assets/image (390).png>)

my-wordpress service의 성능 지표를 5분 단위로 확인해 봅니다.

![](<../.gitbook/assets/image (386).png>)

### 2.Pod Logging 확인

wordpress pod에 대한 application log와 performace log를 확인합니다.

Cloudwatch - Performanc monitoring - EKS Pods - filter : my-wordpress - 컨테이너 성능 wordpress 선

![](<../.gitbook/assets/image (263).png>)

Pod의 application log를 확인하기 위해서, Query 를 실행합니다.

![](<../.gitbook/assets/image (414).png>)

![](<../.gitbook/assets/image (478).png>)

Pod performance log를 확인하기 위해서, Query 를 실행합니다.

![](<../.gitbook/assets/image (81).png>)

![](<../.gitbook/assets/image (383).png>)

{% hint style="info" %}
Fluentd 는 JSON 파일을 디버깅을 위해 손쉽게 파싱을 하거나, 사용자가 정의한 어플리케이션 대시 보드를 만들 수 있도록 다양한 필드를 확인 할 수 있습니다.
{% endhint %}

## 5.CloudWatch 알람 구성.

CloudWatch의 뛰어난 기능 중 하나는 사전에 정의한 metric 값을 위반하게 되면 다양한 방법으로 알림을 생성할 수 있다는 것입니다.&#x20;

CloudWatch Container Insight를 통해서, CPU 사용률에 대한 알림을 설정하는 것을 시험해 봅니다.



