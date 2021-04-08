# Stateful Container-EBS

## AWS EBS CSI Driver 구성

참조 URL : [https://docs.aws.amazon.com/ko\_kr/eks/latest/userguide/ebs-csi.html](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/ebs-csi.html)

[Amazon EBS CSI\(Container Storage Interface\) 드라이버](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)에서는 Amazon EKS 클러스터가 영구 볼륨\(Persistant Volume, PV\)을 위해 Amazon EBS 볼륨의 수명 주기를 관리할 수 있게 해주는 CSI 인터페이스를 제공합니다.

{% hint style="info" %}
이 드라이버는 Kubernetes 버전 1.14 이상에서만 지원됩니다. Amazon EKS 클러스터 및 노드. Fargate에서는 드라이버가 지원되지 않습니다. Amazon EKS 클러스터에서는 Amazon EBS CSI 드라이버의 알파 기능을 지원하지 않습니다. 이 드라이버는 베타 릴리스 버전이며, Amazon EKS에서 프로덕션용으로 테스트를 완료하여 지원됩니다. 세부 정보는 변경될 수 있지만 드라이버에 대한 지원은 종료되지 않습니. 
{% endhint %}

### 1.IAM 정책 구성

CSI 드라이버는 Kubernetes Pod 세트로 배포됩니다. 이러한 포드에는 볼륨 생성 및 삭제, 클러스터를 구성하는 EC2 worker node에 볼륨 연결과 같은 EBS API 작업을 수행 할 수있는 권한이 있어야합니다.

[Amazon EBS CSI\(Container Storage Interface\) 드라이버](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)는 Amazon EKS 클러스터가 PV \(Persistant Volume\) 볼륨을 위해 Amazon EBS 볼륨의 Life Cycle 관리할 수 있게 해주는 CSI 인터페이스를 제공합니다.

먼저 정책 JSON 문서를 다운로드하고이 문서에서 IAM 정책을 생성하십시오.

* CSI driver를 위한 IAM Policy 샘플 다운로드
* IAM Policy 생성 

```text
export EBS_CSI_POLICY_NAME="Amazon_EBS_CSI_Driver"
mkdir ${HOME}/environment/ebs_statefulset
cd ${HOME}/environment/ebs_statefulset
curl -sSL -o ebs-csi-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/docs/example-iam-policy.json
export EBS_CNI_POLICY_NAME="Amazon_EBS_CSI_Driver"

aws iam create-policy \
  --region ${AWS_REGION} \
  --policy-name ${EBS_CNI_POLICY_NAME} \
  --policy-document file://${HOME}/environment/ebs_statefulset/ebs-csi-policy.json
    
  export EBS_CNI_POLICY_ARN=$(aws --region ${AWS_REGION} iam list-policies --query 'Policies[?PolicyName==`'$EBS_CNI_POLICY_NAME'`].Arn' --output text)
  echo $EBS_CNI_POLICY_ARN
  
```

### 2. Kubernetes 서비스 계정에 대한 IAM 역할 구성

IAM 역할을 Kubernetes 서비스 계정과 연결할 수 있습니다. 그러면 이 서비스 계정은 해당 서비스 계정을 사용하는 모든 포드의 컨테이너에 AWS 권한을 제공 할 수 있습니다. 이 기능을 사용하면 해당 노드의 포드가 AWS API를 호출 할 수 있도록 더 이상 Amazon EKS 노드 IAM 역할에 대한 확장 권한을 제공 할 필요가 없습니다.

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

![](../.gitbook/assets/image%20%28114%29.png)

### 3. EBS CSI 드라이버 구성 및 배

Helm을 통해서 EBS CSI 드라이버를 다운로드 받습니다.

```text
# add the aws-ebs-csi-driver as a helm repo
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver

# search for the driver
helm search  repo aws-ebs-csi-driver

```

배포를 합니다.

```text
helm upgrade --install aws-ebs-csi-driver \
  --namespace kube-system \
  --set serviceAccount.controller.create=false \
  --set serviceAccount.snapshot.create=false \
  --set enableVolumeScheduling=true \
  --set enableVolumeResizing=true \
  --set enableVolumeSnapshot=true \
  --set serviceAccount.snapshot.name=ebs-csi-controller-irsa \
  --set serviceAccount.controller.name=ebs-csi-controller-irsa \
  aws-ebs-csi-driver/aws-ebs-csi-driver
```

##  스토리지 클래스 생성. 

클러스터가 사용할 스토리지 클래스를 정의해야 하고, PV 클레임에 대한 기본 스토리지 클래스를 정의해야 합니다.

![](../.gitbook/assets/image%20%2893%29.png)

스토리지 클래스를 정의합니다.

```text
cat << EoF > ${HOME}/environment/ebs_statefulset/mysql-storageclass.yaml
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
EoF

```

Storage Class를 생성합니다.

```text
kubectl create -f ${HOME}/environment/ebs_statefulset/mysql-storageclass.yaml

```

생성된 스토리지 클래스를 확인합니다.

```text
kubectl describe storageclass mysql-gp2

```

출력 결과를 확인합니다.

```text
whchoi:~/environment/ebs_statefulset $ kubectl describe storageclass mysql-gp2
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

mysql-gp2 를 storageClassName으로 선언합니다.

```text
volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: mysql-gp2
      resources:
        requests:
          storage: 10Gi
```

## ConfigMap 생성

컨피그맵은 key-vlaue Pair로 기밀이 아닌 데이터를 저장하는 데 사용하는 API 오브젝트입니. [파드](https://kubernetes.io/ko/docs/concepts/workloads/pods/pod-overview/)는 [볼륨](https://kubernetes.io/ko/docs/concepts/storage/volumes/) 내에서 환경 변수, 커맨드-라인 인수 또는 구성 파일로 컨피그맵을 사용할 수 있습니다.

컨피그맵을 사용하면 [컨테이너 이미지](https://kubernetes.io/ko/docs/reference/glossary/?all=true#term-image)에서 환경별 구성을 분리하여, 애플리케이션을 쉽게 이식할 수 있습니다.

### 1.namespace /configmap 생성

Stateful DB를 생성할 것입니다. 새로운 namespace를 생성합니다.

```text
kubectl create namespace mysql

```

Configmap을 생성합니다. 

```text
cd ${HOME}/environment/ebs_statefulset

cat << EoF > ${HOME}/environment/ebs_statefulset/mysql-configmap.yaml
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
EoF

```

Configmap을 생성합니다.

```text
kubectl create -f ${HOME}/environment/ebs_statefulset/mysql-configmap.yaml

```

## MySql을 위한 Service 생성

Staetefulset 멤버를 위한 Headless service 구성과 Client 접속을 위한 구성의 Service 매니페스트를 구성합니다.

```text
cat << EoF > ${HOME}/environment/ebs_statefulset/mysql-services.yaml
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
EoF

```

mysql, mysql-read를 생성합니다.

```text
kubectl create -f ${HOME}/environment/ebs_statefulset/mysql-services.yaml

```

## StatefuleSet 구성

mysql-statefulset.yml을 기반으로 StatefuleSet을 구성합니다.

```text
cd ${HOME}/environment/ebs_statefulset
wget https://eksworkshop.com/beginner/170_statefulset/statefulset.files/mysql-statefulset.yaml
kubectl apply -f ${HOME}/environment/ebs_statefulset/mysql-statefulset.yaml

```

정상적으로 생성되었는지 확인합니다. 

{% hint style="success" %}
생성 되는데 5분 정도 시간이 소요됩니다.
{% endhint %}

```text
kubectl -n mysql rollout status statefulset mysql

```

동적 생성된 PVC를 확인합니다.

```text
kubectl -n mysql get pvc -l app=mysql

```

아래와 같은 결과를 확인할 수 있습니다.

```text
whchoi:~/environment/ebs_statefulset $ kubectl -n mysql get pvc -l app=mysql
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-mysql-0   Bound    pvc-82184a36-d4c4-4371-8deb-a426836af336   10Gi       RWO            mysql-gp2      2m41s
data-mysql-1   Bound    pvc-b492e2aa-1539-447f-a79e-f2316a80dea3   10Gi       RWO            mysql-gp2      116s
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

![](../.gitbook/assets/image%20%28199%29.png)

## SQL Test 

### 1.SQL 테스팅.

다음 명령을 실행하여 **mysql-client** 를 통 일부 데이터를 **mysql-0.mysql** 에 보낼 수 있습니다 .

```text
kubectl -n mysql run mysql-client --image=mysql:5.7 -i --rm --restart=Never --\
  mysql -h mysql-0.mysql <<EOF
CREATE DATABASE test;
CREATE TABLE test.messages (message VARCHAR(250));
INSERT INTO test.messages VALUES ('hello, from mysql-client');
EOF

```

정상적으로 read되는 지 확인해 봅니다.

```text
kubectl -n mysql run mysql-client --image=mysql:5.7 -it --rm --restart=Never --\
>   mysql -h mysql-read -e "SELECT * FROM test.messages"

```

아래와 같이 출력됩니다.

```text
whchoi:~/environment/ebs_statefulset $ kubectl -n mysql run mysql-client --image=mysql:5.7 -it --rm --restart=Never --\
>   mysql -h mysql-read -e "SELECT * FROM test.messages"
+--------------------------+
| message                  |
+--------------------------+
| hello, from mysql-client |
+--------------------------+
```

상호간 분산 테스트를 위해서 아래 명령을 통해 확인 해 봅니다.

```text
kubectl -n mysql run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
   bash -ic "while sleep 1; do mysql -h mysql-read -e 'SELECT @@server_id,NOW()'; done"

```

### 2. Container 장애 시험

새로운 Cloud9창에서 아래 명령을 통해 MySQL을 응답하지 않는 상태로 만듭니다.

```text
kubectl -n mysql exec mysql-1 -c mysql -- mv /usr/bin/mysql /usr/bin/mysql.off

```

앞서 상호 분산테스트 하였던 Cloud9창에서 어떤 변화가 일어나는지 확인해 봅니다. 아래 명령을 통해 mysql 상태를 확인해 봅니다.

```text
kubectl -n mysql get pod mysql-1

```

아래와 같은 출력결과를 확인할 수 있습니다.

```text
whchoi:~/environment $ kubectl -n mysql get pod mysql-1
NAME      READY   STATUS    RESTARTS   AGE
mysql-1   1/2     Running   0          15m
```

다시 정상으로 복귀 시키고, 확인해 봅니다.

```text
kubectl -n mysql exec mysql-1 -c mysql -- mv /usr/bin/mysql.off /usr/bin/mysql
kubectl -n mysql get pod mysql-1

```

아래와 같은 결과를 확인할 수 있습니다.

```text
whchoi:~/environment $ kubectl -n mysql get pod mysql-1
NAME      READY   STATUS    RESTARTS   AGE
mysql-1   2/2     Running   0          17m
```

### 3.Pod 삭제 시험

실패한 포드를 시뮬레이션하려면 다음 명령을 사용하여 mysql-1 포드를 삭제합니다.

```text
kubectl -n mysql delete pod mysql-1

```

앞서 Cloud9 창에서 다시 100에서만 응답되는 것을 확인할 수 있습니다.

StatefulSet 컨트롤러는 실패한 포드를 인식하고 동일한 이름과 링크를 가진 복제본 수를 유지하기 위해 새 포드를 만듭니다.

```text
kubectl -n mysql get pod mysql-1 -w

```

```text
whchoi:~/environment $ kubectl -n mysql get pod mysql-1 -w
NAME      READY   STATUS    RESTARTS   AGE
mysql-1   2/2     Running   0          40s
```

### 4.Scaling 시험

이제 스케일링을 시험해 봅니다. replica를 3개까지 확장합니다.

```text
kubectl -n mysql scale statefulset mysql --replicas=3

```

아래 명령을 통해 확장되는 것을 확인합니다.

```text
kubectl -n mysql rollout status statefulset mysql

```

다시 앞서 Cloud9 창에서 다시 100,101,102에서 분산해 Read가 일어나는 것을 확인 할 수 있습니다.

새로 배포 된 팔로워 \(mysql-2\)가 다음 명령으로 동일한 데이터 세트를 가지고 있는지 확인합니다. 동일한 메세지가 출력됩니다.

```text
kubectl -n mysql run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
 mysql -h mysql-2.mysql -e "SELECT * FROM test.messages"
 
```

이제 다시 축소해 봅니다. 하지만 데이터 또는 PVC를 삭제하지는 않습니다. 수동삭제 해야 합니다.

```text
kubectl -n mysql  scale statefulset mysql --replicas=2
kubectl -n mysql get pods -l app=mysql

```

여전히 PVC가 존재하는 지 확인해 봅니다.

```text
kubectl -n mysql  get pvc -l app=mysql

```

```text
whchoi:~/environment $ kubectl -n mysql  get pvc -l app=mysql
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-mysql-0   Bound    pvc-82184a36-d4c4-4371-8deb-a426836af336   10Gi       RWO            mysql-gp2      31m
data-mysql-1   Bound    pvc-b492e2aa-1539-447f-a79e-f2316a80dea3   10Gi       RWO            mysql-gp2      31m
data-mysql-2   Bound    pvc-ba84a117-2dc0-455b-ae2a-206466191114   10Gi       RWO            mysql-gp2      8m35s
```



