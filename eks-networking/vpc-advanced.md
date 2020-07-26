# VPC Advanced

## EKS VPC CNI 소개 . 

Amazon EKS는 Kubernetes용 Amazon VPC 컨테이너 네트워크 인터페이스\(CNI\) 플러그인을 통해 기본 VPC 네트워킹을 지원합니다. 이 CNI 플러그인을 사용하면 Kubernetes 포드에서 VPC 네트워크에서와 마찬가지로 포드 내 동일한 IP 주소를 보유합니다. CNI 플러그인은 [GitHub](https://github.com/aws/amazon-vpc-cni-k8s)에서 유지 관리하는 오픈 소스 프로젝트입니다. Amazon VPC CNI 플러그인은 AWS의 Amazon EKS 및 자체 관리형 Kubernetes 클러스터에서 사용할 수 있도록 완벽하게 지원됩니다.

Amazon EKS는 공식적으로 [Amazon VPC CNI 플러그인](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/pod-networking.html)만 지원합니다. 그러나 Amazon EKS는 업스트림 Kubernetes를 실행하며 Kubernetes 준수 인증을 받았으므로 대체 CNI 플러그인은 Amazon EKS 클러스터에서 작동합니다. 프로덕션 환경에서 대체 CNI 플러그인을 사용하려는 경우 상용 지원을 받거나 오픈 소스 CNI 플러그인 프로젝트를 문제 해결하고 수정을 제공할 수 있는 전문 지식을 내부적으로 보유하는 것이 좋습니다.

| 파트너 | 제품 | 설명서 |
| :--- | :--- | :--- |
| Tigera | [Calico](https://www.tigera.io/partners/aws/) | [설치 지침](https://docs.projectcalico.org/getting-started/kubernetes/managed-public-cloud/eks) |
| Isovalent | [Cilium](https://cilium.io/contact-us-eks/) | [설치 지침](https://docs.cilium.io/en/v1.7/gettingstarted/k8s-install-eks/) |
| Weaveworks | [Weave Net](https://www.weave.works/contact/) | [설치 지침](https://www.weave.works/docs/net/latest/kubernetes/kube-addon/#-installing-on-eks) |

![](../.gitbook/assets/image%20%28110%29.png)

CNI 플러그인은 Kubernetes 노드에 VPC IP 주소를 할당하고 각 노드의 포드에 대한 필수 네트워킹을 구성하는 역할을 합니다. ​플러그인에는 두 가지 기본 구성 요소가 있습니다.

* L-IPAM 데몬은 탄력적 네트워크 인터페이스와 인스턴스의 연결, 탄력적 네트워크 인터페이스에 보조 IP 주소 할당 및 예약 시 Kubernetes 포드에 할당할 각 노드에 있는 IP 주소의 "웜 풀" 유지 관리를 책임집니다.
* CNI 플러그인 자체는 호스트 네트워크 연결\(예: 인터페이스 및 가상 이더넷 페어 구성\) 및 포드 네임스페이스에 대한 올바른 인터페이스 추가를 책임집니다.

추가 상세 내용은 github에서 확인 가능합니다. \([https://github.com/aws/amazon-vpc-cni-k8s/blob/master/README.md](https://github.com/aws/amazon-vpc-cni-k8s/blob/master/README.md)\)

```text
maxPods = (number of interfaces - 1) * (max IPv4 addresses per interface - 1) + 2
```

{% hint style="warning" %}
AWS EKS VPC CNI 구성은 ENI에 할당된 Secondary IP 를 Pod가 동일하게 할당받습니다. 따라서 인스턴스 유형별 네트워크 인터페이스당 IP 주소의 한계를 그대로 가지고 갑니다.

[https://docs.aws.amazon.com/ko\_kr/AWSEC2/latest/UserGuide/using-eni.html\#AvailableIpPerENI](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI)
{% endhint %}

## VPC CNI 모니터링.

아래 주요 명령을 통해서, ENI Secondary IP가 Pod에 매핑된 현황을 볼 수 있습니다.

kubectl을 통한 Pod와 Node간의 연결과 IP 할당을 확인합니다. 

```text
kubectl get pods --all-namespaces -o wide
```

출력결과 예제

```text
whchoi98:~/environment $ kubectl get pods --all-namespaces -o wide
NAMESPACE           NAME                                                  READY   STATUS    RESTARTS   AGE     IP              NODE                                               NOMINATED NODE   READINESS GATES
2048-game           2048-deployment-dd74cc68d-2bnxc                       1/1     Running   0          3d19h   10.11.58.105    ip-10-11-55-30.ap-northeast-2.compute.internal     <none>           <none>
```

worker node에서도 Pod별 할당된 내용들을 상세하게 볼 수 있습니다.

```text
# VPC CNI 구성 정보를 볼 수 있습니다.
curl http://localhost:61679/v1/networkutils-env-settings | python -m json.tool    
# Pod의 IP mapping 현황을 볼 수 있습니다.
curl http://localhost:61679/v1/pods | python -m json.tool
# ENI에 할당된 주소들과 Pod에 mapping 된 현황을 볼 수 있습니다.
curl http://localhost:61679/v1/enis | python -m json.tool
```

출력 결과 예제는 아래와 같습니다.

```text
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

```text
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

```text
echo "export VPC_ID=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values=eksworkshop | jq -r '.Vpcs[].VpcId')" | tee -a ~/.bash_profile
```



```text
aws ec2 associate-vpc-cidr-block --vpc-id $VPC_ID --cidr-block 100.64.0.0/16
```



```text
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

![](../.gitbook/assets/image%20%28107%29.png)



```text
aws ec2 describe-availability-zones --region ap-northeast-2 --query 'AvailabilityZones[*].ZoneName'
```

```text
echo "export AZ1=ap-northeast-2a" | tee -a ~/.bash_profile
echo "export AZ2=ap-northeast-2b" | tee -a ~/.bash_profile
echo "export AZ3=ap-northeast-2c" | tee -a ~/.bash_profile
```



```text
CUST_SNET1=$(aws ec2 create-subnet --cidr-block 100.64.0.0/19 --vpc-id $VPC_ID --availability-zone $AZ1 | jq -r .Subnet.SubnetId)
CUST_SNET2=$(aws ec2 create-subnet --cidr-block 100.64.32.0/19 --vpc-id $VPC_ID --availability-zone $AZ2 | jq -r .Subnet.SubnetId)
CUST_SNET3=$(aws ec2 create-subnet --cidr-block 100.64.64.0/19 --vpc-id $VPC_ID --availability-zone $AZ3 | jq -r .Subnet.SubnetId)
```



```text
aws ec2 describe-subnets --filters Name=cidr-block,Values=10.11.0.0/19 --output text
```



```text
whchoi98:~/environment $ aws ec2 describe-subnets --filters Name=cidr-block,Values=10.11.0.0/19 --output text                                                
SUBNETS False   ap-northeast-2a apne2-az1       8149    10.11.0.0/19    False   True    909121566064    available       arn:aws:ec2:ap-northeast-2:909121566064:subnet/subnet-0d4864467efbf04a4      subnet-0d4864467efbf04a4        vpc-086186d07739fc568
TAGS    aws:cloudformation:stack-id     arn:aws:cloudformation:ap-northeast-2:909121566064:stack/eksworkshop/3c1848b0-c7f9-11ea-afac-0689b5dc6f6c
TAGS    aws:cloudformation:logical-id   PublicSubnet01
TAGS    Name    eksworkshop-PublicSubnet01
TAGS    kubernetes.io/cluster/eksworkshop       shared
TAGS    aws:cloudformation:stack-name   eksworkshop
TAGS    kubernetes.io/role/elb  1
```



```text
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



![](../.gitbook/assets/image%20%2895%29.png)



```text
SNET1=$(aws ec2 describe-subnets --filters Name=cidr-block,Values=10.11.0.0/19 | jq -r '.Subnets[].SubnetId')
RTASSOC_ID=$(aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=$SNET1 | jq -r '.RouteTables[].RouteTableId')

```



