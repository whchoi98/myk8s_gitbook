# Stateful Container-EBS

## AWS EBS CSI Driver 구성

### 1.IAM 정책 구성

CSI 드라이버는 Kubernetes Pod 세트로 배포됩니다. 이러한 포드에는 볼륨 생성 및 삭제, 클러스터를 구성하는 EC2 worker node에 볼륨 연결과 같은 EBS API 작업을 수행 할 수있는 권한이 있어야합니다.

[Amazon EBS CSI\(Container Storage Interface\) 드라이버](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)는 Amazon EKS 클러스터가 PV \(Persistant Volume\) 볼륨을 위해 Amazon EBS 볼륨의 Life Cycle 관리할 수 있게 해주는 CSI 인터페이스를 제공합니다.

먼저 정책 JSON 문서를 다운로드하고이 문서에서 IAM 정책을 생성하십시오.

```text
mkdir ~/environment/ebs_csi_driver
cd ~/environment/ebs_csi_driver
curl -sSL -o ebs-cni-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/v0.4.0/docs/example-iam-policy.json

export EBS_CNI_POLICY_NAME="Amazon_EBS_CSI_Driver"

aws iam create-policy \
  --region ${AWS_REGION} \
  --policy-name ${EBS_CNI_POLICY_NAME} \
  --policy-document file://ebs-cni-policy.json
  
  export EBS_CNI_POLICY_ARN=$(aws --region ${AWS_REGION} iam list-policies --query 'Policies[?PolicyName==`'$EBS_CNI_POLICY_NAME'`].Arn' --output text)
  echo $EBS_CNI_POLICY_ARN
  
```

### 2. Kubernetes 서비스 계정에 대한 IAM 역할 구성

`eksctl`가 만든 IAM 정책이 포함 된 IAM 역할을 생성하고, 이를 `ebs-csi-controller-irsa`CSI 드라이버가 사용할 Kubernetes 서비스 계정과 연결합니다 .

```text
# 앞서서 이 과정을 빈번하게 수행했습니다. 만약 수행했다면 이미 연결되어 있다고 출력될 것입니다.
eksctl utils associate-iam-oidc-provider --region=$AWS_REGION --cluster=eksworkshop --approve

eksctl create iamserviceaccount --cluster eksworkshop \
  --name ebs-csi-controller-irsa \
  --namespace kube-system \
  --attach-policy-arn $EBS_CNI_POLICY_ARN \
  --override-existing-serviceaccounts \
  --approve
  
```

아래와 같이 IAM 대시보드에서 확인 할 수 있습니다.

![](../.gitbook/assets/image%20%28112%29.png)

### 3. EBS CSI 드라이버 구성

아래 파일을 먼저 다운로드 받습니다.

```text
cd ~/environment/ebs_csi_driver
wget https://raw.githubusercontent.com/aws-samples/eks-workshop/main/content/beginner/170_statefulset/ebs_csi_driver.files/kustomization.yml
wget https://raw.githubusercontent.com/aws-samples/eks-workshop/main/content/beginner/170_statefulset/ebs_csi_driver.files/deployment.yml
wget https://raw.githubusercontent.com/aws-samples/eks-workshop/main/content/beginner/170_statefulset/ebs_csi_driver.files/attacher-binding.yml
wget https://raw.githubusercontent.com/aws-samples/eks-workshop/main/content/beginner/170_statefulset/ebs_csi_driver.files/provisioner-binding.yml

```

배포를 합니다.

```text
kubectl apply -k ~/environment/ebs_csi_driver

```

##  스토리지 클래스 생성. 

클러스터가 사용할 스토리지 클래스를 정의해야 하고, PV 클레임에 대한 기본 스토리지 클래스를 정의해야 합니다.

![](../.gitbook/assets/image%20%2893%29.png)

새로운 디렉토리를 만들고 파일을 복제합니다.

```text
mkdir ~/environment/templates
cd ~/environment/templates
cp ~/environment/myeks/storage-class/mysql-storageclass.yml ./

```

아래와 같은 파일을 확인 할 수 있습니다. \(mysql-storageclass.yml\)

```text
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: mysql-gp2
provisioner: ebs.csi.aws.com # Amazon EBS CSI driver
parameters:
  type: gp2
  encrypted: 'true' # EBS volumes will always be encrypted by default
reclaimPolicy: Delete
mountOptions:
- debug

```

Storage Class를 생성합니다.

```text
kubectl create -f mysql-storageclass.yml 

```

생성된 스토리지 클래스를 확인합니다.

```text
kubectl describe storageclass mysql-gp2

```

출력 결과를 확인합니다.

```text
whchoi98:~/environment/templates $ kubectl describe storageclass mysql-gp2
Name:                  mysql-gp2
IsDefaultClass:        No
Annotations:           <none>
Provisioner:           ebs.csi.aws.com
Parameters:            encrypted=true,type=gp2
AllowVolumeExpansion:  <unset>
MountOptions:
  debug
ReclaimPolicy:      Delete
VolumeBindingMode:  Immediate
Events:             <none>
```

