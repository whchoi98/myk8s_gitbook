# Container Insights

## 소개

CloudWatch Container Insights를 사용해 컨테이너식 애플리케이션 및 마이크로서비스의 지표 및 로그를 수집하고 집계하며 요약할 수 있습니다. Container Insights는 Amazon EC2의 Amazon Elastic Container Service\(Amazon ECS\), Amazon Elastic Kubernetes Service\(Amazon EKS\) 및 Kubernetes 플랫폼에서 사용할 수 있고 Amazon ECS 지원은 Fargate에 대한 지원을 포함합니다.

이 지표에는 CPU, 메모리, 디스크, 네트워크 같은 리소스 사용률이 포함되어 있습니다. 또한 Container Insights는 컨테이너 재시작 오류 같은 진단 정보를 제공하여 문제를 격리하고 신속하게 해결할 수 있도록 도와줍니다. Container Insights에서 수집한 지표에 대해 CloudWatch 경보를 설정할 수도 있습니다.

Container Insights에서 수집한 지표는 CloudWatch 자동 대시보드에서 사용할 수 있습니다. CloudWatch Logs Insights를 사용해 컨테이너 성능 및 로그를 분석하고 이와 관련해 발생하는 문제를 해결할 수 있습니다.

운영 데이터는 _성능 로그 이벤트_로서 수집됩니다. 이들은 정형 JSON 스키마를 사용하는 항목이기 때문에 카디널리티가 높은 데이터를 적정 규모로 수집 및 저장할 수 있습니다. 이러한 데이터를 토대로 CloudWatch는 집계 지표를 클러스터, 노드, Pod, 작업 및 서비스 수준에서 CloudWatch 지표로 생성합니다.

