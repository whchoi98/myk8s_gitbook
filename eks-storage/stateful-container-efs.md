---
description: 'Update : 2023-05-11'
---

# Stateful Container-EFS

## EFS 소개

Amazon Elastic File System(Amazon EFS)은 AWS 클라우드 서비스 및 온프레미스 리소스와 함께 사용할 수 있는 간단하고 확장 가능하며 완벽하게 관리되는 탄력성을 기반으로 하는 완전관리형 NFS 파일 시스템을 제공합니다.&#x20;

애플리케이션 중단 없이 온디맨드 방식으로 페타바이트까지 확장할 수 있도록 구축되었으며, 파일을 추가 및 제거함에 따라 자동으로 확장 및 축소되므로 워크로드의 성장을 수용하기 위해 용량을 프로비저닝하고 관리할 필요가 없습니다.&#x20;

Amazon EFS는 Network File System 버전 4(NFSv4.1 및 NFSv4.0) 프로토콜을 지원하므로 현재 사용하는 애플리케이션과 도구들는 Amazon EFS와 원활하게 작동합니다. 여러 Amazon EC2 인스턴스는 Amazon EFS 파일 시스템에 동시에 액세스할 수 있으므로 둘 이상의 인스턴스 또는 서버에서 실행되는 워크로드 및 애플리케이션에 공통의 데이터 소스를 제공합니다.



## EFS 파일 시스템 생성

### 1.환경변수 구성

EFS 파일 시스템은 AWS Management Console 또는 AWS CLI를 사용하여 생성하고 구성할 수 있습니다. EFS 파일 시스템은 EKS 클러스터 VPC 내에서 실행되는 EC2 워커 노드에서 동시에 액세스할 수 있습니다. 인스턴스는 Mount Target이라는 네트워크 인터페이스를 사용하여 파일 시스템에 연결합니다. 먼저 EKS 클러스터의 이름, 클러스터가 배포된 VPC 및 해당 VPC와 연결된 IPv4 CIDR 블록과 관련된 환경 변수 집합을 정의합니다.

(앞서 LAB을 수행했다면, cluster 이름과 VPC ID 정보는 환경변수에 이미 저장되어 있습니다.)

```
echo ${ekscluster_name}
echo ${vpc_ID}
CIDR_BLOCK=$(aws ec2 describe-vpcs --vpc-ids $vpc_ID --query "Vpcs[].CidrBlock" --output text)
echo ${CIDR_BLOCK}

```

### 2.EFS접근을 위한 Security Group 생성

mount target과 연결할 security group을 만듭니다. 그런 다음 EKS 클러스터 VPC의 CIDR 블록에 속하는 IP 주소에서 포트 2049의 NFS 프로토콜을 사용하는 모든 인바운드 트래픽을 허용하는 수신 규칙을 이 보안 그룹에 추가합니다. 이 규칙은 EKS 클러스터의 모든 워커 노드에서 파일 시스템에 대한 NFS 액세스를 허용합니다.

```
MOUNT_TARGET_GROUP_NAME="eks-efs-group"
MOUNT_TARGET_GROUP_DESC="NFS access to EFS from EKS worker nodes"
MOUNT_TARGET_GROUP_ID=$(aws ec2 create-security-group --group-name $MOUNT_TARGET_GROUP_NAME --description "$MOUNT_TARGET_GROUP_DESC" --vpc-id $vpc_ID | jq --raw-output '.GroupId')
aws ec2 authorize-security-group-ingress --group-id $MOUNT_TARGET_GROUP_ID --protocol tcp --port 2049 --cidr $CIDR_BLOCK

```

3.EFS 생성

이제 EFS를 생성합니다.

```
FILE_SYSTEM_ID=$(aws efs create-file-system | jq --raw-output '.FileSystemId')

```

다음 명령을 사용하여 파일 시스템의 LifeCycleState를 확인하고 다음 단계로 진행하기 전에 **`"available"`**으로 변경될 때까지 기다립니다.

```
aws efs describe-file-systems --file-system-id $FILE_SYSTEM_ID

```

생성한 EKS 클러스터는 클러스터 VPC의 Public Subnet에 워커 노드(EC2)로 구성됩니다.&#x20;

각 Public Subnet은 서로 다른 가용 영역에 상주합니다. 앞에서 언급했듯이 워커 노드는 mount target을 사용하여 EFS 파일 시스템에 연결합니다. EKS 클러스터의 워커 노드가 모두 파일 시스템에 액세스할 수 있도록 각 EKS 클러스터 VPC의 가용 영역에 마운트 대상을 생성하는 것이 가장 좋습니다. 다음 명령 세트는 EKS Cluster가 구성된 VPC에서 퍼블릭 서브넷을 식별하고 각 서브넷에 mount target을 생성하고 해당 mount target을 위에서 생성한 보안 그룹과 연결합니다.

