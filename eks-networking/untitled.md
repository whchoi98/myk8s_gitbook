---
description: 'Update : 2021-09-27'
---

# Multus

## Multus 소개&#x20;

[Multus](https://github.com/Intel-Corp/multus-cni)는 쿠버네티스의 CRD 기반 네트워크 오브젝트를 사용하여 쿠버네티스에서 멀티 네트워킹 기능을 지원하는 멀티 CNI 플러그인입니다.

Multus는 CNI 명세를 구현하는 모든 [레퍼런스 플러그인](https://github.com/containernetworking/plugins) 및 3rd Party 플러그인을 지원합니. 또한, Multus는 쿠버네티스의 클라우드 네이티브 애플리케이션과 NFV 기반 애플리케이션을 통해 쿠버네티스의 [SRIOV](https://github.com/hustcat/sriov-cni), [DPDK](https://github.com/Intel-Corp/sriov-cni), [OVS-DPDK 및 VPP](https://github.com/intel/vhost-user-net-plugin) 워크로드를 지원합니다.

Multus는 Pod에 멀 네트워크 인터페이스를 첨부할 수 있는 Kubernetes용 오픈 소스 CNI 플러그인입니다. Multus는 메타 플러그인을 기반으로 Pod에 첨부된 다중 네트워크 인터페이스를 작동하는 추가 CNI 플러그인을 지원합니다. 다중 인터페이스를 포함한 Pod가 일반적으로 필요한 사용 사례에는 Kubernetes에 대한 5G 및 스트리밍 네크워크 등이 있습니다. EKS가 지원하는 Multus를 사용하여 이러한 환경에 걸쳐 어드밴스드 네트워킹을 활성화함으로써 사용자에게 고품질 콘텐츠를 전달하는 컨테이너화 네트워크 기능을 실행할 수 있습니다.

## Multus 구성을 위한 사전 준비

이 랩에서는 아래에서 처럼 Multus 구성을 위한 EKS Cluster를 구성합니다.&#x20;

![](<../.gitbook/assets/image (468).png>)

### Task1. Cloud9 구성

AWS 서비스에서 Cloud9을 선택하고, `"Environments"`를 설정합니다.

Cloud9 의 이름과 Description을 설정합니다.

![](<../.gitbook/assets/image (273).png>)

인스턴스 타입과 운영체제, 그리고 절전모드 환경을 선택합니다. 절전모드 환경은 기본 30분입니다. 아래와 같이 변경합니다.

* Instance type : m5.large
* cost-saving : Never

<div align="left">

<img src="../.gitbook/assets/image (69).png" alt="">

</div>

### Task2. SSH Key 구성 및 패키지 설치

[EKS 환경구성](../eks/)에서 적용한 방식과 동일하게 적용합니다. 먼저 Cloud9 IDE 터미널에서 ssh key를 생성합니다.

```
#ssh key를 생성합니다.
cd ~/envvironment
ssh-keygen

```

key는 "eksworkshop"으로 구성합니다.

```
eksworkshop

```

pem 을 설정하고, 권한을 부여합니다.

```
cd ~/envvironment
mv ./eksworkshop ./eksworkshop.pem
chmod 400 ./eksworkshop.pem

```

아래와 같이 aws cli v2.x 로 업그레이드 하고, kubelet 을 설치합니다.

```
#AWS CLI를 2.0으로 업그레이드합니다.
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

#AWS CLI 자동완성을 설치합니다.
which aws_completer
export PATH=/usr/local/bin:$PATH
source ~/.bash_profile
complete -C '/usr/local/bin/aws_completer' aws

# EKS 1.21.2 기반 설치
cd ~
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.21.2/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc

# Kubectl 자동완성 설치
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

# SSM (Session Manager) 설치
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/linux_64bit/session-manager-plugin.rpm" -o "session-manager-plugin.rpm"
sudo sudo yum install -y session-manager-plugin.rpm

```

### Task3. Multus 를 위한 Git Clone 및 S3 구성

Multus 구성을 위한 Git Clone을 수행하고, S3 Bucket에 업로드합니다.&#x20;

```
#multus 구성을 위한 git clone을 실행합니다.
cd ~/envvironment
git clone https://github.com/aws-samples/eks-install-guide-for-multus.git

# US-WEST-2에 S3 Bucket을 생성합니다. Bucket Name은 고유해야 합니다.
cd ~/envvironment
export bucket_name=whchoimultus
aws s3 mb s3://$bucket_name --region us-east-1

# 생성된 Bucket에 git을 업로드 합니다.
cd ~/envvironment
aws s3 sync ./eks-install-guide-for-multus s3://$bucket_name
aws s3 ls s3://$bucket_name/cfn/templates/infra/
aws s3 ls s3://$bucket_name/cfn/templates/nodegroup/

# object가 외부에서 접근할 수 있도록 , Read 권한을 부여합니다.
# aws s3api put-object-acl --bucket $bucket_name --key cfn/templates/infra/eks-infra.yaml --acl public-read  
# aws s3api put-object-acl --bucket $bucket_name --key cfn/templates/nodegroup/eks-nodegroup-multus.yaml --acl public-read
```

## Cloudfomration 기반 배포

### Task4. EKS Infra 배포

EKS Multus 구성을 위한 VPC를 구성합니다. S3에서 앞서 배포되어 있는 eks\_infra.yaml의 Object URL을 복사합니다

![](<../.gitbook/assets/image (314).png>)

**`CloudFormation - 스택 - 스택생성`**  을 선택합니다. 앞서 복사해 둔 eks-infra.yaml 의 Object URL을 Cloudformation S3 URL에 입력하고, 스택을 배포합니다.

![](<../.gitbook/assets/image (272).png>)

Cloudformation Stack의 세부정보를 아래 예를 참조해서 입력합니다.&#x20;

![](<../.gitbook/assets/image (152).png>)

![](<../.gitbook/assets/image (342).png>)

* Stack Name : eks-multus-cluster
* Availability Zone : ap-northeast-2a, ap-northeast-2b
* PublicSubnetAz1Cidr : 10.0.0.0/24
* PublicSubnetAz2Cidr: 10.0.1.0/24
* PrivateSubnetAz1Cidr: 10.0.2.0/24
* PrivateSubnetAz2Cidr: 10.0.3.0/24
* MultusSubnet1Az1Cidr: 10.0.4.0/24
* MultusSubnet1Az2Cidr: 10.0.5.0/24
* MultusSubnet2Az1Cidr: 10.0.6.0/24
* MultusSubnet2Az2Cidr: 10.0.7.0/24
* Bastion Keyname : eksworkshop

### Task5. EKS multus nodegroup 배포

EKS VPC Infra가 배포가 완료되었으면, EKS WorkerNode를 구성하기 위한 준비를 합니다

EKS nodegroup을 배포하기 위해 , Lambda function을 S3에 업로드합니다. Lambda function은 앞서 multus git에서 다운로드 하였습니다.

```
cd ~/envvironment
aws s3 cp  ~/environment/eks-install-guide-for-multus/cfn/templates/nodegroup/lambda_function.zip s3://$bucket_name  

# object가 외부에서 접근할 수 있도록 , Read 권한을 부여합니다.
aws s3api put-object-acl --bucket $bucket_name --key lambda_function.zip --acl public-read  
```

S3에 업로드한 EKS Nodegroup용 Cloudformation Stack yaml 파일의 Object URL을 복사해 둡니다.

![](<../.gitbook/assets/image (489).png>)

Cloudformation 에서 새로운 Stack을 배포합니다.

**`CloudFormation - 스택 - 스택생성`**  을 선택합니다. 앞서 복사해 둔 eks-nodegroup-multus.yaml 의 Object URL을 Cloudformation S3 URL에 입력하고, 스택을 배포합니다. (Task3 배포과정과 동일합니다. )&#x20;

Cloudformation Stack의 세부정보를 아래 예를 참조해서 입력합니다.&#x20;

![](<../.gitbook/assets/image (449).png>)

![](<../.gitbook/assets/image (154).png>)

![](<../.gitbook/assets/image (427).png>)

* Stack Name : ng1
* Cluster Name : eks-multus-cluster
* ClusterControlPlaneSecurityGroup - eks-multus-cluster-EksControlSecurityGroup-xxxx
* NodeGroupName : ng1
* Min/Desired/MaxSize : 1 /2/3
* KeyName : eksworkshop
* VpcId : vpc-eks-multus-cluster
* Subnets : privateAz1-eks-multus-cluster (main primary K8s networking network 입니다.)
* MultusSubnets : multus1Az1 , Multus2Az1
* MultusSecurityGroups : multus-Sg-eks-multus-cluster
* LambdaS3Bucket : 앞서 생성한 Bucket Name (예. whchoimultus)
* LambdaS3Key : lambda\_function.zip&#x20;
* 나머지는 기본값으로 사용합니다.&#x20;

## EKS 관리용 Bastion Host 구성

### Task6. Bastion Host 구성

Bastion Host 에서 EKS 제어를 위해, 기본 설정을 합니다. EC2 대시보드를 선택하고, "MyBastionHost"를 선택하고, Public IPv4를 복사하고 SSH로 접속합니다

![](<../.gitbook/assets/image (413).png>)

아래와 같이 접속한 Bastion Host에 Kubectl 을 설치합니다. &#x20;

```
### Kubelet 설치 
curl -o kubectl https://amazon-eks.s3-us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/kubectl
curl -o kubectl.sha256 https://amazon-eks.s3.us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/kubectl.sha256
openssl sha1 -sha256 kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
kubectl version —short —client

```

Bastion Host에 사용자의 AccessKey/SecreteKey를 구성하고, Kubeconfig를 업데이트 합니다.&#x20;

```
### Access Key/ SecreteKey를 구성합니다. 
aws configure
export AWS_ACCESS_KEY_ID=
export AWS_SECRET_ACCESS_KEY=
export AWS_DEFAULT_REGION=ap-northeast-2
Default output format=json

### Kubeconfig update
aws eks update-kubeconfig --name eks-multus-cluster
kubectl get svc

```

kubectl 명령을 통해 정상적으로 출력이 생성되는지 확인합니다.&#x20;

```
[ec2-user@ip-10-0-0-141 ~]$ kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   172.20.0.1   <none>        443/TCP   15m

```

Bastion Host에서 Kubernetes ConfigMap을 업데이트 합니다.&#x20;

```
### aws-auth-cm 다운로드
curl -o aws-auth-cm.yaml https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/aws-auth-cm.yaml

### 다운 받은 파일을 아래와 같이 수정합니다. file 명 - aws-auth-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::045527256907:role/ng1-NodeInstanceRole-NKIZLNU39KA5
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
### rolearn 부분은 Cloudformation의 EKS Nodegroup을 배포한 출력에서 확인 할 수 있습니다. 

```

rolearn 부분을 확인하기 위해서 ,  nodegroup 스택의 출력에서 NodeInstanceRole 의 값을 확인해서 rolearn을 수정합니다.&#x20;

![](<../.gitbook/assets/image (255).png>)

변경된 aws-auth-cm.yaml 파일을 업데이트 합니다.&#x20;

```
### configmap 업데이트 
kubectl apply -f aws-auth-cm.yaml

### node가 정상적으로 출력되는 지 확인합니다. 수분 정도 소요됩니다. 
kubectl get nodes

```

## Multus PlugIn / App 구성.

### Task7. Multus 구성을 위한 CRD, NAD

Multus Plugin 구성을 위한 NAD DaemonSet를 아래와 같이 Bastion Host에서 실행합니다.&#x20;

```
### Multus Daemonset 구성
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/multus/v3.7.2-eksbuild.1/aws-k8s-multus.yaml

```

Multus Plugin 구성을 위한 NAD CRD를 아래와 같이 Bastion Host에서 실행하고, 2개의 CRD를 구성합니다.&#x20;

```
cat << EOF > ./ipvlan-conf-1.yaml
---
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: ipvlan-conf-1
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "ipvlan",
      "master": "eth1",
      "mode": "l3",
      "ipam": {
        "type": "host-local",
        "subnet": "10.0.4.0/24",
        "rangeStart": "10.0.4.70",
        "rangeEnd": "10.0.4.80",
        "gateway": "10.0.4.1"
      }
    }'
EOF

cat << EOF > ./ipvlan-conf-2.yaml
---
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: ipvlan-conf-2
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "ipvlan",
      "master": "eth2",
      "mode": "l3",
      "ipam": {
        "type": "host-local",
        "subnet": "10.0.6.0/24",
        "rangeStart": "10.0.6.70",
        "rangeEnd": "10.0.6.80",
        "gateway": "10.0.6.1"
      }
    }'
EOF

```

ipvlan-config-1

* main plugin - ipvlan
* eth - eth1
* mode - L3
* IPAM - host-local

ipvlan-config-2

* main plugin - ipvlan
* eth - eth1
* mode - L3
* IPAM - host-local

task 8. App 구성

아래와 같이 4개의 App을 구성해 봅니다. sampleapp-1,2는 eth1개가 binding 되고 ,sampleapp-dual1,2는 eth2개가 바인딩 됩니다.&#x20;

```
### 1개 eth 바인딩 
cat << EOF > ./sampleapp-1.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: sampleapp-1
  annotations:
      k8s.v1.cni.cncf.io/networks: ipvlan-conf-1
spec:
  containers:
  - name: multitool
    command: ["sh", "-c", "trap : TERM INT; sleep infinity & wait"]
    image: praqma/network-multitool
EOF

### 1개 eth 바인딩 
cat << EOF > ./sampleapp-2.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: sampleapp-2
  annotations:
      k8s.v1.cni.cncf.io/networks: ipvlan-conf-2
spec:
  containers:
  - name: multitool
    command: ["sh", "-c", "trap : TERM INT; sleep infinity & wait"]
    image: praqma/network-multitool
EOF

### 2개 eth 바인딩 
cat << EOF > ./sampleapp-dual-1.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: sampleapp-dual-1
  annotations:
      k8s.v1.cni.cncf.io/networks: ipvlan-conf-1, ipvlan-conf-2
spec:
  containers:
  - name: multitool
    command: ["sh", "-c", "trap : TERM INT; sleep infinity & wait"]
    image: praqma/network-multitool
EOF

### 2개 eth 바인딩 
cat << EOF > ./sampleapp-dual-2.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: sampleapp-dual-2
  annotations:
      k8s.v1.cni.cncf.io/networks: ipvlan-conf-1, ipvlan-conf-2
spec:
  containers:
  - name: multitool
    command: ["sh", "-c", "trap : TERM INT; sleep infinity & wait"]
    image: praqma/network-multitool
EOF

```

## Multus 시험

아래와 같이 Container에 접속해서 각 IP 정보들과 같은 서브넷 같의 통신 유무를 확인해 봅니다.

```
kubectl exec -it sampleapp-dual-1 -- ip r
kubectl exec -it sampleapp-dual-2 -- ip r
kubectl exec -it sampleapp-1 -- ip r
kubectl exec -it sampleapp-2 -- ip r

kubectl exec -it sampleapp-1 -- /bin/sh
kubectl exec -it sampleapp-2 -- /bin/sh
kubectl exec -it sampleapp-dual-1 -- /bin/sh
kubectl exec -it sampleapp-dual-2 -- /bin/sh

```

\