## ConfigMap 생성

컨피그맵은 key-vlaue Pair로 기밀이 아닌 데이터를 저장하는 데 사용하는 API 오브젝트입니. [파드](https://kubernetes.io/ko/docs/concepts/workloads/pods/pod-overview/)는 [볼륨](https://kubernetes.io/ko/docs/concepts/storage/volumes/) 내에서 환경 변수, 커맨드-라인 인수 또는 구성 파일로 컨피그맵을 사용할 수 있습니다.

컨피그맵을 사용하면 [컨테이너 이미지](https://kubernetes.io/ko/docs/reference/glossary/?all=true#term-image)에서 환경별 구성을 분리하여, 애플리케이션을 쉽게 이식할 수 있습니다.

### 1.namespace /configmap 생성

Stateful이 필요한 DB를 생성할 것입니다. 새로운 namespace를 생성합니다.

```text
kubectl create namespace mysql

```

Configmap을 생성합니다. mysql구성을 위한 샘플 configmap을 다운받습니다.

```text
cd ~/environment/templates
cp ~/environment/myeks/configmap/mysql-configmap.yml ./

```

mysql-configmap.yml을 확인합니다.

ConfigMap은 master.cnf, slave.cnf를 저장하고 StatefulSet에 정의 된 리더 및 팔로어 포드를 초기화 할 때 전달합니다.  __master.cnf는 바이너 로그 옵션 \(log-bin\)이 있고,팔로어 서버로 데이터 변경 부분을 전송합니다. 또한 slave.cnf는 팔로어 서버로 super-read-only로 구동됩니다.

```text
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: mysql
  labels:
    app: mysql
data:
  master.cnf: |
    # Apply this config only on the leader.
    [mysqld]
    log-bin
  slave.cnf: |
    # Apply this config only on followers.
    [mysqld]
    super-read-only
```

Configmap을 생성합니다.

```text
kubectl create -f ~/environment/templates/mysql-configmap.yml

```

## MySql을 위한 Service 생성

Staetefulset 멤버를 위한 Headless service 구성과 Client 접속을 위한 구성의 서비스 매니페스트를 구성합니다.

```text
cd ~/environment/templates
cp ~/environment/myeks/configmap/mysql-servies.yml ./
```

mysql-service.yml을 확인합니다.

```text
# Headless service for stable DNS entries of StatefulSet members.
apiVersion: v1
kind: Service
metadata:
  namespace: mysql
  name: mysql
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  clusterIP: None
  selector:
    app: mysql
---
# Client service for connecting to any MySQL instance for reads.
# For writes, you must instead connect to the leader: mysql-0.mysql.
apiVersion: v1
kind: Service
metadata:
  namespace: mysql
  name: mysql-read
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  selector:
    app: mysql
```

mysql-service를 생성합니다.

## StatefuleSet 구성

StatefuleSet을 생성합니다. StatefulSet구성을 위한 샘플 statefuleset 매니페스트 파을 다운받습니다.

```text
cd ~/environment/templates
cp ~/environment/myeks/configmap/mysql-statefulset.yml ./

```

mysql-statefulset.yml을 기반으로 StatefuleSet을 구성합니다.

```text
kubectl create -f ~/environment/templates/mysql-statefulset.yml

```

정상적으로 생성되었는지 확인합니다. 

```text
kubectl -n mysql rollout status statefulset mysql
kubectl -n mysql get pods -l app=mysql
```

아래와 같은 결과를 확인할 수 있습니다.

```text
whchoi98:~/environment/myeks (master) $ kubectl -n mysql rollout status statefulset mysql
Waiting for 3 pods to be ready...
Waiting for 2 pods to be ready...
Waiting for 1 pods to be ready...
partitioned roll out complete: 3 new pods have been updated...

whchoi98:~/environment/myeks (master) $ kubectl -n mysql get pods -l app=mysql --watch
NAME      READY   STATUS    RESTARTS   AGE
mysql-0   2/2     Running   0          3m59s
mysql-1   2/2     Running   0          3m12s
mysql-2   2/2     Running   0          2m17s
```

PVC를 아래 명령을 통해 확인합니다.

```text
kubectl -n mysql get pvc -l app=mysql

```

아래와 같은 출력 결과를 확인 할 수 있습니다.

```text
whchoi98:~/environment/myeks (master) $ kubectl -n mysql get pvc -l app=mysql
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-mysql-0   Bound    pvc-c08c7ae2-e3da-46a8-8910-54ac87e0ee20   10Gi       RWO            mysql-gp2      4m58s
data-mysql-1   Bound    pvc-f68bfe62-2ce3-4371-b89a-def75716a106   10Gi       RWO            mysql-gp2      4m11s
data-mysql-2   Bound    pvc-8057ab7e-ffd8-41af-a6cd-72eb178d3e4e   10Gi       RWO            mysql-gp2      3m16s
```

EC2 대시보드의 볼륨에서도 동일한 결과를 확인 할 수 있습니다.

![](../.gitbook/assets/image%20%2899%29.png)

SQL Test 