```
aws efs create-mount-target --file-system-id $FILE_SYSTEM_ID --subnet-id $PublicSubnet01 --security-groups $MOUNT_TARGET_GROUP_ID
aws efs create-mount-target --file-system-id $FILE_SYSTEM_ID --subnet-id $PublicSubnet02 --security-groups $MOUNT_TARGET_GROUP_ID  
aws efs create-mount-target --file-system-id $FILE_SYSTEM_ID --subnet-id $PublicSubnet03 --security-groups $MOUNT_TARGET_GROUP_ID 

```

다음 명령을 사용하여 mount target의 LifeCycleState를 확인하고 **`"available"`** 으로 변경될 때까지 기다렸다가 다음 단계로 진행합니다. 모든 탑재 대상이 사용 가능한 상태로 전환되는 데 몇 분 정도 걸립니다.

```
aws efs describe-mount-targets --file-system-id $FILE_SYSTEM_ID
```

AWS Management Console의 EFS 대시보드에서 탑재 대상의 상태를 확인할 수도 있습니다. 방금 만든 파일 시스템을 선택한 다음 네트워크 액세스 관리를 클릭하여 탑재 대상을 확인합니다.

**`Amazon EFS - 파일 시스템 - 네트워크`**

<figure><img src="../.gitbook/assets/image (106).png" alt=""><figcaption></figcaption></figure>

## EFS CSI 드라이버 생성

### 3.Service Account / IAM 정책 생성

CSI 드라이버 Service Account에서 AWS API를 호출할 수 있도록 하는 IAM 정책을 만듭니다.

```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/docs/iam-policy-example.json
aws iam create-policy \
    --policy-name AmazonEKS_EFS_CSI_Driver_Policy \
    --policy-document file://iam-policy-example.json
    
```

### 4.IAM Role 생성 및 정책, Service Account 연결

IAM 역할을 생성하여 여기에 IAM 정책을 연결합니다. Kubernetes 서비스 계정에 IAM 역할 ARN을 추가하고 IAM 역할에 Kubernetes 서비스 계정 이름을 추가합니다. `eksctl` 또는 AWS CLI를 사용하여 역할을 생성할 수 있습니다.

```
eksctl create iamserviceaccount \
    --cluster ${ekscluster_name} \
    --namespace kube-system \
    --name efs-csi-controller-sa \
    --attach-policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AmazonEKS_EFS_CSI_Driver_Policy \
    --approve \
    --region ${AWS_REGION}
    
```

5.Amazon EFS 드라이버 설치

Helm 또는 매니페스트를 사용하여 Amazon EFS CSI 드라이버를 설치합니다.

```
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
helm repo update
helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
    --namespace kube-system \
    --set image.repository=602401143452.dkr.ecr.ap-northeast-2.amazonaws.com/eks/aws-efs-csi-driver \
    --set controller.serviceAccount.create=false \
    --set controller.serviceAccount.name=efs-csi-controller-sa
    
```



6.샘플 어플리케이션 배포 시험

[Amazon EFS Container Storage Interface(CSI) 드라이버](https://github.com/kubernetes-sigs/aws-efs-csi-driver) GitHub 리포지토리의 [동적 프로비저닝](https://github.com/kubernetes-sigs/aws-efs-csi-driver/tree/master/examples/kubernetes/dynamic\_provisioning) 예제를 사용해 봅니다.

Amazon EFS의 `StorageClass` 매니페스트를 다운로드하고, 스토리지 클래스를 배포합니다.

```
mkdir ~/environment/efs
cd ~/environment/efs
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/dynamic_provisioning/specs/storageclass.yaml
sed -i "s/fileSystemId: fs-92107410/fileSystemId: $FILE_SYSTEM_ID/g" storageclass.yaml 
kubectl apply -f ~/environment/efs/storageclass.yaml 
kubectl get sc,pv

```

`PersistentVolumeClaim`을 사용하는 포드를 배포하여 자동 프로비저닝을 테스트합니다. Persistent 볼륨이 `Bound` 상태가 `PersistentVolumeClaim`으로 설정되어 생성되었는지 확인합니다.

```
cd ~/environment/efs
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/dynamic_provisioning/specs/pod.yaml
kubectl apply -f pod.yaml
kubectl get sc,pv

```

EFS 볼륨에 마운트 되어 있는 Pod가 정상적으로 데이터를 쓰고 있는지 확인해 봅니다.

```
kubectl exec efs-app -- bash -c "cat data/out"

```