Container Insights에서 수집된 지표는 사용자 지정 지표로 청구됩니다. CloudWatch 요금에 대한 자세한 내용은 [Amazon CloudWatch 요금](https://aws.amazon.com/cloudwatch/pricing/)을 참조하십시오.

Amazon EKS 및 Kubernetes에서 Container Insights는 컨테이너화된 버전의 CloudWatch 에이전트를 사용하여 클러스터에서 실행 중인 모든 컨테이너를 검색합니다. 이어서 성능 스택의 모든 계층에서 성능 데이터를 수집합니다.

랩에서는 아래와 같은 도구들을 사용하여, 랩을 구성합니다.

* **Helm** : 클러스터에 Wordpress를 설치합니다. \(PHP/Nginix 구성을 하셔도 무방합니다.\)
* **CloudWatch Container Insights** : 클러스터에서 로그 및 지표를 수집합니다.
* **Siege** : Wordpress 및 EKS Cluster를 로드 테스트합니다.
* **CloudWatch Container Insights 대시 보드** : 컨테이너 성능 및 로드에 대 시각화합니다.
* **CloudWatch 지표** : 워드 프레스 포드에 과부하가 걸렸을 때 알람을 설정합니다.

## Pod 배포 .

### 1.WordPress/DB 설치

[Helm Chart](helm.md#4-helm-nginx)에서 등록된 Bitnami Repo에서 설치합니다. 

```text
helm repo list

```

Bitnami Repo가 등록되어 있지 않다면 아래와 같이 먼저 등록을 합니다.

```text
helm repo add bitnami https://charts.bitnami.com/bitnami

```

아래와 같이 새로운 namespace를 만들고, helm을 통해 설치를 시작합니다.

```text
kubectl create namespace wordpress
helm -n wordpress install my-wordpress bitnami/wordpress

```

dkfo아래와 같이 배포 상태를 확인해 봅니다.

```text
kubectl -n wordpress rollout status deployment my-wordpress 
kubectl -n wordpress get all

```

Helm Chart를 통해 실행된, 내용을 살펴 봅니다.

```text
kubectl -n wordpress describe deployments my-wordpress

```

Wordpress 노출 주소를 확인하기 위해, Service External-IP를 확인해 봅니다.

```text
kubectl -n wordpress get svc

```

아래와 같은 출력 결과를 확인 할 수 있습니다.

```text
whchoi98:~/environment $ kubectl -n wordpress get svc
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP                                                                    PORT(S)                      AGE
my-wordpress           LoadBalancer   172.20.162.6   ac4026294517a4cd69df77ce5ebc4547-1003101175.ap-northeast-2.elb.amazonaws.com   80:31492/TCP,443:32711/TCP   8m3s
my-wordpress-mariadb   ClusterIP      172.20.3.221   <none>   
```

### 2.WordPress Page  접속 확인. 

External IP \(ELB 주소\)로 접속합니다.

![](../.gitbook/assets/image%20%2899%29.png)

아래와 같이 URL에 대해 변수에 저장합니다.

```text
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

![](../.gitbook/assets/image%20%2894%29.png)

![](../.gitbook/assets/image%20%28101%29.png)

## Container Insight 설치.

### 1.Worker Node IAM 역할에 정책 추가. 

Cloudwatch를 통해 수집하려면, EKS Cluster에 Cloudwatch agent를 설치해야 합니다. 이를 위해 worker node에 필요한 IAM 역할에 필요한 정책을 추가해야 합니다.  

Role을 확인하기 위해 아래 과정을 수행합니다.

```text
STACK_NAME=$(eksctl get nodegroup --cluster eksworkshop-eksctl -o json | jq -r '.[].StackName')
echo $STACK_NAME
```

Role이 Private/Public 2개의 노드입니다. 그중 public node의 role을 "--stack-name" 뒤의 값에 입력합니다. 해당 값을 사용자 bash profile에 입력하고, role 이름을 다시 확인합니다.

```text
PUB_ROLE_NAME=$(aws cloudformation describe-stack-resources --stack-name eksctl-eksworkshop-nodegroup-ng1-public | jq -r '.StackResources[] | select(.ResourceType=="AWS::IAM::Role") | .PhysicalResourceId')
echo "export PUB_ROLE_NAME=${PUB_ROLE_NAME}" | tee -a ~/.bash_profile
test -n "$PUB_ROLE_NAME" && echo PUB_ROLE_NAME is "$PUB_ROLE_NAME" || echo PUB_ROLE_NAME is not set

```

정책\(Policy\)를 Worker node 의 IAM 역할\(Role\)에 연결합니다.

```text
aws iam attach-role-policy \
  --role-name $PUB_ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

```

위의 과정을 IAM 대시보드에서 수행해도 됩니다.

정상적으로 연결되었는지 확인합니다.

```text
aws iam list-attached-role-policies --role-name $PUB_ROLE_NAME | grep CloudWatchAgentServerPolicy || echo 'Policy not found'

```

아래와 같은 결과를 확인 할 수 있습니다.

```text
whchoi98:~/environment $ aws iam list-attached-role-policies --role-name $PUB_ROLE_NAME | grep CloudWatchAgentServerPolicy || echo 'Policy not found'
            "PolicyName": "CloudWatchAgentServerPolicy",
            "PolicyArn": "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
```

![](../.gitbook/assets/image%20%2897%29.png)

### 2. Container Insight 설치

다음과 같은 내용을 배포하고 구성합니다.

* "Amazon-Cloudwatch" namespace를 만듭니다.
* 2개의 DaemonSet을 만듭니다. \(Cloudwatch-agent, fluentd\)
  * Cloudwatch-agent : Metric을 Cloudwatch로 전송
  * fluentd : log를 Cloudwatch에 전송.
* 2개 DaemonSet을 위한 보안 정책적용. \(SecurityAccount, ClusterRole, ClusterRoleBinding\)
* 2개 DaemonSet을 위한 ConfigMap

```text
curl -s https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluentd-quickstart.yaml | sed "s/{{cluster_name}}/eksworkshop/;s/{{region_name}}/${AWS_REGION}/" | kubectl apply -f -

```

또는 아래와 같이 로컬의 파일을 수정해서 적용합니다.

```text
cd ~/environment/myeks/
sed "s//eksworkshop/;s//${AWS_REGION}/" cwagent-fluentd-quickstart.yaml | kubectl apply -f -
kubectl -n amazon-cloudwatch get daemonsets

```

아래와 같은 결과를 확인 할 수 있습니다.

```text
whchoi98:~/environment/myeks (master) $ kubectl -n amazon-cloudwatch get daemonsets
NAME                 DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
cloudwatch-agent     6         6         6       6            6           <none>          28s
fluentd-cloudwatch   6         6         5       6            5           <none>          28s
```

CloudWatch 대쉬보드 - Container Insight에 접속해서 각 주요 기능을 살펴 봅니다.

Container Insight - Resource

![](../.gitbook/assets/image%20%2896%29.png)

Container Insight - container map

![](../.gitbook/assets/image%20%28100%29.png)

![](../.gitbook/assets/image%20%2895%29.png)

Container Insight - Performance Dashboards

![](../.gitbook/assets/image%20%2898%29.png)



## 3.Load Test 구성.

siege 도구를 통해 HTTP 부하를 테스트 합니다. 

{% hint style="info" %}
Siege\([https://www.joedog.org/siege-home/](https://www.joedog.org/siege-home/)\)는 HTTP 부하 테스트와 벤치마킹 유틸리티 입니다. 한번에 복수개의 URL 로 테스트가 가능하고, Basic 인증을 지원하며 HTTP와 HTTPS 프로토콜로 테스트가 가능하는 등 ApacheBench 의 제한을 어느정도 해소 해 줍니다. 또 ApacheBench 와 비슷한 인터페이스를 가지고 있어 다른 툴과의 연계가 자유로운 편입니다. 하지만 Thread 기반으로 구현되어 있어 동시성에 제한이 있으며, 미미할 수 있지만 Context Switching 등으로 인하여 성능에도 영향이 있습니다.
{% endhint %}

아래와 같이 Cloud9 IDE에 설치합니다. 외부 다른 리눅스나 mac os에서 설치해도 됩니다.

```text
sudo yum -y install siege

```

아래 결과를 통해 표기된 웹페이지로 URL로 부하를 줍니다.

```text
kubectl -n wordpress get svc my-wordpress

```

또는 아래 결과를 통해 표기된 웹페이지 URL을 변수에 저장하고 , 사용합니다.

```text
export my_wordpress_url=$(kubectl -n wordpress get svc my-wordpress -o jsonpath="{.status.loadBalancer.ingress[].hostname}")

```

출력 결과는 아래와 같습니다. EXTERNAL-IP 주소입니다.

```text
whchoi98:~ $ kubectl -n wordpress get svc my-wordpress
NAME           TYPE           CLUSTER-IP     EXTERNAL-IP                                                                    PORT(S)                      AGE
my-wordpress   LoadBalancer   172.20.162.6   ac4026294517a4cd69df77ce5ebc4547-1003101175.ap-northeast-2.elb.amazonaws.com   80:31492/TCP,443:32711/TCP   138m

whchoi98:~ $ export my_wordpress_url=$(kubectl -n wordpress get svc my-wordpress -o jsonpath="{.status.loadBalancer.ingress[].hostname}")                                          
whchoi98:~ $ echo $my_wordpress_url 
ac4026294517a4cd69df77ce5ebc4547-1003101175.ap-northeast-2.elb.amazonaws.com
```

siege는 아래와 같은 옵션으로 부하를 줄 수 있습니다.

> -c : 동시 접속 유 , -t: 부하 시간 -r : 반복 횟수 , -d: 요청시 Delay time , -d : 요청결과 출력



## 4.메트릭과 로그 수집.

## 5.CloudWatch 알람 구성.



