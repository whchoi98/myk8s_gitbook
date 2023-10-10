# VPC Advanced

## EKS VPC CNI 소개 .&#x20;

Amazon EKS는 Kubernetes용 Amazon VPC 컨테이너 네트워크 인터페이스(CNI) 플러그인을 통해 기본 VPC 네트워킹을 지원합니다. 이 CNI 플러그인을 사용하면 Kubernetes 포드에서 VPC 네트워크에서와 마찬가지로 포드 내 동일한 IP 주소를 보유합니다. CNI 플러그인은 [GitHub](https://github.com/aws/amazon-vpc-cni-k8s)에서 유지 관리하는 오픈 소스 프로젝트입니다. Amazon VPC CNI 플러그인은 AWS의 Amazon EKS 및 자체 관리형 Kubernetes 클러스터에서 사용할 수 있도록 완벽하게 지원됩니다.

Amazon EKS는 공식적으로 [Amazon VPC CNI 플러그인](https://docs.aws.amazon.com/ko\_kr/eks/latest/userguide/pod-networking.html)만 지원합니다. 그러나 Amazon EKS는 업스트림 Kubernetes를 실행하며 Kubernetes 준수 인증을 받았으므로 대체 CNI 플러그인은 Amazon EKS 클러스터에서 작동합니다. 프로덕션 환경에서 대체 CNI 플러그인을 사용하려는 경우 상용 지원을 받거나 오픈 소스 CNI 플러그인 프로젝트를 문제 해결하고 수정을 제공할 수 있는 전문 지식을 내부적으로 보유하는 것이 좋습니다.

| 파트너        | 제품                                            | 설명서                                                                                         |
| ---------- | --------------------------------------------- | ------------------------------------------------------------------------------------------- |
| Tigera     | [Calico](https://www.tigera.io/partners/aws/) | [설치 지침](https://docs.projectcalico.org/getting-started/kubernetes/managed-public-cloud/eks) |
| Isovalent  | [Cilium](https://cilium.io/contact-us-eks/)   | [설치 지침](https://docs.cilium.io/en/v1.7/gettingstarted/k8s-install-eks/)                     |
| Weaveworks | [Weave Net](https://www.weave.works/contact/) | [설치 지침](https://www.weave.works/docs/net/latest/kubernetes/kube-addon/#-installing-on-eks)  |

![](<../.gitbook/assets/image (230).png>)

CNI 플러그인은 Kubernetes 노드에 VPC IP 주소를 할당하고 각 노드의 포드에 대한 필수 네트워킹을 구성하는 역할을 합니다. ​플러그인에는 두 가지 기본 구성 요소가 있습니다.

* L-IPAM 데몬은 탄력적 네트워크 인터페이스와 인스턴스의 연결, 탄력적 네트워크 인터페이스에 보조 IP 주소 할당 및 예약 시 Kubernetes 포드에 할당할 각 노드에 있는 IP 주소의 "웜 풀" 유지 관리를 책임집니다.
* CNI 플러그인 자체는 호스트 네트워크 연결(예: 인터페이스 및 가상 이더넷 페어 구성) 및 포드 네임스페이스에 대한 올바른 인터페이스 추가를 책임집니다.

추가 상세 내용은 github에서 확인 가능합니다. ([https://github.com/aws/amazon-vpc-cni-k8s/blob/master/README.md](https://github.com/aws/amazon-vpc-cni-k8s/blob/master/README.md))

```
maxPods = (number of interfaces - 1) * (max IPv4 addresses per interface - 1) + 2
```

{% hint style="warning" %}
AWS EKS VPC CNI 구성은 ENI에 할당된 Secondary IP 를 Pod가 동일하게 할당받습니다. 따라서 인스턴스 유형별 네트워크 인터페이스당 IP 주소의 한계를 그대로 가지고 갑니다.

[https://docs.aws.amazon.com/ko\_kr/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI](https://docs.aws.amazon.com/ko\_kr/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI)
{% endhint %}

## VPC CNI 모니터링.

아래 주요 명령을 통해서, ENI Secondary IP가 Pod에 매핑된 현황을 볼 수 있습니다.

kubectl을 통한 Pod와 Node간의 연결과 IP 할당을 확인합니다.&#x20;

```
kubectl get pods --all-namespaces -o wide
```

출력결과 예제

```
~/environment $ kubectl get pods --all-namespaces -o wide
NAMESPACE           NAME                                                  READY   STATUS    RESTARTS   AGE     IP              NODE                                               NOMINATED NODE   READINESS GATES
2048-game           2048-deployment-dd74cc68d-2bnxc                       1/1     Running   0          3d19h   10.11.58.105    ip-10-11-55-30.ap-northeast-2.compute.internal     <none>           <none>
```

worker node에서도 Pod별 할당된 내용들을 상세하게 볼 수 있습니다.

```
# VPC CNI 구성 정보를 볼 수 있습니다.
curl http://localhost:61679/v1/networkutils-env-settings | python -m json.tool    
# Pod의 IP mapping 현황을 볼 수 있습니다.
curl http://localhost:61679/v1/pods | python -m json.tool
# ENI에 할당된 주소들과 Pod에 mapping 된 현황을 볼 수 있습니다.
curl http://localhost:61679/v1/enis | python -m json.tool
```

출력 결과 예제는 아래와 같습니다.

```
[ec2-user@ip-10-11-16-31 ~]$ curl http://localhost:61679/v1/networkutils-env-settings | python -m json.tool                                                  
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   247  100   247    0     0   241k      0 --:--:-- --:--:-- --:--:--  241k
{
    "AWS_VPC_CNI_NODE_PORT_SUPPORT": true,
    "AWS_VPC_ENI_MTU": 9001,
    "AWS_VPC_K8S_CNI_CONFIGURE_RPFILTER": true,
    "AWS_VPC_K8S_CNI_CONNMARK": 128,
    "AWS_VPC_K8S_CNI_EXCLUDE_SNAT_CIDRS": null,
    "AWS_VPC_K8S_CNI_EXTERNALSNAT": false,
    "AWS_VPC_K8S_CNI_RANDOMIZESNAT": 1
}
```

```
[ec2-user@ip-10-11-16-31 ~]$ curl http://localhost:61679/v1/enis | python -m json.tool | more
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  3020    0  3020    0     0  2949k      0 --:--:-- --:--:-- --:--:-- 2949k
{
    "AssignedIPs": 13,
    "ENIIPPools": {
        "eni-0487f50f9f0e7924d": {
            "AssignedIPv4Addresses": 0,
            "DeviceNumber": 2,
            "ID": "eni-0487f50f9f0e7924d",
            "IPv4Addresses": {
                "10.11.11.120": {
                    "Address": "10.11.11.120",
                    "Assigned": false,
                    "UnassignedTime": "0001-01-01T00:00:00Z"
                },
이하 생략.

[ec2-user@ip-10-11-16-31 ~]$ curl http://localhost:61679/v1/pods | python -m json.tool | more
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1852  100  1852    0     0  1808k      0 --:--:-- --:--:-- --:--:-- 1808k
{
    "2048-deployment-dd74cc68d-5qw8k_2048-game_0b7b3d2485d7fc13746b6fbd95dc2781683dc6ef0102eab5ab966f550455b711": {
        "DeviceNumber": 0,
        "IP": "10.11.2.40"
    },
```

## 다중 CIDR 주소 할당.

참조 : [https://aws.amazon.com/ko/premiumsupport/knowledge-center/eks-multiple-cidr-ranges/](https://aws.amazon.com/ko/premiumsupport/knowledge-center/eks-multiple-cidr-ranges/)

### 1.VPC ID를 변수 저장.

**VPC\_ID** 변수에 VPC를 연결하기 위 다음 명령을 실행합니다.

```
export VPC_ID=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values=eksworkshop | jq -r '.Vpcs[].VpcId')
```

### 2.CIDR 블록을 연결

VPC에 대해 **100.64.0.0/16** 범위의 CIDR 블록을 연결하려면 다음 명령을 실행합니다.

```
aws ec2 associate-vpc-cidr-block --vpc-id $VPC_ID --cidr-block 100.64.0.0/16
```

다음과 같은 출력 결과를 확인 할 수 있습니다.

```
whchoi98:~/environment $ aws ec2 associate-vpc-cidr-block --vpc-id $VPC_ID --cidr-block 100.64.0.0/16
{
    "CidrBlockAssociation": {
        "AssociationId": "vpc-cidr-assoc-0676cfb3e514a5234",
        "CidrBlock": "100.64.0.0/16",
        "CidrBlockState": {
            "State": "associating"
        }
    },
    "VpcId": "vpc-086186d07739fc568"
}
```

![](<../.gitbook/assets/image (41).png>)

### 3.서브넷 생성.&#x20;

AWS 리전의 모든 가용 영역을 나열하려면 다음 명령을 실행합니다.

```
aws ec2 describe-availability-zones --region ap-northeast-2 --query 'AvailabilityZones[*].ZoneName'
```

서브넷을 추가할 가용 영역을 선택한 다음 해당 가용 영역을 변수에 할당합니다.

```
echo "export AZ1=ap-northeast-2a" | tee -a ~/.bash_profile
echo "export AZ2=ap-northeast-2b" | tee -a ~/.bash_profile
echo "export AZ3=ap-northeast-2c" | tee -a ~/.bash_profile
```

VPC에서 새 CIDR 범위를 사용하여 새 서브넷을 생성하려면 다음 명령을 실행합니다.

```
CUST_SNET1=$(aws ec2 create-subnet --cidr-block 100.64.0.0/19 --vpc-id $VPC_ID --availability-zone $AZ1 | jq -r .Subnet.SubnetId)
CUST_SNET2=$(aws ec2 create-subnet --cidr-block 100.64.32.0/19 --vpc-id $VPC_ID --availability-zone $AZ2 | jq -r .Subnet.SubnetId)
CUST_SNET3=$(aws ec2 create-subnet --cidr-block 100.64.64.0/19 --vpc-id $VPC_ID --availability-zone $AZ3 | jq -r .Subnet.SubnetId)
```

Amazon EKS가 서브넷을 검색할 수 있도록 모든 서브넷에 태그를 지정해야 합니다. 기존에 생성되어 있는 예제를 살펴봅니다.

```
aws ec2 describe-subnets --filters Name=cidr-block,Values=10.11.0.0/19 --output text
```

출력 결과는 아래 같습니다.

```
whchoi98:~/environment $ aws ec2 describe-subnets --filters Name=cidr-block,Values=10.11.0.0/19 --output text                                                
SUBNETS False   ap-northeast-2a apne2-az1       8149    10.11.0.0/19    False   True    909121566064    available       arn:aws:ec2:ap-northeast-2:909121566064:subnet/subnet-0d4864467efbf04a4      subnet-0d4864467efbf04a4        vpc-086186d07739fc568
TAGS    aws:cloudformation:stack-id     arn:aws:cloudformation:ap-northeast-2:909121566064:stack/eksworkshop/3c1848b0-c7f9-11ea-afac-0689b5dc6f6c
TAGS    aws:cloudformation:logical-id   PublicSubnet01
TAGS    Name    eksworkshop-PublicSubnet01
TAGS    kubernetes.io/cluster/eksworkshop       shared
TAGS    aws:cloudformation:stack-name   eksworkshop
TAGS    kubernetes.io/role/elb  1
```

키–값 페어를 설정하여 서브넷에 이름 태그를 추가합니다. 또한 Amazon EKS에서 검색할 서브넷에 태그를 지정합니다.

```
aws ec2 create-tags --resources $CUST_SNET1 --tags Key=Name,Value=eksworkshop-Secondary-PublicSubnet01
aws ec2 create-tags --resources $CUST_SNET1 --tags Key=kubernetes.io/cluster/eksworkshop,Value=shared
aws ec2 create-tags --resources $CUST_SNET1 --tags Key=kubernetes.io/role/elb,Value=1
aws ec2 create-tags --resources $CUST_SNET2 --tags Key=Name,Value=eksworkshop-Secondary-PublicSubnet02
aws ec2 create-tags --resources $CUST_SNET2 --tags Key=kubernetes.io/cluster/eksworkshop,Value=shared
aws ec2 create-tags --resources $CUST_SNET2 --tags Key=kubernetes.io/role/elb,Value=1
aws ec2 create-tags --resources $CUST_SNET3 --tags Key=Name,Value=eksworkshop-Secondary-PublicSubnet03
aws ec2 create-tags --resources $CUST_SNET3 --tags Key=kubernetes.io/cluster/eksworkshop,Value=shared
aws ec2 create-tags --resources $CUST_SNET3 --tags Key=kubernetes.io/role/elb,Value=1

```

아래와 같은 결과를 VPC 대쉬보드에서 확인 할 수 있습니다. 3개의 새로운 서브넷이 생성되고, 태그가 추가되었습니다.

![](<../.gitbook/assets/image (234).png>)

### 4. 서브넷에 라우팅 테이블 연결.

VPC에서 퍼블릭 라우팅 테이블을 찾아서, 해당 라우팅 테이블의 키를 기준으로 Public Routing에 신규 생성된 서브넷을 연결합니다.

```
SNET1=$(aws ec2 describe-subnets --filters Name=cidr-block,Values=10.11.0.0/19 | jq -r '.Subnets[].SubnetId')
RTASSOC_ID=$(aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=$SNET1 | jq -r '.RouteTables[].RouteTableId')
aws ec2 associate-route-table --route-table-id $RTASSOC_ID --subnet-id $CUST_SNET1
aws ec2 associate-route-table --route-table-id $RTASSOC_ID --subnet-id $CUST_SNET2
aws ec2 associate-route-table --route-table-id $RTASSOC_ID --subnet-id $CUST_SNET3

```

정상적으로 Public Routing Table에 추가 되었는지 확인합니다.

![](<../.gitbook/assets/image (360).png>)

### 5.CNI Plugin 구성.

최신 버전의 CNI 플러그인이 있는지 확인하려면 다음 명령을 실행합니다.

```
kubectl describe daemonset aws-node --namespace kube-system | grep Image | cut -d "/" -f 2

```

CNI 플러그인 버전을 다운로드 받습니다. (예제에서는 v1.6을 다운 받습니다.)

```
wget https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/v1.6/aws-k8s-cni.yaml

```

CNI 플러그인에 대한 사용자 지정 네트워크 구성을 활성화하려면 다음 명령을 실행합니다.

```
cd ~/environment/myeks
kubectl apply -f aws-k8s-cni.yaml
```

worker node를 식별하기 위한 **ENIConfig** 라을 추가하려면 다음 명령을 실행합니다.

```
kubectl set env ds aws-node -n kube-system AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true

```

정상적으로 라벨이 추가 되었는지 확인합니다.&#x20;

```
 kubectl describe daemonset aws-node -n kube-system | grep -A5 Environment
 
```

아래와 같은 출력 결과물을 얻을 수 있습니다.

```
whchoi98:~/environment/myeks (master) $ kubectl describe daemonset aws-node -n kube-system | grep -A5 Environment
    Environment:
      AWS_VPC_K8S_CNI_LOGLEVEL:            DEBUG
      AWS_VPC_K8S_CNI_VETHPREFIX:          eni
      AWS_VPC_ENI_MTU:                     9001
      MY_NODE_NAME:                         (v1:spec.nodeName)
      AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG:  true
```

### 6.CRD 생성.&#x20;

[CRD](broken-reference)는 앞서 소개한 Chapter를 확인합니다.&#x20;

ENIConfig CRD에 대한 Customer Resource를 생성합니다. 이때 반드시 ENIConfig CRD가 준비되어 있어야 합니다.

```
kubectl get crd | grep "eni"

```

아래와 같은 출력 결과를 얻을 수 있습니다.

```
whchoi98:~/environment/myeks (master) $ kubectl get crd | grep "eni"
eniconfigs.crd.k8s.amazonaws.com              2020-07-17T07:03:34Z
```

새로 구성할 Subnet들을 출력합니다.

```
aws ec2 describe-subnets  --filters "Name=cidr-block,Values=100.64.*" --query 'Subnets[*].[CidrBlock,SubnetId,AvailabilityZone]' --output table

```

아래와 같이 출력됩니다.

```
whchoi98:~/environment/myeks (master) $ aws ec2 describe-subnets  --filters "Name=cidr-block,Values=100.64.*" --query 'Subnets[*].[CidrBlock,SubnetId,AvailabilityZone]' --output table
-------------------------------------------------------------------
|                         DescribeSubnets                         |
+----------------+----------------------------+-------------------+
|  100.64.32.0/19|  subnet-0aebaccb17574b415  |  ap-northeast-2b  |
|  100.64.0.0/19 |  subnet-0ef2de2d3c6504a3f  |  ap-northeast-2a  |
|  100.64.64.0/19|  subnet-006a77bdca2ec4085  |  ap-northeast-2c  |
+----------------+----------------------------+-------------------+
```

Instance ID와 Mapping되어 있는 Security ID를 출력합니다

```
INSTANCE_IDS=(`aws ec2 describe-instances --query 'Reservations[*].Instances[*].InstanceId' --filters "Name=tag-key,Values=alpha.eksctl.io/nodegroup-name" "Name=tag-value,Values=ng1-public" --output text`)
for i in "${INSTANCE_IDS[@]}"
do
  echo "SecurityGroup for EC2 instance $i ..."
  aws ec2 describe-instances --instance-ids $i | jq -r '.Reservations[].Instances[].SecurityGroups[].GroupId'
done  

```

출력되는 결과는 아래와 같습니다.

```
SecurityGroup for EC2 instance i-0ca4ddf4adea01038 ...
sg-0db3cb147dca2658b
sg-0df26a1d81042cc02
SecurityGroup for EC2 instance i-0c54ecd3e71935c55 ...
sg-0db3cb147dca2658b
sg-0df26a1d81042cc02
SecurityGroup for EC2 instance i-012b9094504eb1038 ...
sg-0db3cb147dca2658b
```

VPC/EC2 대시보드를 통해서도 확인이 가능합니다.

![](<../.gitbook/assets/image (477).png>)



![](<../.gitbook/assets/image (447).png>)

![](<../.gitbook/assets/image (297).png>)

출력된 서브넷과 Security를 Custom Resource로 생성합니다.

eksworkshop-Secondary-PublicSubnet01 를 위한 yaml - group1-pod-netconfig.yaml

```
apiVersion: crd.k8s.amazonaws.com/v1alpha1
kind: ENIConfig
metadata:
 name: group1-pod-netconfig
spec:
 subnet: subnet-0ef2de2d3c6504a3f
 securityGroups:
 - sg-0db3cb147dca2658b
 - sg-0df26a1d81042cc02
 
```

eksworkshop-Secondary-PublicSubnet02 를 위한 yaml - group2-pod-netconfig.yaml

```
apiVersion: crd.k8s.amazonaws.com/v1alpha1
kind: ENIConfig
metadata:
 name: group1-pod-netconfig
spec:
 subnet: subnet-0aebaccb17574b415
 securityGroups:
 - sg-0db3cb147dca2658b
 - sg-0df26a1d81042cc02

```

eksworkshop-Secondary-PublicSubnet03 를 위한 yaml - group3-pod-netconfig.yaml

```
apiVersion: crd.k8s.amazonaws.com/v1alpha1
kind: ENIConfig
metadata:
 name: group1-pod-netconfig
spec:
 subnet: subnet-006a77bdca2ec4085
 securityGroups:
 - sg-0db3cb147dca2658b
 - sg-0df26a1d81042cc02

```

CRD를 적용합니다.

```
kubectl apply -f group1-pod-netconfig.yaml
kubectl apply -f group2-pod-netconfig.yaml
kubectl apply -f group3-pod-netconfig.yaml

```

Custom network config에 Annotate를 적용합니다.

```
kubectl annotate nodes ip-10-11-16-31.ap-northeast-2.compute.internal k8s.amazonaws.com/eniConfig=group1-pod-netconfig
kubectl annotate nodes ip-10-11-55-30.ap-northeast-2.compute.internal k8s.amazonaws.com/eniConfig=group2-pod-netconfig
kubectl annotate nodes ip-10-11-90-240.ap-northeast-2.compute.internal k8s.amazonaws.com/eniConfig=group3-pod-netconfig
```















